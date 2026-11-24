# 执行指南：在各 AI 编码助手中使用本技能

本文给出可复制的操作流程。前提：你已经读过 [`README.md`](README.md) 与 [`architecture.md`](architecture.md)；具体脚本参数见 [`modules_reference.md`](modules_reference.md)。

## 1. 安装与平台适配

### 1.1 通用原则

唯一规范入口是仓库根目录 [`SKILL.md`](../SKILL.md)（`README.md:163-174`）。各平台的“skills 目录”机制不同，但做法一致：

```bash
git clone https://github.com/jin-s13/ai-research-writing-skill.git
# 将整个仓库复制或软链接到 agent 的 skills 目录
```

也可以不安装：打开仓库后直接说：

```text
Use the AI Research Writing Skill in this repository.
Follow SKILL.md and load only the relevant references for my task.
```

### 1.2 常见助手的具体做法

| 助手 | 推荐方式 | 注意事项 |
|---|---|---|
| Claude Code / 兼容 Skills 的 CLI | 将仓库放到项目的 `.claude/skills/ai-research-writing-skill/`（或平台全局 skills 目录），确保 `SKILL.md` 位于该技能根目录 | 入口文件必须叫 `SKILL.md`；`validate_skill()` 要求 frontmatter 只有 `name`/`description`（`research_quality_gate.py:102-116`） |
| Cursor | 在 `.cursorrules` 或规则文件中写明“先读 `SKILL.md`，再按路由读 `references/`”；把仓库放在工作区内 | 不要让 agent 默认加载整个 references 树 |
| OpenAI Codex / 通用 coding agent | 在任务提示中给出仓库绝对路径与入口文件，并要求执行 `scripts/` 下的确定性检查 | 脚本只依赖 Python 3 标准库；CI 覆盖 3.10–3.13 |
| 无 skills 机制的 assistant | 直接把 `SKILL.md` 内容作为 system/task 前缀，并允许读取 `references/` 与执行脚本 | 保持“产物写在用户论文项目”的边界 |

平台差异**不改变**入口契约；不要为某个助手复制一份分叉的 `SKILL.md`。

### 1.3 环境要求

- Python 3.10+（CI 矩阵为 3.10/3.11/3.12/3.13，`.github/workflows/quality.yml:16`）。
- 可选：LaTeX 工具链（`pdflatex`/`bibtex` 或 `latexmk`/`tectonic`）；没有工具时只能做静态检查并记录 blocker。
- 在线引用验证需要网络与一枚可联系邮箱；OpenAlex 交叉核验可选 `OPENALEX_API_KEY`。

## 2. 任务路由提示词（直接复制）

下列提示词对应 `SKILL.md:15-27` 的路由表。始终要求 agent **先报告它选择了哪条路由、读了哪些文件**。

```text
I only have a rough idea. Use AI Research Writing Skill to brainstorm the paper story,
identify the research gap, define claims to make/avoid, and create a concrete plan for
evidence, experiments, figures, and writing.
```

```text
I have code, notes, method design, and some experiment logs/results. Use AI Research Writing Skill
to generate a complete paper package: paper_story.md, claim_evidence_map.md, literature positioning,
verified BibTeX, figures, tables, LaTeX draft, and build_check.md.
```

```text
I have a paper draft. Use AI Research Writing Skill as a skeptical ICML reviewer: write detailed
reviewer comments, identify rejection risks, and convert the high-risk issues into concrete revisions.
```

```text
I have a draft and want to revise it. Use AI Research Writing Skill to improve the paper section by
section, preserve supported claims, weaken unsupported claims, fix citations, improve figures/tables,
and update the LaTeX package.
```

投稿阶段：

```text
Use the submission route from SKILL.md. Load references/submission-packaging.md and
references/citation-checklist.md, run every declared check, record the build with
record_build.py --run, and do not report completion while blockers remain.
```

## 3. 完整论文：逐步执行脚本

设：
- `SKILL=/path/to/ai-research-writing-skill`
- `PAPER=/path/to/your-paper-project`

### 3.1 启动

