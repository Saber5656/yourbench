# Review resolution addendum

- Repository: `Saber5656/yourbench`
- Pull request: #33
- Original PR head before this resolution addendum: `154252d9ff1fb25c82cdff8dc7d4aef22725953a`
- Scope: the review findings listed below are converted into normative design contracts and focused verification gates.
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates this review evidence and requires a fresh review.
- This addendum records design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTNj-U86PBnO6` — Add `python-multipart` to the v1 allowlist

**Normative resolution**: The v1 runtime dependency allowlist explicitly includes `python-multipart` before FastAPI/Starlette form routes are implemented, because CSRF, vote, and task forms use request form parsing.

**Focused verification gate**: In a clean environment, install only the allowlisted dependencies and exercise each planned POST form; assert form parsing succeeds and the dependency audit reports no undeclared runtime import.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 2. Thread `PRRT_kwDOTNj-U86PBnPK` — Complete stale runs even when no pending outputs remain

**Normative resolution**: `runs.sweep_orphans` considers stale runs whose outputs are all terminal but whose run row is still `running`, and completes them idempotently; it must distinguish genuinely active work using the configured stale marker/heartbeat rather than only counting pending outputs.

**Focused verification gate**: Simulate a crash after the last output commits and before run completion, plus an actively running long job; assert the former is repaired and the latter is not swept prematurely.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 3. Thread `PRRT_kwDOTNj-U86PBnPQ` — Clarify env-path normalization for `MYBENCH_CONFIG`

**Normative resolution**: An empty `MYBENCH_CONFIG` is treated as unset. A non-empty value is normalized with the shared `Path(value).expanduser().resolve(strict=False)` rule, and the same normalized path semantics are used consistently by `data_dir()` and `db_path()`.

**Focused verification gate**: Test unset, empty, `~`, relative, absolute, and nonexistent values; assert one deterministic resolution order and identical paths across config/data/DB helpers.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 4. Thread `PRRT_kwDOTNj-U86PBnPU` — Keep the atomicity test aligned with the public contract

**Normative resolution**: The public `create_with_outputs()` contract remains `Sequence[tuple[str, str]]`; the rollback test induces the NOT NULL violation through a lower-level malformed-row fixture/direct insert and asserts transaction atomicity without passing `model_id=None` through the public API.

**Focused verification gate**: Run the atomicity test with a valid public call and a controlled lower-level invalid row; assert no partial run/output rows survive on failure.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 5. Thread `PRRT_kwDOTNj-U86PBnPa` — Handle missing adapter modules, not just missing attributes

**Normative resolution**: Provider loading converts both `ModuleNotFoundError` from a missing adapter module and missing exported symbols into `NotImplementedError("provider '<key>' not implemented yet")`, while unrelated import failures remain diagnosable.

**Focused verification gate**: Test absent module, placeholder module without the symbol, and a module with the symbol; assert the first two have the same documented not-implemented outcome.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 6. Thread `PRRT_kwDOTNj-U86PBnPc` — Use the registry symbol from issue 10

**Normative resolution**: All acceptance criteria and integration references use the defined `PROVIDER_SPECS` registry; the undefined `PROVIDERS` name is removed or explicitly aliased only if the alias is part of the public contract.

**Focused verification gate**: Run a repository-wide symbol/reference check and the provider integration test; assert the referenced registry exists and contains the expected provider entries.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 7. Thread `PRRT_kwDOTNj-U86PBnPd` — Treat unimplemented providers as per-output failures

**Normative resolution**: The run engine catches provider-factory `NotImplementedError` for each output, records that output as failed with the documented error, and continues independent outputs; one placeholder adapter cannot abort the whole run.

**Focused verification gate**: Run mixed implemented/unimplemented providers and all-unimplemented providers; assert per-output failure rows, stable completion, and no uncaught exception abort.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 8. Thread `PRRT_kwDOTNj-U86PBnPf` — Replace the undefined timeout placeholder

**Normative resolution**: Timeout error text is generated from the named `settings.timeout_seconds` value (or one canonical named constant), producing the deterministic form `timeout: exceeded <seconds>s`; the undefined `{n}` placeholder is removed.

**Focused verification gate**: Use multiple configured timeout values and assert exact persisted/displayed error strings, including fractional/rounding policy if supported.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 9. Thread `PRRT_kwDOTNj-U86PBnPk` — Defer config/DB startup until `require_context()` runs

**Normative resolution**: The CLI group callback performs only logging and basic path setup. Config loading, redaction installation, DB migration, and orphan sweeping occur behind `require_context()`, and only commands that need shared context call it. `init`, `version`, and `config path` therefore work without a valid config.

**Focused verification gate**: Invoke those config-independent commands with missing/invalid config and an unavailable DB; assert they succeed, while context-dependent commands initialize exactly once through the helper.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 10. Thread `PRRT_kwDOTNj-U86PBnPp` — Flatten provider text before printing single-line progress/summary rows

**Normative resolution**: A shared single-line sanitizer strips terminal controls and replaces newlines, carriage returns, and tabs with spaces before any provider/model/error text is printed in progress or summary rows.

**Focused verification gate**: Feed hostile provider text containing ANSI controls, CR/LF, tabs, and long content; assert every row remains one physical line and preserves the length/redaction policy.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 11. Thread `PRRT_kwDOTNj-U86PBnPv` — Add the missing task-page dependency

**Normative resolution**: The web dashboard issue declares direct dependencies `07, 08, 09, 22, 25`, matching the task-page CTA and its task/run data reads; it does not rely on transitive wording.

**Focused verification gate**: Run the dependency/reference consistency check and verify `/tasks/new` requirements trace directly to all five issue specs.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 12. Thread `PRRT_kwDOTNj-U86PBnrZ` — Delay per-pair reveal until run voting is exhausted

**Normative resolution**: A blind run never reveals model identity after an individual vote while another pair remains. Model names and mapping become visible only when all required pairwise voting is exhausted (or the run is explicitly transitioned to revealed before serving further votes).

**Focused verification gate**: Use three or more outputs and vote through pairs in scheduler order; assert each pre-reveal response contains no mapping and every stored vote is blind until the terminal reveal transition.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 13. Thread `PRRT_kwDOTNj-U86PBnrf` — Remove model ids from fake output content

**Normative resolution**: Fake provider output is deterministic but contains no model/config slug. Model identity is carried only in private server-side metadata and is never rendered through the blind `/vote` response.

**Focused verification gate**: Run fake providers with distinct IDs, fetch the blind vote page/response, and assert neither IDs nor a recoverable mapping appears in visible output or serialized public fields.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 14. Thread `PRRT_kwDOTNj-U86PBnrk` — Hide per-output metadata before reveal

**Normative resolution**: Before reveal, per-output latency, token counts, provider metadata, and any ordering that permits correlation are withheld or aggregated. They are exposed only after the run's reveal transition.

**Focused verification gate**: Inspect blind run HTML/JSON for all metadata fields and correlate adversarially chosen timings/counts; assert no per-output mapping signal is exposed pre-reveal.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 15. Thread `PRRT_kwDOTNj-U86PBnrm` — Allow redirect errors to remain non-retryable

**Normative resolution**: A 3xx redirect is represented as `ProviderError(kind=connection, retryable=False)` (or an explicitly equivalent non-retryable kind), preserving the no-follow policy and one-attempt behavior.

**Focused verification gate**: Return 301/302/307/308 from the adapter and assert exactly one attempt, the documented error kind, and no retry/backoff.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 16. Thread `PRRT_kwDOTNj-U86PBnro` — Add form parsing support before building POST routes

**Normative resolution**: Form parsing support is a prerequisite to CSRF/vote/task POST handlers: add the allowlisted multipart dependency or explicitly specify a compatible URL-encoded parser. The chosen contract must be implemented before routes are accepted.

**Focused verification gate**: Exercise CSRF token, vote, and task form submissions in a clean install; assert parsing, validation, and error responses work without an undeclared package.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 17. Thread `PRRT_kwDOTNj-U86PBnrr` — Permit same-origin status polling in CSP

**Normative resolution**: The CSP for the running page includes `connect-src 'self'` for same-origin status polling, while retaining restrictive defaults and script/resource boundaries.

**Focused verification gate**: Load the running page with JavaScript enabled and assert the status request is permitted, cross-origin requests remain blocked, and the no-script fallback still works.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.

## 18. Thread `PRRT_kwDOTNj-U86PBnrv` — Avoid sweeping legitimate long-running runs

**Normative resolution**: Orphan sweeping uses a heartbeat/lease or a stale threshold derived from configured timeout and run shape; age alone cannot mark a live run failed. Completion is idempotent and refuses to overwrite an active lease.

**Focused verification gate**: Simulate a run older than one hour with a valid heartbeat and timeout up to 3600 seconds, plus a dead process with an expired lease; assert only the dead run is repaired.

**Completion boundary**: this section is a contract for the later implementation/full-validation gate, not evidence that that gate has already passed.