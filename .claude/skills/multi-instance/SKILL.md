---
name: multi-instance-claude-desktop
description: Run multiple Claude Desktop profiles in parallel on Linux with per-profile login, shared chat history, and X-button MCP cleanup. Includes window-event-driven cleanup, watcher singleton pattern, WM_CLASS hybrid fix, and NVIDIA-aware launch. TRIGGER when user wants to set up parallel Claude Desktop instances, debug multi-profile issues, fix MCP leak, fix black screen on Electron launch, fix taskbar icon colors, or work on the claude-launch script.
maturity: opus
maturity_score: 7
lesson_count: 17
---

# Skill: Multi-Instance Claude Desktop

## When to Use This Skill

Trigger on any of these scenarios:
- User wants 2+ parallel Claude Desktop instances
- User reports MCP server processes leaking after Claude Desktop usage
- User reports black screen on profile launch
- User reports taskbar shows wrong color icon for profile
- User wants chat history shared across multiple profiles
- User wants to script Claude Desktop multi-instance setup on Linux
- Debugging cascade quit (closing one profile closes all)
- Working on `claude-launch` script in `~/.local/bin/`
- Working on `~/.local/share/applications/claude-desktop-*.desktop` files

## Pre-Execution Checklist

BEFORE making any changes:

```
□ Read CLAUDE.md (project context + critical constraints)
□ Read docs/ARCHITECTURE.md (the WHY behind decisions)
□ Read docs/lessons/anti-patterns.md (what NOT to try — we've been there)
□ Check if user's symptom matches an entry in docs/lessons/errors.md
□ If yes, apply documented fix from errors.md
□ If no, plan carefully + add new entry to errors.md after fix
```

## Critical Constraints (HARD RULES)

NEVER do these (each broke the project at some point):

1. **NEVER patch `app.asar`** — root-owned, breaks on `dnf upgrade`
2. **NEVER `systemd-run --user --scope` wrap Electron** — black screen
3. **NEVER redirect Electron stdout to a file** — black screen on Wayland+NVIDIA
4. **NEVER kill cowork-vm-service from non-Profile-1 cleanup** — cascade failure
5. **NEVER assume `--class` flag works on main BrowserWindow** — Electron bug
6. **NEVER assume X-button kills Electron process** — background mode
7. **NEVER touch Profile 1's `.desktop` carelessly** — DNF override + xdg-mime
8. **NEVER `rm -rf` near $HOME** — safety hooks block it

## Architecture Quick Reference

```
~/.config/Claude/                 ← Profile 1 (master, real files)
~/.config/Claude-{2,3,4,5}/       ← Other profiles (symlinks to master + per-profile reals)

Shared symlinks (chat/agent/sessions):
  claude-code → master
  claude-code-sessions → master
  local-agent-mode-sessions → master
  git-worktrees.json → master
  pending-uploads → master
  buddy-tokens.json → master

Per-profile real files (login/cache):
  config.json (oauth token)
  claude_desktop_config.json (MCP config)
  ant-did (UUID)
  bridge-state.json
  claude-code-vm/ (REAL DIR, not symlink — cascade-quit prevention!)
  Cookies, IndexedDB/, Cache/
```

## Critical Mechanisms

### 1. Window-event-driven cleanup
- Watcher polls wmctrl every 2s
- If no window for 3 consecutive scans (6s threshold) → cleanup
- 3-scan threshold prevents false positives from minimize/raise

### 2. Watcher singleton (PID file 2-line format)
```
~/.cache/claude-launch/profile-N.pid:
  Line 1: electron PID
  Line 2: watcher PID
```
New launch reads line 2, kills old watcher BEFORE spawning new.

### 3. WM_CLASS hybrid fix (B+A)
- B: `xdotool search --sync` event-based wait (~300ms)
- A: 30s × 5s backup scan for raise/dialog windows

### 4. raise_existing_profile = secondary electron launch
Spawn 2nd Electron with same --user-data-dir. Electron single-instance lock kicks in: secondary exits, primary opens fresh window. Same as Profile 1's default behavior.

## Workflow: Diagnosing User-Reported Bug

```
1. Reproduce
   - claude-launch status (check current state)
   - tail ~/.cache/claude-launch.log (recent events)
   - tail ~/.config/Claude-N/logs/main.log (backend events)

2. Match to known error
   - Read docs/lessons/errors.md
   - If exact match → apply documented fix
   - If similar but not exact → ask user for more detail (process tree, screenshots)

3. If new bug
   - Bisect: git log claude-launch — what changed recently?
   - bash -x bin/claude-launch <args> 2>&1 | tail -50 (debug script execution)
   - free -h (check if RAM exhausted — common false-cause)
   - nvidia-smi (check GPU contention)

4. Fix
   - Edit bin/claude-launch
   - Test (see workflow below)
   - Document in errors.md
   - Commit with descriptive message
```

## Workflow: Testing Script Changes

After ANY edit to `bin/claude-launch`:

