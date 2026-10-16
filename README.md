# statepath — Move a Python app's file state onto Postgres

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/statepath**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

statepath replaces the path object your app already uses, so its read_text, write_text, glob and open calls go to Postgres. You move in steps you can undo, and a verifier tells you when the next step is safe. It also backs the Postgres store up, restores it safely and prunes old backups.

## Get it

Buy statepath at https://dominares.org/tools/statepath: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the statepath command.

Requirements: A Postgres database. psycopg 3 is installed with it.

This page documents version 1.1.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

Move a Python app that keeps its state in files onto Postgres without rewriting its file code, then snapshot, restore and prune it there.

```bash
statepath retain . --keep 3        # a dry run: lists the files it would delete, deletes nothing
```

That last line needs no database. Everything else below does: PostgreSQL 12 or newer, and its
URL in `STATEPATH_DATABASE_URL`.

Lots of Python apps keep their state as JSON files, uploads and log files in a directory. That
works until you need two servers, a host with no persistent disk, or a real backup story. The
usual fix is to rewrite every `open()`, `read_text()` and `json.dump()` to use a database, and
that rewrite is where data gets lost.

statepath swaps the path object instead of the code. `statepath.root("/var/lib/myapp")` returns:

| mode         | you get      | reads from | writes to                  |
|--------------|--------------|------------|----------------------------|
| `filesystem` | `pathlib.Path` | disk     | disk                       |
| `dual`       | `DualPath`   | disk       | disk **and** Postgres      |
| `postgres`   | `StatePath`  | Postgres   | Postgres                   |

Your code keeps calling `read_text`, `write_text`, `exists`, `glob`, `unlink`, `mkdir`, `open`,
`/`, `.parent` and `.with_name` as it does today. You pick the mode with one environment variable,
and you roll back the same way.

What else is in the box:

- **`JsonStore.mutate()`**: a read-modify-write that cannot lose an update. It uses a file lock on
  disk and a Postgres advisory lock in `postgres` mode, so it works across threads, processes and
  servers.
- **Append-only logs** (`append_line`, `read_lines`, `replace_lines`) that store one row per line
  in Postgres, so appending to a 30 MB log does not rewrite 30 MB.
- **Binary files** (uploads, images, PDFs) stored byte for byte.
- **A CLI**: `statepath seed` copies existing files in, `statepath verify` compares the two stores
  by content, `statepath repair` fixes what `verify` flagged, and `statepath drain` copies
  Postgres back to disk (deletes included) so you can leave `postgres` mode without losing writes.
- **Snapshot and restore** (`statepath snapshot`, `restore`, `check`): the Postgres store as one
  `.tar.gz` with checksums, a restore that first saves whatever it would overwrite, and a
  snapshot that fails instead of reporting success when there is nothing in the store.
- **Retention** (`statepath retain`, `statepath.sweep_all()`): keep the newest N backups of each
  kind in a directory, on disk or in Postgres, and report backup directories nobody set a policy for.
- **Soft delete** (`statepath.lifecycle`): two states, a retention window, restore and sweep,
  with the old `deleted` / `archived` flags folded in on read so you need no migration.
- **Dual mode that cannot take your site down.** If Postgres is slow or unreachable, the disk
  write still succeeds, the failure is counted, and a circuit breaker stops retrying for 30
  seconds at a time.

### Who it is for

Developers with a working Python app (Flask, FastAPI, Django, a bot, a worker) that stores state
under a directory and now needs to run on more than one machine, or on a host without a durable
disk. It is not an ORM and it does not make your JSON queryable. It moves the files you already
have into a database, safely, in steps you can undo.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

Installing it adds the `statepath` package, its one dependency (`psycopg[binary]` 3), and the
`statepath` command.

### 60-second quickstart

