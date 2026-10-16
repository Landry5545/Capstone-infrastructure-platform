# Backup/Restore Validation Drill — Wiki.js PostgreSQL

## Summary

Ran an end-to-end backup/restore drill against Wiki.js's PostgreSQL database (`wikijs-db`) to validate that the backup procedure documented in the Administrator Guide actually produces a restorable backup — not just a command that runs without erroring. The drill surfaced a real gotcha in the naive restore procedure, which was corrected and re-validated.

## Objective

Prove that:
1. A `pg_dump` backup captures real, usable data
2. The documented restore procedure actually restores that data
3. Any gaps between "the command ran" and "the data is actually back" are caught before they're needed in a real incident

## Environment

- Host: `server-vm` (10.10.10.10)
- Container: `wikijs-db` (`postgres:15-alpine`), app container `wikijs` (`requarks/wiki:2`)
- Database: `wikijs`, user: `wikijs`

## Procedure and Findings

### 1. Take a real backup

```bash
docker exec wikijs-db pg_dump -U wikijs wikijs > ~/wikijs-backup-test.sql
ls -lh ~/wikijs-backup-test.sql
```

Result: `97K` file written. Confirmed non-trivial, non-empty dump — not a 0-byte failed-auth file.

### 2. Make a visible, trivial change

Created a new "Home" page in the Wiki.js UI with body text `BACKUP DRILL TEST — DELETE ME`, published it, and confirmed it rendered live at `http://wiki.lab.local/en/home`.

### 3. First restore attempt — failed silently (gotcha found)

```bash
docker exec -i wikijs-db psql -U wikijs -d wikijs < ~/wikijs-backup-test.sql
```

This completed "successfully" in the sense that it ran without stopping, but produced a long list of errors:
ERROR: multiple primary keys for table “X” are not allowed
ERROR: relation “X” already exists
ERROR: constraint “X” for relation “Y” already exists


**Root cause:** `pg_dump`'s default (plain SQL) output contains `CREATE TABLE` / `ADD CONSTRAINT` statements but no `DROP` statements. Restoring it into a database that already has that schema (i.e., the live, running Wiki.js database) causes every schema-creation statement to fail as "already exists." Because `psql` doesn't halt on error by default, the dump kept running statement-by-statement, but the net effect was that **no data was actually overwritten** — the test page was still present afterward.

**This is the actual finding worth flagging:** a restore procedure that "runs clean" is not proof it restored anything. The Administrator Guide's original restore command (`psql ... < backup.sql` against the live DB) would have given a false sense of confidence in a real incident — operators would see output scroll by and assume it worked.

### 4. Corrected restore procedure — validated working

```bash
# Stop the app so nothing writes mid-restore
docker stop wikijs

# Drop and recreate the database cleanly
docker exec wikijs-db psql -U wikijs -d postgres -c "DROP DATABASE wikijs;"
docker exec wikijs-db psql -U wikijs -d postgres -c "CREATE DATABASE wikijs OWNER wikijs;"

# Restore into the now-empty database
docker exec -i wikijs-db psql -U wikijs -d wikijs < ~/wikijs-backup-test.sql

# Bring the app back up
docker start wikijs
```

Output this time: clean `CREATE TABLE` / `ALTER TABLE` / `CREATE INDEX` statements, zero `ERROR:` lines.

### 5. Verification

After restart and a hard refresh of `http://localhost:3001/en/home`, Wiki.js returned to its pre-backup "Welcome to your wiki! Let's get started and create the home page" state — the test page and its content were gone, confirming the restore replaced live data with the backed-up state.

## Conclusion

- `pg_dump` backups for Wiki.js are valid and capture real data.
- The restore procedure **must** include dropping and recreating the target database first — restoring directly against a live, populated database silently no-ops instead of overwriting.
- Administrator Guide / runbook should be updated (or already reflects, pending review) the corrected 4-step restore procedure above, not a bare `psql < backup.sql`.

## Follow-ups

- Apply the same corrected procedure pattern to Snipe-IT (MariaDB) and Uptime Kuma if/when their backup procedures are documented or tested.
- Consider scripting the stop → drop/recreate → restore → start sequence as a single `restore.sh` to remove the chance of an operator skipping the drop/recreate step under incident pressure.
