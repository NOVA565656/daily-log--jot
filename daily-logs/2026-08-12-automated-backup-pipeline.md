# Automated Backup Pipeline: VPS Disaster Recovery (The Hard Way)

**Date:** 2026-08-12  
**Time:** 01:30 PM - 02:30 AM (yes, really)  
**Status:** Finally working. Maybe. Let's see if cron actually runs it tomorrow.

---

## The Setup

I decided today was the day to stop living dangerously with my VPS config files scattered everywhere like digital breadcrumbs. Critical stuff: nginx configs, database backups, app secrets (well, *supposed* to be secret—more on that later). Time to build an automated backup pipeline:

- **Source:** Critical files on my VPS (`/etc`, `/var/www/app/config`, database dumps)
- **Transport:** `rsync` (local compression) → `rclone` (S3-compatible upload)
- **Destination:** Backblaze B2 bucket (cheap, reliable, disaster recovery vibes)
- **Scheduler:** Cron (runs nightly at 02:00 AM)

Sounds simple, right? Ha. This is where 13 hours of my life went.

---

## The Script

Here's the core backup script I wrote at 01:30 PM (when I was still optimistic):

```bash
#!/bin/bash

# backup.sh - Daily VPS backup to Backblaze B2

set -e  # Exit on any error

# Load environment variables (S3 credentials, bucket name, etc.)
source /home/admin/.env

BACKUP_DIR="/tmp/backup-$(date +%Y%m%d-%H%M%S)"
REMOTE_BUCKET="s3://my-backup-bucket"
LOG_FILE="/var/log/backup.log"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Rsync critical files from various locations
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting backup..." >> "$LOG_FILE"

rsync -av --delete \
  /etc \
  /var/www/app/config \
  /root/.ssh \
  "$BACKUP_DIR/vps-files/" >> "$LOG_FILE" 2>&1

# Compress everything
tar -czf "$BACKUP_DIR/backup-$(date +%Y%m%d).tar.gz" -C "$BACKUP_DIR" vps-files/

# Upload to Backblaze B2 via rclone
rclone copy "$BACKUP_DIR/backup-$(date +%Y%m%d).tar.gz" "$REMOTE_BUCKET/daily/" \
  --config /home/admin/.config/rclone/rclone.conf >> "$LOG_FILE" 2>&1

# Cleanup
rm -rf "$BACKUP_DIR"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Backup complete!" >> "$LOG_FILE"
```

Simple, elegant, *completely broken*. Read on.

---

## The chmod +x Disaster (01:45 PM - 02:15 PM)

**The Struggle:**

At 01:45 PM, I tested the script manually. Worked fine:

```bash
$ ./backup.sh
[2026-08-12 01:47:22] Starting backup...
[2026-08-12 01:48:15] Backup complete!
```

Felt good. Felt *really* good. Then I set up the cron job:

```bash
0 2 * * * /home/admin/backup.sh >> /var/log/backup-cron.log 2>&1
```

Went to bed. Woke up at 02:30 AM because I'm paranoid. Checked the logs:

```bash
$ tail -f /var/log/backup-cron.log
# ... nothing. Empty. Total silence.
```

Panic. 02:15 AM, I'm digging through `/var/spool/mail/admin`, cron logs, systemctl status... nothing. The cron job ran (I could see it in the syslog), but the script didn't execute.

**The Facepalm:**

02:17 AM, I run:

```bash
$ ls -la /home/admin/backup.sh
-rw-r--r-- 1 admin admin 1247 Aug 12 01:30 backup.sh
```

There it is. No `x`. Of course. I wrote a file, never made it executable, then wondered why cron couldn't run it. Cron runs as a limited user and doesn't get the "benefit" of my shell assumptions.

**The Fix:**

```bash
chmod +x /home/admin/backup.sh
```

Ran the cron job manually to test:

```bash
$ /usr/bin/env bash -c "0 2 * * * /home/admin/backup.sh"
# (or just wait for cron at 2 AM)
```

