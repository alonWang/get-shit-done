# GSD 完整流程梳理

> 通过一个真实示例演示 get-shit-done 工具的完整工作流程。

---

## 为什么要这样设计？

### 核心问题：Context Rot（上下文腐化）

传统 AI 辅助开发的典型失败模式：

```
用户："帮我做一个博客系统"
Claude：好的，这里是 200 个任务的完整计划...
         [50 个任务后，Claude 开始忘记早期决策]
         [100 个任务后，生成的代码与最初的架构相矛盾]
         [用户发现问题，一切从头来过]
```

随着对话增长，上下文窗口被旧代码、旧讨论填满，Claude 对原始意图的把握越来越弱。

### 解决思路

GSD 通过三个核心原则对抗上下文腐化：

| 原则 | 机制 | 效果 |
|------|------|------|
| **原子化执行** | 每个 Plan 只有 2-5 个任务，每个执行子 Agent 获得全新的 200k 上下文 | 不因积累而退化 |
| **分层决策** | 先锁定"做什么"（Spec），再锁定"怎么做"（Discuss），最后执行 | 执行时无歧义 |
| **文件链路** | 每步产出结构化文件，下一步读文件而非读对话历史 | 上下文精准可控 |

### 选项驱动的交互哲学

GSD 尽量让用户**选择**而非**输入**，原因是：

- **自由输入**容易产生歧义（"现代感"、"快一点"——这些词无法转化为代码）
- **预设选项**经过 Claude 分析代码库后生成，已包含权衡说明
- **"Other" 出口**始终存在，用户可以随时切换到自由表达

```
Claude: 评论列表如何排序？
  ○ 最新在前（默认，适合讨论场景）
  ○ 最早在前（适合有时序的内容，如教程）
  ○ 点赞数排序（需要点赞功能，当前 Phase 未包含）
  ● Other → 用户输入: "按楼层排序，像论坛那样"
```

当用户选 "Other" 并表达想自由描述时，系统立刻切换为纯文本对话，不再强行呈现选项。

---

## 整体流程

```
[首次] /gsd-new-project
         │
         ▼
    .planning/
    ├── PROJECT.md       ← 项目愿景与原则
    ├── REQUIREMENTS.md  ← 需求清单（带 REQ-ID）
    ├── ROADMAP.md       ← Phase 划分与目标
    ├── STATE.md         ← 当前进度与决策日志
    └── config.json      ← 模式配置

         │
    ┌────▼──────────────────────────────────────────────┐
    │              主循环（每个 Phase 重复）              │
    │                                                    │
    │  [可选] /gsd-spec-phase N                          │
    │         ↓ 锁定"做什么"                              │
    │         → {N}-SPEC.md                             │
    │                                                    │
    │  Step 1: /gsd-discuss-phase N                      │
    │         ↓ 锁定"怎么做"                              │
    │         → {N}-CONTEXT.md                          │
    │         → {N}-DISCUSSION-LOG.md                   │
    │                                                    │
    │  Step 2: /gsd-plan-phase N                         │
    │         ↓ 生成原子任务计划                           │
    │         → {N}-RESEARCH.md                         │
    │         → {N}-01-PLAN.md, {N}-02-PLAN.md ...      │
    │                                                    │
    │  Step 3: /gsd-execute-phase N                      │
    │         ↓ 波次并行执行                               │
    │         → {N}-01-SUMMARY.md, {N}-02-SUMMARY.md ...│
    │         → {N}-VERIFICATION.md                     │
    │                                                    │
    │  Step 4: /gsd-verify-work N                        │
    │         ↓ 人工验收                                  │
    │         → {N}-HUMAN-UAT.md                        │
    │                                                    │
    │  Step 5: /gsd-ship N                               │
    │         ↓ 创建 PR                                   │
    └────────────────────────────────────────────────────┘
         │
         ▼
    下一个 Phase...
```

---

## 真实示例：为 Next.js 博客添加评论功能

**背景：** 已有一个 Next.js 博客，有文章列表和详情页。现在要给每篇文章加上评论区——用户可以发评论、查看评论、删除自己的评论。

