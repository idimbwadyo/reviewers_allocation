# Reviewers Allocation

Design and reference implementation for a fair, efficient, and transparent system to assign academic reviewers to research projects.

Goals:
- 2 reviewers per project.
- Each reviewer handles 2–3 projects (balanced workload).
- Avoid conflicts of interest (COIs) with project PIs.
- Maintain anonymity between projects and reviewers.
- Ensure process transparency and reproducibility.

This repository describes the system, data formats, optimization model, and a proposed CLI. It’s implementation‑agnostic but assumes a typical Python stack (pandas + OR‑Tools/PuLP) for concreteness.

## Objectives
- Fairness: balanced reviewer workloads (2–3 assignments) and even distribution across areas.
- Feasibility: resolve most calls with hard constraints; clearly diagnose infeasible cases.
- Quality: prefer reviewer–project matches with higher expertise/affinity when available.
- Anonymity: operate on pseudonymous IDs; keep mapping files private.
- Transparency: export full audit logs, inputs, parameters, and deterministic seeds.

## Data Inputs
You can provide data either as a single Excel workbook or as normalized CSVs. Excel is recommended for authoring; CSVs are the anonymized representation used by the solver.

### Option A: Single Excel workbook (`data/input.xlsx`)
- Sheet `Projects` (one row per submission)
  - `Project`: project title or unique identifier (free text)
  - `PI1`: first PI name (required)
  - `PI2`: second PI name (optional)
  - `PI3`: third PI name (optional)
  - `Theme`: thematic area (e.g., "AI for Health", "Robotics")

- Sheet `Reviewers` (one row per potential reviewer)
  - `Reviewer`: reviewer full name
  - `Faculty`: Yes/No (eligibility; only "Yes" are considered)
  - `Expertise`: comma/semicolon‑separated areas (e.g., "LLMs, Finance; 3D Models")

Preprocessing rules (performed by the loader):
- Filter reviewers where `Faculty != "Yes"`.
- Conflicts of interest (COI): any reviewer whose name matches any of `PI1..PI3` (case‑insensitive, trimmed) for a project is marked conflicted with that project.
- Theme/Expertise parsing: split on commas/semicolons, trim whitespace, normalize casing.
- Anonymization: map all names/titles to pseudonymous IDs used by the solver; keep mappings in `private/`.
- Per‑project `needs` defaults to 2 (can be overridden in config for special cases).

### Option B: Normalized CSVs (anonymized)
- `projects.csv`
  - `project_id`: string (anonymized ID)
  - `pi_id`: string (anonymized PI ID)
  - `area`: string (subject area/track; derived from `Theme`)
  - `needs`: int (default 2; reviewers per project)

- `reviewers.csv`
  - `reviewer_id`: string (anonymized ID)
  - `areas`: string (semicolon‑separated areas derived from `Expertise`)
  - `min_load`: int (default 2)
  - `max_load`: int (default 3)

- `conflicts.csv` (one of the two forms is supported)
  - Form A: `reviewer_id,project_id`
  - Form B: `reviewer_id,pi_id` (expanded to projects by PI)

- `affinity.csv` (optional)
  - `reviewer_id,project_id,score` (numeric; larger means better fit)

Notes:
- All IDs here are pseudonyms. Keep real‑ID mappings in `private/` and never publish them.
- If `needs` is omitted, it is assumed to be 2 for all projects.

## Optimization Model
We formulate assignment as a 0/1 integer program over a bipartite graph of reviewers and projects.

Variables:
- `x[r,p] ∈ {0,1}`: 1 if reviewer `r` is assigned to project `p`.

Hard constraints:
- Project coverage: ∀ project `p`: `Σ_r x[r,p] = needs_p` (typically 2).
- Reviewer load bounds: ∀ reviewer `r`: `min_load_r ≤ Σ_p x[r,p] ≤ max_load_r` (default 2–3).
- Conflicts: if `(r,p)` is conflicted then `x[r,p] = 0`.

Optional constraints (enable via config):
- Area matching: only allow `(r,p)` if area overlap exists.
- Project diversity: limit how many assignments a single reviewer gets within the same track.
- Institutional limits: cap assignments per institution (requires institution fields).

Objective (lexicographic or weighted sum):
1) Minimize load imbalance: `Σ_r |Σ_p x[r,p] − target|` (target typically 2 or 2.5).
2) Maximize total affinity: `− Σ_{r,p} (−affinity[r,p]) * x[r,p]`.
3) Minimize soft violations if softening is enabled (see feasibility below).

Determinism:
- Break ties with a fixed reviewer/project ordering and seeded randomization to ensure reproducibility.

## Workflow
1) Prepare inputs: either author `data/input.xlsx` with `Projects` and `Reviewers` sheets, or provide normalized CSVs (`projects.csv`, `reviewers.csv`, `conflicts.csv`, optional `affinity.csv`).
2) Configure `config.yaml` (see below) for constraints, objective weights, and solver settings.
3) Run the solver to produce assignments and audit artifacts.
4) Validate outputs with the checker to ensure constraints hold.
5) Distribute anonymized assignments; keep any mapping files private.

