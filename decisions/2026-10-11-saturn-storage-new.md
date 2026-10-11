# Saturn Storage new: explicitly requested private design variant

Date: 2026-10-11. Decision owner: the operator, through the direct request in
the implementation chat. Status: authorized for local implementation and
testing; publication/deployment is not part of this request.

## Request and scope

The operator asked to use the current public shared-link design for an optional
Saturn **Storage new** tab, explicitly preserving the original Storage design
and functionality. The request names Part 01 as the design reference.

Part 01 §11.1 requires a direct operator request before applying this private
case elsewhere. That request provides authorization here. This decision records
the named exception; it does not extend the permission to other screens.

## Applicable guide and difference

Part 01 §§1–10 define the dark operational interface; §11 records the current
shared-link case with its warm light palette, body typography, muted type tints,
card composition and shadows. Those are material visual differences. The
authorized scope is the authenticated owner Storage new workspace only. Shared
controls, backend access policy, confirmations and the original Storage remain
on their existing contracts. Global theme tokens and public shared links are
not replaced.

## Implementation and compatibility

The optional `/storage-new` route reuses the existing file controller/state and
handlers. Cards expose selection, current location, full names, modification
times, sizes, sharing status and downloads. Images use the existing authenticated
preview endpoint with a type-icon fallback. No new public preview capability or
credential is created. Root upload/move/trash protections remain in force.

Seven-item saved navigation lists gain the tab after Storage, preserving order;
eight-item lists retain their chosen position. The API supports both complete
generations. No schema/data migration is required. Web and API update together
for the new preference shape.

## Acceptance and rollback

The original `/files` remains independently available. Tests must cover both
presentations, navigation compatibility, representative file operations,
auth loss, loading/errors, empty/single/many entries, long names, keyboard,
drag/drop, downloads, responsive layouts and configured accent contrast.
Browser fixtures are explicitly synthetic and do not prove production persistence.
Rollback of the user-facing experiment is immediate by opening Storage; removal
of the new route/tab requires a separate operator decision. Release publication
remains deferred.

Service details and verification commands:
[Storage new contract](../../saturn/docs/implementation/STORAGE_NEW.md).
