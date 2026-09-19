# Part 09. Service Agents: Deployment, Initialization And Lifecycle

This document is the Exocortex-wide source of truth for deploying, enrolling,
operating and updating the Linux service agents **Neptune**, **Gryphon** and
**Wyvern**, including their consuming services. It supplements the reusable deployment,
backup, update and security Parts with the concrete Exocortex topology.

This Part supersedes every former direct-connection compatibility note. Gryphon
is the single Telegram gateway: consuming services MUST NOT embed their own
Telegram polling or webhook runtime and MUST NOT store a bot token as an
application setting. Updater, Neptune, Gryphon and Wyvern ownership, trust and operator
flows are defined only in Parts 09 and 10.

This Part is subordinate only to the [Part 00 documentation authority](./PART_00_SYSTEM_UNIFICATION_SPECIFICATION.md) and takes precedence over conflicting project-local documentation.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** are
normative.

## 1. Scope And Ownership

### Service-owned schedule decision (2026-09-19)

The operator approved moving automatic schedule management from the central
panel into each owning service. Settings → Backup is now the sole operator
surface for that service's enable/interval controls and explicit backup runs.
This supersedes the former centralized-only schedule rule. Central storage
retains identity, quota, revocation and fleet observation responsibilities.
The handover contract in section 5 preserves existing policies and pending
work; documenting it does not claim that deployed software already implements it.

### Wyvern integration decision (2026-09-19)

The approved Wyvern extension is a shared LLM gateway per host. A consuming
service installer MUST ensure/reuse the host Updater, then install/reuse one
compatible Wyvern instance through a typed component operation and enroll only
its own client. Standalone Wyvern installation uses the same Updater-first
boundary. Uninstalling a consumer MUST NOT remove the shared gateway or another
consumer's binding. Default data-plane communication uses a local Unix socket;
a dedicated domain is not required. Cross-host HTTPS is an explicit placement
choice, not an automatic outage fallback.

Wyvern opens the outgoing provider request using its configured Adapter
credential. Its internal client identity is independently authenticated. The
Adapter is the complete configured API profile; Driver names the protocol
implementation. Provider keys live in Volt through Kernel, with scoped machine
grants that prevent ordinary consumers and the legacy shared token from
resolving Wyvern credentials or aliases to them. Reload MUST account for Volt
value revisions even when the Register reference revision does not change.

Implementation status and acceptance are tracked in the Wyvern repository.
This extension does not declare an unpublished image installable and does not
waive the existing signed release, update backup/receipt or recovery contracts.
The fixed Wyvern lifecycle uses externally authoritative Kernel/Volt state.
Updater MUST verify the signed image/API/config/capability contract, serialize
operations, drain accepted requests and journal activation/rollback/repair.
Runtime rollback restores only the prior deployment and admission state, never
global Register or Volt. This exception is specific to Wyvern: no caller may
supply a general backup exemption. Encrypted host recovery includes its private
identities and media metadata; executable deployment files are re-provisioned
from a trusted signed installer. Qualification is recorded separately from
production deployment.

| Component | Host ownership | Primary responsibility | Consuming modules |
| --- | --- | --- | --- |
| Updater | One root-owned daemon per Linux host | Performs allow-listed privileged installation, enrollment and verified update jobs | Kernel, Volt, Chronos, Saturn, Neptune, Gryphon |
| Neptune Linux | One unprivileged `neptuned` daemon per Linux host | Exports module-owned recovery archives and optional dedicated mirrors to Saturn | Kernel, Volt, Chronos, Saturn; future approved modules |
| Gryphon Linux | One gateway service per deployment | Owns Telegram bot tokens, webhooks, update deduplication, callbacks and service-scoped bindings | Chronos and Saturn |
| Wyvern | One shared unprivileged gateway per Linux host, or explicit remote HTTPS instance | Owns provider Adapters, keys, request transport and media handles; consumer domains retain prompts, jobs and commits | Mastermind, Laboratory |
| Saturn | Central control plane and storage gateway | Issues single-use setup codes, enforces storage identity/quotas, receives archives/mirrors and may relay service-owned policy/commands; no central schedule editor | All Neptune deployments |

An application web process MUST NOT receive `sudo`, a Docker socket, the Gryphon
administrative socket or arbitrary command execution. UI actions call the
application backend; the backend calls the local Updater through its Unix socket
using the token for that registered service. Updater independently selects the
approved installer, repository, artifact and service profile.

## 2. Trust And Communication Topology

