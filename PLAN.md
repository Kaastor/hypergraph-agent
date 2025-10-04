Yes—it sounds great. Below is a **complete, pick‑up‑and‑build implementation plan** that turns your PoC v2 + Composability Add‑On into a working, test‑driven, modular system. It’s organized as a sequence of **small, non‑breaking steps**. Every step has: goal, why now, files to touch, code/testing notes, acceptance criteria, and a tiny “leave‑trail” task so you can stop and later resume smoothly.

> **How to use this plan**
>
> * Keep a rolling `NEXT.md` at the repo root. At the **end of each session**, update the “Next Action” line in the current step or the next one.
> * Work step‑by‑step. Each step extends but doesn’t break prior logic; CI stays green throughout.
> * If new requirements appear, add them as **feature flags** or a new module/service so the base remains stable.

---

## 0) One‑time bootstrap

### 0.1 Repo, tooling, CI skeleton

**Goal.** Create baseline project with formatting, linting, typing, tests, and containers.

**Why now.** Ensures safety, determinism, and no spaghetti code. Everything after this inherits guardrails.

**Create/modify**

```
poc/
  app/ __init__.py
  tests/ __init__.py
  scripts/ reproduce.sh
  docker/ Dockerfile, docker-compose.yml
  Makefile
  pyproject.toml (or requirements.txt + constraints.txt)
  .pre-commit-config.yaml
  .github/workflows/ci.yml   # or your CI provider
  README.md
  NEXT.md                    # running notebook for you
```

**Choices**