```bash
# T1: Syntax
bash -n bin/claude-launch && echo OK

# T2: Status (Profile 1 detection regression check)
~/.local/bin/claude-launch status
# Must show all running profiles including Profile 1

# T3: Fresh launch render
~/.local/bin/claude-launch kill 4
sleep 8
~/.local/bin/claude-launch 4
# Wait 5s. Window must show CONTENT (not black).

# T4: X-button cleanup
# Click X on Profile 4 window
# After 15s:
pgrep -f "user-data-dir=Claude-4" | wc -l
# Must be 0 (cleanup ran)

# T5: Race tolerance
# Click X
# Within 1s click dock icon
# Window content must appear within 12s

# T6: Watcher singleton
# After 3 rapid relaunches:
ps -ef | grep "claude-launch 4" | grep -v grep | wc -l
# Must be 1-2 (1 supervisor + 1 wm-class fixer)

# T7: Cowork preservation
COWORK=$(pgrep -f cowork-vm-service.js)
~/.local/bin/claude-launch kill 4
sleep 10
[ "$(pgrep -f cowork-vm-service.js)" = "$COWORK" ] && echo "✓ Cowork unchanged"

# T8: WM_CLASS correctness
wmctrl -lx | grep claude
# Each profile must show its WM_CLASS (claude, Claude-2, Claude-3, Claude-4)
# NOT all "claude.Claude"
```

If ANY test fails, DO NOT commit. Roll back, debug.

## Workflow: Adding a New Profile (e.g., Profile 5)

```bash
# 1. Create .desktop entry
cat > ~/.local/share/applications/claude-desktop-5.desktop <<EOF
[Desktop Entry]
Name=Claude (Profile 5)
Exec=$HOME/.local/bin/claude-launch 5 %u
Icon=claude-desktop-5
Type=Application
Terminal=false
Categories=Office;Utility;
MimeType=x-scheme-handler/claude;
StartupWMClass=Claude-5
EOF

# 2. Generate icon (see docs/icons.md for ImageMagick recipe)
for size in 16 24 32 48 64 256; do
  magick "/usr/share/icons/hicolor/${size}x${size}/apps/claude-desktop.png" \
    -modulate 100,100,200 \
    "$HOME/.local/share/icons/hicolor/${size}x${size}/apps/claude-desktop-5.png"
done
gtk-update-icon-cache -t -f ~/.local/share/icons/hicolor

# 3. Update GNOME dock favorites (optional)
# 4. update-desktop-database ~/.local/share/applications/

# 5. First-time launch
claude-launch 5
# Creates ~/.config/Claude-5/, copies templates, generates UUID
```

## Workflow: Recovery From Broken State

If everything is weird (windows missing, processes leaked, status confused):

```bash
# 1. Kill everything cleanly
claude-launch kill-all

# 2. Wait 10s
sleep 10

# 3. Manual orphan sweep
claude-launch sweep

# 4. Verify clean
claude-launch status
# Should show "(hicbiri acik degil)"

# 5. Clean stale systemd scopes (rare)
systemctl --user list-units 'app-electron-*.scope'
# If any stale: systemctl --user reset-failed 'app-electron-*.scope'

# 6. Relaunch
claude-launch all
```

## Decision Tree: Black Screen Diagnosis

```
Black window on launch
├── Is system RAM exhausted? (free -h: available < 1 GB?)
│   └── YES → Close other apps, retry. Not script's fault.
├── Was script recently changed?
│   ├── YES → bisect changes. Common culprits:
│   │   ├── Stdout redirected to file? → Use /dev/null (E04)
│   │   ├── systemd-run --scope wrapping? → Remove (E03)
│   │   └── New env var being set? → Test removal
│   └── NO → check NVIDIA driver state, X server health
└── Is gio launch vs CLI launch glitch? → race condition (E10)
    └── wait_for_cleanup() should handle. Check if it's being called.
```

## After Each Successful Use of This Skill

If you encountered a NEW error → add to `docs/lessons/errors.md`
If you discovered a NEW workflow → add to `docs/lessons/golden-paths.md`
If you tried something that BROKE → add to `docs/lessons/anti-patterns.md`
Increment `lesson_count` in this SKILL.md frontmatter.

## Reference Files

- `bin/claude-launch` — the script (single source of truth)
- `desktop-entries/claude-desktop-{2,3,4}.desktop` — launcher entries
- `docs/ARCHITECTURE.md` — deep dive
- `docs/lessons/errors.md` — error catalog
- `docs/lessons/golden-paths.md` — proven workflows
- `docs/lessons/anti-patterns.md` — what NOT to try
- `docs/journey.md` — chronological project history

## External References

- Upstream: [aaddrick/claude-desktop-debian](https://github.com/aaddrick/claude-desktop-debian)
- MCP leak: [#1935](https://github.com/anthropics/claude-code/issues/1935), [#11778](https://github.com/anthropics/claude-code/issues/11778), [#33947](https://github.com/anthropics/claude-code/issues/33947)
- Repo: https://github.com/CyPack/multi-instance-claude-desktop
