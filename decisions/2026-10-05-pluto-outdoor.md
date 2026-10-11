# Pluto outdoor agent — approved 2026-10-05

The operator approved a Linux-only standalone agent in `pluto/` for backing up ordinary files and folders outside Exocortex. Pluto explicitly has no Updater or Neptune runtime/install dependency. This exception supersedes the default shared-agent dependency and lifecycle requirements in Parts 04, 05, 09 and 13 for Pluto only.

Pluto owns its daemon, root-operated TUI, bootstrap and manually requested signed self-updates. One installation has one backup profile/pipeline. Configuration, source paths, interval and the separate copying switch are managed in TUI. Saturn manages pipeline enrollment, observation and confirmed revoke.

Kernel issues a separately revocable **Outdoor service token** per external installation. It authorizes only `POST /api/v1/register/resolve` for `services.saturn.sni`, `services.saturn.port` and `repositories.pluto.url`, and denies secret values and all internal control APIs. It is not an unrestricted Kernel service token. Saturn's scoped data credential is obtained independently by redeeming a one-time Pluto setup code.

Saturn's **Pluto** pipeline type requires only a connection name, without a Host name. Its folder is `# sync outdoor connections/<connection name>`. Each pipeline gets a stable isolated folder identity; file paths retain their absolute source hierarchy without the leading `/`. Data is ordinary files, not service recovery ZIPs. Pluto has read/write capability within its folder and cannot move or delete data. Source deletion never propagates to storage. The public Pluto API keeps its stable logical `pluto/<connection>` alias; the physical root name is never part of the credential.

Keep at most ten active versions per file, including the current file. Expire metadata and physically remove older version archives. Retry incomplete retention on metadata access. Scope isolation, revocation and checksums apply to every transfer. Files and their retained versions remain available after pipeline revoke.

Pluto appears as a separate category within **Neptune fleet**, alongside existing Linux and Windows agents. This is a common observation surface and does not add a dependency on Neptune's daemon or control socket.

Initial constraints: Linux amd64/arm64, ordinary files and folders, small configuration and completed database-backup files, a 256 MiB per-file transfer bound, no symbolic links or special files. Unix permission/ownership restoration and multi-profile agents are outside this initial version.

## 2026-10-11: Source management and file links

The operator requested arrow-key TUI navigation, additive source management that preserves already configured paths, and correction of empty backups from `/etc/letsencrypt/live`. This request authorizes a scoped amendment to the initial prohibition on symbolic links above: links to ordinary files are dereferenced, and their bytes are stored at the selected link's original path. The target may be outside the selected source directory, as with Let's Encrypt's `live` to `archive` layout, but remains subject to the dedicated `pluto` account's read/traversal permissions and the existing file-size and snapshot-integrity limits. Directory links and special files remain prohibited; missing, cyclic or inaccessible targets cause visible partial failures. Restores produce ordinary files, without symlink metadata. The agent gains no new host privileges.

The TUI adds new sources without replacing existing ones, deduplicates additions and offers separate removal of individual sources, including inaccessible paths. Changes persist atomically under the service lock and are rejected during an active copy. Removing a source leaves remote backups intact; removing the final source disables copying. The existing full-list `sources` control action remains available for compatible local clients; the TUI uses separate `sources-add` and `sources-remove` actions.

Verification covers navigation and return from nested menus, terminal restoration on normal exit and interruption, persistence and failed source mutations, Let's Encrypt-style relative file links and renewal, deduplication, partial failures, and rejection of directory/FIFO/cyclic/broken links. Existing agent update/rollback tests continue to apply because configuration format and bootstrap privileges are unchanged. Reverting the implementation restores the previous TUI and link prohibition while preserving the source list and previously copied remote bytes.