```python
import json
import statepath

STATE = statepath.root("./data")          # mode comes from STATEPATH_MODE (default: filesystem)

settings = STATE / "settings.json"
settings.parent.mkdir(parents=True, exist_ok=True)
settings.write_text(json.dumps({"theme": "dark"}))

# A read-modify-write that is safe with many workers:
store = statepath.JsonStore(STATE / "counters.json")
with store.mutate(default={}) as s:
    s.value["visits"] = s.value.get("visits", 0) + 1

# An append-only log:
statepath.append_json_line(STATE / "events.jsonl", {"event": "signup"})
print(statepath.read_lines(STATE / "events.jsonl"))
```

Run it once as it is: everything lands in `./data`. Now point it at Postgres and run it again:

```bash
export STATEPATH_DATABASE_URL=postgresql://user:pass@localhost:5432/myapp
statepath init                              # create the state tables
statepath seed --state-dir ./data --apply   # copy what is already on disk
STATEPATH_MODE=postgres python app.py       # same code, now reading and writing Postgres
```

### The rollout

Take it one step at a time. Each step can be undone.

1. **Install and deploy in `filesystem` mode.** Nothing changes. Route your paths through
   `statepath.root()` and your read-modify-writes through `JsonStore.mutate()`.
2. **Seed.** `statepath seed --state-dir DIR` shows what it would copy. Add `--apply` to copy.
   Every file is read back and compared, and the run fails if any file does not match.
3. **Turn on `dual`.** Set `STATEPATH_MODE=dual`. Reads still come from disk; every write is also
   sent to Postgres. Put `statepath.status(STATE)` on your health endpoint.
4. **Verify until it is clean.** Run `statepath verify --state-dir DIR` on a schedule. It exits 0
   only when every file matches by content and Postgres holds nothing the disk does not. Fix
   what it reports:
   - a file missing from Postgres or different there: `statepath repair --state-dir DIR PATH... --apply`
   - a key whose file was deleted on disk (for example by `os.remove` or `shutil.rmtree`, which
     bypass the mirror): `statepath repair --state-dir DIR --delete-stale --apply`. Left in
     place, such a file would come back after the flip.
5. **Flip to `postgres`.** Set `STATEPATH_MODE=postgres`. From now on the disk is no longer
   written.

**Going back from `postgres`.** Drain first, then change the mode, in that order:

```bash
statepath drain --state-dir DIR --mode postgres --apply   # copy Postgres onto disk
export STATEPATH_MODE=dual                                # then switch back
```

If you switch the mode first, every write made since the flip stays in Postgres and your app
stops seeing it. The drain carries deletes back too: an in-scope file on disk whose key is gone
from Postgres is deleted (lock and temp files, and empty log files, are left alone). The dry run
lists what it would write and delete.

Two refusals protect you here. `drain --apply` refuses to run unless you say the app is in
`postgres` mode, because draining while the disk is authoritative would overwrite newer files
with older copies. And if Postgres holds no key at all in the scope while the disk has files, the
drain deletes nothing and exits 1: a wrong `--prefix` looks exactly like that.

### Backups: snapshot, restore, retention

A backup job that tars the state directory keeps "succeeding" after the move to Postgres, because
the directory is empty. These commands read Postgres itself.

```bash
export STATEPATH_DATABASE_URL=postgresql://user:pass@localhost:5432/myapp
statepath snapshot --out backups/state-$(date +%F).tar.gz   # exit 1, no file, if the store is empty
statepath check backups/state-2026-10-04.tar.gz             # checksums only, needs no database
statepath restore backups/state-2026-10-04.tar.gz           # dry run: what would change
statepath restore backups/state-2026-10-04.tar.gz --apply   # write it, saving what it overwrites
```

**What a snapshot is.** One gzip tar. Every key is a member at `data/<key>`: a document's text, a
blob's bytes, a log's lines joined with newlines, so `tar xzf` gives you your files back. One
extra member, `data/.statepath/manifest.json`, records each key's kind (document, log or blob),
its SHA-256, the prefix, the capture time and which keys were redacted. The whole store is read
in one `REPEATABLE READ` read-only transaction, so a snapshot never pairs a document from before
a write with a log from after it. `snapshot` writes the file beside its destination, reads it
back and checks it, and only then renames it into place; it fails (exit 1, nothing left behind)
when the store holds no keys in scope, unless you pass `--allow-empty`. A key whose name is
unsafe (it contains a backslash, or sits under `.statepath/`) cannot be archived. The snapshot is
still written, but it lists those keys on the command line, records them in the manifest as
`skipped_unsafe`, and exits 1 (so does `check` on that file) because it holds less than the store;
`--allow-skipped` (on `snapshot` and on `check`) accepts that.

