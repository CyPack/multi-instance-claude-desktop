# Golden Paths — Proven Workflows

Patterns that work reliably. Copy-paste safe. Each entry: scenario → exact steps → expected outcome → why it works.

---

## GP01 — Setup from scratch (clean install, 0 profiles)

**Scenario**: Fresh system, only default Profile 1 exists.

**Steps**:
```bash
# 1. Install
git clone https://github.com/CyPack/multi-instance-claude-desktop.git
cd multi-instance-claude-desktop
install -Dm755 bin/claude-launch ~/.local/bin/claude-launch
install -Dm644 desktop-entries/claude-desktop-2.desktop ~/.local/share/applications/
install -Dm644 desktop-entries/claude-desktop-3.desktop ~/.local/share/applications/
install -Dm644 desktop-entries/claude-desktop-4.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications/

# 2. (Optional) Generate colored icons
# See docs/icons.md

# 3. First-time launch each profile
claude-launch 2  # Creates ~/.config/Claude-2/, copies config from master
claude-launch 3  # Same for Profile 3
claude-launch 4  # Same for Profile 4

# 4. Login each profile separately (one-time)
# Click "Login" in each profile's window, complete OAuth flow
```

**Expected outcome**:
- 4 Claude Desktop windows visible, each with own colored icon
- Each profile has independent login
- Chat history is shared across all 4 (via symlinks)
- `claude-launch status` shows 4 instances

**Why it works**: `ensure_profile()` handles first-time setup atomically — creates dir, sets symlinks, copies config templates, generates UUID. No manual filesystem work needed.

---

## GP02 — Daily usage (open all 4 in morning)

```bash
# Option A: dock icons (one click each)
# Option B: terminal one-shot
claude-launch all  # Launches 1+2+3+4 sequentially with 1s gap
```

**Why the 1s gap matters**: Cowork-vm-service first-time bind needs to settle before next profile tries to connect. Without the gap, cowork bind contention can cause launch failures.

---

## GP03 — Close everything cleanly (end of day)

```bash
claude-launch kill-all
```

**Order matters**: kill-all closes 2/3/4/5/1 in that order. Profile 1 last because it owns cowork-vm-service. If you kill Profile 1 first, cowork dies; remaining profiles see "Cowork unavailable" briefly.

**Alternative — leave them running**: Closing all profiles every day is unnecessary. Claude Desktop is designed for long-running sessions. Sleeping/locking your system is fine.

---

## GP04 — Single profile open-close cycle (X-button → reopen)

```bash
# 1. Click X on Profile 4 window
# 2. Wait 6-10 seconds (watcher cleanup needs ~6s)
# 3. Click dock icon for Profile 4
```

**Expected outcome**:
- Cleanup runs: MCP children gone, scope freed
- Reopen: fresh window with WM_CLASS=Claude-4, glitch ~2s

**Why the wait matters**: If you reopen within 5s, `wait_for_cleanup()` blocks the new launch until cleanup is done. So you can reopen instantly — script handles the race for you.

---

## GP05 — Migrating an existing profile to new launcher

If you have Profile 2/3/4 dirs created by an earlier method (manual), the new launcher will work with them BUT some safety guarantees aren't activated until ensure_profile() runs.

```bash
# 1. Close the profile you want to migrate
claude-launch kill 2

# 2. Trigger ensure_profile by relaunching
claude-launch 2
```

`ensure_profile()` will:
- Detect existing dir, skip creation
- Run drift repair: convert `claude-code-vm/` from symlink to real dir if needed
- Set dir permissions to 700
- Verify all expected symlinks present, recreate if missing

---

## GP06 — Adding a 5th (or 6th, Nth) profile

The script supports profile numbers 2-5 out of the box. Just create a `.desktop` entry:

```bash
# 1. Create .desktop (template based on claude-desktop-4.desktop)
cat > ~/.local/share/applications/claude-desktop-5.desktop <<EOF
[Desktop Entry]
Name=Claude (Profile 5)
Exec=/home/ayaz/.local/bin/claude-launch 5 %u
Icon=claude-desktop-5
Type=Application
Terminal=false
Categories=Office;Utility;
MimeType=x-scheme-handler/claude;
StartupWMClass=Claude-5
EOF

# 2. Generate icon (see docs/icons.md)
# 3. update-desktop-database ~/.local/share/applications/
# 4. claude-launch 5  # First-time launch creates Claude-5 dir
```

To support N=6+, edit `bin/claude-launch`'s `case` statement:
```bash
case "${1:-}" in
  2|3|4|5|6|7|8) launch_profile "$1" ;;
  ...
```

---

