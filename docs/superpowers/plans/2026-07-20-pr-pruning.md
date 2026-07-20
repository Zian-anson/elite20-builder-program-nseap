# 线3 设计总部 PR 收口 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 评审设计总部 5 个 open PR（#1 #4 #5 #6 #7），cherry-pick #1（knowledge-base/nseap/）与 #4（teams/agent-team/）到本地并完成验证、git 版本化，产出评审报告与 close 评论文案。

**Architecture:** 非整 PR merge，改为单文件内容拷贝（`gh api contents/<path>?ref=<head_sha>`）落地本地；git init 建基线 commit，每 PR cherry-pick 后单独 commit；`.cherry-pick-map.json` 记录文件→head sha 映射供下轮同步对齐。

**Tech Stack:** gh CLI（GitHub API）、git、node ≥18、python3 标准库、bash。

**Spec:** `docs/superpowers/specs/2026-07-19-pr-pruning-design.md`（v2，已批准）

**关键常量（已核）:**
- PR #1 head: `edfc3071aa9459c3e404734e2d5d63d21be5a7bb`
- PR #4 head: `0143c5d8a8bb693c721557bc69f8b09cbbff2fab`，base: `8134a04f90917ebaf6337d5d07aadb6864cf485c`
- PR #7 head: `7807c09913556c8016eb668bbaeab5d2aa5fe506`
- 上游设计总部 main HEAD: `2d9810f79b955c5fc50406ffdcdb8c3ef47967ea`
- 项目根: `/Users/an/工作空间/项目/Elite20-Builder-Program`

---

## Task 1: git init + .gitignore + baseline commit

**Files:**
- Create: `/Users/an/工作空间/项目/Elite20-Builder-Program/.gitignore`

- [ ] **Step 1: 写 .gitignore**

```bash
cat > /Users/an/工作空间/项目/Elite20-Builder-Program/.gitignore <<'EOF'
.DS_Store
node_modules/
.next/
outputs/*.inspect.ndjson
.env.local
.env*.bak
EOF
```

注：spec §6 列前 4 行；追加 `.env.local` / `.env*.bak` 两行与平台仓自身 gitignore 对齐（token 事件后最小安全动作）。

- [ ] **Step 2: git init + 初始 commit**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
git init
git add -A
git commit -m "baseline: 7-19 upstream sync state (platform@8e95299, challenges@67e27a7, design@2d9810f)"
```

Expected: `git log --oneline` 显示 1 个 commit；`git status` 输出 `nothing to commit, working tree clean`。

- [ ] **Step 3: 验证未追踪大目录**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
git ls-files | grep -c node_modules || echo "OK: 0 node_modules files tracked"
git ls-files | wc -l
```

Expected: 第一行输出 `OK: 0 node_modules files tracked`；第二行记录 tracked 文件总数（预期数千，写入评审报告 §证据段）。

---

## Task 2: 运行时依赖前置检查 + PR #1 dirty 冲突调查

**Files:** 无文件改动（纯检查）。

- [ ] **Step 1: 运行时检查**

```bash
python3 --version
node -v
```

Expected: Python 3 可用；node 输出 `v18.x` 或更高。PR #4 自带验证脚本仅使用 Python 标准库。

- [ ] **Step 2: PR #1 dirty 冲突文件调查**

```bash
gh api "repos/a976xw7td/elite20-builder-program-nseap/compare/2d9810f79b955c5fc50406ffdcdb8c3ef47967ea...xnsnhwh-svg:main" \
  --jq '.files[] | select(.status=="modified") | .filename'
```

Expected: 输出 PR #1 中**同时被上游 main 和 PR head 修改**的文件（冲突候选）。预期含 `.gitignore`、`README.md`、`teams/knowledge-team/README.md`。**将这些文件名记录下来**（写进 Task 9 评审段的"冲突"字段）。

- [ ] **Step 3: 检查上游 main 是否已含 cherry-pick 目标路径**

```bash
gh api "repos/a976xw7td/elite20-builder-program-nseap/git/trees/2d9810f79b955c5fc50406ffdcdb8c3ef47967ea?recursive=1" \
  --jq '.tree[].path' | grep -E '^(nseap-knowledge-base|teams/agent-team)' || echo "OK: 上游 main 无此两路径，cherry-pick 不会踩到上游已前进版本"
```

Expected: `OK: 上游 main 无此两路径...`。若有输出 → 停下来把输出文件清单与 head 版本逐个 diff 后再继续。

---

## Task 3: PR #5 评审段（轻批）

**Files:**
- Create: `源文件/PR评审报告/2026-07-19.md`

- [ ] **Step 1: 建目录与报告骨架**

```bash
mkdir -p /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告
cat > /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告/2026-07-19.md <<'EOF'
# 设计总部 open PR 评审报告 — 2026-07-19

> 评审范围：a976xw7td/elite20-builder-program-nseap 5 个 open PR（#1 #4 #5 #6 #7）
> 评审人：牛保康（子安）
> 评审基线：本地 7-19 同步状态（platform@8e95299, challenges@67e27a7, design@2d9810f）
> 处置动作（close/评论）推迟到 GitHub 闭环阶段单独执行，本文仅产出评审结论与推荐。

EOF
```

- [ ] **Step 2: 写入 PR #5 评审段**

向报告追加（用 `cat >>` heredoc）：

