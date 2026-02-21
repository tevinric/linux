# Google Drive Backup Setup Guide - PostgreSQL Database

Complete step-by-step guide for setting up automated PostgreSQL backups to Google Drive on a Linux VPS using rclone. This guide is written to be application-agnostic — wherever your own app's names and values are needed, a clearly marked callout will tell you exactly what to substitute.

> **How to use this guide:**
> Anywhere you see a block like this:
>
> > ⚙️ **Customise this:** description of what to change
>
> ...that is your signal to substitute the placeholder value with one specific to your application. All other steps are identical regardless of your app.

---

## Placeholder Reference

Before you begin, decide on the following values and keep them handy. You will use them throughout this guide.

| Placeholder | What it represents | Example |
|---|---|---|
| `<REMOTE_NAME>` | The name you give your rclone remote | `my_app` |
| `<GDRIVE_FOLDER>` | The Google Drive folder name for backups | `my_app` |
| `<APP_DIR>` | Your project root directory name | `my-app` |
| `<CONTAINER_NAME>` | Your PostgreSQL Docker container name | `postgres_my_app` |
| `<DB_NAME>` | Your PostgreSQL database name | `my_app_db` |
| `<DB_USER>` | Your PostgreSQL database user | `my_app_user` |
| `<DB_NAME_ENV_VAR>` | Env variable holding the DB name | `MYAPP_DB_NAME` |
| `<DB_USER_ENV_VAR>` | Env variable holding the DB user | `MYAPP_DB_USER` |
| `<BACKUP_PREFIX>` | Prefix used in backup filenames | `my_app_backup` |

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Install and Configure rclone](#install-and-configure-rclone)
3. [Create Backup Script](#create-backup-script)
4. [Test Manual Backup](#test-manual-backup)
5. [Set Up Automated Backups](#set-up-automated-backups)
6. [Monitor Backups](#monitor-backups)
7. [Backup Rotation Strategy](#backup-rotation-strategy)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- A running application with a PostgreSQL Docker container
- Google Account with Google Drive access
- Root or sudo access on your Linux VPS
- At least 1GB free space on VPS for temporary backup files

> ⚙️ **Customise this:** Confirm your PostgreSQL container is running before proceeding. The container name will be used throughout this guide as `<CONTAINER_NAME>`.

---

## Install and Configure rclone

### 1. Install rclone

```bash
# Download and install rclone
curl https://rclone.org/install.sh | sudo bash

# Verify installation
rclone version

# Expected output: rclone v1.xx.x
```

### 2. Configure rclone for Google Drive

**Option A: Interactive Configuration (If your VPS has a browser)**

```bash
# Start rclone configuration
rclone config

# Follow the prompts:
# n) New remote
# name> <REMOTE_NAME>
# Storage> drive
# client_id> (Press Enter - leave blank)
# client_secret> (Press Enter - leave blank)
# scope> 1 (Full access)
# root_folder_id> (Press Enter - leave blank)
# service_account_file> (Press Enter - leave blank)
# Edit advanced config? n
# Use web browser to automatically authenticate? y
```

> ⚙️ **Customise this:** Replace `<REMOTE_NAME>` with the name you want to give your rclone remote (e.g. `my_app`). This name is how rclone identifies your Google Drive connection — you will use it in every rclone command throughout this guide.

A browser window will open. Sign in with your Google account and authorize rclone.

---

**Option B: Remote Configuration (For VPS without browser — most common setup)**

This is the most common scenario for VPS setups. You will use your **local machine** to authenticate with Google, then transfer the token to your VPS.

#### Step 1: Install rclone on BOTH machines

**On your VPS** (already done above if you followed Step 1).

**On your local machine (Mac/Windows/Linux):**

- **Mac:** `brew install rclone`
- **Windows:** Download from https://rclone.org/downloads/ and extract the `.exe`
- **Linux:** `curl https://rclone.org/install.sh | sudo bash`

#### Step 2: Get the auth token on your LOCAL machine

Open a terminal on your **local machine** and run:

```bash
rclone authorize "drive"
```

This will:
1. Open a browser window automatically
2. Ask you to sign in to your Google account
3. Ask you to grant rclone access to Google Drive
4. After you click **Allow**, return a token in your terminal

You will see something like this in your terminal:

```
If your browser doesn't open automatically go to the following link:
http://127.0.0.1:53682/auth?...

Log in and authorize rclone for access

Waiting for code...
Got code

Paste the following into your remote machine --->
{"access_token":"ya29.A0...","token_type":"Bearer","refresh_token":"1//0g...","expiry":"2026-02-21T15:32:11.123456789+02:00"}
<---End paste
```

**Copy the entire JSON block** — everything between the arrows, including the curly braces `{ }`. Save it somewhere temporarily (e.g. a text file).

> **Tip:** If the browser doesn't open automatically, rclone will print a URL starting with `http://127.0.0.1:53682/auth?...`. Manually copy and paste that URL into any browser on any machine you are signed into Google with. Complete the auth flow there, then return to the terminal — rclone will detect the completion and print the token.

#### Step 3: Configure rclone on your VPS

SSH into your VPS and run:

```bash
rclone config
```

You will be walked through a series of prompts. Here is exactly what to enter at each step:

```
No remotes found, make a new one?
n) New remote
s) Set configuration password
q) Quit config

n/s/q> n
```

```
name> <REMOTE_NAME>
```

> ⚙️ **Customise this:** Enter the remote name you chose (e.g. `my_app`).

```
Type of storage to configure.
...
XX / Google Drive
   \ "drive"
...

Storage> drive
```

*(Type `drive` — easier than finding the number, which changes between rclone versions)*

```
Google Application Client ID - leave blank normally.
client_id>
```
*(Press Enter — leave blank)*

```
Google Application Client Secret - leave blank normally.
client_secret>
```
*(Press Enter — leave blank)*

```
Scope that rclone should use when requesting access from drive.
 1 / Full access all files...
   \ "drive"
 2 / Read-only access...
...

scope> 1
```

```
ID of the root folder - leave blank to use highest level.
root_folder_id>
```
*(Press Enter — leave blank)*

```
Service Account Credentials JSON file path - leave blank to use interactive login
service_account_file>
```
*(Press Enter — leave blank)*

```
Edit advanced config?
y) Yes
n) No (default)

y/n> n
```

```
Use web browser to automatically authenticate rclone with remote?
 * Say Y if the machine running rclone has a web browser you can use
 * Say N if running rclone on a (remote) machine without web browser access

y/n> n
```

**This is the critical step.** Because you said `n`, rclone will now ask you to paste your token:

```
Please paste token here:
```

Paste the entire JSON token you copied from your local machine in Step 2:

```
{"access_token":"ya29.A0...","token_type":"Bearer","refresh_token":"1//0g...","expiry":"..."}
```

Press **Enter**.

```
Configure this as a Shared Drive (Team Drive)?
y) Yes
n) No (default)

y/n> n
```

You will then see a summary:

```
[<REMOTE_NAME>]
type = drive
scope = drive
token = {"access_token":"ya29...","refresh_token":"1//0g..."}
--------------------
y) Yes this is OK (default)
e) Edit this remote
d) Delete this remote

y/e/d> y
```

Type `y` and press Enter. Then type `q` to quit the config wizard.

```
e/n/d/r/c/s/q> q
```

#### Step 4: Check where the config is stored (for reference)

```bash
# View the rclone config file location
rclone config file

# View its contents (contains your token)
cat ~/.config/rclone/rclone.conf
```

You will see your remote saved there. **Keep this file secure** — it contains your Google Drive access token.

---

### 3. Test Google Drive Connection

```bash
# List Google Drive root directory
rclone lsd <REMOTE_NAME>:

# Create backup folder on Google Drive
rclone mkdir <REMOTE_NAME>:<GDRIVE_FOLDER>

# Verify folder was created
rclone lsd <REMOTE_NAME>:

# You should see your <GDRIVE_FOLDER> listed
```

> ⚙️ **Customise this:** Replace `<REMOTE_NAME>` with your rclone remote name and `<GDRIVE_FOLDER>` with the name of the folder you want created in Google Drive for your backups (e.g. `rclone mkdir my_app:my_app`).

---

## Create Backup Script

### 1. Create Backup Script File

```bash
# Navigate to your project directory
cd <APP_DIR>

# Create scripts directory
mkdir -p scripts
cd scripts

# Create backup script
nano postgresql_backup.sh
```

> ⚙️ **Customise this:** Replace `<APP_DIR>` with your project's root directory name (e.g. `cd my-app`).

### 2. Add Backup Script Content

Copy and paste the following script, then update the values marked with `# <-- CHANGE THIS` comments:

```bash
#!/bin/bash

################################################################################
# PostgreSQL Backup Script for Google Drive
# Backs up PostgreSQL database from Docker container to Google Drive via rclone
################################################################################

# Exit on any error
set -euo pipefail

# Detect script directory and calculate project root if not already set
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DETECTED_PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

# Try to load environment variables from .env or env.sh
if [ -f "$DETECTED_PROJECT_ROOT/.env" ]; then
    set -a
    source "$DETECTED_PROJECT_ROOT/.env"
    set +a
    echo "Loaded environment from .env"
elif [ -f "$DETECTED_PROJECT_ROOT/env.sh" ]; then
    source "$DETECTED_PROJECT_ROOT/env.sh"
    echo "Loaded environment from env.sh"
else
    echo "WARNING: No .env or env.sh file found at $DETECTED_PROJECT_ROOT"
fi

# Use PROJECT_ROOT from env.sh if set, otherwise use detected path
PROJECT_ROOT="${PROJECT_ROOT:-$DETECTED_PROJECT_ROOT}"
echo "Project root: $PROJECT_ROOT"

# ============================================================
# CONFIGURATION — update these values for your application
# ============================================================
CONTAINER_NAME="<CONTAINER_NAME>"                    # <-- CHANGE THIS: your PostgreSQL Docker container name
DB_NAME=${<DB_NAME_ENV_VAR>:-}                       # <-- CHANGE THIS: env variable holding your DB name
DB_USER=${<DB_USER_ENV_VAR>:-}                       # <-- CHANGE THIS: env variable holding your DB user
RCLONE_REMOTE="<REMOTE_NAME>:<GDRIVE_FOLDER>"        # <-- CHANGE THIS: your rclone remote and Google Drive folder
BACKUP_PREFIX="<BACKUP_PREFIX>"                      # <-- CHANGE THIS: prefix for backup filenames (e.g. my_app_backup)
# ============================================================

BACKUP_DIR="$PROJECT_ROOT/backups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_PREFIX}_${TIMESTAMP}.sql"
BACKUP_FILE_GZ="${BACKUP_FILE}.gz"

# Create backup directory first
mkdir -p "$BACKUP_DIR"
LOG_FILE="$BACKUP_DIR/backup_log.txt"

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to log messages
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Function to print colored messages
print_success() {
    echo -e "${GREEN}✓ $1${NC}"
    log_message "SUCCESS: $1"
}

print_error() {
    echo -e "${RED}✗ $1${NC}"
    log_message "ERROR: $1"
}

print_info() {
    echo -e "${YELLOW}ℹ $1${NC}"
    log_message "INFO: $1"
}

# Start backup process
log_message "========================================="
log_message "Starting backup process"
print_info "Starting PostgreSQL backup..."

# Validate required environment variables
if [ -z "$DB_NAME" ] || [ -z "$DB_USER" ]; then
    print_error "Missing required environment variables!"
    print_error "Please ensure your DB name and DB user env variables are set."
    print_error "Check your .env file at: $PROJECT_ROOT/.env"
    exit 1
fi

print_info "Database: $DB_NAME"
print_info "User: $DB_USER"

# Check if container is running
if ! docker ps | grep -q "$CONTAINER_NAME"; then
    print_error "PostgreSQL container '$CONTAINER_NAME' is not running!"
    print_error "Start it with: docker-compose up -d"
    exit 1
fi

print_success "PostgreSQL container is running"

# Create database dump
print_info "Creating database dump..."
if docker exec "$CONTAINER_NAME" pg_dump -U "$DB_USER" -d "$DB_NAME" > "$BACKUP_DIR/$BACKUP_FILE"; then
    print_success "Database dump created: $BACKUP_FILE"
else
    print_error "Failed to create database dump"
    exit 1
fi

# Check if dump file is not empty
if [ ! -s "$BACKUP_DIR/$BACKUP_FILE" ]; then
    print_error "Backup file is empty!"
    rm -f "$BACKUP_DIR/$BACKUP_FILE"
    exit 1
fi

FILE_SIZE=$(du -h "$BACKUP_DIR/$BACKUP_FILE" | cut -f1)
print_info "Backup file size: $FILE_SIZE"

# Compress backup file
print_info "Compressing backup file..."
if gzip "$BACKUP_DIR/$BACKUP_FILE"; then
    print_success "Backup compressed: $BACKUP_FILE_GZ"
else
    print_error "Failed to compress backup"
    exit 1
fi

COMPRESSED_SIZE=$(du -h "$BACKUP_DIR/$BACKUP_FILE_GZ" | cut -f1)
print_info "Compressed file size: $COMPRESSED_SIZE"

# Upload to Google Drive
print_info "Uploading to Google Drive..."
if rclone copy "$BACKUP_DIR/$BACKUP_FILE_GZ" "$RCLONE_REMOTE" --progress; then
    print_success "Backup uploaded to Google Drive successfully"
else
    print_error "Failed to upload backup to Google Drive"
    exit 1
fi

# Verify upload
print_info "Verifying upload..."
if rclone lsf "$RCLONE_REMOTE" | grep -q "$BACKUP_FILE_GZ"; then
    print_success "Backup verified on Google Drive"
else
    print_error "Backup verification failed"
    exit 1
fi

# Clean up local backups older than 7 days
print_info "Cleaning up old local backups (older than 7 days)..."
find "$BACKUP_DIR" -name "${BACKUP_PREFIX}_*.sql.gz" -type f -mtime +7 -delete
print_success "Old local backups cleaned up"

# Clean up old backups on Google Drive (keep last 30 days)
print_info "Cleaning up old Google Drive backups (older than 30 days)..."
rclone delete "$RCLONE_REMOTE" --min-age 30d --include "${BACKUP_PREFIX}_*.sql.gz"
print_success "Old Google Drive backups cleaned up"

log_message "Backup completed successfully"
print_success "Backup process completed!"
print_info "Backup file: $BACKUP_FILE_GZ"
print_info "Location: Google Drive -> $RCLONE_REMOTE"

# Send summary to log
echo "==========================================" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"

# Upload log file to Google Drive
print_info "Uploading log file to Google Drive..."
if rclone copy "$LOG_FILE" "$RCLONE_REMOTE/logs/"; then
    print_success "Log file uploaded to Google Drive ($RCLONE_REMOTE/logs/)"
else
    print_error "Failed to upload log file to Google Drive"
fi
```

### 3. Make Script Executable

```bash
# From your project directory
cd <APP_DIR>

# Make script executable
chmod +x scripts/postgresql_backup.sh

# Verify permissions
ls -l scripts/postgresql_backup.sh

# Expected output: -rwxr-xr-x ... postgresql_backup.sh
```

> ⚙️ **Customise this:** Replace `<APP_DIR>` with your project's root directory name.

---

## Test Manual Backup

### 1. Run Backup Script Manually

```bash
# Navigate to your project directory
cd <APP_DIR>

# Run backup script (DO NOT use 'source' or '.' - this will close your terminal!)
./scripts/postgresql_backup.sh
```

**IMPORTANT:** Always run the script with `./postgresql_backup.sh`. Never use `source ./postgresql_backup.sh` or `. ./postgresql_backup.sh` as this will cause your terminal to close if the script encounters an error.

**Expected Output:**
```
ℹ Starting PostgreSQL backup...
✓ PostgreSQL container is running
ℹ Creating database dump...
✓ Database dump created: <BACKUP_PREFIX>_20260206_143022.sql
ℹ Backup file size: 2.4M
ℹ Compressing backup file...
✓ Backup compressed: <BACKUP_PREFIX>_20260206_143022.sql.gz
ℹ Compressed file size: 487K
ℹ Uploading to Google Drive...
✓ Backup uploaded to Google Drive successfully
ℹ Verifying upload...
✓ Backup verified on Google Drive
ℹ Cleaning up old local backups (older than 7 days)...
✓ Old local backups cleaned up
✓ Backup process completed!
ℹ Uploading log file to Google Drive...
✓ Log file uploaded to Google Drive (<REMOTE_NAME>:<GDRIVE_FOLDER>/logs/)
```

### 2. Verify Backup on Google Drive

**Method 1: Via rclone**
```bash
# List backups on Google Drive
rclone ls <REMOTE_NAME>:<GDRIVE_FOLDER>

# Check backup details
rclone lsl <REMOTE_NAME>:<GDRIVE_FOLDER>

# Download and verify a backup (optional)
rclone copy <REMOTE_NAME>:<GDRIVE_FOLDER>/<BACKUP_PREFIX>_20260206_143022.sql.gz /tmp/
gunzip /tmp/<BACKUP_PREFIX>_20260206_143022.sql.gz
head -n 20 /tmp/<BACKUP_PREFIX>_20260206_143022.sql
```

> ⚙️ **Customise this:** Replace `<REMOTE_NAME>`, `<GDRIVE_FOLDER>`, and `<BACKUP_PREFIX>` with your values. The filename will include the timestamp generated at the time of the backup run.

**Method 2: Via Web Browser**
1. Go to https://drive.google.com
2. Navigate to your `<GDRIVE_FOLDER>` folder
3. Verify backup files are present

### 3. Check Backup Logs

**Local Logs:**
```bash
# From your project directory
cd <APP_DIR>

# View backup log
cat backups/backup_log.txt

# View last 50 lines of log
tail -n 50 backups/backup_log.txt

# Monitor log in real-time (during backup)
tail -f backups/backup_log.txt
```

**Google Drive Logs:**

The backup script automatically uploads the log file to Google Drive after each run. Logs are stored in the `logs` subfolder inside your backup folder.

```bash
# List log files on Google Drive
rclone lsl <REMOTE_NAME>:<GDRIVE_FOLDER>/logs/

# Download the log file from Google Drive to view it
rclone copy <REMOTE_NAME>:<GDRIVE_FOLDER>/logs/backup_log.txt /tmp/
cat /tmp/backup_log.txt
```

You can also view the logs via the Google Drive web interface:
1. Go to https://drive.google.com
2. Navigate to `<GDRIVE_FOLDER>` -> `logs`
3. Open `backup_log.txt` to view the backup history

---

## Set Up Automated Backups

### 1. Configure Cron Job

```bash
# Edit crontab
crontab -e

# If asked to choose an editor, select nano (usually option 1)
```

### 2. Add Cron Schedule

**Generate Your Cron Line Automatically:**

Run this command from your project directory to generate the correct cron line:

```bash
# Navigate to your project directory
cd <APP_DIR>

# Get absolute path
APP_PATH=$(pwd)

# Generate cron line for daily backups at 2:00 AM
echo "0 2 * * * $APP_PATH/scripts/postgresql_backup.sh >> $APP_PATH/backups/cron_log.txt 2>&1"
```

Copy the output and paste it into your crontab.

**Other Schedule Options:**

```bash
# Every 12 hours (2:00 AM and 2:00 PM)
echo "0 2,14 * * * $APP_PATH/scripts/postgresql_backup.sh >> $APP_PATH/backups/cron_log.txt 2>&1"

# Every 6 hours
echo "0 */6 * * * $APP_PATH/scripts/postgresql_backup.sh >> $APP_PATH/backups/cron_log.txt 2>&1"

# Weekly on Sunday at 3:00 AM
echo "0 3 * * 0 $APP_PATH/scripts/postgresql_backup.sh >> $APP_PATH/backups/cron_log.txt 2>&1"
```

**To add to crontab:**
1. Generate your preferred schedule line using the commands above
2. Copy the output
3. Run `crontab -e`
4. Paste the line
5. Save and exit

**Cron Schedule Format:**
```
* * * * * command
│ │ │ │ │
│ │ │ │ └─── Day of week (0-7, Sunday = 0 or 7)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

### 3. Save and Exit

- Press `Ctrl + O` to save
- Press `Enter` to confirm
- Press `Ctrl + X` to exit

### 4. Verify Cron Job

```bash
# List current cron jobs
crontab -l

# Expected output: Your scheduled backup command

# Check cron service status
sudo systemctl status cron

# Restart cron service (if needed)
sudo systemctl restart cron
```

### 5. Test Cron Execution (Optional)

```bash
# From your project directory
cd <APP_DIR>

# Generate a test cron line that runs every 2 minutes
APP_PATH=$(pwd)
echo "*/2 * * * * $APP_PATH/scripts/postgresql_backup.sh >> $APP_PATH/backups/cron_log.txt 2>&1"

# Copy the output, then edit crontab
crontab -e

# Paste the line into crontab, save and exit

# Wait 2-3 minutes and check log
tail -f backups/cron_log.txt

# Remove test cron job after successful test
crontab -e
```

---

## Monitor Backups

### 1. Check Backup Status

```bash
# From your project directory
cd <APP_DIR>

# View recent cron job output
tail -n 100 backups/cron_log.txt

# View backup script log
tail -n 100 backups/backup_log.txt

# Count backups on Google Drive
rclone lsf <REMOTE_NAME>:<GDRIVE_FOLDER> | wc -l

# List all backups with sizes
rclone lsl <REMOTE_NAME>:<GDRIVE_FOLDER>

# Check last backup time
rclone lsl <REMOTE_NAME>:<GDRIVE_FOLDER> | tail -1
```

### 2. Create Monitoring Script

```bash
# From your project directory
cd <APP_DIR>

# Create monitoring script
nano scripts/check_backup_status.sh
```

Add the following content, updating the values marked with `# <-- CHANGE THIS`:

```bash
#!/bin/bash

# Get script directory and project root
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

BACKUP_PREFIX="<BACKUP_PREFIX>"                   # <-- CHANGE THIS
RCLONE_REMOTE="<REMOTE_NAME>:<GDRIVE_FOLDER>"     # <-- CHANGE THIS

echo "=== Backup Status Report ==="
echo ""

# Check last local backup
LAST_LOCAL=$(ls -t "$PROJECT_ROOT/backups/${BACKUP_PREFIX}_*.sql.gz" 2>/dev/null | head -1)
if [ -n "$LAST_LOCAL" ]; then
    echo "Last local backup:"
    ls -lh "$LAST_LOCAL"
else
    echo "No local backups found"
fi

echo ""

# Check last Google Drive backup
echo "Last 3 Google Drive backups:"
rclone lsl "$RCLONE_REMOTE" | tail -3

echo ""

# Check cron log for errors
echo "Recent errors in cron log (if any):"
grep -i "error\|failed" "$PROJECT_ROOT/backups/cron_log.txt" | tail -5

echo ""
echo "=== End of Report ==="
```

Make it executable:

```bash
# From your project directory
chmod +x scripts/check_backup_status.sh

# Run it
./scripts/check_backup_status.sh
```

### 3. Set Up Email Notifications (Optional)

Install mail utilities:

```bash
# Install mailutils
sudo apt install -y mailutils

# Test email
echo "Test email from VPS" | mail -s "Test Subject" your-email@example.com
```

> ⚙️ **Customise this:** Replace `your-email@example.com` with your actual email address.

Modify backup script to send email on failure by adding the following at the end of `postgresql_backup.sh` (before the final exit):

```bash
# Send email notification
APP_NAME="<APP_NAME>"    # <-- CHANGE THIS: a human-readable name for your app
NOTIFY_EMAIL="your-email@example.com"  # <-- CHANGE THIS

if [ $? -eq 0 ]; then
    echo "Backup completed successfully at $(date)" | mail -s "$APP_NAME Backup Success" "$NOTIFY_EMAIL"
else
    echo "Backup failed at $(date). Check logs." | mail -s "$APP_NAME Backup FAILED" "$NOTIFY_EMAIL"
fi
```

---

## Backup Rotation Strategy

The script automatically implements a rotation strategy:

- **Local Backups:** Kept for 7 days, then deleted
- **Google Drive Backups:** Kept for 30 days, then deleted

### Customize Retention Period

Edit the backup script to change retention:

```bash
# From your project directory
cd <APP_DIR>

nano scripts/postgresql_backup.sh

# Find and modify these lines:

# For local backups (currently 7 days):
find "$BACKUP_DIR" -name "${BACKUP_PREFIX}_*.sql.gz" -type f -mtime +7 -delete
# Change +7 to +14 for 14 days, +30 for 30 days, etc.

# For Google Drive backups (currently 30 days):
rclone delete "$RCLONE_REMOTE" --min-age 30d --include "${BACKUP_PREFIX}_*.sql.gz"
# Change 30d to 60d for 60 days, 90d for 90 days, etc.
```

### Calculate Storage Requirements

**Example calculation for daily backups:**

- Backup size: estimate based on your database size when compressed (typically 10–20% of uncompressed)
- Daily backups for 30 days: `compressed_size × 30`
- Google Drive free tier: 15GB

> ⚙️ **Customise this:** Run a manual backup once and check the compressed file size reported in the output. Use that to calculate your actual storage needs.

---

## Troubleshooting

### Issue: rclone not found

```bash
# Reinstall rclone
curl https://rclone.org/install.sh | sudo bash

# Verify installation
which rclone
rclone version
```

### Issue: Google Drive authentication failed

```bash
# Reconfigure rclone
rclone config

# Delete old remote and create new one:
# d) Delete remote
# name> <REMOTE_NAME>
# n) New remote
# ... (follow setup steps again)
```

If the token is rejected or has expired, go back to your **local machine** and re-run:

```bash
rclone authorize "drive"
```

Copy the new token and paste it when re-running `rclone config` on the VPS. Tokens generally last a long time due to the `refresh_token` field, but if you manually revoke rclone's access from your Google account settings, you will need to redo the full auth process.

### Issue: Backup script fails

```bash
# Check Docker container is running
docker ps | grep <CONTAINER_NAME>

# Check database connection
docker exec -it <CONTAINER_NAME> psql -U <DB_USER> -d <DB_NAME> -c "SELECT 1;"

# From your project directory
cd <APP_DIR>

# Run script with verbose output to see exactly where it fails
bash -x scripts/postgresql_backup.sh

# Check script permissions
ls -l scripts/postgresql_backup.sh
chmod +x scripts/postgresql_backup.sh
```

> ⚙️ **Customise this:** Replace `<CONTAINER_NAME>`, `<DB_USER>`, and `<DB_NAME>` with your actual values.

### Issue: Cron job not running

```bash
# Check cron service
sudo systemctl status cron

# Start cron service
sudo systemctl start cron

# Check cron logs
grep CRON /var/log/syslog | tail -20

# Verify crontab syntax
crontab -l

# Make sure script has full paths (no relative paths)
crontab -e
# Use: /home/username/... instead of ~/...
```

### Issue: Upload to Google Drive slow

```bash
# Check bandwidth (limit to 1MB/s for testing)
rclone copy --progress --bwlimit 1M backups/backup_file.sql.gz <REMOTE_NAME>:<GDRIVE_FOLDER>/

# Check Google Drive quota
rclone about <REMOTE_NAME>:

# Test connection speed
rclone test speed <REMOTE_NAME>:
```

### Issue: Backup file is empty

```bash
# Check PostgreSQL container logs
docker logs <CONTAINER_NAME>

# From your project directory
cd <APP_DIR>

# Verify database credentials are set in your .env or env.sh
cat .env | grep DB_
# or
cat env.sh | grep DB_

# Test pg_dump manually
docker exec <CONTAINER_NAME> pg_dump -U <DB_USER> -d <DB_NAME> | head -20
```

> ⚙️ **Customise this:** Replace `<CONTAINER_NAME>`, `<DB_USER>`, and `<DB_NAME>` with your actual values.

---

## Security Considerations

1. **rclone Configuration:** Stored at `~/.config/rclone/rclone.conf` — keep permissions restricted
2. **Backup Files:** Contain sensitive data — ensure proper file permissions on the `backups/` directory
3. **Google Drive:** Consider using a dedicated service account for production environments
4. **Encryption:** Consider encrypting backups before upload:
   ```bash
   # Encrypt backup before upload
   openssl enc -aes-256-cbc -salt -in backup.sql.gz -out backup.sql.gz.enc -k YOUR_PASSWORD
   ```
5. **Access Control:** Limit who has access to the Google Drive backup folder

---

## Next Steps

1. **Set up a restore procedure** and document it alongside this guide
2. **Test the restore process** regularly — a backup you cannot restore is not a backup
3. **Monitor backup health** by checking logs weekly or setting up email alerts
4. **Document credentials** — securely store any encryption passwords and Google account details used

---

## Useful Commands Reference

```bash
# From your project directory
cd <APP_DIR>

# List all backups on Google Drive
rclone lsl <REMOTE_NAME>:<GDRIVE_FOLDER>

# Download a specific backup
rclone copy <REMOTE_NAME>:<GDRIVE_FOLDER>/backup_file.sql.gz ~/downloads/

# Delete a specific backup
rclone delete <REMOTE_NAME>:<GDRIVE_FOLDER>/backup_file.sql.gz

# Check Google Drive available space
rclone about <REMOTE_NAME>:

# Sync local backups to Google Drive (careful — can delete remote files!)
rclone sync backups <REMOTE_NAME>:<GDRIVE_FOLDER>

# Run backup manually
./scripts/postgresql_backup.sh

# Check backup status
./scripts/check_backup_status.sh
```

---

**Last Updated:** February 2026
