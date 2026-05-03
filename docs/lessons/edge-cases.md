# Edge Cases — Weird Scenarios We've Hit

Patterns that don't fit "error" or "anti-pattern" but matter. Each entry: scenario → behavior → handling.

---

## EC01 — User runs Claude Desktop while sweeping

**Scenario**: User manually runs `claude-launch sweep` while a Claude Desktop session is actively spawning new MCP servers (e.g., user just opened a new Code mode session that's spawning supabase-mcp, t4f-api-client, etc.).

**Behavior**: Sweep targets PPID=1 orphans by claude-related pattern. Newly-spawning MCPs briefly have PPID != 1 (their direct parent — npm exec wrapper — is alive). They survive sweep.

**Handling**: Acceptable. Sweep is for cleanup of LEAKED orphans, not active sessions. Active MCPs untouched.

**If issue**: Don't run sweep during active session. Or wait 30s after closing UI before sweeping.

---

## EC02 — Profile dir manually deleted while running

**Scenario**: User does `rm -rf ~/.config/Claude-3` while Profile 3 is running.

**Behavior**: Electron continues running with deleted dir (file descriptors remain valid in /proc). When Electron tries to write (config save, cache update), gets EBADF or files appear in nonexistent path. Eventually crashes.

**Handling**: Don't do this. Use `claude-launch kill 3` first. If you've already done it: kill -KILL the process, restart.

---

## EC03 — Cowork-vm-service crashes mid-session

**Scenario**: Cowork crashes (segfault, OOM kill, etc.). Sandbox dies.

**Behavior**: 
- All profiles' "Cowork" tab shows error
- Existing chats continue (cowork is only for code execution, not chat)
- Next profile launch (or claude-launch sweep) detects no cowork → upstream launcher-common.sh respawns it

**Handling**: Open any new profile or close+reopen one. Cowork respawns automatically.

---

## EC04 — User logs OUT of a profile

**Scenario**: User clicks "Logout" in Profile 3's UI.

**Behavior**: Profile 3's `oauth:tokenCache` cleared from `config.json`. Profile 3 shows login screen on next interaction. OTHER profiles unaffected (they have their own tokens).

**Handling**: Expected behavior. User logs back in via OAuth.

**Gotcha**: On next `claude-launch 3`, our `sync_token()` sees Profile 3 has no token + Profile 1 (master) has token → COPIES Profile 1's token to Profile 3. Now Profile 3 logged in as Profile 1's account.

If user wanted Profile 3 to stay logged out or use a different account: they need to login Profile 3 BEFORE `sync_token()` runs. Workaround: comment out `sync_token "$profile"` line in `launch_profile()`.

---

## EC05 — User switches DNF/RPM and updates Claude Desktop

**Scenario**: `sudo dnf upgrade claude-desktop` to a new version.

**Behavior**:
- `/usr/lib/claude-desktop/` updated (new app.asar, new launcher-common.sh)
- `/usr/share/applications/claude-desktop.desktop` updated
- BUT `~/.local/share/applications/claude-desktop*.desktop` (user override) NOT touched
- BUT `~/.local/bin/claude-launch` NOT touched

Our user-side scripts and entries survive upgrades cleanly.

**Caveat**: If new Claude Desktop version changes Electron's behavior (e.g., implements --class properly, or fixes MCP leak natively), our workarounds may become unnecessary. Watch upstream for changes.

---

## EC06 — User has only 1 profile (just Profile 1)

**Scenario**: User installs claude-launch but never creates Profile 2/3/4.

**Behavior**: 
- `claude-launch` (no args) launches Profile 1 directly
- Profile 1 .desktop is unchanged (still calls `/usr/bin/claude-desktop` directly)
- claude-launch's status shows only Profile 1 if running

This is fine. Project supports 1-N profiles.

---

## EC07 — Profile dir exists but has no SingletonLock (clean state)

**Scenario**: After kill-all, Profile dir exists but no SingletonLock symlink.

**Behavior**: `profile_already_running()` returns 1 (lock file doesn't exist). `launch_profile()` proceeds with normal launch.

**Handling**: Normal. Expected after clean shutdown.

---

## EC08 — Two users on same machine (multi-user)

**Scenario**: User A and User B both want multi-instance setup.

**Behavior**:
- Each user has their own `~/.config/Claude*/`, `~/.local/bin/claude-launch`, `~/.cache/claude-launch/`
- Cowork-vm-service socket: `/run/user/$UID/cowork-vm-service.sock` is per-user (XDG_RUNTIME_DIR)
- No conflict.

**Handling**: Each user installs claude-launch independently in their own home.

---

## EC09 — User is on PURE Wayland (no XWayland)

**Scenario**: GNOME Wayland session WITHOUT XWayland. Or user explicitly sets `CLAUDE_USE_WAYLAND=1`.

**Behavior**:
- xdotool DOESN'T WORK (X11-only)
- wmctrl DOESN'T WORK
- WM_CLASS fix breaks (window appears with "claude", stays as "claude")
- Window watcher breaks (can't detect windows)
- supervise_profile loop runs forever, never triggers cleanup

**Handling**: 
- Workaround: don't use pure Wayland mode. Stick with XWayland.
- Long-term fix: use Wayland-native tools (wlr-randr, etc.) or DBus interfaces.
- Check for support: `[ -n "$DISPLAY" ] && have_xtools=1` — fallback gracefully.

---

## EC10 — Stress test: 50 profiles requested

**Scenario**: User creates Claude-5 through Claude-50 .desktop entries, tries `claude-launch all`.

**Behavior**:
- Each Electron uses ~300 MB RAM. 50 = 15 GB RAM. System OOM.
- 50 cgroup scopes from GNOME. systemd handles fine but slow.
- 50 cowork connections to single socket. Socket buffer overflow possible.

**Handling**: Don't. Project is designed for 4-5 profiles practical use. For more, need different architecture (centralized chat history server, virtualized profile state).

---

## EC11 — `claude-launch status` during launch_profile execution

**Scenario**: Two terminals — terminal 1 runs `claude-launch 4`, terminal 2 immediately runs `claude-launch status`.

**Behavior**: Status reads PID file. PID file may not yet exist if terminal 1 hasn't reached `printf > pid_file`. Status shows Profile 4 as not running, but actually starting.

**Handling**: Acceptable race. Re-run status 1-2s later for accurate result.

---

## EC12 — Disk full on /home

**Scenario**: User runs out of disk space. New profile creation in `ensure_profile()` fails.

**Behavior**: `mkdir` fails, script exits with error (set -e). User sees error message. Profile not launched.

**Handling**: Free disk space. Retry.

---

## EC13 — Permission denied on /home/$USER

**Scenario**: Some weird permissions issue.

**Behavior**: `mkdir -m 700` or `chmod 700` fails. Script exits.

**Handling**: Fix permissions (`sudo chown -R $USER:$USER ~/.config/Claude*`).

---

## EC14 — Network down, no OAuth refresh

**Scenario**: User offline. Token expires.

**Behavior**: Claude Desktop shows "Token expired" or "Network error" in UI. Backend log: `oauth: failed to refresh token`. Doesn't crash.

**Handling**: Reconnect to network. Auto-refreshes.

**Gotcha**: If user's `oauth:tokenCache` got cleared (network was down during refresh attempt), `sync_token()` may copy Profile 1's token. See EC04.

---

## EC15 — System hibernate/resume during active session

**Scenario**: User hibernates mid-Claude session, resumes hours later.

**Behavior**: 
- Electron process suspended during hibernate
- On resume, processes wake up, but cowork socket may be in unknown state (kernel cleared)
- Window contents may be stale until user interacts

**Handling**: Refresh window or close+reopen. Cowork respawns.

---

## EC16 — User uses Sysrq-trigger / OOM killer

**Scenario**: System OOM killer kills Claude Desktop processes (random which one).

**Behavior**:
- Some Electrons die, others survive
- Watchers detect window-gone via wmctrl scan, trigger cleanup
- PID file may be stale (electron died, watcher still alive)
- Status command may show wrong info briefly

**Handling**: Run `claude-launch sweep` then `claude-launch status` to see clean state. Relaunch profiles as needed.

---

## EC17 — User's NVIDIA driver crashes

**Scenario**: NVIDIA module crashes. All GPU apps die.

**Behavior**: 
- All Claude windows go black or freeze
- xdotool may still respond to PIDs that no longer exist
- Watchers may misbehave (detect "window gone" repeatedly)

**Handling**: Recover GPU (logout+login, or `sudo modprobe -r nvidia && sudo modprobe nvidia`). Restart Claude profiles.

---

## EC18 — User has hostname change

**Scenario**: User changes machine hostname. SingletonLock format is `<hostname>-<pid>`. Old hostname embedded in symlinks.

**Behavior**: 
- Old SingletonLock symlinks point to "old-hostname-12345"
- `profile_already_running()` extracts PID, kills checks alive — works regardless of hostname.
- Electron may complain about hostname mismatch in some weird case.

**Handling**: Usually fine. If issues: `rm ~/.config/Claude-N/SingletonLock` and relaunch.

---

## EC19 — Backup/restore from another machine

**Scenario**: User backs up `~/.config/Claude*` from machine A, restores on machine B.

**Behavior**:
- ant-did UUIDs preserved (Anthropic backend treats as same instances)
- Tokens preserved (Claude OAuth bound to account, not machine)
- Profile 2/3/4 symlinks may be ABSOLUTE PATH — `/home/userA/.config/Claude/...`. On machine B with different username, symlinks broken.

**Handling**: Backup script should use rsync with `--no-preserve-symlinks-targets` or fix symlinks post-restore:
```bash
for f in ~/.config/Claude-*/{claude-code,claude-code-sessions,...}; do
  [ -L "$f" ] && ln -sfn ~/.config/Claude/$(basename $f) "$f"
done
```

---

## EC20 — `xdg-mime` mismatch breaks `claude://` links

**Scenario**: User clicks a `claude://...` URL (e.g., from email). Default handler should open in Profile 1, but mistakenly set to Profile 2.

**Behavior**: Profile 2 opens for the URL. User confused.

**Handling**: Reset:
```bash
xdg-mime default claude-desktop.desktop x-scheme-handler/claude
```

This is documented in `~/.claude/projects/-home-ayaz/memory/tools/claude-desktop.md` (project memory, not in repo).
