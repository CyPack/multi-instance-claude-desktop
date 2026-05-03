# Anti-Patterns — What NOT To Try (We Did, It Broke)

Things that LOOK like good ideas but BROKE this project. Read this BEFORE proposing "improvements". Each entry: anti-pattern → why it seems good → what broke → don't do this.

---

## AP01 — Wrap Electron in `systemd-run --user --scope`

**Why it seems good**: systemd scopes give nice lifecycle management. `systemctl stop scope` kills entire cgroup atomically. KillMode=control-group gives proper SIGTERM→SIGKILL escalation. We can name our units and have them auto-collected on exit.

**What broke**: BLACK SCREEN. Electron's renderer process initialized in the wrapped scope environment with subtly different XDG bindings, file descriptors, or DBUS attribution. GPU compositor failed to init. Window opened with title bar but no content.

**Verified workaround attempts (all failed)**:
- `--disable-gpu` flag → black screen persisted (root cause was env, not GPU)
- Different KillMode values → no effect on render
- Adding `--setenv` overrides → still broken

**Right pattern**: Use `nohup` direct launch. Watcher pattern for lifecycle. See `bin/claude-launch` `launch_main()` and `launch_profile()`.

**Reference error**: E03 in `errors.md`.

---

## AP02 — Redirect Electron stdout/stderr to a real file

**Why it seems good**: Capture Electron output for debugging. We had `>>$LAUNCH_LOG 2>&1`.

**What broke**: BLACK SCREEN (variant 2). On certain Wayland+NVIDIA configurations, Electron's renderer detects stdout being a regular file and disables GPU compositing as a "headless safety". Window structure created, content not rendered.

**Right pattern**: Always redirect to `/dev/null`.
```bash
nohup ... claude-desktop ... >/dev/null 2>&1 &
```

