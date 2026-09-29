# ocogent

OpenCode TUI parallel worktree chats.

Status: design review; no product code scaffolded.

## Product decision

Build a standalone OpenCode plugin first. The first version should let a user start an independent chat in a Git worktree, leave it running while switching to another chat, see its status, return to it, and inspect its diff. The plugin is a prototype of the workflow, not a replacement for OpenCode's session or Git machinery.

After the prototype, propose only the small upstream core or TUI changes that evidence shows are needed for a safe, integrated experience. Do not maintain an OpenCode fork. Do not build remote pairing, a relay, a durable mailbox, or a general agent orchestration service as part of the MVP.

The plan's earlier “agent-to-agent mailbox” framing is retired. Codex's internal child-agent tools and Codex desktop's independent worktree chats are separate features. OpenCode's `subagent` tool is useful context for the former; the product in this plan follows the latter's user workflow.

## Goals

- Create a new, independent root session in a separate Git worktree.
- Keep the original checkout and its active session untouched while the worktree chat runs.
- Let the user switch between existing sessions, see running/idle and attention states, and reopen a session after restarting the TUI.
- Show the selected chat's changes for review.
- Use OpenCode's existing plugin, session, worktree, and TUI APIs wherever their released contracts suffice.
- Make worktree ownership and recovery understandable; never silently discard user work.

## Non-goals for the MVP

- Model-to-model messages, task handoff, sibling-agent communication, mailbox subscriptions, or an agent graph.
- Replacing or extending OpenCode's built-in `subagent` tool with worktree spawning.
- Automatically merging, rebasing, cherry-picking, or applying a worktree's changes to the original checkout.
- Moving a running session between directories or machines.
- Cross-machine pairing, remote control, NAT traversal, or a relay service.
- Reimplementing OpenCode's Git worktree manager, session database, or TUI navigation.
- A maintained OpenCode fork.

## Terminology and topology

- **Project checkout**: the user's existing OpenCode project directory.
- **Worktree**: a separate Git working directory made from the same repository by OpenCode's worktree API.
- **Chat/session**: an OpenCode conversation. A worktree chat is an independent root session created with the worktree directory as its location; it is not a `parentID` child session.
- **Parallel chat**: a second independent root session that may run while the user uses another tab/session.
- **Handoff**: a future, explicit operation that safely transfers a chat's working location between the project checkout and a worktree. It is not part of the plugin MVP because the existing `session.move` operation alone does not define Codex-like Git safety.
- **Subagent**: OpenCode's child-session feature. It remains distinct from an independent worktree chat.

The MVP topology is local and project-scoped:

```text
OpenCode project checkout
├── existing root session (original directory)
└── independent root session (worktree directory)
```

The plugin must not pretend a session-tree registry is the source of truth for these chats. Existing session records and worktree inventory are authoritative; plugin state is limited to namespaced association/presentation metadata that cannot be derived from them.

## User flows

### Start a worktree chat

1. From an existing project session, invoke the plugin's worktree command or palette action.
2. Choose a starting point supported by the released OpenCode worktree API. The first prototype may offer only the current committed checkout if branch/dirty-state semantics are unclear.
3. Create the worktree, then create a new root session whose location is that worktree directory.
4. Open/focus the new session in a TUI tab and submit the user's task.
5. If session creation or prompting fails after the worktree was created, retain the worktree and show its path and recovery action. Do not delete it automatically.

### Work in parallel and switch

1. The user returns to the original session or another root session.
2. The plugin's chat/worktree view shows sessions it can associate with the current project, their worktree path, and available status/attention information.
3. Selecting an entry opens or focuses its existing session tab. It must not start a new turn just to check status.

### Review changes

1. Select a worktree chat and request its current diff.
2. Show the diff and worktree path; explain that the changes remain isolated.
3. MVP ends at review. The user can use normal Git tools to keep or integrate work. The plugin does not claim that “handoff” is safe until the relevant core transition exists and passes its acceptance criteria.

### Resume after restart or failure

1. Re-read sessions and worktrees from OpenCode after startup; do not depend on an in-memory-only registry.
2. Match sessions to worktrees using durable session location plus a namespaced session metadata record where needed.
3. If an association is missing, stale, or ambiguous, show the session/worktree as unassociated and offer inspection. Never guess, move a session, or remove a worktree during recovery.

## Plugin and core responsibility