**What `check` proves.** The archive parses; its manifest is a format this version reads; every key
the manifest names is present and matches its checksum; no member is present that the manifest does
not name; no member has an unsafe name; the snapshot left no key out; and there is at least one
key (`check --allow-empty` and `--allow-skipped` relax the last two). It does not prove the data
is what your app expects.

**What a restore does.**

- It is a dry run unless you pass `--apply`. The dry run says how many keys it would write, how
  many are already identical and how many existing keys it would overwrite (`--json` lists them).
- Before writing, it saves the current content of every key it will change to
  `backups/pre_restore/pre_restore_<stamp>.tar.gz` in the same store (always at the store root,
  whatever `--prefix` you restore under), in the same format. Restoring that copy puts back what
  was overwritten; keys the first restore created stay, because a restore never deletes. Pull it
  from the store with `statepath restore --from-store backups/pre_restore/pre_restore_<stamp>.tar.gz`
  (the restore prints the exact key). If that copy fails or does not read back, nothing is written
  and the exit is 2. A second identical restore changes nothing and makes no copy.
- Keys the archive does not hold are left alone. A restore never deletes.
- Every key is read back after writing; one that does not match is reported and the exit is 1.
- Each key comes back as the kind the manifest says: a restored log is still a log, so the next
  `append_line` adds to it. An archive with no manifest (a tar of a plain state directory) is
  restored using the seeder's rules: a log when the name matches `--log` (default `*.jsonl`,
  `*.ndjson`), a blob when the bytes are not UTF-8, otherwise a document.
- `--prefix P` on `snapshot` archives only keys under `P/` and names the members without it;
  `--prefix` on `restore` writes under a different prefix than the snapshot was taken with.

**Redaction.** `snapshot --redact settings.json=admin_password,api_key` blanks those top-level
JSON fields in every document whose key ends with `settings.json` (in code:
`snapshot_to_file(backend, path, redact=redact_json_fields({...}))`, or any
`(key, bytes) -> bytes` function). A matching document that is not valid JSON ships as `{}`.
Because writing a blank over the live secret would be the data loss, `restore` leaves redacted
keys alone unless you pass `--include-redacted`. A redacted key that does not exist live is the exception: restoring into
an empty store writes the document with those fields blank (and says so), because leaving the
whole document out to protect a secret that is not there would lose everything else in it.

**Retention.** Backups taken before risky writes pile up. `statepath retain DIR --keep N` keeps
the newest N files of each *kind*, not the newest N overall: one chatty writer cannot evict
another writer's only copy. A file's kind is its name with the timestamp removed
(`page.20260805T120000.html` and `page.20260806T130500.html` are one kind). Newest is decided by
modification time (on Postgres, the time of the last write). Add `--max-age-days D` to also delete
anything older, `--postgres` when DIRECTORY is a key prefix in Postgres rather than a path on disk,
and `--apply` to delete (the default is a dry run). The dry run prints the names it would delete
(the first 50; `--json` has all of them).

```python
import statepath

POLICIES = {"pages": 20, "notes": 15, "pre_restore": 10}      # directory -> how many of each kind
statepath.sweep_all(BACKUPS, POLICIES, max_age_days={"pre_restore": 30})   # Path or StatePath
statepath.unmanaged(BACKUPS, POLICIES, exempt={"receipts": "audit record"})  # -> ["new_kind"]
statepath.retain_after_write(path_just_written, POLICIES)     # prune at the write site
```

`unmanaged()` names directories under the backups root that have no policy and are not exempt, so
a new kind of backup cannot quietly grow forever; put it in a health check or a test. Deletion is
best-effort and never raises: what could not be removed is counted in `errors`.

