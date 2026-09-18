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
- [Precticle](#demo)

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
## Precticle Demo
1. step : 
mushahedur-rahman-khan@pulock:~$ ssh-keygen -t ed25519 -C "office-backup"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/mushahedur-rahman-khan/.ssh/id_ed25519): mrk
Enter passphrase for "mrk" (empty for no passphrase): 

mushahedur-rahman-khan@pulock:~$ ssh-keygen -t ed25519 -C "office-backup"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/mushahedur-rahman-khan/.ssh/id_ed25519): 
/home/mushahedur-rahman-khan/.ssh/id_ed25519 already exists.
Overwrite (y/n)? y
Enter passphrase for "/home/mushahedur-rahman-khan/.ssh/id_ed25519" (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/mushahedur-rahman-khan/.ssh/id_ed25519
Your public key has been saved in /home/mushahedur-rahman-khan/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:SjMRKsFSJ+ySuHHYN3wcjlik0pxQUJvcdMov04w9N2Y office-backup
The key's randomart image is:
+--[ED25519 256]--+
|oB*.+ o          |
|.=o%.+..         |
|o*X+=+..         |
|*.=.=*+.         |
| = .+oX E        |
|.    + O .       |
|      .          |
|                 |
|                 |
+----[SHA256]-----+
mushahedur-rahman-khan@pulock:~$ 

2. Step : 
mushahedur-rahman-khan@pulock:~$ ssh-copy-id root@200.97.175.152
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: ssh-add -L
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@200.97.175.152's password: 

Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'root@200.97.175.152'"
and check to make sure that only the key(s) you wanted were added.

3. Step: check go to server without password.
mushahedur-rahman-khan@pulock:~$ ssh root@200.97.175.152
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-136-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro


Expanded Security Maintenance for Applications is not enabled.

9 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

12 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


*** System restart required ***
Last login: Mon Sep  7 09:34:04 2026 from 103.59.178.233
root@srv1839994:~# exit
logout
Connection to 200.97.175.152 closed.

4. Step:

mkdir -p ~/VPS-Backups
mushahedur-rahman-khan@pulock:~$ ls
 'Build a google meet'   English       Movie      Pictures            Videos                      issues
 Desktop                 'ICT Layer'   Music      Public              docker-project              snap
 Documents               Learning      PC_Video   Resources-project   dumps                       software
 Downloads               Logic         Personal   VPS-Backups         github-recovery-codes.txt   test-project
mushahedur-rahman-khan@pulock:~$ nano ~/backup-project.sh
#!/bin/bash

###########################################
 VPS Daily Source Code Backup Script
 Author : Mushahedur 
###########################################

 ===== VPS Information =====
VPS_USER="root"
VPS_IP="300.97.375.252"

 VPS Project Location
PROJECT_PATH="/var/www/html"

 Local Backup Folder
LOCAL_BACKUP="$HOME/VPS-Backups"

 Today's Date
TODAY=$(date +%F)

 Backup File Name
FILE_NAME="html-$TODAY.tar.gz"

echo "========== Backup Started =========="

 Create Today's Folder
mkdir -p "$LOCAL_BACKUP/$TODAY"

 Create tar.gz in VPS (কোনো কিছু বাদ না দিয়ে সম্পূর্ণ ব্যাকআপ)
echo "Creating full compressed tarball on VPS..."
ssh $VPS_USER@$VPS_IP "
tar -czf /tmp/$FILE_NAME -C /var/www html
"

 Download Backup
scp $VPS_USER@$VPS_IP:/tmp/$FILE_NAME "$LOCAL_BACKUP/$TODAY/"

 Remove Temporary Backup From VPS
ssh $VPS_USER@$VPS_IP "rm -f /tmp/$FILE_NAME"

echo "Backup Completed Successfully!"
echo "Saved : $LOCAL_BACKUP/$TODAY/$FILE_NAME"

4. Step:
chmod +x ~/backup-project.sh
mushahedur-rahman-khan@pulock:~$ ~/backup-project.sh
========== Backup Started ==========
Creating full compressed tarball on VPS...
html-2026-09-18.tar.gz                                                                                                                                      100%  542     2.6KB/s   00:00    
Backup Completed Successfully!
Saved : /home/mushahedur-rahman-khan/VPS-Backups/2026-09-18/html-2026-09-18.tar.gz

6. Step:
mushahedur-rahman-khan@pulock:~$ crontab -e
no crontab for mushahedur-rahman-khan - using an empty one
Select an editor.  To change later, run select-editor again.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.basic
  3. /usr/bin/vim.tiny
  4. /usr/bin/code
  5. /usr/bin/antigravity
  6. /bin/ed

Choose 1-6 [1]: 
crontab: installing new crontab

7. Step: *add time*
crontab -l
 Edit this file to introduce tasks to be run by cron.

 Each task to run has to be defined through a single line
 indicating with different fields when the task will be run
 and what command to run for the task
 
 To define the time you can provide concrete values for
 minute (m), hour (h), day of month (dom), month (mon),
 and day of week (dow) or use '*' in these fields (for 'any').
 
 Notice that tasks will be started based on the cron's system
 daemon's notion of time and timezones.

 Output of the crontab jobs (including errors) is sent through
 email to the user the crontab file belongs to (unless redirected).
 
 For example, you can run a backup of all your user accounts
 at 5 a.m every week with:
 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
 
 For more information see the manual pages of crontab(5) and cron(8)

 m h  dom mon dow   command
50 22 * * * /home/mushahedur-rahman-khan/backup-project.sh >> /home/mushahedur-rahman-khan/backup.log 2>&1


## 👨‍💻 Author
- **Name:** Mushahedur Rahman Khan (MRK)
- **Portfolio Website:** [mushahadur.github.io/Portfolio-Website](https://mushahadur.github.io/Portfolio-Website)

Feel free to fork this project, open issues, or submit pull requests to add new automation modules (e.g., MySQL auto-dump features).