| Capability | Standalone plugin MVP | Core/TUI change if needed for parity |
| --- | --- | --- |
| Create a worktree | Call the released worktree API and report its returned path | Add one atomic “create worktree chat” operation only if the prototype exposes partial-creation or branch/dirty-state gaps |
| Create an independent chat | Call session create with the worktree location, then prompt it | Core owns atomic creation, rollback/recovery metadata, and any supported starting-branch/dirty-state transfer |
| Parallel runs | Open/focus root-session tabs; rely on OpenCode session execution | Native TUI worktree/chat chooser or tab indicators if plugin slots/keymaps are not sufficient |
| List/status | Read TUI session data/events and session APIs; derive only presentation state | Expose missing session list/inbox/status operations in the plugin domain if model tools or plugin runtime actually need them |
| Switch sessions | Use the TUI tabs `open`/`focus` API | Native routing/keybindings only if plugin-owned navigation proves awkward or incomplete |
| Diff | Use session diff/client API | Core owns any diff behavior that requires Git refs or conflict-aware transitions |
| Handoff/merge | Not in MVP; display diff and let user use Git | Core must own preflight, Git state changes, collision handling, and recoverable transition; TUI should expose the operation once safe |
| Messages/tasks/wait/interrupt | Not in MVP | Optional later plugin tools; core/API additions only for missing durable operations or event waits |
| Cross-machine | Not in MVP | Authenticated session routing and target scoping must be designed before transport; likely core/server work, not a TUI-only plugin |

The plugin can offer commands, keymap/palette actions, a panel or route, durable plugin storage, and session tabs through `@opencode/plugin/tui`. Prefer these existing surfaces. Do not replace the main session screen or create a private scheduler/agent runtime.

## State and persistence

OpenCode session and worktree records are the source of truth. The plugin should persist only what cannot be reconstructed, in a namespaced session metadata object (for example `metadata.ocogent`) or the supported durable plugin store:

- associated worktree path and project identity;
- origin session ID when the user explicitly starts from another conversation;
- creation time and the plugin schema version for this metadata.

Treat title and session directory as display/current-location data, not as authorization. Derive running/idle from OpenCode session state/events, not a stale copy in plugin storage. Persisted plugin data is a cache/association aid and must be rebuildable or safely ignored.

Worktree lifecycle for the MVP is `creating → associated → reviewable → retained`; on partial failure it is `orphaned/needs attention`. Do not add automatic expiration or cleanup. If a worktree is removed outside the plugin, retain the session record and display that its directory is unavailable. If a session is deleted, do not remove the worktree unless the user explicitly confirms a separate removal action after checking Git status.

On startup, refresh sessions and worktrees. Handle duplicate metadata, deleted directories, inaccessible repositories, missing sessions, plugin upgrades, and interrupted create/prompt sequences as visible recovery states. Recovery must be idempotent and must not alter Git state.

## Worktree and Git safety

- Require a local Git repository and use OpenCode's supported worktree strategy/API; no ad hoc `git worktree` subprocess in the plugin MVP.
- Never attach two active sessions to the same writable directory.
- Do not assume worktree creation carries uncommitted or ignored files from the source checkout. Verify the exact released behavior before advertising that choice.
- Do not switch the original checkout's branch as a side effect of creating a parallel chat.
- Do not automatically delete a worktree or its branch, even when a session completes.
- Do not call `session.move` to implement handoff. It changes a session's location but does not itself promise safe Git transfer, conflict checks, preservation of ignored files, or rollback.
- Before any later handoff/removal operation, require an idle session, inspect queued input and active child work, inspect Git status/branch ownership, and present the exact source/destination and expected Git operations. Core must repeat safety checks at execution time to close races.
- A later handoff must define behavior for dirty and untracked files, ignored files, branch already checked out elsewhere, merge conflicts, cancellation, interrupted processes, and recovery. If these cannot be handled safely, keep manual review/integration as the product boundary.

## TUI commands and presentation

Prototype only the small surface needed for the flows:

- `/worktree` (or a palette/keymap action): create a new independent session in a worktree.
- `/chats` (or a panel): list project sessions and worktree associations; select to open/focus.
- A per-entry status marker and worktree path; show errors and recovery actions inline or through standard toasts/dialogs.
- A diff action for the selected session.

Use plugin TUI `data.session.list/status/family`, events, `ui.tabs.open/focus`, plugin panel/route/slot APIs, and `ctx.client` operations when those are present in the exact target release. Do not require a custom visual agent graph or a new tab system for MVP. Keep keyboard bindings configurable and make all actions reachable without mouse interaction.