这个功能涉及 DB schema、API 端点、UI 组件，横跨多个关注点，是演示完整流程的理想案例。

---

### 命令实现机制（先读懂工具本身）

在开始示例之前，先理解 GSD 的命令是如何工作的。

**命令即 Markdown 文件：**

```
get-shit-done/
├── commands/
│   └── gsd/
│       ├── discuss-phase.md   ← Claude Code slash command 定义
│       ├── plan-phase.md
│       ├── execute-phase.md
│       └── ...（80+ 个命令）
├── get-shit-done/
│   ├── workflows/             ← 实际的工作流逻辑
│   │   ├── discuss-phase.md
│   │   ├── plan-phase.md
│   │   └── ...
│   ├── templates/             ← 文件模板
│   └── references/            ← 规范文档
└── bin/
    └── install.js             ← 安装脚本
```

**每个命令文件的结构（以 `discuss-phase.md` 为例）：**

```markdown
---
name: gsd:discuss-phase
description: Gather phase context through adaptive questioning...
argument-hint: "<phase> [--all] [--auto] [--chain] [--batch]"
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUserQuestion    ← 允许向用户提问
  - Task               ← 允许启动子 Agent
---

<objective>
提取下游 Agent 需要的实现决策...
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/discuss-phase.md   ← 引入实际工作流
@~/.claude/get-shit-done/templates/context.md
</execution_context>
```

**安装流程：** `bin/install.js` 将 `commands/gsd/` 和 `get-shit-done/` 目录复制到 `~/.claude/`，使 Claude Code 能够识别这些 slash commands。

**工作流文件**（`workflows/discuss-phase.md`）是真正的业务逻辑所在——它用自然语言描述了 Claude 应该执行的每一步，包括何时调用 `AskUserQuestion`、何时启动 `Task` 子 Agent、如何处理各种边界情况。

**SDK（`sdk/src/`）** 是 TypeScript 辅助层，提供 `template.select`、`plan-parser`、`query` 等工具函数，供工作流在需要时调用。

---

### 交互模式谱系

GSD 支持从完全手动到完全自动的连续谱系：

```
慢 ◄────────────────────────────────────────────► 快

全手动          批量问答        半自动         全自动
(默认)         (--batch)      (--chain)      (--auto / yolo)

每步确认        多问题          Discuss 后     跳过所有
             一次呈现         自动 Plan+     问答，Claude
                            Execute       自选推荐项
```

| 模式 | 命令写法 | 适用场景 |
|------|---------|---------|
| 全手动 | `/gsd-discuss-phase 1` | 新功能、重要决策，需要细细确认 |
| 批量 | `/gsd-discuss-phase 1 --batch` | 时间有限，想一次性回答多个问题 |
| 半自动 | `/gsd-discuss-phase 1 --chain` | 讨论完毕后无需再触发后续命令 |
| 全自动 | `/gsd-discuss-phase 1 --auto` | 熟悉的技术栈，信任 Claude 的判断 |
| Yolo | `config.json: "mode": "yolo"` | 快速迭代，全程无确认 |

---

### 第零步：初始化项目

```bash
/gsd-new-project
```

**Claude 做什么：**
1. 询问 5-10 个问题（用 `AskUserQuestion`，每次 1-2 个）
2. 调查已有代码库（读 `package.json`、目录结构、现有模型）
3. 进行领域研究（竞品分析、技术栈可行性）
4. 生成需求清单和路线图

**用户交互示例：**

```
Claude: 这个项目要解决什么核心问题？
  ○ 为读者提供互动讨论空间（适合社区型博客）
  ○ 让读者给文章留下反馈（适合个人博客/教程网站）
  ○ 建立作者与读者的双向沟通渠道
  ● Other → "我想要评论功能，像 Medium 那种"

Claude: 用户需要登录才能评论吗？
  ● 是的，需要账号
  ○ 支持匿名评论（需要防垃圾评论机制）
  ○ 两种都支持
```

**产出文件：**