```markdown
## PR #5 — feat(platform-team): 提交 learning-platform — Landing Page/LMS/提交流程/作品集/教师控制台/个人中心/部署配置

- **作者**：wujy309-jpg
- **head**：`platform-team-submit` @ `6bb8203`
- **base**：`main`
- **创建/更新**：2026-07-10T07:51:50Z / 2026-07-10T07:51:50Z
- **mergeable**：clean
- **体量**：32 文件，+5812 -0

### 合规性
- [x] 是否含二进制/疑似 secret — 否
- [x] 是否用 feature 分支 — 是（`platform-team-submit`）
- [ ] body 是否完整填写 — **否**。Summary、Related Challenge、Checklist、Notes for Reviewers 全空，PR 模板未填
- [x] PR 描述与实际文件一致性 — 标题所列页面（Landing/LMS/提交/作品集/教师控制台/个人中心）与文件清单一致

### 内容实质
32 个文件全部位于 `teams/platform-team/learning-platform/`：一个纯前端 Next.js 脚手架（dashboard/challenges/docs/github/knowledge/lms/portfolio/profile/submit/submissions/teacher/login/landing 页面 + Header/Sidebar 组件），数据源为 `lib/data.ts`（266 行硬编码 mock 数据），无 API 路由、无后端、无鉴权；附带 Dockerfile/docker-compose/CI 配置。

### 与本地基线对照
- **重复**：页面路由与本地 `platform/ai-x-challenge-learning-mvp/app/(app)/` 高度重叠（dashboard/challenges/submit/submissions/teacher 等同名页面）
- **新增价值**：无实质新增——本地平台仓（上游 `8e95299`）已含全部页面 + REST API + RBAC + Redis Stream + DeepSeek 评分 + 飞书通知 + Agent Envelope + 审计链路，远超本 PR 的 mock 版
- **冲突**：无直接冲突（PR 文件在 `teams/platform-team/` 下，本地无此目录）

### 处置推荐
**close 候选** — 功能已被平台仓 `a976xw7td/ai-x-challenge-learning-mvp`（本地同步至 `8e95299`）完整取代；作为 Platform Team 工作历史有归档价值，无合并价值。

### cherry-pick 清单
N/A — 推荐 close

### 证据
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/5`（mergeable=clean, changed_files=32）
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/5/files --paginate`（32 文件全在 `teams/platform-team/learning-platform/`）
- 本地 `platform/ai-x-challenge-learning-mvp/README.md`（上游 `8e95299` 重写版，含三层校验+内容外置架构）

```

- [ ] **Step 3: 验证写入**

```bash
grep -c "^## PR #5" /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告/2026-07-19.md
```

Expected: `1`

---

## Task 4: PR #6 评审段（轻批）

**Files:**
- Modify: `源文件/PR评审报告/2026-07-19.md`（追加）

- [ ] **Step 1: 写入 PR #6 评审段**

```markdown
## PR #6 — feat(team3): add ontology engineering v1 — 22 W3C-standard files + MVP demo

- **作者**：cx677
- **head**：`team3/ontology-engineering-v1` @ `dcf0acc`
- **base**：`main`
- **创建/更新**：2026-07-10T10:17:07Z / 2026-07-10T10:17:07Z
- **mergeable**：clean
- **体量**：16 文件，+6471 -0

### 合规性
- [x] 是否含二进制/疑似 secret — 否
- [x] 是否用 feature 分支 — 是（`team3/ontology-engineering-v1`）
- [x] body 是否完整填写 — 是（含 Summary、新增文件表、验证方式、关联文档）
- [ ] PR 描述与实际文件一致性 — **部分不符**。body 称"22 个 W3C 标准文件"，实际 16 个文件；body 称 MVP demo 在 `teams/agent-team/demo/`，与文件一致，但本体工程文件被放在 `agents/team3-ontology-core/`——与同 PR 内 demo 所在目录（`teams/agent-team/`）归属分裂

### 内容实质
16 个文件：OWL 主本体/SHACL/SWRL/SPARQL/5 个 JSON Schema/Zod 导出（放 `agents/team3-ontology-core/`）+ demo.mjs/3 份架构文档/README 更新（放 `teams/agent-team/`）。与同作者 PR #4（同日早 4 小时提交，37 文件）内容高度重叠——本体工程文件（ttl/shapes/swrll/sparql/schemas/zod）两 PR 逐行相同。

### 与本地基线对照
- **重复**：与 PR #4 重叠（本体工程文件逐行相同）；与本地 `ontology/agent-ontology.md`（380 行 markdown 版）是同一本体的不同形态
- **新增价值**：无独立于 #4 的新增——缺 #4 含有的 8 份抽取报告、3 份 ADR、quickstart、小白讲解、Mock→真实迁移方案
- **冲突**：目录归属分裂（本体在 `agents/`、demo 在 `teams/`），与项目"按 team 归置产出"的既有结构不一致

### 处置推荐
**close 候选** — 与 PR #4 为同作者同日重复 PR，内容为 #4 子集且目录归属分裂；保留 #4，关闭 #6。

### cherry-pick 清单
N/A — 推荐 close

### 证据
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/6`（mergeable=clean, changed_files=16）
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/6/files --paginate`（16 文件清单）
- 与 PR #4 文件清单比对：本体工程 9 文件逐行相同（ttl/shapes/swrll/sparql/4 schema/zod）

```

- [ ] **Step 2: 验证写入**

```bash
grep -c "^## PR #6" /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告/2026-07-19.md
```

Expected: `1`

---

## Task 5: PR #7 评审段（轻批，含 zip 解压取证）