**For debugging**: Use `~/.config/Claude-N/logs/main.log` (Claude Desktop's own log, not Electron stdout). Or `~/.cache/claude-desktop-debian/launcher.log` (system launcher log).

**Reference error**: E04.

---

## AP03 — Track watcher process by cmdline pgrep

**Why it seems good**: Easy. `pgrep -af "supervise_profile $N"` should find the watcher.

**What broke**: Watchers are bash subshells `( supervise_profile ... ) &`. Their cmdline shows the PARENT script's cmdline (`bash /home/ayaz/.local/bin/claude-launch <N>`), NOT the function name. pgrep finds 0 matches.

**Right pattern**: Write watcher PID to a file at spawn time. Read file later.
```bash
( supervise_profile $N $electron_pid ) &
WATCHER_PID=$!
disown
printf '%s\n%s\n' "$electron_pid" "$WATCHER_PID" > "$LAUNCH_CACHE/profile-$N.pid"
```

PID file has 2 lines: line 1 = electron, line 2 = watcher. Singleton invariant.

**Reference error**: E12.

---

## AP04 — Watch `kill -0 $electron_pid` for cleanup trigger

**Why it seems good**: Process death is the obvious "app closed" signal. `kill -0` is fast and standard.

**What broke**: Claude Desktop has macOS-style background mode. X button destroys window but Electron process STAYS ALIVE. Process never dies → watcher never fires → MCP children never cleaned up.

**Right pattern**: Watch for WINDOW destruction, not process death.
```bash
while true; do
  has_window=$(wmctrl -lx | grep -c "$WM_CLASS")
  if [ "$has_window" = 0 ]; then
    missing_count=$((missing_count + 1))
    [ $missing_count -ge 3 ] && break  # threshold for false-positive resistance
  else
    missing_count=0
  fi
  sleep 2
done
cleanup
```

3-scan threshold prevents false positives from minimize/raise.

**Reference error**: E09.

---

## AP05 — `xdotool windowmap + activate` to "reopen" hidden window

**Why it seems good**: When user clicks dock icon for already-running profile, we should "raise" the existing window. xdotool windowmap+activate sounds like the right tool.

**What broke**: After X-button click, the window is DESTROYED (not just hidden). xdotool windowmap finds no window to map. User sees nothing happen.

**Right pattern**: Trigger Electron's built-in single-instance event.
```bash
# Spawn secondary Electron with same --user-data-dir
nohup env ... claude-desktop --user-data-dir=$dir --class=Claude-$N \
  >/dev/null 2>&1 &
disown

# Result:
#   - Secondary Electron acquires lock attempt → fails (primary holds it)
#   - Secondary exits immediately
#   - Primary receives `second-instance` IPC event
#   - Primary calls createWindow() → fresh window appears
```

This is exactly how Profile 1's default `.desktop` Exec line works. We replicate it for Profile 2/3/4.

**Reference error**: E09.

---

## AP06 — Remove cowork-vm-service from Profile 1's process tree

**Why it seems good**: "Cowork shouldn't be tied to a specific profile." Spawn it as a standalone systemd service.

**What broke**: Cowork expects to be a child of an Electron process. Initialization handshake happens over inherited stdio file descriptors, not just the socket. Spawning cowork standalone misses this handshake → cowork comes up but can't authenticate IPC requests.

**Right pattern**: Accept the upstream architecture. Cowork lives where it lives. If Profile 1 closes, cowork dies, gets respawned on next launch via launcher-common.sh.

**Document**: User should know Profile 1 should be the LAST to close.

**Reference error**: E08.

---

## AP07 — Patch app.asar to fix WM_CLASS / MCP cleanup / etc.

**Why it seems good**: Source-level fix is "cleaner". Just edit the JS, repackage asar.

**What broke**: Multiple problems:
1. `app.asar` is in `/usr/lib/claude-desktop/`, owned by root. Changes lost on `sudo dnf upgrade claude-desktop`.
2. asar files have integrity checksums. Modified asar → app may refuse to run.
3. Patches need re-applied after every Anthropic release. Maintenance burden.
4. Community forks (Cresnova/claude-desktop-mcp-fix) prove this is fragile.

**Right pattern**: All fixes live in OUR script (claude-launch), in OUR home dir. Wrappers + watchers + xdotool. Survives upgrades cleanly.

---

## AP08 — Use `pkill -f electron` for cleanup

**Why it seems good**: Simple. pkill all electrons.

**What broke**: Kills OTHER Electron apps too — Discord, Slack, VSCode, OBS. Massive collateral damage.

**Right pattern**: Use specific user-data-dir match.
```bash
# Bad:
pkill -f electron

# Good (only Profile N):
pkill -f "user-data-dir=$HOME/.config/Claude-$N"
```

For Profile 1 (no user-data-dir flag):
```bash
# Match Electron mains running app.asar, exclude --type= (helpers)
# and exclude --user-data-dir= (Profile 2/3/4)
pgrep -af 'electron.*app\.asar' \
  | grep -v -- '--type=' \
  | grep -v -- '--user-data-dir=' \
  | head -1
```

---

## AP09 — Skip the `wait_for_cleanup()` race protection

**Why it seems good**: It adds 5s delay. Why bother — let the user retry if it fails.

**What broke**: User's actual usage pattern is "click X, immediately click dock icon". Without wait_for_cleanup, the new launch races with old electron's exit, hits stale SingletonLock, gets confused. Black window or no window at all.

**Right pattern**: Keep wait_for_cleanup. The 5s wait is a worst case — typical wait is <1s (most cases find clean state immediately and break out of loop).

**Reference error**: E10.

---

## AP10 — Use `gdbus call ... org.gnome.Shell.Eval` to manipulate windows

**Why it seems good**: Direct Mutter API access for window operations.

**What broke**: GNOME 47+ disabled Eval method by default for security. Returns `(false, '')`. No way to query/manipulate Mutter window state from outside.

**Right pattern**: Use X11 tools (xdotool, wmctrl, xprop) which work on XWayland-backed windows. Don't rely on GNOME-specific APIs.

---

## AP11 — Wrap launches in fancy retry loops

**Why it seems good**: "If launch fails, retry automatically."

**What broke**: Each "launch attempt" spawns a full Electron process tree (electron + zygotes + helpers + cowork connect). 3 retries × 4 profiles = 12 Electron stacks competing for RAM/GPU. System OOM'd.

**Right pattern**: Single launch, fail loud, document failure modes. User retries manually. Backpressure (wait_for_cleanup) prevents most legitimate retries.

---

## AP12 — Periodic `pkill claude-desktop` cleanup cron

**Why it seems good**: "Just clean up stale stuff every hour."

**What broke**: Killed user's ACTIVE Claude session in middle of a chat. User lost message context.

**Right pattern**: Cleanup is event-triggered (X-button click), not scheduled. Manual `claude-launch sweep` available for panic recovery.

---

## AP13 — Trust `systemd --user` for everything (services, timers, scopes)

**Why it seems good**: systemd is THE Linux init system. Everything should be a unit.

**What broke** (multiple):
1. `systemd-run --scope` for Electron → AP01 (black screen).
2. systemd timer for periodic sweep → AP12 (killed active sessions).
3. systemd user service for cowork → AP06 (no IPC handshake).

**Right pattern**: Use systemd for what it's good at (long-running daemons with simple lifecycle). For complex GUI apps with race-sensitive init, use simple shell + watchers.

---

## AP14 — Hardcode paths like `/home/ayaz`

**Why it seems good**: We KNOW where files are. Just write the path.

**What broke**: Repo is shared on GitHub. Other users have different `$HOME`. Script broken for them.

**Right pattern**: Always use `$HOME` or `${XDG_*_HOME}` variables. Sanitize before push.
```bash
# Bad:
pid_file=/home/ayaz/.cache/claude-launch/profile-2.pid

# Good:
pid_file="${XDG_CACHE_HOME:-$HOME/.cache}/claude-launch/profile-2.pid"
```

---

## AP15 — Use Wayland-native instead of XWayland (--ozone-platform=wayland)

**Why it seems good**: Wayland is the future. Drop X11 baggage.

**What broke**: 
1. `xdotool` doesn't work (X11-only). Lose WM_CLASS fix capability.
2. `wmctrl` doesn't work. Lose window watcher capability.
3. Various Electron rendering quirks under native Wayland on NVIDIA.

**Right pattern**: Use XWayland (`--ozone-platform=x11`, which is Claude Desktop's default). All our X11 tools work. Wayland session integration is fine through XWayland bridge.

---

## AP16 — Test glitch by polling 1 frame at a time (no wait)

**Why it seems good**: High-resolution glitch measurement.

**What broke**: Test polling at 50ms is itself CPU-heavy. Skews the measurement (system load increases, glitch appears longer).

**Right pattern**: Poll at 100-200ms for measurement. Measurement overhead negligible at this resolution.

---

## AP17 — Assume `pgrep -f` matches everything we expect

**Why it seems good**: pgrep is the "right" tool for finding processes by name.

**What broke**: pgrep matches by entire cmdline (with -f). Patterns can match unintended processes (e.g., your own grep, your own monitor scripts). Always:
```bash
# Add specific exclusions
pgrep -f "$PATTERN" | xargs -I{} sh -c '
  cmdline=$(cat /proc/{}/cmdline | tr "\0" " ")
  case "$cmdline" in
    *--type=*) ;;            # exclude Chromium helpers
    *grep*) ;;               # exclude your own grep
    *) echo {} ;;
  esac
'
```

---

## AP18 — Edit script with `sed -i` for "small fixes"

**Why it seems good**: One-liner.

**What broke**: sed -i doesn't validate bash syntax. We've seen sed change `${VAR}` to `$VAR` (no harm) but also change `$()` boundaries (catastrophic). After sed, ALWAYS:
```bash
bash -n bin/claude-launch  # syntax check
```

Or better, use a proper editor that does live syntax highlighting.

---

## AP19 — Trust user's first description of bug

**Why it seems good**: "User reported black screen. Must be GPU."

**What broke**: User said "black screen" 3 times. We chased GPU bugs (`--disable-gpu`, NVIDIA env). Real cause: scope wrapping, then stdout redirection. Always:
1. Reproduce
2. Diff working state vs broken state
3. Bisect to find which change broke it

Don't trust first hypothesis. Verify with `bash -x` or strace.

---

## AP20 — Write your own monitor when system has structured logs

**Why it seems good**: Custom monitor sees exactly what we want.

**What broke**: We wrote a 60-line bash monitor when:
- `~/.cache/claude-launch.log` already had launcher events
- `~/.config/Claude-N/logs/main.log` already had backend events
- `journalctl --user` had systemd unit events

Our monitor was redundant. Custom monitors have bugs (we hit `${$()##}` syntax error in one).

**Right pattern**: Use existing logs. Add log entries to your script if specific events aren't captured. `tail -F` + grep is more reliable than custom polling.
