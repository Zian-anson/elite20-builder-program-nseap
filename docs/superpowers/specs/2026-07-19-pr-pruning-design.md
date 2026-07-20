# Spec — 线3：设计总部 issue 收口 (PR Pruning & Local Cherry-Pick)

- **日期**：2026-07-19（二次修订 2026-07-20）
- **作者**：牛保康（子安）
- **范围**：收口 `a976xw7td/elite20-builder-program-nseap` 仓库 5 个 open PR（#1 #4 #5 #6 #7），让本地基线吸收有价值内容、为后续线1（平台深化）扫除基线漂移风险
- **状态**：设计已与用户对齐（含二次修订后 git init 决定），待实施
- **版本**：v2（v1 二次审查发现 14 处遗漏/错位后修订）

## 版本追踪决策（关键）

**本地工作区在本 spec 启动时为非 git 整合目录**。按用户拍板，线3 实施启动前在项目根执行：

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
git init
git add -A && git commit -m "baseline: 7-19 upstream sync state (platform@8e95299, challenges@67e27a7, design@2d9810f)"
```

之后每完成一个 PR 的 cherry-pick，单独 `git commit` 记录该 PR 的落地动作与 head sha。

## 依赖与安全边界

- 用户当前 GitHub 账号 `Zian-anson` 对 `a976xw7td/elite20-builder-program-nseap` 仅有 `pull` 权限（无 `push`/`triage`/`maintain`/`admin`），**不是该 org 的 collaborator**
- 用户在本会话中明文贴出过 2 个 GitHub PAT，**均视为已永久泄露**；文档不保留其前缀或片段
- GitHub 闭环动作（close PR / 评论）推迟到 §8 单独处理，依赖用户吊销旧 token、新建 fine-grained PAT、并被 `a976xw7td` 加为 collaborator 三件前置完成
- 本 spec 文档内**不出现任何 PAT/token 字符串**；GitHub 读操作仅通过本机 `gh` 凭据执行，凭据无效时停止外部动作

---

## 1. 背景与目标

### 1.1 上游 5 个 PR 处于未决状态，影响后续工作基线稳定性

| PR | 标题 | head → base | 体量 | mergeable | 核心问题 |
|---|---|---|---|---|---|
| #7 | Challenge Scoring System v1.0 | main → main | 1 文件 +0 行 | clean | 仅 1 个 zip 二进制 `challenge-scoring-system 3.zip` 入 git，body 描述的 18 挑战源码不在 PR 内。cherry-pick 前需下载 zip 解压查看内部结构以加固 close 证据 |
| #4 | Team 3 Agent 本体工程化 + MVP Demo | feat/team3-agent-ontology-mvp → main | 37 文件 +16280 -143 | clean | **完整版**：8 份抽取报告 + 3 ADR + 22 W3C 文件 + demo。README -143 行（PR 描述称不修改主仓已有文件，与实际有出入） |
| #6 | Team 3 Ontology engineering v1 | team3/ontology-engineering-v1 → main | 16 文件 +6471 -0 | clean | **精简版**：4 小时后同作者重复 PR，内容是 #4 子集，且目录归属分裂（本体在 `agents/team3-ontology-core/`、demo 在 `teams/agent-team/demo/`） |
| #5 | Platform Team learning-platform | platform-team-submit → main | 32 文件 +5812 -0 | clean | 前端 mock + `lib/data.ts` 266 行假数据；与平台仓 `a976xw7td/ai-x-challenge-learning-mvp`（7-16 同步至 `8e95299`，含完整 RBAC/Redis/DeepSeek/飞书）相比属旧版 |
| #1 | Team 6 Knowledge Cognitive Cell MVP | main → main | 177 文件 +38463 -1 | **dirty** | 重复两套知识库（`knowledge-base/knowledge-cognitive-cell/` 与 `nseap-knowledge-base/`）；根目录散落 7 个 IEEE/P3394 docx；中文目录 `例子/`；head=main 反模式。**dirty 的冲突文件清单需在 cherry-pick 前先拉出，避免在取文件时踩到上游已前进版本** |

### 1.2 收口目标

- **可审**：每个 PR 拿到一份定量化评审段（合规性 / 内容实质 / 与本地基线对照 / 处置推荐 / cherry-pick 清单）
- **可落地**：#1 + #4 中有价值内容 cherry-pick 到本地，与本地基线对齐、过验证
- **可放可不放**：#5 #6 #7 推荐给上游 close，但本地不强行 push close 动作（依赖权限，见 §8）
- **不破坏本地基线**：cherry-pick 后本地 `platform/ai-x-challenge-learning-mvp` 的 `npm run lint` + `npm run build` 仍须通过
- **版本可追踪**：cherry-pick 文件逐 PR 单独 commit；每文件记录 head sha 供下轮同步对齐

---

## 2. 范围与产出物

| 产出物 | 路径 | 体量（估） | 是否阻塞后续 |
|---|---|---|---|
| **Git 基线 commit** | 项目根 `.git/` | 7-19 同步基线状态完整快照 | 是 |
| 评审报告 | `源文件/PR评审报告/2026-07-19.md` | 5 章节（模板见 §10） | 是 |
| cherry-pick #1 增量 + commit | `knowledge-base/nseap/`（与本地 `knowledge-base/` 并列） | ~30 文件，commit message 含 head sha | 是（线4 Companion Agent 需要） |
| cherry-pick #4 增量 + commit | `teams/agent-team/`（PR 已按此目录归属） | ~32 文件，commit message 含 head sha | 是（线1 平台深化使用 Ontology Schema） |
| 同步记录补充 | `源文件/上游同步记录/2026-07-19.md`（已建） | +100 行末尾状态段（含 head sha 映射表） | 否 |
| cherry-pick head sha 映射文件 | `.cherry-pick-map.json`（项目根） | 每文件一行 head sha 映射，JSON 格式 | 是（下轮同步用） |
| GitHub push/close 评论 gist | `docs/superpowers/specs/2026-07-19-pr-close-messages.md` | 5 段推荐文案 | **推迟**到 §8 |

**不在范围**：
- 修改上游 GitHub 仓库内容（close/merge/评论）的执行 — 推迟到 §8
- 重写 #4 或 #6 内容（PR -143 行改动不直接落地，仅审查后单独评价）
- 修复 #1 dirty 状态（不在上游仓库操作；靠本地单文件内容拷贝绕开 dirty）
- 与作者本人的对话沟通（不在 spec 范围内）
- 新 `package.json` / 新 Node 依赖引入（cherry-pick 文件即使含 `package.json` 也不触发 `npm install`；本 spec 全部验证用现有 node + python3 环境）

---

## 3. 目录归属决策

### 3.1 #1 知识库 → `knowledge-base/nseap/`

PR 内部有重复两套知识库（同 author 同主题）：

| PR 内路径 | 性质 | 是否 cherry-pick |
|---|---|---|
| `nseap-knowledge-base/` | 演进版（含 handoff-v1.0-prompt.md、SQLite 迁移、v0.3 worklog） | ✅ 是 |
| `knowledge-base/knowledge-cognitive-cell/` | 早期版本（无 handoff 文档） | ❌ 否（重复） |
| 根目录 7 个 `*.docx`（IEEE 草稿、P2807.8 等） | 学术资料散落入库 | ❌ 否 |
| `例子/C2S_Final_BigData_Submission/` | 中文目录散落根 | ❌ 否 |
| `本体抽取-Skill.md`、`知识库进展与下一步-对齐设计方案.md`、`CHANGES-原文优先架构.md` | 根目录散落 | ❌ 否 |
| `docs/elite20-builder-submission-ai-protocol.md`、`docs/team-mvp-submission-standard.md` | 设计文档 | ❌ 否（避免与本地 `docs/` 已有技术白皮书/流程文档评审责任重叠；如后续评审需这些文件，单独走评审流程） |
| `teams/knowledge-team/README.md` (+19) | Team 6 门面更新 | ❌ 否（本地同步基线 7-10 后无 `teams/knowledge-team/README.md`；cherry-pick 它就是**新增**而非更新。按「只收有评审证据的资产」原则跳过，由 §8 与作者沟通时单独询问是否有意新增） |

**植入位置**：本地 `knowledge-base/nseap/`，与本地已有 `knowledge-base/faq/`、`knowledge-base/prompts/` 等并列子目录隔离。

### 3.2 #4 Ontology 工程 → `teams/agent-team/`

PR 已把所有新增文件放在 `teams/agent-team/`（注意 `scripts/` 实际嵌套在 `ontology/scripts/` 下，不是 `teams/agent-team/scripts/`）：

```
teams/agent-team/
  README.md (+268 -143)         # 含 -143 行，需审查后单独决定是否落地
  docs/                          # 架构文档（3 份）
    Team3-架构总览.md
    Team3-小白讲解.md
    主仓-vs-Team3-架构对比.md
  mvp-demo/demo.mjs              # 可运行 demo
  ontology/
    README.md (+299)
    core/
      agent-ontology.ttl         # 1137 行 OWL 2 DL
      agent-ontology-shapes.ttl  # 671 行 SHACL
      agent-ontology-rules.swrll # 1050 行 SWRL
    docs/
      adr/0001-use-owl-turtle.md
      adr/0002-use-neo4j-and-fuseki.md
      adr/0003-red-line-formalization.md
      guides/quickstart.md
      json-schema-report.md
    graph/fuseki/red-line-queries.sparql
    schemas/
      agents/agent-manifest.schema.json
      messages/{message-envelope,message-payloads}.schema.json
      records/{challenge,submission}-record.schema.json
      typescript-zod/zod-from-schemas.ts
    scripts/                     # ⚠️ 注意嵌套层级
      docker-compose.yml
      load-fuseki.sh
      load-neo4j.sh
      ontology-validate.yml
      python/
        run_red_line_queries.py
        validate_json_schemas.py
        validate_owl.py
      validate.sh
  reports/                       # 8 份抽取报告（每份 900-1600 行）
    Team3-{事件,关系,实体概念,技能,流程,规则,语义模块}抽取.md
    本体构建方法论.md