### Soft delete: `statepath.lifecycle`

One answer to "make this go away" for records: two states, `active` and `deleted`, a restore, a
retention window (30 days by default) and a sweep. It is pure: no I/O, and the clock is a
parameter.

```python
from statepath import lifecycle

row = {"id": 7}
lifecycle.apply_state(row, "deleted", actor="sam")   # sets lifecycle, lifecycle_at_utc, lifecycle_by
lifecycle.state_of(row)                              # "deleted"
lifecycle.days_left(row)                             # 30 (rounded up, so it opens at the promised number)
lifecycle.filter_view(rows, "deleted")               # the Deleted list; unknown views show Active
lifecycle.restore(row)                               # the one undo
kept, expired = lifecycle.sweep(rows, retention_days=30)
```

Records written before you adopted it need no migration: `state_of` reads a `deleted` flag as
deleted, an `archived` flag as active, and `lifecycle: "done"` or `"put_away"` as active. A
deleted record with no readable timestamp is never swept. `deleted` is still written as a
mirror (for code you have not converted); `archived` is never written.

`statepath sweep-deleted FILE [--rows-key K] [--backup-dir DIR] [--days N] [--apply]` (or
`statepath.sweep_deleted(path, ...)`) removes expired records from a JSON file of records, as a
locked read-modify-write. With a backup directory the removed records are copied there first and
**nothing is removed if the copy fails**. The copy is named
`<file name>.<date>T<time>Z_<microseconds>.<8 hex of the file's path>.json`, so sweeping many
files that share a name (one `records.json` per account) into one backup folder keeps every copy.
`--days` must be at least 1.

### Configuration reference

#### Environment variables

| variable | meaning | default |
|---|---|---|
| `STATEPATH_MODE` | `filesystem`, `dual` or `postgres`. An unknown value falls back to `filesystem` and logs a warning. | `filesystem` |
| `STATEPATH_DATABASE_URL` | Postgres connection string for the app and the CLI. `DATABASE_URL` is used if this is unset. In `postgres` mode a missing URL is an error. | none |
| `STATEPATH_STATEMENT_TIMEOUT_MS` | statement timeout in `postgres` mode. | `30000` |
| `STATEPATH_STATE_DIR` | default `--state-dir` for the CLI. | none |
| `STATEPATH_TEST_DATABASE_URL` | the only database the test suite may empty. Never read by the app. | none |

Timeouts: `dual` mode connects with a 2 s timeout and a 3 s statement timeout, because the
disk is authoritative and a slow mirror must not hold a request. `postgres` mode waits up to
10 s to connect and bounds statements at `STATEPATH_STATEMENT_TIMEOUT_MS`. The CLI runs with no
statement timeout, because seeding a large log is not request work. All connections use TCP
keepalives, so a dead peer is detected in about a minute.

#### `statepath.root(path, *, key="", mode=None, backend=None)`

- `path`: the directory your app keeps state in.
- `key`: the key prefix this directory maps to in Postgres. Use one prefix per app or customer
  when they share a database. Default: no prefix.
- `mode`: overrides `STATEPATH_MODE`. An invalid value raises `ValueError`.
- `backend`: a `PgBackend` to use instead of the one built from `STATEPATH_DATABASE_URL`. Call
  `backend.ensure_schema()` once yourself.

When the database cannot be used, the two modes behave differently on purpose:

- **`dual`** falls back to a plain `Path` and records why in `statepath.status()` (no URL set, or
  Postgres unreachable). The files are still the truth, so the app keeps working; nothing is
  mirrored. The fallback lasts as long as your app keeps that `Path`. Calling `root()` again after
  a 60-second cooldown picks a recovered database up, so either call `root()` per request (it is
  cheap: the connection is cached per process) or restart the app once Postgres is back.
  `verify` shows the writes the mirror missed, and `repair` copies them.
- **`postgres`** raises: `RuntimeError` when no URL is set, the connection error when Postgres is
  unreachable. Answering from a stale disk would be worse than not starting.

