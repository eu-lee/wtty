# 12 — feat/wt-manage  (Remove worktrees + deactivate sessions)

**Base:** `main` (M1–M4 + base-ref, PR #9). · **Status:** READY.
· **Worktree:** `~/Documents/Code/ghostty-wt-manage`.

## Purpose

From the human (2026-07-17): the sidebar can scroll between worktrees and add
new ones, but there's no way to *remove* a worktree or to *tear down a live
session* without deleting the worktree. Two distinct actions, both surfaced from
a sidebar row:

1. **Deactivate ("Close Session")** — drop a worktree's live workspace (its whole
   split tree, and therefore its ptys) so it returns to *inactive*. The worktree
   stays on disk; revisiting it lazily creates a fresh session, exactly as a
   first visit does today. This is the inverse of switching-into a worktree.
2. **Remove Worktree… (delete)** — `git worktree remove <path>`: delete the
   worktree from disk and drop it from the sidebar. Tears down its live session
   first if it has one. Never touches the branch (branch deletion is out of
   scope — a removed worktree's branch stays reusable).

**Definitions (same model as M3 / plan 09):** a worktree is *active* when it has
a live workspace in this window — it is the attached tree
(`WorktreeWorkspaceManager.activePath`) or has a detached workspace
(`manager.detached` keys). Everything else is *inactive*.

## Reference: read before touching anything

- `macos/Sources/Features/Terminal/TerminalController+WorktreeSwitching.swift` —
  `switchToWorktree`, `attachFallbackWorkspace`, and the detach/attach ownership
  rules (undo-manager invalidation, focus restoration). Deletion/deactivation of
  the *attached* worktree reuses this switching machinery.
- `macos/Sources/Features/Worktrees/WorktreeWorkspaceManager.swift` — the
  workspace store. `detached[key]`, `activePath`, `key(_:)`, `needsConfirmQuit`,
  and the `ghosttyDidCloseSurface` teardown that already drops a workspace when
  its last surface exits. Deactivation is the *deliberate* version of that.
- `macos/Sources/Features/Worktrees/Worktree.swift` — `GitWorktreeModel` and the
  `createWorktree` shape (runner, `createTimeout`, `WorktreeCreateError`). The
  new `removeWorktree` mirrors it.
- `macos/Sources/Features/Worktrees/WorktreeSidebarViewModel.swift` — published
  state and the `createWorktree` → refresh → `createError` inline-error pattern
  to mirror for removal. `WorktreeSidebar` pure helpers live here (add the new
  predicates alongside).
- `macos/Sources/Features/Terminal/WorktreeSidebarViewController.swift` — the row
  view (`WorktreeSidebarRowView`) and `installWorktreeSidebar` (where `onSelect`
  is wired — new callbacks wire here the same way).

## Design decisions (locked)

- **Affordance = per-row context menu** (`.contextMenu` on the row). It's
  uncluttered (keeps the TUI-esque row look plan 10 is chasing), keyboard/theme
  agnostic, and survives plan 10's `List` → `ScrollView`/`LazyVStack` change
  (swipe actions would not). Items: **Close Session** (only when the row is
  active) and **Remove Worktree…** (only when `!isMain`).
- **Confirmation dialogs ARE appropriate here.** The M4 "never alerts" rule is
  specifically about surfacing *git failures* — those still render inline. A
  destructive, on-disk, irreversible action (delete) and destroying live
  processes (deactivate with a running command) are exactly what a confirmation
  dialog is for, and match Ghostty's own close-surface confirmation. So: an
  `NSAlert` confirm before delete, and before any teardown that would kill a
  surface whose process `needsConfirmQuit`. Failures are still inline, never
  alerts.
- **The main worktree is never removable.** `git worktree remove` cannot remove
  the main working tree; the menu item is absent for `isMain`.
- **Branch is left intact.** `git worktree remove` only; no `git branch -d`. Note
  in the README that the branch remains.
- **Attached-target rule.** Deleting or deactivating the *currently attached*
  worktree first switches the window to the **main worktree** (via the existing
  `switchToWorktree` path, which detaches the target into `manager.detached`),
  then operates on the now-detached workspace. Main is always present and never
  itself a delete target, so it is always a valid fallback. Special case:
  deactivating **main while it is attached** switches to any other active
  worktree first; if there is none, it is a no-op (the window can't be left with
  no tree).

## Scope

### 1. Git model — `removeWorktree` (`Worktree.swift`)

- `GitWorktreeModel.removeWorktree(path: URL, force: Bool = false, forCwd cwd:
  URL) async -> Result<Void, WorktreeCreateError>`. Resolve the repo root like
  `createWorktree` does, then run `git worktree remove <path>` (append `--force`
  when `force`) at the root, with `createTimeout` (removal can touch many files).
  Reuse `WorktreeCreateError` (the `.git(stderr)` path already produces a clean
  user-facing message; a dirty tree surfaces as e.g. "contains modified or
  untracked files, use --force to delete it"). Do **not** add client-side
  validation — git owns it.

### 2. View model (`WorktreeSidebarViewModel.swift`)

- **Active set (published):** `@Published private(set) var activeWorktreePaths:
  Set<URL> = []`, storing canonical keys (`WorktreeWorkspaceManager.key`). The
  controller updates it after every switch / detach / deactivate / delete /
  workspace-close. **Heads-up:** plan 09 (`feat/wt-picker`, sibling, unmerged)
  introduces this *same* property for its active-first sort and active-only
  cycling — define it identically (name, type, canonical-key semantics) so
  whichever lands second keeps a single copy.
- **Predicates (pure, on `WorktreeSidebar`):**
  - `static func canRemove(_ worktree: Worktree) -> Bool { !worktree.isMain }`.
  - Add `isActive(_ worktree:in active: Set<URL>)` comparing via
    `canonicalPath` (or expose a small VM helper `isActive(_:)` reading
    `activeWorktreePaths`). Keep the comparison canonical — never raw paths.
- **Removal path:** `func removeWorktree(_ worktree: Worktree) async` mirroring
  `createWorktree`: guard `canRemove`, call `model.removeWorktree`, on success
  `await refresh(cwd:)`, on failure set a new `@Published private(set) var
  removeError: String?` (separate channel from `createError`; render near the
  list). Add `clearRemoveError()`. A `force` retry variant (`removeWorktree(_:
  force:)` or a bool param) backs the "Force remove" follow-up.
  - The *workspace teardown* is NOT done here (the VM doesn't own the manager) —
    the controller tears the workspace down first, then calls this. Keep this
    method purely the git-remove + refresh + error step so it stays unit-testable
    with a fake runner.

### 3. Controller orchestration (`TerminalController+WorktreeSwitching.swift`)

- `func deactivateWorktree(_ worktree: Worktree)`:
  - Resolve `key`. If it's a detached workspace: if any of its surfaces
    `needsConfirmQuit`, confirm first; then `manager.detached.removeValue(forKey:
    key)` and refresh `activeWorktreePaths`.
  - If it's the *attached* worktree: switch to the fallback (main; or another
    active worktree when the target is main) via `switchToWorktree`, which
    detaches it; then drop the now-detached workspace as above. No-op if no
    fallback exists.
- `func deleteWorktree(_ worktree: Worktree)`:
  - Guard `WorktreeSidebar.canRemove`.
  - If active, deactivate first (reusing the logic above — switch away if
    attached, then drop the detached workspace) so no live tree points at a
    directory about to be deleted.
  - `await viewModel.removeWorktree(worktree)`; on the dirty/locked git failure,
    the inline `removeError` renders and a "Force remove" affordance offers the
    `force: true` retry (second confirm).
  - Update `activeWorktreePaths` and let the VM refresh drop the row.
- Publish an `activeWorktreePaths` snapshot helper: a single
  `refreshActiveWorktreePaths()` that recomputes `{activePath} ∪ detached.keys`
  and assigns it on the VM; call it from `switchToWorktree`,
  `attachFallbackWorkspace`, `deactivate`, `delete`, and alongside the existing
  `ghosttyDidCloseSurface` drop (so a shell-exit that empties a detached
  workspace also updates the set — route that through the controller or post a
  notification the controller observes).

### 4. UI (`WorktreeSidebarViewController.swift`)

- `WorktreeSidebarRowView` (or the row site in `list`): add `.contextMenu`
  with, conditionally:
  - **Close Session** — shown when the row is active
    (`viewModel.isActive(worktree)`); calls `viewModel.onDeactivate?(worktree)`.
  - **Remove Worktree…** — shown when `WorktreeSidebar.canRemove(worktree)`;
    presents the confirm `NSAlert` (destructive style, red default), then calls
    `viewModel.onDelete?(worktree)`.
- New callbacks on the VM (mirroring `onSelect`): `onDeactivate: ((Worktree) ->
  Void)?`, `onDelete: ((Worktree) -> Void)?`, wired in `installWorktreeSidebar`
  to `deactivateWorktree` / `deleteWorktree`.
- Render `removeError` inline (small red caption, like `createError`) near the
  list; include the "Force remove" button when the error is the
  dirty/untracked case.
- Keep everything else (switching on tap, filter, new-worktree section)
  unchanged.

## Out of scope

- Deleting the git **branch** (worktree removal only). No pruning of stale
  admin entries beyond what `git worktree remove` does. No multi-select / bulk
  remove.
- New **keybind actions** for remove/deactivate (context-menu only for v1); note
  a follow-up could add `worktree_close`/`worktree_remove` actions the way plan
  09 adds `worktree_picker`.
- Any change to switching (M3), creation (M4/plan 08), the picker (plan 09), or
  the sidebar restyle (plan 10). No Linux work.
- Dirty/ahead-behind indicators, filesystem watchers, persistence (still v1
  non-goals).

## Verify

- `zig build` clean (`DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`,
  `PATH="/opt/homebrew/opt/zig@0.15/bin:$PATH"`; add
  `-Demit-macos-app=false` only when not testing the app).
- New/updated macOS tests, all green:
  `cd macos && xcodebuild test -scheme Ghostty -destination 'platform=macOS'
  -only-testing:GhosttyTests/WorktreeCreateTests
  -only-testing:GhosttyTests/WorktreeRemoveTests
  -only-testing:GhosttyTests/WorktreeSidebarViewModelTests
  -only-testing:GhosttyTests/WorktreeCycleTests`.
  - `WorktreeRemoveTests` (new, mirror `WorktreeCreateTests`): `FakeRemoveRunner`
    captures args → `git worktree remove <path>` is built; `force` appends
    `--force`; a dirty-tree stderr surfaces as an inline `.git` message; success
    triggers a refresh.
  - VM/pure coverage: `canRemove` false for main / true otherwise;
    `isActive` reads the canonical active set; `removeWorktree` refuses main;
    `removeError` is set on failure and cleared by `clearRemoveError`.
- Manual (Debug app, a test repo with 3+ worktrees):
  1. Open a background worktree (switch into it, switch away) → it's active
     (has a detached session). Right-click → **Close Session** → row goes
     inactive; `ps` shows its shell gone. Revisit → a fresh session starts.
  2. Start `sleep 999` in a background worktree, Close Session → confirm dialog;
     confirm → process gone; cancel → session intact.
  3. Right-click a clean inactive worktree → **Remove Worktree…** → confirm →
     directory gone from disk, row gone, branch still present (`git branch`).
  4. Remove a *dirty* worktree → inline error (no alert), row intact; **Force
     remove** → gone.
  5. **Remove the currently-attached worktree** → window switches to main, then
     the worktree is removed; no orphan ptys (`ps`), focus lands in main.
  6. **Close Session on main while attached** with another active worktree →
     window switches to it; with no other active worktree → no-op.
  7. Main worktree row has no **Remove Worktree…** item.
- Update the README worktree section: document Close Session (deactivate) and
  Remove Worktree… (delete leaves the branch; dirty needs Force).

## Workflow

- Commit here on `feat/wt-manage`; push to `origin` (**eu-lee/ghostty only** —
  see AGENTS.md). Open a PR with `gh pr create --repo eu-lee/ghostty --base
  main`. **Do not merge** — the human merges.
- Commit messages end with `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`;
  PR bodies end with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
- Conflict heads-up: `feat/wt-picker` (plan 09) and `feat/wt-ui-overhaul` (plan
  10) also edit `WorktreeSidebarViewController.swift` and the VM. This plan owns
  row *actions* (context menu) + `removeWorktree`; 09 owns the active-set + row
  *indicators* (design `activeWorktreePaths` identically to 09), 10 owns the row
  *look*. Whichever lands second rebases; resolutions should compose.