**Files:**
- Modify: `源文件/PR评审报告/2026-07-19.md`（追加）
- Create: `/tmp/pr7-zip/`（临时，取证后删除）

- [ ] **Step 1: 下载 zip**

```bash
mkdir -p /tmp/pr7-zip
cd /tmp/pr7-zip
# 文件名含空格，URL 编码 %20
gh api "repos/a976xw7td/elite20-builder-program-nseap/contents/challenge-scoring-system%203.zip?ref=7807c09913556c8016eb668bbaeab5d2aa5fe506" \
  --jq '.content' | base64 -d > pr7.zip
ls -la pr7.zip
file pr7.zip
```

Expected: `file` 输出含 `Zip archive data`。若 `gh api` 报 "too large"（>1MB），改用 download_url：
```bash
URL=$(gh api "repos/a976xw7td/elite20-builder-program-nseap/contents/challenge-scoring-system%203.zip?ref=7807c09913556c8016eb668bbaeab5d2aa5fe506" --jq '.download_url')
curl -sL "$URL" -o pr7.zip
```

- [ ] **Step 2: 解压并查看内部结构**

```bash
cd /tmp/pr7-zip
unzip -o pr7.zip -d extracted >/dev/null
find extracted -type f | head -60
find extracted -type f | wc -l
find extracted -type d | head -30
```

Expected: 记录 zip 内实际文件清单。**关键判断**：zip 内是否含 body 声称的 18 challenge source files / 17 ontologies / 17 scoring skill packages / demo.py。将结论写进评审段"内容实质"。

- [ ] **Step 3: 写入 PR #7 评审段**

先按 Step 2 结果选择"内容实质"段的两种写法之一（zip 内含 18 挑战源码 / zip 内容与 body 不符），然后追加：

```markdown
## PR #7 — feat: Add Challenge Scoring System v1.0

- **作者**：cx677
- **head**：`main` @ `7807c09`
- **base**：`main`
- **创建/更新**：2026-07-14T13:24:24Z / 2026-07-14T13:24:24Z
- **mergeable**：clean
- **体量**：1 文件，+0 -0

### 合规性
- [ ] 是否含二进制/疑似 secret — **是**。PR 内容为单个二进制 zip（`challenge-scoring-system 3.zip`），源码不可审、不可 diff、不可增量维护
- [ ] 是否用 feature 分支 — **否**。head 为 `main`（fork 主分支直推）
- [ ] body 是否完整填写 — 部分。有 Summary/What's Included/Usage，但 What's Included 列表渲染异常（表格未生效）
- [x] PR 描述与实际文件一致性 — **内容基本一致、交付形式不合规**。解压后数量与 body 声明吻合，但 PR 页面仅显示 1 个 zip

### 内容实质
PR 仅含 1 个二进制 zip。解压后确认其中有 18 个核心挑战、17 份本体、17 个评分 Skill 包与 `demo.py`，另夹带 `node_modules`、`__MACOSX`、docx 和截图；内容存在，但交付形式错误。
本地 `challenges/official/`（同步至 `67e27a7`）已含 Richard 7.14 版 20 个挑战完整定义，本 PR 的评分维度需以源码形式重新提交才可评审。

### 与本地基线对照
- **重复**：与本地 `challenges/official/` 20 个挑战定义重叠（本 PR 是"评分系统"维度补充）
- **新增价值**：评分维度自动生成思路（5 步：completeness → quality → penalty → bonus → finalize）有参考价值，但当前交付形式不可用
- **冲突**：无二进制外冲突

### 处置推荐
**close 候选** — 二进制 zip 入 git 违反可审、可 diff、可维护的基本原则；需作者以源码目录结构重提（参考 challenges 仓库的目录组织）。

### cherry-pick 清单
N/A — 推荐 close

### 证据
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/7/files`（仅 1 文件 `challenge-scoring-system 3.zip`，+0 -0）
- zip 解压文件清单（Task 5 Step 2 输出；取证后临时目录已清理）

```

- [ ] **Step 4: 清理临时文件 + 验证写入**

```bash
rm -rf /tmp/pr7-zip
grep -c "^## PR #7" /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告/2026-07-19.md
```

Expected: `1`

---

## Task 6: PR #4 评审段（重批，含 README base/head diff 取证）

**Files:**
- Modify: `源文件/PR评审报告/2026-07-19.md`（追加）

- [ ] **Step 1: 取证 README -143 行内容**

```bash
cd /tmp
gh api "repos/a976xw7td/elite20-builder-program-nseap/contents/teams/agent-team/README.md?ref=8134a04f90917ebaf6337d5d07aadb6864cf485c" \
  --jq '.content' | base64 -d > pr4-readme-base.md
gh api "repos/a976xw7td/elite20-builder-program-nseap/contents/teams/agent-team/README.md?ref=0143c5d8a8bb693c721557bc69f8b09cbbff2fab" \
  --jq '.content' | base64 -d > pr4-readme-head.md
diff pr4-readme-base.md pr4-readme-head.md | head -80
wc -l pr4-readme-base.md pr4-readme-head.md
```

Expected: 看到 -143 行具体被删内容。**判断**：删除的是 Team 3 成员名单/职责分工/验收清单等关键信息，还是过时的占位文本。结论写进评审段。

- [ ] **Step 2: 写入 PR #4 评审段**

```markdown
## PR #4 — feat(team3): Agent 本体工程化 + MVP Demo

