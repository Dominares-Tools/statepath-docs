# Changelog

## 1.1.1

Fixes from a review of 1.1.0. Nothing is removed from the API; the reports of `retain()` and
`retain_by_age()` gain a `files` list.

- `sweep_deleted()`: the copy of the removed records is now named with microseconds and a hash of
  the file's path, and never overwrites an existing copy. Before, sweeping several files with the
  same name into one backup folder kept the last copy only while reporting success for all.
- `restore`: an archive with no manifest is read by its names. `tar czf x.tgz -C dir .` used to
  restore its keys as `./a.json` (the app saw nothing), and a flat tar that started with a folder
  lost that folder from one key. A top folder is now stripped only when every file shares it.
- `retain`: the dry run (and the report, and `--json`) names the files it would delete. A dry run
  with `--max-age-days` no longer counts a file twice.
- `snapshot`: lists keys it had to leave out (unsafe names), exits 1 for that unless
  `--allow-skipped`; `check` reports them as a problem unless `--allow-skipped`.
- `restore --from-store KEY`: restore from a snapshot held in the store, such as the safety copy.
- `restore`: exits 2 (a refusal) when the safety copy cannot be saved. A redacted key that does not
  exist live is restored with its fields blank instead of being dropped.
- `check --allow-empty`; `sweep-deleted --days` below 1 is refused.
- README: the limits now say that `snapshot` reads its finished file back into memory, and what
  restoring the safety copy does and does not undo.

## 1.1.0

Backups. The 1.0 line could move state onto Postgres but had no way to back it up, restore it
or stop old backups piling up. Existing behaviour is unchanged.

- `statepath snapshot` / `statepath.snapshot_to_file()`: the Postgres store as one `.tar.gz`
  (members at `data/<key>`, plus a manifest with each key's kind and SHA-256), read in one
  `REPEATABLE READ` transaction, written beside its destination, read back and checked before it is
  renamed into place. A store with nothing in scope is an error (`EmptySnapshot`, exit 1), not a
  success, unless `--allow-empty`. `--prefix`, `--redact SUFFIX=FIELD,FIELD`.
- `statepath restore` / `restore_state_archive()`: a dry run unless `--apply`; saves what it
  overwrites to `backups/pre_restore/pre_restore_<stamp>.tar.gz` first and refuses to write if that
  copy fails; restores each key as the kind the manifest records (a log stays a log); never
  deletes; reads every key back. Redacted keys are left alone unless `--include-redacted`. An
  archive without a manifest restores by the seeder's rules (`--log PATTERN`).
- `statepath check` / `check_archive()`: verifies a snapshot's format, checksums and member names
  without a database.
- `statepath retain` and `retain()`, `retain_by_age()`, `sweep_all()`, `retain_dir()`,
  `retain_after_write()`, `unmanaged()`: keep the newest N backups of each kind (a name with its
  timestamp removed), optionally cap by age, on a directory, a `DualPath` or a `StatePath`. Policies
  are arguments, not a built-in table.
- `statepath.lifecycle`: two record states (`active`, `deleted`), restore, views, counts, a
  retention window and sweep, with legacy `deleted` / `archived` flags and `done` / `put_away`
  folded in on read. Pure; the window is a parameter.
- `statepath sweep-deleted` and `sweep_deleted()`: remove expired records from a JSON file of
  records under a lock, copying them first and removing nothing if the copy fails.
- CHANGELOG entries for 1.0.1 no longer name the publisher, so the brand-free test passes.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show statepath`, `statepath.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated to version 1.3.
- One public support address in the licence and the README.

## 1.0.0

First release.

- `statepath.root()`: one call that returns a `Path`, `DualPath` or `StatePath` for the mode set
  in `STATEPATH_MODE` (`filesystem`, `dual`, `postgres`).
- `StatePath`: a `pathlib.Path` stand-in backed by Postgres, checked against real `pathlib` by a
  conformance suite. Supports text and binary files, `open()` for reading and whole-file
  writing, `glob`/`rglob`/`iterdir`, `stat`, `rename` and ordering.
- `DualPath`: writes to disk and Postgres, reads from disk, with a circuit breaker so a slow or
  dead database never breaks a request. Failed mirror writes are counted in
  `shadow_failures()`.
- `JsonStore.mutate()`: lost-update-proof read-modify-write across threads, processes and
  servers (file locks on disk, advisory locks on Postgres), with nested spans joined.
- `locked_store()`, `interprocess_lock()`, `lock_path_for()`: the lock primitives.
- Append-only logs (`append_line`, `append_json_line`, `read_lines`, `replace_lines`) stored as
  one row per line, listed through an indexed registry so a large log does not slow directory
  listings. Appending to a file stored whole converts it to log rows first.
- Atomic writes (`atomic_write_text`, `atomic_write_json`, `atomic_write_bytes`) and
  `copy_file_into_state()`.
- `statepath` CLI: `init`, `seed`, `verify`, `repair` and `drain`, with dry runs by default and
  exit codes 0 / 1 / 2 for ok / drift / could not check. `verify` fails on keys whose file was
  deleted on disk, `repair --delete-stale` removes them, and `drain` carries deletes back to disk
  (and refuses to delete anything when Postgres holds no key in scope).
- `status()` for health endpoints: requested vs active mode, start-up degrade reason, mirror
  failures.
- Test suite that runs against a real, throwaway Postgres and refuses to run against a database
  that does not look disposable.