```
.planning/
├── PROJECT.md
│   ├── "What This Is: Next.js 博客评论系统"
│   ├── "Core Value: 让读者能在文章下互动"
│   └── "Constraints: Next.js 14, PostgreSQL, 已有 Prisma"
│
├── REQUIREMENTS.md
│   ├── COMMENT-01: 已登录用户可在文章下发评论
│   ├── COMMENT-02: 所有人可查看评论列表
│   ├── COMMENT-03: 用户可删除自己的评论
│   └── AUTH-01: 依赖已有的用户认证系统
│
├── ROADMAP.md
│   ├── Phase 1: 评论数据层（DB schema + Prisma model）
│   ├── Phase 2: 评论 API（CRUD 端点）
│   └── Phase 3: 评论 UI（评论区组件）
│
└── config.json
    └── { "mode": "interactive", "workflow": { "research": true } }
```

---

### 第一步：Discuss Phase — 锁定"怎么做"

```bash
/gsd-discuss-phase 1
```

**Claude 做什么：**
1. 读取 ROADMAP.md 了解 Phase 1 的目标（"评论数据层"）
2. 扫描代码库——发现已有 `prisma/schema.prisma`，有 `User`、`Post` 模型
3. 识别"灰色地带"（需要用户决策的实现细节）：
   - 评论是否支持嵌套回复？
   - 软删除还是硬删除？
   - 需要哪些字段？
4. 呈现灰色地带，让用户选择要讨论哪些

**用户交互（多选灰色地带）：**

```
Claude: Phase 1 涉及以下实现决策，请选择你想讨论的：
  ☑ 评论结构（是否支持嵌套/回复）
  ☑ 删除策略（软删除 vs 硬删除）
  ○ 字段设计（内容长度限制、时间戳等）← 未选，Claude 自行决定
```

**逐项讨论（以"评论结构"为例）：**

```
Claude: 评论结构
  Phase 1 要构建数据模型，这个决策会影响 schema 设计。

  支持嵌套回复（如 Reddit 风格）？
  ● 否，只有顶层评论（简单，适合博客场景）
  ○ 支持一层回复（parent_id 字段，中等复杂度）
  ○ 支持多层嵌套（递归结构，实现复杂）

Claude: 删除策略
  ● 软删除（is_deleted 字段，保留数据，支持审计）
  ○ 硬删除（直接从 DB 删除，更简单）
```

**产出文件：**

`.planning/phases/01-comment-data-layer/01-CONTEXT.md`
```markdown
# Phase 1: Comment Data Layer - Context

## Phase Boundary
构建评论数据模型。只做 DB schema 和 Prisma model，不包含 API 和 UI。

## Implementation Decisions

### 评论结构
- D-01: 只支持顶层评论，不支持嵌套回复
- D-02: 不需要 parent_id 字段

### 删除策略
- D-03: 软删除，使用 is_deleted: Boolean 字段
- D-04: 软删除时保留内容（deletedAt 记录时间）

### Claude's Discretion
- 内容最大长度（建议 2000 字符）
- 索引策略

## Existing Code Insights
- 已有 User model（id, email, name）
- 已有 Post model（id, title, content, authorId）
- 使用 Prisma，迁移通过 prisma migrate dev
```

`.planning/phases/01-comment-data-layer/01-DISCUSSION-LOG.md`
```markdown
# Phase 1: Discussion Log（仅供人工审阅，不被 Agent 读取）

## 评论结构
| 选项 | 描述 | 选择 |
|------|------|------|
| 顶层评论 | 简单，适合博客 | ✓ |
| 一层回复 | parent_id 字段 | |
| 多层嵌套 | 递归结构 | |

## 删除策略
用户选择：软删除（保留审计能力）
```

---

### 第二步：Plan Phase — 生成原子任务

```bash
/gsd-plan-phase 1
```

**Claude 做什么：**
1. **领域研究**：调查 Prisma 最佳实践、软删除模式
2. **生成计划**：基于 CONTEXT.md 的决策创建原子任务
3. **计划检查**：验证计划能否实现 Phase 目标，不通过则迭代
4. **呈现摘要**（非 `--auto` 模式）：让用户确认

由于 Phase 1 只有一个关注点，生成一个 Plan：

**产出文件：**