- **作者**：cx677
- **head**：`feat/team3-agent-ontology-mvp` @ `0143c5d`
- **base**：`main`
- **创建/更新**：2026-07-10T06:00:06Z / 2026-07-10T06:00:06Z
- **mergeable**：clean
- **体量**：37 文件，+16280 -143

### 合规性
- [x] 是否含二进制/疑似 secret — 否。宽泛规则仅误报 6 个 Skill 标题锚点，人工复核均非凭证
- [x] 是否用 feature 分支 — 是（`feat/team3-agent-ontology-mvp`）
- [x] body 是否完整填写 — 是（变更清单表、红线覆盖表、验证步骤、影响范围、产出示例）
- [ ] PR 描述与实际文件一致性 — **轻微不符**。body 明确声明"❌ 不修改主仓已有文件"，实际 `teams/agent-team/README.md` 被整体重写；base 158 行变为 head 283 行，成员、架构红线与验收类别均在新版保留并扩充

### 内容实质
37 个文件、16K 行，Team 3 本体工程化完整交付：
- 8 份语义抽取报告（事件/关系/实体概念/技能/流程/规则/语义模块 + 方法论，共 ~8000 行）
- 3 份架构文档（架构总览/小白讲解/主仓对比）
- OWL 2 DL 主本体（1137 行）+ SHACL 约束（671 行）+ SWRL 推理（1050 行）
- 5 个 JSON Schema（Agent Manifest / Message Envelope / Payloads / Challenge Record / Submission Record）+ 28 个 Zod 运行时导出
- 10 条红线 × 5 维度 = 50 处形式化强制；SPARQL 监控查询（222 行）
- 可运行 MVP demo（demo.mjs）+ Neo4j/Fuseki 加载与验证脚本

### 与本地基线对照
- **重复**：与本地 `ontology/agent-ontology.md`（380 行 markdown 版）是同一本体的不同形态——markdown 版保留，工程化版新增，不覆盖
- **新增价值**：本地完全缺 OWL/SHACL/SWRL/SPARQL/Zod 工程化形式化；缺 8 份抽取报告与 ADR；缺可运行 demo。线1 平台深化可直接消费 Zod schema
- **冲突**：无路径冲突（本地无 `teams/agent-team/`）；README -143 行相对 PR base 的删除内容需作者确认（见合规性）

### 处置推荐
**cherry-pick** — Team 3 正式交付物，内容与 #6 重叠但更完整；目录归属统一（全在 `teams/agent-team/`）。README -143 行不自动回滚，取证结论推到 GitHub 闭环阶段与作者沟通。

### cherry-pick 清单
全部 37 文件，落地 `teams/agent-team/`（逐文件清单与 head sha 映射见 `.cherry-pick-map.json`）

### 证据
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/4`（mergeable=clean）
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/4/files --paginate`（37 文件清单）
- README base/main/head 三版 diff（Task 6 Step 1）
- Demo、OWL、5 个 JSON Schema 均通过；Submission Record 与平台字段 27/27 同名

```

- [ ] **Step 3: 验证写入**

```bash
grep -c "^## PR #4" /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告/2026-07-19.md
```

Expected: `1`

---

## Task 7: PR #4 cherry-pick 落地（37 文件 → teams/agent-team/）

**Files:**
- Create: `teams/agent-team/`（37 文件）
- Create: `.cherry-pick-map.json`（项目根）

- [ ] **Step 1: 生成 PR #4 文件清单**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
gh api "repos/a976xw7td/elite20-builder-program-nseap/git/trees/0143c5d8a8bb693c721557bc69f8b09cbbff2fab?recursive=1" \
  --jq '.tree[] | select(.type=="blob") | .path' | grep '^teams/agent-team/' > /tmp/pr4-files.txt
wc -l /tmp/pr4-files.txt
```

Expected: `37`（与 PR files API 的 changed_files 数一致）。若不符，停下来比对差异。

- [ ] **Step 2: 逐文件下载落地**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
FAIL=0
while IFS= read -r path; do
  mkdir -p "$(dirname "$path")"
  if ! gh api "repos/a976xw7td/elite20-builder-program-nseap/contents/$path?ref=0143c5d8a8bb693c721557bc69f8b09cbbff2fab" \
      --jq '.content' | base64 -d > "$path"; then
    echo "FAIL: $path"; FAIL=1
  fi
done < /tmp/pr4-files.txt
echo "FAIL=$FAIL"
find teams/agent-team -type f | wc -l
```

Expected: `FAIL=0`；`find` 输出 `37`。

- [ ] **Step 3: 行数抽验（与 PR additions 对照）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
wc -l teams/agent-team/ontology/core/agent-ontology.ttl \
      teams/agent-team/ontology/core/agent-ontology-shapes.ttl \
      teams/agent-team/ontology/core/agent-ontology-rules.swrll \
      teams/agent-team/mvp-demo/demo.mjs \
      teams/agent-team/README.md
```

Expected: 1137 / 671 / 1050 / 474 / 268 附近（README 是 base+268-143 后的行数，与 PR files API 各文件 additions 对照，偏差 ±2 以内视为换行符差异可接受；否则重新下载该文件）。

