# Runbook: Restic Backups for Personal Cloud Services

## Purpose

Create nightly encrypted snapshots of self-hosted service data from the Mac mini to the attached Passport drive.

## Paths

| Purpose | Path |
|---|---|
| Source data | `$HOME/personal-cloud-data` |
| Passport volume | `/Volumes/passport` |
| Backup parent directory | `/Volumes/passport/backups` |
| Restic repository | `/Volumes/passport/backups/personal-cloud.restic` |
| Restic password file | `$HOME/.config/restic/personal-cloud-password` |
| Backup script | `$HOME/bin/backup-personal-cloud.fish` |
| LaunchAgent | `$HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist` |
| Standard log | `$HOME/Library/Logs/personal-cloud-backup.log` |
| Error log | `$HOME/Library/Logs/personal-cloud-backup-error.log` |

The examples assume the Compose file is stored under:

```text
$HOME/personal-cloud
```

Change `COMPOSE_DIR` in the script if required.

---

## 1. Install Restic

```fish
brew install restic
restic version
```

Confirm the Passport drive is mounted:

```fish
test -d /Volumes/passport
and echo "Passport mounted"
or echo "ERROR: Passport not mounted"
```

Check capacity:

```fish
df -h /Volumes/passport
```

---

## 2. Create the Repository Password

```fish
mkdir -p $HOME/.config/restic

openssl rand -base64 32     > $HOME/.config/restic/personal-cloud-password

chmod 600 $HOME/.config/restic/personal-cloud-password
```

Store a second copy in a password manager.

The repository cannot be recovered without this password.

---

## 3. Initialise the Repository

Set variables for the current fish session:

```fish
set -gx RESTIC_REPOSITORY     /Volumes/passport/backups/personal-cloud.restic

set -gx RESTIC_PASSWORD_FILE     $HOME/.config/restic/personal-cloud-password
```

Create the backup directory:

```fish
mkdir -p /Volumes/passport/backups
df -h /Volumes/passport/backups
```

Initialise the repository:

```fish
restic init
```

Verify:

```fish
restic snapshots
```

---

## 4. Install the Backup Script

Create the script:

```fish
mkdir -p $HOME/bin
nano $HOME/bin/backup-personal-cloud.fish
```

Paste:

```fish
#!/opt/homebrew/bin/fish

set -g DATA_DIR "$HOME/personal-cloud-data"
set -g COMPOSE_DIR "$HOME/personal-cloud"

set -g PASSPORT_VOLUME "/Volumes/passport"
set -g RESTIC_REPOSITORY "$PASSPORT_VOLUME/backups/personal-cloud.restic"
set -g RESTIC_PASSWORD_FILE "$HOME/.config/restic/personal-cloud-password"

set -g HOST_TAG "mac-mini"
set -g BACKUP_TAG "personal-cloud"

# launchd provides a minimal environment.
set -gx PATH /opt/homebrew/bin /usr/local/bin /usr/bin /bin /usr/sbin /sbin

set -g running_services
set -g services_stopped 0

function log
    echo (date "+%Y-%m-%d %H:%M:%S") $argv
end

function fail
    log "ERROR:" $argv
    exit 1
end

function restart_services
    if test $services_stopped -eq 1
        if test (count $running_services) -gt 0
            log "Restarting previously running services:"                 (string join ", " $running_services)

            docker compose                 --project-directory "$COMPOSE_DIR"                 start $running_services

            if test $status -ne 0
                log "ERROR: Failed to restart one or more services"
                return 1
            end
        end

        set -g services_stopped 0
    end

    return 0
end

function handle_exit --on-event fish_exit
    restart_services
end

function handle_signal --on-signal INT --on-signal TERM
    log "Received termination signal"
    restart_services
    exit 130
end

function require_command
    command -q $argv[1]
    or fail "Required command not found:" $argv[1]
end

require_command restic
require_command docker
require_command realpath

test -d "$DATA_DIR"
or fail "Data directory does not exist: $DATA_DIR"

test -d "$COMPOSE_DIR"
or fail "Compose directory does not exist: $COMPOSE_DIR"

test -f "$COMPOSE_DIR/compose.yaml"     -o -f "$COMPOSE_DIR/compose.yml"     -o -f "$COMPOSE_DIR/docker-compose.yaml"     -o -f "$COMPOSE_DIR/docker-compose.yml"
or fail "No Compose file found in: $COMPOSE_DIR"

test -d "$PASSPORT_VOLUME"
or fail "Passport drive is not mounted at: $PASSPORT_VOLUME"

test -r "$RESTIC_PASSWORD_FILE"
or fail "Restic password file is unavailable: $RESTIC_PASSWORD_FILE"

test -d "$RESTIC_REPOSITORY"
or fail "Restic repository does not exist: $RESTIC_REPOSITORY"

set resolved_volume (realpath "$PASSPORT_VOLUME")
set resolved_repository (realpath "$RESTIC_REPOSITORY")

string match --quiet "$resolved_volume/*" "$resolved_repository"
or fail "Repository is not located on the Passport drive"

set -gx RESTIC_REPOSITORY
set -gx RESTIC_PASSWORD_FILE

log "Finding currently running Compose services"

set -g running_services (
    docker compose         --project-directory "$COMPOSE_DIR"         ps --services --filter status=running
)

if test $status -ne 0
    fail "Could not inspect Compose services"
end

if test (count $running_services) -gt 0
    log "Stopping services:" (string join ", " $running_services)

    docker compose         --project-directory "$COMPOSE_DIR"         stop $running_services

    if test $status -ne 0
        fail "Could not stop Compose services"
    end

    set -g services_stopped 1
else
    log "No Compose services are currently running"
end

log "Creating restic snapshot"

restic backup "$DATA_DIR"     --host "$HOST_TAG"     --tag "$BACKUP_TAG"

set backup_status $status

restart_services
set restart_status $status

if test $backup_status -ne 0
    fail "Restic backup failed with status $backup_status"
end

if test $restart_status -ne 0
    fail "Backup succeeded, but service restart failed"
end

log "Applying retention policy"

restic forget     --host "$HOST_TAG"     --tag "$BACKUP_TAG"     --keep-daily 7     --keep-weekly 5     --keep-monthly 12     --keep-yearly 3     --prune

if test $status -ne 0
    fail "Retention or prune operation failed"
end

log "Checking repository"

restic check

if test $status -ne 0
    fail "Repository check failed"
end

log "Backup completed successfully"

restic snapshots     --host "$HOST_TAG"     --tag "$BACKUP_TAG"     --latest 1
```