The host operator may also use `sudo updater tui`. This console is bundled
with Updater and controls the current host's Updater, Neptune, Gryphon and
Wyvern. It uses a separate root-owned mode-`0600`
Unix socket at `/run/exocortex-admin/updater.sock`. That directory is not mounted
into consuming service containers. Linux peer credentials must additionally
identify UID 0. Shared Wyvern updates and management require root operator
dispatch; service tokens can install/reuse Wyvern and link only their own
registered head. The service socket and its per-head tokens retain their existing
scope; the operator facade selects a registered head, validates a typed action
and delegates using the daemon-owned head credential. It must not forward an
arbitrary path, shell command or executable supplied by the terminal.

The systemd runtime-directory declaration preserves both socket directories.
The console is a separate process from the daemon, holds no persistent secrets
or authoritative application state, and reconnects to durable job metadata
after a transport failure or daemon self-update. Read-only local diagnostics
remain available when the operator API is down. Enrollment codes and bot tokens
are transient masked input. Schedule ownership, bot/service/user trust
decisions, signed releases and rollback rules remain as specified below.

```text
browser
  -> authenticated module API
     -> /run/exocortex/updater.sock + per-head token
        -> allow-listed Neptune/Gryphon install, enroll or update job

module backup builder
  <- loopback/private export request from neptuned
neptuned
  -> Kernel Register (non-secret coordinates)
  -> Saturn HTTPS check-in, archive ingest and optional WebDAV mirror

Telegram
  -> public TLS webhook -> Gryphon
Gryphon
  -> authenticated Chronos/Saturn command adapter
Chronos/Saturn
  -> /run/gryphon/client.sock for status, linking and outbound notifications
```

The Updater socket is `/run/exocortex/updater.sock`. Each registered head has a
separate control token and exact service/profile match. Neptune uses
`/run/neptune/neptuned.sock` for local authenticated control. Gryphon exposes
`/run/gryphon/client.sock` to service containers and keeps
`/run/gryphon-admin/admin.sock` for root/local administration only.

Saturn never opens an inbound port on a Neptune host. `neptuned` polls Saturn
over outbound HTTPS, reports observed state and consumes monotonically revised
desired state and queued commands. A temporary Saturn outage does not erase the
last applied schedule.

## 3. Initial Deployment

### 3.1 Main modules and Updater

The normal module installer (`kernel-install`, `volt-install`,
`chronos-install` or `saturn-install`) installs or reuses the single host
Updater and registers its own head with a dedicated token. Initial deployment
MUST finish health verification before the UI offers privileged component
actions.

The active systemd unit MUST preserve the Updater runtime directory across
daemon restarts so existing container bind mounts continue to see the socket.
At startup the Updater also compares the host socket-directory identity with
the directory visible in each running registered container. When a stale bind
mount is detected, it recreates only the affected service container with its
existing Compose configuration. Healthy mounts are untouched; stopped or
uncreated containers are not started implicitly. An installation created by an
older unit may require one explicit container recreation before this protection
is active.

### 3.2 Neptune Linux

There is exactly one Neptune Linux daemon per host. It runs as a dedicated
unprivileged user. Its registry and durable journal live under
`/var/lib/neptune`; secret references and per-project credentials live under
`/etc/neptune` with restrictive ownership. Installation MUST verify the release
manifest and archive checksum and MUST NOT create a second daemon for another
module on the same host.

Neptune releases use `neptune-vMAJOR.MINOR.PATCH` and publish platform- and
architecture-specific archives and manifests for supported targets. The
installer selects the host architecture, verifies the manifest-declared bytes,
installs the native service and proves `/run/neptune/neptuned.sock` health.

Each module exposes an authenticated local export endpoint backed by the same
logical archive builder used by manual download and restore. Neptune treats the
archive as exact bytes: it does not unpack, rename, re-encrypt or recompress a
recovery ZIP.

The privileged CLI remains the installation, repair and emergency fallback:

```text
sudo kernel-install backup
sudo volt-install backup
sudo chronos-install backup
sudo saturn-install backup
```

If Neptune is absent, Settings MUST offer Initialize through the authenticated
local Updater. Updater installs a signed release, verifies health and enrolls the
requesting head. If a healthy instance already exists it is reused without a
download, restart or duplicate installation. A setup code is never persisted by
the web service. The operator sees a durable terminal job result.

### 3.3 Gryphon Linux

Gryphon is installed once and owns every Telegram bot token. Saturn Settings
provides typed installation and bot-registration actions through Updater. The
web process transiently forwards the operator-provided bot token and clears the
input; it never persists it or mounts the Gryphon admin socket. The equivalent
privileged recovery CLI remains:

```text
gryphon bot connect <bot-alias>
gryphon bot list
```

