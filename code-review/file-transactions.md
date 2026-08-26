# File Transactions

> Review contract for source reads, saves, imports, rename/move/delete, version
> conflicts, and cleanup handoff.

## Scope and Owners

- The filesystem path seam owns retained absolute spelling, comparison
  identity, roots, containment, realpath checks, case behavior, and Unicode.
- Folder-relative path policy owns normalized in-folder names and protected
  write rules.
- Shared save and library mutation services own content versions and the
  transaction sequence. HTTP and MCP routes are adapters, not alternate
  mutation implementations.
- Content capability is format- and surface-specific: the Workbench edits
  Markdown and JSON, while library/Agent operations accept Markdown, HTML, and
  JSON text. Preview-only binary formats reject content writes. Rename, move,
  and delete are file mutations and do not imply content editability. The
  product-facing boundary lives in the
  [Documents format matrix](../design-docs/design/documents.md#format-capability-matrix).
- The conversion owner provides the cancellation barrier and post-mutation
  rediscovery/cleanup handoff.

## Path Invariants

- Every source operation resolves within an authorized member folder. Existing
  and creatable targets use realpath-aware containment; lexical containment is
  never claimed as symlink safety.
- Retained spelling is used for disk I/O and display. Platform-aware comparison
  identity is used only for equality, maps, and subtree operations.
- Drive and UNC roots remain intact. Creatable targets preserve the requested
  leaf spelling so case-only rename works on case-insensitive filesystems.
- A basename rename stays in the source parent; a folder-relative target is a
  move. Case-only rename may use a temporary same-directory hop.
- App-maintained derived paths are never writable source targets. Dot-directory
  Agent configuration remains writable even though dot-directories stay out of
  the knowledge index.
- Request-handling Adapters perform potentially slow realpath,
  canonicalization, existence, and type checks asynchronously. A sync path
  resolver is allowed only where its caller is already outside the shared Node
  request-handling event loop or the work is otherwise bounded.

### Known gap — request-path filesystem liveness

`FilesystemPathModule.resolveUnderAsync` is available and the Preparation
prepare, reprocess, and cancellation paths use it, but other request Adapters
still call the sync resolver. On macOS, its containment checks also reach the
sync mounted-volume identity implementation. Path safety remains enforced, but
slow or network-mounted folders can still block unrelated requests, and no
focused concurrency evidence currently establishes the required liveness.

## Save and Version Contract

- Text versions are hashes of complete source bytes, not mtimes.
- Renderer saves and Agent/MCP writes use the same version authority.
- Every content-write Adapter enforces the same accepted text-format set. A
  public tool description must not advertise a narrower or broader set than
  the operation actually accepts.
- A renderer save barrier compares the live editor value with its accepted
  source baseline. The asynchronously rendered dirty indicator is not a
  durability authority and cannot make context release skip a fresh edit.
- A byte-identical save is a no-op and retains the current version.
- Markdown and JSON persistence preserves supported BOM and line-ending
  conventions without manufacturing unrelated source changes.
- JSON Tree operations enter this same save path as minimal source patches.
  They preserve untouched whitespace, property order, escape spelling, numeric
  lexemes, and trailing-newline state; no whole-document serializer is a save
  authority.
- A `FILE_CHANGED` conflict must never automatically retry without
  `baseVersion`. The dirty editor buffer and newer disk source both remain
  recoverable until an explicit reload, merge, or overwrite decision.
- A synchronous editor-change marker makes the live editor value save
  authority before the renderer's dirty presentation commits. Merely mounting
  or reading an editor never publishes its normalized serialization.
- Conflict comparison content and version come from one disk read. While the
  conflict is unresolved, navigation, folder switch, window close, and reload
  remain behind the failed save barrier.
- Merge creates a dirty draft against the newer disk version. It is not
  durable until the normal versioned save path accepts it.

Regression shape:

```text
editor reads V1 → Agent writes V2 with base V1
→ stale editor write with base V1 is rejected
→ no automatic unversioned V3 may erase V2
```

### Renderer conflict recovery

Shipping behavior shows the newer disk source beside the unsaved editor source
and accepts one explicit decision at a time. Resolution and confirmed
close-and-discard share one owner; once either starts, competing choices stay
blocked until that decision succeeds, fails, or is cancelled:

- Reload adopts the disk snapshot.
- Overwrite publishes the editor source without a stale base.
- Merge opens a dirty draft with conflict markers and saves it against the disk
  snapshot through the ordinary versioned path.

### Known gap — instruction seeding on folder entry

The Shipping `/api/folder` path creates a missing root `AGENTS.md` create-only
when an ordinary existing folder joins or re-enters the library. The file is
visible, user-owned, and never overwrites an existing instruction, but the
write occurs before the user explicitly starts a folder Agent and is therefore
more invasive than J02's navigation promise and this contract's normal
explicit-mutation rule.

This exception is accepted for the current release so every opened folder is
Agent-ready. Future design should move seeding to an attributable Agent-start
or project-setup decision, or introduce an equally explicit folder-level
choice. Reviewers must not use this exception to justify other source writes on
folder open.

## Mutation Sequence

For rename, move, and delete:

1. Validate source, target, authorization, version, and collision intent.
2. Cancel every queued/running conversion or preview under the affected source
   and await native handle/lane release.
3. Perform the filesystem mutation.
4. Retire old derived, attention, ordering, and index ownership.
5. Rediscover preparation/index work under any new source identity.
6. Notify renderer state without treating that notification as truth.

Validation must happen before cancellation so an invalid request cannot disturb
healthy work.

`server/library-file-mutations.ts` is the shared source-mutation owner. The
active-folder HTTP routes and library/MCP operations only normalize their
transport-specific arguments and map results. A delete acknowledgement waits
for old source and derived index identities; rename/move removes the old
identity before reporting any optional new-identity indexing lag.

### Link Cascade

Rename and move can rewrite other files' Markdown/HTML relative links so they
keep resolving after the identity change.

- `server/links.ts` plans the affected link set against pre-rename disk state
  (`planRenameLinks`), then rewrites and saves affected files after the
  filesystem mutation (`applyRenamePlan`); a write failure rolls back both the
  rewrites and the disk rename.
- Cascade defaults on for every rename/move except a JSON target. The UI
  (`POST /api/rename-preview` in `server/routes/file-mutations.ts`, and folder
  rename in `server/routes/folders.ts`) previews the affected file/link count
  and lets the user opt out per rename; the MCP `move_file` tool applies
  cascade silently unless the caller passes `cascade: false`, with no preview
  step.
- Rewrite scope is inline Markdown link and image syntax plus HTML anchor
  `href` attributes only; reference-style links (a separate `[id]:` target
  definition) and non-anchor tags such as `img src` are not tracked and go
  stale silently on rename.
- Cross-folder moves and folder-level rename/move through MCP are out of
  scope: `moveLibraryFile` only supports moves within one member folder root,
  and only file-level moves are exposed to MCP.

## Import Publication

- A clipboard image reaches publication only after the default-off capture
  setting is enabled and the user accepts that specific offer. Dismissal and
  clipboard observation never create a source file.
- Multipart import streams into disk-backed OS-temp staging; it does not hold a
  multi-gigabyte body in memory.
- Publication copies into a hidden same-directory temporary, then uses an
  atomic no-clobber link where supported. A fallback exclusively creates the
  destination, records its identity, writes through that owned handle, syncs,
  and commits.
- Cancellation and recovery remove only an identity-proven partial reservation
  or owned temporary. They never delete a completed, externally replaced, or
  ambiguously owned destination.
- A successful publication is not rolled back because optional indexing fails.
  Direct-text indexing has its own bounded read budget.

## Folder Removal vs Filesystem Delete

Removing a library member deletes StashBase-owned state only. Deleting a folder
inside the active tree is a confirmed filesystem delete followed by subtree
cleanup. These operations must never share ambiguous copy or confirmation.

## Resource Bounds

Import accepts at most `8 GiB` per file and at most 500 files per request. Each
file is disk-staged and published independently; these limits are not
permission to buffer the request in memory. Direct-text indexing after a
successful import remains independently capped at `8 MiB`. Direct HTTP/MCP
text reads and manifest-known derived-text reads also reject responses above
`8 MiB`; sidebar previews read at most a small prefix.

## Implementation Map

| Role | Stable entry points |
|---|---|
| Path Interface | `FilesystemPathModule` in `server/filesystem-path.ts` and in-folder policy in `server/folder-relative-path.ts` |
| Save Interface | `validateEditableFileWrite`, `upsertSavedFile`, and `saveFileContent` in `server/file-save.ts` |
| Active-folder Adapter | `server/routes/files.ts`, `server/routes/file-mutations.ts`, and `server/routes/upload.ts` |
| Source Mutation Module | `server/library-file-mutations.ts` |
| Link Cascade Module | `server/links.ts` (`planRenameLinks`, `cascadeRenameLinks`, `applyRenamePlan`) |
| Library/MCP Adapter | `LibraryOperations` and MCP/HTTP transport adapters |
| Publication Module | `server/import-publication.ts` |
| Lifecycle Adapter | conversion cancellation, cleanup, and reconcile Modules in [Data Lifecycle](data-lifecycle.md) |
| Focused evidence | `server/filesystem-path.test.ts`, `folder-relative-path.test.ts`, `files.test.ts`, `upload.test.ts`, `library-file-mutations.test.ts`, `library-operations/index.test.ts`, renderer persistence tests, and the J03 conflict Journey |

## Validation

Run:

```bash
pnpm typecheck
pnpm test:conversion-scheduler
pnpm test:library-files
pnpm test:renderer
```

Run `pnpm test:e2e:functional` for user-visible CRUD, failed-save navigation,
or conflict UX. Cover POSIX, Windows drive/UNC, case-only rename, symlink
escape, target collision, disconnect/crash recovery, and the
`V1 → V2 → conflict` sequence at the lowest deterministic layer.

Related journeys: [J02](../design-docs/user-journeys.md#j02-add-and-open-a-folder),
[J03](../design-docs/user-journeys.md#j03-read-and-edit-source-documents),
[J04](../design-docs/user-journeys.md#j04-prepare-a-hard-to-read-file),
[J07](../design-docs/user-journeys.md#j07-converge-chat-into-a-document), and
[J08](../design-docs/user-journeys.md#j08-connect-an-external-agent-through-mcp),
plus the [J10](../design-docs/user-journeys.md#j10-turn-a-local-project-into-durable-agent-assisted-work)
core loop and
[J11](../design-docs/user-journeys.md#j11-turn-a-conversation-into-a-project)
for safe project target creation.