```bash
# 1) 在 PAPER 中创建 paper_state.json / task packet / plan/progress.md（按 references/artifacts.md）
# 2) 早期质量门：让缺口可见
python3 "$SKILL/scripts/research_quality_gate.py" "$PAPER"
```

若存在上游 handoff：

```bash
python3 "$SKILL/scripts/check_research_handoff.py" "$PAPER" --require-unblocked
```

### 3.2 证据与主张

```bash
# 从 Abstract/Introduction 启动 claim map（启发式，需人工补证据）
python3 "$SKILL/scripts/extract_claims.py" "$PAPER/paper/main.tex" > "$PAPER/claim_evidence_map.md"

# 数值溯源：先写 numeric_evidence.json（v2），再检查
python3 "$SKILL/scripts/check_numeric_evidence.py" "$PAPER"
```

数值 registry 示例（`references/numeric-evidence.md:5-20`）：

```json
{
  "schema_version": "ai-research-writing/numeric-evidence-v2",
  "entries": [{
    "value": 0.912,
    "source": "experiments/raw_results.csv",
    "selector": {"kind": "csv", "column": "accuracy", "where": {"method": "ours"}},
    "aggregate": "mean",
    "representations": ["raw", "percent"],
    "tolerance": 0.0001,
    "note": "Mean test accuracy over matching rows"
  }]
}
```

### 3.3 图表

```bash
python3 "$SKILL/scripts/make_latex_table.py" "$PAPER/results.csv" \
  --caption "Main results." --label tab:main --precision 2 \
  --output "$PAPER/paper/tables/main.tex"

python3 "$SKILL/scripts/fetch_template.py" --list
python3 "$SKILL/scripts/fetch_template.py" icml2026 --output "$PAPER/.venue-template"
```

概念图（overview/method/teaser）由 agent 的图像生成能力直接产出并写进论文，Prompt 按 `figure-workflow.md:118-129` 的六段结构保存为 `figures/*_prompt.md`；精确数字图/表格只能确定性生成。

### 3.4 引用

```bash
# 1) 写 citation_requests.json（每个学术引用含逐句 claim_support）
python3 "$SKILL/scripts/verify_citations.py" "$PAPER" --mailto you@example.org
python3 "$SKILL/scripts/check_citation_lock.py" "$PAPER"

# 2) 内部 key 一致性
python3 "$SKILL/scripts/check_citations.py" "$PAPER/paper/main.tex" "$PAPER/paper/references.bib"
```

终态下 `citation_verification.md` 必须有 Key/Status 表，且每个 cited key 状态为 `verified`/`software-doc`/`repository-verified`。

### 3.5 评审与收尾

```bash
python3 "$SKILL/scripts/check_todos.py" "$PAPER/paper/main.tex" "$PAPER/paper/references.bib" "$PAPER/paper/tables" "$PAPER/paper/figures"
python3 "$SKILL/scripts/parse_build_log.py" "$PAPER/paper/main.log"
python3 "$SKILL/scripts/camera_ready_check.py" "$PAPER/paper/main.tex" --checklist "$PAPER/paper/checklist.tex"
python3 "$SKILL/scripts/record_build.py" "$PAPER" --run
python3 "$SKILL/scripts/research_quality_gate.py" "$PAPER"
```

顺序很重要：`record_build.py --run` 会真的执行 `paper_state.json` 中声明的 `build.command`，并在失败时把 `build.status` 写为 `failed`。确认命令、cwd（`main_tex.parent`）与工具版本后再跑。

## 4. 不同任务的最小命令集

| 任务 | 最少产物 | 最少检查 |
|---|---|---|
| Story brainstorm | `paper_story.md` | 无（人工审查 claim 强度） |
| 单节修改 | 修改的节 + claim/evidence notes | `check_todos.py`, `check_citations.py`（如改了引用） |
| Related Work | inventory/matrix/positioning + citation 更新 | `check_citations.py`, `check_citation_lock.py` |
| 图/表 | plan/spec + 资产 + 源 | `make_latex_table.py`（表格）、`resolve_graphics` 隐式校验 |
| 引用修复 | `references.bib`, `citation_verification.md`, requests + lock | `verify_citations.py`, `check_citation_lock.py` |
| Peer review | review + evidence notes（可在回复中） | 不需要完整 state/build |
| Full paper | state + 核心工件 | 全套 + `record_build.py` |
| Submission | full-paper 集 + checklist + readiness + 记录构建 | 全套 + `camera_ready_check.py` + `--run` |

