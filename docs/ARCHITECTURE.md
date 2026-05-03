# Architecture — Multi-Instance Claude Desktop

Deep technical dive. Read AFTER `CLAUDE.md`.

---

## Table of Contents

1. [Profile Filesystem Layout](#1-profile-filesystem-layout)
2. [Process Topology](#2-process-topology)
3. [Cgroup / Systemd Scope Layer (X11/Wayland reality)](#3-cgroup--systemd-scope-layer)
4. [The 7 Subsystems](#4-the-7-subsystems)
5. [Lifecycle: Launch → Live → Close → Cleanup](#5-lifecycle-launch--live--close--cleanup)
6. [WM_CLASS Hybrid Fix (B+A pattern)](#6-wm_class-hybrid-fix)
7. [Cowork-vm-service Preservation](#7-cowork-vm-service-preservation)
8. [Race Condition Surface (and how we tame it)](#8-race-condition-surface)
9. [GPU/CPU/RAM Resource Sharing](#9-gpucpuram-resource-sharing)
10. [Failure Modes + Recovery](#10-failure-modes--recovery)

---

## 1. Profile Filesystem Layout

### Master profile (Profile 1)

```
~/.config/Claude/                      ← Master, all real files
├── ant-did                            ← UUID, generated once
├── bridge-state.json                  ← Per-profile (real)
├── claude-code/                       ← Claude Code CLI version cache  ★SHARED
├── claude-code-sessions/              ← In-Desktop Code session history ★SHARED
├── local-agent-mode-sessions/         ← Skills, agent state, scheduled tasks ★SHARED
├── git-worktrees.json                 ★SHARED
├── pending-uploads/                   ★SHARED
├── buddy-tokens.json                  ← Daily token counter ★SHARED
├── claude-code-vm/                    ← Cowork VM state (PER-PROFILE REAL DIR)
├── config.json                        ← Chromium prefs + oauth:tokenCache
├── claude_desktop_config.json         ← MCP servers config
├── Cookies, IndexedDB/, Cache/, ...   ← Chromium per-profile state
└── SingletonLock, SingletonCookie     ← Runtime locks
```

### Profile N (N=2,3,4)

```
~/.config/Claude-N/
├── ant-did                            ← Per-profile UUID (real)
├── bridge-state.json                  ← Per-profile (real)
├── claude-code              → ../Claude/claude-code              ★symlink
├── claude-code-sessions    → ../Claude/claude-code-sessions     ★symlink
├── local-agent-mode-sessions → ../Claude/local-agent-mode-sessions ★symlink
├── git-worktrees.json       → ../Claude/git-worktrees.json       ★symlink
├── pending-uploads          → ../Claude/pending-uploads          ★symlink
├── buddy-tokens.json        → ../Claude/buddy-tokens.json        ★symlink
├── claude-code-vm/                    ← REAL DIR (cascade-quit prevention!)
├── config.json                        ← Per-profile (own login token)
├── claude_desktop_config.json         ← Per-profile MCP config
├── Cookies, IndexedDB/, Cache/, ...   ← Per-profile Chromium state
└── SingletonLock                      ← Runtime
```

**Why `claude-code-vm` is REAL DIR per-profile, not symlinked**:
History: It WAS symlinked initially. Result: closing one profile released a lock in claude-code-vm/, other profiles' Cowork VMs detected the release as a "everyone quit" signal → cascade quit. Fix: per-profile real dir, each profile has its own VM state.

**Why other dirs ARE symlinked**:
- `claude-code-sessions/` — user wants to see same Code mode session history in all profiles
- `local-agent-mode-sessions/` — skills/scheduled-tasks should run regardless of which profile is open
- `pending-uploads/` — file upload queue is global
- `buddy-tokens.json` — daily token counter is per-account, not per-profile

---

## 2. Process Topology

When all 4 profiles are running, the process tree looks like:

```
systemd (PID 1)
├── gnome-shell                                  ← Desktop
│
├── /usr/lib/.../electron .../app.asar           ← Profile 1 main
│   ├── electron --type=zygote                   ← Chromium zygote (×N)
│   ├── electron --type=utility (NetworkService)
│   ├── electron --type=utility (AudioService)
│   ├── electron --type=renderer                 ← Chromium renderer
│   ├── electron .../cowork-vm-service.js        ★SHARED across profiles!
│   └── (MCP server children if any)
│
├── /usr/lib/.../electron --user-data-dir=Claude-2 ← Profile 2 main
│   ├── (zygotes, helpers same as P1)
│   └── (its own MCP servers)
│
├── /usr/lib/.../electron --user-data-dir=Claude-3 ← Profile 3 main
├── /usr/lib/.../electron --user-data-dir=Claude-4 ← Profile 4 main
│
├── bash claude-launch <args>                    ← Watcher #1 (Profile 2)
│   └── (sleeping, periodic wmctrl scan)
├── bash claude-launch <args>                    ← Watcher #2 (Profile 3)
├── bash claude-launch <args>                    ← Watcher #3 (Profile 4)
└── bash claude-launch <args>                    ← Watcher #4 (Profile 1, attached)
```

### Cowork-vm-service is SHARED

A single `cowork-vm-service.js` Electron utility process at `/run/user/$UID/cowork-vm-service.sock` serves all profiles. It's launched as a child of **whichever profile starts first** (typically Profile 1). Other profiles connect to it via the socket.

**Implication**: If Profile 1 closes, cowork dies → Profiles 2/3/4 lose VM service. They auto-respawn cowork on next launch (launcher-common.sh handles it).

---

## 3. Cgroup / Systemd Scope Layer

When launched via `.desktop` file from GNOME (gio launch), GNOME wraps the process in a systemd transient scope:

```
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/app.slice/
├── app-electron-12345.scope/    ← Profile 1
├── app-electron-12678.scope/    ← Profile 2
├── app-electron-12999.scope/    ← Profile 3
└── app-electron-13212.scope/    ← Profile 4
```

This scope is created BY GNOME automatically. Each Electron's children (renderers, helpers, MCPs) live in the same cgroup.

### Why we DON'T add our own `systemd-run --user --scope` wrapper

We tried it (v2 of claude-launch). Result: black screen on launch.
Root cause: `systemd-run --scope` modifies the process environment in subtle ways (different stdin handling, different inherited descriptors, different XDG bindings). Electron's renderer process detected this and failed to initialize the GPU compositor. Window opened with title bar but no content.

**Lesson**: Trust GNOME's auto-scope. Don't add nested scopes. See `docs/lessons/anti-patterns.md` §1.

---

## 4. The 7 Subsystems

claude-launch coordinates these 7 subsystems. Each has its own concern.

| # | Subsystem | File/Function | What It Does |
|---|---|---|---|
| 1 | **Profile init** | `ensure_profile()` | Creates Claude-N dir on first use, sets up symlinks, generates ant-did, runs cascade-quit drift repair |
| 2 | **Token sync** | `sync_token()` | Copies oauth token from master ONLY when target has none (preserves cross-account login) |
| 3 | **Singleton lock check** | `profile_already_running()` | Reads SingletonLock, validates target PID is alive |
| 4 | **Race protection** | `wait_for_cleanup()` | Waits up to 5s for old profile cleanup; removes stale SingletonLock |
| 5 | **Watcher singleton** | `kill_old_watchers()` | Reads PID file line 2 (watcher PID), kills before spawning new |
| 6 | **Window watcher** | `supervise_profile()` | wmctrl-scan loop: 3 consecutive scans without window → cleanup |
| 7 | **Window class fixer** | `fix_wm_class_persistent()` | xdotool --sync event-based fix + 30s backup scan |

Plus:
- **Process cleanup**: `cleanup_profile_children()` — pkill -f user-data-dir + orphan MCP sweep
- **Raise**: `raise_existing_profile()` — secondary electron launch (Electron single-instance trigger)

---

## 5. Lifecycle: Launch → Live → Close → Cleanup

### Launch (Profile N, where N = 2,3,4)

```
T+0ms     User clicks dock icon
T+10ms    GNOME shell reads claude-desktop-N.desktop, runs Exec= line
T+20ms    bash claude-launch N starts
T+30ms    profile_already_running() → checks SingletonLock
            → if YES: raise_existing_profile() → done
T+50ms    wait_for_cleanup() — waits for any old cleanup to settle
T+100ms   kill_old_watchers() — reads PID file line 2, kills old watcher (singleton)
T+150ms   ensure_profile() — verifies Claude-N dir, repairs drift
T+200ms   sync_token() — only if target has no oauth token yet
T+250ms   nohup env COWORK_VM_BACKEND=bwrap claude-desktop --user-data-dir=Claude-N \
            --class=Claude-N >/dev/null 2>&1 &
T+300ms   Capture electron PID via pgrep
T+800ms   Electron BrowserWindow appears (WM_CLASS="claude" lowercase!)
T+850ms   GNOME taskbar shows ORANGE icon (matches claude-desktop.desktop)
T+900ms   PID file written: line1=electron_pid, line2=watcher_pid
T+910ms   supervise_profile() spawned in background
T+920ms   fix_wm_class_persistent() spawned in background
T+930ms   xdotool --sync detects window create event
T+1100ms  xdotool set_window --class Claude-N
T+1200ms  GNOME taskbar updates → PROFILE COLOR icon
T+30000ms fix_wm_class_persistent backup scan ends
```

### Live state

- Electron main process alive
- Multiple zygote/utility/renderer children
- supervise_profile background loop polling wmctrl every 2s
- WM_CLASS = "Claude-N" (correct)
- PID file present in `~/.cache/claude-launch/profile-N.pid`

### Close (X button click)

```
T+0ms     User clicks X
T+50ms    Electron destroys window (Claude Desktop background mode — process stays alive!)
T+100ms   Window disappears from wmctrl
T+2000ms  supervise_profile detects: 1st missing scan (counter=1)
T+4000ms  supervise_profile detects: 2nd missing scan (counter=2)
T+6000ms  supervise_profile detects: 3rd missing scan (counter=3) → THRESHOLD
            log: "Claude-N window gone (X clicked), cleanup tetiklendi"
T+6050ms  cleanup_profile_children(N):
            pkill -TERM -f "user-data-dir=Claude-N"
T+9050ms  pkill -KILL fallback
T+11050ms sweep_orphan_mcps() — kill any PPID=1 MCP children
T+11100ms rm PID file
```

**Total cleanup time**: ~11s after X click. MCP children gone, scope freed, RAM returned.

### Why the 3-scan threshold?

If we trigger on 1st missing scan, we get false positives when user minimizes a window (mutter sometimes briefly removes from `_NET_CLIENT_LIST`). 3 scans = ~6s cushion that distinguishes "minimized/raised" from "destroyed".

---

## 6. WM_CLASS Hybrid Fix

### The bug

Electron's `--class=Claude-N` flag is supposed to set WM_CLASS on the X11 window. It works for utility processes (zygote, network) but **NOT for the main BrowserWindow** — that always opens with WM_CLASS="claude" lowercase.

This is partly Electron itself and partly Claude Desktop's `[Frame Fix]` patches that intercept BrowserWindow constructor.

### The cost of the bug

GNOME's taskbar matching algorithm:
```
new window appears
  → read _NET_WM_CLASS
  → search ~/.local/share/applications/*.desktop for StartupWMClass match
  → if "claude" (lowercase) found → match claude-desktop.desktop (Profile 1)
  → show ORANGE icon
```

So when Profile 2 opens, taskbar briefly shows Profile 1's ORANGE icon. ~5-6s later, our script fixes WM_CLASS and the icon updates to lavender.

### Fix architecture (B+A hybrid)

```
PHASE 1: Find electron PID (1s polling, max 30s)
  - 0.2s polling tightening reduces start jitter on slow systems

PHASE 2 (B): xdotool --sync wait
  - Blocks (kernel idle, ~0% CPU) on X CreateWindow event
  - Returns within ~100-300ms of window appearing
  - Immediately scans all PID's windows + xdotool set_window --class

PHASE 3 (A): 30s backup scan (5s interval)
  - Catches new windows opened by raise/dialog after initial fix
  - Also handles cases where Phase 2 timed out
```

### Resource cost

- xdotool process: 2.3 MB RAM (kernel-sleeping during --sync)
- CPU: ~0.5s total per launch
- Compared to old polling (~1-2s CPU): actually CHEAPER

---

## 7. Cowork-vm-service Preservation

cowork-vm-service.js is the bridge between Claude Desktop and the bubblewrap (or KVM) sandbox where Claude can run code. It's a separate Electron utility process listening on `/run/user/$UID/cowork-vm-service.sock`.

### Architecture decision: SHARED, not per-profile

Each Claude Desktop instance could spawn its own cowork. But that means:
- 4 cowork daemons × ~30 MB RAM each = 120 MB unnecessary
- 4 sandboxes (bubblewrap mounts) instead of 1
- Lock contention on `/dev/dri/*` for GPU acceleration

We use the upstream architecture: ONE cowork, shared via socket. First profile to launch starts it.

### Failure mode: Profile 1 closes → cowork dies

When Profile 1 closes, its child processes (including cowork) die. Profiles 2/3/4 lose VM connection. UI shows "Cowork unavailable" until next profile launch (launcher-common.sh `cleanup_orphaned_cowork_daemon()` checks if any UI is alive; if zero, kills+respawns cowork on next launch).

This is **acceptable** because users typically don't close Profile 1 alone — they close everything together or close the secondary profiles.

### Why we DON'T move cowork to its own systemd service

Tried. Cowork expects to be a child of an Electron main process (uses IPC over stdio in addition to the socket). Spawning it standalone misses initialization handshake.

---

## 8. Race Condition Surface

### Race 1: Rapid X-button + immediate dock-click

User: clicks X, immediately clicks dock icon.
- Old (v1) launcher: SingletonLock from old electron still alive (electron not yet exited) → new launch sees "already running" → no new instance → user sees nothing.
- v3 fix: `wait_for_cleanup()` waits up to 5s for old PID to exit, removes stale SingletonLock if found.

### Race 2: Multi-watcher proliferation

User: clicks X+open 8 times rapidly. Each launch spawns a watcher. Old watchers don't die when new launch happens. Result: 8 parallel watchers per profile, all triggering cleanup on next X.
- v3 fix: PID file 2-line format (line1=electron, line2=watcher). New launch reads line 2, kills old watcher BEFORE spawning new. Singleton invariant.

### Race 3: WM_CLASS late binding

Window appears at T+800ms with WM_CLASS="claude". GNOME shell binds taskbar icon at T+850ms (orange). Our fix at T+1100ms changes WM_CLASS to "Claude-N".
- v3 fix: xdotool --sync (Phase 2) detects window event within ~300ms instead of 5s polling. Glitch reduced from 6s → 2s.

### Race 4: Cowork respawn during multi-launch

User launches Profile 2, 3, 4 all at once (claude-launch all). Each launch checks "is cowork alive?". If launcher-common.sh sees cowork alive, all good. If cowork was killed mid-launch, multiple profiles try to respawn it simultaneously → socket bind contention.
- Mitigation: `claude-launch all` adds `sleep 1` between profiles to serialize cowork bind.

### Race 5: PID file race (cleanup vs new launch)

Cleanup function runs `rm PID_file`. If new launch starts in same second and writes new PID file, cleanup might delete the new file.
- Acceptable risk: PID file is rebuilt on next launch. No functional impact.

---

## 9. GPU/CPU/RAM Resource Sharing

### GPU (NVIDIA RTX 3070)

Each Electron uses ~50-150 MB VRAM for compositor + renderer textures. 4 profiles = ~200-600 MB VRAM. Well under 8 GB available.

GPU contention is minimal because:
- Most renderers are idle (waiting for IPC)
- Compositor is single-threaded per process

`bin/claude-launch` has commented-out flags for sharing GPU with other AI workloads:
```bash
# CLAUDE_EXTRA_FLAGS+=(--disable-gpu --disable-software-rasterizer)
# Or env: CLAUDE_DISABLE_GPU=1 claude-launch 4
```

### CPU

Each Electron main + renderer + helpers = ~5-15% CPU when idle. 4 profiles idle = ~20-60% of one core. Not an issue on multi-core.

### RAM

Each Electron full stack: ~300-500 MB. 4 profiles = 1.5-2 GB. Well under 16 GB system RAM.

**Critical observation**: When system RAM exhausted (other apps + 4 Claudes pushing >12 GB used), Electron renderer fails to allocate during cold start → BLACK SCREEN. This is not Claude's fault — it's overall system RAM pressure.

Mitigation: Close Discord, Brave, Spotify before opening 4 profiles. Or use `claude-launch sweep` to clean orphan MCPs.

---

## 10. Failure Modes + Recovery

| Mode | Detection | Recovery |
|---|---|---|
| **Watcher process dies** (rare, OOM) | Status shows `[no-watcher]` | `claude-launch kill <N>` + relaunch |
| **Stale SingletonLock** | New launch shows "already running" but no window | `wait_for_cleanup()` auto-cleans, or manual `rm ~/.config/Claude-N/SingletonLock` |
| **Black window after launch** | Render never shows | Close other RAM-heavy apps, retry |
| **Wrong taskbar color** | Profile shows orange instead of color | Wait 2s (auto-fix), or `dash-to-dock disable+enable` |
| **MCP children leak** | `claude-launch sweep` shows orphans | Run sweep command |
| **Cascade quit (one profile closes, all close)** | Multiple profiles die together | Bug — check `claude-code-vm/` is REAL DIR per-profile (not symlink) |
| **PID file corruption** | Status shows wrong PID | `rm ~/.cache/claude-launch/profile-N.pid` and relaunch |
| **GNOME shell scope conflict** | `app-electron-NNNN.scope` failed | `systemctl --user reset-failed 'app-electron-*.scope'` |

---

## Reference Material

- Upstream: [aaddrick/claude-desktop-debian](https://github.com/aaddrick/claude-desktop-debian)
- MCP leak issues: [#1935](https://github.com/anthropics/claude-code/issues/1935), [#11778](https://github.com/anthropics/claude-code/issues/11778), [#33947](https://github.com/anthropics/claude-code/issues/33947), [#19201](https://github.com/anthropics/claude-code/issues/19201), [#45507](https://github.com/anthropics/claude-code/issues/45507)
- Electron `--class` bug: see Electron tracker for `--class flag not applied to BrowserWindow`
- systemd transient scopes: [systemd-run(1)](https://www.freedesktop.org/software/systemd/man/systemd-run.html), [systemd.kill(5)](https://www.freedesktop.org/software/systemd/man/systemd.kill.html)
