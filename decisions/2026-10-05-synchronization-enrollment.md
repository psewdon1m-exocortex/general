# Operator decision — synchronization enrollment, 2026-10-05

The operator approved implementation and testing of the six enrollment changes
discussed in the Saturn task. This decision updates the corresponding workflows
in Parts 03, 09 and 10; older project guides describing mandatory paired Volt /
Mastermind channels, a manually entered recovery origin or a passphrase for each
new host archive are superseded for these operations.

1. Host recovery identity names derive from component and Server ID.
2. Updater resolves Saturn through its own authenticated Kernel Register; the
   last resolved origin and an explicit operator override support recovery.
3. New host archives use a generated persistent recovery key, with an explicit
   private-file export for off-host recovery. Legacy passphrase archives remain
   readable. Keys never enter ordinary API responses, Register or job logs.
4. Volt / Mastermind enrollment permits archive, mirror or both. Only selected
   capabilities can execute; service-owned scheduling remains authoritative.
5. Saturn creates the Windows connection folder and returns its assignment on
   enrollment and heartbeat. Windows no longer asks for a destination folder.
6. Windows authorization binds to a stable folder resource ID, validates both
   MOVE / COPY endpoints and protects the assigned root. Existing unbound
   credentials fail closed until a new isolated enrollment is established.

Acceptance includes real PostgreSQL migration round trips, cross-device DAV
denials and permitted nested mutations, archive-only / mirror-only / paired
policy and enrollment tests, generated-key and legacy restoration, Kernel
resolution and fallback tests, UI accessibility and responsive checks.

Rollback preserves stored folders and archives. Downgrades must revoke isolated
Windows credentials rather than expanding their scopes to all of sync. Existing
paired service registrations retain their capabilities. Deployment is separate
from this authorized implementation; no live enrollment, restore or key export
is performed by the verification fixtures.