依据：`references/artifacts.md:56-67`。

## 5. 与其他系统协作（handoff）

上游研究系统（例如 auto-research）导出 `research_handoff.json`，本技能只接手写作。规则（`references/research-handoff.md`）：

1. 先 `check_research_handoff.py ... --require-unblocked`；
2. 核对 analysis / decision / raw evidence / blockers 是否矛盾，不得抹掉 negative runs；
3. 把 blockers 原样复制到 `paper_state.json`；
4. 从证据构建 story，**不要**采用上游建议标题；
5. 从 `workflow.md` 的“证据承载核心草稿”继续。

## 6. 故障排查表

| 现象 | 典型原因 | 处理 |
|---|---|---|
| `Required file does not exist` | `paper_state.json` 路径写错或 `main_tex` 未创建 | 按报错路径补文件；不要删状态字段 |
| `Missing LaTeX input declared by ...` | `\input{}` 指向不存在的文件，或漏写 `.tex` 后缀 | 修正 TeX 输入图（`resolve_tex()` 会自动补 `.tex`） |
| `Missing graphic declared by ...` | `\includegraphics` 路径错或后缀不在允许集合 | 补齐文件/后缀（pdf/png/jpg/jpeg/eps/svg） |
| `Unverified numbers ...` | 论文数字未登记或注册值与重算不符 | 更新 `numeric_evidence.json` 的 selector/aggregate/value；禁止直接关闭检查 |
| `citation_lock.json is stale` | 请求文件改了但没重新验证 | 重跑 `verify_citations.py`，但先确认 claim_support 是否真的变更 |
| `citation lock is older than 180 days` | 锁过期 | 重新在线验证；可用 `--max-age-days` 调整，但需记录理由 |
| `A terminal stage cannot contain unresolved blockers` | 还有 blocker | 解决它或把 stage 退回非终态 |
| `Build input digest is stale` | 改了 TeX/图/bib 但没重建 PDF | 重新运行声明的 build 与 `record_build.py --run` |
| `PDF is older than one or more build inputs` | 同上（mtime 检查） | 同上 |
| `Process instruction leaked into manuscript` | “写作要求/图位/请用户” 等过程话术进入 TeX | 移到 `plan/` 或 review 文件，不要留在正文 |
| `Vendored template directories are not allowed` | 把作者套件直接放进 `templates/` | 用 `fetch_template.py` 下载到论文项目 |
| `research_quality_gate.py` 自动进入 skill 模式 | 项目根同时有 `SKILL.md` 与 `references/` | 用 `--mode full-paper` 显式覆盖（仍需与 state mode 一致） |

## 7. 扩展与贡献流程

1. 在论文项目中验证需求，再决定是否属于技能层；**论文特有产物不要提交到技能仓库**。
2. 新增参考文件：更新 `references/README.md` 地图与 `SKILL.md` 路由表。
3. 新增脚本：stdlib-only、fail-closed、更新 `scripts/README.md` 与 CI。
4. 改 schema：同步 JSON schema、脚本字段集、测试、文档。
5. 本地回归：

   ```bash
   python3 -m unittest discover -s tests -v
   python3 -m py_compile scripts/*.py
   python3 scripts/research_quality_gate.py . --mode skill
   python3 scripts/research_quality_gate.py examples/paper-about-ai-research-writing-skill
   ```

6. 最终报告按 `references/submission-packaging.md:108-116`：文件、检查、构建状态、blocker、commit/push（如执行）。

## 8. 安全与卫生

- 不要提交私有 PDF、专有日志、API key 或审稿保密材料。
- 论文 PDF/下载文献仅在许可允许且用户明确要求时入库；否则记 `metadata-only` / `needs-access`（`references/artifacts.md:54`, `citation-workflow.md:96-98`）。
- 投稿前检查匿名性、致谢、仓库链接与文件元数据。
- 任何“跳过验证”的提议都应转化为显式 blocker，而不是让质量门变绿。
