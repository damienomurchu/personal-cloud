# Runbook: Back Up Personal Cloud Data with Restic

## Configuration

Assumed paths:

```bash
DATA_DIR="$HOME/personal-cloud-data"
COMPOSE_DIR="$HOME/personal-cloud"
PASSPORT_VOLUME="/Volumes/Passport"
RESTIC_REPOSITORY="$PASSPORT_VOLUME/Backups/personal-cloud.restic"
RESTIC_PASSWORD_FILE="$HOME/.config/restic/personal-cloud-password"
```

Update `COMPOSE_DIR` if the Compose file is stored elsewhere.

## 1. Install Restic

```bash
brew install restic
restic version
```

## 2. Create Repository Password

```bash
mkdir -p "$HOME/.config/restic"

openssl rand -base64 32 \
  > "$HOME/.config/restic/personal-cloud-password"

chmod 600 "$HOME/.config/restic/personal-cloud-password"
```

Store a second copy in a password manager.

## 3. Initialise Repository

```bash
export RESTIC_REPOSITORY="/Volumes/Passport/Backups/personal-cloud.restic"
export RESTIC_PASSWORD_FILE="$HOME/.config/restic/personal-cloud-password"

test -d "/Volumes/Passport" || {
  echo "Passport drive is not mounted"
  exit 1
}

mkdir -p "/Volumes/Passport/Backups"
restic init
```

Verify:

```bash
restic snapshots
```

## 4. Install Backup Script

Create:

```text
~/bin/backup-personal-cloud
```

Contents:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

readonly DATA_DIR="${HOME}/personal-cloud-data"
readonly COMPOSE_DIR="${HOME}/personal-cloud"
readonly PASSPORT_VOLUME="/Volumes/Passport"
readonly RESTIC_REPOSITORY="${PASSPORT_VOLUME}/Backups/personal-cloud.restic"
readonly RESTIC_PASSWORD_FILE="${HOME}/.config/restic/personal-cloud-password"

containers_stopped=false

