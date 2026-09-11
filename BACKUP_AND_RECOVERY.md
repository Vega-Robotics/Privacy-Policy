# MessClub — MongoDB Backup & Recovery

This app's data (customers, hotels, subscriptions, meal balances,
reservations, tickets, payments, hotel payouts, attendance, analytics,
notifications, security logs) all lives in one MongoDB database. A MongoDB
failure or an accidental bad write should never mean permanent data loss.
This document explains how backups work for this project and what to do if
something goes wrong.

---

## 1. What data is included

Every collection in the `mess_platform` database (or whatever you named it
in `MONGO_URI`), including:

- `users` — customers, hotel owners, admin
- `hotels` — hotel profiles, menus embedded/related
- `subscriptionplans`, `subscriptions` — plans and active/expired subscriptions
- `mealreservations`, `mealusages` — reservations and per-meal consumption
- `payments` — subscription purchase records (mock or real Razorpay)
- `hotelpayouts` — what's owed/paid to each hotel
- `reviews`, `dailymeals`, `foods` — content
- `notifications`
- `hotelviews` — aggregated hotel-view analytics (spec sections 14-18)
- `securitylogs` — lightweight security/abuse audit trail (auto-expires after 90 days — see section 5)
- `captchas`, `otps` — short-lived, auto-expiring (TTL indexes) — **never need backing up**, they're
  disposable challenge/verification state, not data

## 2. Recommended approach

### Option A — MongoDB Atlas automated backups (recommended if you're on Atlas)

If `MONGO_URI` points at an Atlas cluster (`...mongodb.net`), Atlas can take
automated, continuous backups for you:

1. In the Atlas dashboard, open your cluster → **Backup**.
2. Enable **Cloud Backup** (continuous backups on M10+ tiers, or the free
   snapshot-based backup on shared/free tiers where available).
3. Set a retention policy (Atlas defaults are reasonable — e.g. keep daily
   snapshots for 7 days, weekly for a month).
4. Atlas lets you restore to a new cluster or in-place from the dashboard,
   or download a snapshot — no `mongodump`/`mongorestore` needed.

This is the lowest-effort, most reliable option and needs no extra
infrastructure on your side. If you're on a paid Atlas tier, turn this on
and you're done with the "automated backups" requirement.

### Option B — Manual backup with `mongodump` / `mongorestore`

Works regardless of where MongoDB is hosted (Atlas, self-hosted, Docker,
etc.) and is a good supplement to Option A even if you have Atlas backups.

**Never put your real connection string in a script committed to Git.**
Always read it from an environment variable.

#### Creating a backup

```bash
# Reads MONGO_URI from your shell environment - never hardcode it here.
mongodump --uri="$MONGO_URI" --out="./backups/$(date +%Y-%m-%d_%H-%M)"
```

This creates one timestamped folder per backup (e.g.
`./backups/2026-09-09_14-30/`) containing a `.bson` + `.json` metadata file
per collection.

Compress it for storage/transfer:

```bash
tar -czf messclub-backup-$(date +%Y-%m-%d).tar.gz ./backups/2026-09-09_14-30
```

#### Restoring from a backup

```bash
# Restores into whatever database MONGO_URI points at. --drop replaces
# existing collections with the backup's version - omit --drop to merge
# instead (only safe if you're restoring into an empty database).
mongorestore --uri="$MONGO_URI" --drop "./backups/2026-09-09_14-30"
```

Always restore into a **test/staging database first** if you're not certain
you want to fully replace production data with the backup.

#### Automating it (cron example, Linux/macOS)

```bash
# crontab -e
# Runs a backup every day at 2:00 AM server time
0 2 * * * MONGO_URI="mongodb+srv://..." /path/to/backup-script.sh >> /var/log/messclub-backup.log 2>&1
```

Where `backup-script.sh` runs the `mongodump` command above and then, ideally,
uploads the resulting archive somewhere off-server (see section 4).

## 3. How often to back up

- **Automated**: daily, at minimum. If Atlas continuous backups are
  available (Option A), that effectively gives you point-in-time recovery
  and daily is a reasonable manual/cron supplement on top.
- **Manual/on-demand**: always take one immediately before:
  - Any production `syncIndexes()`/schema-changing deploy
  - Any bulk data migration or admin bulk-edit script
  - Any planned maintenance where you're touching the database directly

## 4. Where backups should live

- **Never inside the Git repository.** Backups contain real customer PII
  (names, emails, phone numbers) and must not end up in source control or
  any public bucket.
- Store backups somewhere separate from the production database itself —
  e.g. a private S3/Cloud Storage bucket, or Atlas's own backup storage
  (Option A already satisfies this). If production MongoDB and your only
  backup copy are on the same host/disk, that's not a real backup.
- Encrypt backups at rest if your storage provider supports it (most
  S3-compatible buckets do this by default or with one setting).
- Keep at least a few generations of backups (e.g. last 7 daily + last 4
  weekly), not just the most recent one — a corruption that's gone
  unnoticed for a few days is still recoverable this way.

## 5. Security logs are NOT part of the durable backup set

`securitylogs` (failed logins, CAPTCHA failures, registration attempts,
etc. — see spec section 8 / `models/SecurityLog.js`) has a 90-day TTL index
and is meant purely as a rolling abuse-pattern signal, not a permanent
audit trail. It doesn't need to be included in long-term backups — if you
have a compliance requirement for a longer-lived audit trail, export it to
separate storage periodically instead of relying on the live collection.

## 6. How to verify a backup is good

Don't assume a backup succeeded just because the command exited 0:

```bash
# 1. Check the dump actually has content for your main collections
ls -la ./backups/2026-09-09_14-30/mess_platform/
# Expect to see users.bson, hotels.bson, payments.bson, etc. all >0 bytes

# 2. Periodically (e.g. monthly), do a real test restore into a throwaway
#    local/staging database and sanity-check record counts:
mongorestore --uri="mongodb://localhost:27017/messclub_restore_test" "./backups/2026-09-09_14-30"
mongosh "mongodb://localhost:27017/messclub_restore_test" --eval "
  db.users.countDocuments();
  db.hotels.countDocuments();
  db.payments.countDocuments();
"
```

If those counts look roughly right compared to what you expect from
production, the backup is good. An untested backup is not a backup you can
trust in an emergency — actually restoring it somewhere at least once is the
only way to know it works.

## 7. If the database becomes corrupted or unreachable

1. **Don't panic-write to production.** Stop the backend (or at least
   anything that writes) so nothing makes the situation worse.
2. If on Atlas: check Atlas's own health/incident status first — a lot of
   "corruption" scares are actually a transient connectivity issue, not
   real data loss, and Atlas point-in-time recovery may let you roll back
   to just before the bad event without a manual restore at all.
3. If a manual restore is needed: spin up a fresh MongoDB instance/cluster,
   `mongorestore` your most recent verified-good backup into it, update
   `MONGO_URI` to point at the restored instance, and restart the backend.
4. After recovery, compare row counts / spot-check recent records against
   what users report is missing, so you know how much (if any) data between
   the last backup and the incident was lost, and can communicate that
   clearly if needed.
5. Once stable again, figure out *why* it happened (bad migration? disk
   full? credential leak?) before resuming normal writes — restoring a
   backup doesn't fix whatever caused the problem in the first place.
