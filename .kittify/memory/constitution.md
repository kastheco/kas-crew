# kas-crew Constitution

> Updated: 2026-02-20
> Version: 1.0.0

## Purpose

This constitution captures the technical standards and governance rules for kas-crew,
a TUI-based agent orchestrator for managing concurrent AI coding sessions.
Forked from claude-squad, extended with spec-kitty integration and per-instance
agent/model switching.

## Technical Standards

### Languages and Frameworks

- **Go** (1.23+)
- **bubbletea** for TUI (Elm architecture: Model/Update/View)
- **lipgloss** for terminal styling
- **bubbles** for TUI components (spinner, list)
- **cobra** for CLI command structure
- **go-git** for git worktree management
- **creack/pty** for tmux PTY integration
- Agent-agnostic: supports claude, aider, codex, gemini, opencode, amp via `-p` flag

### Testing Requirements

- Use `go test ./...` for all testing
- All features must have corresponding tests
- Standard library `testing` package with `testify` for assertions
- Mock tmux sessions and git worktrees for unit tests (no real subprocess spawning)
- No hard coverage target, but untested features are not considered complete

### Performance and Scale

- TUI must remain responsive at all times - never block the Update loop
- Support up to 10 concurrent instances without degradation
- tmux capture-pane polling at 100ms for preview, 500ms for metadata
- Minimize unnecessary allocations in hot paths

### Architecture Principles

- **No manager AI agent** - the TUI is the orchestrator. Zero token cost for orchestration.
- **tmux-native** - all agents run in tmux sessions. Preview via `capture-pane`, interaction via `attach-session`. No headless pipe mode.
- **git worktrees per instance** - each session gets an isolated branch and worktree. No conflicts during parallel work.
- **Agent-agnostic** - the `-p` flag or config `default_program` selects the agent CLI. kas-crew doesn't know or care which AI is running.
- **spec-kitty integration** (planned) - task source adapter reads `kitty-specs/*/tasks/WP*.md` frontmatter for WP status, dependencies, and suggested roles.
- **Per-instance agent switching** (planned) - each instance can run a different agent program, changeable at spawn time.
- **Daemon mode** - background process polls sessions for auto-accept mode.

### Deployment and Constraints

- **Linux**: Primary platform (full support)
- **macOS**: Secondary platform (supported)
- **Windows**: Experimental (WSL required)
- **Runtime dependencies**: tmux and gh must be installed and in PATH. Agent CLI (claude, aider, etc.) must be in PATH.
- Distributed as a single binary (`go install` or goreleaser)

## Governance

### Amendment Process

Constitution amendments via pull request. Strong justification required for changes.

### Compliance Validation

Code reviewers validate compliance during PR review. Constitution violations
should be flagged and addressed before merge.