Confirm the fish executable path:

```fish
command -v fish
```

If the result is not `/opt/homebrew/bin/fish`, update the script shebang.

Make the script executable:

```fish
chmod 700 $HOME/bin/backup-personal-cloud.fish
```

Validate syntax:

```fish
fish --no-execute $HOME/bin/backup-personal-cloud.fish
```

Run manually:

```fish
$HOME/bin/backup-personal-cloud.fish
```

Do not configure scheduling until the manual run succeeds and services restart.

---

## 5. Verify the Initial Backup

Set the repository variables:

```fish
set -gx RESTIC_REPOSITORY     /Volumes/passport/backups/personal-cloud.restic

set -gx RESTIC_PASSWORD_FILE     $HOME/.config/restic/personal-cloud-password
```

List snapshots:

```fish
restic snapshots
```

Inspect the latest snapshot:

```fish
restic ls latest
```

Show snapshot statistics:

```fish
restic stats latest
```

Run a full integrity check after the first backup:

```fish
restic check --read-data
```

---

## 6. Configure the Nightly LaunchAgent

Create log files:

```fish
mkdir -p $HOME/Library/Logs
touch $HOME/Library/Logs/personal-cloud-backup.log
touch $HOME/Library/Logs/personal-cloud-backup-error.log
```

Create the LaunchAgent:

```fish
set plist     $HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist

mkdir -p $HOME/Library/LaunchAgents

printf '%s
' '<?xml version="1.0" encoding="UTF-8"?>' '<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"' '  "https://www.apple.com/DTDs/PropertyList-1.0.dtd">' '<plist version="1.0">' '<dict>' '    <key>Label</key>' '    <string>net.damien.personal-cloud-backup</string>' '' '    <key>ProgramArguments</key>' '    <array>' "        <string>$HOME/bin/backup-personal-cloud.fish</string>" '    </array>' '' '    <key>StartCalendarInterval</key>' '    <dict>' '        <key>Hour</key>' '        <integer>3</integer>' '        <key>Minute</key>' '        <integer>0</integer>' '    </dict>' '' '    <key>StandardOutPath</key>' "    <string>$HOME/Library/Logs/personal-cloud-backup.log</string>" '' '    <key>StandardErrorPath</key>' "    <string>$HOME/Library/Logs/personal-cloud-backup-error.log</string>" '' '    <key>ProcessType</key>' '    <string>Background</string>' '</dict>' '</plist>' > $plist
```

This runs the backup every night at `03:00`.

Validate:

```fish
plutil -lint $plist
plutil -p $plist
```

Unload any previous registration:

```fish
launchctl bootout     gui/(id -u)     $plist 2>/dev/null
```

Load:

```fish
launchctl bootstrap     gui/(id -u)     $plist
```

Inspect job state:

```fish
launchctl print     gui/(id -u)/net.damien.personal-cloud-backup
```

---

## 7. Test Through Launchd

Trigger an immediate run:

```fish
launchctl kickstart -k     gui/(id -u)/net.damien.personal-cloud-backup
```

Monitor standard output:

```fish
tail -f $HOME/Library/Logs/personal-cloud-backup.log
```

Monitor errors in another terminal:

```fish
tail -f $HOME/Library/Logs/personal-cloud-backup-error.log
```

Check the last exit code:

```fish
launchctl print     gui/(id -u)/net.damien.personal-cloud-backup     | grep -E '"state"|last exit code'
```

Expected successful result:

```text
last exit code = 0
```

Confirm a new snapshot exists:

```fish
restic snapshots --latest 1
```

