# target-prioritization — 候选基因的多源药靶尽调技能 🎯

[![License: PolyForm-NC](https://img.shields.io/badge/License-PolyForm--NC%201.0.0-orange.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/Agents365-ai/target-prioritization?style=flat&logo=github)](https://github.com/Agents365-ai/target-prioritization/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/Agents365-ai/target-prioritization?style=flat&logo=github)](https://github.com/Agents365-ai/target-prioritization/network/members)
[![Latest Release](https://img.shields.io/github/v/release/Agents365-ai/target-prioritization?logo=github)](https://github.com/Agents365-ai/target-prioritization/releases/latest)
[![Last Commit](https://img.shields.io/github/last-commit/Agents365-ai/target-prioritization?logo=github)](https://github.com/Agents365-ai/target-prioritization/commits/main)

[![SkillsMP](https://img.shields.io/badge/SkillsMP-listed-1f6feb)](https://skillsmp.com/)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-8a2be2)](https://github.com/Agents365-ai/365-skills)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-compatible-2ea44f)](https://agentskills.io)
[![Discord](https://img.shields.io/badge/Discord-Join-5865F2?logo=discord&logoColor=white)](https://discord.gg/79JF5Atuk)

[English](README.md) · **中文**

外部参考:[UniProt REST](https://www.uniprot.org/help/api) · [OpenTargets GraphQL](https://platform-docs.opentargets.org/data-access/graphql-api) · [PubMed E-utilities](https://www.ncbi.nlm.nih.gov/books/NBK25501/) · [Human Protein Atlas](https://www.proteinatlas.org/about/help/dataaccess)

把一份排序好的候选基因列表(通常是 scRNA-seq 差异表达输出)变成逐基因的药靶尽调档案 —— 并行查询 UniProt / OpenTargets / PubMed / Human Protein Atlas,按可配置的复合分数重新排序。

- **多源证据** —— UniProt(亚细胞定位、表面 / 分泌 / MHC 标签)、OpenTargets(可成药性、获批药物、疾病关联,内含 GWAS 等遗传学证据)、PubMed(总论文数 + 可配置的疾病聚焦 + 细胞 / 谱系场景计数)、Human Protein Atlas(组织 / 单细胞 specificity 标签 + nCPM、表达 cluster、癌症 prognostic 概览)
- **并行抓取** —— Python `ThreadPoolExecutor` 同时调度所有来源
- **复合分重排** —— `druggability / disease_genetics / tractability / tissue_specificity / cell_context_score / expression / novelty / over_studied` 各项分数全部可在 `weights.yaml` 中按需调权
- **零 API key、零 Python 第三方依赖** —— 仅需 stdlib + `curl` 可达网络
- **可重打分** —— raw JSON 缓存让你改完 `weights.yaml` 后秒级 rerun,无需重新拉接口

兼容 Claude Code、OpenClaw 与 SkillsMP —— 任何支持 [Agent Skills](https://agentskills.io) 规范的 agent 都能加载。

## 🔄 工作流程

```
输入基因列表(CSV / TXT)
        │
        ▼
scripts/orchestrate.py
        │
        ├─► fetch_uniprot.py        → 蛋白定位、表面 / 分泌 / MHC、信号肽
        ├─► fetch_opentargets.py    → 可成药性、获批药物、聚焦疾病的临床试验、
        │                              疾病关联(整合 GWAS 等遗传证据)
        ├─► fetch_pubmed.py         → 总文献数 + 聚焦疾病计数 + 细胞场景计数、
        │                              成熟度标签
        └─► fetch_hpa.py            → HPA 组织 / 单细胞 specificity 与 nCPM、
                                       表达 cluster、癌症 prognostic 概览
        │
        ▼
scripts/aggregate.py — 复合分 + 分层
        │
        ▼
output/
  ├─ raw_data/*.json          (审计快照,可用于重新打分)
  ├─ targets_summary.csv      (按复合分排序的平铺表)
  └─ targets_report.md        (逐基因 markdown 档案;rationale 留空给 Claude 填写)
```

## 安装

```bash
# Claude Code —— 通过 plugin marketplace
> /plugin marketplace add Agents365-ai/365-skills
> /plugin install target-prioritization

# 任意 agent(Cursor、Copilot 等)—— 通过 SkillsMP CLI
npx skills add Agents365-ai/365-skills -g

# 手动安装 —— clone 后软链到 ~/.claude/skills/
git clone https://github.com/Agents365-ai/target-prioritization.git
ln -s "$PWD/target-prioritization/skills/target-prioritization" \
      ~/.claude/skills/target-prioritization
```

依赖只有 Python 3.9+ 和能访问 `curl` 的网络。

## 使用方式

直接用自然语言描述即可,只要涉及 DE 列表筛选 / 三层化 / 药靶尽调,技能就会自动触发:

```
> 帮我把这些候选基因按可成药性筛一下:TP53、EGFR、MYC、KRAS、BRCA1

> 给 /path/to/de_output.csv 里 DE 排名前 50 的基因做一份靶点档案

> 把 druggability 权重调到 0.4 重打一次分
```

技能对学科领域无偏:肿瘤(TP53 / EGFR / MYC)、神经退行(APP / TREM2 / SNCA)、代谢
病(PCSK9 / GCK / PNPLA3)、自免、心血管等任何人类基因场景都可用。其中聚焦疾病
/ 细胞场景的 PubMed 查询以及 OpenTargets 的「聚焦疾病」标签都可一行改 —— 详见
下方的[切换学科/细胞场景](#切换学科--细胞场景)。

或直接调用 orchestrator:

```bash
python3 ~/.claude/skills/target-prioritization/scripts/orchestrate.py \
    --input  /path/to/de_output.csv \
    --output /tmp/targets_run1 \
    --top 50
```

跑完后让 Claude 按 `prompts/rationale_template.md` 给 `targets_report.md` 填 rationale 与建议下一步。

## 输出格式

`targets_summary.csv` —— 每个基因一行,可直接在 Excel / pandas 中排序。列分组示例:

| 字段组 | 示例列 |
|---|---|
| 分数 | `composite_score`、`tier`,以及各分量(`druggability`、`disease_genetics`、`tractability`、`tissue_specificity`、`cell_context_score`、`expression`、`novelty`、`over_studied_penalty`) |
| UniProt | `uniprot_id`、`protein_name`、`subcellular_location`、`is_surface`、`is_secreted`、`is_mhc`、`has_transmembrane` |
| OpenTargets | `approved_drug_count`、`highest_clinical_phase`、`any_focus_disease_drug`、`focus_disease_drugs`、`tractability_small_molecule`、`tractability_antibody` |
| 疾病遗传 | `any_disease_assoc`、`is_focus_disease_associated`、`focus_disease_traits`、`max_focus_disease_assoc_score`、`max_disease_assoc_score` |
| PubMed | `pubmed_total`、`pubmed_focus_disease`、`pubmed_cell_context`、`maturity_tag` |
| HPA | `hpa_tissue_specificity_tag`、`hpa_tissue_top_types`、`hpa_cell_specificity_tag`、`hpa_cell_top_types`、`hpa_focus_cell_hits`、`hpa_expression_cluster`、`hpa_n_prognostic_cancers`、`hpa_cancer_specificity` |

分层(min-max 归一后):`Tier-1-priority` (≥0.75)、`Tier-2-candidate` (≥0.50)、`Tier-3-watchlist` (≥0.30)、`Tier-4-deprioritized` (<0.30)。

`targets_report.md` —— 按复合分排序、逐基因一节的 markdown 档案;每节有元信息表与留空的 rationale + 下一步槽位,由 Claude 读 JSON 缓存后填写。

## 切换学科 / 细胞场景

技能默认配置面向自免 + T 细胞,但本身是学科无关的。三处编辑即可切换:

- `scripts/fetch_opentargets.py` 和 `scripts/aggregate.py::FOCUS_DISEASE_TERMS`
  —— 用于在 OT 药物适应症和 disease association 里识别「在范围内」的小写关键词:

  | 学科 | 示例 `FOCUS_DISEASE_TERMS` | 示例 `FOCUS_CELL_TYPES`(HPA 单细胞名,大小写敏感) |
  |---|---|---|
  | 肿瘤 | `("cancer", "carcinoma", "lymphoma", "leukemia", "tumor")` | `("Macrophages", "Fibroblasts", "T-cells")` |
  | 神经退行 | `("alzheimer", "parkinson", "huntington", "als")` | `("Excitatory neurons", "Microglial cells", "Astrocytes")` |
  | 代谢 / 肝 | `("diabetes", "obesity", "fatty liver", "nash")` | `("Hepatocytes", "Kupffer cells")` |
  | 心血管 | `("heart failure", "atherosclerosis", "myocardial", "hypertension")` | `("Cardiomyocytes", "Endothelial cells")` |

- `scripts/aggregate.py::FOCUS_CELL_TYPES` —— 决定 `cell_context_score` 中
  哪些 HPA 单细胞类型算作「目标细胞」(必须与 HPA 字符串完全一致)。
- `scripts/fetch_pubmed.py::CONTEXTS` —— 调整 `focus_disease` 和 `cell_context`
  的 PubMed 查询模板,仅用于 dossier 里的文献计数。

CSV 字段名已经使用中性命名(`focus_disease_*`、`cell_context`、
`hpa_focus_cell_hits`),切换后下游不需要任何改动。

## 复合分

```
composite = w1 · druggability      + w2 · disease_genetics + w3 · tractability
          + w4 · tissue_specificity + w5 · cell_context_score
          + w6 · expression         + w7 · novelty
          - w8 · over_studied_penalty
```

权重全部在 `weights.yaml` 里,跑命令时可 `--weights` 临时覆盖。改完权重重新打分仅 ~1s,不重新拉接口:

```bash
python3 ~/.claude/skills/target-prioritization/scripts/aggregate.py \
    --raw-dir /tmp/targets_run1/raw_data \
    --output-dir /tmp/targets_run1 \
    --weights ~/.claude/skills/target-prioritization/weights.yaml \
    --input-csv /path/to/expression_table_pass_either_1s.csv
```

## 数据来源

| 来源 | 提供字段 | 限流处理 |
|---|---|---|
| **UniProt REST** | 蛋白名 / 功能摘要 / 亚细胞定位 / 表面 / 分泌 / MHC / 跨膜 / 信号肽 | 100 req/s,按 `accession` 批量 |
| **OpenTargets GraphQL** | Ensembl ID、可成药性(SM + Ab + Pr + OC)、获批药物、最高临床期、聚焦疾病标签药物、疾病关联(整合 GWAS Catalog 等多种遗传学证据) | 单端点,限额宽松 |
| **PubMed E-utilities** | 总文献数 + 两个可配置场景计数(`focus_disease`、`cell_context`)+ 成熟度标签 | 无 API key 时 3 req/s |
| **Human Protein Atlas** | 组织 / 单细胞 specificity 标签、top tissue nTPM、top single-cell nCPM、表达 cluster、prognostic 癌种数、cancer specificity | 无显式限流;fetcher 每基因 sleep 0.15s |

## 与 Claude Code 原生能力对比

| 能力 | Claude Code 原生 | target-prioritization |
|---|---|---|
| 单基因 UniProt 查询 | ✅ 通过 web | ✅ 批量、结构化 |
| OpenTargets 药物 / 疾病关联 | ✅ 通过 web | ✅ schema 映射、聚焦疾病自动打标 |
| PubMed 文献计数(含疾病场景) | ⚠ 慢、需手动反复查询 | ✅ 并行、去重 |
| 复合分 + 分层 | ❌ | ✅ 权重可配置 |
| 可复核审计快照 | ❌ | ✅ raw JSON 缓存 |
| 不重新拉接口的重打分 | ❌ | ✅ ~1s |

## 何时 **不要** 使用本技能

- **单基因查询** —— 杀鸡用牛刀;让 Claude 直接 web-search 即可
- **非人类基因** —— 接口以人为主,fetcher 会静默返回空
- **癌症 driver 分析** —— 建议用 CGC / OncoKB
- **不带药靶意图的纯文献综述** —— 用 `scholar-deep-research` 或 `literature-review`

## 🔗 相关技能

| 技能 | 使用场景 |
|---|---|
| [scholar-deep-research](https://github.com/Agents365-ai/scholar-deep-research) | rationale 阶段需要更深入的逐基因文献证据时 |
| [paper-fetch](https://github.com/Agents365-ai/paper-fetch) | 抓取本技能列出的聚焦疾病 PMID 全文 |

## 💬 社区

- **Discord:** https://discord.gg/79JF5Atuk
- **微信:** 扫描下方二维码

<p align="center">
  <img src="https://raw.githubusercontent.com/Agents365-ai/images_payment/main/qrcode/agents365ai_wechat_1.png" width="200" alt="微信交流群">
</p>

## 👤 作者

**Agents365-ai**

- GitHub: https://github.com/Agents365-ai
- Bilibili: https://space.bilibili.com/441831884

## 📄 License

[PolyForm Noncommercial 1.0.0](LICENSE) —— 个人、学术、公共研究及其它非商业用途免费使用。商用需另行向作者获取授权。