The connect operation reads the bot token without echoing it, verifies the bot
identity with Telegram, registers the webhook and stores a protected Gryphon
copy. Connected domain services never retain the bot token. Their installers
provision only `/etc/gryphon/clients/chronos.token` or
`/etc/gryphon/clients/saturn.token` and mount the service client socket.

Native releases use `gryphon-vMAJOR.MINOR.PATCH` with an
`exocortex.gryphon.release.v1` manifest. Initial installation extracts a
verified archive, runs `packaging/linux/install.sh` as root, configures
the protected Kernel bootstrap connection in `/etc/gryphon/gryphon.env` and starts
`gryphon.service`. The listener on port `18380` accepts only Telegram webhooks
and MUST be published behind HTTPS at that public origin. Persistent database
and protected secret copies under `GRYPHON_DATA_DIR` are operated and backed up
together.

## 4. Neptune Initialization

### 4.1 Operator workflow

1. In Saturn → **Synchronization**, create the required Linux pipeline identity
   and a single-use setup code. The code expires after 15 minutes.
2. On the target module host, open Settings → **Backup**.
3. When the panel reports `Detected · not linked`, select **Initialize**,
   then paste the code and confirm in the **Initialize Neptune** overlay.
4. The backend forwards only the setup code and its own registered service
   identity to Updater. The setup code MUST NOT be persisted by the module.
5. Updater validates the exact module profile, exchanges the code with Saturn,
   writes the scoped credentials, registers or repairs the project in Neptune
   and restarts only the required local service when necessary.
6. The UI polls the persisted initialization job through a service-authenticated
   status endpoint until a terminal state. A successful request is not a
   successful initialization.
7. After `COMPLETED`, the module reloads Neptune status and confirms the expected
   pipeline set. `FAILED` shows the sanitized terminal reason and a retry path.

