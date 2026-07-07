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
   - Reveal banner first (when param present and the vote exists): `you
     preferred {X} / tie / both bad — left was {model_a}, right was
     {model_b}` with a link to the run; invalid/missing vote id → banner
     silently omitted (no error).
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
2. `POST /votes` (CSRF): fields `output_a_id`, `output_b_id`, `winner`,
   `note`, filter passthroughs. Repo `votes.create` computes `blind` and
   enforces §8.2 integrity (`ValueError` → 400 error page; stale double-POST
   of the same pair is *allowed* — it is a re-vote by §8.2). Success → 303
   `/vote?reveal={vote_id}&{filters}`.
3. `vote.js` (progressive enhancement only): keydown `1`→A, `2`→B, `t`→tie,
   `x`→both bad (submits the matching button; ignores keystrokes when focus
   is in the note input); no other behavior. Page fully functional without it.
4. Layout: `.pair-grid` side-by-side ≥ 900px, stacked below (CSS in issue
   22's app.css already; extend there if needed — keep zero inline styles).
5. mypy-strict clean.

## Acceptance Criteria

TestClient with seeded fixtures:

- [ ] GET renders both outputs sanitized (XSS fixture output shows escaped),
      four buttons, hidden ids present.
- [ ] **Leak test**: for a fixture with distinctive model ids/base_urls, the
      full response body contains none of: model ids, `provider`, base_url
      host, `raw_model`, `latency`, token counts (§8.6 list) — while the
      SAME fixture's `/runs/{id}` revealed page does (sanity that the test
      strings are detectable).
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

`uv run pytest tests/test_web_vote.py -q`; manual: full fake-provider voting
session in the browser, keyboard shortcuts included.

## Dependencies

09, 22, 23 (26 for reveal-state integration fixtures).

## Non-goals

Vote history/deletion (29); LLM-judge voting (v2); pair skipping ("skip"
button is v2 — not in DESIGN v1).

## Design References

DESIGN §8.2, §8.4, §8.5, §8.6, §11.2, §12, §13.4.
