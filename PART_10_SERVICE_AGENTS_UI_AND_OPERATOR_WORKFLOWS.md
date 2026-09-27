# Part 10. Service Agents: UI And Operator Workflows

This document defines reusable Settings cards and initialization workflows for
applications consuming backup, messaging and LLM agents. The display labels
`Gryphon Connection` and `Wyverne Connection` apply wherever those agents
are consumed; they do not prescribe a fixed list of applications.
Component deployment and protocol rules
are in [Part 09](./PART_09_SERVICE_AGENTS_DEPLOYMENT_AND_LIFECYCLE.md); shared visual foundations are in
[Part 01](./PART_01_INTERFACE_AND_INTERACTION_UNIFICATION.md). This Part takes
precedence over conflicting project-local UI documentation.

The linked raster exports are composition references:

- [Logs](./src/example-logs.png)
- [Backup](./src/example-backup.png)
- [Updates](./src/example-updates.png)
- [Wyverne Connection before binding](<./src/example - settings - wyvern/example - settings - wyvern connection - dont connected.png>)
- [Wyverne Connection after binding](<./src/example - settings - wyvern/example - settings - wyvern connection - connected.png>)
- [Wyverne Connection error](<./src/example - settings - wyvern/example - settings - wyvern connection - error.png>)
- [Wyverne Adapter choice before binding](<./src/example - settings - wyvern/example - setting -wyvern - connection choice overlay - not connected.png>)
- [Wyverne Adapter choice after binding](<./src/example - settings - wyvern/example - setting -wyvern - connection choice overlay - connected.png>)
- [Gryphon Connection before binding](./src/exampe-bot_connection-before.png)
- [Gryphon Connection after binding](./src/example-bot_connection-after.png)

