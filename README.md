# Recommender Systems: Architecture & Engineering

A practical series on recommender system architecture and engineering by Huang Hui, based on experience building large-scale production systems. Topics span offline data and feature pipelines, online retrieval and ranking, and the engineering decisions that connect them.

基于大规模推荐系统生产实践，系统梳理从离线数据与特征生产，到在线召回、排序与工程实现的核心链路。

[简体中文](./zh/README.md) | [English](./en/README.md)

---

## What's Inside

- **Architecture**: Core modules, data flows, service boundaries, and how offline and online systems work together.
- **Engineering**: Concrete designs and implementation trade-offs, including similarity-based caching and Process Chain scheduling.

| Series | Chinese content | English translations |
| --- | --- | --- |
| Architecture | 13 completed articles; article 14 is a draft | Planned; none available yet |
| Engineering | 2 articles available; series ongoing | Planned; none available yet |

Start with the [Chinese reading guide](./zh/README.md) or browse the [English translation roadmap](./en/README.md). Both series will continue to grow.

## Repository Structure

```
.
├── README.md                ← Project overview & license
├── zh/
│   ├── README.md            ← Chinese index & reading guide
│   ├── architecture/        ← Chinese architecture articles
│   └── engineering/         ← Chinese engineering practice articles
├── en/
│   ├── README.md            ← English index & reading guide
│   ├── architecture/        ← English architecture articles (planned)
│   └── engineering/         ← English engineering articles (planned)
├── assets/
│   ├── architecture/        ← Images & diagrams
│   └── engineering/
├── LICENSE-CC-BY-4.0        ← License for articles, images & diagrams
└── LICENSE-MIT              ← License for source code (if any)
```

The empty English article directories shown above are planned and will appear in Git when translations are added.

## License

This repository applies different licenses for **article content** and **source code**:

- **Articles, Markdown documents, images and diagrams** (under `zh/`, `en/`, `assets/`):
  Licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
  See [`LICENSE-CC-BY-4.0`](./LICENSE-CC-BY-4.0).
  [![CC BY 4.0](https://i.creativecommons.org/l/by/4.0/88x31.png)](https://creativecommons.org/licenses/by/4.0/)

  > You are free to share, adapt, and use these works for commercial purposes, as long as appropriate attribution is given.

- **Sample scripts, utility source code** (if any):
  Licensed under the [MIT License](https://opensource.org/licenses/MIT).
  See [`LICENSE-MIT`](./LICENSE-MIT).

© Huang Hui.