## Security and session targeting

The MVP is local to the current OpenCode server/project. A session ID or a matching title is not proof that a session belongs to the current project. Before displaying or acting on an entry, verify its project/location against the current project and confirm the associated worktree is managed by the same repository.

Do not expose general session mutation to model-facing tools in MVP. If message/task tools are added later:

- Scope targets to sessions explicitly associated with the current project/workflow; do not accept arbitrary session IDs as authority.
- Verify scope server-side on every operation, including list, message, interrupt, and wait; a plugin-local “observed sessions” registry is not an authorization boundary.
- Keep queue-only delivery distinct from a task that starts or steers a turn. Never allow a message to wake a session.
- Preserve OpenCode tool and permission checks. Agent-originated text is untrusted input and cannot bypass permissions or execute without a turn-starting operation.
- Define race behavior for session deletion, session movement, queued prompts, and interrupt during a run.

Remote access is deferred. Before reconsidering it, specify identity/pairing, revocation, least-privilege scopes, target authorization, transport security, host availability, replay protection, and behavior when either machine is offline. Do not expose an OpenCode server password or full API credential to another agent or peer host. OpenCode server transport alone is not a pairing or scoped-authorization design.

## Stages and decision gates

### Stage 0 — compatibility and API spike

1. Treat `~/opencode` and `~/codex` as read-only design references. The inspected OpenCode checkout is commit `7ef4a1a56e` (`v2.0.0-651-g7ef4a1a`); this is not the standalone build target and is not equivalent to the `v2.0.19` npm package just because package metadata in that checkout says `2.0.19`.
2. Target the published OpenCode `2.0.19` plugin/client/schema packages for the first prototype, with exact versions and a committed lockfile. Use `effect@4.0.0-rc.112` exactly where imported directly. Confirm the released package actually exports the TUI APIs the prototype needs before coding. If not, stop at the spike and select a released version that does; do not silently import from or link to the local clone.
3. Verify the released API signatures and behavior for worktree create/list/remove; session create with a location; session prompt; session diff; session list; TUI data status/events; and tab open/focus.
4. In a temporary Git repository, establish what worktree creation does with the selected ref, uncommitted changes, ignored files, and branch allocation. Record only behavior guaranteed by the target release.
5. Decide whether the published TUI plugin can create and focus a root session using its public client, or whether a narrowly scoped upstream API is needed.

Gate: proceed only if a released plugin/TUI API can create an isolated directory and a root session there without private imports. Otherwise propose the smallest upstream API change and wait for a released package before scaffolding.

## Open questions

Resolve these at the stated gates rather than guessing in the implementation:

- **Stage 0:** Does the selected published plugin package include the TUI `client`, session data, and tabs surfaces shown by the local checkout? If not, which published version first provides them?
- **Stage 0:** What exact worktree base/ref and dirty-file behavior does that release guarantee? The first UI must expose only choices the API can honor safely.
- **Stage 0:** Can the TUI plugin create a session at the new worktree location and focus it without private APIs or a core session-routing change?
- **Stage 1:** Is durable session metadata sufficient to restore worktree associations after restart, or is the plugin durable store required? Prefer the session record if it avoids a second source of truth.
- **Stage 1 gate:** Does the panel/command workflow feel native enough in the TUI, or is a small upstream TUI affordance needed?
- **Stage 2:** Which specific partial-failure, worktree initialization, or Git transition problem did the prototype demonstrate that requires an atomic core operation?
- **Before any handoff:** What exact Git operations, guards, rollback, and recovery contract will preserve both working directories?
- **Before optional mailbox/remote stages:** Who may address which sessions, what events constitute a wait result, and how are credentials scoped and revoked?

### Stage 1 — standalone plugin prototype

- Create the standalone `ocogent` package in this directory.
- Implement worktree chat creation, root-session association, tab open/focus, session list/status view, diff inspection, and recoverable partial-failure UI.
- Persist minimal association metadata and rebuild state from OpenCode after restart.
- Keep all worktree removal and Git integration manual.
- Use exact published package pins, a lockfile, and no `file:` dependencies, `bun link`, copied OpenCode source, or imports from `~/opencode` or `~/codex`.
- Do not add agent message/task/list/wait/interrupt tools unless the user workflow demonstrates a concrete need after the chat prototype.

