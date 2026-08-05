# ADR-0001: Use Restic for Personal Cloud Backups

- **Status:** Accepted
- **Date:** 2026-08-05
- **Owner:** Damien
- **Scope:** Personal cloud services hosted on the Mac mini

## Context

Self-hosted service data is stored under:

```text
$HOME/personal-cloud-data/
├── forgejo/
├── open-webui/
└── uptime-kuma/
```

Backups are stored on the attached Passport drive:

```text
/Volumes/passport/backups/
```

The backup system must provide:

- Recoverable point-in-time versions
- Protection from accidental deletion and overwritten data
- Efficient storage of repeated snapshots
- Encryption at rest
- Retention management
- Repository integrity checks
- A clear restore procedure

A plain `rsync` mirror copies current state but also propagates deletion, corruption, and operator error. Snapshot-style backups can be built around `rsync`, but that would require custom handling for retention, hard links, partial runs, locking, verification, and encryption.

## Decision

Use **restic** to create encrypted, deduplicated snapshots of:

```text
$HOME/personal-cloud-data
```

Store the repository at:

```text
/Volumes/passport/backups/personal-cloud.restic
```

Use the following retention policy:

```text
Daily:   7
Weekly:  5
Monthly: 12
Yearly:  3
```

Before each backup:

1. Verify the Passport drive is mounted.
2. Detect currently running Compose services.
3. Stop only those services.
4. Create the restic snapshot.
5. Restart the previously running services.
6. Apply retention and prune unreferenced data.
7. Run a repository integrity check.

Run the backup nightly using a macOS `launchd` user agent.

Run an additional manual backup before:

- Service upgrades
- Database migrations
- Major configuration changes
- Storage migrations

## Rationale

Restic is preferred over plain `rsync` because it provides a backup data model rather than only file synchronisation.

Restic provides:

- Point-in-time snapshots
- Deduplication
- Encryption
- Retention policies
- Integrity checking
- Recovery of deleted or overwritten files
- Safe interruption of backup operations
- Restore by snapshot, path, host, or tag

`rsync` remains suitable for:

- One-off migrations
- Readable mirrors
- Replicating an existing restic repository
- Copying large non-versioned media collections

## Scheduler Decision

Use `launchd` rather than cron.

Reasons:

- Native macOS scheduler and service manager
- Explicit stdout and stderr log paths
- Job state available through `launchctl`
- Immediate test execution through `launchctl kickstart`
- More predictable macOS background execution
- No dependency on interactive fish shell configuration

The backup script remains self-contained and fish-compatible. It defines all required paths and environment variables explicitly.

## Consequences

### Positive

- Multiple historical restore points
- Efficient storage consumption
- Encrypted backup contents
- One backup boundary for all current services
- Straightforward recovery and migration
- Future off-site replication can copy the restic repository

### Negative

- Backup contents are not directly browsable without restic
- Loss of the restic repository password makes backups unrecoverable
- Services are briefly unavailable during each backup
- The attached Passport drive does not protect against theft, fire, or simultaneous loss of the Mac mini and drive
- A user LaunchAgent runs only while the user is logged in

## Operational Requirements

- Store a second copy of the restic password in a password manager.
- Fail closed when `/Volumes/passport` is unavailable.
- Never create a fallback backup directory on the internal disk.
- Return a non-zero exit code on failure.
- Restart services even when backup creation fails.
- Restore into a temporary directory before replacing production data.
- Test a restore at least quarterly.
- Review backup logs regularly.
- Add off-site replication as a later phase.
- Add external failure notification when an appropriate notification service is available.

## Revisit When

Reassess this decision when:

- A service moves from SQLite to an external database.
- Backup downtime becomes unacceptable.
- The Passport drive is replaced by a NAS.
- Off-site object storage is introduced.
- Data size materially increases backup duration.
- Services require application-native backup commands.
- The Mac mini no longer remains logged into the owning user session.