## GP07 — Sharing GPU with other AI workloads (e.g., embedding models)

Edit `bin/claude-launch`, uncomment the GPU-related line in CLAUDE_EXTRA_FLAGS:

```bash
# === GPU TAMAMEN DISABLE ===
CLAUDE_EXTRA_FLAGS+=(--disable-gpu --disable-software-rasterizer)
```

All Claude Desktop instances now run on CPU. GPU fully available for embedding models.

**Or env-based one-shot**:
```bash
CLAUDE_DISABLE_GPU=1 claude-launch 4   # Just this launch uses no GPU
```

---

## GP08 — Recovering from broken state (everything weird)

If the system is in an unknown state (windows missing, processes leaked, status looks wrong):

```bash
# 1. Kill everything cleanly
claude-launch kill-all

# 2. Wait 10s for cleanup
sleep 10

# 3. Manual sweep of any orphans
claude-launch sweep

# 4. Verify clean state
claude-launch status
# Should show: (hicbiri acik degil)

# 5. Verify cgroup state
systemctl --user list-units 'app-electron-*.scope' --no-pager
# Old scopes: systemctl --user reset-failed 'app-electron-*.scope'

# 6. Relaunch fresh
claude-launch all
```

---

## GP09 — Debugging a specific profile

```bash
# 1. Check process tree
pgrep -af "user-data-dir=Claude-3"

# 2. Check window
wmctrl -lx | grep Claude-3

# 3. Check backend log (real-time)
tail -F ~/.config/Claude-3/logs/main.log

# 4. Check launcher events
tail -F ~/.cache/claude-launch.log | grep "Claude-3\|Profile 3"

# 5. Check PID file (singleton invariant)
cat ~/.cache/claude-launch/profile-3.pid
# Line 1: electron PID, Line 2: watcher PID

# 6. Check watcher health
WATCHER=$(sed -n '2p' ~/.cache/claude-launch/profile-3.pid)
kill -0 $WATCHER && echo "watcher alive" || echo "watcher dead"
```

---

## GP10 — Verifying credential safety before pushing changes

Before pushing any changes to git or sharing config files:

```bash
# Email/username scan
grep -rE "@gmail|@hotmail|<your-username>" .

# Token / secret scan
grep -rE "token|secret|password|api.?key|bearer" \
  --include='*.sh' --include='*.json' --include='*.md' .

# Path leakage scan (your home dir)
grep -r "/home/$USER" .
```

All should return empty (or only commented references). If anything leaks, sanitize before commit.

---

## GP11 — Test scenarios (sanity check after script changes)

After modifying `bin/claude-launch`, run these tests:

### T1: Syntax
```bash
bash -n bin/claude-launch && echo OK
```

### T2: Help
```bash
claude-launch --help    # Shows usage
```

### T3: Status (Profile 1 detection)
With all 4 profiles open, `claude-launch status` must show Profile 1 (regression of E07).

### T4: Fresh launch (rendering)
```bash
claude-launch kill 4
sleep 8
claude-launch 4
# Wait 5s, check window content (not black)
```

### T5: X-button cleanup
1. Note Profile 4 PID
2. Click X
3. Wait 15s
4. `pgrep -f "user-data-dir=Claude-4"` should be empty

### T6: Race tolerance
1. Click X on Profile 4
2. Within 1 second click dock icon
3. Within 12s, content should appear (wait_for_cleanup handles race)

### T7: Watcher singleton
After 3 rapid relaunches:
```bash
ps -ef | grep "claude-launch 4" | grep -v grep | wc -l
# Should be 1-2 (1 supervisor + 1 wm-class fixer = 2 max)
```

### T8: Cowork preservation
Close Profile 4, check Profile 1's cowork still alive:
```bash
COWORK_BEFORE=$(pgrep -f cowork-vm-service.js)
claude-launch kill 4
sleep 10
COWORK_AFTER=$(pgrep -f cowork-vm-service.js)
[ "$COWORK_BEFORE" = "$COWORK_AFTER" ] && echo "✓ Cowork unchanged"
```

---

## GP12 — Memory + log cleanup (periodic maintenance)

```bash
# Trim launcher log to last 1000 lines
tail -n 1000 ~/.cache/claude-launch.log > /tmp/launch.log.new
mv /tmp/launch.log.new ~/.cache/claude-launch.log

# Clean up stale PID files (electron + watcher both dead)
for f in ~/.cache/claude-launch/profile-*.pid; do
  e=$(sed -n '1p' "$f"); w=$(sed -n '2p' "$f")
  if ! kill -0 $e 2>/dev/null && ! kill -0 $w 2>/dev/null; then
    rm "$f"
  fi
done
```
