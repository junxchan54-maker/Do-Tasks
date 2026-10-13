---
name: "dotasks"
description: "面向长工期任务的文件化任务管理：任务状态 + 进展事件流 + 阶段快照，支持跨 session 断点续接与隐私脱敏。触发词：记件事、待办、列一下任务、继续上次的任务"
allowed-tools:
  - read
  - write
  - edit
  - exec
---

# dotasks — 长任务断点续接 · 文件化任务管理

**核心思路：Agent 直接用 read/write/edit 操作 JSON Lines 文件，不依赖外部脚本。**

**目标不是"记待办"，而是任何一次新 session 都能在 30 秒内恢复到上次的现场。**

三类文件各司其职：

| 文件 | 回答的问题 | 写入方式 |
|------|-----------|---------|
| `tasks.jsonl` | 要做什么？什么状态？ | 追加为主，单行改动用 edit |
| `progress.jsonl` | 做到哪了？为什么这么做？ | **纯追加，永不修改** |
| `checkpoints/` | 这一刻的完整现场是什么？ | 每次新增一个文件 |

---

## 1. 文件位置与优先级

| 范围 | 路径 |
|------|------|
| 全局（默认） | `~/.dotasks/` |
| 项目级 | `<git-root>/.dotasks/` |

**优先级规则（必须先判断，不要两边都写）：**

1. 用户显式指定了范围 → 用用户指定的
2. 当前目录在 git 仓库内**且** `<git-root>/.dotasks/` 已存在 → 用项目级
3. 其他情况 → 用全局 `~/.dotasks/`

**首次创建规则：**

- 项目级首次创建 → 在该目录写 `.gitignore`（内容 `*`），避免任务数据被提交进业务仓库
- 全局 `~/.dotasks/` 本身不在任何仓库内 → **不需要** `.gitignore`
- 目录默认结构：

```
.dotasks/
├── tasks.jsonl
├── progress.jsonl
├── checkpoints/
│   └── 20260608T160000-cp01.json
└── archive.jsonl          # 清理时归档，不物理删除
```

**时间戳获取：** 必须用系统命令取真实时间，禁止手写/推算。

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"          # 写 created/ts 用这个
date +"%Y%m%d"                          # 文件名日期用这个
```

---

## 2. 数据结构

### 2.1 tasks.jsonl

每行一个 JSON 对象，追加写入：

```jsonl
{"schema":1,"id":"20260608T160000-a3f1","content":"调研 Rust 异步模型","status":"pending","tag":"dev","priority":"P1","parent":null,"depends_on":[],"created":"2026-06-08T08:00:00Z"}
{"schema":1,"id":"20260608T160500-b7c2","content":"修 bug #1423","status":"done","tag":"fix","priority":"P0","parent":null,"depends_on":["20260608T160000-a3f1"],"created":"2026-06-08T08:05:00Z","resolved":"2026-06-08T09:00:00Z"}
```

| 字段 | 必要 | 说明 |
|------|------|------|
| `schema` | 是 | 结构版本号，当前 `1`。用于后续兼容迁移 |
| `id` | 是 | 见下方 ID 规则 |
| `content` | 是 | 任务内容（**写入前必须脱敏**，见第 6 节） |
| `status` | 是 | `pending` / `in_progress` / `blocked` / `done` / `idea` / `cancelled` |
| `tag` | 否 | 分类标签 |
| `priority` | 否 | `P0` / `P1` / `P2`，默认 `P2` |
| `parent` | 否 | 父任务 id，用于拆解子任务 |
| `depends_on` | 否 | 依赖的任务 id 数组 |
| `created` | 是 | ISO-8601 UTC 时间戳 |
| `resolved` | 否 | 进入 `done` / `cancelled` 时写入 |

**ID 规则（替换原来的 max+1，避免并发撞号）：**

```
id = <UTC时间戳紧凑格式>-<4位随机十六进制>
例：20260608T160000-a3f1
```

生成后必须 `read` 文件确认该 id 未出现；若重复则重新随机一次。

**状态流转规则：**

```
idea ──升级──> pending
pending ──开工──> in_progress
pending/in_progress ──受阻──> blocked
blocked ──解除──> in_progress
in_progress ──完成──> done
任意状态 ──放弃──> cancelled