* **Python** 3.11+, **FastAPI**, **Typer**, **Pydantic v2**, **NetworkX**, **rank_bm25**, optional **faiss-cpu**, **numpy**/**scipy**.
* **Quality**: `black`, `ruff`, `mypy` (strict on core modules), `pytest` + `hypothesis`, `coverage`.
* **Logs/metrics**: `structlog` (JSON Lines), simple CSV counters first.
* **DB**: `sqlite3` (built‑in) or `SQLAlchemy Core`; for graph we’ll still **persist to SQLite** but also **emit a canonical JSON snapshot** for commits.

**Makefile (targets you’ll use constantly)**

```make
.PHONY: setup lint type test unit integ fmt run api ui docker-up docker-down reproduce
setup:  ## create venv, install deps
lint:   ## ruff + black --check
fmt:    ## ruff --fix + black
type:   ## mypy
test:   ## pytest -q
unit:   ## pytest -m "unit"
integ:  ## pytest -m "integration"
run:    ## uvicorn app.api:app --reload
api:    ## curl sanity calls (scripts/api_smoke.sh)
ui:     ## run tiny web ui later
docker-up:
docker-down:
reproduce: ## scripts/reproduce.sh
```

**CI “quality gates”**

* Lint, format, typecheck pass.
* `pytest` succeeds; start with 1 smoke test.

**Acceptance**

* `make setup && make test lint type` is green locally and in CI.
* `NEXT.md` contains:

  * **Next Action:** “Start Step 1. Create domain models and DB schema.”

---

# PART I — Core substrate (safe, deterministic, replayable)

## Step 1 — Domain models & JSON schemas

**Goal.** Define canonical data models for **nodes, edges, plans, attempts, mastery**, matching your PoC.

**Why now.** Everything depends on these shapes; define once, version forever.

**Files**

* `app/models.py` (Pydantic models w/ `model_json_schema()` snapshots)
* `schemas/*.json` (frozen JSON Schemas exported by tests)

**Key notes**

* Version each model via `model_version: SemVer` in metadata.
* Provide `to_canonical_json(obj) -> str` with **sorted keys**, UTF‑8, `\n` newline policy.

**Tests**

* Unit: round‑trip serialize/deserialize equality.
* Property: any dict produced by `.model_dump()` validates against exported schema.

**Acceptance**

* `tests/test_models.py` green.
* `schemas/` files created and pinned in git.

**Leave‑trail**

* NEXT.md → “Implement Graph Store tables and commit hashing.”

---

## Step 2 — Graph Store in SQLite + commit snapshot

**Goal.** Store nodes/edges/resources/items in SQLite; produce a **canonical JSON dump** and **commit hash** (SHA‑256) that seeds plans.

**Files**

* `app/graph_store.py` (CRUD + `export_canonical_json()` + `compute_commit()`)
* `app/db.py` (SQLite connection, migrations)
* `migrations/0001_init.sql`

**Tables (minimum)**

* `nodes(id TEXT PK, layer TEXT, kind TEXT, props JSON, provenance JSON, acl JSON)`
* `edges(src TEXT, dst TEXT, kind TEXT, attrs JSON, valid_since TEXT, valid_until TEXT, PRIMARY KEY(src,dst,kind))`
* `resources(id TEXT PK, meta JSON)`
* `items(id TEXT PK, meta JSON)`
* `schema_version(INT)`

**Notes**

* `export_canonical_json()` outputs **sorted arrays** by `id` and **stable key sorting**.
* `compute_commit()` = `sha256(export_canonical_json())[:7]`.

**Tests**

* Unit: CRUD, idempotent export (two consecutive exports identical).
* Property: Any shuffle of insertion order produces identical export.

**Acceptance**

* `graph_store.compute_commit()` stable; test passes.

**Leave‑trail**

* NEXT.md → “Seed a tiny toy graph + content for early planner tests.”

---

## Step 3 — Seed minimal toy graph & content

**Goal.** Add a tiny RL/NN slice to validate planner DP and witnesses end‑to‑end.

**Files**

* `data/graph.seed.json` (<= 10 concepts, 6 edges incl. `prereq_and`/`prereq_or`)
* `content/rl/td0.md`, `content/nn/backprop.md`
* `scripts/seed_content.py` (imports to SQLite, writes `data/graph.json` snapshot)

**Tests**

* Integration: after seeding, commit is deterministic and equals a known value (snapshot test).
* Content file anchors present.

**Acceptance**

* `make seed` (alias to run `scripts/seed_content.py`) produces same `data/graph.json` every run.

**Leave‑trail**

* NEXT.md → “Build Policy Engine skeleton + versions.”

---

## Step 4 — Policy Engine (skeleton, hard‑blocker)

**Goal.** Implement **policy registry** + **evaluators** for: difficulty bounds, exposure limits (stub), license/copyright, PII:none, grounded tutor guard (stub).

**Files**

* `app/policy.py`
* `app/policy_versions.py` (semver strings)
* `tests/test_policies.py`

**Notes**

* Interface: `PolicyEngine.evaluate(event: PolicyEvent) -> Decision{allow|block, reasons[], policy_versions}`.
* Event categories: `PLAN_REQUEST`, `QUIZ_SERVE`, `QUIZ_ATTEMPT`, `TUTOR_ASK`, `INGEST_RESOURCE`.

**Tests**

* Unit: difficulty blocking on “hard”.
* Unit: copyright snippet length enforcement.
* Property: stricter policy config never reduces block set.

**Acceptance**

* Policy decisions serializable; **no planner yet**, just enforceable rules.

**Leave‑trail**

* NEXT.md → “Planner v0: minimal prerequisite DP.”

---

## Step 5 — Planner v0: minimal prerequisite DP (AND/OR)

**Goal.** Implement exact minimal concept set for a target given **AND/OR prereqs** and a cost model (use uniform costs for now).

**Files**

* `app/planner.py`
* `tests/test_planner_dp.py` (unit on toy DAGs)

**Notes**

* Compute minimal proof tree by DP/memoization; return concept set (no resources yet).
* Accept inputs: `target_id, known_mastered:set[str], cost_model:Callable`.
* Determinism: ensure tie‑breakers sorted by concept id.

**Tests**

* Unit: correctness on small hand‑crafted AND/OR graphs.
* Property: adding mastered prerequisites never increases returned set.

**Acceptance**

* `planner.min_prereq_set()` returns expected sets on fixtures.

**Leave‑trail**

* NEXT.md → “Witness certificates & validator.”

---

Step 5a — NetworkX adapter for Planner (tiny)

Goal: Ensure in‑proc graph uses NetworkX to align with tech req.

Files: app/graph_adapter.py → to_nx(graph_store) -> nx.DiGraph with edge kind and attributes.

Tests: Convert seed graph and confirm planner results unchanged.

---

## Step 6 — Witness certificates & validator

**Goal.** For each coarse recommendation, include ≥1 supporting **witness** (span/exercise) and validate **zoom‑consistency**.

**Files**

* `app/witness.py` (emitter + validator)
* `tests/test_witness.py`

**Notes**

* Each concept maps to at least one `resource#anchor` or `item` via typed edges `teaches`, `assesses`.
* `validate_zoom(plan) -> bool` with an **explain** report.

**Tests**

* Unit: missing witness → invalid.
* Integration: seeded content produces valid witness paths.

**Acceptance**

* Zoom‑consistency property test passes for plan stubs.

**Leave‑trail**

* NEXT.md → “Packer v0: top‑k fallback; budgets; ablation flags.”

---

## Step 7 — Packer v0 (top‑k), budgets, ablation flags

**Goal.** Choose a resource pack that fits a budget (time/tokens). Start with simple **top‑k by heuristic**; wire **feature flags**.

**Files**

* `app/packer.py`
* `app/flags.py` (`ABLATE_PACKER`, `ABLATE_ZOOM`, `ABLATE_TYPED`)
* `tests/test_packer.py`

**Notes**

* Heuristic score = coverage − redundancy (Jaccard overlap) + support bonus − risk.
* Respect `budget_min`/`budget_max` per resource.
* Emit inclusions/exclusions with **marginal utility**.

**Tests**

* Unit: respects budget hard cap.
* Unit: ablation switch changes selection (snapshot).

**Acceptance**

* Deterministic selection with same inputs/seed.

**Leave‑trail**

* NEXT.md → “Scheduler v0: topo order + spacing stub.”

---

## Step 8 — Scheduler v0 (topo + spacing stub)

**Goal.** Order selected items by **topological order**, with a basic spacing rule: every 3rd step schedule a review if available.

**Files**

* `app/scheduler.py`
* `tests/test_scheduler.py`

**Notes**

* Use BKT decay **placeholder** (constant threshold) for now; real BKT later.

**Tests**

* Unit: respects prereq ordering.
* Unit: spacing inserts review at correct cadence.

**Acceptance**

* Stable ordered schedule for a given pack.

**Leave‑trail**

* NEXT.md → “Expose /plan via FastAPI + Typer CLI.”

---

## Step 9 — API & CLI: `/plan` end‑to‑end

**Goal.** Wire `POST /plan` (returns plan JSON with witnesses, schedule, hash) and Typer CLI `uni plan ...`.

**Files**

* `app/api.py` (FastAPI)
* `ui/cli/main.py` (Typer)
* `tests/test_api_plan.py` (integration, uses httpx)

**Notes**

* Plan hash = sha256 of the entire plan dict (canonical).
* Add: `seed = override_seed or int(graph_commit[:8], 16)` and record same in telemetry. This matches your tech requirement “commit hash of graph JSON is the plan seed.” [(API /plan) + app/rand.py.]
* Add a `PolicyEvent(kind="PLAN_REQUEST", payload={target, budget, student, policies...})`. If any guard fails (e.g., difficulty constraints for the student), block the plan and return the policy reasons. (You already do this for tutor/quiz; this makes planning consistent.) [`policy.py`]

**Tests**

* Integration: same (graph commit, policies, seed) → **identical plan hash**.
* Property: zoom‑consistency holds for returned plan.

**Acceptance**

* `curl` to `/plan` returns valid plan with `hash`.

**Leave‑trail**

* NEXT.md → “Quiz engine + exposure limits.”

---

## Step 10 — Quiz engine & exposure limits

**Goal.** Serve quizzes by concept; enforce **per‑student item exposure ≤ 3×**.

**Files**

* `app/quiz.py`
* `app/attempt_store.py` (SQLite tables: `attempts`, `item_exposure`)
* `tests/test_quiz.py`

**Tests**

* Unit: serving excludes over‑exposed items.
* Integration: attempt increments exposure counts.

**Acceptance**

* `/quiz/serve` & `/quiz/attempt` (API) behave per spec.

**Leave‑trail**

* NEXT.md → “BKT mastery update & thresholds.”

---

## Step 11 — BKT mastery & thresholds

**Goal.** Implement canonical **4‑param BKT** (`pL0, pT, pS, pG`), per‑concept mastery, threshold τ for “known”.

**Files**

* `app/mastery.py`
* `tests/test_bkt.py`

**Notes**

* Store `mastery` rows per student/concept with timestamps and model version.
* Deterministic: always update in fixed field order; log seed.

**Tests**

* Unit: formula correctness on textbook examples.
* Property: more correct answers → non‑decreasing posterior.

**Acceptance**

* Mastery updates after `quiz/attempt`; exposed via `/mastery/{student_id}`.

**Leave‑trail**

* NEXT.md → “Replan on mastery change (integration).”

---

## Step 12 — Replanning loop integration

**Goal.** After attempts change mastery set, call Planner again and **drop mastered readings**.

**Files**

* `app/api.py` (adjust)
* `tests/test_e2e_plan_quiz_replan.py` (integration)

**Tests**

* E2E: plan → quiz → attempt → replan reduces concept/resource set deterministically.

**Acceptance**

* Green integration test.

**Leave‑trail**

* NEXT.md → “Grounded Tutor skeleton + policy guard.”

---

## Step 13 — Tutor (grounded) + guard

**Goal.** “Grounded‑only” tutor: answers **using only current plan witnesses** (spans/items). Reject if uncited.

**Files**

* `app/tutor.py`
* `tests/test_tutor_grounding.py`
* Policy integration in `policy.py`.

**Notes**

* No external retrieval; only the anchors packaged with the plan.
* Answer must return citations `[resource#anchor]`. Any uncited token → policy reject.

**Tests**

* Unit: question within witnesses → allowed answer with citations.
* Red‑team: outside witness → rejected with reason.

**Acceptance**

* `/tutor/ask` enforces `policy:grounded_tutor`.

**Leave‑trail**

* NEXT.md → “Telemetry & replay (manifest logging).”

---

## Step 14 — Telemetry & replay

**Goal.** Log every decision with **graph commit**, **policy versions**, **random seeds**, container digests; expose `/replay/{plan_id}`.

**Files**

* `app/telemetry.py` (JSONL + SQLite manifest)
* `tests/test_replay.py`

**Notes**

* `replay(plan_id)` reconstructs inputs and re‑emits the plan; compare hash.

**Tests**

* Property: Replaying yields **byte‑identical** plan.

**Acceptance**

* Deterministic replay proven in test.

**Leave‑trail**

* NEXT.md → “Baselines B0–B2 runner.”

---

## Step 15 — Baselines B0–B2

**Goal.** Implement **B0 Expert syllabus**, **B1 KT‑only**, **B2 Vector recommender**.

**Files**

* `app/baselines.py`
* `data/syllabus.yaml`
* `tests/test_baselines.py`

**Notes**

* Common interface: `run_baseline(name, config, seed) -> metrics Row`.

**Tests**

* Unit: each baseline returns plausible next resource/item given toy state.

**Acceptance**

* CLI supports `uni baseline run --name B0|B1|B2`.

**Leave‑trail**

* NEXT.md → “Unconstrained assistant baseline (B3) sandbox.”

---

## Step 16 — Baseline B3 (sandboxed)

**Goal.** **B3 Unconstrained assistant**: LLM chat over full content (no policies) **inside sandbox** for lab tests only.

**Files**

* `app/baselines.py` (extend)
* `tests/test_baselines_b3.py`

**Notes**

* Still **no external network**; uses local content. Tag outputs as `unsafe` in metrics so they’re never served to students.

**Tests**

* Unit: responds; flagged as `unsafe`.

**Acceptance**

* Baseline runner records B3 metrics; policy system never routes to it in production paths.

**Leave‑trail**

* NEXT.md → “Simulation harness & synthetic learners.”

---

## Step 17 — Simulation harness

**Goal.** Batch runs with **synthetic learners** (sample BKT params), producing **learning/efficiency curves**.

**Files**

* `app/sim_harness.py`
* `tests/test_sim_harness.py`

**Notes**

* Configurable priors for `pL0, pT, pG, pS`.
* Task loop: planner → quiz → mastery update → replan until termination (budget or mastery).

**Tests**

* Integration: run small cohort with Planner vs B0/B1/B2; outputs Parquet/CSV.

**Acceptance**

* Outputs metrics files deterministically for given seed.
* Ensure every run logs tokens/time, steps‑to‑mastery, scores, violation counts, latency so Analytics can compute efficiency curves straight from the CSV/Parquet.

**Leave‑trail**

* NEXT.md → “Red‑team harness.”

---

## Step 18 — Red‑team harness

**Goal.** Scripted attempts to violate policies (too‑hard content, exposure, answer leak, unguided tutor).

**Files**

* `app/redteam.py`
* `tests/test_redteam.py`

**Acceptance**

* Harness report includes **block rate** and **violation rate** per system; all tests green on toy content.

**Leave‑trail**

* NEXT.md → “Analytics & report builder + `make reproduce`.”

---

Step 18.2 — Red‑team across modules

Goal: Add cases where tutor tries to cite outside the program graph when a module is expanded.

Files: tests/test_redteam_modules.py

Accept: All such attempts are blocked with policy:grounded_tutor.

---

## Step 19 — Analytics & report builder

**Goal.** Aggregate metrics; compute effect sizes; render a small **`report.html`**.

**Files**

* `app/analytics.py` (load Parquet/CSV, compute tables)
* `reports/templates/report.html.j2`
* `scripts/reproduce.sh` (Planner vs B0/B1/B2 + ablations)

**Tests**

* Unit: basic aggregations; deterministic table order.
* Integration: `make reproduce` writes `reports/benchmark report.html`.

**Acceptance**

* `make reproduce` runs end‑to‑end with log‑free seeds and produces the benchmark.
* Ensure every run logs tokens/time, steps‑to‑mastery, scores, violation counts, latency so Analytics can compute efficiency curves straight from the CSV/Parquet.

**Leave‑trail**

* NEXT.md → “Composability: Module Registry.”

---

# PART II — Composability layer

## Step 20 — Module Registry

**Goal.** CRUD for **module manifests**: `exports`, `requires`, resources, versioning.

**Files**

* `app/module_registry.py`
* `modules/nn.backprop/manifest.json` (example you supplied)
* `tests/test_module_registry.py`

**DB**

* `modules(id TEXT PK, version TEXT, manifest JSON, ts TEXT)`
* Immutability: updating requires a new version row.

**Tests**

* Unit: interface validation (exports exist, requires concepts exist/can exist).

**Acceptance**

* `uni module list/get/publish` CLI works.

**Leave‑trail**

* NEXT.md → “Composer/Resolver + equivalence mapping.”

---

## Step 21 — Composer/Resolver + equivalence & lockfile

**Goal.** Given a **Program Spec** (`targets`, `expand_modules[]`), compose a **Program Graph**, **dedupe** via `equivalent-to`, and emit `program.lock`.

**Files**

* `app/composer.py`
* `app/equivalence.py`
* `app/lockfile.py`
* `tests/test_composition.py`

**Notes**

* Steps:

  1. Load module manifests @ specific versions.
  2. Overlay subgraphs into a working graph.
  3. Resolve `equivalent-to`/`alias` to canonical IDs.
  4. Emit frozen **Program Graph** + `program.lock { modules + versions + graph_commit }`.
* Cache minimal‑prereq DP per `(module_version, target)`.

**Tests**

* Property: expanding a module vs. collapsed concept yields identical **exports mastered** under ample budget; subset relation otherwise.
* Cycle detection with helpful error.

**Acceptance**

* `/program/compose` endpoint returns lockfile & commit.

**Leave‑trail**

* NEXT.md → “Planner accepts Program Graph.”

---

Step 21.2 — Policy inheritance at module boundaries

Goal: Composer computes effective policy set = union of global + per‑module; resolve conflicts by stricter bound wins (log resolution).

Files: app/composer.py

Tests: Conflict matrix unit test (difficulty hard vs medium → medium blocked).

---

## Step 22 — Planner over Program Graph

**Goal.** Make planner read from **composed graph** (not global raw graph). Dedup resources in packer.

**Files**

* `app/planner.py` (minor changes)
* `tests/test_planner_program_graph.py`

**Tests**

* Integration: module expansion still passes zoom‑consistency; plan hash stable given same `program.lock`.

**Acceptance**

* Planning with `--lock program.lock` replays identically.

**Leave‑trail**

* NEXT.md → “UI: minimal student flows + instructor editor.”

---

# PART III — Minimal UI & ops

## Step 23 — Minimal Web UI (student flows) + Instructor graph editor

**Goal.** Thin web UI for:

* Student: request plan, see “why”, take quiz, ask grounded tutor.
* Instructor: browse/edit concept graph edges with evidence, publish module.

**Files**

* `ui/web/` (simple FastAPI Jinja or small SPA)
* `tests/test_ui_smoke.py` (selenium‑free, just HTTP smoke)

**Notes**

* Keep UI thin; server renders HTML from API for determinism.
* Instructor edits **write edges with anchors** and bump graph commit; no in‑place manifest edits (new versions only).
* Add a “Expand as module” control next to concepts (e.g., backprop) that calls /program/compose with expand=[module:…] and then re‑plans with the returned program.lock.

**Acceptance**

* Manual run: create plan, click “Why?”, submit quiz, view updated plan, add an edge, see new commit.

**Leave‑trail**

* NEXT.md → “Performance & caching.”

---

## Step 24 — Performance & caching

**Goal.** Ensure:

* Plan generation < target for ~300 concepts.
* Quiz serve < target median.

**Files**

* `app/cache.py` (LRU for DP results, BM25 index cache)
* Bench tests under `tests/perf/` (marked `-m "perf"`; not CI‑gated by default)

**Measures**

* Memoize DP per target+known set+graph commit.
* Prebuild BM25 index at startup; TTL.

**Acceptance**

* Perf smoke tests print stable numbers to logs; cache hit rates visible.
* Add a perf smoke asserting composer time < 200 ms for ~300‑concept graphs (non‑blocking but logged).

**Leave‑trail**

* NEXT.md → “Dockerization & reproducible builds.”

---

## Step 25 — Docker, pinned builds, manifests

**Goal.** Fully reproducible container images and manifests for runs.

**Files**

* `docker/Dockerfile` (pin base image; `pip install -r requirements.txt` with pinned versions)
* `docker-compose.yml` (api + optional faiss service)
* Update `telemetry` to record image digests.

**Acceptance**

* `make reproduce` builds containers, runs sim + red‑team + analytics, produces report with recorded digests.

**Leave‑trail**

* NEXT.md → “Docs & developer workflow.”

---

## Step 26 — Docs & developer workflow polish

**Goal.** Ensure onboarding is trivial and pause/resume is painless.

**Files**

* `README.md` (quickstart + features)
* `HACKING.md` (dev workflow, branching, seeds, determinism)
* `DESIGN.md` (architecture overview with your mermaids)
* `NEXT.md` template + examples
* `CONTRIBUTING.md` (style, tests, commit conventions)

**Acceptance**

* A new clone can follow README and get a plan in minutes; `NEXT.md` shows next action.

---

Step 27 — Ingestor + endpoints

Goal: Parse Markdown/Notebook front‑matter (title, difficulty, license, anchors), create reading/exercise nodes, teaches edges; update anchor index.

Files: app/ingestor.py, app/api.py (POST /ingest/resource), UI affordance (Step 23 tweaks).

Also: POST /graph/edge for instructor edits (with evidence anchors).

Tests: Round‑trip ingest → graph commit changes; anchors are discoverable; license captured for policy checks.

---

Step 28 — Indexer (BM25 + vector stub)

Goal: Build BM25 index over resources; optional FAISS/Annoy stub. Create latent-sim edges with TTL and calibrated confidence.

Files: app/indexer.py (batch build + refresh), store edges in graph with valid_until.

Flags: ABLATE_TYPED=0 toggles to untyped retrieval (BM25 only).

Tests: Deterministic index build; TTL expiry removes edges; ablation toggles behavior.

---

Step 29 — Packer v1 = submodular knapsack

Goal: Replace v0 top‑k with greedy marginal gain / cost under monotone submodular objective. Keep top‑k via ABLATE_PACKER=topk.

Files: app/packer.py

Tests: (a) Respects budget; (b) Greedy 1‑1/e approximation property smoke test on synthetic coverage; (c) Zoom check before return.

---

Step 30 — Lab Runner + autograder

Goal: Execute notebook labs (TD(0), backprop MLP) in a sandboxed process; capture metrics; grade via asserts or tests in cell tags.

Files: app/labs.py (nbconvert or papermill execution), labs/*, app/api.py (POST /lab/run), UI button “Run lab”.

Policy: Enforce difficulty + license + timeout caps.

Tests: Run tiny lab; record score; exposure rate for labs enforced.

---

Step 31 — Robustness & fairness hooks

Goal: (a) Graph noise injection (edge deletions/additions with rate p) in simulation; (b) learner tags (math‑heavy/code‑heavy) driving content selection; (c) sliced metrics by tag.

Files: app/sim_harness.py (conf adds noise + tags), app/analytics.py (slice outputs).

Tests: Slices present; runs reproducible; planner robust to small noise with bounded degradation.

---

Step 32 — Dataset scale‑up & license audit

Goal: Seed toward target counts (200–300 concepts, etc.). Enforce license checks in ingestor; include content manifest with sources.

Files: scripts/seed_content.py (scalable), data/content_manifest.json.

Tests: Seed deterministic IDs; license policy blocks non‑open items.

---

Step 33 — API polish & approvals

33.1 Auth middleware: header token X-Role to gate instructor endpoints.

33.2 /plan/{id}/why endpoint: returns typed paths + citations (already computable from witnesses).

33.3 Instructor approvals: UI & endpoint to mark resource/module approved; policy engine requires approved=true unless in dev mode.

33.4 /sim/run and /redteam/run endpoints: thin wrappers over existing CLIs for parity with PoC API list.

Tests: Role‑based access; approvals block unapproved content; endpoints return deterministic artifacts.

---

Step 34 — Scheduler v1 (BKT‑based spacing)

Goal: Replace spacing stub with BKT‑driven review scheduling using decay heuristic (time‑since‑evidence vs. p(L)).

Files: app/scheduler.py

Tests: More review for low p(L); ablation ABLATE_SPACING=0 disables.

---

## Tests to add (summary)

Indexer/TTL: deterministic edge gen; TTL expiry (Step 28).

Packer v1: budget compliance & marginal gains; zoom check (Step 29).

Lab runner: timeout, grading rubric, policy enforcement (Step 30).

Robustness/fairness: noise sensitivity curves; tagged slices (Step 31).

Policy inheritance: stricter resolution across modules (Step 21.2).

Auth & approvals: RBAC; unapproved content blocked (Step 33.1–33.3).

Scheduler v1: BKT‑informed reviews; ablation disables (Step 34).

Composer perf: measured and logged under tests/perf/ (Step 24 + note).

## Final sanity checklist (you can paste into NEXT.md)

 Step 5a NetworkX adapter wired; planner results unchanged.

 Step 27 Ingestor + /ingest/resource + /graph/edge live; commit hash updates.

 Step 28 Indexer builds BM25; latent‑sim edges with TTL & confidence.

 Step 29 Submodular knapsack packer default; top‑k via flag.

 Step 30 Lab runner executes and grades TD(0) & backprop notebooks.

 Step 31 Robustness/fairness hooks in sim & analytics.

 Step 32 Scaled seed + license audit; content manifest checked in.

 Step 33 Auth middleware; approvals; /plan/{id}/why; sim/redteam endpoints.

 Step 34 Scheduler uses BKT decay; ablation available.

 Composer policy inheritance (21.2) implemented & tested.

 Perf smoke tests capture plan & composer latency.

---

# Detailed Implementation Notes & Examples

## A. Determinism primitives (use everywhere)

* **Seed Registry**: `app/rand.py`

  ```py
  import random, numpy as np
  class SeedRegistry:
      def __init__(self, seed:int): self.seed = seed
      def scope(self, name:str): 
          r = random.Random(self.seed ^ hash(name))
          np_r = np.random.default_rng(self.seed ^ (hash(name) & 0xFFFF))
          return r, np_r
  ```
* **Canonical JSON**: `json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False) + "\n"`; no trailing spaces.

## B. Minimal prerequisite DP (sketch)

```py
# app/planner.py
def min_prereq_set(target:str, known:set[str], graph:Graph) -> set[str]:
    memo = {}
    def cost_of(c:str)->tuple[int,list[str]]:
        if c in known: return (0, [])
        if c not in graph.prereqs: return (1, [c])
        best = (10**9, [])
        for alt in graph.prereqs[c]:           # alt is AND set
            sub_cost = 0; sub_nodes=[]
            for p in alt:
                cst, nodes = memo.get(p) or cost_of(p)
                memo[p] = (cst, nodes)
                sub_cost += cst
                sub_nodes += nodes
            # include c itself
            cand = (sub_cost + 1, sorted(set(sub_nodes+[c])))
            if cand < best: best = cand
        return best
    _, nodes = cost_of(target)
    return set(nodes)
```

## C. Packer greedy (monotone submodular objective)

* Objective:

  ```
  f(S) = coverage(S) – redundancy(S) + witness_bonus(S) – risk(S)
  ```
* At each step, pick resource r with maximal **marginal gain / cost** under remaining budget.

## D. Scheduler spacing

* Maintain `last_seen[concept]`; if gap > heuristic from BKT decay, schedule a review resource/item.

## E. BKT equations

Update with response `correct ∈ {0,1}`:

```
p(L_t|obs) = [p(L_t)(1 - p(S))] / ([p(L_t)(1 - p(S))] + [(1 - p(L_t))p(G)])
p(L_{t+1}) = p(L_t|obs) + (1 - p(L_t|obs)) * p(T)
```

Store params per concept; expose via `/mastery`.

## F. Policy engine shape

```py
@dataclass
class PolicyEvent:
    kind: Literal['PLAN_REQUEST','QUIZ_SERVE','QUIZ_ATTEMPT','TUTOR_ASK','INGEST_RESOURCE']
    payload: dict
@dataclass
class Decision:
    allow: bool
    reasons: list[str]
    policy_versions: dict[str,str]
```

Each policy returns `Decision`; engine AND‑reduces, preferring **block** with merged reasons.

## G. Telemetry manifest

* Write `manifests/{run_id}.json` with:

  * `graph_commit`, `program_lock` hash (if any),
  * `policy_versions`, `seed`, `image_digests`, `timestamp`, `git_commit`.

## H. SQL DDL (excerpt)

```sql
CREATE TABLE IF NOT EXISTS nodes(
  id TEXT PRIMARY KEY, layer TEXT, kind TEXT,
  props TEXT NOT NULL, provenance TEXT NOT NULL, acl TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS edges(
  src TEXT, dst TEXT, kind TEXT, attrs TEXT, valid_since TEXT, valid_until TEXT,
  PRIMARY KEY(src,dst,kind)
);
CREATE TABLE IF NOT EXISTS attempts(
  attempt_id TEXT PRIMARY KEY, student TEXT, quiz_id TEXT, payload TEXT, score REAL, ts TEXT
);
CREATE TABLE IF NOT EXISTS mastery(
  student TEXT, concept TEXT, p_L REAL, model TEXT, params TEXT, ts TEXT,
  PRIMARY KEY(student, concept)
);
CREATE TABLE IF NOT EXISTS item_exposure(
  student TEXT, item_id TEXT, served_count INTEGER, PRIMARY KEY(student, item_id)
);
```

## I. HTTP endpoints (FastAPI, request/response)

* `POST /plan` → `Plan` (your JSON shape)
* `GET /plan/{id}` → plan + witnesses
* `GET /plan/{id}/why` (Step 33.2)
* `POST /quiz/serve` `{concept_ids, n_items}` → `{quiz_id, items}`
* `POST /quiz/attempt` `{quiz_id, responses}` → `{score, mastery_deltas}`
* `POST /tutor/ask` `{plan_id, question}` → `{answer, citations[]}` (or **reject** with reasons)
* `POST /lab/run` (Step 30)
* `POST /ingest/resource` (Step 27)
* `POST /graph/edge` (Step 27)
* `POST /program/compose` `{targets, expand[]}` → `{program_graph_commit, program_lock}`
* `GET /replay/{plan_id}` → `{plan, hash, equal_to_original: bool}`
* `POST /sim/run` -> (Step 33.4)
* `POST /redteam/run` (Step 33.4)

## J. Feature flags (env or query)

* `ABLATE_ZOOM=1`
* `ABLATE_PACKER=topk`
* `ABLATE_SPACING=0`
* `ABLATE_TYPED=0`

## K. Branching & commits

* **Branch per step**: `feat/step-07-packer`
* **Commit style**: Conventional Commits (`feat:`, `fix:`, `test:` …).
* Always end a session with a commit **and** `NEXT.md` update.

## L. Testing strategy (what goes where)

* `tests/unit/*`: pure functions (DP, packer, BKT, policies)
* `tests/integration/*`: API flows, replay, tutor grounding, composition
* `tests/property/*`: determinism, zoom‑consistency
* `tests/perf/*`: opt‑in performance smoke, not CI‑blocking

## M. Observability

* **Request IDs** via middleware; include in JSON logs.
* **Counters**: CSV with columns `(ts,event,latency_ms,ok,policy_blocked,seed,graph_commit,program_lock_hash)`.

## N. Security & auth (development‑grade)

* Simple header token `X-Role: student|instructor` for RBAC on endpoints.
* No PII: anonymized `student` ids; enforce via policy check when creating attempts.

## O. Reproducibility controls

* Pin package versions.
* Record `git_commit`, container digests in manifest.
* **Stable sorts** anywhere selection occurs.

---

# Daily “pause/resume” checklist (repeat every session)

1. **Before coding**

   * Open `NEXT.md` → confirm **Next Action**.
   * `make test` must be green.
   * If failing, revert last WIP or fix tests first.

2. **While coding**

   * Write tests first or in lockstep with code.
   * Keep changes within current step.

3. **Before stopping**

   * `make fmt lint type test` → all green.
   * Update `NEXT.md`:

     * “What I did”
     * **Next Action** (one line, concrete)
     * “Open questions/risks”
   * Commit & push.

**Template for `NEXT.md`**

```md
# NEXT

## What I just finished
- Step 8: Scheduler v0 passes tests.

## Next Action
- Step 9: Add /plan endpoint wiring (FastAPI + httpx test).

## Notes / Risks
- Spacing rule currently constant; will tie to BKT in Step 11.
```

---

# Risks & mitigations (so you don’t get stuck)

* **Determinism drift**: Always canonicalize JSON and stable‑sort selections; write a **golden plan hash** test.
* **Policy bypass in tutor**: Checker must validate **every token segment** has a supporting citation; conservatively reject if uncertain.
* **Graph growth performance**: Memoize DP per `(commit, target, known_set_signature)`; build BM25 index once.
* **Schema creep**: Never edit manifests in place—**new versions only**; maintain migrations under `migrations/`.

---

# Milestone mapping (non‑time‑based)

* **Core**: Steps 1–14
* **Baselines & Sim**: Steps 15–19
* **Composability**: Steps 20–22
* **UI & Ops**: Steps 23–26

Every milestone ends with a runnable artifact (`/plan`, `/replay`, `make reproduce`, `program.lock`, or UI demo) and green CI.

---

## You can start now

Open `NEXT.md` and set:

```
Next Action: Step 1 — Implement domain models (Pydantic) and export JSON Schemas.
```

Then:

```bash
make setup
make fmt lint type test
git checkout -b feat/step-01-models
```

Follow the step card above. When you stop, update `NEXT.md`, commit, and you’ll know exactly what to pick up next evening.
