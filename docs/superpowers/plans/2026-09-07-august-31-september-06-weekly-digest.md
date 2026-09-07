# 2026-08-31 至 2026-09-06 LLM 推理优化周报 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增经过一手来源核验、按技术重要性排序的 `2026-09-06.md`，覆盖 2026-08-31 至 2026-09-06 的 LLM 推理优化动态。

**Architecture:** 先从 GitHub 官方数据源全量采集目标窗口内 vLLM 与 SGLang 的 merged PR 和 Release，再从 arXiv 与项目或厂商官方页面采集论文和技术动态。原始数据保存在未提交的 `.cache/`，最终将互斥分类、去重和重要性排序后的信息写入单一周报文件。

**Tech Stack:** PowerShell、GitHub REST/Search API、arXiv、Markdown、Git。

## Global Constraints

- 时间窗固定为 2026-08-31 00:00 至 2026-09-06 23:59（UTC）。
- 仅统计在时间窗内实际 merged 的 PR，关闭但未合并的 PR 不计入。
- PR、Release、论文、模型和厂商动态优先使用一手来源。
- PR 的性能、显存和能耗数字必须明确归因为作者声明，不能写成独立复现结论。
- 每个 PR 只计入一个主类别；系列 PR 可以合并展示，但分类统计按独立 PR 计数。
- 只改动本任务相关文件，不暂存或提交现有 `.claude/`。
- 本地提交允许执行；任何 `git push` 前必须列出分支、提交和目标 remote，并取得用户明确确认。

## File Map

- Create: `2026-09-06.md` — 最终可发布的中文周报。
- Create: `docs/superpowers/plans/2026-09-07-august-31-september-06-weekly-digest.md` — 本实施计划。
- Read only: `2026-08-30.md` — 栏目、术语、信息密度和 Markdown 格式基线。
- Temporary, do not commit: `.cache/2026-09-06/` — GitHub 查询结果、论文候选和核验记录。

---

### Task 1: 全量采集 GitHub PR 与 Release

**Files:**
- Create temporarily: `.cache/2026-09-06/vllm-prs.json`
- Create temporarily: `.cache/2026-09-06/sglang-prs.json`
- Create temporarily: `.cache/2026-09-06/releases.json`

**Interfaces:**
- Consumes: GitHub 查询 `repo:vllm-project/vllm is:pr is:merged merged:2026-08-31..2026-09-06` 与 `repo:sgl-project/sglang is:pr is:merged merged:2026-08-31..2026-09-06`。
- Produces: 每条含 `number`、`title`、`html_url`、`user.login`、`labels`、`body`、`created_at` 和精确 `merged_at` 的完整 PR 记录，以及两个仓库各自的唯一 PR 总数。

- [ ] **Step 1: 创建任务缓存目录**

  Run: `New-Item -ItemType Directory -Force '.cache/2026-09-06'`

  Expected: 目录存在，且后续 `git status --short` 不把缓存文件列为待提交内容；若仓库未忽略 `.cache/`，则只保留在工作区且绝不执行 `git add .`。

- [ ] **Step 2: 分页查询两个仓库的 merged PR**

  使用 GitHub Search API 的上述精确查询逐页读取，每页 100 条，直到已读取数等于 `total_count`。Search API 返回的 `pull_request.url` 再用于读取 PR 详情中的 `merged_at`；任何一页限流或失败时重试该页，不使用首屏数量推算总数。

- [ ] **Step 3: 查询两个仓库的 Releases**

  读取 `https://api.github.com/repos/vllm-project/vllm/releases?per_page=30` 和 `https://api.github.com/repos/sgl-project/sglang/releases?per_page=30`，以 `published_at` 是否位于 UTC 时间窗内为准保留候选；同时记录窗口内没有 release 的仓库，避免把无发布误写成遗漏。

- [ ] **Step 4: 验证采集完整性**

  对两个仓库分别断言：本地记录数等于 API `total_count`，PR 编号唯一，`merged_at >= 2026-08-31T00:00:00Z` 且 `merged_at < 2026-09-07T00:00:00Z`。把总数和失败断言写入核验记录，不满足时回到 Step 2 补采。

### Task 2: 分类全部 PR 并选出高价值条目

**Files:**
- Create temporarily: `.cache/2026-09-06/pr-classification.md`

**Interfaces:**
- Consumes: Task 1 的完整 PR 记录。
- Produces: 两个仓库各自互斥、可加总到总数的分类计数，以及按工程价值排序的正文候选。

- [ ] **Step 1: 为 vLLM 的全部 PR 指定唯一主类别**

  使用固定集合 `Perf`、`Model`、`Spec`、`KV`、`Quant`、`API`、`Bugfix`、`Docs`、`Other`。分类首先依据代码影响和 PR 正文，再参考标签；安全、崩溃、错误结果和生产回归归入 `Bugfix`，不能仅因同时包含性能变化而重复计数。

- [ ] **Step 2: 为 SGLang 的全部 PR 指定唯一主类别**

  使用固定集合 `Diffusion`、`Perf`、`Sched`、`Spec`、`KV`、`Model`、`Config`、`API`、`Bugfix`、`Docs`、`Other`。SGLang-Diffusion 专属实现归入 `Diffusion`；通用 kernel、调度和 runtime config 分别进入对应类别。

- [ ] **Step 3: 校验分类计数**

  分别验证 vLLM 九类之和等于 vLLM 总数、SGLang 十一类之和等于 SGLang 总数，并检查每个 PR 编号恰好出现一次。任何重复或缺失都先修正分类表，再开始正文节选。