```

**与本地已有内容共存**：本地 `ontology/agent-ontology.md`（380 行 Markdown 版）保留不动；#4 工程化版本新增到 `teams/agent-team/ontology/core/*.ttl` 等位置，不与 #6 那套（`agents/team3-ontology-core/`）重叠。

**README -143 行处理**：
- cherry-pick 前先核本地是否已有 `teams/agent-team/README.md`（本地同步基线 7-10 后未含此目录）
- cherry-pick 后取 PR base commit 与 head commit 两个版本的 README 跑 diff 比对**丢了什么内容**
- PR base = `elite20-builder-program-nseap` 在 PR 创建时的 HEAD，head = PR 自身 head sha `0143c5d`
- 若 -143 行删除的是 Team 3 成员 / 职责分工 / 验收清单等关键信息，单独在评审报告标出但本地不自动回滚；回滚决定推到 §8 闭环时由用户与作者沟通后再决定

---

## 4. 执行编排（方案 A：逐 PR 闭环）

```
前置 ─── git init + baseline commit ──────────────────
 └── git init && git add -A && git commit -m "baseline: 7-19 upstream sync state"

轻批（仅评审，不出脚本验证 — 都推荐 close）
  ├── PR #5  → 评审段 → 处置：close 候选（outdated）
  ├── PR #6  → 评审段 → 处置：close 候选（重复 #4）
  └── PR #7  → 评审段 → 处置：close 候选（zip 入 git；需先解压查内部）
  
重批（评审 + cherry-pick + 脚本验证 + 单独 commit）
  ├── PR #4  → 评审段 → cherry-pick teams/agent-team/ → 4 项验证 → commit
  └── PR #1  → 评审段 → cherry-pick knowledge-base/nseap/ → 2 项验证 → commit

收尾
  ├── 同步记录 2026-07-19.md 末尾补充 cherry-pick 后状态段（含 head sha 映射）
  ├── 最终评审报告产出
  └── GitHub push/close 评论（§8，推迟）
```

PR 闭环内部每个 PR 4 段：① 评审（写报告段）→ ② cherry-pick 决策（如要）→ ③ 验证脚本 → ④ 单独 commit + 入报告。

---

## 5. 验证策略

### 5.0 运行时前置检查（实施前必跑）

```bash
python3 --version
node -v  # 期望 ≥ v18
```

PR #4 自带的两个验证脚本仅使用 Python 标准库，无需安装 `rdflib` 或 `jsonschema`。脚本验证的是结构约束，不等同于第三方语义解析。

### 5.1 #7 / #5 / #6（轻批，不 cherry-pick）

**已在本 spec 的 brainstorming 阶段完成**：
- 文件清单对照（§1.1 表已列）
- body 与实际文件一致性审查（#7 解压后内容与数量基本吻合，但交付为单个 zip）
- 与本地基线对照（#5 #6 已确认被平台仓 `8e95299` / #4 覆盖）

spec 内不再重复脚本验证；仅需把已完成的结论写入评审报告，并把 #7 的 zip **下载解压、查看内部结构**作为 close 证据补强（避免"body 说 18 挑战但 zip 里可能真有 18 挑战源码"这一漏洞）。

### 5.2 #4（重批）

| 步骤 | 命令 | 通过条件 |
|---|---|---|
| ① demo | `cd teams/agent-team/mvp-demo && node demo.mjs` | 4 角色流程打印完成、`mvp-demo/output/` 下 bitable/chat/github 三个 JSON 目录有输出 |
| ② OWL | `python3 teams/agent-team/ontology/scripts/python/validate_owl.py --owl teams/agent-team/ontology/core/agent-ontology.ttl` | 自带脚本的前缀、括号、类、属性、红线与枚举检查通过 |
| ③ JSON Schema | `python3 teams/agent-team/ontology/scripts/python/validate_json_schemas.py --schemas teams/agent-team/ontology/schemas` | 5 个 schema 的 JSON 与结构检查全部通过 |
| ④ 字段对齐 | 用 `node` 比对 #4 `submission-record.schema.json` 与平台 `SubmissionRecordSchema` 的字段集合 | 至少 `student_id / challenge_id / system_validation_status / submitted_at` 4 字段存在；差异写入报告（探索项，不要求人工强一致） |

### 5.3 #1（重批）

| 步骤 | 命令 | 通过条件 |
|---|---|---|
| ① schema parse | `find knowledge-base/nseap/schemas -name "*.json" -exec node -e "JSON.parse(require('fs').readFileSync(process.argv[1]))" {} \;` | 全部 schema 可被 JSON.parse（用 `find -exec` 避免 glob 无匹配把字面 `*.json` 传给 node） |
| ② build-knowledge-data 实际构建 | `node knowledge-base/nseap/scripts/build-knowledge-data.js`（如需环境变量则先查需何种） | 无崩溃、产出 `knowledge-base/nseap/app/knowledge-data.json` 等文件作为构建证据保留（**非** dry-run，脚本实际写文件） |

### 5.4 基线回归（cherry-pick 后必跑）

```bash
cd platform/ai-x-challenge-learning-mvp
npm run lint        # 期望仅 2 个上游既有警告
npm run build       # 当前基线：14 个 page.tsx、19 个 API route.ts，构建全部成功
```

cherry-pick 不应触碰平台仓；构建产物清单应与 7-19 同步基线一致。

### 5.5 Token/secret 扫描（cherry-pick 后必跑）

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
grep -rE "ghp_|github_pat_|sk-[A-Za-z0-9_-]{20,}|FEISHU_APP_SECRET=[a-zA-Z0-9]" \
  knowledge-base/nseap/ teams/agent-team/ 2>/dev/null
```

任何命中均需人工复核；宽泛 `sk-` 规则可能命中 Skill 标题锚点。实际凭证命中时，该文件**不 commit**，并在评审报告标注。

---

## 6. 风险与边界

| 风险 | 处置 |
|---|---|
| #1 文件污染本地 `knowledge-base/` | 仅取 `nseap-knowledge-base/` 演进版，植入 `knowledge-base/nseap/`，与 PR 内重复目录不收 |
| #4 README -143 行删了 Team 3 名单等关键信息 | cherry-pick 后跑 diff 比对 base/head 两版本；丢失部分在评审报告标出，不自动回滚 |
| #1 5 个 docx 体积大 | 不 cherry-pick docx；只取 markdown/json/schema/script |
| PR #1 dirty 状态使 cherry-pick 不可整 PR | 不做整 PR cherry-pick，做**单文件内容拷贝**：`gh api repos/.../contents/<path>?ref=<head_sha>` 取回内容直写本地；**cherry-pick 前先 `gh api .../pulls/1` 拉冲突文件清单** |
| 单文件 `gh api contents` 调用 60+ 次触发 GitHub rate limit | 用 keyring 里在用的 PAT 做认证（rate limit 5000/hr）；若接近耗尽则 sleep；不要并发 |
| cherry-pick 文件含疑似 token/secret 字段 | §5.5 扫描；命中则不 commit 该文件，单独在评审报告标注 |
| 验证脚本参数或解释错误 | 使用 §5.2 的完整参数；报告只声明脚本实际覆盖的结构检查 |
| `build-knowledge-data.js` 需要 Node 环境变量 | 实施前先读脚本头部确认变量清单；若缺变量导致失败，查 PR body / README 或打"环境未就绪"标记 |
| 平台 build 失败 | 失败原因独立排查，回归 7-19 同步基线后再 cherry-pick |
| `git init` 后 `.DS_Store` / `node_modules` 污染 commit | 首次 commit 前补 `.gitignore`：`.DS_Store`、`node_modules/`、`.next/`、`outputs/*.inspect.ndjson` |

---

## 7. 验收边界（线3 本地部分）

线3 **本地部分**验收通过必须同时满足：

1. ✅ 5 个 PR 评审报告完成（模板见 §10；**轻批 PR（#7 #5 #6）的 cherry-pick 清单段标 `N/A — 推荐 close`**）
2. ✅ #1 + #4 cherry-pick 文件落地、§5 验证脚本除"缺库非阻塞"外全部通过
3. ✅ 同步记录 `源文件/上游同步记录/2026-07-19.md` 补完 cherry-pick 后状态段（含文件 → head sha 映射表）
4. ✅ 平台基线无破坏：`npm run lint` + `npm run build` 同 7-19 同步后状态
5. ✅ §5.5 token/secret 扫描零命中（或命中文件已标注未 commit）
6. ✅ 项目根 `git log` 至少含以下 commit：
   - `baseline: 7-19 upstream sync state`
   - `cherry-pick PR #4 (head 0143c5d): teams/agent-team/`
   - `cherry-pick PR #1 (head edfc307): knowledge-base/nseap/`
7. ✅ 每个 cherry-pick 文件对应 commit message 或单独 `.cherry-pick-map.json` 记录 head sha，供下轮同步对齐

**GitHub push/close 不计入**线3 验收。推迟到 §8 单独处理、单独验收。

---

## 8. GitHub 闭环（推迟）

### 8.1 依赖前置条件（用户负责，3 件事）

1. **吊销泄露 token**：在 github.com/settings/tokens 吊销本会话中曾暴露的两枚旧 token
2. **新建 fine-grained PAT**：仅授权 `a976xw7td/elite20-builder-program-nseap`，Permissions 设 `Pull requests: write`、`Issues: write`、`Contents: read`，过期 7 天
3. **被加为 collaborator**：联系 org owner `a976xw7td`，把 `Zian-anson` 加为该仓库 collaborator（权限至少为 `triage`）

3 件事完成前，**任何 close / 评论 / merge 动作都不可执行**。

### 8.2 闭环动作清单（本地完工后单独执行）

| PR | 动作 | 评论 gist 文件 |
|---|---|---|
| #7 | close | `docs/superpowers/specs/2026-07-19-pr-close-messages.md` §PR-7 段 |
| #5 | close | 同上 §PR-5 段 |
| #6 | close | 同上 §PR-6 段 |
| #4 | 评审 + 待二次修订 close message | 同上 §PR-4 段（需补 README -143 行 review 结果） |
| #1 | 评审 + 推荐拆分重提 | 同上 §PR-1 段 |

**评论 gist 文件 `docs/superpowers/specs/2026-07-19-pr-close-messages.md`**：本 spec 落地后作为附件产出，含 5 段本地化中文文案，由用户执行 `gh pr close`/`gh pr comment` 时复制粘贴。文件不落任何 token/secret 字符串。

---

## 9. 后续动作衔接

- 线1（平台深化）：消费 `teams/agent-team/ontology/schemas/typescript-zod/zod-from-schemas.ts` 中的 Zod schema 作为平台运行时校验来源
- 线2（挑战本地化）：消费 `knowledge-base/nseap/schemas/{knowledge-item,prompt}.schema.json` 作为知识库结构定义
- 线4（Companion Agent 8-19）：消费 `knowledge-base/nseap/knowledge-base/07-agents/knowledge-librarian-agent.md` 作为 Knowledge Librarian Agent 能力参考
- GitHub 闭环完成后，本地 cherry-pick 内容会被未来上游同步覆盖判断；下轮同步按 §11 策略对齐

---

## 10. 评审报告模板（每 PR 一节必填）

```markdown
## PR #N — <PR 标题>

- **作者**：<author>
- **head**：`<head_ref>` @ `<head_sha>`
- **base**：`<base_ref>`
- **创建/更新**：<created_at> / <updated_at>
- **mergeable**：<state>
- **体量**：<changed_files> 文件，+<additions> -<deletions>

### 合规性
- [ ] body 是否完整填写
- [ ] 是否含二进制/疑似 secret
- [ ] 是否用 feature 分支（避免 main→main）
- [ ] PR 描述与实际文件一致性

### 内容实质
<3-5 句总结该 PR 实际交付了什么，列出关键文件>

### 与本地基线对照
- **重复**：<与本地哪部分重叠>
- **新增价值**：<本地缺什么，PR 提供什么>
- **冲突**：<与本地既有内容冲突点>

### 处置推荐
<close 候选 / cherry-pick / 拆分重提> — <理由 1-2 句>

### cherry-pick 清单
<N/A — 推荐 close> 或：
| PR 内路径 | 本地植入位置 | head sha | 验证结果 |
|---|---|---|---|
| ... | ... | ... | ... |

### 证据
- gh api 命令记录（关键查询）
- 文件清单 hash 或 head sha
- 验证脚本输出（如适用）
```

---

## 11. 下轮同步策略（新增）

当未来从上游再次同步时，需对齐本地 cherry-pick 与上游最新 head：

| 场景 | 识别方式 | 处理 |
|---|---|---|
| 上游 main 合并了 PR #4 | `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/4/merged_at` 非空 | 把本地 cherry-pick 的 `teams/agent-team/` 与上游 main 版本 diff，若上游更新则覆盖本地并更新 commit |
| 上游 main 合并了 PR #1 或 #1 被拆分重提后合并 | `gh api .../pulls/1/merged_at` 非空 或 出现新 PR | 同上，覆盖 `knowledge-base/nseap/` 或整合到新位置 |
| PR #4 / #1 作者在 head 分支上更新但未合并 | 对比 `.cherry-pick-map.json` 里的 head sha 与 `gh api .../pulls/<n>` 的当前 head sha | 若 head sha 已前进，记录到同步记录"待评审更新"段，不自动覆盖；人工决策 |
| PR 被 close 未合并 | `gh api .../pulls/<n>` state=closed && merged_at=null | 本地 cherry-pick 内容保留；标记为"本地分叉" |
| #7 zip 文件消失 / PR 被 close | 同上 | 本地不受影响（未 cherry-pick） |

**操作规则**：每轮同步先跑 §11 表格判断，再决定是否覆盖本地 cherry-pick 内容；不得凭"上游新"直接覆盖——本地 cherry-pick 可能与作者后续 head 分叉。