Lesson: **Cron is not your interactive shell. Be explicit. Be paranoid. Use full paths. Use chmod +x.**

---

## The Trailing Slash Trap (02:20 PM - 02:35 PM)

**The Struggle:**

After the chmod fix, the script *ran*, but something was wrong. The backup directory structure looked like:

```
backup-20260812-023045/
└── vps-files/
    ├── vps-files/  ← WAIT, THIS SHOULDN'T EXIST
    │   ├── etc/
    │   ├── var/
    │   └── ...
    └── etc/
    └── var/
    └── ...
```

I had nested the source directories *inside* the destination. The backup was doubled. The tar.gz was massive. I was uploading garbage to B2.

**Why?**

I wrote:

```bash
rsync -av --delete /etc /var/www/app/config "$BACKUP_DIR/vps-files/" 
```

With the trailing slash on the destination (`vps-files/`), rsync copies files *into* that directory *as a new subdirectory*. So:

- `/etc` → `$BACKUP_DIR/vps-files/etc/` ✓ (correct)
- BUT ALSO: `$BACKUP_DIR/vps-files/vps-files/` ← the destination itself got created inside itself

Without the trailing slash:

```bash
rsync -av --delete /etc /var/www/app/config "$BACKUP_DIR/vps-files"
```

It would treat `vps-files` as a new directory name and drop everything there *directly*.

**The Real Fix:**

For multiple sources into one destination, you need:

```bash
rsync -av --delete \
  /etc/ \
  /var/www/app/config/ \
  /root/.ssh/ \
  "$BACKUP_DIR/vps-files/"
```

Trailing slashes on *sources* (means "copy the contents, not the directory itself") and a trailing slash on the destination (means "this is the target directory").

Or, if you want the source directory names preserved:

```bash
rsync -av --delete \
  /etc \
  /var/www/app/config \
  /root/.ssh \
  "$BACKUP_DIR/"
```

I read the rsync man page. Five. Times. At 02:30 AM. It was painful.

---

## Cron & Credentials (02:35 PM - 02:45 PM)

**The Rookie Mistake:**

Original script (before I caught myself):

```bash
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export RCLONE_CONFIG_B2_ACCESS_KEY="blah_blah_blah"
export RCLONE_CONFIG_B2_SECRET_KEY="more_blah_blah"
```

Hardcoded. In the script. Committed to git (well, I caught it before pushing, thank god).

**The Fix:**

Created `/home/admin/.env`:

```bash
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export RCLONE_CONFIG_B2_ACCESS_KEY="blah_blah_blah"
export RCLONE_CONFIG_B2_SECRET_KEY="more_blah_blah"
```

Then in the script:

```bash
source /home/admin/.env
```

Added `.env` to `.gitignore`:

```
# .gitignore
.env
.env.local
*.log
```

**Permissions:**

```bash
chmod 600 /home/admin/.env  # Only readable by owner
```

This way:
- Secrets live outside the repo
- Cron can still load them via `source`
- I don't accidentally commit credentials
- Backups still have access at 02:00 AM

---

## Final Status (02:45 AM)

✓ Script is executable  
✓ Rsync uses correct trailing slash logic  
✓ Credentials are in `.env` and git-ignored  
✓ Cron job set up  
✓ First test run completed successfully  
✓ Backup uploaded to Backblaze B2  
✓ I'm going to sleep now

Fingers crossed this actually runs tomorrow at 2 AM without silently failing again.

**Next time:** Monitoring. AlertManager. Slack notifications when backups fail. But that's a problem for Future Me.

---

## Key Takeaways

1. **chmod +x matters.** Cron doesn't use your shell magic. Make scripts executable.
2. **Trailing slashes in rsync are subtle but critical.** Test with `--dry-run` first.
3. **Never hardcode secrets.** Use `.env` files, load them at runtime, and `.gitignore` them hard.
4. **Test cron jobs manually before relying on them.** Run `sudo -u <cron_user> /path/to/script` to simulate cron's environment.

---

*Posted at 02:45 AM. Still caffeinated. Would do this again (but better).*
