# 交互式知识地图优化计划

> 状态：RFC 草案
>
> 目标：在不破坏现有 Markdown 知识库和稳定 `Pxxx` 标识的前提下，为人形机器人运动智能知识库增加可交互、可比较、可追溯的研究地图。

## 背景

当前知识库已经具备两项稀缺资产：

1. 以研发问题为中心的六条技术路线；
2. 对约 145 篇论文进行逐篇中文分析，而不是只维护链接列表。

现有阅读方式适合 Agent 检索，也适合读者沿目录逐页阅读，但面对大量论文时仍有三个摩擦点：

- 很难从一个技术问题快速反查相关论文及其差异；
- 很难区分一篇论文的主要贡献、次要贡献和仅被采用的工程组件；
- 很难观察同一问题在时间上的继承、分叉、组合与验证过程。

本提案的核心不是增加一张无约束的论文关系图，而是建立一层轻量的 **Contribution Knowledge Model**，让同一份内容可以同时生成 Roadmap、局部关系图、时间线、贡献矩阵和专题叙事。

相关方案调研见[知识可视化方案调研](知识可视化方案调研.md)。

## 产品原则

### 1. 人工 Roadmap 是主导航

六条技术路线继续作为一级结构。自动聚类、引用图和语义相似度只能用于补充推荐，不能覆盖人工分类。

### 2. Contribution 是一等实体

论文不能只与技术点建立“相关”关系。论文与技术点之间需要显式的贡献声明，说明：

- 解决了什么问题；
- 提出了什么新机制或新系统组织方式；
- 证据覆盖哪些机器人、任务和实验条件；
- 相比前作真正新增了什么；
- 尚未解决什么。

### 3. 默认展示局部视图

首页不展示包含全部节点的 force-directed graph。用户先从 Roadmap 进入某个技术点，再展开局部论文和贡献邻域，以减少视觉噪声并保留空间记忆。

### 4. 图与矩阵互补

关系图适合探索和传播；贡献矩阵适合严肃比较。两者必须来自同一数据源，并能够互相定位。

### 5. 静态优先、Git 原生

第一阶段不引入 Neo4j、在线编辑服务或账号系统。数据由 YAML/JSON Schema 管理，通过 CI 生成静态 JSON、JSONL 和可选 SQLite，再部署到 GitHub Pages。

## 建议数据模型

第一阶段定义五类稳定实体：

| ID | 实体 | 用途 |
|---|---|---|
| `Txxx` | Technical Point | Roadmap 中的问题、能力或系统环节 |
| `Pxxx` | Paper | 复用现有论文稳定编号 |
| `Kxxx` | Contribution Claim | 论文对某个技术点的主要或次要贡献 |
| `Rxxx` | Research Artifact | 代码、数据集、模型、仿真器或工具 |
| `Exxx` | Evidence | 真机实验、消融、指标或适用边界 |

后续可复用现有公司编号，并扩展机器人本体、实验室和组织实体。

### 论文约束

每篇论文建议满足：

- 恰好一个 Primary Contribution；
- 最多两个 Secondary Contribution；
- 恰好一个 Primary Roadmap Point；
- 最多两个 Secondary Roadmap Point；
- 工程组件和评测选择单独记录，不冒充贡献；
- 每项贡献指向具体章节、图表或官方材料。

示例：

```yaml
id: P033
roadmap:
  primary: T-SIM2REAL-03
  secondary:
    - T-TRACKING-05
contribution_ids:
  - K-P033-01
  - K-P033-02
evaluation_scope:
  robots:
    - humanoid-platform-a
  simulators:
    - simulator-a
  real_robot: true
```

```yaml
id: K-P033-01
paper_id: P033
roadmap_point_id: T-SIM2REAL-03
role: primary
claim:
  problem: 预设域随机化难以表达有结构的真实动力学偏差
  intervention: 使用真实 rollout 学习状态转移残差并修正训练动力学
  claimed_effect: 改善参考动作策略的真实机器人迁移
  limitation: 结论仍受机器人、动作分布和真实数据规模约束
relations:
  extends:
    - K-P028-01
```

## 信息架构

### Roadmap View

回答“这个领域有哪些核心问题”。一级路线与二级技术点位置固定，每个技术点展示论文数量、真机验证数量、开源数量和时间跨度。

### Contribution View

回答“同一个问题下，各篇论文分别做了什么”。点击技术点后，仅展示其贡献声明、关联论文和证据节点。

### Timeline View

回答“这条研究路线如何演进”。横轴采用时间，纵轴采用技术家族或 Roadmap 子问题；边必须具有明确语义，例如：

- `extends`
- `reframes`
- `combines`
- `scales`
- `improves_robustness`
- `validates_on_real_robot`

### Contribution Matrix

论文作为行，技术贡献点作为列，单元格区分 Primary、Secondary、Evidence 和 Component。该视图用于同期工作比较和研究空白识别。

### Story Mode