The service entry is a card-local `Initialize` action, opening the common
[Part 01 section 10.9 overlay](./PART_01_INTERFACE_AND_INTERACTION_UNIFICATION.md#109-service-initialization-overlays).
Code creation remains identity/enrollment management and is not schedule
editing. Initialization reuses a compatible existing daemon and enrolls only
the requested service. Completion unlocks that service's schedule controls;
it does not enable a new schedule or run a backup implicitly. Transport
unavailability is not proof that installation is absent.

The initialization state vocabulary is `REQUESTED`, `INSTALLING`, `ENROLLING`,
`COMPLETED` and `FAILED`. Temporary loss of the module API while its container
is recreated is an expected reconnect state, not an automatic failure.

### 4.2 Per-module profiles

| Module | Required Neptune registration | Successful postcondition |
| --- | --- | --- |
| Kernel | Recovery archive only, namespace `kernel` | Kernel archive project exists and is linked |
| Chronos | Recovery archive only, namespace `chronos` | Chronos archive project exists and is linked |
| Saturn | Recovery archive only, namespace `saturn` | Saturn archive project exists and is linked |
| Volt | Recovery archive plus dedicated `personal.volt` mirror | Both independent pipelines exist with the exact Volt profile |

Volt is deliberately not an archive-only special case. A single Volt setup code
provisions two credentials and two workers:

```text
recovery ZIP:  namespace=volt -> immutable Saturn backup archive
mirror:        mirrorRoot=volt, mode=single-file,
               targetFilename=personal.volt -> Saturn WebDAV /volt
```

The workers have independent schedules, states, retries and credentials and run
concurrently. The UI MUST wait for the Updater job to finish, reload Neptune
state and verify both rows. If one pipeline is absent or has the wrong profile,
Volt reports partial configuration and offers **Repair Neptune pipelines**;
it MUST NOT claim that backup is ready.

## 5. Ongoing Neptune Interaction

### 5.1 Policy Ownership And Control

The owning application's Settings → Backup authors its schedule and explicit
run requests. Remove schedule enable/interval editors and service backup-run
buttons from the central panel. Central inventory may display desired/applied
state, last success and health, and manage identities, quotas, enrollment and
revocation. Those storage security controls may block execution but cannot
silently rewrite the service's schedule.

One versioned policy record is authoritative for each service/deployment and
pipeline. Its physical persistence may remain in an existing control-plane
backend, but all operator writes originate from the owning service's
authenticated workflow and are restricted to that scope. A deployment profile
names this backend and its protocol; the application and agent must not each
maintain an independently writable schedule. The agent keeps applied execution
state, not an alternative operator policy.

The browser calls only its authenticated service backend. The backend resolves
required internal coordinates through the registry and uses a typed scoped
schedule/run operation via the authorized local agent or existing control-plane
transport. It cannot forward arbitrary command, path, identity or URL fields.
Server-derived service/pipeline identity, authorization and expected policy
revision prevent access to another service's schedule.

A policy update contains enabled state and interval hours plus its expected
revision; validation and commit are atomic. Return committed desired revision,
agent-applied revision, next due, last success and any pending/error state.
Use an idempotency key for mutations and reject stale concurrent revisions
rather than let the last browser overwrite an unrelated change. A saved policy
is shown as `Pending application` until the agent acknowledges it.

### 5.2 Scheduling And Execution Semantics

The archive and mirror paths remain separate:

- the module owns logical recovery format and restore semantics;
- Neptune spools exact archive bytes, records size and SHA-256, and uploads with
  a stable idempotency key;
- resumable upload continues from Saturn's accepted offset after restart;
- archive completion requires a Saturn receipt before spool deletion;
- disabling a schedule blocks new work but does not corrupt active work;
- a mirror compares content identity and updates only its dedicated root;
- producer credentials cannot write WebDAV mirrors, and mirror/device
  credentials cannot create recovery archives.

A genuinely new profile starts disabled with a 24-hour interval; the minimum
supported interval is one whole hour. Enforce finite integer hours and any
declared protocol maximum on client and server. Existing profiles keep their
actual enabled/interval values instead of receiving new-install defaults.

Enabling a schedule or committing a changed interval sets its next due relative
to the acknowledged policy activation time plus that interval. It does not
implicitly run a backup. After a scheduled run completes, the next due follows
the pipeline's documented fixed-delay policy; UTC timestamps avoid local DST
changes shifting intervals. Surface the computed next due instead of asking
the browser to predict it.

`Back up now` is a separate idempotent command and may be used while automatic
backup is disabled. It does not change enabled/interval or, by default, the
existing scheduled next due. If an applicable project run is already active,
return/observe that run or reject as busy; do not overlap exports or enqueue
duplicates. Manual and scheduled due work are coalesced according to the
single-project execution lock and a documented policy.

Disabling prevents new scheduled runs, while an accepted transfer finishes or
recovers to a safe terminal result under its retry policy. Network retries
preserve exact archive bytes, checksum, receipt and upload offset; a saved
schedule alone is never proof that a backup reached storage.

The agent executes without an open browser or live service Settings page.
Persist applied policy, current run and recovery state so a restart does not
erase the schedule or replay a backlog of missed intervals. At most one
coalesced overdue run per pipeline is considered on recovery, subject to
permission, backpressure and the existing project/host concurrency limits.
The UI distinguishes desired, applied, due, overdue, running, retrying and last
remote commitment; a remote outage leaves the last applied policy intact.
Archive and dedicated-mirror schedules remain independent in their owner card.

### 5.3 Handover From Central Schedule Editing

1. Inventory each current service/deployment/pipeline policy, revision,
   enabled/interval, next due and active/queued run identity. Preserve this
   state in the supported backup/recovery boundary before migration.
2. Import or reuse the existing authoritative policy without new-install
   defaults. Each service initially reads the exact prior values.
3. Switch authoring for that scope atomically with an ownership revision/fence.
   Disable the former central editor and its write authorization before the
   new service workflow becomes the sole writer. If the record stays central,
   restrict that same record's write path instead of creating a second copy.
4. Reject stale central schedule commands/revisions after cutover. Dedupe or
   explicitly resolve old queued run commands; never replay them as new runs.
   An active archive/mirror transfer continues with its existing identity.
5. Verify the desired and agent-applied policy, next due, one manual run and
   one automatic run from the service interface. Test an agent restart and
   central transport outage without resetting settings or duplicating work.
6. Rollback of the migration restores one writer and the preserved policy/
   queue boundary; it cannot leave both central and service editing active.

Service backup/restore includes schedule intent and reconciliation metadata.
These requirements describe the approved target contract, not an assertion
that an older deployment's UI or APIs already support it.

## 6. Gryphon Initialization And Binding

Gryphon has three deliberately separate layers:

1. **Bot registration:** a privileged operator gives a Telegram bot token to
   Gryphon with `gryphon bot connect`; only Gryphon stores it.
2. **Service function connection:** the consuming application selects a ready
   bot with **Link <service> function** inside **Gryphon Connection**. The service
   receives only its scoped client credential.
3. **Telegram user binding:** after the function is connected, **Link Telegram
   account** creates a service-scoped one-time `/link CODE` challenge. The operator
   sends it in a private chat with the selected bot. Linking one service does not
   authorize the same Telegram identity for another service.

The equivalent emergency CLI operations are:

```text
gryphon link issue chronos
gryphon link issue saturn
```

The UI MUST show the selected bot username, code expiry and a copy action. Codes
are single-use, short-lived and never accepted from group chats. Unlinking a
service function or Telegram user binding is explicit, audited and immediately
revokes the corresponding scope without deleting an otherwise shared bot.

Gryphon invokes only authenticated, allow-listed command adapters. Chronos and
Saturn do not poll Telegram, register webhooks or deduplicate Telegram updates.
Outbound reminders, summaries and responses go back through the service-scoped
Gryphon client socket.

Before these bindings, a missing or unenrolled shared gateway is ensured/reused
by the card's `Initialize` workflow through the typed local helper. Installing
a daemon, linking a service function and authorizing a Telegram user are
different actions. The old `Bot connection` card title is retired in favor
of `Gryphon Connection`; this UI rename does not rename socket/API identifiers.

## 7. Updates

The universal application/shared-component contract is
[Part 05 section 34](./PART_05_CI_RELEASES_AND_LOCAL_UPDATES.md#34-operator-update-ui).
The concrete protocol bindings below do not limit its applicability to a fixed
list of applications. Every consuming interface follows the same six visual
references, exact-target confirmation and durable job lifecycle.

### 7.1 Main applications

Every application discovers and applies its own releases through the local
update-helper contract in [Part 05](./PART_05_CI_RELEASES_AND_LOCAL_UPDATES.md).
An exact standard full ZIP saved on the operator's computer and its scoped
receipt are verified before application mutation; health checks and rollback
determine the terminal result. Shared-component updates omit this application
backup gate, not release verification or state preservation.

### 7.2 Neptune and Gryphon

Component checks and installs use typed Updater component endpoints for
`neptune-linux` and `gryphon-linux`. Updater resolves the approved repository
from Kernel Register, selects an exact compatible release, verifies the
manifest and checksum, stages the artifact, performs the component-specific
atomic replacement, restarts the component and verifies its Unix-socket health.
The web client cannot supply a URL, executable path or command.

| Operation | Local Updater route |
| --- | --- |
| Initialize/repair Neptune profile | `POST /v1/components/neptune-linux/initialize` |
| Read Neptune initialization result | `GET /v1/components/neptune-linux/initializations/{id}` with the authenticated head identity |
| Check/update Neptune Linux | `POST /v1/components/neptune-linux/check` / `POST /v1/components/neptune-linux/update` |
| Check/update Gryphon Linux | `POST /v1/components/gryphon-linux/check` / `POST /v1/components/gryphon-linux/update` |

These are daemon-local routes. Browser-facing module endpoints proxy only the
fields required by their UI and never expose the Updater token.

Shared-agent release policy and fleet observation may remain central, while
each local module panel exposes its approved check/install workflow. This does
not return backup schedule/run authoring to the central panel. Each consumed agent's version and update controls
appear in the corresponding Settings card/group of every consuming application;
the component's topology determines its impact and authorization.

Updater self-update is a separate binary lifecycle. Updating the executable
does not by itself replace arbitrary packaging or systemd definitions. Unit or
package changes require the installer/package repair path unless the signed
Updater release contract explicitly carries and applies them.

## 8. Failure, Recovery And Audit

- Every accepted initialization, repair, link, unlink, update and remote-run
  request produces a durable identifier and an audit event without secrets.
- UI success is based on a terminal job plus refreshed observed state, never on
  HTTP acceptance alone.
- A setup code, service token, producer token, bot token or `/link` code is
  never logged, returned in ordinary status, included in backup or stored in
  browser persistence.
- Timeouts preserve the job identifier and offer status reload; they do not
  encourage submitting duplicate jobs blindly.
- Retrying the same idempotent request returns or resumes the existing job.
- Revoked identities stop new work. Active archive/mirror operations terminate
  at a safe boundary and retain enough state for diagnosis.
- Manual CLI repair remains documented for an unavailable UI, absent agent,
  damaged packaging or an older deployment that predates the UI contract.

## 9. Required Evidence

An implementation is complete only when evidence covers:

- absent, detected/unlinked, initializing, linked, partial, failed, offline and
  update-available states;
- exact service/profile enforcement and cross-service setup-code rejection;
- final-state polling through the authenticated module proxy;
- module restart/reconnect during initialization;
- Volt archive and mirror workers running independently and concurrently;
- Saturn desired-state retention across temporary loss of connectivity;
- Gryphon bot registration, service linking and Telegram user binding as three
  separate authorization decisions;
- token redaction and inability of a service container to access Gryphon admin
  operations or another Updater/Neptune profile;
- verified component update, health failure and rollback/repair behavior.
