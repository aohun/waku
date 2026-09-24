# Plan 001: File-tree Copy Path and Add to Chat

> This plan is an outcome contract, not a step-by-step script. Understand the
> requirement and the recorded decisions, then design the implementation
> yourself against the live code. Run milestone validations as you go only if
> you are also the verifier — a delegated executor implements only, and
> verification happens outside its session. Stop on any STOP condition. When
> complete, update this plan in `plans/README.md`.
>
> Drift check: `git diff --stat 5454cd4..HEAD -- src/app/right_panel.rs src/app/composer.rs src/ui/menu.rs locales/app.yml locales/zh-CN.yml locales/ja.yml apps/web/src/lib/attachments.ts crates/waku-protocol/src/protocol.rs crates/waku-core/src/daemon.rs`

## Status

- Priority: P2
- Effort: S
- Risk: LOW
- Depends on: none
- Category: feature
- Execution: subagent
- Planned at: `5454cd4`, 2026-09-24

## Requirement

The native right-panel working-tree (Files) has no per-row context menu. Users
cannot copy a file or directory path, or stage that path into the active
composer, without leaving the tree.

When this is done: right-clicking (and the keyboard equivalent) any file or
directory row in the native working tree shows a two-item menu — Copy Path and
Add to Chat. Copy Path puts the entry's absolute path on the clipboard. Add to
Chat stages a composer attachment chip for that path via the daemon path-import
command (same end state as attaching a daemon host file from the composer
file picker), including directories.

## Decisions & tradeoffs

- **Add to Chat stages an attachment chip, not raw composer text**: Rejected:
  inserting `@path` or absolute path as plain text — chip + `@` on submit is
  the existing attach model and matches drop/paste. Based on:
  `src/app/composer.rs` `stage_daemon_attachment` (~2012) and web
  `addDaemonFile` / `importDaemonPathAttachment` in
  `apps/web/src/lib/attachments.ts` (~37).

- **Copy Path copies the absolute path only**: Rejected: relative-only, or two
  menu items (absolute + relative). Matches VS Code "Copy Path" and
  `WorkingTreeEntry.absolute_path` already on each row
  (`src/app/right_panel.rs` ~10 / render loop ~2740).

- **Menu contains only Copy Path and Add to Chat**: Rejected: full screenshot
  menu (Open in VS Code, Open with, Save as…). Out of scope for this slice;
  those need separate product work.

- **Native macOS client only**: Rejected: also adding a web `TreeRow` context
  menu in the same change. Web already has daemon path import; parity can
  follow later.

- **Add to Chat must use `ImportPathAttachment`, not client filesystem upload**:
  Rejected: calling `stage_attachment_paths` (reads client FS via
  `attachment_upload_from_path`). Tree paths are daemon-host paths; client
  upload breaks remote daemons. Protocol + daemon already implement
  `ImportPathAttachment` (`crates/waku-protocol/src/protocol.rs` ~217,
  `crates/waku-core/src/daemon.rs` ~673); native currently only calls
  `ImportAttachment` from `composer.rs` ~1969. Wire tree rows through
  background `ImportPathAttachment` then `stage_daemon_attachment`.

- **Reuse `skills.copy_path` for the Copy Path label; add a new
  `files.add_to_chat` (and locale translations) for Add to Chat**: Rejected:
  inventing a second copy-path key, or reusing `composer.attach_daemon` (that
  string describes the file-picker control, not a menu action). Based on
  existing `skills.copy_path` / `skills.path_copied` in `locales/app.yml`
  (~1080) and missing add-to-chat key today.

- **Files and directories both get both actions**: Rejected: files-only Add to
  Chat. Composer already supports `is_dir` attachments
  (`stage_daemon_attachment` ~2028; directory zip/size limits in composer
  attach helpers).

- **Keyboard parity required (decided while planning)**: Context menu must be
  openable without a mouse (same pattern as session rows: Shift+F10 /
  `open_context_menu`), because AGENTS.md requires every mouse-reachable
  control to be keyboard-operable. Based on `src/app/sidebar.rs` ~1985–2005
  wrapping rows in `context_menu`.

## Direction

