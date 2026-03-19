# Restic S3 Backup

Automated backup script for restic with S3-compatible storage, including retention policy, logging, and cron scheduling.

## Prerequisites

```bash
apt install moreutils -y
```

## Setup

```bash
mkdir -p ~/.restic/
touch ~/.restic/{s3.backup,env.s3-config,excludes.txt,sources.txt,logging.log}
chmod +x ~/.restic/s3.backup
```

## Configuration

### Environment (`~/.restic/env.s3-config`)

```bash
export AWS_ACCESS_KEY_ID=<id>
export AWS_SECRET_ACCESS_KEY=<key>
export RESTIC_REPOSITORY=s3:contoso.com/bucket
export RESTIC_COMPRESSION=max
export RESTIC_PASSWORD=<password>

# Option A: Single directory
export SOURCEDIR=/etc/

# Option B: Multiple directories (comment out SOURCEDIR)
# export FILES_FROM=~/.restic/sources.txt

export EXCLUDE=~/.restic/excludes.txt
export LOGFILE=~/.restic/logging.log
```

> Use either `SOURCEDIR` or `FILES_FROM`, not both.

### Sources (`~/.restic/sources.txt`)

Only needed when using `FILES_FROM`. Use full paths:

```
/root/opt/docker/
/root/tmp/
/etc/
```

### Excludes (`~/.restic/excludes.txt`)

```
*.tmp
*.cache
node_modules
.Trash
```

## Backup Script (`~/.restic/s3.backup`)

```bash
#!/bin/bash
set -e
source ~/.restic/env.s3-config

# validate variables
if [ -z "$SOURCEDIR" ] && [ -z "$FILES_FROM" ]; then
  echo "Fehler: Weder SOURCEDIR noch FILES_FROM gesetzt"
  exit 1
fi
: "${EXCLUDE:?Variable EXCLUDE nicht gesetzt}"
: "${LOGFILE:?Variable LOGFILE nicht gesetzt}"

# unlock stale locks
restic unlock || true

# backup
if [ -n "$FILES_FROM" ]; then
  restic backup --files-from "$FILES_FROM" \
    --exclude-file "$EXCLUDE" \
    --tag configs --verbose \
    2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"
else
  restic backup "$SOURCEDIR" \
    --exclude-file "$EXCLUDE" \
    --tag configs --verbose \
    2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"
fi

# retention policy
restic forget \
  --keep-daily 30 \
  --keep-weekly 52 \
  --keep-monthly 120 \
  --keep-yearly 5 \
  --prune \
  2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

sleep 3

# cleanup
restic unlock || true
restic cache --cleanup

# log snapshots
restic snapshots --verbose \
  2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

# optional: integrity check (recommended weekly via cron)
# restic check --verbose 2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

cat "$LOGFILE"
exit 0
```

## Cron

```bash
crontab -e
```

```
00 12 * * * bash ~/.restic/s3.backup
```

## Useful Commands

```bash
# follow log
tail -fn 20 ~/.restic/logging.log

# list snapshots
source ~/.restic/env.s3-config && restic snapshots

# restore specific snapshot
source ~/.restic/env.s3-config && restic restore <snapshot-id> --target /tmp/restore

# check repository integrity
source ~/.restic/env.s3-config && restic check --verbose
```
