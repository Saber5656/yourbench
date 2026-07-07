# ADR-001: Rename the project from `yourbench` to `mybench`

Date: 2026-07-08
Status: Accepted

## Context

The repository was created as `yourbench`. Hugging Face ships a widely known
OSS project of the exact same name (github.com/huggingface/yourbench, paper at
COLM 2025) in the adjacent problem space "evaluate LLMs on your own data".
Publishing a second `yourbench` guarantees confusion. Evidence and alternatives
are recorded in `docs/research/prior-art-and-naming.md`.

## Decision

- Product, repository, PyPI package, and CLI command are all named **`mybench`**.
- One name everywhere: `pip install mybench` / `uv tool install mybench`
  installs the `mybench` console script; the import package is `mybench`.
- The GitHub repository `Saber5656/yourbench` is renamed to `Saber5656/mybench`
  (owner action — requires repo Administration permission; GitHub redirects the
  old URL automatically).

## Consequences

- Docs, code, and packaging use `mybench` exclusively from day one; no
  transition aliases needed because nothing was released under the old name.
- Accepted trade-offs: generic name with weak SEO; small dormant/unrelated
  same-name projects exist (Shopify/mybench: MySQL benchmarking; mybench.io:
  recruiting). PyPI name availability verified 2026-07-08.
- The local checkout directory should be renamed by the owner
  (`mv ~/dev/yourbench ~/dev/mybench`) after the current work session.