Their true-black palette, square borders, typography, title hierarchy, row
geometry, spacing rhythm and full-width action placement are normative as
described below. Example names, versions and status values are placeholders.
They are embedded with exact crop measurements and corrected labels in
[Part 01 section 10.5](./PART_01_INTERFACE_AND_INTERACTION_UNIFICATION.md#105-settings-templates).
The schedule controls in the Backup example are now functional: the operator
approved moving schedule editing into each service. Incorrect draft headings,
example identities and combined trust decisions are corrected by the written
contract. This is the approved standard, not a claim that every deployed
implementation has already adopted it.

## 1. Shared Panel Anatomy

### Wyvern extension accepted 2026-09-19

Each service using an external LLM MUST have its own `Wyverne Connection` card with
instance identity, local/remote connection state, linked client identity,
allowed Adapter selection and functional bindings when needed. Provider keys
and remote model setup belong to the shared Adapter management flow in the
existing Updater TUI. A client card MUST NOT expose another client's bindings,
provider credentials or a host-wide administrative socket.

Installed, reachable, client-linked, Adapter-selected and function-ready are
distinct observations. An enabled runtime alone MUST NOT imply that a consumer
can use its LLM function. A failed request or temporary outage preserves the
selected binding; it does not silently select another Adapter.

Host reload/drain/update actions identify their shared impact and use typed
Updater operations with durable request/job IDs. Unknown or unavailable actions
remain unavailable until their lifecycle is implemented and verified. The
terminal profile in Part 01 applies to TUI rendering; the card anatomy below
applies to consumer web Settings. Implementation status is tracked in the
[Wyvern ledger](../wyvern/docs/IMPLEMENTATION.md); acceptance of this extension
does not imply production deployment. The source implements these flows.
Provider keys and Kernel Access Keys use transient masked input and never
appear in job receipts. Adapter edits and own-client binding changes MUST
carry the observed revision and a stable operation ID; stale edits fail
without replacing newer configuration.

### Web card layout

Each panel is a full-width titled `4x` Settings card with the current persisted
ordinal, four-dot reorder handle, outer border and inset outline defined by Part
01. It uses the shared tokens:

- `#000000` page and control surfaces;
- `#FFFFFF` primary text and outer lines;
- 80%-white secondary text and inset lines;
- configured accent for focus, selected and intentional primary emphasis;
- `#62FF8C` success/ready and `#F83D3D` error/denied, always paired with text;
- Space Grotesk for page/service display names and Consolas for card titles,
  labels, values, actions and status text;
- square corners, no shadows and no gradients.

The desktop card keeps the reference `1610px` outer width at the `1919px`
viewport and derives all internal positions from card-local padding. The title
header is `55px` high. Content starts `27px` below the divider; named groups are
separated by `38–48px` according to the reference rhythm. Standard buttons and
status rows are `40px` high. Status rows expand horizontally but never compress
their label, terminal text or `18px` semantic square. Long versions and errors
wrap beneath the value rather than increasing every sibling row.

At narrow widths the card becomes one column; paired controls stack; buttons
become full-width; no horizontal scrollbar is introduced. Focus, hover,
disabled, pending and error states preserve geometry. Every action is reachable
by keyboard, every overlay traps and restores focus, and color is never the only
status signal.

## 2. Backup And Neptune Panel

The [Backup panel](./src/example-backup.png), `1610x782px`, defines the
grouping and alignment. The owning service controls its own schedules and
explicit remote runs here. Central storage no longer exposes another editor
or backup-run action for those service policies.

Every module Backup card contains, in this order:

1. **System snapshot** — `Create and download snapshot`, actual full logical
   scope and exclusions. A single explicit action creates/downloads the ZIP.
2. **Restore snapshot** — `Browse local snapshot archive`, native picker
   inside the restore overlay, preflight and explicit replacement confirmation.
3. **Automatic backup to <storage>** — explanation, local agent state,
   Initialize/Repair, enabled state, interval, next due and manual remote run.
4. **Neptune version** — actual installed component version and its own
   `Check Neptune for updates` action using the universal update overlay.

Manual snapshot download saves to the operator computer. The remote-run action
uses the automatic pipeline to upload the same standard ZIP format to storage;
it does not start a browser download or substitute for the update saved-copy
gate. Neither workflow retains a permanent archive on the application host.

### 2.1 Status and action matrix

| Observed state | Required text | Primary action |
| --- | --- | --- |
| Agent confirmed absent | `Not installed`; local update-helper availability | `Initialize` opens the scoped form |
| Installed, service missing | `Detected · not linked` | `Initialize` reuses the daemon and enrolls this service |
| Job accepted/running | Current state and stable job ID | Disabled pending action; automatic polling |
| Linked and complete | Storage binding, complete pipeline set and last observation | Enable/interval controls and `Back up now` for each pipeline |
| Required pipeline missing/wrong | `Partial configuration` and per-pipeline rows | `Repair Neptune pipelines` with scoped confirmation |
| Unreachable with cached state | `Offline · last seen …` | Retry/reload without discarding job ID |
| Terminal failure | Sanitized reason and failed state | Retry after correction |

`Initialize` opens `Initialize Neptune` using section 8. Its protected
setup-code field uses the enrollment protocol's 32-character format and
15-minute lifetime; the application Access Key has no such restriction.
The instruction links to authorized storage identity/setup-code management,
not to a schedule editor. Resolve the current approved destination through
the control plane. Completion requires every declared pipeline to be enrolled
and verified. Preserve imported schedule values; initialization of a genuinely
new profile leaves automatic backup disabled with a 24-hour interval.

### 2.2 Schedule Controls And Persistence

The schedule group contains a labeled `Enable automatic backups` checkbox,
`Interval in hours` numeric field, independent `Back up to <storage> now`
action, and textual desired/applied policy status. Show next run, last
successful commitment, current run/retry state and sanitized error.

The owning application's authenticated API is the operator write path.
Each mutation binds to its server-derived service/deployment/pipeline and
expected policy revision; browser input cannot select another owner's scope.
There is no global page Save button. Toggle commits once on change. Interval
accepts finite integer hours, minimum 1 and any declared protocol maximum;
commit once on Enter/blur, deduplicating their overlap. Editing individual
digits is not a sequence of schedule changes.

Keep validation errors next to the field. During save, disable duplicate
mutations and distinguish draft, saving, saved/pending-application and applied.
A failed server commit restores the last confirmed value; an invalid local
draft stays explained until corrected/cancelled. A revision conflict refreshes
the confirmed policy and preserves the user's proposed change for explicit
review; it never silently overwrites a newer policy. Successful persistence
alone does not authorize a green `Schedule applied` indicator.

While the agent is offline, a reachable authoritative backend may accept a
durable desired-policy change as pending. If it cannot persist it, report
unavailability instead of simulating a queue in browser storage. The UI
continues observing applied revision after reconnect, page reload and other-tab
edits. Dates are stored in UTC and shown with the operator's timezone.

Enabling/changing interval schedules the next run; it does not start a backup.
Disabling blocks future scheduled work without corrupting an accepted transfer.
`Back up now` works independently of enabled state, carries an idempotency
key and follows a durable run ID through export, upload/retry and remote
commitment. Disable/observe it while the same project is busy. Queued is shown
only if the server actually supports and accepts a durable queue.

The agent, not the page, runs the schedule. Closing the browser, restarting the
service/agent or losing a control-plane connection must not reset its applied
policy. The agent reports truthful due/overdue and retry state. The lifecycle,
single-writer migration and missed-run policy are defined in
[Part 09 section 5](./PART_09_SERVICE_AGENTS_DEPLOYMENT_AND_LIFECYCLE.md#5-ongoing-neptune-interaction).
Backup/restore includes these settings under Part 03.

### 2.3 Pipeline Profiles

| Profile | Required groups | Success criterion |
| --- | --- | --- |
| Recovery archive | One scoped archive schedule/status/run group | Registration, applied policy and remote receipt are observable |
| Recovery archive plus dedicated file mirror | Two separately named groups with independent enabled/interval/status | Both declared pipelines are complete; one healthy pipeline is partial success only |
| Folder synchronization | Source-client-owned folder policy with its own status and limits | Correct source/destination scope, policy acknowledgement and transfer result |

An initialization code may establish a combined profile, but does not merge
the workers' schedules, credentials or outcomes. Repair touches only missing
or invalid parts of the declared owner profile.

## 3. Central Storage And Fleet Observation

The central panel retains archive/mirror inventory, identity/setup-code
management, quotas, revocation, agent version, heartbeat and observed/applied
policy. It may display read-only desired/applied schedules and last results
with a current approved link to the owning service's Backup card.
It MUST NOT retain schedule editing, schedule enable toggles or explicit
service backup-run buttons. A hidden old endpoint cannot remain an alternative
operator write path that bypasses the service policy scope.

Privileged identity/quota/revocation actions still require owner authorization
and fresh reauthentication where specified. Their authority is not delegated
to ordinary service cards. A storage-side revocation/suspension blocks access
and is visible to the owner service; it does not pretend the operator disabled
their schedule. Shared fleet release operations follow the update contract
and do not reinstate centralized backup scheduling.

Preserve the distinction between UI ownership and physical policy storage:
an existing central backend may persist and relay the service-owned record
through scoped APIs. There is exactly one record/revision and one authoring
workflow for that scope, as described in Part 09's migration contract.

## 4. Gryphon Connection

The [before](<./src/example - settings - gryphon/example - settings - gryphon - not connected.png>) and
[after](<./src/example - settings - gryphon/example - settings - gryphon - сonnected.png>) examples define the layout.
The old `Bot connection` title MUST be replaced by `Gryphon Connection`
in every consuming Settings view, navigation/accessibility label and embedded
operator guide. Product-specific aliases in those PNGs are replaced by the
current service's real identity.

Every application that consumes the messaging gateway renders this card.
An application that does not consume it does not render a dummy connection.
The card contains one **Gryphon bot binding** group: local-agent reachability,
applied-connection reachability, the selected adapter alias or `none`, and a
link/change function action. The bot-selection overlay shows paired bots,
non-secret details for the active adapter and service-scoped unlink.
Registration, `/link CODE` pairing and shared release checks/updates occur in
`sudo updater tui` and do not have actions in service Settings.

### 4.1 Status and action matrix

| State | Required controls |
| --- | --- |
| Gateway confirmed absent or service unenrolled | Keep function linking disabled; show agent reachability as unavailable |
| Previously detected agent unreachable | Preserve last-known binding and show agent reachability as unavailable |
| Gateway ready and service unenrolled | Show `Link Gryphon function`; list only paired adapters in the overlay |
| Service connected | Show the selected alias, `Change Gryphon function`, and service-scoped `Unlink all adapters` in the overlay |
| Service binding unavailable | Show applied-connection reachability as unavailable; do not imply a verified account |

The link-function overlay lists Gryphon-provided paired adapter aliases and
usernames. It never asks for or displays a Telegram API token or `/link` code.
When the owner selects an adapter, Gryphon automatically applies that adapter's
verified Telegram account to this service connection. The UI reports success
only after Gryphon confirms both the connection and its service binding.
Unlink is a consequential action with consequence text. It removes this
service's use of the selected adapter without deleting a shared Gryphon
registration or changing another service's binding.

Adapter registration and first pairing take place only in `sudo updater tui`:
the operator supplies alias and token, then sends the TUI's one-use `/link CODE`
in a private chat with the adapter. Consumer Settings cannot read or collect the
token or issue a code. If no paired adapter exists, explain the TUI prerequisite
without inventing a ready option. A missing Gryphon instance is installed only
from the TUI. Existing card and overlay geometry remains the visual reference;
the controls above define their current behavior.

## 5. Updates Panel

The [Updates panel](./src/example-updates.png) defines the alignment for application
version, local update-helper reachability, registry reachability and the
full-width `Check for updates` action. It represents the application release
path; applicable shared-component controls identify their own targets separately.
Gryphon's card provides the TUI instruction instead of a service-side check.
Its source canvas is `1610x566px`.

The permanent card contains:

- installed application version;
- local update-helper availability;
- approved registry/repository-policy availability;
- `Check for updates`, which opens the discovery and installation overlay.

### 5.1 Universal Dialog And References

The normative six-image sequence, exact palette/font/spacing ledger and
responsive/accessibility behavior are embedded in
[Part 01 section 10.8](./PART_01_INTERFACE_AND_INTERACTION_UNIFICATION.md#108-update-dialog-templates).
The complete action, backup and job contract is
[Part 05 section 34](./PART_05_CI_RELEASES_AND_LOCAL_UPDATES.md#34-operator-update-ui).
These rules cover every current and future consuming application; names and
versions in the images are illustrative data.

Use the same overlay for the application and each consumed shared component:

1. Entry opens the overlay and starts discovery; `Check again` repeats it.
2. Distinguish checking, available, no newer compatible version, blocked,
   offline/stale and failed discovery. Show actual target/version and the
   checks performed; discovery never installs.
3. An application's `Install <version>` opens the mandatory standard full-ZIP
   warning. `Create backup and install` continues only after verified native
   save completion. For an ordinary download, require explicit saved-copy
   acknowledgement before enabling the separate Install action.
4. A shared component's overlay names target/shared impact; its Install action
   explicitly confirms the operation without an application backup warning,
   creation or download.
5. Accepted work shows a durable job ID, actual state/message and a measured
   or indeterminate progress bar. Disable duplicate mutations. Closing the
   overlay, reload and reconnect preserve observation of the existing job.
6. Verified completion refreshes installed version/health and performs fresh
   discovery. Initial no-update has no fabricated job card; post-install
   no-update retains the completed job. Failure and rollback remain explicit.

The application's own installed version and the update helper's version are
separate observations. Each consumed component has its own version/check/
install controls in the corresponding Settings card or named group. This
includes the update helper itself and applies regardless of component role;
unused agents receive no misleading connected/update card.

### 5.2 Shared Scope And Recovery

A shared-component confirmation identifies that the host instance may serve
multiple applications. Authorized actions use typed operations and exact
targets; they do not create another daemon or reset enrollment per consumer.
Their ordinary check, install and final verification are reproducible in the
connected application's interface without a native CLI.

Remote operations show queued/waiting/offline until the target agent reports
the selected running version and health. An accepted command or successful
poll alone is not completion. Progress never invents a percentage from time.

For application recovery, the panel states that a later rollback or recovery
after helper/host restart requires the original saved ZIP. A server-backup
message in an old raster is obsolete; no archive is retained on the application
host. The rollback control uses the authoritative job capability and requires
explicit confirmation at a safe boundary. A regular host reboot preserves
installed data and does not ask the operator to restore every application.

An approved application theme changes documented visual tokens, not the
sequence, required controls, data or recovery contract. Capture its adapted
states against the same measurements and acceptance matrix.

## 6. Copy, Feedback And Error Recovery

- Use contextually labeled `Initialize`, `Repair Neptune pipelines`,
  `Link Gryphon function`, `Change Gryphon function`, `Check for updates` and `Install <component>
  <version>` consistently.
- Avoid `Connected` without naming what is connected: gateway, service
  function, Telegram user, archive pipeline and mirror pipeline are different
  states.
- Accepted asynchronous work uses informational/pending feedback. Only a
  terminal verified outcome uses success.
- Errors state what remained unchanged and provide the next safe action. Secret
  material, raw headers, filesystem credentials and unrestricted command output
  never appear.
- A disabled action has adjacent explanatory text; hover alone is not the
  explanation.
- Notices follow the shared Part 01 duration and stacking rules, while durable
  job progress remains inside the overlay or panel and is not communicated only
  by a transient toast.

## 7. UI Acceptance Matrix

Capture or test each applicable module at the standard desktop viewport and one
narrow viewport:

- Backup: absent, detected/unlinked, initializing, linked, offline and failed;
- Multi-pipeline backup: both healthy, archive-only, mirror-only and wrong-profile;
- Schedules: disabled, edited, saving, saved/pending, applied, revision conflict,
  agent offline, run active/retrying, overdue and restored pending verification;
- Gryphon: unavailable, ready/unlinked, no paired adapter, function linked,
  user bound and service binding revoked;
- Wyverne: absent, unreachable, client enrolled, no allowed Adapter, selected,
  function-ready, incompatible and failed; another client's profile is inaccessible;
- Initialize: preflight, invalid/expired input, installing, reusing, enrolling,
  verifying, complete, partial failure, uncertain submission and reopened job;
- Logs: empty, initial loading, live newest-first entries, older-page loading,
  end of history, bounded export failure and revision-logging preference failure;
- Updates: checking, initially current, available, blocked, save pending,
  cancelled/failed save, acknowledgement required, install accepted,
  determinate/indeterminate progress, applying/reconnecting, completed with
  recheck, rejected, rolled back and rollback failed;
- keyboard order, visible focus, focus trap/restore, reduced motion and status
  text independent of color;
- long versions, long adapter usernames and sanitized multi-line failures without
  overlap, clipping or horizontal scroll;
- visual comparison against the applicable linked card states for palette, borders,
  typography, spacing, row height, action width and responsive composition.

Application and shared-component updates additionally pass the full
[universal update acceptance matrix](./PART_06_UNIFIED_ACCEPTANCE_CHECKLIST.md#391-universal-update-workflow-acceptance).
The first-transition bridge in Part 05 section 35 is documented separately from
the ordinary browser workflow and does not replace the required UI controls.

## 8. Initialize Overlay Workflow

Neptune uses the
[shared Initialize overlay](./PART_01_INTERFACE_AND_INTERACTION_UNIFICATION.md#109-service-initialization-overlays).
Gryphon uses its paired-adapter selection overlay. Gryphon and Wyvern keep
their existing cards, while installation and registration/enrollment are
controlled through `sudo updater tui`.
Installation, client enrollment, functional binding and end-user authorization
are separate operations with separate outcomes.

### 8.1 Preflight And Component Fields

Opening an applicable initialization dialog performs read-only discovery. Show
the owning service,
declared component/profile, observed installed version and whether the
operation will install a missing instance or reuse an existing instance.
Do not infer absence from a timeout, denied request or disconnected socket.
Missing permission, unsupported lifecycle capability and an unavailable
installer block submission with a specific explanation.

| Component role | Allowed initialization input | Verified initialization result |
| --- | --- | --- |
| Backup agent | Empty masked setup code; read-only owner and declared archive/mirror/folder profile | Existing or newly installed daemon is healthy, this service is enrolled, and every required pipeline has the correct scoped source/destination and credentials |
| Messaging gateway | No service Settings initialization input; direct the root operator to `sudo updater tui` | TUI installs/reuses the shared gateway and enrolls the service client; paired adapter selection remains a separate Settings action |
| LLM gateway | No service Settings initialization input; direct the root operator to `sudo updater tui` | TUI installs/reuses the shared gateway and enrolls the client; the service selects an allowed Adapter separately |

No form accepts arbitrary installation commands, package URLs, another
service's client identity, shared administrative credentials or unregistered
destinations. Addresses and management links come from the control plane.
Backup setup-code format/expiry is defined in Part 09; an invalid, expired,
already consumed or wrong-profile code is not a generic network failure.
Other forms do not invent the same code requirement.

Provider keys and adapter tokens stay in their privileged gateway management
boundary. A consumer card can display safe aliases, usernames, permitted
Adapters and its own binding status. It cannot retrieve credentials or list
other consumers' bindings. Empty fields and retries never erase existing
shared credentials. Clear transient enrollment secrets after accepted
submission or dismissal; retain only safe request/job metadata for recovery.

### 8.2 Execution, Reconnect And Idempotency

1. Explicit Initialize submits one authenticated, CSRF-protected where
   applicable, typed action. The backend derives owner/component scope and
   validates enrollment and lifecycle permissions independently of the UI.
2. Use a stable operation ID. Concurrent cards/tabs attaching the same service
   observe the same in-flight operation or receive a truthful conflict.
   Concurrent consumers reuse one host instance; enrollment of one cannot
   overwrite another's configuration.
3. After acceptance, show durable Job, actual State, sanitized message and the
   measured/indeterminate progress defined for updates. Expose install/reuse,
   enrollment and verification phases only when the backend reports them.
   A phase counter or elapsed time is not a percentage of completed work.
4. Poll or subscribe through the authenticated owning backend with bounded
   retry/backoff. Ignore stale observations from an older request or target.
   A transport interruption shows reconnecting/stale; it is neither success
   nor a reason to submit a second operation.
5. Close ends observation in the overlay, not execution on the host. The card
   retains `View progress`. Reload and renewed authentication rediscover the
   existing scoped job. Resolve a lost submission acknowledgement by operation
   ID before offering a retry; never persist the secret input to achieve this.
6. Report success only after terminal job success and fresh verification of
   the declared result. If verification is unavailable, display that pending
   condition. Keep the result until Done/Close and refresh the originating card.

Initialization does not create an application update backup, enable a schedule,
start a remote backup, select an unrelated Gryphon or Wyvern adapter, bind an end user or
update an already installed component implicitly. An incompatible installed
version uses the separate verified component-update workflow before enrollment
can proceed. Actual installation uses the approved signed lifecycle path.

### 8.3 Failure And Scoped Repair

Show the failed phase, safe reason, completed prerequisites and next action.
Examples include expired enrollment, denied owner scope, incompatible component,
missing required pipeline and failed readiness verification. Never expose
tokens, raw request headers, environment contents or unbounded command output.

An installed daemon with failed enrollment remains installed. Retry reuses it.
A partially registered multi-pipeline profile offers `Repair Neptune pipelines`
and lists the missing/incorrect owner-scoped parts before explicit confirmation.
Preserve healthy pipelines, existing schedules, active transfers and every
other consumer. Destructive replacement or revocation requires its own
consequence statement and confirmation; a routine Initialize is not such consent.

Pre-submit Cancel/Escape leaves the host unchanged. After acceptance, label
the action Close; do not imply cancellation if the protocol cannot safely
cancel the work. Error correction may require a fresh one-time code, but not
uninstalling a healthy shared daemon. Ordinary initialize, observe, repair
and subsequent own-service binding flows must be available through the
consuming service's interface.

## 9. Wyverne Connection

The [Wyverne Connection state gallery](./PART_01_INTERFACE_AND_INTERACTION_UNIFICATION.md#1056-wyverne-connection)
defines the next card visual target and Adapter-choice overlay. `Wyverne
Connection` is the required display label. Laboratory and Mastermind keep
their current card design until a separate UI rollout; the gallery does not
claim that the new overlay has already shipped.
The existing Wyvern component/repository/protocol identifiers remain unchanged;
UI copy is not a migration of executable names, API routes or registry keys.

The target card shows gateway reachability, the selected Adapter's observed
state, its safe name and a function-link/change action; the overlay lists only
Adapters granted to this client. Use the geometry and state crops in Part 01
section 10.5.6. The images' statement that Gryphon owns LLM connections is a
copy error: Wyvern owns provider access. Names and providers are sample data.

| Observed state | Required action and feedback |
| --- | --- |
| Confirmed absent or this client unenrolled | In the current card, keep `Initialize` as TUI guidance; no service-side installation or enrollment |
| Previously known gateway unreachable | Last-known identity/binding plus last-seen time; Retry without resetting selection |
| Client enrolled, no permitted ready Adapter | Explain that Adapter administration happens in `sudo updater tui`; do not ask for a provider key |
| Ready Adapter available, function unlinked | `Link Wyverne function`; show only this client's allowed choices and intended function scope |
| Function linked | Active Adapter alias, own binding and readiness; explicit change and scoped unlink in the choice overlay |
| Selected Adapter failed or incompatible | Preserve selection, show the concrete failure and offer retry or an explicit allowed change |
| Existing component version group | Keep it until the separate card rollout; its check/update control explains the root TUI path and does not start a service-side request. The new state gallery omits this group |

Adapter creation, configuration, grants and shared release checks/updates take
place in `sudo updater tui` without choosing a service for release operations.
Laboratory and Mastermind Settings only select from allowed Adapters and bind
their own functions. Opening Settings or selecting an Adapter sends no provider
probe or separate release check. The target overlay's `Unlink all adapters`
removes this service's function bindings only, after confirmation; it does not
delete an Adapter or revoke another client. Its details pane shows no provider
keys, secret values or other clients' configuration.

Changing an Adapter or functional binding carries the observed revision and
stable operation ID. On conflict, refresh and ask for explicit review of the
proposed change; do not retry a stale write automatically. The same error
handling, keyboard behavior and distinct lifecycle/binding confirmation used
for Gryphon apply here. Unlinking this service removes its own function
binding, not the shared Adapter or another client's registration.

Function-ready requires an authoritative scoped readiness result for the
selected Adapter and declared capability. Reachability alone does not prove
the model path works. A functional probe uses an approved bounded synthetic
request, no user's private content, and explicit invocation if it can incur
provider usage; opening the card never silently sends model requests.
Show probe failure honestly and keep transport, authorization, Adapter and
function readiness as distinct observations.