- [ ] **Step 4: 按工程价值节选并合并系列**

  栏目内依次优先：跨框架或旗舰模型影响、明确的吞吐/延迟/显存收益、kernel/调度/KV/投机解码架构变化、高风险正确性或安全修复、一般模型/API/维护。每个展示项保留 PR 链接、编号、作者和一句中文技术说明；同目标系列可合并展示，但其成员编号都保留。

### Task 3: 采集新闻与论文

**Files:**
- Create temporarily: `.cache/2026-09-06/news-and-papers.md`

**Interfaces:**
- Consumes: arXiv、GitHub Releases、官方项目仓库、厂商公告和模型官方页面。
- Produces: 目标时间窗内经过日期与内容核验的框架发布、模型生态动态和推理优化论文候选。

- [ ] **Step 1: 核验框架发布与模型生态动态**

  对 Task 1 的 release 候选逐条检查 tag、`published_at` 和 release notes；另外只搜索 2026-08-31 至 2026-09-06 发布的官方模型、推理框架或基础设施公告。候选必须对部署、兼容性、性能、模型支持或推理架构有直接工程影响。

- [ ] **Step 2: 搜索目标主题论文**

  在 arXiv 搜索 `LLM inference`、`LLM serving`、`KV cache`、`speculative decoding`、`LLM quantization`、`inference parallelism`、`attention kernel` 和 `MoE inference`，只保留首次提交日期位于 `2026-08-31..2026-09-06` 的工作；排除只讨论训练且不包含推理方法或推理实验的论文。

- [ ] **Step 3: 逐条核验论文事实**

  从 arXiv 摘要页或原文确认标题、首次提交日期、方法核心、实验硬件或框架、性能基线和关键定量结果。正文中的数字使用“作者报告”“论文声称”等归因措辞，无法从一手来源确认的候选不收录。

- [ ] **Step 4: 去重并按决策价值排序**

  合并同一工作的多个版本或配套仓库；优先保留可改变 kernel、调度、KV、量化、并行或部署决策的材料。一般模型能力新闻和与推理无直接关联的行业新闻不进入周报。

### Task 4: 编写 `2026-09-06.md`

**Files:**
- Create: `2026-09-06.md`
- Read only: `2026-08-30.md`

**Interfaces:**
- Consumes: Tasks 1–3 的准确计数、分类表、代表性 PR、release、新闻和论文记录。
- Produces: 与仓库现有格式一致的完整中文 Markdown 周报。

- [ ] **Step 1: 建立固定标题结构**

  依次使用：`# LLM 推理优化周报（2026-08-31 ~ 2026-09-06，UTC）`、`## 本周观察摘要`、`## Part 1：vLLM & SGLang 上周 PR 汇总`、两个仓库子节、`## Part 2：上周大模型推理优化技术新闻`、框架发布、模型生态、论文和技术趋势子节。

- [ ] **Step 2: 写观察摘要与数据口径**

  用两至四段串联本周最重要的技术主线；随后明确列出两个仓库的 PR 总数、UTC 窗口、全量采集和互斥分类方法，并声明所有性能数字未经独立复现。

- [ ] **Step 3: 写 vLLM 与 SGLang 分类汇总**

  各栏目标题包含 Task 2 的精确分类数量。代表项统一写为 `[PR 标题](链接) — 作者 — 中文说明`；栏目末尾概括未展开的长尾工作，仓库末尾给出各类计数加总等式。

- [ ] **Step 4: 写新闻、论文与技术趋势**

  新闻和论文均链接到一手来源并注明窗口内日期；趋势使用“从本周数据看”等措辞标记归纳，且每条趋势都能由本期至少两项已核验材料支持。

### Task 5: 验证、审阅与本地提交

**Files:**
- Verify: `2026-09-06.md`
- Include: `docs/superpowers/plans/2026-09-07-august-31-september-06-weekly-digest.md`
- Exclude: `.claude/`, `.cache/`

**Interfaces:**
- Consumes: 完整周报和本实施计划。
- Produces: 通过结构、事实和工作区检查的本地 commit。

- [ ] **Step 1: 检查结构、占位文本和空白**

  Run: `rg -n '^#{1,4} ' 2026-09-06.md`

  Expected: 标题顺序与 Task 4 Step 1 一致，且两个仓库的分类标题完整。

  Run: `rg -n 'TBD|TODO|待补|PLACEHOLDER' 2026-09-06.md`

  Expected: 无输出。

  Run: `git diff --check`

  Expected: 无空白错误。

- [ ] **Step 2: 检查链接、日期、计数和重复编号**

  抽取 Markdown URL 并逐个发送只读请求；GitHub PR 必须可访问且 `merged_at` 位于目标时间窗。重新计算分类等式，并分别扫描两个仓库的 PR 编号，确认除明确合并展示外不存在重复引用。

- [ ] **Step 3: 审阅最终 diff 和提交范围**

  Run: `git diff -- 2026-09-06.md docs/superpowers/plans/2026-09-07-august-31-september-06-weekly-digest.md`

  Expected: 只包含本期周报和本计划；设计文档已在先前 commit 中。

  Run: `git status --short`

  Expected: `2026-09-06.md` 和本计划为待提交文件；`.claude/` 与 `.cache/` 不得暂存。

- [ ] **Step 4: 创建本地提交**

  Run: `git add -- 2026-09-06.md docs/superpowers/plans/2026-09-07-august-31-september-06-weekly-digest.md`

  Run: `git commit -m "Add 2026-09-06 weekly digest"`

  Expected: 新提交仅包含周报和实施计划，工作区中用户原有文件保持不变。

- [ ] **Step 5: 报告结果但不推送**

  列出本地分支、相对 `origin/main` 的新提交、验证结果和未跟踪的用户文件。除非用户在看到这些信息后明确确认，否则不执行 `git push`。
