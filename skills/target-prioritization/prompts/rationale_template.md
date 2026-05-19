# Per-gene rationale template

When filling the **Rationale** and **Suggested next step** slots in
`targets_report.md`, follow this template per gene. Keep it tight — 2-3
sentences total for rationale, 1 sentence for next step.

## Rationale (2-3 sentences)

Sentence 1 — **most compelling evidence** (pick the single strongest signal
from the dossier row):
- If `is_ibd_associated` → "OpenTargets surfaces an IBD disease
  association (score [max_ibd_assoc_score]) — likely backed by GWAS /
  text-mining evidence…"
- Else if `any_ibd_drug` → "Already targeted by an approved IBD drug
  ([drug name])…"
- Else if `n_supporting_de_files >= 2` → "Cross-lineage DE convergence:
  significant in [N] sibling DE outputs…"
- Else if `is_surface` AND `highest_clinical_phase >= 3` → "Surface protein
  with phase III drugs in adjacent indications…"
- Else if `is_surface` AND `maturity_tag in {novel, moderate}` → "Surface
  protein with moderate prior immunology literature ([N] PubMed hits) —
  tractable for antibody/CAR approaches…"
- Else use whatever component scores highest in the breakdown row.

Sentence 2 — **main risk or caveat**:
- `maturity_tag = uncharted` → "Very thin literature ([N] hits) — risk of
  unknown off-target biology."
- `maturity_tag = saturated` → "Heavily studied ([N] hits); likely IP
  crowded."
- `is_mhc = True` → "MHC-family gene — broad-spectrum effects, hard to
  inhibit selectively."
- `composite` driven by single dimension only → "Score concentrated in one
  dimension — confirm with orthogonal evidence before pursuing."
- No GWAS + no convergence → "Stat signal from DE only; lacks genetic /
  cross-pipeline corroboration."

Sentence 3 (optional) — **specific UST/IBD context** if obvious from the
dossier (e.g. "Persists in NR Th1 cells at post-treatment, consistent with
IFN-γ axis escape").

## Suggested next step (1 sentence)

Be concrete. Examples:
- "siRNA knockdown in primary CD4 T cells from CD biopsies; readout: IL-17A
  + IFN-γ by intracellular cytokine staining."
- "IHC on archival CD vs healthy TI biopsies to confirm protein-level
  upregulation."
- "Treat ex vivo lamina propria T cells with [tool compound from
  approved_drugs] and compare with UST."
- "Cross-check expression in an independent UST-treated cohort (e.g.
  Martin 2019 GSE134809)."
- "If no tool compound exists: structure-based virtual screen against
  UniProt:[id]."

## Executive summary (top 5–10 genes)

3–5 sentences total at the top of the report. Cover:
1. How many genes scored Tier-1 vs Tier-2.
2. The 2–3 most compelling individual candidates and the headline reason.
3. Any pattern across the top genes (e.g. "5/10 are surface IFN-response
  genes, suggesting type I/II IFN axis is the dominant escape pathway").
4. Caveats / what's missing (e.g. "All candidates derive from a single
  contrast — recommend cross-checking against Th17 NR analysis").

Keep it factual. Do not invent biology not supported by the dossier rows
or the original DE context.