#### `statepath.status(path=None)`

A dict for your health endpoint: `requested` (the mode asked for), `active` (what `path` really
is), `degraded` (why a configured database is not in use) and `shadow_failures` (failed mirror
writes by operation, plus `_circuit_open_for_seconds` while the breaker is open). The last two
are empty on a healthy process.

#### What `StatePath` supports

`/`, `joinpath`, `name`, `stem`, `suffix`, `parent`, `parents`, `with_name`, `with_suffix`,
`with_stem`, `resolve`, `relative_to` (returns a `PurePosixPath`), `is_relative_to`, `match`,
`as_posix`, `exists`, `is_file`, `is_dir`, `is_symlink`, `read_text`, `read_bytes`, `write_text`,
`write_bytes`, `open` (`r`, `rb`, `w`, `wb`), `mkdir`, `unlink`, `touch`, `rename`/`replace`,
`iterdir`, `glob`, `rglob`, `stat` (`st_size`, `st_mtime`, `st_mode`), ordering and hashing.

A conformance suite runs these operations against a real `Path` in a temp directory and a
`StatePath` on a real Postgres and asserts the two agree, exceptions included (a missing file
raises `FileNotFoundError`, a directory read raises `IsADirectoryError`, and so on). A file
written with `write_bytes` or `open("wb")` reads back with `read_text` like any file, and CRLF
text reads with universal newlines while `read_bytes` returns the exact bytes.

Where it differs from `Path`, on purpose:

- Directories are key prefixes. A directory exists while something is stored under it, so an
  empty directory made with `mkdir` does not exist, and `iterdir()` on it yields nothing.
  `mkdir(exist_ok=True)` never queries the database (it runs before every save).
- `str(p)` and `p.parts` are the key (`sessions/a.json`), with no leading `/`.
- `stat()` on a directory reports size 0 and mtime 0.
- `open()` supports `r`, `rb`, `w` and `wb`; append and `+` modes raise (see the limits below).
  A `w` handle stores the file when it closes, and stores nothing if the `with` block raises.
- Anything else on `pathlib.Path` (`chmod`, `symlink_to`, `owner`, `samefile`, ...) is not there
  and raises `AttributeError`.

#### Scopes: what `seed`, `verify`, `repair` and `drain` cover

By default the whole state directory is in scope. CLI flags (or `statepath.Scope(...)` in code):

| flag | `Scope` field | meaning |
|---|---|---|
| `--dir NAME` (repeatable) | `dirs` | only these top-level directories |
| `--root-file NAME` (repeatable) | `root_files` | also these top-level files, when `--dir` is used |
| `--exclude PATTERN` (repeatable) | `exclude` | skip paths matching this glob, e.g. `cache/*` |
| `--log PATTERN` (repeatable) | `log_patterns` | file names that are append-only logs (default `*.jsonl`, `*.ndjson`) |
| `--prefix P` | `prefix` | the key prefix in Postgres, matching `root(..., key=P)` |

`--database-url URL` works before or after the command name. `repair` also takes
`--delete-stale` (delete keys whose file is gone from disk: the named paths, or every such key in
scope when you name none) and, like `drain`, `--mode`: `repair --apply` refuses to run in
`postgres` mode, where the files on disk are the stale copy.

`*.lock` and `*.tmp` files are never copied. Files that are not valid UTF-8 are stored as
binary. **Get `--log` right:** a real append-only log seeded as a whole document loses its
history on the next append. The test suite demonstrates this.

#### CLI exit codes

| code | meaning |
|---|---|
| 0 | every file matches, or the run finished and verified |
| 1 | drift (missing, different, or only in Postgres), a run that did not verify, or a drain that refused to delete |
| 2 | could not check: no database URL, no directory, Postgres unreachable, or a refusal (a drain, a repair, or a restore whose archive is damaged or whose safety copy could not be saved) |

Every command that writes is a dry run unless you pass `--apply`. `--json` prints the result as
JSON.

### Running the tests

