# CLAUDE.md — multi-instance-claude-desktop Project Context

> **For AI agents working on this codebase**: Read this file FIRST. Then read `docs/ARCHITECTURE.md` and `docs/lessons/anti-patterns.md` BEFORE making any changes. This project went through 30+ iterations and we know which approaches break things.

## What This Project Solves

Run **N parallel Claude Desktop instances** on Linux with:
- Per-profile login + isolation (Firefox-style profiles for Claude)
- Shared chat history + agent sessions across all profiles
- Automatic MCP server cleanup when user clicks X (workaround for upstream MCP leak bug)
- Renderer-safe launch (no GPU/dbus context corruption)
- Race-tolerant (rapid open-close cycles handled)

**Tested on**: Fedora 43, Wayland + XWayland, NVIDIA RTX 3070, GNOME 47, Claude Desktop v1.4758.0 (`aaddrick/claude-desktop-debian`).

## Repo Layout

```
.
├── CLAUDE.md                   ← THIS FILE (read first)
├── README.md                   ← User-facing install + commands
├── LICENSE                     ← MIT
├── bin/claude-launch           ← THE script (~26 KB, single source of truth)
├── desktop-entries/            ← .desktop files for Profile 2/3/4
│   ├── claude-desktop-2.desktop
│   ├── claude-desktop-3.desktop
│   └── claude-desktop-4.desktop
├── docs/
│   ├── ARCHITECTURE.md         ← Deep technical dive
│   ├── journey.md              ← Chronological how-we-got-here story
│   └── lessons/
│       ├── errors.md           ← Errors we hit + root cause + fixes
│       ├── golden-paths.md     ← Proven workflows
│       └── anti-patterns.md    ← What NOT to try (we did, it broke)
└── .claude/skills/multi-instance/
    └── SKILL.md                ← Claude Code skill format (auto-loadable)
```

## Critical Constraints (HARD RULES)

| Rule | Why |
|---|---|
| **NEVER patch `app.asar`** | Root-owned, breaks on `dnf upgrade claude-desktop`, breaks signature |
| **NEVER use `systemd-run --user --scope` to wrap Electron** | Breaks renderer GPU/dbus context → black screen (we lived this) |
| **NEVER redirect Electron stdout/stderr to a real file** | Same renderer corruption symptom → black screen |
| **NEVER kill cowork-vm-service from a non-Profile-1 cleanup path** | Other profiles lose cowork connection → cascade failure |
| **NEVER assume Electron `--class` flag works on main BrowserWindow** | Upstream Electron bug — class always lowercase `claude` until xdotool fix |
| **NEVER assume X-button click kills Electron process** | Claude Desktop has background mode — process stays alive after window destroyed |
| **NEVER touch Profile 1's `.desktop` carelessly** | DNF upgrade overrides it; xdg-mime claude:// handler depends on it |
| **NEVER delete files with `rm -rf` near $HOME** | A safety hook here will block it; use targeted deletion |

## Critical Mechanisms (MUST UNDERSTAND)

### 1. Window-event-driven cleanup (NOT process-PID-driven)
Old approach: `while kill -0 $electron_pid; do sleep 2; done; cleanup`. Fails because Electron stays alive after X-click.
New approach: `while wmctrl shows window; do sleep 2; done; cleanup`. Triggers when user clicks X (window destroyed) regardless of process state.

### 2. Watcher singleton (PID file 2-line format)
PID file is `electron_pid\nwatcher_pid`. New launch reads line 2, kills old watcher BEFORE spawning new one. Prevents multi-watcher proliferation in race storms.

### 3. WM_CLASS hybrid fix (B+A)
- **B (event-based)**: `xdotool search --sync` blocks on X event, fixes WM_CLASS within ~300ms of window appearing
- **A (backup scan)**: 30s × 5s polling for new windows opened by raise/dialog

### 4. raise_existing_profile = secondary electron launch
When user clicks dock icon for an already-running profile: spawn a SECOND Electron with same `--user-data-dir`. Electron's built-in single-instance lock kicks in: secondary exits immediately, primary receives `second-instance` event and opens fresh window. This is **Profile 1's default behavior** that we replicate for Profile 2/3/4.

## When You Edit This Project

1. Read `docs/lessons/anti-patterns.md` to know what's been tried and broken
2. Read `docs/ARCHITECTURE.md` for the WHY behind the WHAT
3. Test each change against `docs/lessons/golden-paths.md` test scenarios
4. If you add a new behavior, document it in `docs/lessons/golden-paths.md`
5. If you hit a new error, add it to `docs/lessons/errors.md`
6. NEVER claim success without:
   - `bash -n bin/claude-launch` (syntax)
   - Live test: kill profile + relaunch + measure glitch + check cleanup

## Quick Commands for Debugging

```bash
~/.local/bin/claude-launch status        # Show all profiles + watcher state
~/.local/bin/claude-launch sweep          # Manual orphan MCP cleanup
tail -f ~/.cache/claude-launch.log        # Live launcher events
ls -la ~/.cache/claude-launch/            # PID files (line1=electron, line2=watcher)
wmctrl -lx | grep -i claude               # Window WM_CLASS
pgrep -af cowork-vm-service.js            # Cowork daemon state
```

## When Things Are Weird

| Symptom | Likely Cause | Fix |
|---|---|---|
| Black window | RAM exhausted (Electron renderer can't allocate) | Close Discord/Brave, check `free -h` |
| Wrong taskbar color | WM_CLASS race (rare after hybrid fix) | Wait 2s, will self-correct |
| Cleanup doesn't fire after X | Watcher process died | `claude-launch kill <N>` then relaunch |
| Profile won't launch | Stale SingletonLock | `claude-launch` script auto-cleans, or `rm ~/.config/Claude-N/SingletonLock` |
| Cowork dies when closing Profile 1 | Expected (architecture) | Auto-respawns on next launch |

## License

MIT — see [LICENSE](LICENSE)
