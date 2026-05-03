# The Journey — How We Got Here

Chronological story of building this project. ~30 iterations across multiple sessions. This file exists because the WHY of decisions matters as much as the WHAT — future maintainers need to know which dead-ends we've walked.

---

## Phase 0 — Original Pain Point

User's actual workflow: 4 monitors, wants 4 different "contexts" in Claude Desktop simultaneously.
- Monitor 1: General chat
- Monitor 2: Code project A
- Monitor 3: Code project B
- Monitor 4: Research

Claude Desktop is single-instance. Click the icon, focus existing window. No way to have 4 windows.

**Initial naive idea**: Just `claude-desktop --user-data-dir=Claude-2`. Works! Now Profile 2 has its own login.

**First problem discovered**: Each profile has its own chat history. User wants ONE chat history visible across all 4 profiles. Otherwise it's 4 separate apps with no unified context.

---

## Phase 1 — Filesystem Surgery (Symlinks)

**Hypothesis**: Symlink Profile 2's chat-history dir to Profile 1's. Both profiles see same data.

**Implemented**:
```bash
ln -sfn ~/.config/Claude/local-agent-mode-sessions ~/.config/Claude-2/local-agent-mode-sessions
ln -sfn ~/.config/Claude/claude-code-sessions ~/.config/Claude-2/claude-code-sessions
# ... etc
```

**Result**: Worked! Chat history shared. Login still independent.

**What we learned (E01 cascade quit)**: We also symlinked `claude-code-vm/`. Then closing Profile 4 caused all 4 profiles to crash. Cowork VM uses lock files in this dir; one profile releasing locks made other profiles' VMs think "everyone quit" → they quit.

**Fix**: Make `claude-code-vm/` per-profile real dir. Document carefully.

This pattern was codified into `ensure_profile()` function in claude-launch.

---

## Phase 2 — Token Sync Storm

**Problem**: Each profile's first-time login required separate OAuth dance. Annoying.

**Tried**: Copy `oauth:tokenCache` from Profile 1's config.json to all profiles' config.json.

**Worked** but introduced a NEW problem: Every launch re-overwrote target's token. If user logged into Profile 2 with a DIFFERENT account, the next launch wiped it back to Profile 1's token. User couldn't have multi-account.

**Fix (sync_token v2)**: Only copy if target has NO token. Preserves cross-account scenarios.

```bash
sync_token() {
  target_tc=$(jq -r '."oauth:tokenCache" // empty' "$target")
  [ -n "$target_tc" ] && return 0   # Don't overwrite existing
  main_tc=$(jq -r '."oauth:tokenCache" // empty' "$MAIN_DIR/config.json")
  [ -z "$main_tc" ] && return 0     # Source has nothing
  jq --arg tc "$main_tc" '."oauth:tokenCache" = $tc' "$target" > "$tmp"
  mv "$tmp" "$target"
}
```

---

## Phase 3 — WM_CLASS Hell (Taskbar Icon Mismatch)

**Problem**: All profiles' taskbar icons looked the same (default Claude orange). Couldn't visually distinguish which window was Profile 2 vs Profile 3.

**First attempt**: Generate hue-shifted PNG icons via ImageMagick. Set `Icon=claude-desktop-2` in `.desktop`. No effect.

**Investigation**: GNOME shell matches windows to .desktop entries via `_NET_WM_CLASS` X11 atom. Default Electron sets WM_CLASS to "Claude" (capital). All profiles have same WM_CLASS → all match same .desktop.

**Second attempt**: Add `--class=Claude-2` to Electron command line. Should set WM_CLASS to "Claude-2".

**Investigation 2**: For utility processes (zygote, network), `--class` works. For the MAIN BrowserWindow (the one user sees), Electron sets WM_CLASS to "claude" (lowercase, hardcoded).

**Real fix**: Post-launch xdotool override.
```bash
xdotool set_window --class "Claude-2" --classname "Claude-2" $WID
```

**Refinement**: This causes 5-6 second "glitch" where icon is orange before becoming color. Three iterations of fix:
1. **v1 (1-shot at +5s)**: 5s sleep + 1 set_window call. Glitch = 6s.
2. **v2 (90s polling)**: 5s polling for 90s, catch new windows. Glitch = 5s, redundant CPU.
3. **v3 (B+A hybrid)**: xdotool `--sync` event-based + 30s backup polling. Glitch = 2s, minimal CPU.

