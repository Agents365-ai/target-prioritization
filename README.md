# target-prioritization — multi-source drug-target due-diligence for ranked gene lists 🎯

[![License: PolyForm-NC](https://img.shields.io/badge/License-PolyForm--NC%201.0.0-orange.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/Agents365-ai/target-prioritization?style=flat&logo=github)](https://github.com/Agents365-ai/target-prioritization/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/Agents365-ai/target-prioritization?style=flat&logo=github)](https://github.com/Agents365-ai/target-prioritization/network/members)
[![Latest Release](https://img.shields.io/github/v/release/Agents365-ai/target-prioritization?logo=github)](https://github.com/Agents365-ai/target-prioritization/releases/latest)
[![Last Commit](https://img.shields.io/github/last-commit/Agents365-ai/target-prioritization?logo=github)](https://github.com/Agents365-ai/target-prioritization/commits/main)

[![SkillsMP](https://img.shields.io/badge/SkillsMP-listed-1f6feb)](https://skillsmp.com/)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-8a2be2)](https://github.com/Agents365-ai/365-skills)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-compatible-2ea44f)](https://agentskills.io)
[![Discord](https://img.shields.io/badge/Discord-Join-5865F2?logo=discord&logoColor=white)](https://discord.gg/79JF5Atuk)

**English** · [中文](README_CN.md)

External references: [UniProt REST](https://www.uniprot.org/help/api) · [OpenTargets GraphQL](https://platform-docs.opentargets.org/data-access/graphql-api) · [PubMed E-utilities](https://www.ncbi.nlm.nih.gov/books/NBK25501/) · [Human Protein Atlas](https://www.proteinatlas.org/about/help/dataaccess)

A skill that turns a ranked gene list (typically a scRNA-seq DE output) into a per-gene drug-target dossier — fanning out parallel queries to UniProt, OpenTargets, PubMed, and the Human Protein Atlas, then re-ranking everything by a composite score combining protein biology, druggability, disease genetics, tissue / cell-type specificity, and research maturity.

- **Multi-source evidence** — UniProt (localization, surface / secreted / MHC), OpenTargets (tractability, approved drugs, disease associations including GWAS-derived signal), PubMed (total + configurable disease-focus + cell-context paper counts), Human Protein Atlas (tissue + single-cell specificity and nCPM, expression cluster, cancer prognostics)
- **Parallel fetchers** — Python `ThreadPoolExecutor` dispatches all sources concurrently
- **Composite re-ranking** — per-component scores (druggability, disease genetics, tractability, tissue specificity, cell context, expression, novelty) combined via fully configurable `weights.yaml`
- **No API keys, no external Python deps** — stdlib + curl-compatible network only
- **Re-scorable** — raw JSON cache lets you tweak `weights.yaml` and rerun aggregate in seconds without re-fetching

Works with Claude Code, OpenClaw, and SkillsMP — any agent that supports the [Agent Skills](https://agentskills.io) format.

## 🔄 How it works

```
input gene list (CSV / TXT)
        │
        ▼
scripts/orchestrate.py
        │
        ├─► fetch_uniprot.py        → protein localization, surface, MHC, signal peptide
        ├─► fetch_opentargets.py    → tractability, approved drugs, focus-disease
        │                              trial tags, associated diseases (integrates
        │                              GWAS evidence)
        ├─► fetch_pubmed.py         → total + focus-disease + cell-context paper
        │                              counts, maturity tag
        └─► fetch_hpa.py            → HPA tissue / single-cell specificity tag,
                                       top tissue/cell-type nTPM/nCPM, expression
                                       cluster, cancer prognostics summary
        │
        ▼
scripts/aggregate.py — composite score + tier assignment
        │
        ▼
output/
  ├─ raw_data/*.json          (audit trail; reusable for re-scoring)
  ├─ targets_summary.csv      (composite-ranked flat table)
  └─ targets_report.md        (per-gene markdown dossier; rationale slots
                               left for Claude to fill from the JSON cache)
```

## Install

```bash
# Claude Code — plugin marketplace
> /plugin marketplace add Agents365-ai/365-skills
> /plugin install target-prioritization

# Or any agent (Cursor, Copilot, etc.) via the SkillsMP CLI
npx skills add Agents365-ai/365-skills -g

# Manual install — clone and symlink into ~/.claude/skills/
git clone https://github.com/Agents365-ai/target-prioritization.git
ln -s "$PWD/target-prioritization/skills/target-prioritization" \
      ~/.claude/skills/target-prioritization
```

No dependencies beyond Python 3.9+ and a `curl`-capable network.

## Usage

Just describe what you want — the skill triggers on any DE-list triage request:

```
> Filter these candidate genes for druggable targets — TP53, EGFR, MYC, KRAS, BRCA1

> Make a target dossier for the top 50 genes in /path/to/de_output.csv

> Re-rank these with the druggability weight dialed up to 0.4
```

The skill is domain-agnostic and works equally well for oncology
(TP53 / EGFR / MYC), neurodegeneration (APP / TREM2 / SNCA),
metabolic disease (PCSK9 / GCK / PNPLA3), autoimmunity, and any other
human-gene context. The focus-disease and cell-context PubMed queries
plus the OpenTargets focus-disease tag are user-editable — see
[Retargeting](#retargeting-for-a-different-disease--cell-context) below.

Or call the orchestrator directly:

```bash
python3 ~/.claude/skills/target-prioritization/scripts/orchestrate.py \
    --input  /path/to/de_output.csv \
    --output /tmp/targets_run1 \
    --top 50
```

After the run, ask Claude to fill in the rationale slots of `targets_report.md` using `prompts/rationale_template.md`.

## Output format

`targets_summary.csv` — one row per gene, sortable in Excel/pandas. Columns include:

| Field group | Example columns |
|---|---|
| Score | `composite_score`, `tier`, plus per-component (`druggability`, `disease_genetics`, `tractability`, `tissue_specificity`, `cell_context_score`, `expression`, `novelty`, `over_studied_penalty`) |
| UniProt | `uniprot_id`, `protein_name`, `subcellular_location`, `is_surface`, `is_secreted`, `is_mhc`, `has_transmembrane` |
| OpenTargets | `approved_drug_count`, `highest_clinical_phase`, `any_focus_disease_drug`, `focus_disease_drugs`, `tractability_small_molecule`, `tractability_antibody` |
| Disease genetics | `any_disease_assoc`, `is_focus_disease_associated`, `focus_disease_traits`, `max_focus_disease_assoc_score`, `max_disease_assoc_score` |
| PubMed | `pubmed_total`, `pubmed_focus_disease`, `pubmed_cell_context`, `maturity_tag` |
| HPA | `hpa_tissue_specificity_tag`, `hpa_tissue_top_types`, `hpa_cell_specificity_tag`, `hpa_cell_top_types`, `hpa_focus_cell_hits`, `hpa_expression_cluster`, `hpa_n_prognostic_cancers`, `hpa_cancer_specificity` |

Tiers (after min-max rescaling): `Tier-1-priority` (≥0.75), `Tier-2-candidate` (≥0.50), `Tier-3-watchlist` (≥0.30), `Tier-4-deprioritized` (<0.30).

`targets_report.md` — markdown dossier with one section per gene, sorted by composite score. Each section has a metadata table plus blank rationale + next-step slots for Claude to fill from the JSON cache.

## Retargeting for a different disease / cell context

The skill ships with an autoimmunity + T-cell default but is intentionally
disease-agnostic. Three edits switch the focus:

- `skills/target-prioritization/scripts/fetch_opentargets.py` and
  `scripts/aggregate.py` — set `FOCUS_DISEASE_TERMS` to the lowercased
  substrings that should tag a drug or associated-disease entry as
  "in-scope":

  | Domain | Example `FOCUS_DISEASE_TERMS` | Example `FOCUS_CELL_TYPES` (HPA single-cell names) |
  |---|---|---|
  | Oncology | `("cancer", "carcinoma", "lymphoma", "leukemia", "tumor")` | `("Macrophages", "Fibroblasts", "T-cells")` |
  | Neurodegeneration | `("alzheimer", "parkinson", "huntington", "als")` | `("Excitatory neurons", "Microglial cells", "Astrocytes")` |
  | Metabolic / liver | `("diabetes", "obesity", "fatty liver", "nash")` | `("Hepatocytes", "Kupffer cells")` |
  | Cardiovascular | `("heart failure", "atherosclerosis", "myocardial", "hypertension")` | `("Cardiomyocytes", "Endothelial cells")` |

- `scripts/aggregate.py::FOCUS_CELL_TYPES` — must match HPA's exact
  cell-type strings (case-sensitive). Drives `cell_context_score`.
- `scripts/fetch_pubmed.py::CONTEXTS` — adjust the `focus_disease` and
  `cell_context` PubMed query templates. The `cell_context` slot accepts
  any cell-type / lineage string and is used only for the dossier counts.

CSV columns are already neutrally named (`focus_disease_*`,
`cell_context`, `hpa_focus_cell_hits`); no downstream code changes are
needed after retargeting.

## Composite score

```
composite = w1 · druggability      + w2 · disease_genetics + w3 · tractability
          + w4 · tissue_specificity + w5 · cell_context_score
          + w6 · expression         + w7 · novelty
          - w8 · over_studied_penalty
```

All weights live in `weights.yaml` and can be overridden per run with `--weights`. Re-scoring with new weights costs ~1s (no API re-fetch):

```bash
python3 ~/.claude/skills/target-prioritization/scripts/aggregate.py \
    --raw-dir /tmp/targets_run1/raw_data \
    --output-dir /tmp/targets_run1 \
    --weights ~/.claude/skills/target-prioritization/weights.yaml \
    --input-csv /path/to/expression_table_pass_either_1s.csv
```

## Data sources

| Source | What it gives | Rate-limit handling |
|---|---|---|
| **UniProt REST** | Protein name, function summary, subcellular location, surface / secreted / MHC flags, transmembrane, signal peptide | 100 req/sec, batched via `accession` queries |
| **OpenTargets GraphQL** | Ensembl ID, tractability (SM + Ab + Pr + OC), approved drugs, max clinical phase, focus-disease-tagged drugs, associated diseases (integrates GWAS Catalog + other genetics sources) | Generous, single endpoint |
| **PubMed E-utilities** | Total paper count + two user-configurable context counts (`focus_disease`, `cell_context`) + maturity tag | 3 req/sec without API key |
| **Human Protein Atlas** | Tissue / single-cell specificity tag, top tissue nTPM, top single-cell nCPM, expression cluster, prognostic cancer count + cancer specificity | None documented; fetcher sleeps 0.15s/gene |

## Compared to native Claude Code

| Capability | Native Claude Code | target-prioritization |
|---|---|---|
| Look up UniProt for one gene | ✅ via web | ✅ batched + structured |
| Look up OpenTargets drugs / disease assoc | ✅ via web | ✅ schema-mapped, focus-disease-tagged |
| Count PubMed papers (+ disease context) | ⚠ slow, manual queries | ✅ parallel, deduped |
| Composite re-ranking + tiering | ❌ | ✅ configurable weights |
| Reproducible audit trail | ❌ | ✅ raw JSON cache |
| Re-score without re-fetching | ❌ | ✅ ~1s rerun |

## When NOT to use this skill

- **Single-gene lookups** — overkill; ask Claude to web-search instead
- **Non-human genes** — the APIs are human-centric; fetchers silently return empty
- **Cancer driver analysis** — use CGC / OncoKB
- **Pure literature review without target ambition** — use `scholar-deep-research` or `literature-review` skill

## 🔗 Related skills

| Skill | When to use |
|---|---|
| [scholar-deep-research](https://github.com/Agents365-ai/scholar-deep-research) | If the rationale step needs deeper literature evidence per gene |
| [paper-fetch](https://github.com/Agents365-ai/paper-fetch) | Pulling the full text of the focus-disease PMIDs surfaced by this skill |

## 💬 Community

- **Discord:** https://discord.gg/79JF5Atuk
- **WeChat:** scan the QR code below

<p align="center">
  <img src="https://raw.githubusercontent.com/Agents365-ai/images_payment/main/qrcode/agents365ai_wechat_1.png" width="200" alt="WeChat Community Group">
</p>

## 👤 Author

**Agents365-ai**

- GitHub: https://github.com/Agents365-ai
- Bilibili: https://space.bilibili.com/441831884

## 📄 License

[PolyForm Noncommercial 1.0.0](LICENSE) — free for personal, academic, public-research, and other noncommercial use. Commercial use requires a separate license from the author.
