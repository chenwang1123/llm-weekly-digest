# 2026-08-03 至 2026-08-09 LLM 推理优化周报 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增经过一手来源核验、按技术重要性排序的 `2026-08-09.md`，并补录 2026-07-27 至 2026-08-02 的相关论文。

**Architecture:** 先通过 GitHub API 全量采集目标窗口内 vLLM 与 SGLang 的合并 PR 和 Release，再从 arXiv 与官方站点采集论文和技术动态。原始数据只进入已忽略的 `.cache/`；最终把去重、分组和重要性排序后的结果写入单一周报文件。

**Tech Stack:** PowerShell、GitHub REST/Search API、arXiv、Markdown、Git。

## Global Constraints

- 时间窗口为 2026-08-03 00:00 至 2026-08-09 23:59（UTC）。
- 论文额外覆盖 2026-07-27 至 2026-08-02，并明确标记为上期补录。
- 关键事实使用 GitHub、arXiv、项目或厂商官方页面等一手来源。
- PR 性能数字明确归因于作者声明，不表述为独立复现结论。
- 节选顺序为：跨框架或旗舰模型影响、量化性能收益、核心架构变化、生产正确性、一般功能和维护。
- 保持现有 `.claude/` 未跟踪且不纳入提交。
- GitHub push 前列出分支、commit 与 remote，并取得用户明确确认。

## File Map

- Create: `2026-08-09.md` — 最终周报，沿用仓库现有单文件周报结构。
- Create: `docs/superpowers/plans/2026-08-10-august-03-09-weekly-digest.md` — 本实施计划。
- Read only: `2026-08-02.md`、`2026-07-26.md` — 格式、栏目和表述基线。
- Temporary, ignored: `.cache/` — API 原始响应和中间检查结果，不提交。

---

### Task 1: 全量采集 GitHub 变更

**Files:**
- Temporary: `.cache/vllm-2026-08-03_09.json`
- Temporary: `.cache/sglang-2026-08-03_09.json`
- Temporary: `.cache/releases-2026-08-03_09.json`

**Interfaces:**
- Consumes: GitHub Search API 查询 `repo:<owner/repo> is:pr is:merged merged:2026-08-03..2026-08-09`。
- Produces: 含 PR 编号、标题、作者、URL、创建/合并时间、标签与正文的完整记录；两个仓库各自的准确总数。

- [ ] **Step 1: 查询两个仓库的合并 PR 总数并翻完所有分页**

  使用 GitHub Search API，逐页请求到 `total_count` 覆盖完毕；若未认证限额不足，则使用 GitHub CLI 已有认证或按重置时间继续，禁止用首屏结果推算总数。

- [ ] **Step 2: 查询目标窗口附近的 Release**

  核对 vLLM 与 SGLang Releases API 的 `published_at`、tag、说明和原始链接，时间边界统一换算到 UTC。

- [ ] **Step 3: 校验采集完整性**

  对每个仓库验证：记录数等于 API `total_count`，PR 编号无重复，全部 `merged_at` 位于目标窗口。

### Task 2: 采集论文与技术新闻

**Files:**
- Temporary: `.cache/papers-2026-07-27_08-09.json`
- Temporary: `.cache/news-2026-08-03_09.json`

**Interfaces:**
- Consumes: arXiv 摘要页、论文原文、框架/模型/厂商官方公告。
- Produces: 分成 `2026-07-27..2026-08-02` 和 `2026-08-03..2026-08-09` 的论文候选，以及本周官方技术动态候选。

- [ ] **Step 1: 搜索并核验论文**

  主题限定为 LLM inference、serving、KV cache、speculative decoding、quantization、parallelism、attention/MoE kernels 和推理系统。逐条核对标题、作者、日期、摘要、论文 URL 与代码 URL；排除仅训练优化且与推理无直接关系的条目。

- [ ] **Step 2: 标注补录与新增**

  按 arXiv 首次提交或可核验发布日期分组；跨版本论文只保留一条，补录论文不得混入本周新增。

- [ ] **Step 3: 核验技术新闻**

  只保留在目标窗口内发布、且对 LLM 推理工程有实质影响的官方 Release、模型发布或基础设施公告。

### Task 3: 分类、去重与重要性排序

**Files:**
- Temporary: `.cache/selection-2026-08-09.md`

**Interfaces:**
- Consumes: Tasks 1–2 的完整候选集。
- Produces: 按现有栏目分类、系列合并、重要性降序排列的成稿提纲。

- [ ] **Step 1: 分类全部 PR**

  沿用现有栏目：性能/Kernel、模型支持、投机解码、调度/分布式、KV cache/Memory、扩散模型（SGLang）、API/重构、Bugfix、Docs、Other。

- [ ] **Step 2: 合并系列并去重**

  同一目标的连续 PR 合成一条，保留主要编号与链接；跨栏目只在技术归属最强的栏目出现一次，统计说明明确分类可能存在少量语义边界但不重复计算 PR 总数。

- [ ] **Step 3: 依重要性节选**

  先选跨框架/旗舰模型变化，再选有量化收益的优化、运行时架构变化和高风险正确性修复；普通 CI、依赖、文档和机械重构仅留代表项。

### Task 4: 编写周报

**Files:**
- Create: `2026-08-09.md`

**Interfaces:**
- Consumes: Task 3 的成稿提纲与一手来源链接。
- Produces: 可直接发布的中文 Markdown 周报。

- [ ] **Step 1: 写观察摘要与来源说明**

  用两至四条主线串联本周变化；列明两个仓库的合并 PR 总数、数据时间窗、节选原则和性能数字归因。

- [ ] **Step 2: 写两个仓库的分类 PR 汇总**

  每条包含 PR 编号、原始标题、作者、中文技术说明和 GitHub 链接；栏目内按重要性降序排列，尾部用准确措辞概括未展开长尾。

- [ ] **Step 3: 写新闻、论文和趋势**

  论文明确分为上期补录与本周新增；趋势段仅从已核验材料归纳，并用“从本周数据看”等措辞标示推断。

### Task 5: 验证与本地提交

**Files:**
- Verify: `2026-08-09.md`
- Include: `docs/superpowers/plans/2026-08-10-august-03-09-weekly-digest.md`

**Interfaces:**
- Consumes: 完整周报。
- Produces: 通过检查的本地 commit，以及等待 push 确认的准确摘要。

- [ ] **Step 1: 运行结构与文本检查**

  运行 `rg -n '^#{1,4} ' 2026-08-09.md` 和 `git diff --check`，并单独扫描常见占位标记；预期标题层级完整，无占位内容、尾随空格或空白错误。

- [ ] **Step 2: 运行链接、日期与计数检查**

  抽取全部 URL 检查状态；复核每个列出的 PR 的 merged 状态与日期，确认两个总数和论文分组，并检查 PR 编号在同一仓库内无非预期重复。

- [ ] **Step 3: 审阅最终 diff**

  运行 `git diff -- 2026-08-09.md docs/superpowers/plans/2026-08-10-august-03-09-weekly-digest.md`；确认 `.claude/` 和 `.cache/` 均未暂存。

- [ ] **Step 4: 创建本地 commit**

  暂存周报与计划文件，提交信息使用 `Add 2026-08-09 weekly digest (2026-08-03 ~ 2026-08-09)`。

- [ ] **Step 5: 请求 push 确认**

  报告当前分支、待推送 commits、目标 `origin` URL 与验证结果；在收到明确确认前不执行 `git push`。
