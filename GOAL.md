# Session goal — target-prioritization skill

Handoff for the next Claude Code session opened **inside this directory**.
Read this top-to-bottom before touching files.

---

## 1. What this skill is

A multi-source drug-target due-diligence pipeline for ranked gene lists
(typically scRNA-seq DE outputs). Input: a CSV with `gene` column.
Output: per-gene dossier markdown + summary CSV, ranked by a composite
score combining protein biology, druggability, GWAS, cross-lineage DE
convergence, and literature maturity.

Designed for internal use of a single researcher — not a published research
tool. No publication / patent ambition for the skill itself.

Originating context: CD-UST single-cell project at
`/Volumes/niehu_ext2/niehu/SZBL/Projects/cd_ust_20251119/`. Specifically
the user wants to triage targets out of
`05.integration_harmony/analyze_major_clusters/CD4T_Step3/hu_de_Th1/post_r_vs_nr/expression_table_pass_either_1s.csv`
(and the Th17 sibling) without doing 80 manual lookups.

---

## 2. Current state (what's already built)

Directory layout right now (flat, **not** yet restructured to
semanticscholar pattern):

```
target-prioritization/
├── SKILL.md                  ← top-level skill instructions (long-form)
├── README.md                 ← short quickstart only — NEEDS rewrite
├── GOAL.md                   ← this file
├── weights.yaml              ← composite score weights, all configurable
├── scripts/
│   ├── orchestrate.py        ← main entry point; parallel fetcher dispatch
│   ├── fetch_uniprot.py      ← ✅ working
│   ├── fetch_opentargets.py  ← ⚠️ schema-updated mid-fix, UNTESTED post-edit
│   ├── fetch_gwas.py         ← ⚠️ fully rewritten, UNTESTED
│   ├── fetch_pubmed.py       ← ✅ working
│   ├── fetch_local_de.py     ← ⚠️ written but not run in smoke test
│   └── aggregate.py          ← ✅ working, produces CSV + markdown skeleton
├── prompts/
│   └── rationale_template.md ← LLM prompt template for filling per-gene rationale
└── references/
    └── api_endpoints.md      ← API docs and "how to add a new source"
```

Python stdlib only — no external dependencies.

### Smoke test status

Last test run: 3 genes (`IL23R`, `HLA-DRB5`, `MAP3K8`) on
`/tmp/tp_smoke/test_genes.csv` →`/tmp/tp_smoke/out/`.

- UniProt: ✅ returned correct protein names + localization
- PubMed: ✅ returned correct paper counts + maturity tags
- OpenTargets: ❌ schema mismatch → fixed (see §3.1) but not re-tested
- GWAS Catalog: ❌ `findByGeneName` → 404 → fixed (see §3.2) but not re-tested
- Local DE: ⏭️ not exercised (no `--project-root` passed)

---

## 3. Known bugs mid-fix

### 3.1 OpenTargets — schema migration partially applied

Old OT v4 schema:
- `target.knownDrugs(size: N)` → returned `{count, rows[]}`
- `Drug.isApproved` (boolean)
- `Drug.maximumClinicalTrialPhase` (int)
- KnownDrug row had `phase` (int), `mechanismOfAction` (str), `disease` (object)

**Current v4 schema (verified live 2026-05-19):**
- Field renamed: `target.drugAndClinicalCandidates` → `{count, rows[]}`, **no `size` argument**
- Row type is `ClinicalTargetFromTarget`:
  - `id`, `maxClinicalStage` (String, e.g. "Approved", "Phase III"), `drug`, `diseases`
  - No `phase` (int), no `mechanismOfAction` (str), no `disease` (singular)
- `Drug` has: `name`, `maximumClinicalStage` (String), `drugType`,
  `mechanismsOfAction { rows { mechanismOfAction } }`
  - **No `isApproved`** → derive by checking stage string ∈ {"Approved", "Phase IV"}
- `ClinicalDiseaseListItem` has `diseaseFromSource` (str) + `disease { id name }`