- [ ] **Step 4: 写 .cherry-pick-map.json（PR #4 部分）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
{
  echo '{'
  echo '  "pr4": {'
  echo '    "head_sha": "0143c5d8a8bb693c721557bc69f8b09cbbff2fab",'
  echo '    "picked_at": "2026-07-20",'
  echo '    "files": {'
  FIRST=1
  while IFS= read -r path; do
    [ $FIRST -eq 0 ] && echo ','
    printf '      "%s": "%s"' "$path" "$path"
    FIRST=0
  done < /tmp/pr4-files.txt
  echo ''
  echo '    }'
  echo '  }'
  echo '}'
} > .cherry-pick-map.json
node -e "const m=require('./.cherry-pick-map.json'); console.log('pr4 files:', Object.keys(m.pr4.files).length)"
```

Expected: `pr4 files: 37`

---

## Task 8: PR #4 验证 4 项 + token 扫描 + commit

**Files:** 无新文件（验证 + commit）。

- [ ] **Step 1: demo 验证**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program/teams/agent-team/mvp-demo
node demo.mjs 2>&1 | tail -20
ls output/ 2>/dev/null || ls ../mvp-demo/output/ 2>/dev/null
```

Expected: 脚本正常退出（exit 0）；`output/` 下出现 bitable/chat/github 相关 JSON 输出。若报错，读错误信息——常见原因：demo 依赖相对路径数据文件缺失（检查 PR 内是否有数据文件未在 tree 清单中）→ 记录到评审报告并降级为"非阻塞、demo 数据依赖待补"。

- [ ] **Step 2: OWL 验证**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
python3 teams/agent-team/ontology/scripts/python/validate_owl.py \
  --owl teams/agent-team/ontology/core/agent-ontology.ttl 2>&1 | tail -10
```

Expected: exit 0，自带脚本的前缀、括号、类、属性、红线与枚举检查通过。

- [ ] **Step 3: JSON Schema 验证**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
python3 teams/agent-team/ontology/scripts/python/validate_json_schemas.py \
  --schemas teams/agent-team/ontology/schemas 2>&1 | tail -10
```

Expected: exit 0，5 个 schema 的 JSON 与结构检查通过。

- [ ] **Step 4: 字段对齐（探索性）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
echo "=== #4 submission-record properties ==="
node -e "console.log(Object.keys(require('./teams/agent-team/ontology/schemas/records/submission-record.schema.json').properties).sort().join('\n'))"
echo "=== 平台侧 zod submission 相关文件 ==="
grep -rln -i "submission" platform/ai-x-challenge-learning-mvp/lib/schemas/ 2>/dev/null || grep -rln -i "submission" platform/ai-x-challenge-learning-mvp/lib/ --include="*.ts" | head -5
```

然后读平台侧对应文件中 zod object 的字段名（`grep -E "^\s+\w+:" <file>`），与 #4 properties 集合做 venn 比对。**通过条件**：#4 schema 至少含 `student_id / challenge_id / system_validation_status / submitted_at` 4 字段；不一致项写进评审报告 PR #4 段"证据"。

- [ ] **Step 5: token/secret 扫描（PR #4 范围）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
grep -rE "ghp_|github_pat_|sk-[A-Za-z0-9_-]{20,}|FEISHU_APP_SECRET=[a-zA-Z0-9]" teams/agent-team/ || echo "CLEAN"
```

Expected: 无实际凭证。宽泛 `sk-` 规则可能误报 Skill 标题锚点，需人工复核；真实命中文件移出 commit 范围并在报告标注。

- [ ] **Step 6: commit**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
git add teams/agent-team/ .cherry-pick-map.json
git commit -m "cherry-pick PR #4 (head 0143c5d): teams/agent-team/"
git log --oneline | head -3
```

Expected: 新 commit 在 baseline 之上。

---

## Task 9: PR #1 评审段（重批）

**Files:**
- Modify: `源文件/PR评审报告/2026-07-19.md`（追加）

- [ ] **Step 1: 写入 PR #1 评审段**

Task 2 Step 2 已确认冲突文件为 `.gitignore`、`README.md`、`teams/knowledge-team/README.md`：

```markdown
## PR #1 — Team 6: Add Knowledge Cognitive Cell MVP

- **作者**：xnsnhwh-svg
- **head**：`main` @ `edfc307`
- **base**：`main`
- **创建/更新**：2026-07-01T03:33:21Z / 2026-07-10T15:10:25Z
- **mergeable**：**dirty**（与上游 main 冲突）
- **体量**：177 文件，+38463 -1

### 合规性
- [ ] 是否含二进制/疑似 secret — **是**。含 7+ 个 docx（IEEE 草稿、P2807.8、P3394、PRODUCT-SPEC 等）散落于仓库根与多处目录
- [ ] 是否用 feature 分支 — **否**。head 为 `main`（fork 主分支直推，rebase 必出乱）
- [x] body 是否完整填写 — 是（Type/Summary/Deliverables/Notes for Reviewers）
- [ ] PR 描述与实际文件一致性 — **部分不符**。Deliverables 所列（Knowledge Cell 设计/静态 demo/模板/schema/搜索索引）确有对应文件，但 PR 夹带大量未声明内容：根目录 7 个 docx、中文目录 `例子/`、根目录散落 `本体抽取-Skill.md` 等 3 份 md、**两套重复知识库**（`knowledge-base/knowledge-cognitive-cell/` 与 `nseap-knowledge-base/`，同主题两版本）

