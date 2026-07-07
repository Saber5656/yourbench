# Title

Web: run trigger, run detail w/ blindness rules, status polling, reveal

## Summary

Implement `web/runs_manager.py` (in-process background runs),
`web/routes/runs.py` (`POST /tasks/{id}/runs`, `GET /runs/{id}`,
`GET /runs/{id}/status.json`, `POST /runs/{id}/reveal`),
`templates/run_detail.html`, and `static/runstatus.js` polling.

## Context

DESIGN §11.5–11.6 and §8.5: the run detail page is where blindness is most at
risk — it must show run health without leaking the model↔content mapping
until reveal conditions are met.

## Scope

- `src/mybench/web/runs_manager.py`
- `src/mybench/web/routes/runs.py`
- `src/mybench/web/templates/run_detail.html`
- `src/mybench/web/static/runstatus.js`
- `tests/test_web_runs.py`

## Detailed Requirements

1. `RunsManager` (DESIGN §11.6):
   - `__init__(config, db_path)`; `async start(task_id, model_ids,
     overrides) -> int`: opens its own connection, validates via the runner's
     rules (bad input → `ValueError` propagates to the route → 400
     re-render), schedules `asyncio.create_task(self._execute(...))`, and
     returns the run_id **after** the run rows exist (so the page can render
     immediately). Mechanism: `_execute` calls `execute_run(...)` and the
     manager obtains the id via the engine's `on_created` callback
     (DESIGN §7.1, implemented in issue 13), awaited through an
     `asyncio.Future` resolved by the callback; validation `ValueError`s
     raised before `on_created` fires must reject that Future so `start()`
     re-raises them synchronously to the route.
   - Tracks `self._tasks: dict[int, asyncio.Task]`; done-callback pops the
     entry and logs exceptions at ERROR.
   - `shutdown()`: cancel outstanding tasks (awaited, best-effort);
     registered on app shutdown; §7.4 sweep covers the residue (accepted
     §18 U8).
   - Each `_execute` uses its **own** DB connection (thread=event loop; no
     sharing with request connections).
2. `POST /tasks/{id}/runs` (CSRF): form per issue 25 (`models` multi-value,
   override fields); < 2 selected / archived / unknown → 400 re-render of
   task detail with banner; success → 303 `/runs/{run_id}`.
3. `GET /runs/{id}` — `run_detail.html` per DESIGN §11.5, three visual states:
   - **Running**: per-model status table (model_id, status, latency, tokens,
     error — model names WITH status is allowed; content is NOT shown while
     running), auto-refresh via `runstatus.js` (+ `<noscript><meta
     http-equiv="refresh" content="5"></noscript>` fallback emitted only
     while running).
   - **Completed & blind** (unvoted pairs remain, `revealed_at` NULL):
     model status table WITHOUT content mapping + anonymized output cards
     labeled `Output {output_id}` ordered by output id, content via
     `markdown_safe` + raw `<details>`; `Vote on this run` link →
     `/vote?run={id}`; reveal form (`POST /runs/{id}/reveal`, CSRF) with the
     §11.5 warning text.
   - **Revealed** (`revealed_at` set, or auto: no unvoted pair remains):
     cards show model names + latency/tokens; if auto-condition met and
     `revealed_at` NULL → call `runs.reveal` during GET handling (idempotent).
   - Truncation flag: outputs whose finish_reason ∈ {`length`,`max_tokens`}
     get a visible `truncated` badge (DESIGN §6.1).
   - Failed outputs: error text via `escape_pre`, never markdown (§12).
4. `GET /runs/{id}/status.json`: exactly
   `{"status": str, "pending": int, "succeeded": int, "failed": int,
   "votable": bool}` via `runs.status_counts` — **no model names in the JSON**
   (it is fetched pre-reveal; DESIGN §11.2).
5. `POST /runs/{id}/reveal` (CSRF): `runs.reveal`, 303 back.
6. `runstatus.js`: vanilla JS, no globals leaked (IIFE), reads run id from
   `data-run-id` attribute, polls every 2 s while `status == "running"`,
   `location.reload()` when status changes; stops polling otherwise.
7. mypy-strict clean.

## Acceptance Criteria

TestClient (+ manual async control via fake providers with event gates):

- [ ] POST run: 303 to run page; run + pending outputs exist immediately;
      bad selections → 400 with banner, no rows.
- [ ] status.json shape golden; never contains model ids (string assert on
      raw body).
- [ ] Blind state: page body does NOT contain any model id / provider /
      base_url / raw_model string anywhere in output cards region — but DOES
      contain them in the status table region; card order is output-id order;
      vote link present.
- [ ] Auto-reveal: fixture with all pairs voted → GET flips `revealed_at`,
      cards show model names; second GET stable.
- [ ] Manual reveal POST → non-blind warning honored (subsequent votes get
      blind=0 — cross-check via issue 09 semantics in an integration test).
- [ ] Running state: content absent entirely; noscript refresh present only
      while running; `runstatus.js` served with CSP-compatible `<script src>`.
- [ ] Failed output error with `<script>` renders escaped; truncated badge on
      `finish_reason="length"` and `"max_tokens"`.
- [ ] RunsManager: exception inside a run logged and task removed from
      registry; shutdown cancels a gated run without hanging (≤ 3 s).
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_web_runs.py -q`; manual: trigger a fake-provider
run in the browser, watch polling, verify blind → vote → reveal cycle.

## Dependencies

08, 13, 22, 23, 25 (form origin).

## Non-goals

Vote UI (27), history list (29), streaming, cross-process run visibility
(§18 U5).

## Design References

DESIGN §11.5, §11.6, §8.5, §7, §12, §13.4; §18 U8.