Wrap each working-tree row in the existing `context_menu` primitive (see
sidebar session rows). Handlers: clipboard write of `absolute_path`; background
daemon `ImportPathAttachment` then `stage_daemon_attachment` with draft-owner
guarding like other attach paths. Keep all I/O off the render path (existing
source tests already forbid FS work in `render_right_panel_working_tree`).

### Milestone 1: Context menu + both actions wired

Working-tree file and directory rows expose Copy Path (absolute → clipboard)
and Add to Chat (daemon path import → composer chip). Locales cover EN and
existing JH/ZH files. Validation:
`cargo test --lib -- the_working_tree_render_path_does_no_filesystem_work` →
exit 0, plus any new focused unit tests added for this feature → exit 0.

## Landmines

- **Do not call `stage_attachment_paths` from the tree**: it uploads from the
  *client* filesystem (`composer.rs` ~1946–1970). Remote daemon hosts will
  fail or attach the wrong file. Mirror web `importDaemonPathAttachment`.

- **Render path FS ban**: `the_working_tree_render_path_does_no_filesystem_work`
  (`right_panel.rs` ~1382) forbids `std::fs::`, `read_dir`, etc. inside
  `render_right_panel_working_tree`. Menu open and clipboard are fine;
  path import must stay in a background spawn.

- **Draft-owner race**: other attach paths capture
  `selected_composer_draft_key()` before spawn and bail if it changed when
  applying (`composer.rs` ~1952–1984). Add to Chat must do the same so a
  session switch does not stage onto the wrong draft.

- **`context_menu` + left-click**: sidebar wraps the interactive row so left
  click still expands/opens; do not replace left-click with menu-only
  behavior (`sidebar.rs` ~2005–2025).

## Scope

In scope:
- `src/app/right_panel.rs` (and any small helper colocated on `Waku` that the
  tree menu must call, e.g. a thin ImportPathAttachment staging helper if it
  does not already exist on the same type)
- `src/app/composer.rs` only if a shared staging helper must be exposed or
  extracted for ImportPathAttachment (prefer reuse over duplication)
- `locales/app.yml`
- `locales/zh-CN.yml`
- `locales/ja.yml`
- `plans/README.md` (status)
- `plans/001-file-tree-copy-add-to-chat.md` (this file)

Out of scope:
- `apps/web/**` — web tree context menu deferred
- Open in VS Code / Open with / Save as… and other screenshot menu items
- Copy Relative Path as a separate action
- Diff-tree / other right-panel surfaces beyond the working-tree Files list
- Changing daemon or protocol attachment APIs

## Commands

| Purpose | Command | Expected result |
| --- | --- | --- |
| Focused unit tests | `cargo test --lib -- the_working_tree_render_path_does_no_filesystem_work` | exit 0 |
| Feature unit tests | `cargo test --lib -- right_panel` | exit 0 (includes new tests if added) |
| Manual UI (acceptance) | With `bun ./scripts/dev.ts` already owning Waku Debug.app: right-click a file and a directory in the Files tree → Copy Path pastes absolute path; Add to Chat shows a composer chip | both actions work for file and directory |

## Done criteria

- [ ] All listed non-acceptance commands pass.
- [ ] Working-tree rows show a two-item context menu: Copy Path, Add to Chat.
- [ ] Copy Path writes the entry absolute path to the clipboard.
- [ ] Add to Chat stages via `ImportPathAttachment` + existing composer
      attachment chips (files and directories).
- [ ] Keyboard can open the same menu (Shift+F10 or equivalent existing
      pattern).
- [ ] No client-FS upload path used for tree Add to Chat.
- [ ] Implementation follows every entry in Decisions & tradeoffs.
- [ ] No out-of-scope files changed.
- [ ] `plans/README.md` status is updated.

## STOP conditions

- A fact cited under Decisions & tradeoffs no longer holds.
- The outcome requires out-of-scope files.
- A validation command fails twice after one reasonable fix.
- `ImportPathAttachment` is unavailable from the native client API (would
  require protocol/client work outside this plan's intent).

## Maintenance notes

Prefer keeping tree attach logic next to other composer staging helpers so
future "add path to chat" entry points (command palette, drag from tree)
reuse one ImportPathAttachment path. When web gains a tree context menu,
reuse `importDaemonPathAttachment` rather than inventing a second model.