只为少量关键主题制作滚动叙事，不为每篇论文单独制作动画。候选主题：

- 从单动作跟踪到通用全身运动基座；
- 域随机化、系统辨识与残差动力学的差别；
- WBC、MPC 与 learned policy 如何组合；
- LocoManip 为什么不是 locomotion 与 manipulation 的简单拼接；
- VLA 在人形机器人控制栈中的真实边界。

## 技术方案

### 推荐组合

- 站点外壳：Observable Framework；
- 局部图：Cytoscape.js；
- 时间线、矩阵和叙事动画：D3 / Observable Plot；
- 图布局：Dagre 或 ELK，局部探索可选 fCoSE；
- 数据源：YAML + JSON Schema；
- 构建产物：`catalog.jsonl`、`graph.json`、可选 `knowledge.sqlite`；
- 部署：GitHub Actions + GitHub Pages。

当前规模没有必要引入图数据库。只有在多人在线编辑、复杂路径查询或实体规模达到数万级时，再评估 Neo4j 或 ORKG 后端。

## 目录草案

```text
data/
  roadmap/
  papers/
  contributions/
  artifacts/
  evidence/
schemas/
  roadmap.schema.json
  paper.schema.json
  contribution.schema.json
  artifact.schema.json
  evidence.schema.json
scripts/
  validate_entities.py
  build_catalog.py
  build_views.py
generated/
  catalog.jsonl
  graph.json
  knowledge.sqlite
site/
  src/
    index.md
    roadmap.md
    timeline.md
    matrix.md
```

现有 `论文逐篇解读/Pxxx.md` 继续作为长文内容，不要求第一阶段迁移。

## 分阶段实施

### Phase 0：Schema Spike

范围：选择 Motion Tracking、Robust Recovery 和 Sim2Real 中 8–12 篇论文。

交付：

- Roadmap Point、Paper、Contribution、Evidence 的最小 Schema；
- 一份标注规则；
- 一组人工标注样例；
- 校验脚本，检查 ID、Primary 数量和引用完整性。

验收：两名贡献者能够独立标注同一篇论文，并对 Primary Contribution 得到基本一致的结果。

### Phase 1：交互式 Companion MVP

范围：扩展到 20–30 篇论文。

交付：

- Roadmap View；
- 技术点局部 Contribution View；
- 论文详情抽屉；
- 按年份、真机、机器人、仿真器、开源状态筛选；
- GitHub Pages 自动部署；
- `catalog.jsonl` 与 `graph.json`。

验收：用户可以在三次点击内，从技术路线进入某篇论文，并理解其主要贡献、对比工作与证据范围。

### Phase 2：全量研究地图

范围：覆盖现有约 145 篇论文。

交付：

- Contribution Matrix；
- 时间约束研究演进图；
- OpenAlex / Semantic Scholar 元数据增量同步；
- SQLite 导出和来源报告；
- 论文、技术点、机器人和项目的独立详情页。

验收：能够稳定回答“某个技术点有哪些 Primary Contribution 且完成真机验证”等结构化查询。

### Phase 3：协作与专题叙事

交付：

- 基于 PR 的贡献标注模板；
- 冲突与复核规则；
- ORKG 兼容导出；
- 6–10 个 Story Mode 专题；
- 可选的标注辅助界面。

## 推荐 PR 拆分

1. `docs: add interactive knowledge map RFC and research notes`
2. `feat: add contribution knowledge schemas and validator`
3. `data: annotate initial motion tracking paper set`
4. `feat: add roadmap and local contribution graph prototype`
5. `ci: build and deploy interactive companion site`
6. `feat: add contribution matrix and timeline views`

首个实现 PR 不应直接改造全部论文，也不应一次引入完整在线平台。

## 风险与应对

| 风险 | 应对 |
|---|---|
| 贡献标签逐渐膨胀 | 强制一主两次，并将组件、实验选择独立建模 |
| 自动聚类与人工 Roadmap 冲突 | 自动结果只作为候选推荐，不写入主分类 |
| 大图不可读 | 固定 Roadmap 布局，按技术点懒加载局部邻域 |
| 内容迁移成本过高 | 先使用独立 YAML 样例，不改写现有长文 |
| 外部 API 不稳定 | 构建时缓存元数据，人工贡献字段不依赖外部服务 |
| 视觉效果优先于研究价值 | 以贡献矩阵和可追溯证据作为核心验收项 |

## 首个 Demo 建议

主题：**从 Motion Tracking 到通用人形运动基座**。

围绕五个问题组织：

1. 如何稳定跟踪单条参考动作；
2. 如何覆盖大规模动作分布；
3. 如何处理人体与机器人形态失配；
4. 如何在扰动或失配后恢复；
5. 如何从仿真迁移到真实机器人。

论文详情的首屏只显示：

- 它解决了什么；
- 最重要的贡献是什么；
- 与上一代方案的区别；
- 它没有解决什么。

之后再链接到现有 Pxxx 完整解读。
