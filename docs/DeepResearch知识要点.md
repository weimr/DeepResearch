# DeepResearch 知识要点

> 唯一口径：源项目 `d:/ActTo/DeepResearch`（10/20 已删除笔记库 `agi_notes/projects/DeepResearch` 整份过期副本——它缺 `frontend/src/lib/` 四个文件）。
> 数字纪律：凡数字必须标来源（ADR-xxx 原文 / `docs/项目工作量.md`）；未记录的写「待取数」，绝不编。

## 1. 概览与定位

- README 原文定位（第 3 行，照抄）：「当前项目展示了如何使用Langgraph搭建一个DeepResearch的应用」
- 自拟口径（含价值半句，标注为自己的提炼）：以 LangGraph 编排 Plan → Research → Write 多 Agent 子图、接阿里百炼 MCP 做搜索，把宽泛研究话题自动产成**带引用、经评估**的研究报告
- README 通读漏项：「项目结构」只列 frontend / backend，**未提 `docs/`**（实际含 12 篇 ADR + 项目工作量 + 大模型安全设计 + 评估框架 + 测试指引）
- 规模数字（来源＝项目工作量.md，非 ADR）：13,964 行 / 84 文件 / 约 10.25 人月 / 测试源码比 0.94:1 → 取值时间待按定格原则确认
- 关键目录（10/20 已圈定，明细表见 1018.md ⓪ 段 ③）
  - backend ×3：`graph.py`（196 行，MainGraph 六节点＋条件路由＋子图组装）／`base_agent.py`（628 行全项目最大，Agent 类层次＋RateLimiter 令牌桶＋`_retry_with_classified_errors`）／`state.py`（71 行，8 个 TypedDict＋Annotated reducer）
  - frontend ×3：`App.tsx`（294 行，双通道 `useStream`+`useResearchStream`、四种提交场景路由）／`lib/useResearchStream.ts`（242 行，SSE 提交、token 流式、Plan 暂停继续、`Last-Event-ID` 断线重连）／`components/ActivityTimeline.tsx`（155 行，节点事件时间线）
  - 弃用说明：`main.tsx`（71 行入口）无取舍可讲；`prompts.py`（477 行／11 个模板）归入提示词维度，不占三张牌

## 2. 架构决策（四格：问题 / 选型 / 放弃项 / 取舍）

### 2.1 决策链地图（前段 001-008）

<!-- 贴 1018.md ① 表精华；出口标准要求本节存在 -->
待填

### 2.2 决策链地图（后段 009-012）

待填（W2）

### 2.3 依赖关系（照 ADR_README 抄，用于串讲演进主线）

- 001 → 002 → 003 → 004 → {009, 012, 008}；007 → 008（007 是 008 的前提）；010 独立增强；011 横切
- 业务约束（不纳入 ADR 范围）：CI/CD、分布式追踪、灰度发布 / 多租户与权限 / 公网安全防护 → 面试问「还想做什么」时的边界答案

## 3. 架构设计：多 Agent 拆分与协作（主挂 ADR-003）

- 三个子图职责：主图只管 Plan 确认与调度 / ResearchAgent 封装「搜索→评估→补搜」循环 / WriterAgent 封装「提纲→草稿→Critic→修订」循环
- 协作方式：子图各自独立编译，主图条件边串联；状态经 `OverallState` 传递（字段 8 → 23，来源 ADR-003）
- Debate Loop 退出条件：`ready_for_polish=True` 或 `revision_count >= 3`；Critic 评分 <6 自动修订（来源 ADR-003）
- README 自述四节点：generate_search / web_search / critique / final_answer

## 4. 知识库：选型 / 设计 / 验证（主挂 ADR-004 + 009 + 012）

- 选型：Milvus 自托管（数据不出域、Dynamic Field 免 migration、IVF_FLAT 适配 10 万级事实量级）；**Redis 只做搜索缓存，不做向量检索**（纯 Redis 不支持向量检索，来源 ADR-004）
- 缓存降级：Redis 不可用 → 进程内内存缓存，业务无感（来源 ADR-004）
- 生命周期四档：off / inform / freshness（默认）/ lifecycle；敏感度阈值 7 / 30 / 180 天；衰减公式 `decay = max(0.3, 1 - age/(2·max_age))`（来源 ADR-009）
- 精排：DashScope `gte-rerank`，KB 与 Web 双独立开关，召回扩展 `limit = max(top_k*3, 20)`，`rerank()` 永不抛异常只降级（来源 ADR-012）
- 验证：见第 5 节（LLM-as-Judge 组件级 + 分层测试）

## 5. 质量与上线标准（主挂 ADR-005 + 011）

- 分层测试：Unit ~120 <5s ／ Integration ~90 <15s ／ E2E ~4 5-10min；214 个测试 <20s（来源 ADR-011）
- Mock 边界：LLM／MCP／Milvus／Embedding 四类隔离；**Redis 用真实本地实例不 mock**（来源 ADR-011）
- LLM-as-Judge：E2E 5 维度 × 1-5 分 ＋ 组件级 7 维度；单次完整评估 10+ 次 LLM 调用，只用于回归与重大变更（来源 ADR-005）
- 评估结果：`eval_report_20260613_142601.json` → 综合得分待取数
- 口径隔离：以上数字来源均为 ADR／评估报告，**与中台 AI 应用助手 2026-10-18 线上定格数字不同源，不可混用**

## 6. 待取数与欠账登记

| 项 | 需要的数 | 可能来源 | 状态 |
|----|----------|----------|------|
| 评估综合得分 | E2E 5 维度均分 | eval_report JSON | ⏳ 待取数 |
| 项目规模口径 | 人月／行数（若对外引用） | 项目工作量.md（实测时间 2026-06-13） | ⏳ 待确认定格口径 |
| 主线深挖 | graph.py 精读 | 源码 | 挂 W2 Day1 时段⓪ |
