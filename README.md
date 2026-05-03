# Multi-Session Claude Desktop

Run **multiple Claude Desktop profiles** in parallel on Linux (Fedora/Debian/Ubuntu) with:

- **Per-profile isolation** (separate login, settings, cookies)
- **Shared chat history & sessions** (single source of truth via symlinks)
- **Window-close → MCP cleanup** (X button kills all MCP children, no leak)
- **systemd-free, scope-free** lightweight watcher pattern (renderer-safe)
- **NVIDIA-aware launch** (preserves GPU env across `gio launch`)
- **WM_CLASS hybrid fix** (~300ms taskbar icon glitch instead of ~6s)
- **Race-tolerant** (rapid X-tıkla + dock-tıkla cycles handled)
- **Cowork-vm-service preserved** (single shared instance, never killed by other profiles)

Tested with: Fedora 43 · Wayland + XWayland · NVIDIA RTX 3070 · Claude Desktop v1.4758.0 (`aaddrick/claude-desktop-debian` build) · GNOME Shell 47

---

## Why This Exists

Claude Desktop is single-instance by default. If you want **4 separate workspaces** (4 monitors, 4 different accounts, 4 contexts), the upstream app gives you one window. This launcher gives you 4 — each with its own login, theme, dock icon, isolated state — but **without breaking** chat history sync, agent sessions, or the cowork VM.

It also fixes a known **MCP server process leak** ([anthropics/claude-code#1935](https://github.com/anthropics/claude-code/issues/1935), [#11778](https://github.com/anthropics/claude-code/issues/11778), [#33947](https://github.com/anthropics/claude-code/issues/33947), [#19201](https://github.com/anthropics/claude-code/issues/19201)) by adding a window-watcher that runs `pkill` cleanup when the X button is clicked.

---

## Architecture

```
~/.config/Claude/                   ← Profile 1 (master, untouched)
├── config.json                     ← Per-profile (oauth, prefs)
├── claude_desktop_config.json      ← Per-profile (MCP servers)
├── claude-code-sessions/           ← SHARED via symlink
├── local-agent-mode-sessions/      ← SHARED (skills, scheduled tasks, agent state)
├── git-worktrees.json              ← SHARED
├── pending-uploads/                ← SHARED
├── buddy-tokens.json               ← SHARED (daily token counter)
├── claude-code-vm/                 ← Per-profile REAL DIR (cascade-quit prevention)
└── ant-did                         ← Per-profile UUID

~/.config/Claude-{2,3,4}/           ← Other profiles, mostly symlinks → master
```

### Cleanup Flow (window-watcher pattern)

```
User clicks X
  ↓
Electron hides window (process stays alive — Claude Desktop background mode)
  ↓
Background watcher: 3 consecutive scans see no window for this WM_CLASS
  ↓
cleanup_profile_children: pkill -f user-data-dir + orphan MCP sweep
  ↓
PID file removed, scope cleaned
```

### WM_CLASS Hybrid Fix

```
Initial launch:    xdotool search --sync (X event, ~300ms)
                       ↓ window appears
                   xdotool set_window --class Claude-N
                       ↓
                   30s backup scan (5s interval) for raise/dialog windows
```

---

## Installation

```bash
git clone https://github.com/<your-user>/multi-instance-claude-desktop.git
cd multi-instance-claude-desktop

# 1. Install launcher
install -Dm755 bin/claude-launch ~/.local/bin/claude-launch

# 2. Install desktop entries
install -Dm644 desktop-entries/claude-desktop-2.desktop ~/.local/share/applications/
install -Dm644 desktop-entries/claude-desktop-3.desktop ~/.local/share/applications/
install -Dm644 desktop-entries/claude-desktop-4.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications/

# 3. (Optional) Profile-specific colored icons — see docs/icons.md

# 4. Launch
claude-launch 2  # Profile 2 — opens new isolated instance
claude-launch 3  # Profile 3
claude-launch 4  # Profile 4
```

After launch, log into each profile separately (one-time, oauth tokens stored in their own `config.json`).

### First-time profile creation

When you run `claude-launch 2` for the first time:
- Creates `~/.config/Claude-2/` with the proper symlink/realfile mix
- Copies `config.json` and `claude_desktop_config.json` from master
- Generates a unique `ant-did` UUID
- Launches Electron with `--user-data-dir`, `--class=Claude-2`

---

## Commands

```bash
claude-launch              # Profile 1 (default master)
claude-launch 2|3|4|5      # Specific profile
claude-launch all          # Launch all 4
claude-launch status       # Show open instances + watcher state
claude-launch kill 2       # Close profile + all children (pkill + sweep)
claude-launch kill-extras  # Close 2/3/4/5, keep main alive
claude-launch kill-all     # Close everything
claude-launch sweep        # Manual orphan MCP cleanup (panic recovery)
```

---

## GPU / Resource Tuning

`bin/claude-launch` has commented-out flag presets for sharing GPU with other AI workloads (embedding models, etc.):

```bash
# === GPU TAMAMEN DISABLE (CPU-only software rendering) ===
# CLAUDE_EXTRA_FLAGS+=(--disable-gpu --disable-software-rasterizer)

# === SwiftShader (CPU OpenGL emulation) ===
# CLAUDE_EXTRA_FLAGS+=(--use-gl=swiftshader)

# Or via env (one-shot):
CLAUDE_DISABLE_GPU=1 claude-launch 4
```

---

## Known Limitations

| Issue | Workaround |
|---|---|
| First X-button click on a profile leaks MCP children for ~15s before watcher detects window-gone | By design (3-scan threshold prevents false-positive on minimize) |
| `app.asar` is not modified (unlike forks like Cresnova/claude-desktop-mcp-fix) | Survives `dnf upgrade claude-desktop` cleanly |
| Cowork-vm-service is bound to Profile 1 | If Profile 1 closes, cowork dies too — auto-respawns on next launch |
| Profile 1's `.desktop` not modified | DNF override + xdg-mime claude:// handler stays untouched |

---

## Troubleshooting

### Black screen on profile launch
- **Cause**: System RAM exhausted (Electron renderer can't allocate)
- **Fix**: `free -h` → if `available < 1GB`, close Discord/Brave/etc.

### Wrong taskbar icon (orange instead of profile color)
- **Cause**: WM_CLASS race during cold start
- **Fix**: Already minimized to ~2s by hybrid event-fix. Wait, will self-correct.

### "Profile X already running" but no window visible
- **Cause**: Profile is in Claude Desktop's background mode (no windows)
- **Fix**: Click dock icon again — `raise_existing_profile` triggers Electron single-instance event → fresh window opens

### Cleanup doesn't fire after X click
- **Cause**: Watcher process died (rare, usually after system OOM-kill)
- **Fix**: `claude-launch kill <profile>` then relaunch — new watcher attaches

---

## Acknowledgements

- [aaddrick/claude-desktop-debian](https://github.com/aaddrick/claude-desktop-debian) — base packaging
- Anthropic for Claude Desktop
- Issue references: [claude-code#1935](https://github.com/anthropics/claude-code/issues/1935), [#11778](https://github.com/anthropics/claude-code/issues/11778), [#33947](https://github.com/anthropics/claude-code/issues/33947), [#19201](https://github.com/anthropics/claude-code/issues/19201), [#45507](https://github.com/anthropics/claude-code/issues/45507)

---

## License

MIT — see [LICENSE](LICENSE)