done / cancelled 为终态。若需回退，必须显式说明原因，
并同时清空 resolved 字段。
```

- 写入 `done` 或 `cancelled` 时**必须**同步写 `resolved`
- `status != done/cancelled` 时**不得**存在 `resolved` 字段

### 2.2 progress.jsonl（新增 · 核心）

每行一个事件，**只追加，永不修改**。这是断点续接的主依据。

```jsonl
{"schema":1,"ts":"2026-06-11T03:30:00Z","task":"20260608T160000-a3f1","event":"decision","summary":"选定 tokio 而非 async-std，因为团队已有 tokio 经验","refs":["docs/adr-001.md"],"redacted":false}
{"schema":1,"ts":"2026-06-11T05:10:00Z","task":"20260608T160000-a3f1","event":"blocked","summary":"缺 staging 环境的 Redis 权限，等运维开通","redacted":false}
{"schema":1,"ts":"2026-06-11T07:45:00Z","task":"20260608T160000-a3f1","event":"progress","summary":"完成连接池封装，单测 12/12 通过","redacted":false}
```

| 字段 | 必要 | 说明 |
|------|------|------|
| `ts` | 是 | ISO-8601 UTC |
| `task` | 否 | 关联任务 id（整体进展可省略） |
| `event` | 是 | `progress` / `decision` / `blocked` / `unblocked` / `note` |
| `summary` | 是 | 一句话说明。**决策类必须写"为什么"，不只写"做了什么"** |
| `refs` | 否 | 相关文件/链接/commit |
| `redacted` | 是 | 是否触发过脱敏 |

### 2.3 checkpoints/<ts>-<seq>.json（新增 · 核心）

阶段性完整现场快照，一个文件 = 一刻现场。**只在显式 checkpoint 时写**（不依赖 agentTurn 钩子）。

```json
{
  "schema": 1,
  "cp_id": "20260611T080000-cp01",
  "ts": "2026-06-11T08:00:00Z",
  "active_tasks": ["20260608T160000-a3f1"],
  "done_since_last": ["20260608T160500-b7c2"],
  "progress": "异步模型调研完成，连接池封装已完成并过单测",
  "decisions": ["选定 tokio"],
  "blockers": ["staging Redis 权限待开通"],
  "artifacts": ["src/pool.rs", "docs/adr-001.md"],
  "next_step": "接入 pipeline 并补集成测试",
  "open_questions": ["QPS 上限是否需要压测？"]
}
```

字段说明：`next_step` 是**给下一次 session 的行动指令**，不可为空。

---

## 3. 核心流程

### 3.1 ▶ resume — 恢复现场（最重要）

```
触发："继续上次的任务"/"上次做到哪了"/"接着干"/新 session 开工时主动执行
```

1. 按第 1 节优先级定位 `.dotasks/`
2. 不存在 → 说明这是首次，直接问用户要做什么
3. 存在 → 依次读取：
   - `tasks.jsonl` → 筛出 `pending` / `in_progress` / `blocked`
   - `progress.jsonl` → 取最后 20 条（跨多次 session 的近期上下文）
   - `checkpoints/` → 取**最新的一个**文件
4. 输出一段**现场摘要**（固定模板，见 3.7），格式如下：

```markdown
## 上次做到哪了（截至 2026-06-11 08:00 UTC）

**进行中**
- 🟡 20260608T160000-a3f1 调研 Rust 异步模型（P1，dev）
  下一步：接入 pipeline 并补集成测试

**受阻**
- 🔴 无

**待办（未开工）**
- 🔲 ...

**关键决策**：选定 tokio（原因：团队已有经验）
**产出**：src/pool.rs、docs/adr-001.md
**遗留问题**：QPS 上限是否需要压测？

---
要接着做哪一项？（默认：继续 20260608T160000-a3f1 的下一步）
```

5. **不要**在用户确认前自行开工改代码

### 3.2 添加任务

```
触发："记件事"/"添加任务"/"记录一下"/"这是个想法"
```

1. 读 `tasks.jsonl`
2. 解析意图：
   - 默认 `status = pending`
   - "想法/maybe/idea" → `idea`
   - 提到前面有父任务 → 写 `parent`
3. **先脱敏 content**（第 6 节），再构造 JSON 行
4. 生成 id（可能撞号则重试），确认唯一
5. 追加写入 `tasks.jsonl`
6. 回复确认（回显 id + 内容）

### 3.3 列出任务

```
触发："列一下任务"/"我的待办"/"看看任务"
```

1. 读 `tasks.jsonl`
2. 过滤：
   - 无参数 → `status not in (done, cancelled)`
   - "只看 dev 的" → 加 `tag == "dev"`
   - "看看 blocked 的" → 按 status 过滤
   - "看全部" → 含 done/cancelled
3. 按 `priority` 升序（P0 优先）、再按 `created` 排序
4. 用 3.7 的固定模板输出

状态符号（**统一使用，不得自创**）：

| status | 符号 |
|--------|------|
| pending | 🔲 |
| in_progress | 🟡 |
| blocked | 🔴 |
| done | ✅ |
| cancelled | ⚫ |
| idea | 💡 |

### 3.4 查看单个任务

```
触发："任务 X 是什么"/"看看 task 20260608T160000-a3f1"
```

1. 读 `tasks.jsonl` 定位该行
2. 再读 `progress.jsonl` 中 `task == 该 id` 的全部事件
3. 输出：任务全字段 + 该任务的进展时间线（事件倒序）

### 3.5 更新状态 / 记录进展

```
触发："X 开工了" → in_progress
     "X 卡住了，因为 Y" → blocked（同时写一条 blocked 进展）
     "X 做完了" → done（写 resolved + 一条 progress 事件）
     "这不是任务了" → cancelled