`.planning/phases/01-comment-data-layer/01-01-PLAN.md`
```markdown
---
phase: 01-comment-data-layer
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - prisma/schema.prisma
  - prisma/migrations/
autonomous: true
requirements: [COMMENT-01, COMMENT-02, COMMENT-03]
must_haves:
  truths:
    - "prisma db push 执行成功，无报错"
    - "Comment model 包含 is_deleted 字段"
  artifacts:
    - prisma/schema.prisma（包含 Comment model）
  key_links:
    - "Comment.authorId → User.id"
    - "Comment.postId → Post.id"
---

<objective>
在 Prisma schema 中添加 Comment model，并生成迁移文件。
Purpose: 为评论 API 提供数据基础。
Output: 更新后的 schema.prisma + 迁移文件。
</objective>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/phases/01-comment-data-layer/01-CONTEXT.md
@prisma/schema.prisma
</context>

<tasks>
<task type="auto">
  <name>Task 1: 在 schema.prisma 添加 Comment model</name>
  <files>prisma/schema.prisma</files>
  <action>
    在 Post model 之后添加 Comment model：
    - id: String @id @default(cuid())
    - content: String @db.VarChar(2000)
    - authorId: String（外键 → User）
    - postId: String（外键 → Post）
    - isDeleted: Boolean @default(false)
    - deletedAt: DateTime?
    - createdAt: DateTime @default(now())
    - updatedAt: DateTime @updatedAt
    同时在 User 和 Post model 上添加反向关系字段 comments Comment[]
  </action>
  <verify>grep -n "model Comment" prisma/schema.prisma</verify>
  <done>schema.prisma 包含完整 Comment model，关系字段正确</done>
</task>

<task type="auto">
  <name>Task 2: 生成并应用迁移</name>
  <files>prisma/migrations/</files>
  <action>运行 npx prisma migrate dev --name add-comment-model</action>
  <verify>npx prisma validate</verify>
  <done>迁移文件生成，prisma validate 无错误</done>
</task>
</tasks>

<verification>
npx prisma validate
npx prisma db push --preview-feature（干跑验证）
</verification>
```

---

### 第三步：Execute Phase — 波次并行执行

```bash
/gsd-execute-phase 1
```

**Claude 做什么（编排层，~15% 上下文）：**
1. 读取所有 PLAN.md，分析依赖关系
2. 计算执行波次（Phase 1 只有一个 Plan，一个波次）
3. 为每个 Plan 启动独立的执行子 Agent（`Task` 工具）
4. 每个子 Agent 获得**全新 200k 上下文**，只读需要的文件
5. 合并完成，运行测试套件

**多 Phase 时的波次示例（Phase 2，评论 API）：**

```
Wave 1（并行）：
  ├── Plan 2.01: GET /api/posts/[id]/comments（只读）
  └── Plan 2.02: POST /api/posts/[id]/comments（写入）

Wave 2（并行，依赖 Wave 1）：
  └── Plan 2.03: DELETE /api/comments/[id]（需要已有 comment 路由）
```

**每个子 Agent 执行完毕，创建：**

`.planning/phases/01-comment-data-layer/01-01-SUMMARY.md`
```markdown
---
phase: 01-comment-data-layer
plan: 01
provides:
  - Comment Prisma model（schema + 迁移）
  - User.comments 和 Post.comments 反向关系
key-files:
  created: [prisma/migrations/20250101_add-comment-model/]
  modified: [prisma/schema.prisma]
requirements-completed: [COMMENT-01, COMMENT-02, COMMENT-03]
duration: 3min
---

## 完成情况
添加了 Comment model，包含软删除支持（isDeleted + deletedAt）。
迁移已应用，prisma validate 通过。

## 关键决策
- 使用 cuid() 作为 ID（与项目其他 model 一致）
- isDeleted 而非 deletedAt 作为主要软删除标志（查询更简单）
```

---

### 第四步：Verify Work — 人工验收

```bash
/gsd-verify-work 1
```

**Claude 做什么：**
1. 从 Phase 目标提取可测试的验收项
2. 逐一呈现，用户确认是否通过

**交互示例：**