### 内容实质
177 个文件、38K 行，Team 6 Knowledge Cognitive Cell MVP。核心资产在 `nseap-knowledge-base/`（演进版，含 handoff-v1.0-prompt.md、SQLite 迁移方案、v0.3 worklog）：Knowledge Cell 产品设计（DESIGN.md 759 行）、静态 demo（app/）、7 类知识库目录（00-overview ~ 07-agents）、3 个 JSON Schema（knowledge-item/prompt/rubric）、Node 服务端（server.js 2636 行 + store.js）、搜索索引/数据构建脚本、10 个 markdown 模板。`knowledge-base/knowledge-cognitive-cell/` 为早期重复版本。

### 与本地基线对照
- **重复**：本地 `knowledge-base/`（7-10 同步基线）仅有 FAQ/prompts 等骨架目录；PR 提供完整 Knowledge Cell 实现
- **新增价值**：知识库 schema 三件套（线2 挑战本地化可直接消费）、knowledge-librarian-agent.md（线4 Companion Agent 参考）、搜索索引构建脚本
- **冲突**：mergeable=dirty，冲突文件：`.gitignore`、`README.md`、`teams/knowledge-team/README.md`。这些文件均不在 cherry-pick 范围内

### 处置推荐
**cherry-pick（仅 `nseap-knowledge-base/`）+ 推荐作者拆分重提** — 核心资产有明确价值但交付形态不合规（docx 入库、main→main、重复两套、散落根目录）。cherry-pick 后推荐给作者：拆掉 docx/例子/根目录散落文件、删重复旧版、用 feature 分支重提。

### cherry-pick 清单
`nseap-knowledge-base/` 全部文本文件（除 `PRODUCT-SPEC.docx`），落地 `knowledge-base/nseap/`（逐文件清单与 head sha 映射见 `.cherry-pick-map.json`）

### 证据
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/1`（mergeable=false, mergeable_state=dirty）
- `gh api repos/a976xw7td/elite20-builder-program-nseap/pulls/1/files --paginate`（177 文件清单）
- 冲突文件调查（Task 2 Step 2 compare 输出）
- 3 个 JSON Schema 可解析；知识数据构建 exit 0，生成 14 条知识项

```

- [ ] **Step 2: 验证写入**

```bash
grep -c "^## PR #1" /Users/an/工作空间/项目/Elite20-Builder-Program/源文件/PR评审报告/2026-07-19.md
```

Expected: `1`

---

## Task 10: PR #1 cherry-pick 落地（nseap-knowledge-base/ → knowledge-base/nseap/）

**Files:**
- Create: `knowledge-base/nseap/`（~78 文件）
- Modify: `.cherry-pick-map.json`（追加 pr1 段）

- [ ] **Step 1: 生成 PR #1 文件清单（仅 nseap-knowledge-base/，排除 docx）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
gh api "repos/a976xw7td/elite20-builder-program-nseap/git/trees/edfc3071aa9459c3e404734e2d5d63d21be5a7bb?recursive=1" \
  --jq '.tree[] | select(.type=="blob") | .path' \
  | grep '^nseap-knowledge-base/' \
  | grep -v '\.docx$' > /tmp/pr1-files.txt
wc -l /tmp/pr1-files.txt
grep -c '\.docx$' /tmp/pr1-files.txt || echo "OK: 无 docx"
```

Expected: 约 78 行；`OK: 无 docx`。

- [ ] **Step 2: 逐文件下载落地（路径重映射 nseap-knowledge-base/ → knowledge-base/nseap/）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
FAIL=0
while IFS= read -r path; do
  rel="${path#nseap-knowledge-base/}"
  dest="knowledge-base/nseap/$rel"
  mkdir -p "$(dirname "$dest")"
  if ! gh api "repos/a976xw7td/elite20-builder-program-nseap/contents/$path?ref=edfc3071aa9459c3e404734e2d5d63d21be5a7bb" \
      --jq '.content' | base64 -d > "$dest"; then
    echo "FAIL: $path"; FAIL=1
  fi
done < /tmp/pr1-files.txt
echo "FAIL=$FAIL"
find knowledge-base/nseap -type f | wc -l
```

Expected: `FAIL=0`；`find` 输出与 `/tmp/pr1-files.txt` 行数一致。

- [ ] **Step 3: 行数抽验**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
wc -l knowledge-base/nseap/DESIGN.md \
      knowledge-base/nseap/server/server.js \
      knowledge-base/nseap/schemas/knowledge-item.schema.json \
      knowledge-base/nseap/knowledge-base/07-agents/knowledge-librarian-agent.md
```

Expected: 759 / 2636 / 199 / 50 附近（±2 换行符差异可接受）。

- [ ] **Step 4: 更新 .cherry-pick-map.json（追加 pr1 段）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
node -e "
const fs = require('fs');
const m = JSON.parse(fs.readFileSync('.cherry-pick-map.json', 'utf8'));
const files = {};
fs.readFileSync('/tmp/pr1-files.txt', 'utf8').trim().split('\n').forEach(p => {
  files[p] = 'knowledge-base/nseap/' + p.replace(/^nseap-knowledge-base\//, '');
});
m.pr1 = { head_sha: 'edfc3071aa9459c3e404734e2d5d63d21be5a7bb', picked_at: '2026-07-20', files };
fs.writeFileSync('.cherry-pick-map.json', JSON.stringify(m, null, 2) + '\n');
console.log('pr1 files:', Object.keys(m.pr1.files).length);
console.log('pr4 files:', Object.keys(m.pr4.files).length);
"
```

Expected: `pr1 files: ~78`；`pr4 files: 37`。

---

## Task 11: PR #1 验证 2 项 + token 扫描 + commit