Gate: test the complete new-chat → switch-away → return → inspect-diff → restart-and-resume flow on a clean Git repo and a repo with existing local changes. Decide whether the plugin UX and guarantees are sufficient or whether a core/TUI proposal is justified.

### Stage 2 — targeted upstream work, only with prototype evidence

Possible proposals, in priority order:

1. Atomic core operation to create a session associated with a worktree, including recoverable failure behavior and documented branch/dirty-state semantics.
2. Core-owned safe handoff operation after its Git contract is specified and tested; native TUI chooser/status affordance to expose it.
3. Add only the missing plugin `SessionDomain` operations needed by demonstrated plugin use (for example `list` or `inbox`). The underlying client `SessionApi` already includes these, while the Effect plugin domain is a restricted `Pick`.
4. A first-class permission action only if the eventual model-facing message/task tools require it.

Keep `ocogent` buildable against released packages. Upstream changes are not a reason to add a monorepo dependency or maintain a fork; consume them after release. If a proposal is rejected, retain the narrower plugin flow or revise the feature boundary.

### Stage 3 — optional mailbox tools

Only if the parallel-chat workflow demonstrates a need for model-to-model coordination, specify a separate small feature set. Decide whether to support independent root sessions, child subagents, or both; do not conflate them. Define exact queue/wake/steer semantics, durable target discovery, descendant/project authorization, `wait` event semantics, cancellation/timeouts, restart behavior, permission rules, and output/reporting before implementation.

At the inspected OpenCode API version, `session.synthetic` exposes `delivery` and `resume`; queue-only delivery is supported by the API. `session.wait` waits for the addressed session's current run to settle; it is not a mailbox/event wait. A future mailbox wait therefore needs explicit event/timeout semantics or a core event API. OpenCode's model-visible names are typically flattened with underscores; verify names through the exact released tool registry before choosing names. Do not document dotted names like `agent.message` as established model-facing identifiers.

### Stage 4 — remote, reconsider separately

No remote implementation is planned. Reopen this only after local adoption and an approved identity/authorization design. First establish whether an existing supported OpenCode remote connection model provides authenticated project/session routing with the required scopes. If it does not, write a separate threat and operations design before choosing core server work or a relay. Reachability between two machines alone is insufficient.

## Dependency, build, and release strategy

- Keep this as a standalone plugin project; do not add it to the OpenCode monorepo.
- Initial compatibility target: `@opencode/plugin@2.0.19`, `@opencode/client@2.0.19`, and `@opencode/schema@2.0.19` when imported directly; `effect@4.0.0-rc.112` when imported directly. Pin exact versions, commit the package-manager lockfile, and declare the supported OpenCode runtime version. Change the target only through an explicit compatibility update.
- Match the plugin entrypoint to its use: `@opencode/plugin/effect` for server/model plugin APIs and `@opencode/plugin/tui` for TUI extension APIs. Do not assume TUI APIs are available from the server-side plugin entrypoint.
- Keep the test/typecheck/build workflow runnable with both reference clones unavailable. Never use workspace aliases or package source paths from those clones.
- Do not publish before Stage 1 acceptance criteria pass. Tag releases against the tested OpenCode version and document upgrade compatibility.
- An OpenCode core/TUI PR is built and checked in its own checkout only when that PR is the active task; it is never a dependency of the published standalone plugin build.

## Verification and acceptance criteria

### Plugin acceptance

- `bun typecheck`, `bun test`, and `bun run build` pass using only pinned published dependencies.
- Tests use temporary real Git repositories for worktree lifecycle and exercise actual session/worktree API contracts where feasible; mocks may isolate UI presentation but must not stand in for Git safety behavior.
- Automated checks cover: create success; session-create failure after worktree creation; prompt failure; worktree removed externally; session missing; duplicate/stale association metadata; restart recovery; inaccessible repository; target session from another project; and no unintended cleanup or turn start.
- TUI smoke test verifies create, switch away, status update, focus existing session, diff view, and resume after TUI restart. Confirm queue/status inspection does not issue a prompt.
- Run the package checks with `~/opencode` and `~/codex` unavailable to prove there is no local checkout dependency.

### Any future handoff acceptance

- Define and test branch ownership, dirty/untracked/ignored files, conflicts, cancellation, concurrent activity, process interruption, and recovery.
- Refuse or clearly recover if a target branch is checked out elsewhere or a relevant session/agent is active.
- Never lose either checkout's changes; every failure leaves an inspectable source and destination state.
- Core performs preflight again at commit time; TUI/plugin preflight alone is insufficient.