log() {
    printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

restart_services() {
    if [[ "${containers_stopped}" == true ]]; then
        log "Restarting services"
        (
            cd "${COMPOSE_DIR}"
            docker compose start
        )
        containers_stopped=false
    fi
}

cleanup() {
    exit_code=$?
    restart_services

    if (( exit_code != 0 )); then
        log "Backup failed with exit code ${exit_code}"
    fi

    exit "${exit_code}"
}

trap cleanup EXIT INT TERM

[[ -d "${DATA_DIR}" ]] || {
    log "ERROR: Missing data directory: ${DATA_DIR}"
    exit 1
}

[[ -d "${COMPOSE_DIR}" ]] || {
    log "ERROR: Missing Compose directory: ${COMPOSE_DIR}"
    exit 1
}

[[ -d "${PASSPORT_VOLUME}" ]] || {
    log "ERROR: Passport drive is not mounted at ${PASSPORT_VOLUME}"
    exit 1
}

[[ -r "${RESTIC_PASSWORD_FILE}" ]] || {
    log "ERROR: Missing restic password file"
    exit 1
}

[[ -d "${RESTIC_REPOSITORY}" ]] || {
    log "ERROR: Restic repository does not exist"
    exit 1
}

export RESTIC_REPOSITORY
export RESTIC_PASSWORD_FILE

log "Stopping services"
(
    cd "${COMPOSE_DIR}"
    docker compose stop
)
containers_stopped=true

log "Creating snapshot"
restic backup "${DATA_DIR}" \
    --host "mac-mini" \
    --tag "personal-cloud" \
    --exclude-caches

restart_services

log "Applying retention"
restic forget \
    --host "mac-mini" \
    --tag "personal-cloud" \
    --keep-daily 7 \
    --keep-weekly 5 \
    --keep-monthly 12 \
    --keep-yearly 3 \
    --prune

log "Checking repository"
restic check --read-data-subset=5%

log "Backup completed"
```

Make executable:

```bash
chmod 700 "$HOME/bin/backup-personal-cloud"
```

## 5. Run Initial Backup

```bash
"$HOME/bin/backup-personal-cloud"
```

Verify:

```bash
export RESTIC_REPOSITORY="/Volumes/Passport/Backups/personal-cloud.restic"
export RESTIC_PASSWORD_FILE="$HOME/.config/restic/personal-cloud-password"

restic snapshots
restic stats latest
```

## 6. Schedule with Launchd

Create:

```text
~/Library/LaunchAgents/net.damien.personal-cloud-backup.plist
```

Contents:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "https://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">
<dict>
    <key>Label</key>
    <string>net.damien.personal-cloud-backup</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/me/bin/backup-personal-cloud</string>
    </array>

    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/me/Library/Logs/personal-cloud-backup.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/me/Library/Logs/personal-cloud-backup-error.log</string>
</dict>
</plist>
```

Replace `/Users/me` with the actual home directory path.

Validate and load:

```bash
plutil -lint \
  "$HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist"

launchctl bootstrap \
  "gui/$(id -u)" \
  "$HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist"
```

Run immediately:

```bash
launchctl kickstart \
  -k "gui/$(id -u)/net.damien.personal-cloud-backup"
```

Check status:

```bash
launchctl print \
  "gui/$(id -u)/net.damien.personal-cloud-backup"
```

Check logs:

```bash
tail -100 "$HOME/Library/Logs/personal-cloud-backup.log"
tail -100 "$HOME/Library/Logs/personal-cloud-backup-error.log"
```

## 7. Manual Pre-Change Backup

Before an upgrade or migration:

```bash
"$HOME/bin/backup-personal-cloud"
```

Confirm a new snapshot exists:

```bash
restic snapshots --latest 1
```

Do not proceed if the backup fails.

## 8. Restore

List snapshots:

```bash
export RESTIC_REPOSITORY="/Volumes/Passport/Backups/personal-cloud.restic"
export RESTIC_PASSWORD_FILE="$HOME/.config/restic/personal-cloud-password"

restic snapshots
```

Inspect snapshot contents:

```bash
restic ls latest
```

Restore to a temporary directory:

```bash
RESTORE_DIR="$(mktemp -d "$HOME/restic-restore.XXXXXX")"

restic restore latest \
  --target "${RESTORE_DIR}"

echo "${RESTORE_DIR}"
```

Inspect restored data before replacing production files:

```bash
find "${RESTORE_DIR}" -maxdepth 4 -type f | head -100
```

To restore a specific snapshot:

```bash
restic restore SNAPSHOT_ID \
  --target "${RESTORE_DIR}"
```

To restore one service:

```bash
restic restore latest \
  --target "${RESTORE_DIR}" \
  --include "$HOME/personal-cloud-data/uptime-kuma"
```

## 9. Production Recovery

Stop services:

```bash
cd "$HOME/personal-cloud"
docker compose stop
```

Move current data aside:

```bash
mv "$HOME/personal-cloud-data" \
   "$HOME/personal-cloud-data.failed.$(date '+%Y%m%d-%H%M%S')"
```

Move restored data into place.

Confirm ownership and permissions match the previous data directory.

Start services:

```bash
docker compose start
docker compose ps
```

Validate:

* Forgejo repositories and users
* Open WebUI conversations and configuration
* Uptime Kuma monitors and history
* Container logs
* Service health checks

Do not delete the failed data directory until recovery is confirmed.

## 10. Maintenance

List snapshots:

```bash
restic snapshots
```

Full repository check:

```bash
restic check --read-data
```

Run quarterly or after suspected disk errors.

Show repository size:

```bash
restic stats
```

Unlock after a confirmed interrupted operation:

```bash
restic unlock
```

Do not run `unlock` while another restic process is active.

## 11. Quarterly Restore Test

```bash
RESTORE_DIR="$(mktemp -d "$HOME/restic-test.XXXXXX")"

restic restore latest \
  --target "${RESTORE_DIR}"

find "${RESTORE_DIR}" -type f | head
```

Confirm that data exists for all services.

Record:

```text
Date:
Snapshot:
Restore completed:
Forgejo data present:
Open WebUI data present:
Uptime Kuma data present:
Issues:
```

Remove the test restore:

```bash
rm -rf "${RESTORE_DIR}"
```

## 12. Failure Checks

### Passport drive missing

Expected behaviour:

```text
Backup exits without writing to the internal disk.
```

Check:

```bash
ls -ld "/Volumes/Passport"
diskutil list
```

### Repository locked

Check for active processes:

```bash
pgrep -af restic
```

Only when no restic process is running:

```bash
restic unlock
```

### Backup failed after containers stopped

The script trap should restart services.

Verify:

```bash
cd "$HOME/personal-cloud"
docker compose ps
docker compose start
```

### Repository password unavailable

Recover it from the password manager.

The repository cannot be recovered without the password.
