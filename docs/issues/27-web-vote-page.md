# Title

Web: blind voting page + vote POST + reveal banner

## Summary

Implement `web/routes/vote.py` + `templates/vote.html` / `vote_empty.html` +
`static/vote.js`: the core blind A/B experience — side-by-side anonymized
outputs, four-way vote with optional note, keyboard shortcuts, reveal-after-
vote banner, and filter passthrough.

## Context

This page is the product's reason to exist (DESIGN §8.4) and its most
security- and integrity-sensitive surface: it renders two untrusted model
outputs (B2) and must leak nothing that identifies them (§8.6).

## Scope

- `src/mybench/web/routes/vote.py`
- `src/mybench/web/templates/vote.html`, `vote_empty.html`
- `src/mybench/web/static/vote.js`
- `tests/test_web_vote.py`

## Detailed Requirements

1. `GET /vote` (filters `?category=`, `?task=`, `?run=`, optional
   `?reveal={vote_id}`):
   - Reveal banner first (when param present and the vote exists), rendered
     in an element with the stable attribute `id="reveal-banner"` (issue 30
     strips it by id for leak checks): `you preferred {X} / tie / both bad
     — left was {model_a}, right was {model_b}` with a link to the run;
     invalid/missing vote id → banner silently omitted (no error).
   - Next pair via `votes.next_pair(conn, ...)` with filters; None →
     `vote_empty.html` (message + links: run a task / view leaderboard;
     when filtered, offer clearing filters).
   - **Left/right coin flip in the route** (`random.random() < 0.5`) mapping
     `{lo,hi}` → displayed A(left)/B(right) (DESIGN §8.4).
   - Render: task display title + collapsible task prompt (`<details>`,
     `markdown_safe`); two cards `Response A` / `Response B` (`markdown_safe`
     + raw `<details><pre>`); four submit buttons (one form, `winner` values
     `a`/`b`/`tie`/`both_bad`); optional `note` text input; hidden fields
     `output_a_id`, `output_b_id`, `csrf_token`, and the current filters (so
     the redirect keeps them).
   - Blindness (§8.6): response must not contain model ids/provider/
     base_url/raw_model/latency/token counts for the pair (task metadata is
     fine). Enforced by test.
2. `POST /votes` (CSRF via `Depends(csrf_protect)`): fields `output_a_id`,
   `output_b_id`, `winner`, `note`, filter passthroughs. The route parses
   the ids as ints (unparsable → 400 error page), loads both outputs,
   verifies they share a run, and derives `run_id` from them; then calls
   `votes.create(conn, run_id=..., output_a_id=..., output_b_id=...,
   winner=..., note=...)` — the repo computes `blind` in-transaction and
   enforces §8.2 integrity (`ValueError` → 400 **error page** per DESIGN
   §11.2: tampered hidden fields are not user typos; stale double-POST of
   the same pair is *allowed* — a re-vote by §8.2). Success → 303
   `/vote?reveal={vote_id}&{filters}`.
3. `vote.js` (progressive enhancement only): keydown `1`→A, `2`→B, `t`→tie,
   `x`→both bad (submits the matching button; ignores keystrokes when focus
   is in the note input); no other behavior. Page fully functional without it.
4. Layout: uses `.pair-grid` from issue 22's `app.css` (side-by-side ≥
   900px, stacked below). This issue does not modify `app.css`; if a style
   gap emerges, it is an issue-22 follow-up, not scope creep here. Zero
   inline styles.
5. mypy-strict clean.

## Acceptance Criteria

TestClient with seeded fixtures:

- [ ] GET renders both outputs sanitized (XSS fixture output shows escaped),
      four buttons, hidden ids present; raw `<details><pre>` view is
      escaped (no raw `<script` from the fixture).
- [ ] **Leak test with distinctive fixture values** (§8.6): seed model ids
      like `leakcheck-alpha`, base_url host `leakhost.example`, raw_model
      `raw-leak-1`, latency `31337`, completion tokens `4242`; the full
      `/vote` response body contains none of those strings — while the SAME
      fixture's revealed `/runs/{id}` page does (proves the strings are
      detectable).
- [ ] CSRF negatives on `POST /votes`: missing cookie, missing field,
      mismatched token, hostile Origin → each 403 and no vote row.
- [ ] Left/right randomization: with patched RNG both orders occur; hidden
      `output_a_id` matches what is displayed left (DOM order assertion).
- [ ] POST happy path: 303 with reveal param + filters preserved; vote row
      winner/blind correct; reveal banner then names both models and links
      the run.
- [ ] POST after run reveal → vote stored `blind=0` (integration with 08/09).
- [ ] POST integrity violations (cross-run ids, failed output id, bad winner,
      oversized note) → 400, no row.
- [ ] Filters: `?run=` limits pairs to that run; `?category=` respected;
      empty-state page for exhausted filters with clear-filter link.
- [ ] Least-voted scheduling respected across sequential GET/POST cycles
      (vote all pairs once → each pair served exactly once before repeats).
- [ ] `vote.js` served via `<script src>`; page contains no inline handlers.
- [ ] ruff, mypy strict, pytest green (leak + render tests count toward the
      §15 95% gate for this route's module).

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_web_vote.py -q`; optional manual QA (non-gating): full fake-provider voting
session in the browser, keyboard shortcuts included.

## Dependencies

09, 22, 23. (Reveal-state fixtures come from the 08/09 repos directly —
`runs.reveal` + `votes.create` — not from issue 26's pages; cross-page flow
is covered by issue 30.)

## Non-goals

Vote history/deletion (29); LLM-judge voting (v2); pair skipping ("skip"
button is v2 — not in DESIGN v1).

## Design References

DESIGN §8.2, §8.4, §8.5, §8.6, §11.2, §12, §13.4.
