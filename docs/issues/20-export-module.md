# Title

Export module (JSON/CSV, 0600, no-overwrite)

## Summary

Implement `mybench/export.py`: full-database JSON export and flat votes CSV
export with secure file creation, refusing to overwrite without force.

## Context

DESIGN §10.9 (formats), §13.6/§13.7 (sensitivity + permissions). Data
ownership is a design value (§1): the user must always be able to get
everything out in open formats.

## Scope

- `src/mybench/export.py` (formatting + file writing only)
- `src/mybench/db/__init__.py` (add the read-only `dump_all(conn) ->
  dict[str, list[dict]]` helper of DESIGN §4.4 — full rows of the four §4.3
  tables in id order, column names as keys)
- `tests/test_export.py`

All public functions take `conn: sqlite3.Connection` as their first
parameter (typed).

## Detailed Requirements

1. `def export_json(conn: sqlite3.Connection, out_path: Path, *,
   force: bool = False, now: str | None = None) -> None` — reads via
   `db.dump_all` (no SQL in this module); document shape (DESIGN §10.9):
   `{"mybench_export": 1, "exported_at": <utc iso>, "tasks": [...],
   "runs": [...], "outputs": [...], "votes": [...]}`; each array = full table
   rows as objects with column names as keys, in id order;
   `model_snapshot_json`/`params_json` embedded as parsed JSON objects (not
   double-encoded strings). UTF-8, `ensure_ascii=False`, indent 2, trailing
   newline.
2. `def export_votes_csv(conn: sqlite3.Connection, out_path: Path, *,
   force: bool = False) -> None` — exact header (DESIGN §10.9):
   `vote_id,created_at,blind,task_id,task_title,category,run_id,model_left,
   model_right,winner_model,note`; `winner_model` = the winning model id for
   a/b, literal `tie` / `both_bad` otherwise; `task_title` = the issue-07
   `Task.display_title` rule (reuse via `votes.list_votes` join data or a
   shared helper — do not re-derive a different fallback); `csv.writer` with
   default quoting (QUOTE_MINIMAL handles newlines/commas in notes/titles);
   UTF-8 no BOM.
3. File creation (both): `os.open(path, O_WRONLY|O_CREAT|O_EXCL, 0o600)`
   wrapped in a text IO; when the file exists and not `force` → raise
   `FileExistsError` (CLI in issue 21 maps it to exit 1); with `force` →
   replace atomically (write `path.tmp` 0600 then `os.replace`).
4. `def export_stats(conn: sqlite3.Connection) -> dict[str, int]` — keys
   exactly `tasks`, `runs`, `outputs`, `votes` (DESIGN §10.9); backs issue
   21's summary line. Also export the module constant
   `EXPORT_SENSITIVITY_NOTE = "note: export contains your prompt and output
   text — handle accordingly"` for issue 21 (single source of the §13.6
   reminder).
5. No key material can appear by construction (DB never stores it — assert in
   a test that the export text does not contain values of env vars set during
   the test fixture).
6. mypy-strict clean; stdlib only.

## Acceptance Criteria

- [ ] JSON: populated fixture (2 tasks, 2 runs, 4 outputs incl. a failed
      one, 4 votes covering all four winner values incl. a non-blind one)
      round-trips `json.loads`; embedded params/snapshots are objects;
      arrays id-ordered; document keys exact.
- [ ] CSV: golden file comparison for the same fixture, including a note
      containing `", \n"` (quoting correctness), a NULL-title task using the
      display-title fallback, and winner_model mapping for all four winner
      values.
- [ ] New files have mode 0600 (both formats, both create and force paths).
- [ ] Existing path without force → `FileExistsError`, file untouched; with
      force → replaced, mode 0600, no `.tmp` residue.
- [ ] `export_stats` counts match fixture.
- [ ] Export content contains no fixture env-var secret values.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_export.py -q`; optional manual QA (non-gating): run against a demo DB and
open the CSV in a spreadsheet app.

## Dependencies

07, 08, 09 (tables + display-title helper).

## Non-goals

Import (v1 non-goal, §2.2); CLI flag surface (21); compression/encryption.

## Design References

DESIGN §10.9, §13.6, §13.7, §1 (data ownership).