```
Claude: 请验证以下内容（Phase 1: 评论数据层）

  ✓ 验收项 1: Comment model 存在于 schema.prisma？
    ○ 通过
    ○ 失败 → 描述问题

  ✓ 验收项 2: 数据库迁移已成功应用？
    ● 通过
    ○ 失败

  ✓ 验收项 3: Comment 与 User、Post 有正确的外键关系？
    ● 通过
    ○ 失败
```

若用户报告失败：Claude 自动创建 gap-closure 计划，运行 `/gsd-execute-phase 1 --gaps-only` 修复，然后再次验收。

**产出文件：**

`.planning/phases/01-comment-data-layer/01-HUMAN-UAT.md`
```markdown
# Phase 1: Human UAT Results
所有验收项通过。无 gap。
```

---

### 第五步：Ship — 创建 PR

```bash
/gsd-ship 1
```

Claude 自动生成 PR，标题和描述从 Phase 目标与 SUMMARY.md 提取，过滤掉 `.planning/` 目录的提交。

---

### 后续 Phase

Phase 1 完成后，继续：

```bash
/gsd-discuss-phase 2   # 评论 API
/gsd-plan-phase 2
/gsd-execute-phase 2
/gsd-verify-work 2
/gsd-ship 2

/gsd-discuss-phase 3   # 评论 UI
...
```

或用 `/gsd-manager` 打开仪表盘，并行管理多个 Phase：

```
════════════════════════════════════════════════════════
 GSD ► 博客评论功能
════════════════════════════════════════════════════════
 ██████░░░░░░░░░░░ 33%  (1/3 phases)

 | # | Phase 名称     | D | P | E | 状态         |
 |---|----------------|---|---|---|-------------|
 | 1 | 评论数据层     | ✓ | ✓ | ✓ | ✓ 已完成    |
 | 2 | 评论 API       | ✓ | ○ | · | ○ 可以规划  |
 | 3 | 评论 UI        | ○ | · | · | · 待讨论    |

▶ 下一步
  → Plan Phase 2（后台运行）
  → Discuss Phase 3（内联交互）
```

---

## 关键文件产出清单

| 文件 | 创建时机 | 被谁读取 |
|------|---------|---------|
| `.planning/PROJECT.md` | new-project | 所有 Agent |
| `.planning/REQUIREMENTS.md` | new-project | discuss, plan, executor |
| `.planning/ROADMAP.md` | new-project | 所有 Agent |
| `.planning/STATE.md` | new-project，每步更新 | 所有 Agent（会话起点） |
| `{N}-SPEC.md` | spec-phase（可选） | discuss, verifier |
| `{N}-CONTEXT.md` | discuss-phase | researcher, planner |
| `{N}-DISCUSSION-LOG.md` | discuss-phase | 仅人工审阅 |
| `{N}-RESEARCH.md` | plan-phase | planner |
| `{N}-NN-PLAN.md` | plan-phase | execute 子 Agent |
| `{N}-NN-SUMMARY.md` | execute-phase | orchestrator，未来 planner |
| `{N}-VERIFICATION.md` | execute-phase | verify-work |
| `{N}-HUMAN-UAT.md` | verify-work | ship |

---

## 命令速查表

```bash
# 项目初始化（只做一次）
/gsd-new-project
/gsd-new-project --auto          # 自动模式，Claude 自选推荐项

# 每个 Phase 的完整流程
/gsd-discuss-phase 1             # 全手动讨论
/gsd-discuss-phase 1 --batch     # 批量问答
/gsd-discuss-phase 1 --chain     # 讨论后自动接 plan + execute
/gsd-discuss-phase 1 --auto      # 全自动，跳过所有问答
/gsd-plan-phase 1
/gsd-execute-phase 1
/gsd-execute-phase 1 --interactive   # 每个 Plan 前暂停确认
/gsd-verify-work 1
/gsd-ship 1

# 导航
/gsd-next                        # 自动检测下一步该做什么
/gsd-manager                     # 打开多 Phase 管理仪表盘

# 实用工具
/gsd-quick [任务描述]             # 临时任务，不走完整流程
/gsd-debug [问题描述]             # 调试已有问题
/gsd-execute-phase 1 --gaps-only # 只修复验收失败的 gap
```