The B+A naming: B = "block on event" (xdotool --sync), A = "active scan" (periodic polling).

---

## Phase 4 — The MCP Leak Disaster

**Symptom report**: User checked process list after some hours. 60+ orphan Python processes from MCP servers. RAM usage skyhigh.

**Investigation**: Read app.asar source via npx asar.

Found: `StdioClientTransport.close()` only kills DIRECT child (the npm/npx/bash wrapper). The actual MCP server (node/python) is the wrapper's child. When wrapper dies, MCP server gets reparented to PID 1. systemd user manager re-attributes orphans to currently-running Electron's cgroup. Looks like new-electron's child but is actually a zombie from a previous session.

**Verified upstream issue**: [claude-code#1935](https://github.com/anthropics/claude-code/issues/1935) and 4 related issues. Fix proposed but not merged in Anthropic side.

**Our workaround design discussion**:
- Option A: systemd-run --scope wrapping. Killing scope = kill cgroup atomically.
- Option B: Watcher + pkill pattern.
- Option C: Modify app.asar.

**Tried Option A**: systemd-run scope. → BLACK SCREEN (E03). Massive failure.
**Discarded Option C**: app.asar editing too fragile.
**Implemented Option B**: Worked.

---

## Phase 5 — The Black Screen Saga

We fought black screen for 4+ iterations.

**Iteration 1**: After adding systemd-run scope wrapping, ALL profiles showed black screen. Title bar visible, body black.
- Hypothesis: GPU/Wayland conflict. Tried `--disable-gpu` → still black.
- Hypothesis: NVIDIA driver env missing in scope. Verified `__GLX_VENDOR_LIBRARY_NAME` etc. → present.
- Hypothesis: scope environment subtly different. Could not pinpoint exact diff.
- Resolution: ABANDON systemd-run wrapping. Roll back to direct nohup. → render works.

**Iteration 2**: Even after rolling back, some launches showed black screen.
- Hypothesis: Race condition on rapid relaunch.
- Verified by stress test: 1s gap between kill and relaunch caused black; 30s gap was fine.
- Cause: Electron starting before old electron's locks released.
- Fix: `wait_for_cleanup()` adds backpressure. Glitch resolved.

**Iteration 3**: Still some launches black, even after wait_for_cleanup.
- Hypothesis: stdout being a regular file (LAUNCH_LOG redirection).
- Fix: Change `>>$LAUNCH_LOG 2>&1` to `>/dev/null 2>&1`. → render works.
- Lesson: Don't redirect Electron stdout to a file.

**Iteration 4**: Profile 2 showed black, Profile 1 worked. Same launcher.
- Investigation: Profile 1 .desktop calls `/usr/bin/claude-desktop` directly (no wrapper). Profile 2 .desktop calls `claude-launch 2`.
- Hypothesis: Wrapper environment differs from direct.
- Attempt: env diff between PID 30478 (P1) and PID 191848 (P2). Almost identical.
- Resolved: Was actually iteration 3's stdout redirection issue affecting some launches but not others. Fully fixed in iteration 3.

**Iteration 5 (last)**: User reports "P1 and P4 render OK, P2 and P3 don't." But both P2 and P3 had been launched with the FIXED launcher (post stdout fix).
- Reason: Background watcher process for P2 and P3 was an OLD watcher (pre-fix). They had spawned the old version's electron. When we tested P4 with new launcher, it worked. P2/P3 needed kill+relaunch to pick up the fix.
- Fix: Document migration in CLAUDE.md. New launcher takes effect on next launch, not running instances.

---

## Phase 6 — Race Storm Stress Test

User stress-tested with 8 rapid kill/relaunch of Profile 2. Caused:
1. 8+ parallel watchers spawned (multi-watcher proliferation, E06)
2. Log spam: "cleanup complete" 10+ times for same kill
3. CPU load 60+ during storm
4. WM_CLASS glitch on some windows (E05 variant)

**Investigation**: Each `claude-launch 2` spawned `( supervise_profile 2 $pid ) &` background subshell. Old subshells weren't terminated. After 8 launches, 8 watchers, all triggering cleanup on next X.

**Initial (failed) fix attempt**: pgrep cmdline match.
```bash
pgrep -af "supervise_profile $profile " | xargs kill
```
Failed because subshells inherit parent script's cmdline (E12).

**Working fix**: PID file 2-line format.
```
~/.cache/claude-launch/profile-N.pid:
  Line 1: <electron_pid>
  Line 2: <watcher_pid>
```

`kill_old_watchers()` reads line 2, kills before spawning new. Singleton invariant.

---

## Phase 7 — Window-Closed-But-Process-Alive Bug

User reports: clicks X. Window disappears. Clicks dock icon. Sees "Profile X already running" message but no window opens.

**Investigation**:
```
ps -ef | grep electron  # Process IS alive
wmctrl -lx | grep claude  # No window
xdotool search --pid $PID  # 2 windows (utility, hidden)
```

Electron is in background mode. X-button destroyed window but process stays alive (Claude Desktop has macOS-style background behavior, Linux platform inherits this even though it's atypical for Linux).

**Initial wrong fix**: xdotool windowmap+activate. The "windows" xdotool found were Electron's UTILITY windows (not user-visible). Mapping them did nothing.

**Right fix**: Use Electron's built-in single-instance mechanism.
```bash
raise_existing_profile() {
  # Spawn second Electron with same --user-data-dir
  # Electron's requestSingleInstanceLock() detects primary
  # Secondary exits immediately, primary gets second-instance event
  # Primary calls createWindow() → fresh window
  nohup ... claude-desktop --user-data-dir=$dir --class=Claude-$N \
    >/dev/null 2>&1 &
  disown
}
```

This is identical to Profile 1's behavior with default Claude (clicking dock icon when running opens new window via this mechanism).

**Bonus fix needed**: New window has WM_CLASS="claude" again. Re-trigger fix_wm_class_persistent.

---

## Phase 8 — The "Cleanup Not Triggering" Problem

After Phase 7 fix, X-button cleanup STILL didn't trigger reliably. Process stayed alive, MCP children leaked.

**Investigation**: Watcher was waiting for `kill -0 $electron_pid` to fail. Electron stays alive after X (background mode). Watcher never fires.

**Root cause**: We were watching the WRONG signal. Process death is not the right trigger.

**Fix**: Watch for WINDOW destruction.
```bash
supervise_profile() {
  while true; do
    has_window=$(wmctrl -lx | grep -c "$WM_CLASS")
    if [ "$has_window" = 0 ]; then
      missing_count=$((missing_count + 1))
      [ $missing_count -ge 3 ] && break  # 3 scans = 6s threshold
    else
      missing_count=0
    fi
    sleep 2
  done
  cleanup
}
```

3-scan threshold prevents false positives from minimize/raise events.

---

## Phase 9 — System RAM Exhaustion

User had Discord + Brave (with many tabs) + 4 Claude profiles. Tried to relaunch Profile 4. BLACK SCREEN.

Initial assumption: regression of E03/E04. But everything was already fixed.

**Investigation**: `free -h`
```
Mem: 12 GB used / 15 GB total / 506 MB free
Swap: 19 GB used / 106 GB
```

System RAM exhausted. Electron renderer can't allocate memory for V8/WebGL. Returns null pointer, content render aborts.

**Diagnosis confirmed**: Closed Discord. Reopened Profile 4. Render worked.

**Lesson**: Black screen has TWO main causes:
1. Environment corruption (E03/E04) — fixed in script.
2. RAM exhaustion — system-wide, not script's fault.

Documented in CLAUDE.md troubleshooting.

---

## Phase 10 — The "Why Two Orange Icons?" Investigation

User reports: 2 of the 4 taskbar icons are orange (Profile 1 color), should be color-distinct.

**Investigation**:
```bash
wmctrl -lx | grep -i claude
# 0x... claude.Claude         (Profile 1 — correct, orange)
# 0x... Claude-2.Claude-2     (Profile 2 — correct, lavender)
# 0x... claude.Claude         ← WRONG! This one is Profile 4 with wrong WM_CLASS
# 0x... Claude-3.Claude-3     (Profile 3 — correct, green)
```

Profile 4 had a window with WM_CLASS="claude" (matching Profile 1's icon).

**Investigation deeper**: This was a window opened by `raise_existing_profile()` (Phase 7 fix). Secondary Electron created the window, primary opened it via second-instance event, but the new window came with WM_CLASS="claude" again (Electron bug).

`fix_wm_class_persistent()` had ALREADY completed its 30s scan from initial launch. It wasn't re-triggered for the new window.

**Fix**: Re-trigger fix_wm_class_persistent inside raise_existing_profile.
```bash
raise_existing_profile() {
  # ... secondary launch ...
  ( fix_wm_class_persistent "$profile" "$dir" ) >/dev/null 2>&1 &
  disown
}
```

---

## Phase 11 — The "Glitch Too Long" Optimization

After everything worked, user noted: "When I open a profile, taskbar shows orange for ~6 seconds, then becomes color. Annoying."

**Analysis** (decomposed glitch into phases):
- T+0ms: window appears with WM_CLASS="claude"
- T+0ms - T+5000ms: fix_wm_class_persistent's PHASE 1 polling for electron PID (1s polling, finds in ~1s)
- T+1000ms - T+6000ms: PHASE 2 polling (5s sleep per iter)
- T+6000ms: First xdotool set_window call, glitch ends

So glitch = ~6s, dominated by Phase 2's 5s sleep.

**Hybrid B+A design proposed**:
- B = "block on event": `xdotool search --sync --pid $PID`. X server tells us when window appears (~100-300ms).
- A = "active scan": Keep 30s × 5s polling as backup for raise/dialog windows.

**Resource analysis** (proven before implementation):
- B: xdotool sleeps in kernel waiting for X event. ~2.3 MB RAM, ~0% CPU.
- A: 30s × 5s = 6 scans. Each scan ~50ms CPU. Total ~300ms.
- Combined: WAY less than old polling-only approach.

**Implemented**: glitch reduced from ~6s to ~2s.

Phase 1 polling tightened from 1s to 0.2s (5× faster electron detection on cold start).

---

## Phase 12 — GitHub Repository

User: "Push this to GitHub. Name it `multi session claude desktop`."

**Sanity check**: "session" is wrong term. We have multi-INSTANCE (separate Electron processes), not multi-session (Claude Desktop already supports multiple chat sessions in one instance).

**Discussion**: User agreed. Renamed to `multi-instance-claude-desktop`.

**Repo structure decision**:
- `bin/claude-launch` — the script
- `desktop-entries/` — .desktop templates
- `docs/` — documentation
- `LICENSE`, `README.md`, `CLAUDE.md`

**Credential safety**: 
- Sanitized `/home/ayaz` → `$HOME` in script and .desktop entries
- grep'd for token/secret/password — only OAuth field NAMES (not values) found in script
- grep'd for email/username — none

**Push**: `gh repo create CyPack/multi-instance-claude-desktop --public --source=. --push`. Done.

---

## Phase 13 — Documentation Pack (this set of files)

Realized: Future maintainers (human or AI) will face the same dead-ends without this knowledge. Created:
- `CLAUDE.md` — orientation + critical constraints
- `docs/ARCHITECTURE.md` — technical deep dive
- `docs/lessons/errors.md` — every error we hit + fix
- `docs/lessons/golden-paths.md` — proven workflows
- `docs/lessons/anti-patterns.md` — what NOT to try
- `docs/journey.md` — this file (the story)

The pattern: error catalog + golden paths + anti-patterns + journey is from observability practice (post-mortem culture). Heavily inspired by SRE workbook patterns.

---

## Numbers (for the curious)

- **Total iterations**: ~30 across multiple sessions
- **Lines of script**: 5759 (v1) → 26037 (v3)
- **Errors cataloged**: 17
- **Anti-patterns documented**: 20
- **Glitch reduction**: 6000ms → 2000ms (3x improvement, with --sync potential to ~300ms)
- **MCP leak fix RAM saved**: ~440 MB (measured on user's system after first cleanup test)
- **Bash crashes during dev**: 5+ (set -euo pipefail caught silent failures)
- **Black screens debugged**: 4 different root causes
- **GitHub stars (as of 2026)**: ?