### Optional mailbox acceptance

- A queue-only message is persisted and never starts or steers a turn.
- A task's idle/running behavior is explicit and tested; interrupt affects only an authorized target.
- Wait has a defined event set, timeout, cancellation, and return state; it does not busy-poll or wait only for the current turn when the product promises mailbox updates.
- Unauthorized cross-project, unrelated-root, deleted, or moved sessions are rejected consistently.

## Reference findings

Local references are for design only and must be checked against the exact released package before implementation.

OpenCode, inspected at `~/opencode` commit `7ef4a1a56e`:

- `packages/plugin/src/tui/context.ts`: plugin data/session list, family, status, events, tabs open/focus, panel/route/keymap and client context (`:69-101`, `:481-500`, `:516-532`).
- `packages/plugin/src/effect/session.ts`: restricted `SessionDomain` (`:153-170`); includes create/prompt/synthetic/move/wait/interrupt but not list/inbox.
- `packages/client/src/effect/api/api.ts`: full session list/create/location/move/prompt/synthetic/wait types (`:115-132`, `:199-209`, `:259-333`) and `SessionApi` including list, diff, inbox, interrupt (`:1411-1455`).
- `packages/plugin/src/effect/worktree.ts`: worktree API plus strategy registration (`:1-20`).
- `packages/core/src/tool/plugin/subagent.ts`: current `subagent` is a parent-linked child session with background support, not a root worktree chat (`:29-48`, `:184-213`).
- `packages/plugin-browser/`: package/build example if useful after the Stage 0 compatibility gate.

Codex, inspected at `~/codex`:

- `codex-rs/tui/src/chatwidget/worktree_picker.rs`: explicit new/fork conversation choices and current-checkout vs new-worktree flow (`:25-80`, `:84-118`).
- `codex-rs/tui/src/app/managed_worktree_creation.rs`: local worktree creation preflight and recovery concerns (`:60-95` onward).
- `codex-rs/core/src/tools/handlers/multi_agents_v2/`: internal agent communication tools; a distinct feature from independent worktree chats.
- `codex-rs/tui/src/app/agents_overview_*`: TUI presentation of agent threads, useful only if optional mailbox/agent coordination is later selected.

Official OpenAI documentation clarifies the reference product behavior; it is not an OpenCode implementation specification:

