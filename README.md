# CryptoTrace
Cryptosporidium tracing &amp; genotyping platform
## 项目目录结构（Cp / Ch 物种模块分开，gp60 / mlst / wgs-snp 三分型）

下面这套结构同时适用于：

* **静态站**（你离线跑完分析，把结果丢进 `db/`，前端只读 JSON/TSV）
* **动态站**（后端跑分析，把结果写入同样的结构，前端不需要改）

---

# 1) 顶层目录

```text
crypto-typing-site/
├── web/                       # 前端（静态站页面、JS、CSS）
│   ├── index.html
│   ├── assets/
│   └── app/                   # 你的前端代码（React/Vue/原生都行）
│
├── db/                        # 数据库（前端只读这里）
│   ├── index.json             # 全站索引（物种列表、统计、版本号）
│   ├── schemas/               # 字段规范（可选，但强烈建议保留）
│   ├── collections/           # 公开合集（按物种组织）
│   ├── Cp/                    # C. parvum 模块（独立）
│   └── Ch/                    # C. hominis 模块（独立）
│
├── pipelines/                 # 分析流程脚本（离线或后端作业都从这里跑）
│   ├── 00_reference/
│   ├── 01_gp60/
│   ├── 02_mlst/
│   ├── 03_wgs_snp/
│   └── 99_export_db/
│
├── docs/                      # 论文方法/平台说明/版本变更
└── README.md
```

---

# 2) `db/` 的统一约定（前端只认这个“数据协议”）

## 2.1 全站索引：`db/index.json`

```json
{
  "db_version": "2026-02-02",
  "species": [
    {"id": "Cp", "name": "Cryptosporidium parvum"},
    {"id": "Ch", "name": "Cryptosporidium hominis"}
  ],
  "modules": ["gp60", "mlst", "wgs_snp"],
  "counts": {
    "Cp": {"n_samples": 0},
    "Ch": {"n_samples": 0}
  }
}
```

---

# 3) 物种模块内部结构（Cp / Ch 完全同构）

以 `db/Cp/` 为例（`db/Ch/` 一模一样）：

```text
db/Cp/
├── meta/
│   ├── species.json                 # 物种模块信息（版本、参考基因组、scheme版本）
│   ├── samples.tsv                  # 样本元数据主表（核心）
│   └── dictionaries/                # 可选：国家/宿主/项目名等字典，方便前端展示
│       ├── host_map.json
│       └── country_map.json
│
├── gp60/
│   ├── typing.tsv                   # 每样本 gp60 分型结果
│   ├── alleles.fasta                # gp60 等位型库（可选：对外展示/下载）
│   ├── summary.json                 # subtype 频率统计（前端直接读）
│   └── index.json                   # gp60 模块索引（文件路径、统计、版本号）
│
├── mlst/
│   ├── scheme.json                  # Cp_MLST_v1 定义（loci 列表、长度、规则）
│   ├── typing.tsv                   # 每样本 MLST allele + ST 结果
│   ├── alleles/                     # 每个位点 allele 库（TSV/FASTA）
│   │   ├── CP47.fasta
│   │   ├── MSC6-7.fasta
│   │   └── ...
│   ├── profiles.tsv                 # ST 定义表（allele组合→ST）
│   ├── summary.json                 # ST 频率统计
│   └── index.json
│
└── wgs_snp/
    ├── wgs.json                     # WGS-SNP 规则（参考、mask、过滤、阈值）
    ├── tree.nwk                     # 全局树（Newick）
    ├── clusters.tsv                 # 聚类结果（cluster_id、level、成员数）
    ├── sample_cluster.tsv           # 每样本所属cluster（可分多尺度level）
    ├── nn_topk.json                 # 每样本最近邻TopK（强烈推荐）
    ├── qc.tsv                       # 每样本WGS QC（callable sites、missing等）
    ├── summary.json                 # cluster规模、分布统计
    └── index.json
```

> **关键点**：前端不需要懂你的分析细节，只要这些文件齐、字段一致，就能展示。

---

# 4) 核心表字段规范（最少必备字段）

## 4.1 `meta/samples.tsv`（全站最重要的一张表）

**每个物种一张**，例如 `db/Cp/meta/samples.tsv`