---

## 8. Manual Pre-Change Backup

Run before upgrades, migrations, or significant configuration changes:

```fish
$HOME/bin/backup-personal-cloud.fish
```

Confirm a new snapshot:

```fish
restic snapshots --latest 1
```

Do not continue with the change if the backup fails.

---

## 9. Restore Test

Create a temporary directory:

```fish
set restore_dir (mktemp -d $HOME/restic-restore.XXXXXX)
echo $restore_dir
```

Restore the latest snapshot:

```fish
restic restore latest --target $restore_dir
```

Inspect restored files:

```fish
find $restore_dir -maxdepth 5 -type f | head -50
```

Confirm all service directories exist:

```fish
find $restore_dir -type d     \( -name forgejo        -o -name open-webui        -o -name uptime-kuma \)
```

Remove the test restore:

```fish
rm -rf $restore_dir
```

Perform this test at least quarterly.

---

## 10. Production Restore

List snapshots:

```fish
restic snapshots
```

Restore the selected snapshot to a temporary location:

```fish
set restore_dir (mktemp -d $HOME/restic-restore.XXXXXX)

restic restore SNAPSHOT_ID     --target $restore_dir
```

Stop services:

```fish
docker compose     --project-directory $HOME/personal-cloud     stop
```

Move the current data aside:

```fish
mv $HOME/personal-cloud-data     $HOME/personal-cloud-data.failed.(date "+%Y%m%d-%H%M%S")
```

Copy or move the restored `personal-cloud-data` directory into place.

Confirm ownership and permissions match the previous directory.

Restart services:

```fish
docker compose     --project-directory $HOME/personal-cloud     start
```

Validate:

- Forgejo repositories and users
- Open WebUI conversations and configuration
- Uptime Kuma monitors and history
- Container state and health checks
- Container logs

Do not delete the failed data directory until recovery is confirmed.

---

## 11. Routine Operations

Run backup manually:

```fish
$HOME/bin/backup-personal-cloud.fish
```

List snapshots:

```fish
restic snapshots
```

Inspect latest snapshot:

```fish
restic ls latest
```

Show repository statistics:

```fish
restic stats
```

Run structural integrity check:

```fish
restic check
```

Run full data check:

```fish
restic check --read-data
```

View logs:

```fish
tail -100 $HOME/Library/Logs/personal-cloud-backup.log
tail -100 $HOME/Library/Logs/personal-cloud-backup-error.log
```

Trigger the scheduled job immediately:

```fish
launchctl kickstart -k     gui/(id -u)/net.damien.personal-cloud-backup
```

Disable nightly backups:

```fish
launchctl bootout     gui/(id -u)     $HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist
```

Re-enable nightly backups:

```fish
launchctl bootstrap     gui/(id -u)     $HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist
```

---

## 12. Failure Handling

### Passport drive missing

Expected behaviour:

- Backup exits with a non-zero status.
- No backup directory is created on the internal disk.
- Services are not stopped.

Check:

```fish
ls -ld /Volumes/passport
diskutil list
```

### Backup fails after services stop

The script exit handler should restart previously running services.

Verify:

```fish
docker compose     --project-directory $HOME/personal-cloud     ps

docker compose     --project-directory $HOME/personal-cloud     start
```

### Repository locked

Check for active restic processes:

```fish
pgrep -af restic
```

Only when no restic process is running:

```fish
restic unlock
```

### Repository password unavailable

Recover the password from the password manager.

The repository cannot be recovered without it.

### LaunchAgent does not run

Validate the plist:

```fish
plutil -lint     $HOME/Library/LaunchAgents/net.damien.personal-cloud-backup.plist
```

Inspect job state:

```fish
launchctl print     gui/(id -u)/net.damien.personal-cloud-backup
```

Inspect logs:

```fish
tail -100 $HOME/Library/Logs/personal-cloud-backup.log
tail -100 $HOME/Library/Logs/personal-cloud-backup-error.log
```

Confirm:

- The user is logged in.
- `/Volumes/passport` is mounted.
- Docker is running.
- The script is executable.
- The script shebang matches `command -v fish`.

---

## 13. Retention Policy

The script retains:

```text
Daily:   7
Weekly:  5
Monthly: 12
Yearly:  3
```

Preview retention without deleting data:

```fish
restic forget     --host mac-mini     --tag personal-cloud     --keep-daily 7     --keep-weekly 5     --keep-monthly 12     --keep-yearly 3     --dry-run
```

The script applies the same policy with `--prune` after every successful backup.

---

## 14. Current Limitations

This design protects against:

- Accidental deletion
- Overwritten files
- Internal Mac mini storage failure
- Failed upgrades where a previous snapshot is needed

It does not fully protect against:

- Failure of the Passport drive
- Theft or fire affecting both devices
- Malware with access to both source and backup
- Loss of the restic password
- Backup failures that are never reviewed

Recommended next phase:

```text
Mac mini data
    ↓
Restic repository on Passport
    ↓
Encrypted off-site copy of the repository
    ↓
External backup failure notification
```