**Files:** 无新文件（验证 + commit）。

- [ ] **Step 1: schema parse 验证**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
find knowledge-base/nseap/schemas -name "*.json" -exec node -e "JSON.parse(require('fs').readFileSync(process.argv[1], 'utf8')); console.log('OK', process.argv[1])" {} \;
```

Expected: 3 行 `OK ...`（knowledge-item.schema.json / prompt.schema.json / rubric.schema.json），无 throw。

- [ ] **Step 2: build-knowledge-data.js 构建验证**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
grep -n "process.env" knowledge-base/nseap/scripts/build-knowledge-data.js || echo "无环境变量依赖"
cd knowledge-base/nseap
node scripts/build-knowledge-data.js 2>&1 | tail -10
ls -la app/knowledge-data.json data/knowledge-db.json 2>/dev/null
```

Expected: 无环境变量依赖或依赖已满足；脚本 exit 0；产出文件存在且非空。若脚本报缺依赖（如 `require('xxx')` 本地无 node_modules）→ 记录"非阻塞、缺依赖所致"，不 npm install（spec §2 边界）。

- [ ] **Step 3: token/secret 扫描（PR #1 范围）**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
grep -rE "ghp_|github_pat_|sk-[A-Za-z0-9_-]{20,}|FEISHU_APP_SECRET=[a-zA-Z0-9]" knowledge-base/nseap/ || echo "CLEAN"
```

Expected: `CLEAN`。若命中 → 文件移出 commit 范围并标注。

- [ ] **Step 4: commit**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
git add knowledge-base/nseap/ .cherry-pick-map.json
git commit -m "cherry-pick PR #1 (head edfc307): knowledge-base/nseap/"
git log --oneline | head -4
```

Expected: 3 个 commit（baseline / PR#4 / PR#1）。

---

## Task 12: 基线回归 + 同步记录 + 报告收尾 + close messages + 最终 commit

**Files:**
- Modify: `源文件/上游同步记录/2026-07-19.md`（末尾追加状态段）
- Create: `docs/superpowers/specs/2026-07-19-pr-close-messages.md`
- Modify: `源文件/PR评审报告/2026-07-19.md`（补验证结论）

- [ ] **Step 1: 平台基线回归**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program/platform/ai-x-challenge-learning-mvp
npm run lint 2>&1 | tail -6
npm run build 2>&1 | tail -15
```

Expected: lint 仅 2 个上游既有警告（submit/page.tsx exhaustive-deps、login/page.tsx no-html-link-for-pages）；build 成功。当前源码基线为 14 个页面文件、19 个 API handler。

- [ ] **Step 2: 同步记录末尾追加 cherry-pick 状态段**

向 `源文件/上游同步记录/2026-07-19.md` 追加：

```markdown

## 2026-07-20 cherry-pick 补充

本轮同步完成后，线3 对设计总部 5 个 open PR 做了评审与本地 cherry-pick：

| PR | head sha | 处置 | 本地落点 |
|---|---|---|---|
| #1 | `edfc3071aa9459c3e404734e2d5d63d21be5a7bb` | cherry-pick（仅 nseap-knowledge-base/，排除 docx） | `knowledge-base/nseap/` |
| #4 | `0143c5d8a8bb693c721557bc69f8b09cbbff2fab` | cherry-pick（全部 37 文件） | `teams/agent-team/` |
| #5 | — | 推荐 close（被平台仓 8e95299 取代） | 不落地 |
| #6 | — | 推荐 close（与 #4 重复且目录归属分裂） | 不落地 |
| #7 | — | 推荐 close（二进制 zip 入库，需源码重提） | 不落地 |

- 逐文件 head sha 映射：`.cherry-pick-map.json`
- 评审报告：`源文件/PR评审报告/2026-07-19.md`
- close 评论文案：`docs/superpowers/specs/2026-07-19-pr-close-messages.md`
- GitHub close/评论动作推迟：依赖用户吊销泄露 token、新建 fine-grained PAT、被 a976xw7td 加为 collaborator
- 下轮同步：先按 spec §11 表格判断 PR 状态，再决定是否覆盖本地 cherry-pick 内容
- 项目根已 git init；commit 序列：baseline → PR#4 cherry-pick → PR#1 cherry-pick → docs
```

- [ ] **Step 3: 评审报告补验证结论**

在报告 PR #4 段"证据"小节补：demo/OWL/JSON Schema/字段对齐 4 项验证的实际输出摘要；PR #1 段"证据"小节补：schema parse / build-knowledge-data 2 项验证输出摘要。末尾追加：

```markdown

## 总结

- 5 个 PR 评审完成：2 个 cherry-pick 落地（#1 #4），3 个推荐 close（#5 #6 #7）
- 平台基线回归通过（lint/build 与 7-19 同步状态一致）
- token/secret 扫描：未发现实际凭证；宽泛规则误报的 Skill 锚点已人工排除
- GitHub 闭环（close/评论）推迟，依赖见 `docs/superpowers/specs/2026-07-19-pr-pruning-design.md` §8
```

- [ ] **Step 4: 写 close 评论文案**

创建 `docs/superpowers/specs/2026-07-19-pr-close-messages.md`：

```markdown
# PR close/评论文案 — 2026-07-19

> 用法：GitHub 闭环前置条件满足后，逐段复制执行。
> 本文件不含任何 token/secret。

## §PR-7

**动作**：`gh pr close 7 --repo a976xw7td/elite20-builder-program-nseap -c "$(cat <<'MSG' ... )"`

