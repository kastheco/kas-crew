# kas-crew Architecture Intelligence

> Codebase discoveries and architectural knowledge.
> Forked from smtg-ai/claude-squad v1.0.14.
> Updated: 2026-02-20

## System Overview

kas-crew is a Go/bubbletea TUI that manages multiple AI coding agent sessions in
isolated git worktrees. Each session runs in its own tmux session with live terminal
preview. The human drives orchestration directly -- no manager AI agent, zero token
cost for orchestration.

## Package Architecture

```
main.go                     Entry point, cobra commands, tea.Program setup
app/
  app.go                    Main bubbletea Model (home), Init/Update/View, key handling
  app_test.go               App tests
  help.go                   Help screen overlay content and state machine
config/
  config.go                 Config (default_program, auto_yes, daemon_poll_interval, branch_prefix)
  config_test.go            Config tests
  state.go                  AppState (help screens seen, instance persistence)
daemon/
  daemon.go                 Background daemon for auto-yes polling
keys/
  keys.go                   Key bindings and key name constants
log/
  log.go                    Logging (writes to /tmp/kascrew.log)
session/
  instance.go               Instance struct (title, path, branch, status, tmux+git lifecycle)
  storage.go                Instance serialization/deserialization (JSON)
  git/
    worktree.go             GitWorktree (create, setup, cleanup, prune)
    worktree_git.go         Git operations (diff, commit, push, branch management)
    worktree_ops.go         Worktree file operations
  tmux/
    tmux.go                 TmuxSession (start, attach, detach, capture-pane, status monitor)
    tmux_unix.go            Unix-specific SysProcAttr
    tmux_windows.go         Windows stub
    tmux_test.go            Tmux tests
ui/
  list.go                   Instance list panel (left side, 30% width)
  preview.go                Terminal preview pane (capture-pane content)
  preview_test.go           Preview integration tests
  diff.go                   Git diff pane
  tabbed_window.go          Tabbed container (preview tab + diff tab)
  menu.go                   Bottom menu bar with keybind hints
  overlay/
    text_input.go           Text input overlay (prompts)
    text.go                 Text display overlay (help screens)
    confirmation.go         Confirmation modal overlay
    overlay.go              Overlay positioning utilities
cmd/
  cmd.go                    Executor interface for command execution
web/                        Web UI (not yet integrated into TUI)
```

## Instance Lifecycle

```
NewInstance -> Start (firstTimeSetup=true) -> Running
                                                |
                                        +-------+-------+
                                        |       |       |
                                      Attach  Pause   Kill
                                        |       |       |
                                      Detach  Resume  Cleanup
                                        |       |
                                      Running Running
```

### Start Flow
1. `NewInstance()` - creates Instance struct with title, path, program
2. `Start(firstTimeSetup=true)`:
   a. `git.NewGitWorktree()` - creates branch `{prefix}{sanitized-title}` and worktree
   b. `gitWorktree.Setup()` - `git worktree add`
   c. `tmuxSession.Start(worktreePath)` - `tmux new-session -d -s kascrew_{name} -c {path} {program}`
   d. Attaches PTY to tmux session for size management

### Preview
- `tmuxSession.CapturePaneContent()` calls `tmux capture-pane -t {session} -p`
- Polled every 100ms via `previewTickMsg`
- Status detection (`HasUpdated()`) via `statusMonitor` polling tmux pane content

### Auto-Yes
- `statusMonitor` detects when agent is waiting for user input (prompt detection)
- If `AutoYes` is true, `TapEnter()` sends Enter key via `tmux send-keys`
- Daemon mode polls all sessions independently of TUI

### Pause/Resume
- Pause: commits dirty worktree, detaches tmux, removes worktree (keeps branch)
- Resume: re-creates worktree from branch, restores or creates new tmux session

## Key Interfaces

### Instance (session/instance.go)
The core unit of work. Wraps a tmux session + git worktree pair.
- `Start(firstTimeSetup bool)` - initialize or restore
- `Kill()` - terminate and cleanup
- `Pause()` / `Resume()` - checkpoint and restore
- `Preview()` - capture terminal output
- `Attach()` / detach via ctrl-q
- `SendPrompt(string)` - send text to tmux session
- `HasUpdated() (bool, bool)` - check activity and prompt state

### TmuxSession (session/tmux/tmux.go)
Manages a single tmux session.
- `Start(workdir)` - create detached session, attach PTY
- `Attach()` -> `chan struct{}` - attach terminal, returns channel that closes on detach
- `CapturePaneContent()` - read current pane content
- `SetDetachedSize(w, h)` - resize for preview rendering
- `statusMonitor` - goroutine that polls pane content for activity detection

### GitWorktree (session/git/worktree.go)
Manages git worktree and branch lifecycle.
- `Setup()` - `git worktree add`
- `Cleanup()` - remove worktree and delete branch
- `Remove()` / `Prune()` - remove worktree keeping branch
- `Diff()` - get diff stats against base commit
- `CommitChanges(msg)` / `PushChanges(msg, force)`

### Storage (session/storage.go)
JSON serialization of instances to `~/.kas-crew/state.json`.
- `SaveInstances([]*Instance)` - serialize all instances
- `LoadInstances() ([]*Instance)` - deserialize and restore sessions

## bubbletea Message Flow

```
User input (tea.KeyMsg) -> Update() -> state machine dispatch -> tea.Cmd -> tea.Msg -> Update()

Periodic:
  previewTickMsg (100ms)    -> refresh preview pane
  tickUpdateMetadataMsg (500ms) -> check activity, auto-yes, diff stats
  spinner.TickMsg           -> animate spinners
```

## Config

Config stored at `~/.kas-crew/config.json`:
```json
{
  "default_program": "claude",
  "auto_yes": false,
  "daemon_poll_interval": 1000,
  "branch_prefix": "{username}/"
}
```

## Planned Features (from kasmos)

### 1. spec-kitty Task Source Integration

Port `internal/task/speckitty.go` from kasmos. Key components:

- **wpFrontmatter struct**: parses YAML frontmatter from `kitty-specs/*/tasks/WP*.md`
  - Fields: work_package_id, title, lane, dependencies, subtasks, phase
- **SpecKittySource**: implements Source interface (Type, Path, Load, Tasks)
- **Auto-detection**: scans `kitty-specs/*/tasks/WP*.md` glob, picks feature with
  most recent active (non-done) WPs
- **Dependency resolution**: `resolveDependencyStates()` marks tasks as blocked
  when upstream deps aren't done
- **Lane mapping**: planned->unassigned, doing->in-progress, for_review->for-review, done->done
- **Role inference**: phase field maps to suggested role (planner/coder/reviewer/release)

Integration points in kas-crew:
- Instance list could show WP status alongside session status
- New instance creation could pre-populate from WP task descriptions
- WP lane could auto-update when session completes

### 2. Per-Instance Agent/Model Switching

Currently: global `-p` flag or config `default_program`.
Target: each Instance has its own `Program` field (already exists in struct),
selectable at spawn time via the new-instance dialog.

Integration points:
- Extend `N`/`Shift-N` flow to include program selection
- Show active program per instance in the list
- Allow changing program on session resume