```bash
cd statepath
python -m pytest
```

The tests use a real Postgres. If `initdb` and `pg_ctl` are on your PATH (they ship with the
PostgreSQL server package), the suite starts a throwaway cluster on a free local port and deletes
it at the end. On Debian and Ubuntu those two programs are not on PATH by default; run
`export PATH=/usr/lib/postgresql/*/bin:$PATH` first (with your version number in place of `*` if
several are installed). The test database uses the `en_US.UTF-8` collation when your system has
it, because that is the collation where a missing prefix index shows up.

Or set `STATEPATH_TEST_DATABASE_URL` to a scratch database. The suite empties the state tables,
so it refuses to start unless that database's name contains `test`.

### Honest limits

- **Not a query layer.** Documents are stored as text, byte for byte. Nothing indexes the JSON
  inside them. If you need to query fields, add your own `jsonb` generated columns or tables.
- **Whole-document writes.** Every `write_text` sends the whole document. That is fine for
  settings and indexes up to a few MB. It is slow for a 50 MB JSON file you rewrite on every
  request. Split such a file, or turn it into an append-only log.
- **`open("a")` on a `StatePath` raises.** Use `statepath.append_line()`. On a `DualPath`,
  `open("a")` works on the file but is not mirrored. `open("w")` is mirrored when the handle
  closes.
- **A log file must end with a newline** to round-trip. Seeding a log whose last line has no
  newline is reported as a mismatch rather than stored silently. Appending to a file that was
  stored whole (seeded without the right `--log` pattern, or rewritten with `write_text`) converts
  it to log rows first, once, so no history is lost; `--log` still matters for speed on large logs.
- **`glob` on a `StatePath`** follows `pathlib`'s rules for `*`, `?`, `[...]`, directory parts
  (`sessions/*.json`) and `**` (`**/*.json`, `a/**/b.json`). A pattern ending in `**` (whose
  meaning changed in Python 3.13), `..` and absolute patterns raise `NotImplementedError` rather
  than return a set that might be wrong. Results are sorted.
- **Code that calls `Path(state_path)`, `os.fspath()` or `shutil` on a `StatePath` raises
  `TypeError`.** That is deliberate: the alternative is a silent write to local disk. Use
  `statepath.path_like()` where you normalise paths, and `copy_file_into_state()` instead of
  `shutil.copy2`.
- **Locks are advisory.** They only exclude code that takes them: `JsonStore.mutate()`,
  `locked_store()` and `interprocess_lock()`.
- **One connection per thread, no pool.** This suits apps with a modest number of worker
  threads. Put PgBouncer or your host's pooler in front if you run hundreds.
- **Archives are held in memory.** The snapshot is written to disk as a stream, but `snapshot` then
  reads the whole finished file back to check it, and `restore` and `check` load the whole archive
  and unpack every member. Peak memory is about the archive plus its unpacked content. Fine for tens
  of MB; not tested beyond that.
- **A tar of a plain directory (no manifest)** is read by its names: every file under one shared
  top folder (`tar czf x.tgz statedir`) loses that folder; `tar czf x.tgz -C statedir .` keeps
  every name as it is. Anything else keeps its names as they are.
- **A log with no lines is not in the store**, so it is not in the snapshot and a restore cannot
  bring it back as an empty file.
- **A snapshot is a point in time, not a replication stream.** Writes after the read are not in it.
  Schedule `snapshot`, then `check`, and copy the file off the database host.
- **Retention judges "newest" by modification time.** On Postgres that is the time of the last
  write to the key, so a `restore` that rewrites a backup folder resets it. Names that start with
  a `YYYYMMDDTHHMMSS` stamp are aged by that stamp in `--max-age-days`.
- **`sweep-deleted` knows one shape**: a JSON file holding a list of records, or an object holding
  that list under one key. Records elsewhere (a database table, many files) use
  `lifecycle.sweep()` and your own writes.
- **Postgres is the only database supported.**

---

Docs mirror of the private repository Dominares-Tools/statepath; updated with each release.