建议字段（TSV）：

```text
sample_id    species   project   collection_id   host   host_group   country   admin1   city   latitude   longitude   collection_date   platform   read_layout   sra_accession   coverage_est   notes
```

* `sample_id`：全站唯一（推荐 `Cp_000001` 这种）
* `collection_id`：可为空；用于 Collections 页面
* `host_group`：比如 human/cattle/goat/wildlife（方便筛选）
* `latitude/longitude`：没有就空，但字段保留
* `coverage_est`：估计覆盖度，WGS模块QC也会用

---

## 4.2 gp60：`gp60/typing.tsv`

```text
sample_id   gp60_subtype   gp60_allele_id   identity   coverage   depth   qc_flags
```

* `qc_flags`：用 `;` 分隔，如 `LOW_COVERAGE;MIXED_ALLELES`

---

## 4.3 MLST：`mlst/typing.tsv`

```text
sample_id   scheme   ST   missing_loci   locus_CP47   locus_MSC6-7   locus_DZHRGP   locus_HSP70   locus_ACTIN   qc_flags
```

* `scheme`：建议固定为 `Cp_MLST_v1` / `Ch_MLST_v1`
* 每个位点列写 allele number（缺失写 `NA`）
* `missing_loci`：整数
* `qc_flags`：如 `MIXED_LOCI;LOW_DEPTH_CP47`

---

## 4.4 WGS-SNP：最近邻 `wgs_snp/nn_topk.json`

结构建议（前端最省事）：

```json
{
  "TopK": 10,
  "metric": "snp_distance",
  "items": {
    "Cp_000001": [
      {"sample_id": "Cp_000120", "dist": 12},
      {"sample_id": "Cp_000087", "dist": 15}
    ],
    "Cp_000002": [
      {"sample_id": "Cp_000099", "dist": 3}
    ]
  }
}
```

---

## 4.5 WGS-SNP：聚类

`wgs_snp/sample_cluster.tsv`

```text
sample_id   level   cluster_id
Cp_000001   L1      Cp_WGSclu_0007
Cp_000001   L2      Cp_WGSclu_0312
```

`wgs_snp/clusters.tsv`

```text
level   cluster_id        n_members   max_pairwise_dist   countries_top   hosts_top
L1      Cp_WGSclu_0007    38          25                  US;UK;CN        human;cattle
```

---

# 5) Collections 结构（可选但很像 Pathogenwatch）

```text
db/collections/
├── index.json
├── Cp/
│   ├── Cp_col_0001.json
│   └── Cp_col_0002.json
└── Ch/
    └── Ch_col_0001.json
```

每个 collection JSON 建议包含：

* collection 名称、简介、引用信息
* 样本列表（或筛选规则）
* 预计算统计（gp60/ST/cluster 频率）

---

# 6) 版本与可追溯（强烈建议你现在就固化）

每个物种模块 `meta/species.json`：

```json
{
  "species_id": "Cp",
  "species_name": "Cryptosporidium parvum",
  "reference": {
    "genome": "Cp_ref_v1",
    "source": "CryptoDB/NCBI",
    "build_date": "2026-01-20"
  },
  "typing_versions": {
    "gp60_lib": "gp60lib_v1",
    "mlst_scheme": "Cp_MLST_v1",
    "wgs_snp": "Cp_WGS_v1"
  }
}
```

---

# 7) 你按这个结构，MVP 最少需要生成哪些文件？

对每个物种（Cp 与 Ch）至少要有：

1. `meta/samples.tsv`
2. `gp60/typing.tsv` + `gp60/summary.json`
3. `mlst/typing.tsv` + `mlst/summary.json`
4. `wgs_snp/tree.nwk`
5. `wgs_snp/sample_cluster.tsv` + `wgs_snp/clusters.tsv`
6. `wgs_snp/nn_topk.json`
7. `wgs_snp/qc.tsv`

做到这 7 项，你就能实现：

* Genomes（样本库浏览）
* Typing（gp60/MLST/WGS-SNP 三视图）
* Map（按 subtype/ST/cluster 着色）
* Tree（按 subtype/ST/cluster 上色 + 点样本联动信息）

---