### Example `config.yaml`
```yaml
solver: ortools      # or pulp
time_limit_s: 60
seed: 42
constraints:
  enforce_area_match: false
  enforce_institution_caps: false
objective:
  weight_load_balance: 1.0
  weight_affinity: 1.0
feasibility:
  allow_soften_project_needs: false  # if true, needs may drop to 1 with penalty
  allow_soften_min_load: false       # if true, some reviewers may have 1 assignment
```

## CLI (proposed)
Reference interface (implementation to follow):

```
# From Excel workbook
python -m allocator assign \
  --excel data/input.xlsx \
  --config config.yaml \
  --out out/

# From normalized CSVs
python -m allocator assign \
  --projects data/projects.csv \
  --reviewers data/reviewers.csv \
  --conflicts data/conflicts.csv \
  --affinity data/affinity.csv \
  --config config.yaml \
  --out out/

python -m allocator check --in out/assignments.csv --projects data/projects.csv --reviewers data/reviewers.csv
```

Command behavior:
- `assign`: reads Excel or CSVs, performs anonymization and COI detection, solves, and writes `out/assignments.csv`, `out/summary.md`, `out/audit.json`, `out/diagnostics.csv`.
- `check`: validates that outputs satisfy hard constraints and reports any violations.

## Outputs
- `out/assignments.csv`: `project_id,reviewer_id` pairs (anonymized IDs only).
- `out/summary.md`: human‑readable summary (coverage, load histograms, solver status).
- `out/audit.json`: inputs digests, config, seed, objective breakdown, constraint checks.
- `out/diagnostics.csv`: projects at risk (low affinity, tight constraints), reviewer load details.

## Anonymity
- Use pseudonymous `project_id`, `reviewer_id`, and `pi_id` everywhere in the optimization and outputs.
- Keep `private/mapping_projects.csv` and `private/mapping_reviewers.csv` with real IDs locally only.
- Do not export mapping files; only share anonymized outputs.

## Transparency
- Track all parameters in `config.yaml`.
- Record a deterministic seed and sorted entity orders.
- Export an `audit.json` including: config, seed, input file hashes, objective components, and per‑constraint satisfaction.
- Provide a `summary.md` explaining rationale and highlighting any softened constraints (if enabled).

## Feasibility & Fallbacks
If the model is infeasible (e.g., too many conflicts or too tight loads):
- Report infeasibility quickly with a minimal unsatisfiable core when possible (smallest conflicting subset of constraints).
- Suggest remedies (e.g., widen reviewer max_load to 4 for N reviewers, relax area matching, recruit more reviewers in area X).
- Optional softening (disabled by default): introduce slack variables with high penalties to allow limited deviations (e.g., 1 reviewer for a project or 1 assignment for a reviewer). All softenings must be explicitly reported in the audit and summary.

## Minimal Example
Excel (`data/input.xlsx`)

- `Projects`
  - Project: "Towards Safer LLMs"
  - PI1: "Alice Smith"; PI2: ""; PI3: ""
  - Theme: "AI for Safety"

- `Reviewers`
  - Reviewer: "Bob Jones"; Faculty: "Yes"; Expertise: "AI Safety; LLMs"
  - Reviewer: "Alice Smith"; Faculty: "Yes"; Expertise: "NLP" (conflicted with the project via PI1)

Expected preprocessing:
- Only faculty reviewers are considered.
- Conflicts detect that reviewer "Alice Smith" is conflicted with any project listing "Alice Smith" as PI1–PI3.

CSV (normalized) equivalent
`data/projects.csv`
```
project_id,pi_id,area,needs
P001,PI10,ML,2
P002,PI11,NLP,2
P003,PI12,ML,2
```

`data/reviewers.csv`
```
reviewer_id,areas,min_load,max_load
R01,ML;NLP,2,3
R02,ML,2,3
R03,NLP,2,3
```

`data/conflicts.csv`
```
reviewer_id,project_id
R01,P001
```

`data/affinity.csv` (optional)
```
reviewer_id,project_id,score
R01,P002,0.9
R02,P001,0.8
R02,P003,0.7
R03,P002,0.85
R03,P003,0.6
```

Expected properties of the solution:
- Every project gets exactly 2 reviewers.
- Each reviewer has 2–3 assignments.
- No row in `conflicts.csv` appears in `assignments.csv`.
- Higher affinity matches are preferred, subject to fairness constraints.

## Reproducibility
- Fix `seed` and ensure stable sort orders of `project_id` and `reviewer_id`.
- Hash inputs and store digests in `audit.json`.
- Run in a pinned environment (e.g., `requirements.txt` or `poetry.lock`).

## Suggested Stack
- Python 3.10+
- pandas for CSV I/O and preprocessing
- OR‑Tools CP‑SAT (preferred) or PuLP + CBC/Gurobi/CPLEX
- PyYAML for config parsing

## Roadmap
- Implement `allocator` CLI with OR‑Tools backend.
- Add feasibility diagnostics and minimal conflict sets.
- Add HTML report with charts for loads and coverage.
- Add unit tests for validator and COI expansion.

## Ethics & Governance
- Treat COIs strictly; default to disallow unknown/ambiguous relationships.
- Preserve anonymity throughout, with access to mapping files on a need‑to‑know basis.
- Document any manual overrides in the audit log.
