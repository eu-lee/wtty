# 09 — feat/wt-sidebar-keyboard  (Keyboard-native sidebar focus)

**Base:** `feat/wt-new-flow-m4` (stacks on PR #8; retarget the PR to `main` after #8
merges). · **Status:** READY. · **Worktree:** `~/Documents/Code/ghostty-wt-keyboard`.

## Purpose

Today the only way to interact with an *open* sidebar is the mouse. Make the sidebar
a first-class stop in the split-focus cycle: when it's open, `cmd+[` / `cmd+]`
(the default `goto_split:previous/next` keybinds) can move focus into it as if it
were another pane, and from there the list is fully keyboard-drivable. Requirement
from the human (2026-07-17): "if the tab is open, we can cycle to it as if it were
a new pane" — no mouse required.

## Target behavior

- Sidebar **open**, focus in a terminal split: `cmd+[` when there is no previous
  split (you're at the first/leftmost one) moves focus **into the sidebar list**
  instead of doing nothing/wrapping. Symmetrically `cmd+]` from the sidebar moves
  focus to the first terminal split. (`cmd+[` from the sidebar: your call — either
  no-op or wrap to the last split; pick one and note it in the PR.)
- Focus lands on a highlighted row (start at the active/selected worktree). Arrow
  keys move the highlight, **Return switches** to the highlighted worktree (existing
  `viewModel.select` → M3 switch path) and returns focus to the terminal, **Escape**
  returns focus to the terminal without switching.
- Sidebar **closed**: `cmd+[` / `cmd+]` behave exactly as today. Zero behavior
  change when the feature isn't in play.

## Implementation notes (verify all of this against the code first)

- **How goto_split reaches Swift:** keybinds are processed by the *focused surface*
  (Zig core), which raises a notification the macOS app handles (start from
  `BaseTerminalController` / `SplitTree` focus handling; grep `gotoSplit`,
  `focusSplit`, `FocusDirection` in `macos/Sources`). Intercept at that handler:
  if direction is previous/next, the sidebar is open, and the split tree has no
  split in that direction from the focused surface, move first responder into the
  sidebar instead.
- **The catch — no surface focus in the sidebar:** once the sidebar is first
  responder, key events no longer reach any terminal surface, so Zig keybinds
  (`cmd+]`, Escape, arrows, Return) will NOT fire. Handle keys on the macOS side
  while the sidebar has focus: SwiftUI `.focused`/`.onMoveCommand`/`.onKeyPress`
  (or an NSEvent local monitor scoped to sidebar-focused) — whatever proves
  reliable inside `NSHostingView`. This is the risky part; prototype it first.
- **Files:** `macos/Sources/Features/Terminal/WorktreeSidebarViewController.swift`
  (list view + a `focusSidebarList()` / `blurSidebarList()` seam on the
  controller), `WorktreeSidebarViewModel.swift` (highlighted-row state if you put
  it in the view model for testability), `BaseTerminalController.swift` /
  `TerminalController.swift` (goto_split interception), `SplitTree.swift`
  (read-only: how "no neighbor in that direction" is determined).
- The M4 "New worktree…" field already grabs focus via `@FocusState` — keep the
  two focus paths from fighting (e.g. arrows shouldn't move the row highlight
  while the branch-name field is being edited).

## Out of scope

- No new keybind actions or config keys (reuse `goto_split`); no changes to the
  Zig core; no `goto_split:left/right/up/down` sidebar entry (previous/next only);
  no type-ahead search in the list (the filter field exists).

## Verify

- `zig build` clean (`DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`,
  `PATH="/opt/homebrew/opt/zig@0.15/bin:$PATH"`).
- Existing suites stay green: `cd macos && xcodebuild test -scheme Ghostty
  -destination 'platform=macOS' -only-testing:GhosttyTests/WorktreeCreateTests
  -only-testing:GhosttyTests/WorktreeSidebarViewModelTests
  -only-testing:GhosttyTests/WorktreeCycleTests`. Add unit tests for any pure
  focus/highlight logic you extract; the first-responder dance itself needs a
  manual pass in the Debug app (`macos/build/Debug/Ghostty.app`).
- Manual: open sidebar → `cmd+[` from a single split focuses the list → arrows
  move highlight → Return switches worktree and focus returns to the terminal →
  `cmd+[`, then Escape backs out without switching. With sidebar closed, `cmd+[`
  and `cmd+]` are unchanged. With two splits, `cmd+[` from the right split still
  goes to the left split first, and only enters the sidebar from the leftmost.

## Workflow

- Commit here on `feat/wt-sidebar-keyboard`; push to `origin` (**eu-lee/ghostty
  only** — see AGENTS.md). Open a PR with `gh pr create --repo eu-lee/ghostty
  --base feat/wt-new-flow-m4`. **Do not merge** — the human merges.
- Commit messages end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`;
  PR bodies end with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
- Heads-up: `08-wt-new-base` (sibling branch) also edits
  `WorktreeSidebarViewController.swift`; expect a small conflict at merge time.