```

1. 读 `tasks.jsonl`，按 id 精确匹配；只有 id 匹配不到时才用内容模糊匹配，且**必须回显匹配到的那一行让用户确认**
2. **改单行用 `edit`，`old_string` 必须包含该行完整内容**（含 id），确保唯一；若同一行内容出现多次，改为"读全文 → 改目标行 → 覆写"并先备份到 `.dotasks/.backup/`
3. 状态变更**必须**同步追加一条 `progress.jsonl` 事件，说明变更原因
4. `done` / `cancelled` 必须写 `resolved`；回退终态必须删除 `resolved` 并说明原因

### 3.6 写入 checkpoint

```
触发："存个档"/"记录一下现场"/"checkpoint"/一个阶段完成时主动询问
```

1. 读 `tasks.jsonl` + `progress.jsonl` 尾部
2. 按 2.3 的字段组装，`next_step` 必填
3. **新增**一个 `checkpoints/<ts>-<seq>.json` 文件（绝不覆盖旧快照）
4. 同时在 `progress.jsonl` 追加一条 `note` 事件指向该 cp
5. 回复：本次快照摘要 + 下一次 resume 将读到什么

### 3.7 清理 / 归档

```
触发："清一下已完成"/"归档已完成"
```

1. 读 `tasks.jsonl`，筛出 `done` / `cancelled`
2. **展示**将被动到的全部任务（id + 内容 + 完成时间），请求确认
3. 确认后：
   - 把待移除行**追加**进 `archive.jsonl`（带归档时间戳），**不物理删除**
   - 从 `tasks.jsonl` 移除这些行（此为唯一全量覆写场景，覆写前备份到 `.dotasks/.backup/`）
4. 回复：归档条数 + archive 文件位置
5. 恢复入口：`"看归档"` → 读 `archive.jsonl`

### 3.8 查询进展 / 历史

```
触发："回顾一下今天做了什么"/"看看上次的记录"/"这个任务为什么这么定"
```

- 今日 → 读 `progress.jsonl`，筛 `ts` 落在今天（本地时区）
- 指定日期段 → 按 `ts` 过滤
- 按 `ts` 排序，输出 markdown 时间线

---

## 4. 输出模板（统一，不得自创格式）

列任务固定格式：

```markdown
🔲 20260608T160000-a3f1  [P1/dev]  调研 Rust 异步模型
🟡 20260608T170000-c9d0  [P1/dev]  接入 pipeline ← 父: a3f1
🔴 20260608T180000-e1b2  [P2/ops]  staging 权限待开通（等运维）
```

规则：`符号 id [优先级/标签] 内容`，括号内只写非默认值；子任务后缀 `← 父: <短id>`。

---

## 5. 与平台能力的边界（避免双写不一致）

| 场景 | 用哪个 | 说明 |
|------|--------|------|
| 单次 session 内的临时步骤拆分 | 平台内置任务工具 | 会话结束即失效，**不要**写入 dotasks |
| 需要跨 session 存活的长期任务 | dotasks `tasks.jsonl` | 只写一次，不要两边都写 |
| 长期事实、用户偏好、项目约定 | 平台 MEMORY | 与任务状态无关的稳定信息 |
| 任务进展、决策、阶段现场 | dotasks `progress.jsonl` + `checkpoints/` | 会随任务推进变化的动态信息 |
| 每日工作总结 | 平台记忆日志 | dotasks 的 progress 是任务粒度，不是日记 |

**判定口径：会随任务推进而失效/变化的信息 → dotasks；稳定不变的信息 → MEMORY。**

---

## 6. 隐私脱敏（强制，覆盖所有文件）

**必须在写入前执行**，且**不只针对日志——`tasks.jsonl` 的 `content` 是最高风险位置**，密钥、客户名、内部代号最常出现在任务描述里。

### 6.1 覆盖范围

`tasks.jsonl` · `progress.jsonl` · `checkpoints/*.json` · `archive.jsonl` —— 全部。

### 6.2 规则（按序应用）

1. **键值类**：匹配 `(password|passwd|api[_-]?key|secret|private[_-]?key|auth[_-]?token|access[_-]?token|bearer|credential)` 后紧跟的 `=`/`:`/空白 后的值 → 值替换为 `***`
   - 例：`api_key=sk-abc123...` → `api_key=***`
   - `Bearer eyJhbGci...` → `Bearer ***`
2. **中国大陆高频 PII**（补齐，原方案缺失）：
   - 手机号 `1[3-9]\d{9}` → `138****5678`（留前 3 后 4）
   - 身份证 `\d{17}[\dXx]` → `110101********1234`（留前 6 后 4）
   - 银行卡 `\d{16,19}` → 留前 4 后 4
   - 邮箱 → 用户名部分首尾各留 1 位，中间 `***`
   - 统一社会信用代码 `[0-9A-HJ-NPQRTUWXY]{18}` → 留前 4 后 4
3. **通用兜底**：连续 ≥32 位无空格的字母数字混合串（疑似 token/hash）→ 留前 6 后 4，中间 `***`
4. **显式声明**：若同一条内容中同时命中 ≥2 类敏感项，标记 `redacted: true`

### 6.3 禁止的旧逻辑

- ❌ **不要**用"脱敏后缩减超过 50% 就整条替换为 `[PRIVACY_FILTERED]`"——字符缩减比例与隐私强度无关，含大量编号的正常内容会被误杀。
- ✅ 正确做法：**只替换命中的敏感片段**，其余内容完整保留；实在无法判断时，向用户确认而不是整条丢弃。

### 6.4 兜底原则

脱敏是**不确定过程**。当 Agent 无法确定某段是否敏感时，宁可先问用户，也不要写入。

---

## 7. 场景触发速查

| 用户说 | 操作 |
|--------|------|
| "记件事：X" | 添加任务（脱敏后 pending） |
| "这是个想法：X" | 添加任务（status = idea） |
| "列一下任务 / 我的待办" | 列出未完成任务 |
| "X 开工了" | 状态 → in_progress + progress 事件 |
| "X 卡住了因为 Y" | 状态 → blocked + blocked 事件 |
| "X 做完了 / 搞定了" | 状态 → done + resolved + progress 事件 |
| "存个档 / checkpoint" | 写入 checkpoint |
| "继续上次的任务 / 接着干" | **resume 恢复现场** |
| "回顾一下今天" | 查询 progress 时间线 |
| "清一下已完成" | 归档（先展示确认，不物理删除） |
| "看归档" | 读 archive.jsonl |

**不应触发的情况（负例）：**

- 日常闲聊里出现"任务""待办"等词，但**没有**明确指向本系统的操作意图 → 不读写任何文件
- 用户只是询问某个概念 → 纯回答
- 单次会话内的临时步骤规划 → 用平台内置任务工具，不写 dotasks

---

## 8. 规则

1. **`progress.jsonl` 纯追加，永不修改**——这是断点续接的基石。
2. **`tasks.jsonl` 以追加为主**；单行改动用 `edit`（`old_string` 含完整行+id 保证唯一），全量覆写仅限归档且必须先备份。
3. **任何写入前先脱敏**，覆盖全部文件，尤其 `content`。
4. **时间戳必须用系统命令获取**，禁止手写或推算。
5. **id 用时间戳+随机后缀**，不用 max+1；生成后校验唯一。
6. **`resolved` 与 `status` 必须一致**：终态必有、非终态必无。
7. **归档不删除**，写入 `archive.jsonl`。
8. **删除/归档前必须展示待操作清单并获得确认。**
9. **每次 resume / checkpoint 必须给出 `next_step`**——没有下一步的动作不算完成一次交接。
10. **无明确意图时不读写任务文件。**
11. **不重复记账**：会话内临时步骤用平台任务工具，跨会话长期任务才用 dotasks。
12. **无法判断是否敏感时，先问用户。**

---

## 9. 最小可用示例（首次使用）

```bash
# 1. 建目录（项目级示例）
mkdir -p .dotasks/checkpoints .dotasks/.backup
printf '*\n' > .dotasks/.gitignore

# 2. 取当前时间
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

然后告诉 Agent：`记件事：把 XX 模块的迁移方案定下来` → 它会写第一行任务；
干一段后说 `存个档` → 生成第一个 checkpoint；
下次新会话说 `继续上次的任务` → 自动恢复现场。
