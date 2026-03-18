# preparations
```bash
apt install moreutils -y

mkdir -p ~/.restic/
touch ~/.restic/s3.backup
touch ~/.restic/env.s3-config
touch ~/.restic/excludes.txt
touch ~/.restic/logging.log
chmod +x ~/.restic/s3.backup

cd ~/.restic/
```

# environment config
```bash
#choose your editor to edit env file
nvim ~/.restic/env.s3-config
nano ~/.restic/env.s3-config
```

```bash
export AWS_ACCESS_KEY_ID=<id>
export AWS_SECRET_ACCESS_KEY=<key>
export RESTIC_REPOSITORY=s3:contoso.com/bucket
export RESTIC_COMPRESSION=max
export RESTIC_PASSWORD=<password>

export SOURCEDIR=/etc/
export EXCLUDE=~/.restic/excludes.txt
export LOGFILE=~/.restic/logging.log
```

```bash
#choose your editor to edit backup file
nvim ~/.restic/s3.backup
nano ~/.restic/s3.backup
```

# backup script
```bash
#
set -e
source ~/.restic/env.s3-config

# Variablen prüfen
: "${SOURCEDIR:?Variable SOURCEDIR nicht gesetzt}"
: "${EXCLUDE:?Variable EXCLUDE nicht gesetzt}"
: "${LOGFILE:?Variable LOGFILE nicht gesetzt}"

# lock aufheben (fehler ignorieren falls kein lock)
restic unlock || true

# backup durchführen
restic backup "$SOURCEDIR" \
  --exclude-file "$EXCLUDE" \
  --tag configs --verbose \
  2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

# retention policy
restic forget \
  --keep-daily 30 \
  --keep-weekly 52 \
  --keep-monthly 120 \
  --keep-yearly 5 \
  --prune \
  2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

sleep 3

# Cleanup
restic unlock || true
restic cache --cleanup

# snapshots anzeigen und loggen
restic snapshots --verbose \
  2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

# optional: integrity check (empfohlen wöchentlich per cron)
# restic check --verbose 2>&1 | ts '[%Y-%m-%d %H:%M:%S]' | tee -a "$LOGFILE"

cat "$LOGFILE"
exit 0
```

# crontab job
```bash
crontab -e
00 12 * * * <username> bash ~/.restic/s3.backup
```
