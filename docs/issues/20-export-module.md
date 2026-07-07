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

- `src/mybench/export.py`
- `tests/test_export.py`

## Detailed Requirements

1. `def export_json(conn, out_path: Path, *, force: bool = False,
   now: str | None = None) -> None` — document shape (DESIGN §10.9):
   `{"mybench_export": 1, "exported_at": <utc iso>, "tasks": [...],
   "runs": [...], "outputs": [...], "votes": [...]}`; each array = full table
   rows as objects with column names as keys, in id order;
   `model_snapshot_json`/`params_json` embedded as parsed JSON objects (not
   double-encoded strings). UTF-8, `ensure_ascii=False`, indent 2, trailing
   newline.
2. `def export_votes_csv(conn, out_path: Path, *, force: bool = False)
   -> None` — exact header (DESIGN §10.9):
   `vote_id,created_at,blind,task_id,task_title,category,run_id,model_left,
   model_right,winner_model,note`; `winner_model` = the winning model id for
   a/b, literal `tie` / `both_bad` otherwise; `task_title` = display title
   fallback; `csv.writer` with default quoting (QUOTE_MINIMAL handles
   newlines/commas in notes/titles); UTF-8 no BOM.
3. File creation (both): `os.open(path, O_WRONLY|O_CREAT|O_EXCL, 0o600)`
   wrapped in a text IO; when the file exists and not `force` → raise
   `FileExistsError` (CLI in issue 21 maps it to exit 1); with `force` →
   replace atomically (write `path.tmp` 0600 then `os.replace`).
4. Row counts returned? No — return `None`; expose
   `def export_stats(conn) -> dict[str, int]` (counts per table) for the CLI
   summary line instead.
5. No key material can appear by construction (DB never stores it — assert in
   a test that the export text does not contain values of env vars set during
   the test fixture).
6. mypy-strict clean; stdlib only.

## Acceptance Criteria

- [ ] JSON: populated fixture (2 tasks, 2 runs, 4 outputs incl. a failed one,
      3 votes incl. tie/both_bad/nonblind) round-trips `json.loads`; embedded
      params/snapshots are objects; arrays id-ordered; document keys exact.
- [ ] CSV: golden file comparison for the same fixture, including a note
      containing `", \n"` (quoting correctness) and winner_model mapping for
      all four winner values.
- [ ] New files have mode 0600 (both formats, both create and force paths).
- [ ] Existing path without force → `FileExistsError`, file untouched; with
      force → replaced, mode 0600, no `.tmp` residue.
- [ ] `export_stats` counts match fixture.
- [ ] Export content contains no fixture env-var secret values.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_export.py -q`; manual: run against a demo DB and
open the CSV in a spreadsheet app.

## Dependencies

07, 08, 09 (tables + display-title helper).

## Non-goals

Import (v1 non-goal, §2.2); CLI flag surface (21); compression/encryption.

## Design References

DESIGN §10.9, §13.6, §13.7, §1 (data ownership).