**What I did**: rewrote `TARGET_QUERY` in
`scripts/fetch_opentargets.py` and added `PHASE_MAP`,
`stage_to_phase()`, `is_approved_stage()` helpers. The parser loop now
walks `drugAndClinicalCandidates.rows[*]` with the new field names.
**Not yet smoke-tested.**

### 3.2 GWAS Catalog — endpoint deprecated

Old endpoint `…/associations/search/findByGeneName?geneName=X` → 404 for
all genes.

**Working endpoint** (verified live):
- `…/singleNucleotidePolymorphisms/search/findByGene?geneName=X&page=N&size=50`
  returns SNPs only — no trait info attached.
- To get traits, follow each SNP →
  `…/singleNucleotidePolymorphisms/{rs}/associations` →
  `_links.efoTraits.href` → list of trait names.

That's an N+1+M chain. To keep cost bounded I capped to
`MAX_SNPS_PER_GENE = 20` and `MAX_PAGES = 2`. Untested.

**Alternative worth considering**: skip dedicated GWAS fetcher entirely
and rely on `OpenTargets.target.associatedDiseases` (already retrieved)
to detect IBD / Crohn's / UC genetic signal. That's what OT does
internally (GWAS Catalog is one of OT's evidence sources). Cleaner and
1 HTTP call vs ~20+. Decide before testing.

### 3.3 Local DE fetcher untested

`scripts/fetch_local_de.py` heuristically scans for sibling DE CSVs
under `--project-root` (matches dir patterns `hu_de_*`, `pert_de_*`,
`cluster_degs*`, `de_*`). Logic looks sound but never run. First real
test should be against
`/Volumes/niehu_ext2/niehu/SZBL/Projects/cd_ust_20251119/` (the CD-UST
repo where the user's data lives).

---

## 4. New user requirements (the actual goal of next session)

### 4.1 License

User said "**不可商用**" (non-commercial). Semanticscholar-skill uses MIT
(commercial OK) — we deviate. Recommended licenses:

- **PolyForm Noncommercial 1.0.0** — software-friendly, modern, clear.
  Text: <https://polyformproject.org/licenses/noncommercial/1.0.0/>
- Or **CC BY-NC 4.0** — well-known but designed for content, not code;
  weaker for software.

Recommend PolyForm-NC. Drop full text into `LICENSE`. Copyright holder:
`Agents365-ai` (matches the existing 365-skills repo conventions).

Update license badge in README from `MIT` (in
semanticscholar's README) to `PolyForm-NC 1.0.0`.

### 4.2 Restructure to semanticscholar pattern

Current shape is flat (`scripts/`, `prompts/`, `references/` at the
repo root with `SKILL.md` at the top). Semanticscholar uses the
agent-skills standard layout:

```
semanticscholar-skill/         (repo root — metadata only)
├── LICENSE
├── README.md                  English (badges + features + install)
├── README_CN.md               Chinese mirror
├── .gitignore
├── .github/workflows/sync-365-skills.yml   ← auto-sync to 365-skills monorepo
└── skills/
    └── semanticscholar-skill/
        ├── SKILL.md           ← actual skill body
        └── s2.py              ← (semanticscholar puts everything in one .py)
```

For target-prioritization, the move is:

```
target-prioritization/                    (repo root)
├── LICENSE                               new
├── README.md                             rewrite (badges + features)
├── README_CN.md                          new
├── .gitignore                            new
├── .github/workflows/sync-365-skills.yml new
└── skills/
    └── target-prioritization/            ← move into here
        ├── SKILL.md                      ← currently at root
        ├── weights.yaml
        ├── scripts/*.py
        ├── prompts/rationale_template.md
        └── references/api_endpoints.md
```

`GOAL.md`, `README.md`, `LICENSE`, etc. stay at the **repo root** and
do NOT get synced into 365-skills.

### 4.3 README files

Refer to `/Users/niehu/myagents/myskills/semanticscholar-skill/README.md`
and `README_CN.md` for the pattern.

Sections expected (mirror semanticscholar where applicable):

1. Title + tagline + emoji
2. Badge row — License, GitHub stars/forks, Latest Release, Last Commit,
   SkillsMP, ClawHub, Claude Code Plugin, Agent Skills, Discord
   - **Change `MIT` badge to `PolyForm-NC`**
   - GitHub URL: `Agents365-ai/target-prioritization` (assuming this is
     where it will be published)
3. Language toggle line: `**English** · [中文](README_CN.md)` (and inverse in CN)
4. External references — list the 5 APIs used (UniProt, OpenTargets,
   GWAS Catalog, PubMed, plus internal local-DE scanner)
5. Short feature list (search → fetch → rank, parallel fetchers, no API keys)
6. "How it works" — keep the mermaid/ASCII diagram from current SKILL.md
7. Install — Claude Code (plugin marketplace + manual), OpenClaw,
   SkillsMP — show the three install paths as semanticscholar does
8. Usage — minimal example using the `expression_table_pass_either_1s.csv`
   file from the CD-UST project as the canonical input
9. Output format — describe `targets_summary.csv` columns + per-gene
   markdown section structure
10. Composite score formula + how to re-tune via `weights.yaml`
11. Data sources table (copy from current SKILL.md "Data source notes")
12. Native Claude Code comparison column (semanticscholar README has
    one — match that style)
13. License section pointing at LICENSE file

### 4.4 Add to 365-skills

The `.github/workflows/sync-365-skills.yml` template lives at
`/Users/niehu/myagents/myskills/semanticscholar-skill/.github/workflows/*.yml`.

Read it in full and adapt — main substitutions:

- Source path: `skills/target-prioritization/` (not semanticscholar)
- Target path in 365-skills:
  `plugins/target-prioritization/skills/target-prioritization/`
- Plugin slug: `target-prioritization`
- Repo secret: `SYNC_365_SKILLS_TOKEN` (already exists per
  semanticscholar workflow — reuse the same name)

After the workflow lands, manually add a `plugins/target-prioritization/`
entry to `Agents365-ai/365-skills`'s `marketplace.json` so the plugin
is discoverable. The workflow will then keep the synced copy fresh on
every push to main.

The workflow does two things:
1. `rsync -a --delete` source `skills/target-prioritization/` into
   target `plugins/target-prioritization/skills/target-prioritization/`
2. Parse `version` from the SKILL.md frontmatter metadata JSON, regex-
   patch the version in `marketplace.json`

This means **SKILL.md frontmatter needs a `version`** in
`metadata.openclaw`. Currently the SKILL.md frontmatter is:

```yaml
metadata: {"openclaw":{"requires":{"bins":["python3","curl"]},"emoji":"🎯"}}
```

Add `"version":"0.1.0"` (or whatever seems right for an initial release)
into that JSON.

---

## 5. Concrete TODO for next session

In order:

1. **Test the OpenTargets fix** — re-run the smoke test:
   ```bash
   rm -rf /tmp/tp_smoke && mkdir -p /tmp/tp_smoke
   printf "gene\nIL23R\nHLA-DRB5\nMAP3K8\nTNF\n" > /tmp/tp_smoke/test_genes.csv
   python3 scripts/orchestrate.py \
       --input /tmp/tp_smoke/test_genes.csv \
       --output /tmp/tp_smoke/out --top 4
   ```
   Expected: IL23R should now show `approved_drug_count >= 1` (it's a
   classic druggable target — ustekinumab targets IL23 via IL23R axis;
   risankizumab / guselkumab target IL23A; brazikumab targets IL23A).
   TNF should show many approved drugs (infliximab, adalimumab, etc.).
   If still empty, run the schema discovery commands captured in
   §3.1 to find what else moved.

2. **Decide GWAS strategy**: either test the new SNP-chain fetcher OR
   drop it and rely on OpenTargets `associatedDiseases` (which already
   surfaces IBD signal via OT's integrated GWAS data). My recommendation
   is the second option — simpler, faster, lower failure surface.

3. **Test local_de fetcher** end-to-end against:
   ```
   --project-root /Volumes/niehu_ext2/niehu/SZBL/Projects/cd_ust_20251119
   ```
   Use IL23R / HLA-DRB5 / IFNG — should hit multiple sibling DE files.

4. **Restructure to semanticscholar layout** (§4.2). Use `git mv` so
   history is preserved if this dir is already under version control;
   otherwise plain `mv`.

5. **Add LICENSE** (PolyForm-NC 1.0.0 — fetch full text from
   <https://polyformproject.org/licenses/noncommercial/1.0.0/>).

6. **Write README.md** (English) and **README_CN.md** (Chinese) per §4.3.
   Make sure the License badge says `PolyForm-NC` not `MIT`.

7. **Add .gitignore** — copy from semanticscholar:
   ```
   # Claude
   CLAUDE.md
   .claude/
   logs/
   .env
   .DS_Store
   __pycache__/
   *.pyc
   ```

8. **Add .github/workflows/sync-365-skills.yml** per §4.4, adapted.

9. **Add `version`** to `SKILL.md` frontmatter metadata (§4.4 last paragraph).

10. **Initialize git repo + create empty GitHub repo** at
    `Agents365-ai/target-prioritization`, push, then add the
    `marketplace.json` entry in `Agents365-ai/365-skills` so the
    plugin shows up. (Some of this may already be the user's manual
    step — confirm before doing.)

11. **End-to-end real test** — run on the actual user data:
    ```
    --input /Volumes/niehu_ext2/niehu/SZBL/Projects/cd_ust_20251119/05.integration_harmony/analyze_major_clusters/CD4T_Step3/hu_de_Th1/post_r_vs_nr/expression_table_pass_either_1s.csv
    --output /tmp/tp_th1_post_r_vs_nr
    --top 80
    --project-root /Volumes/niehu_ext2/niehu/SZBL/Projects/cd_ust_20251119
    ```
    Then have Claude fill the rationale slots in `targets_report.md` per
    `prompts/rationale_template.md`. Spot-check the top-5 to confirm the
    composite score isn't crowning obvious junk.

---

## 6. Useful reference paths

- Semanticscholar template (the pattern to mimic for steps 4–8):
  `/Users/niehu/myagents/myskills/semanticscholar-skill/`
  - README pattern: `README.md` + `README_CN.md`
  - Workflow template: `.github/workflows/*.yml`
  - Directory layout: `skills/<name>/SKILL.md` + helper scripts
- 365-skills publishing rules:
  `/Users/niehu/myagents/myskills/CLAUDE.md` (sections "Publishing
  Rules", "Install Paths", "GitHub Topics")
- Real user data root: `/Volumes/niehu_ext2/niehu/SZBL/Projects/cd_ust_20251119/`
- Originating DE files (input candidates for end-to-end test):
  - `…/CD4T_Step3/hu_de_Th1/post_r_vs_nr/expression_table_pass_either_1s.csv`
  - `…/CD4T_Step3/hu_de_Th17/post_r_vs_nr/expression_table_pass_either_1s.csv`

---

## 7. Things explicitly NOT in scope for next session

- No eval/benchmark loop (`skill-creator` style). User explicitly said
  "节约时间即可" — just make it work for internal use.
- No publication or patent prep. Skill outputs are internal hypothesis-
  generation only.
- No DGIdb / Reactome / OpenTargets Genetics integration in v1 — the 5
  current sources are enough. Document hooks in
  `references/api_endpoints.md` for future additions.

---

## 8. Quick sanity-check before declaring done

1. `python3 scripts/orchestrate.py --help` exits clean.
2. End-to-end run on 4-gene test produces all 4 raw JSON files + CSV +
   markdown skeleton.
3. IL23R and TNF both show `approved_drug_count > 0` and IBD-tagged
   drugs.
4. `targets_summary.csv` opens in Excel without parse errors.
5. README badges render (preview locally with VSCode markdown).
6. `LICENSE` says PolyForm-NC, not MIT.
7. `skills/target-prioritization/SKILL.md` has a `version` field.

Good luck — the foundation is solid, just needs the schema fixes
finished and the packaging layer wrapped around it.
