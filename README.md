# VPS Daily Backup (Ubuntu → Ubuntu)

Automatically back up a live production project (Laravel / Next.js / any web app) from an Ubuntu VPS to a local Ubuntu PC every day. No manual commands after the one-time setup.

- **Password-less** login with an SSH key
- **Server-side compression**: the VPS creates a `tar.gz`, so only one compressed file crosses the network
- **Dated folders**: `~/VPS-Backups/YYYY-MM-DD/`
- **Scheduled with cron**, with a log file for troubleshooting
- **Cleans up after itself**: the temporary archive on the VPS is deleted after download

---

## Table of Contents

- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Configuration](#configuration)
- [Verify a Backup](#verify-a-backup)
- [Restore](#restore)
- [Optional: Automatic Retention](#optional-automatic-retention)
- [Security Notes](#security-notes)
- [Limitations](#limitations)
- [Troubleshooting](#troubleshooting)
- [Practical](#demo)

---

## How It Works

```
 Local Ubuntu PC                              Ubuntu VPS
┌──────────────────┐                        ┌──────────────────────┐
│ cron (scheduled) │                        │ /var/www/html        │
│        │         │  1. ssh: run tar       │                      │
│        ▼         │ ─────────────────────▶ │ tar -czf /tmp/x.tgz  │
│ backup-project.sh│                        │                      │
│        │         │  2. scp: download      │                      │
│        ▼         │ ◀───────────────────── │ /tmp/x.tgz           │
│ ~/VPS-Backups/   │                        │                      │
│   YYYY-MM-DD/    │  3. ssh: rm /tmp/x.tgz │                      │
│                  │ ─────────────────────▶ │ (temp file removed)  │
└──────────────────┘                        └──────────────────────┘
```

1. **cron** starts the script at the scheduled time.
2. The script connects to the VPS over SSH (using a key, so no password prompt) and runs `tar -czf` to compress the project into `/tmp`.
3. The archive is downloaded with `scp` into a folder named after today's date.
4. The temporary archive on the VPS is removed.

---

## Prerequisites

| Where | Requirement |
|---|---|
| Local PC | Ubuntu, `openssh-client`, `cron` |
| VPS | Ubuntu, SSH access, `tar` (installed by default) |
| Network | Local PC can reach the VPS on port 22 |

---

## Setup

### 1. Set up password-less SSH login (one time)

On the **local PC**, generate a key:

```bash
ssh-keygen -t ed25519 -C "office-backup"
```

> ⚠️ If you already have `~/.ssh/id_ed25519`, **do not overwrite it**. Overwriting replaces the key used for your other servers and services (GitHub, etc.). Create a separate key instead:
>
> ```bash
> ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_vps -C "office-backup"
> ssh-copy-id -i ~/.ssh/id_ed25519_vps.pub VPS_USER@YOUR_SERVER_IP
> ```
>
> Then tell SSH to use it for this host in `~/.ssh/config`:
>
> ```
> Host YOUR_SERVER_IP
>     IdentityFile ~/.ssh/id_ed25519_vps
> ```

Copy the public key to the VPS (you will be asked for the password **once**):

```bash
ssh-copy-id VPS_USER@YOUR_SERVER_IP
```

Test it. This must log you in **without** asking for a password:

```bash
ssh VPS_USER@YOUR_SERVER_IP
```

> The key is created with an empty passphrase so that cron can use it unattended. See [Security Notes](#security-notes).

### 2. Create the local backup folder

```bash
mkdir -p ~/VPS-Backups
```

### 3. Create the backup script

```bash
nano ~/backup-project.sh
```

```bash
#!/bin/bash

###########################################
# VPS Daily Source Code Backup Script
###########################################

# Stop on the first error, treat unset variables as errors,
# and fail a pipeline if any command in it fails.
set -euo pipefail

# ===== VPS Information =====
VPS_USER="your_vps_user"
VPS_IP="YOUR_SERVER_IP"

# ===== Project on the VPS =====
# Parent directory and folder name of the project.
# Example: /var/www/html  ->  PROJECT_PARENT=/var/www  PROJECT_NAME=html
PROJECT_PARENT="/var/www"
PROJECT_NAME="html"

# ===== Local Backup Folder =====
LOCAL_BACKUP="$HOME/VPS-Backups"

# ===== Names =====
TODAY=$(date +%F)
FILE_NAME="${PROJECT_NAME}-${TODAY}.tar.gz"

echo "========== Backup Started: $(date) =========="

# Create today's folder
mkdir -p "$LOCAL_BACKUP/$TODAY"

# Create the tar.gz on the VPS (full backup, nothing excluded)
echo "Creating compressed archive on VPS..."
ssh "$VPS_USER@$VPS_IP" "tar -czf /tmp/$FILE_NAME -C $PROJECT_PARENT $PROJECT_NAME"

# Download the archive
echo "Downloading archive..."
scp "$VPS_USER@$VPS_IP:/tmp/$FILE_NAME" "$LOCAL_BACKUP/$TODAY/"

# Remove the temporary archive from the VPS
ssh "$VPS_USER@$VPS_IP" "rm -f /tmp/$FILE_NAME"

echo "Backup Completed Successfully!"
echo "Saved : $LOCAL_BACKUP/$TODAY/$FILE_NAME"
ls -lh "$LOCAL_BACKUP/$TODAY/$FILE_NAME"
```

**Why `set -euo pipefail`?** Without `set -e`, the script keeps going after a failed step. For example, if the download (`scp`) failed, the script would still delete the archive on the VPS and print "Completed Successfully". With `set -e`, it stops at the first failure and the log shows what went wrong.

### 4. Make it executable and test it

```bash
chmod +x ~/backup-project.sh
~/backup-project.sh
```

Expected output:

```
========== Backup Started: ... ==========
Creating compressed archive on VPS...
Downloading archive...
html-2026-09-18.tar.gz                    100%  ...
Backup Completed Successfully!
Saved : /home/your_user/VPS-Backups/2026-09-18/html-2026-09-18.tar.gz
```

> 🔍 **Check the file size.** A real project is usually megabytes. If your archive is only a few hundred bytes, `PROJECT_PARENT` / `PROJECT_NAME` probably point to an empty or default web root. See [Verify a Backup](#verify-a-backup).

### 5. Schedule it with cron

```bash
crontab -e
```

Add one line (use **absolute paths**; cron does not expand `~`):

```cron
0 10 * * * /home/your_user/backup-project.sh >> /home/your_user/backup.log 2>&1
```

The example above runs every day at **10:00 AM**. The five time fields are:

```
┌───────── minute (0-59)
│ ┌─────── hour (0-23)
│ │ ┌───── day of month (1-31)
│ │ │ ┌─── month (1-12)
│ │ │ │ ┌─ day of week (0-7, Sunday = 0 or 7)
│ │ │ │ │
0 10 * * *  command
```

More examples:

| Schedule | Cron expression |
|---|---|
| Every day at 10:00 AM | `0 10 * * *` |
| Every day at 10:50 PM | `50 22 * * *` |
| Every Monday at 2:00 AM | `0 2 * * 1` |
| Every 6 hours | `0 */6 * * *` |

The `>> backup.log 2>&1` part appends both normal output and errors to `backup.log`.

Check that cron has your job and watch the log:

```bash
crontab -l
tail -f ~/backup.log
```

---

## Configuration

| Variable | Meaning | Example |
|---|---|---|
| `VPS_USER` | SSH user on the VPS | `ubuntu` |
| `VPS_IP` | VPS IP address or hostname | `203.0.113.10` |
| `PROJECT_PARENT` | Directory that contains the project | `/var/www` |
| `PROJECT_NAME` | Project folder name | `ecommerce` |
| `LOCAL_BACKUP` | Where backups are stored locally | `$HOME/VPS-Backups` |

### Excluding heavy or rebuildable folders

By default the script backs up **everything**. To skip folders that can be rebuilt (for a Laravel / Next.js project), change the `tar` line:

```bash
ssh "$VPS_USER@$VPS_IP" "tar \
  --exclude='vendor' \
  --exclude='node_modules' \
  --exclude='storage/logs' \
  --exclude='.git' \
  -czf /tmp/$FILE_NAME -C $PROJECT_PARENT $PROJECT_NAME"
```

`vendor` can be restored with `composer install` and `node_modules` with `npm install`. Do **not** exclude `.env`, `storage/app/public`, or `public/uploads` (user-uploaded files).

---

## Resulting Folder Structure

```
~/VPS-Backups/
├── 2026-09-16/
│   └── html-2026-09-16.tar.gz
├── 2026-09-17/
│   └── html-2026-09-17.tar.gz
└── 2026-09-18/
    └── html-2026-09-18.tar.gz
```

---

## Verify a Backup

Is today's backup there, and how big is it?

```bash
ls -lh ~/VPS-Backups/$(date +%F)
```

What is inside? (lists the first entries without extracting)

```bash
tar -tzf ~/VPS-Backups/$(date +%F)/html-$(date +%F).tar.gz | head
```

Test that the archive is not corrupted:

```bash
gzip -t ~/VPS-Backups/$(date +%F)/html-$(date +%F).tar.gz && echo "OK"
```

---

## Restore

Extract into a temporary folder first and inspect before touching production:

```bash
mkdir -p /tmp/restore
tar -xzf ~/VPS-Backups/2026-09-18/html-2026-09-18.tar.gz -C /tmp/restore
```

Copy back to the VPS if needed:

```bash
scp -r /tmp/restore/html/* VPS_USER@YOUR_SERVER_IP:/var/www/html/
```

---

## Optional: Automatic Retention

To delete local backups older than 30 days, add this to the end of the script:

```bash
# Delete backup folders older than 30 days
find "$LOCAL_BACKUP" -mindepth 1 -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;
```

Run it without `-exec rm -rf {} \;` first (just `-print`) to see what would be deleted.

---

## Security Notes

- **Never commit real server details.** Keep your real IP, username and keys out of this repository. Use placeholders in the code you publish.
- **Avoid `root` for backups.** Create a dedicated user on the VPS that can read the project files, and use that user in `VPS_USER`.
- **Protect the private key.** `~/.ssh/id_ed25519` must never leave your PC. Its permissions should be `600`:
  ```bash
  chmod 600 ~/.ssh/id_ed25519
  ```
- **Empty passphrase trade-off.** cron cannot type a passphrase, so anyone who obtains the private key can log in to the VPS. Using a dedicated, low-privilege backup user limits the damage.
- **Backups contain secrets.** A full backup includes `.env` (database passwords, API keys). Restrict access to the backup folder:
  ```bash
  chmod 700 ~/VPS-Backups
  ```
  For extra safety, encrypt archives (for example with `gpg`) before storing them elsewhere.
- **Do not commit `.tar.gz` backups or `backup.log`** to Git. Add them to `.gitignore`.

---

## Limitations

- **Database is not included.** This backs up files only. Back up MySQL separately (for example with `mysqldump` run over SSH) and store the dump alongside the source backup.
- **The local PC must be on** at the scheduled time. cron does not run missed jobs after a shutdown. If that is a problem, choose a time when the PC is normally on, or use `anacron` / a systemd timer with `Persistent=true`.
- **Timezone.** cron uses the **local PC's** timezone, not the VPS's.
- **Single copy.** Backups live on one machine. For real disaster recovery, keep a second copy off-site (external drive or cloud storage).
- **No failure alerts.** Errors only appear in `backup.log`. Consider adding an email or Telegram notification on failure.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Script asks for a password | The key was not copied. Re-run `ssh-copy-id`, then test `ssh VPS_USER@YOUR_SERVER_IP`. |
| Works manually, fails in cron | Use absolute paths in the crontab. Check `~/backup.log` for the error. |
| `Permission denied` on the VPS | `VPS_USER` cannot read the project files. Check ownership and permissions. |
| Archive is only a few hundred bytes | `PROJECT_PARENT` / `PROJECT_NAME` point to the wrong or an empty folder. Run `ls -la /var/www/...` on the VPS. |
| `tar: file changed as we read it` | A file was written during compression. Usually harmless on a live site; re-run if the archive is critical. |
| Cron job never runs | `crontab -l` to confirm it is saved; `systemctl status cron` to confirm the service is running. |

---
## Practical Demo
--

## 🛠️ Step-by-Step Implementation Ledger

### Step 1: Cryptographic Key Generation
Proactively instantiate an advanced `Ed25519` private/public key string flagged with an identifying comment string:
```bash
ssh-keygen -t ed25519 -C "office-backup"
```
*(When prompted for paths and passphrases, bypass by pressing `Enter` to allow seamless automation without human intervention).*

### Step 2: Establish Remote Access Authorization
Ship the newly provisioned public certificate string over onto the authoritative remote user space on the VPS cluster:
```bash
ssh-copy-id root@200.97.175.152
```

### Step 3: Verify Secure Passwordless Handshake
Test the terminal link to confirm direct entry into the remote machine without any manual password requests:
```bash
ssh root@200.97.175.152
```

### Step 4: Provision & Deploy the Automation Script
1. Formulate a shell target script locally in your home workspace:
   ```bash
   nano ~/backup-project.sh
   ```
2. Populate the script layout with the operational pipeline context:
   ```bash
   #!/bin/bash

   # ===== VPS Information =====
   VPS_USER="root"
   VPS_IP="400.197.375.152"
   PROJECT_PATH="/var/www/html"
   LOCAL_BACKUP="$HOME/VPS-Backups"
   TODAY=$(date +%F)
   FILE_NAME="html-$TODAY.tar.gz"

   echo "========== Backup Started =========="
   mkdir -p "$LOCAL_BACKUP/$TODAY"

   # Create full compressed archive inside remote staging path
   echo "Creating full compressed tarball on VPS..."
   ssh $VPS_USER@$VPS_IP "tar -czf /tmp/$FILE_NAME -C /var/www html"

   # Download archive safely over SCP channel
   scp $VPS_USER@$VPS_IP:/tmp/$FILE_NAME "$LOCAL_BACKUP/$TODAY/"

   # Purge staging storage from target node
   ssh $VPS_USER@$VPS_IP "rm -f /tmp/$FILE_NAME"

   echo "Backup Completed Successfully!"
   echo "Saved : $LOCAL_BACKUP/$TODAY/$FILE_NAME"
   ```
3. Grant strict execution capabilities over to the kernel runtime layer:
   ```bash
   chmod +x ~/backup-project.sh
   ```

### Step 5: Clock and Bind Automation using the Cron Engine
Open your default user space task matrix editor:
```bash
crontab -e
```
Inject the operational sequence line to trigger standard backups **every single evening at exactly 10:50 PM (22:50)**, routing standard outputs out directly into local logging tracks:
```text
50 22 * * * /home/mushahedur-rahman-khan/backup-project.sh >> /home/mushahedur-rahman-khan/backup.log 2>&1
```

---

## 📂 Active Storage Mapping Output
Upon successful cron or manual firing, backup files map sequentially inside your storage tracks:
```text
/home/mushahedur-rahman-khan/VPS-Backups/
└── 2026-09-18/
    └── html-2026-09-18.tar.gz   <- Production Source Archive [Verified]
```

---

## 👨‍💻 Engineering Profile
- **Lead Automator:** Mushahedur Rahman Khan (MRK)
- **Interactive Portfolio:** [mushahadur.github.io/Portfolio-Website](https://github.io)

Feel free to fork this platform repository, raise technical issue logs, or submit optimizations to extend data retention models!