感谢提交 Challenge Scoring System 的想法。本 PR 存在几个无法合并的问题：

1. PR 内容仅为 1 个二进制 zip（`challenge-scoring-system 3.zip`），源码不可审、不可 diff、不可增量维护；
2. 解压后内容与数量基本符合描述，但全部封装在 zip 中，PR 页面不可逐文件审查；
3. head 使用 main 分支直推，不利于迭代。

建议：以源码目录结构重新提交（参考 nseap-elite20-challenges 仓库的目录组织），每个挑战的评分维度/skill 包以文本文件（md/json/py）形式入库。评分五步流水线（completeness → quality → penalty → bonus → finalize）的思路有价值，期待源码版。

## §PR-5

感谢 Platform Team 的早期 mock。本 PR 的页面（Landing/LMS/提交流程/作品集/教师控制台）已在平台仓 a976xw7td/ai-x-challenge-learning-mvp 中完整实现并远超此版本（含 REST API、RBAC、Redis Stream、DeepSeek 评分、飞书通知、Agent Envelope、审计链路）。本 PR 作为团队工作历史有归档价值，但无合并价值，建议关闭。

## §PR-6

本 PR 与同作者 PR #4（同日早 4 小时）内容高度重叠：本体工程 9 个核心文件（ttl/shapes/swrll/sparql/schemas/zod）两 PR 逐行相同，且本 PR 缺少 #4 含有的 8 份抽取报告、3 份 ADR 与配套文档；同时本 PR 目录归属分裂（本体在 agents/team3-ontology-core/、demo 在 teams/agent-team/）。建议保留更完整、归属统一的 #4，关闭本 PR。

## §PR-4

本体工程化交付内容完整（8 份抽取报告 + OWL/SHACL/SWRL/SPARQL/Zod 五维形式化 + 可运行 demo），评审通过樱桃挑选方式进入本地基线。合并前请确认一个问题：PR 描述声明"不修改主仓已有文件"，但 `teams/agent-team/README.md` 实际从 base 158 行整体重写为 283 行；成员、架构红线与验收类别虽已保留并扩充，请确认这是有意变更。

## §PR-1

Knowledge Cognitive Cell MVP 的核心资产（nseap-knowledge-base/ 演进版：DESIGN/schema/模板/脚本/服务端）有明确价值。但本 PR 无法直接合并：

1. 含 7+ 个 docx 二进制散落仓库根与多处目录；
2. head 为 main 分支直推，且当前与上游 main 冲突（dirty）；
3. 夹带未声明内容：中文目录 `例子/`、根目录散落 md、两套重复知识库（knowledge-base/knowledge-cognitive-cell/ 与 nseap-knowledge-base/）。

建议：删除 docx 与散落文件、去重（保留 nseap-knowledge-base/ 演进版）、改用 feature 分支后重提。
```

- [ ] **Step 5: 最终 commit**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
git add .gitignore 源文件/PR评审报告/2026-07-19.md 源文件/上游同步记录/2026-07-19.md \
  docs/superpowers/specs/2026-07-19-pr-pruning-design.md \
  docs/superpowers/specs/2026-07-19-pr-close-messages.md \
  docs/superpowers/plans/2026-07-20-pr-pruning.md
git commit -m "docs: finalize PR pruning review"
git log --oneline
```

Expected: 4 个 commit（baseline / PR#4 / PR#1 / docs）。

- [ ] **Step 6: 验收清单终验**

```bash
cd /Users/an/工作空间/项目/Elite20-Builder-Program
echo "=== 1. 报告 5 章节 ==="
grep -c "^## PR #" 源文件/PR评审报告/2026-07-19.md   # 期望 5
echo "=== 2. cherry-pick 文件数 ==="
git ls-files teams/agent-team | wc -l                # 期望 37
git ls-files knowledge-base/nseap | wc -l            # 期望 84
echo "=== 3. map 文件 ==="
node -e "const m=require('./.cherry-pick-map.json'); console.log('pr1:', Object.keys(m.pr1.files).length, 'pr4:', Object.keys(m.pr4.files).length)"
echo "=== 4. commit 数 ==="
git log --oneline | wc -l                            # 期望 4
echo "=== 5. token 扫描全局 ==="
grep -rE "ghp_[A-Za-z0-9]|github_pat_[A-Za-z0-9]" --include="*.md" docs/ 源文件/PR评审报告/ || echo "CLEAN"
```

Expected: 全部符合注释中的期望值。

---

## Self-Review 记录

- **Spec coverage**：spec §1-§11 全部有对应 Task：§3.1→Task 10、§3.2→Task 7、§4→Task 1-12 顺序、§5.0→Task 2、§5.1→Task 3-5、§5.2→Task 8、§5.3→Task 11、§5.4→Task 12 Step 1、§5.5→Task 8/11/12、§6 风险→Task 1(.gitignore)/2(依赖)/5(zip)/6(README diff)/10(dirty 绕开)、§7 验收→Task 12 Step 6、§8→Task 12 Step 4、§10 模板→Task 3-6/9 报告段结构、§11→同步记录段
- **Placeholder 扫描**：运行时取证占位已按实际结果回填；仅保留命令示意中的 `<path>` / `<file>` 参数
- **Type consistency**：head sha 常量全部在文件头定义并在 Task 中一致引用；路径 `teams/agent-team/`、`knowledge-base/nseap/`、`.cherry-pick-map.json` 全程一致