- [Worktrees](https://learn.chatgpt.com/docs/environments/git-worktrees): independent parallel chats, background work, and explicit Local/Worktree handoff.
- [Long-running work](https://learn.chatgpt.com/docs/long-running-work): use separate chats for independent parallel tasks and avoid two tasks writing to the same source.
- [Remote connections](https://learn.chatgpt.com/docs/remote-connections): connected-host operation, SSH host use, and handoff between hosts; remote access depends on host identity, security controls, and host availability.
- [Codex App Server](https://learn.chatgpt.com/docs/app-server): protocol for rich Codex clients; it does not establish a generic cross-chat mailbox or an OpenCode pairing design.

## Proposed revision — TUI worktree chats (2026-09-29)

This addendum is the current proposed MVP and supersedes conflicting earlier proposals in this document. It remains a specification, not an implementation authorization. Product decisions below are based on the local OpenCode checkout and must be revalidated against the exact published runtime at the Stage 0 gate.

### Goal and boundaries

Build a standalone, plugin-first OpenCode TUI extension that lets a user create an independent root chat in a Git worktree from the currently selected session, then find and switch among those chats using OpenCode's existing session UI. The intended feel is Codex-style parallel worktree conversations: distinct root sessions/checkouts, visible as ordinary OpenCode sessions, with explicit user-controlled review. This does not mean copying Codex's internal agent protocol or implementing a second TUI.

MVP includes one “New worktree chat” command-palette/keymap action, worktree and root-session creation, optional initial prompt only when explicitly supplied by the user, project-wide association/status discovery, switching/focusing, and a review/diff entry point using existing OpenCode surfaces where available. Keep the plugin standalone and default to a TUI-only package entrypoint. Do not maintain an OpenCode fork. Consider a narrowly scoped upstream core/TUI change only if a reproducible Stage 0 or Stage 1 finding proves the public plugin API cannot support a required MVP behavior; present that evidence and keep the feature usable without a private fork.

Non-goals for MVP: a custom `/worktree` or `/chats` command, a new chat browser replacing `/sessions`, agent mailboxes/messages/tasks, synthetic queues, wait/interrupt orchestration, automatic merge/handoff, automatic worktree cleanup, cross-machine/remote execution, host pairing, and a maintained OpenCode fork. These require separate designs and decision gates. In particular, do not infer that Codex/ChatGPT Worktrees documentation promises an equivalent OpenCode cross-session mailbox.

### Terminology and topology

- A *worktree chat* is an ordinary, independent root OpenCode session whose registered location is a Git worktree. It is not a child/subagent session.
- A *source session* is the currently selected OpenCode session. Its registered location and ref determine the base for the new worktree.
- A *project* is the OpenCode project identified by the source session's `projectID`; it is the scope for worktree/session discovery.
- OpenCode owns session/worktree records and Git operations. The plugin presents the action, coordinates API calls, correlates records, and reports recoverable failures. No second daemon/server is introduced for MVP.

The source of truth for a creation request is the selected session route resolved through TUI data/client APIs, not the process CWD, an arbitrary active project directory, or an unregistered path. Require a selected session with a valid registered location; otherwise disable the action or ask the user to open/select a session. The source session may itself be in a worktree: base the new worktree on that session's registered location/ref, as decided, and let the exact OpenCode release validate project membership.

### Runtime pin, package shape, and Stage 0 gate

The initial compatibility target is the published OpenCode 2.0.19 package family, not the local checkout's apparent package version. The inspected `~/opencode` tree reports commit `7ef4a1a56e` / `v2.0.0-651-g7ef4a1a`; its package metadata alone does not prove source/API identity with the published 2.0.19 release. Pin `@opencode/plugin` to `2.0.19`; use exact `@opencode/client` or schema packages only if the plugin imports them directly. Declare the supported runtime target and lock exact development dependencies. Re-run compatibility checks before any target-version update.

Use Bun `1.4.2` (`/home/flub/.bun/bin/bun` in the inspected environment) and commit the plugin's own lockfile. For MVP, avoid Effect and its RC dependency entirely. The package should expose only `./tui` (for example `"exports": {"./tui": "./dist/tui.js"}`) and register via the target runtime's TUI plugin configuration, first tested through `opencode plugin add <package>` in the scratch project; record the exact config destination and loader behavior observed at Stage 0. Import the SDK from `@opencode/plugin/tui`; this is distinct from the plugin's own `ocogent/tui` export. Do not add a root/default or `./server` export unless the compatibility probe demonstrates that the target runtime requires it.

Declare `@opencode/plugin@2.0.19` as the runtime dependency. Since the TUI SDK imports Solid types/runtime, declare `solid-js >=1.9.0` as a peer and pin `solid-js@1.9.15` exactly in the development lock for compatibility testing. The inspected local checkout's lock uses Solid `1.9.15`, `@opentui/core@0.5.12`, and `@opentui/solid@0.5.12`; local lock evidence is not proof of the published package's exact transitive versions. Use existing dialog/command/panel APIs and avoid custom JSX initially. Only if custom OpenTUI/JSX components are needed, declare `@opentui/core` and `@opentui/solid` as peers and pin `0.5.12` for tests after verifying against the release. Do not directly depend on client/schema/theme packages unless source imports them.

Stage 0 is a disposable compatibility spike before implementation: in a scratch directory (not this repository), use the pinned Bun toolchain and `bun add --exact` to install the published target packages; verify that the `./tui` export loads through the actual plugin configuration path, TUI types compile, and the needed worktree/session methods and fields exist. Scratch package/lock changes and Bun cache writes are acceptable; do not mutate this repository during the spike. Test whether a package with no server entry is accepted. Only if it fails for a confirmed runtime loader requirement, test and document a minimal no-op `./server` entry using `Plugin.define`; add that fallback only after evidence. If required API behavior is absent, record the smallest upstream change and stop for a decision before expanding scope.

### Plugin and core responsibility

| Concern | MVP owner | Contract / limit |
|---|---|---|
| Palette action and keybinding | TUI plugin | One action, stable ID such as `ocogent.new-worktree-chat`; no custom slash command. |
| Source selection | TUI plugin | Resolve current route's `sessionID`, load the registered session, use its `projectID` and `location.directory`. |
| Worktree creation | OpenCode core API | Call `ctx.client.worktree.create({ projectID, from, ... })`; core validates project inventory and owns Git worktree operations. |
| Root session creation | OpenCode core API | Call `ctx.client.session.create({ location: { directory }, title, ... })`; verify returned session has expected project/location. |
| Optional first prompt | TUI plugin via core API | Call `ctx.client.session.prompt({ sessionID, text })` only after explicit user entry/confirmation; failure must not discard the worktree/session. |
| Discovery/status/switch | TUI plugin + existing OpenCode data | Query project-scoped sessions/worktrees; correlate by project and canonical location; use session APIs/events for live state, not copied stale status. |
| Git safety, session persistence | OpenCode core | Plugin must not shell out to implement worktree mutation or silently clean up. |
| Merge/handoff, mailboxes, remote | Deferred | Requires separately scoped public APIs/security/ownership design; no plugin-side simulation of missing core semantics. |

The inspected TUI context exposes `ctx.client` as direct OpenCode client methods (`ctx.client.worktree.create`, `ctx.client.session.create`), not an `.api` nesting. `ctx.ui.router.current()` supplies the current route/session ID; resolve the session via `ctx.data.session.get(sessionID)` or the pinned release's equivalent. `SessionInfo` includes `projectID`, `location`, and creation time in the inspected source. Confirm all names and semantics in the Stage 0 target package rather than importing private monorepo files.

### User flow and TUI behavior

1. User opens/selects a source session in the desired project/worktree.
2. User invokes the single “New worktree chat” palette/keymap action. If no selected session, invalid location, unsupported project, or non-Git source exists, explain the reason without creating anything.
3. Show a short confirmation/input dialog with source session/location, proposed worktree branch/directory behavior, and optional initial prompt. Do not start a model turn by default.
4. Create a worktree from the selected session's registered location/ref, then create a normal root session at the returned directory. Validate the session's returned project/location before presenting success.
5. Focus/open the new session when tabs are available; otherwise navigate to it using the normal session router. The existing `/sessions` command remains the session discovery/resume surface. Add a small association/status affordance or dialog only if needed to explain worktree location and review state.
6. Let the user inspect changes using OpenCode's existing diff/session UI. No automatic merge, checkout, commit, branch switch, or deletion.

The inspected tabs API returns false when tabs are disabled. Therefore tabs are an enhancement, not a prerequisite: the plugin must still create, navigate, list/identify, and resume a worktree chat with `tabs.enabled() === false`. Avoid a second `/chats` listing command; use project-wide session data behind the one action/dialog only when it materially helps selection or recovery.

### API flow and lifecycle

Use the pinned public client equivalent of this sequence (adjust only to the verified target signatures):

1. Resolve the current session and require `session.projectID` plus `session.location.directory`.
2. Verify/normalize the source path and use it as `from`. Let core validate that it belongs to the specified project. Do not substitute the project root or another session's directory.
3. Call `ctx.client.worktree.create({ projectID: session.projectID, from: session.location.directory, ... })`; inspect the returned directory/result and surface branch/path details.
4. Call `ctx.client.session.create({ location: { directory: worktree.directory }, title: ..., ... })` to make an independent root session. Confirm association fields returned by core.
5. If user explicitly supplied an initial prompt, call `ctx.client.session.prompt({ sessionID: created.id, text })`. Do not use a synthetic queue. If a prompt fails, keep the worktree/session and report that the chat exists but did not start.
6. Refresh/query project sessions and worktrees and focus/navigate to the created session.

Do not promise semantics beyond the pinned core. The inspected Git implementation discovers the repository from the source directory and adds the worktree detached at the requested ref or HEAD. Verify the target release's exact base-ref behavior. The plugin must not claim that dirty, untracked, or ignored source state is copied unless verified and supported by the public API contract. Any failure after worktree creation leaves the worktree intact and displays its path plus a recovery action (retry session creation where safe, open existing session, or inspect manually). Never automatically remove a partial worktree.

### Persistence, discovery, and recovery

OpenCode session and worktree records are primary. Query `ctx.client.session.list({ project: projectID })` and `ctx.client.worktree.list({ projectID })` or verified target equivalents for project-wide discovery; TUI `data.session.list()` may be location-scoped and must not be assumed to include every worktree. Correlate a session to a worktree by same project and canonical registered location. Use `data.session.status` and live events where available. Do not maintain a duplicate status database.

Persist only a small namespaced association on the created root session if the target API supports safe metadata: `metadata.ocogent = { kind: "worktree-chat", sourceSessionID?: string }`. Do not duplicate project ID, directory, branch, or created time when those are derivable from OpenCode records. If custom session metadata is not available in the target API, use no sidecar for MVP; derive the relationship from the worktree/session project+location records and clearly mark unknown origin. Do not write into project Git files or commit user data.

Derived UI lifecycle states are `creating`, `associated`, `reviewable`, and `retained`; these are presentation states, not persisted state. A worktree without a matching session is an orphan/needs-attention case; a session whose worktree directory was removed is unavailable; a deleted session does not authorize worktree deletion. On restart, rediscover from OpenCode APIs and metadata, report inconsistent records, and provide explicit open/retry/manual recovery options. Never auto-clean stale/orphan records. Treat duplicate metadata as a warning and resolve by exact project/location, not guessing.

### Git safety and security

- Scope every API request to the source session's verified `projectID`; reject mismatches and avoid ambiguous session targeting.
- Treat path strings from UI, metadata, and API responses as untrusted: canonicalize for comparison, do not infer authorization from display names, and do not construct shell commands from them.
- Let OpenCode core validate project membership and own Git worktree mutations. No plugin `exec`/shell fallback for MVP.
- Do not overwrite an existing worktree/session; present collisions and let core/API return authoritative errors.
- Never auto-remove worktrees, delete sessions, merge, move branches, commit, or discard changes. Make partial success visible and recoverable.
- Remote or cross-machine operation remains disabled/deferred until host identity, authorization, project scoping, transport, liveness, and recovery are designed. Official Codex remote docs are product context, not permission or API contract for this plugin.

### Failure handling and tests

Stage 0 compatibility checks are a hard gate: no implementation begins until the published entrypoint/config loading, dependency peer behavior, and public API surface have been exercised. Stage 1 tests use temporary real Git repositories and the exact supported OpenCode packages/runtime. Cover:

- selected source session in the main checkout and in a registered worktree; verify `projectID`, `from`, base/ref, returned location, and root-session association;
- no current session, missing/unregistered location, non-Git source, inaccessible repository, invalid project membership, and API rejection before mutation;
- successful creation without an initial prompt; explicit prompt starts only the newly created session and never another target;
- worktree succeeds but session creation fails; session exists but initial prompt fails; neither causes cleanup or hidden state;
- tabs enabled and disabled; create, navigate/focus, switch away, return through `/sessions`, and TUI restart recovery;
- project-wide session/worktree discovery, canonical path association, duplicate/stale metadata, externally removed worktree, deleted session, and orphan worktree;
- status/diff inspection does not send a prompt; no custom queue/wait behavior exists;
- no automatic cleanup, Git mutation beyond core worktree creation, unrelated-project targeting, or reference-clone dependency.

Run formatting/typecheck/tests/build using only the plugin's pinned published dependencies and Bun 1.4.2. Smoke-test installer/config loading against the target runtime. Keep `~/opencode` and `~/codex` unavailable during the isolated package checks. A failure that points to an absent public API is evidence for a narrowly scoped upstream request, not a reason to silently vendor private source or fork OpenCode.

### Release, deferred phases, and decision gates

- Stage 0: scratch compatibility and package-loader spike; record exact commands, versions, outcomes, and whether a no-op server fallback is required.
- Stage 1: TUI-only MVP described above; publish only after acceptance tests pass and supported OpenCode version is explicit.
- Stage 2: only after MVP feedback, evaluate richer chat picker/panel and status UX using supported TUI APIs; do not add duplicate session commands by default.
- Separate future RFCs: explicit handoff/merge with Git preflight and recovery; mailbox/task/wait/interrupt coordination with exact target authorization and event semantics; remote/cross-machine operation with identity/security design. None are implicit extensions of MVP.
- Upstream core/TUI changes are conditional and minimal, justified by a concrete required capability absent from the pinned public API. Avoid a maintained fork; until an upstream change is released, either retain a documented limitation or pause the blocked feature rather than ship a private fork dependency.

### Remaining assumptions

- OpenCode 2.0.19 is the initial compatibility target, even though local checkout source is newer/different; Stage 0 may revise this target before coding.
- The target plugin loader accepts a TUI-only package under its TUI plugin configuration. A no-op server export is not assumed.
- Custom per-session metadata may not be exposed by the pinned API; association must gracefully work without it.
- OpenCode's public worktree endpoint is the sole Git mutation mechanism; the plugin does not promise copied dirty/untracked state.
- The selected session's registered location/ref is the chosen source even when it is itself a worktree.
