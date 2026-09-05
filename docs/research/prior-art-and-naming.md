# Research: Prior Art and Project Naming

Date: 2026-07-08
Status: informs ADR-001 (rename), DESIGN.md §1–§2

## 1. Why this research exists

The repository was created as `yourbench` ("A private leaderboard for blind A/B
testing LLMs on your own real tasks"). Before freezing requirements we checked
whether the name and the product concept collide with existing projects, because
this repository is intended to become a public OSS project.

## 2. Name collision: Hugging Face `yourbench`

Hugging Face publishes an established OSS project with the exact same name:

| Property | Value |
|---|---|
| Repository | <https://github.com/huggingface/yourbench> |
| Tagline | "Benchmark Large Language Models Reliably On Your Data" |
| Paper | "YourBench: Easy Custom Evaluation Sets for Everyone" (arXiv:2504.01833, COLM 2025) |
| HF org | <https://huggingface.co/yourbench> (Spaces, datasets, blog posts) |
| Function | Generates synthetic evaluation sets (QA benchmarks) from user documents; automated scoring |

The two products occupy the same conceptual territory ("evaluate LLMs on *your*
data/tasks") while doing different things. Keeping the name guarantees
confusion in search results, package installs, and issue reports.

## 3. Candidate replacement: `mybench` (user-proposed)

Availability evidence collected 2026-07-08:

| Check | Method | Result |
|---|---|---|
| PyPI `mybench` | `GET https://pypi.org/pypi/mybench/json` | **404 — available** |
| PyPI `my-bench`, `taskarena`, `localarena`, `myarena`, `ownbench` | same | all 404 — available |
| GitHub prominent repos | `gh search repos mybench --sort stars` | Largest is `Shopify/mybench` (10 stars, Go framework for **MySQL database** benchmarks). No LLM-related project. |
| Other web presence | web search `"mybench" software tool` | `mybench.io` (tech-recruiting service, unrelated domain), `jeremy.zawodny.com/mysql/mybench` (dormant Perl MySQL benchmark from early 2000s), SourceForge `myBench` (dormant) |

Assessment:

- No dominant same-name OSS project exists; the PyPI name is free, so
  repository name = PyPI package name = CLI command name can all be `mybench`.
- Residual weaknesses (accepted): the name is generic (weak SEO) and says
  "bench" while the core mechanism is an arena (blind pairwise voting). The
  product's *output* is a personal benchmark/leaderboard, so the name is not
  misleading.
- Decision recorded in ADR-001.

## 4. Prior art (product mechanics)

| Project | What it does | Relevance / difference |
|---|---|---|
| LMArena (Chatbot Arena) | Public crowdsourced blind pairwise battles; Bradley-Terry leaderboard (arXiv:2403.04132) | Direct inspiration for the voting/rating mechanics. Public, crowd-based, not runnable on private tasks. |
| Hugging Face YourBench | Generates synthetic QA benchmarks from user documents; automated grading | Same "personal evaluation" goal, but synthetic-question + auto-scored. mybench uses the user's *actual prompts* and *human blind preference*. |
| promptfoo | Declarative eval harness; side-by-side model comparison; assertions | Test-suite style, config-driven, assertion-scored. No blind voting, no persistent preference leaderboard. |
| OpenAI Evals / lm-evaluation-harness | Public benchmark harnesses | Static datasets, automated metrics; not personal-task oriented. |
| `llm` CLI (simonw) | Run prompts against many models, log to SQLite | Overlaps with the "run one prompt against N models" step only; no blind comparison or rating layer. |

Gap mybench fills: **persistent, private, human-preference leaderboard built
from the user's own real prompts, with blindness enforced by the tool**.

## 5. Sources

- <https://github.com/huggingface/yourbench>
- <https://arxiv.org/abs/2504.01833> (YourBench paper)
- <https://arxiv.org/abs/2403.04132> (Chatbot Arena paper)
- <https://github.com/Shopify/mybench>
- <http://jeremy.zawodny.com/mysql/mybench/>
- <https://mybench.io/>
- PyPI JSON API checks (see table in §3), run 2026-07-08
