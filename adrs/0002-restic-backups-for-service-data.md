# ADR: Restic Backups for Self-Hosted Service Data

* **Status:** Accepted
* **Date:** 2026-08-05
* **Owner:** Damien
* **Scope:** Mac mini personal cloud

## Context

Self-hosted services persist data under:

```text
~/personal-cloud-data/
├── forgejo/
├── open-webui/
└── uptime-kuma/
```

Backups must be stored on the externally attached Passport drive.

A plain `rsync` mirror would copy the current state but would also propagate deletion, corruption, or operator error. The requirement is to retain multiple recoverable point-in-time versions.

## Decision

Use **restic** to create encrypted, deduplicated snapshots of `~/personal-cloud-data` in a repository on the Passport drive.

Target repository:

```text
/Volumes/Passport/Backups/personal-cloud.restic
```

Before each snapshot:

1. Verify the Passport drive is mounted at the expected path.
2. Stop the relevant containers.
3. Back up the complete data directory.
4. Restart the containers.
5. Apply the retention policy.
6. Run a lightweight repository integrity check.

Initial retention policy:

```text
Daily:   7
Weekly:  5
Monthly: 12
Yearly:  3
```

Backups will run nightly through macOS `launchd`.

Additional backups should be run manually before:

* Service upgrades
* Database migrations
* Major configuration changes
* Storage migrations

## Rationale

Restic provides:

* Point-in-time snapshots
* Deduplication
* Encryption
* Retention policies
* Repository integrity checking
* Recovery of deleted or overwritten data
* Safe interruption and resumable operation

`rsync` remains appropriate for migrations, readable mirrors, or copying an existing backup repository to another location.

## Consequences

### Positive

* Multiple historical restore points
* Efficient storage use
* Backup contents encrypted at rest
* One backup boundary for all current services
* Straightforward migration to another machine
* Off-site replication can be added later

### Negative

* Files are not directly browsable without restic
* Loss of the repository password makes backups unrecoverable
* Containers are briefly unavailable during backup
* The attached Passport drive is not sufficient protection against theft, fire, or simultaneous hardware loss

## Operational Requirements

* Keep a second secure copy of the restic password.
* Backup jobs must fail if `/Volumes/Passport` is unavailable.
* Backup failures must return a non-zero exit code.
* Restores must first target a temporary directory.
* Perform a restore test at least quarterly.
* Add off-site replication as a later phase.
* Add external failure notifications when `ntfy` or equivalent is available.

## Revisit When

Reassess this decision when:

* A service moves from SQLite to an external database.
* Downtime during backup becomes unacceptable.
* Data size makes local backup duration problematic.
* The Passport drive is replaced by a NAS.
* Off-site storage is introduced.
* Per-service backup and restore procedures become necessary.
