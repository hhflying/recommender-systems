# Recommender Systems: Architecture & Engineering

A practical, experience-based series on recommender system architecture and engineering — from offline feature pipelines to online real-time ranking.

[简体中文](../zh/README.md) | [Back to Home](../README.md)

---

## Status

English translations are planned; no English articles are available yet. Chinese originals are available for architecture articles 01–13 and two engineering articles. Architecture article 14 is a draft.

The tables below track planned translations of the current Chinese articles. Links in the **Original (中文)** column lead to Chinese content. English titles will link to translations as they become available. Both series are ongoing.

## Reading Guide

- **Architecture**: Read in order to explore core modules, data flows, service boundaries, and the connections between offline and online systems.
- **Engineering**: Read by topic for implementation details and design trade-offs, including similarity-based caching and Process Chain scheduling.

## Table of Contents

### Architecture Series

| #   | Title                                              | Translation status | Original (中文)                                                     |
| --- | -------------------------------------------------- | ------------------ | ----------------------------------------------------------------- |
| 01  | Recommender System Architecture Overview           | Planned            | [推荐系统架构全景](../zh/architecture/01-overview.md)                     |
| 02  | Engineering Evolution of Recommendation Algorithms | Planned            | [推荐算法的工程演进](../zh/architecture/02-algo.md)                        |
| 03  | Feature Engineering                                | Planned            | [特征工程](../zh/architecture/03-feature.md)                          |
| 04  | Offline System Overview                            | Planned            | [离线系统总览](../zh/architecture/04-offline.md)                        |
| 05  | Content Asset System                               | Planned            | [内容资产体系](../zh/architecture/05-content.md)                        |
| 06  | User Feedback Loop                                 | Planned            | [用户反馈闭环](../zh/architecture/06-feedback.md)                       |
| 07  | Feature Pipeline and Feature Store                 | Planned            | [特征 Pipeline 和 Feature Store](../zh/architecture/07-pipeline.md)  |
| 08  | User Profile System                                | Planned            | [用户画像系统](../zh/architecture/08-userprofile.md)                    |
| 09  | Model Training Pipeline                            | Planned            | [模型训练流程](../zh/architecture/09-model.md)                          |
| 10  | Online System Overview                             | Planned            | [在线系统总览](../zh/architecture/10-online.md)                         |
| 11  | Traffic Splitter                                   | Planned            | [Traffic Splitter](../zh/architecture/11-traffic.md)              |
| 12  | Real-time Recommendation Engine                    | Planned            | [实时推荐引擎](../zh/architecture/12-engine.md)                         |
| 13  | Multi-channel Recall                               | Planned            | [多路召回](../zh/architecture/13-recall.md)                           |
| 14  | Ranking and Prediction Server                      | Planned            | [精排与 Prediction Server](../zh/architecture/14-ranking.md) (draft) |

### Engineering Practice Series

| #   | Title                                                     | Translation status | Original (中文)                                |
| --- | --------------------------------------------------------- | ------------------ | -------------------------------------------- |
| 01  | Cache Design: The Foundation of Sub-100ms Recommendations | Planned            | [Cache 设计](../zh/engineering/01-cache.md)    |
| 02  | Chain-based Scheduling Model: Design and Implementation   | Planned            | [链式调度模型设计和实现](../zh/engineering/02-chain.md) |

## About the Author

**Huang Hui** — A backend architect specializing in recommender systems and content distribution, with nearly two decades of server-side development and system design experience. Previously a Principal Software Engineer at Microsoft, he has also held core engineering and technical leadership roles at NIO, Cheetah Mobile, Yahoo!, and Baidu, focusing on search, recommendation systems, and distributed platform architecture.

- Personal site: [huanghui.net](https://huanghui.net/)
- GitHub: [@hhflying](https://github.com/hhflying)

## License

Articles, images and diagrams are licensed under [CC BY 4.0](../LICENSE-CC-BY-4.0) — you may share, adapt and use them commercially with attribution. Source code (if any) is licensed under [MIT](../LICENSE-MIT).

© Huang Hui
