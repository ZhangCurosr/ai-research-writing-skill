# Prompt 模板与质量闸门完全参考

> 事实来源：本仓库源码与 `references/` 规范文本；`路径:行号` 为本检出版本锚点。
>
> **先破除一个误解**：本技能包里**没有任何 Prompt 渲染引擎**。`scripts/` 下 15 个 Python 文件只做确定性检查与模板下载；所谓“内置 Prompt”全部是写在 `references/*.md` 里的**结构化指令文本**，由 agent 读取后自行填空、自行执行（`docs/modules_reference.md:41-45`）。因此本文的“占位符”是**语义槽位**，不会在运行时被 `str.format()` 替换。

---

## 1. Prompt 体系总览

### 1.1 三层结构

| 层 | 内容 | 位置 | 是否机器校验 |
|---|---|---|---|
| 入口约束 | Mandate / Route / Execution Contract / Gates / Evidence Boundaries | `SKILL.md:8-56` | 部分：frontmatter、本地资源存在性（`research_quality_gate.py:102-164`） |
| 规范模板 | 故事、摘要、章节、文献、引用、图、审稿、投稿的填空骨架 | `references/*.md` | 不校验文本是否“填得好”；只校验终态产物里没有未解决标记 |
| 确定性检查 | 数字、引用、handoff、TODO、构建、总闸门 | `scripts/*.py` | 是；严格 fail-closed |

### 1.2 占位符书写约定

| 形态 | 示例 | 出自 | 填写规则 |
|---|---|---|---|
| 方括号槽 | `[artifact/idea]`, `[technical insight]` | `paper-story.md:73` | 用具体内容替换并**删除方括号**；残留 `[` `]` 不代表失败，但通常是未写完的信号 |
| 尖括号/斜杠枚举 | `evidence-result | concept-method` | `figure-spec.md:12-13` | 二选一或按枚举选一个，删掉其余选项 |
| 省略号续写 | `Line A solves ...`、`However, ...` | `paper-story.md:104-146` | 每个 `...` 是一句实义内容，不要保留省略号 |
| Markdown 表格空列 | `|  | needs evidence |  |` | `extract_claims.py:128-134` | Evidence/Revision 必须由人或 agent 填；Status 终态不能是未解决词 |
| YAML 字段 | `message: "One sentence claim the figure supports."` | `figure-spec.md:9-34` | 保留键名，替换值；`source/backup` 必须指向真实文件 |
| JSON 字段 | `"main_tex": "paper/main.tex"` | `artifacts.md:9-28` | 字段名白名单严格；未知字段会让闸门失败 |

**关键区别**：`SKILL.md:48-56` 的占位标记（如 `CITATION NEEDED`、`公式占位`、`图位`）出现在手稿里会被 `UNRESOLVED_RE` / `PROCESS_RE`（`research_quality_gate.py:39-49`）判为 failure。规范文件里的方括号模板不是失败对象——只有**手稿正文**会被扫描。

---

## 2. 内置模板逐项清单

### 2.1 故事与贡献（`references/paper-story.md`）

| 模板 | 行号 | 槽位 / 输出结构 | 用途 |
|---|---|---|---|
| 12 项 `paper_story.md` 必备章节 | :7-20 | Working title / One-sentence thesis / Problem gap / Technical challenge / Method insight / Method summary / Contributions / Experimental evidence / Claims to make / careful / avoid / Reviewer risks | 故事文档骨架 |
| `project_inventory.md` 6 节 | :22-29 | Repository map / Method evidence / Experiment evidence / Writing assets / Citation assets / Missing inputs | 从仓库起步时盘点 |
| `experiment_inventory.md` 8 节 | :31-40 | benchmark / task / baselines / metrics+direction / main numbers / variance+seeds / caveats / supported claim | 有实验时必建 |
| Thesis Test | :70-76 | `We introduce [artifact/idea] for [task] that enables [capability] by [technical insight], and we show [evidence].` | 一句话检验故事是否收敛；退化成模块列表即失败 |
| Contribution Test 四问 | :78-87 | What is new? / Why nontrivial? / Technical advantage? / Evidence? | 每条贡献逐问 |
| Naming Test | :89-97 | 避免泛化名、避免假统一模型、贴近 task/mechanism/representation、查重名 | 命名 |
| Pattern A 两条研究线之间的 gap | :104-111 | `Line A solves ... / Line B solves ... / However, neither supports ... / We introduce ...` | 跨领域桥接型论文 |
| Pattern B 可执行工作流 | :117-123 | `Existing methods can produce X, but practical use requires Y. / Y is hard because ... / We separate semantic reasoning from deterministic execution.` | pipeline/agent 论文；强调接口、工件、验证、失败恢复 |
| Pattern C benchmark/evaluation | :129-136 | `Prior evaluations measure ... / They miss ... / We introduce a benchmark/protocol that tests ... / The benchmark reveals ...` | 贡献是测量时不要冒充方法论文 |
| Pattern D method insight | :142-148 | `We observe that ... / This suggests ... / We operationalize this insight by ...` | 单观察驱动方法 |
| Story Deliverable 模板 | :163-195 | `# Paper Story` + Thesis / Task Boundary(Inputs/Outputs/Supported/Out-of-scope) / Gap / Technical Challenge / Method Insight / Stages / Contributions / Evidence / Related Work Positioning / Claims×3 / Reviewer Risks | 创建 `paper_story.md` 时直接复制 |
| Claim 四级强度 | :150-157 | `strong / careful / aspirational / forbidden` | 写进 story 与 claim map |

### 2.2 摘要与引言（`references/abstract-introduction.md`）

| 模板 | 行号 | 槽位 | 硬约束 |
|---|---|---|---|
| 摘要预写六问 | :9-18 | 技术问题 / 为何无成熟解 / 核心贡献 / 为何可行 / 技术优势 / 证据 | 任一问答不出 → 先补证据再写 |
| Template A：Challenge → Contribution | :22-31 | 任务需求 → 先前技术挑战 → 方法与核心贡献 → 收益 → 带 benchmark/metric 的主要证据 | 单一中心方法/工作流 |
| Template B：Challenge → Insight → Contribution | :32-41 | 任务 → 挑战 → 一句话 insight → 实现 insight 的方法 → 实现优势 → 证据 | 有清晰设计洞察 |
| Template C：Multiple Contributions | :43-51 | 任务+gap → 贡献1+优势 → 贡献2+优势 → 贡献3+优势 → 实验总结 | 多组件且缺一不可 |
| 摘要诊断（反向检查） | :53-62 | 6 条 reject 条件（任务不清、模块名先于问题、实现步骤列表、结果无 metric 方向、无证据的 strong words、隐藏任务定义差异） | 命中即重写 |
| Introduction 先逆向六问 | :64-72 | reviewer 应记住什么 / 什么证据 / 什么挑战 / 哪个 prior gap / 什么应用背景 | 先逆向再正向 |
| Introduction 正向六段 | :74-81 | 任务应用 → prior gap → 根本挑战 → 方法与为何有效 → 证据 → 贡献 | 段落角色来源 |
| Opening A：Task First | :89-91 | `[Task] aims to produce [output] from [input]. This setting matters for ...` | 新/窄任务 |
| Opening B：Application First | :97-99 | `[Application area] increasingly requires ... Existing methods therefore need ...` | 读者已知任务 |
| Opening C：General to Specific | :105-107 | `[General area] has enabled ... This paper focuses on [specific setting], where ...` | 已知领域的新设定 |
| Opening D：Open With Challenge | :113-115 | `Although recent methods can ..., they still fail when ... because ...` | 失败模式广为人知 |
| Technical Challenge：Existing Task | :121-125 | `This problem is challenging because ... / Traditional methods ... However, ... / Recent methods ... However, ... because ...` | 三段式 |
| Technical Challenge：Classical Insight | :129-133 | `A classical way ... / However, classical methods still ... / Modern methods improve ..., but still fail to ... because ...` | 古典方法→现代方法 |
| Technical Challenge：Novel Task | :137-143 | `In this work, our goal is to ... / This problem is challenging for three reasons. / First, ... / Second, ... / Finally, ...` | 新任务 |
| Pipeline：One Contribution | :149-154 | `We introduce [method/workflow] for [task]. / The key idea is ... / Specifically, ... / This design enables ...` | 单贡献多优势 |
| Pipeline：Two Contributions | :158-163 | `We introduce [framework]. / Its first component ... / However, this leaves ... / To address this, the second component ...` | 两组件递进 |
| Pipeline：New Module | :167-171 | `Existing pipelines already ... / The remaining bottleneck is ... / We add [module] that ...` | 在既有流水线上加模块 |
| Pipeline：Observation Driven | :175-179 | `We observe that ... / This suggests ... / We operationalize this insight by ...` | 观察驱动 |
| Reverse Outline 表 | :185-196 | `Paragraph / Role / First-sentence message / Evidence / Risk / Revision` + 4 条改写规则（:191-196） | 草稿后强制复盘 |
| Contribution Bullet | :202-206 | `Contribution artifact/idea + why it matters + evidence or evaluation location` | 贡献条目 |

**共享硬规则**：`abstract-introduction.md:5-7` —— 摘要/引言每条主要主张必须在 claim-evidence map 有一行；证据缺失或混合时**先弱化主张再润色文字**。

### 2.3 章节写作（`references/section-writing.md`）

| 模板/循环 | 行号 | 槽位 |
|---|---|---|
| 每节六步 Writing Loop | :7-14 | one-sentence section thesis → 每段一个 role → 强主张映射证据 → 一个 message/段 → reverse outline → 跨节术语/流程修补 |
| Method/System 组件五要素 | :24-30 | purpose+motivating failure mode / concrete inputs & outputs / operation or algorithm / invariant, contract, advantage / failure behavior & limitation |
| Agent 系统特别说明 | :32 | 区分 model reasoning 与 deterministic parsers/tools/data/checks；描述 artifact flow 与 control flow，而非实现文件名 |
| Experiments 六项 | :36-45 | research question / dataset+split+benchmark / baselines+fairness / metrics+direction / seeds+uncertainty+compute / main result+interpretation+caveat；:47 规定 `shows` / `suggests` / `is consistent with` 的用法 |
| Limitations | :49-51 | scope / assumptions / failure modes / missing evaluation / reproducibility gaps / cost，写成科学边界而非道歉 |
| Conclusion | :53-55 | 重述问题与已证贡献 → 最强证据 → 主要剩余边界；不得引入新结果 |
| Revision Output | :59-65 | mini-outline / 修订文本 / reverse outline / claim-evidence 变化 / 剩余实验、引用、定义、审稿风险 |

### 2.4 文献与定位（`references/literature-review.md`）

| 模板 | 行号 | 列/槽位 |
|---|---|---|
| Paper Inventory 表 | :23-27 | `Key / Title / Year-Venue / Local file-URL / Role / Status / Notes`；Status 枚举在 :29-37（`downloaded/metadata-only/read/summarized/verified-citation/needs-access/not-relevant`） |
| Paper Note | :43-75 | 标题 → Bibliographic Metadata（Key/Title/Authors/Venue-year/DOI-Arxiv-URL/Local file）→ Problem Setting（Input/Output/Assumptions/Evaluation）→ Core Idea → Evidence（Datasets/Baselines/Metrics/Main results）→ Relation to This Paper（是否同任务/同 I-O/解决什么/未覆盖什么/如何引用/必须避免的主张） |
| Related Work Matrix | :81-85 | `Line of work / Representative papers / What they solve / What they do not cover / How we relate / Citation role`；role 取 `background/contrast/baseline/concurrent` |
| Positioning | :93-113 | Closest Works 表（Paper/Similarity/Difference/Risk if not discussed）+ Our Position 6 项 + Related Work Paragraph Plan 4 项 |
| 工作流 | :115-129 | 搜索 → 保存官方副本 → 核验元数据与稳定 key → 读并总结 → matrix+positioning → 才写 Related Work |

### 2.5 引用、数值、状态（`references/citation-workflow.md`, `numeric-evidence.md`）

| 模板 | 行号 | 内容 |
|---|---|---|
| 来源路由表 | :15-21 | DOI→Crossref/注册机构 (+OpenAlex)；arXiv ID→官方 API；PMID/PMCID→PubMed；仅标题→OpenAlex/S2；软件/数据→官方仓库/docs |
| 六步流程 | :36-45 | Segment → Discover → Resolve → Verify metadata → Read for support → Record |
| 逐句关系词表 | :47-59 | `direct / background / contrast / software / partial / weak / metadata-only`；强技术/比较型主张只允许 `direct/contrast/software` |
| Durable Record 表 | :65-69 | `Key / Identifier / Primary source / Cross-check / Claim supported / Relation / Status / Checked at` |
| 终态/非终态状态 | :71-77 | 终态 `verified / software-doc / repository-verified`；非终态 `metadata-only / conflict / needs-access / placeholder` |
| numeric-evidence v2 JSON | `numeric-evidence.md:5-20` | `schema_version / entries[{value,source,selector,aggregate,representations,tolerance,note}]` |
| selector/aggregate 契约 | `numeric-evidence.md:22-28` | kinds = `json-pointer/csv/tsv/jsonl`；aggregates = `identity/mean/sample-std/population-std/sum/min/max/count` |

### 2.6 状态、任务与交接（`references/artifacts.md`, `task-management.md`, `research-handoff.md`）

| 模板 | 行号 | 关键槽位 |
|---|---|---|
| `paper_state.json` | `artifacts.md:9-28` | schema_version / mode / stage / target_venue / main_tex / bibliography / required_artifacts / blockers / build{status,command,pdf,external_inputs,input_sha256,pdf_sha256} |
| 核心工件表 | `artifacts.md:41-52` | 范围执行 / 仓库实验 / 数值溯源 / story 主张 / 文献 / 引用 / 图表 / 自审 / 审稿 / 构建投稿 |
| Task Packet | `task-management.md:9-19` | Scope / Target venue-template / Inputs and evidence / Files allowed to edit / Required artifacts / Rejection checks / Validation commands / Acceptance criteria |
| Progress | `task-management.md:25-32` | Stage / Inputs consumed / Artifacts produced / Verification run / Remaining scientific risk；:34 禁止终态留 `verification planned/to be recorded/待验证` |
| `research_handoff.json` | `research-handoff.md:11-29` | schema_version / research_question / paper_type / target_venue / quantitative / artifacts{project_inventory,analysis,decision,experiment_inventory,numeric_evidence,literature_inventory,figure_inventory} / blockers |
| 所有权边界 | `research-handoff.md:39-43` | 上游只导证据；本技能拥有 handoff 后所有论文决定；禁止把上游写作 prompt 复制进技能契约 |

### 2.7 图与表（`references/figure-workflow.md`, `figure-spec.md`, `style-presets.md`）

| 模板 | 行号 | 槽位 |
|---|---|---|
| Figure Plan 八项 | `figure-workflow.md:9-18` | Class / Role / Message / Entities / Relationships / Layout / Backend / Fallback |
| 生成图必需输出 | :98-104 | 实际资产 + prompt/spec 源（必需）；TikZ/SVG 参考（可选）；deterministic overlay（可选）；LaTeX include 路径或 blocker |
| 图像 prompt 七要素 | :106-114 | venue+目的 / 视觉风格+精确调色板 / 画布比例 / 布局区域 / 精确标签 / 箭头与反馈路径 / 约束（无 logo、水印、未支持术语、小字） |
| **Diagram Prompt 六段结构** | :118-129 | 1 Framing（venue/章节/读者 takeaway/语气）2 Visual style 3 Color palette（hex+语义）4 Layout（区域/面板/阅读顺序）5 Connections（每条箭头、反馈、虚线失败路径、标签）6 Constraints（不幻觉标签、小字、logo、假数字、无支持主张） |
| 数据图标准 + matplotlib 默认 | :62-90 | 源数据脚本化；PDF/SVG 优先；≥300 DPI；Okabe-Ito 色板；禁 3D/饼图/双轴；metric direction |
| LaTeX Include 模式 | :135-157 | 直接 `\includegraphics`；`\IfFileExists` 回退到 TikZ；结果图 PDF→PNG 回退 |
| Caption 四步 | :169-183 | 展示什么 → 如何读（metric direction）→ takeaway → 必要 caveat；例 `:180-183` |
| Figure Spec YAML | `figure-spec.md:9-34` | figure_id / filename / figure_class / role / message / core_conclusion / evidence_hierarchy{hero,supporting} / entities[] / relationships[] / layout / backend / source / backup / caption_takeaway / evidence_status / reviewer_risk |
| Role taxonomy + 数据图形状表 | :36-47,69-81 | overview/method-detail/... 与 line/bar/heatmap/table 的对应 |
| Caption Contract | :101-103 | `Figure X: [What is shown]. [How to read it]. [Takeaway]. [Caveat if needed].` |
| 视觉预设 | `style-presets.md:14-23,84-135` | 各 venue 单栏/双栏宽度；数据图字体/导出设置；4 种 diagram style；箭头语义表（data/control/feedback/error/optional/comparison） |
| Venue LaTeX 片段 | `venue-templates.md:28-77` | NeurIPS checklist skeleton / Limitations / Reproducibility / Compute / Ethics / LLM Usage 模板 |

### 2.8 审稿与投稿（`references/reviewer-guidelines.md`, `reviewer-self-review.md`, `submission-packaging.md`）

| 模板 | 行号 | 槽位 |
|---|---|---|
| Evidence Notes | `reviewer-guidelines.md:25-34` | `ID / affected claim`, `Location`, `Already supplied / checked`, `Observation and evidence`, `Interpretation or uncertainty: fact/inference/needs clarification`, `Impact and severity`, `Question for authors`, `Minimum evidence that would resolve` |
| Concise Review | :50-59 | Review context / Summary / Strengths / Major concerns / Questions / Recommendation / Confidence and coverage / Ethics |
| 六维评审表 | :74-84 | Originality / Quality-soundness / Clarity / Significance / Reproducibility / Ethics-responsibility（每维 reviewer asks + author action） |
| Review-ready 分节检查 | :133-165 | Abstract / Introduction / Method / Experiments / Limitations-Ethics-Reproducibility |
| 三镜头自审 | `reviewer-self-review.md:16-26` | Methodology / Domain and positioning / Statistics and evidence；:26 禁止一次调用边审边改 |
| 六维评分 | :28-40 | `pass / needs revision / needs evidence / not assessed` × Contribution / Writing clarity / Experimental strength / Evaluation completeness / Method soundness / Responsibility |
| Reviewer Analysis 报告 | :62-94 | Executive Summary / Strengths / Major Risks 表 / Claim and Evidence Gaps 表 / Author Decisions Needed / Revision Priorities |
| Rejection-Risk Audit | :98-109 | thesis / 决策相关反对意见 / 对应证据 / 剩余缺口 / 计划修复 / 严重度；高风险未解决阻止 submission-ready |
| Build 命令 | `submission-packaging.md:19-30` | pdflatex→bibtex→pdflatex×2 或 `latexmk -pdf -interaction=nonstopmode main.tex` |
| Static Checks | :36-41 | `check_citations.py`, `check_todos.py`, `parse_build_log.py`, `camera_ready_check.py` |
| Submission Readiness | :81-95 | Target venue / Template / Main tex / Build status / Citation check / TODO check / Checklist status / Limitations / Reproducibility / Figure-source / Known reviewer risks / Known packaging risks / Next actions |
| Final Response Contract | :108-116 | Files changed / Checks run / Build status / Remaining blockers / Commit or push hash |

---

## 3. `research_quality_gate.py` 逐规则解析

### 3.1 常量与正则（:30-55）

| 常量 | 行号 | 值/语义 |
|---|---|---|
| `SCHEMA_VERSION` | :30 | `ai-research-writing/paper-state-v1` |
| `VALID_MODES` | :31 | `draft / full-paper / submission / skill`（`skill` 只允许通过模式推断或显式 `--mode skill`） |
| `STATE_FIELDS` | :32 | 9 个允许字段；多余字段即 failure |
| `BUILD_FIELDS` | :33-36 | status/command/pdf/external_inputs/input_sha256/pdf_sha256/attestation/exit_code/log/log_sha256/tool_version/ran_at |
| `EXTERNAL_INPUT_FIELDS` | :37 | name/source/archive_sha256 |
| `TERMINAL_STAGES` | :38 | `complete`, `submission-ready` |
| `UNRESOLVED_RE` | :39-43 | TODO/TBD/FIXME/XXX/PLACEHOLDER/CITATION NEEDED/FILL LATER + `待回填/待替换/待补充/公式占位/算法占位` |
| `PROCESS_RE` | :44-49 | `write naturally/avoid ai/replace later/user should/.../figure position` + `写作要求/修改要求/请用户/用户替换/回填模板/讨论提示/图位/表位/实验目的` |
| `STALE_PROGRESS_RE` | :50-53 | `verification planned/pending`、`to be recorded`、`not yet run`、`待验证/待运行/尚未验证` |
| `OPEN_STATUS_RE` | :54 | `needs evidence/pending/unknown/unverified/placeholder/待核验/待补充` |
| `LOCAL_MD_LINK_RE` | :55 | Markdown 本地链接（排除 http(s)、mailto、纯 `#` 锚点） |

`Issue(level, message)`（:59-62）只有 `warning` 与 `failure` 两级；输出按 warning→failure 分组打印，退出码只看 failure 数量（:461-472）。

### 3.2 通用工具函数

- `add()`（:64-65）：追加 issue。
- `require_nonempty()`（:68-80）：路径必须落在项目根内（`relative_to` 逃逸即报 `Required artifact escapes project root`）、必须是文件、必须非空。
- `markdown_rows()`（:82-99）：见证据链文档 §2.6；**找不到表会抛 `ContractError`**，由 `main()` 转成 failure。
- `state_string()`（:167-173）：要求字符串且 `strip()` 后非空；用于 `stage/target_venue/main_tex/bibliography`。
- `external_build_records()`（:175-203）：校验 `build.external_inputs` 是列表、元素是 dict、字段集恰好三个、name/source 非空、sha256 为 64 位小写 hex；返回稳定序列化字符串供摘要使用。

### 3.3 `validate_skill()` 全部检查（:102-164）

| # | 检查 | 行号 | 失败文案（节选） |
|---|---|---|---|
| 1 | `SKILL.md` 存在且非空 | :103 | `Required artifact is missing/empty: SKILL.md` |
| 2 | 以 `---\n...\n---\n` YAML frontmatter 开头 | :106-109 | `must start with YAML frontmatter` |
| 3 | frontmatter 顶层键**按顺序恰好** `["name","description"]` | :110-117 | `must contain only name and description` |
| 4 | `SKILL.md` 中反引号包裹的 `references/...` 或 `scripts/...` 路径都存在（含 `*<>` 的通配写法跳过） | :119-123 | `references a missing local resource` |
| 5 | `README.md`、`README.zh-CN.md`、`references/README.md` 中本地 Markdown 链接可解析 | :125-133 | `Broken local Markdown link in ...` |
| 6 | `scripts/*.py` 全部可 `compile()` | :135-140 | `Python syntax error in ...:lineno: msg` |
| 7 | `templates/manifest.json` 存在 | :142-143 | `templates/manifest.json is required` |
| 8 | `templates` 对象非空 | :144-149 | `requires a non-empty templates object` |
| 9 | 每个模板项含非空字符串 `venue/official_page/license_status/notes` | :151-154 | `requires venue` 等 |
| 10 | `archive_url` 与 `archive_sha256` 必须成对存在 | :155-158 | `must declare archive_url and archive_sha256 together` |
| 11 | 若声明哈希，必须是 64 位小写 hex | :159-161 | `invalid archive_sha256` |
| 12 | `templates/` 下不允许任何 vendored 子目录 | :163-164 | `Vendored template directories are not allowed` |

注意第 5 项**只检查这三个 README**，因此新增 `docs/*.md` 里的链接不进入闸门；这解释了为什么本套文档可以放心新增而不会影响 skill 模式检查（但人工仍应保证链接有效）。

### 3.4 `validate_paper()` 完整判定顺序（:267-436）

这是一棵“早返回”的决策树；看懂顺序才能理解为什么某些检查“没报错”。

| 阶段 | 判定 | 行号 | 结果 |
|---|---|---|---|
| A | 读 `paper_state.json`；JSON 解析失败直接 failure 并 return | :268-276 | 后续检查不执行 |
| B | `mode = --mode or state.mode or "draft"`；不在 `VALID_MODES - {skill}` → `Unknown paper mode` 并 return | :277-281 | 早停 |
| C | `--mode` 与 state.mode 同时存在且不一致 → failure（但不 return） | :282-283 | 继续检查 |
| D | `full-paper/submission` 无 state 文件 → failure 并 return | :285-287 | 早停 |
| E | 无 state（仅 draft）→ warning；要求根目录下**恰好一个** `main.tex`，否则 failure；只跑 `validate_manuscript()` 后 return | :288-295 | 不查数字/引用/构建 |
| F | `schema_version` 不匹配 → failure | :297-298 | 继续 |
| G | state 顶层未知字段 → failure | :299-301 | 继续 |
| H | state.mode 不是三值之一 → failure | :302-303 | 继续 |
| I | `stage/target_venue/main_tex/bibliography` 必须非空字符串；后两者空则 return | :304-310 | `main_tex/bib` 缺失时早停 |
| J | `main_tex`、`bibliography` 必须存在非空且在根内 | :312-313 | 继续 |
| K | `terminal = stage ∈ TERMINAL_STAGES`；`blockers` 必须字符串数组；终态非空即 failure | :314-319 | 继续 |
| L | `required_artifacts` 必须非空字符串数组 | :321-324 | 继续 |
| M | 基线工件：story/claim map/citation verification/build_check/plan-progress；submission 再加 `submission_readiness.md`；与 required 合并去重后逐个 `require_nonempty` | :325-329 | 缺失即 failure |
| N | `plan/task-packets/*.md` 至少一个 | :330-332 | 缺失即 failure |
| O | 终态时 `plan/progress.md` 不得命中 `STALE_PROGRESS_RE` | :334-336 | 命中即 failure |
| P | claim map 存在 → `validate_claim_map()`（终态才检查状态列） | :338-340 | 见 §3.5 |
| Q | 手稿 → `validate_manuscript()`：`collect_tex_text` + `resolve_graphics`；逐拼接行扫 `UNRESOLVED_RE` 与 `PROCESS_RE` | :342-343,253-265 | 任一命中 failure；缺图/缺 input 也 failure |
| R | 若存在 `numeric_evidence.json` → `verify_numeric_evidence()`；findings 前 8 条进消息，其余计 `plus N more` | :344-356 | 未登记数字即 failure |
| S | citation key 收集；BibTeX 重复 key / 缺失 key 检查 | :357-366 | failure |
| T | 终态且 verification 表存在 → `validate_verification_table()`；若返回 scholarly key 集合非空，则要求 requests+lock 存在并 `validate_lock()` | :367-383 | 见证据链文档 §2.6 |
| U | build 检查（见证据链文档 §4.3） | :385-436 | 见上 |

**易误解点**：

- `target_venue` 被校验但随后 `del target_venue`（:305,308）——它必须非空，但不参与后续分支。
- “5 道闸门”在 `docs/README.md:78-92` 里是**概念性**表述；代码里的检查比 5 项多得多，且一些概念闸门（Story 质量、Literature 定位）**没有**对应的机器检查，只能靠 agent/人。
- `draft` 模式没有 state 时只检查一份唯一的 `main.tex`，不会检查构建摘要与引用锁；这不是“通过”，而是“尚不适用”。

### 3.5 三个专用校验器

- `validate_verification_table()`（:205-233）：表头需 `key/status`；`verified/software-doc/repository-verified` 计入 verified，只有 `verified` 计入 scholarly；每个 cited key 都要 verified；任何其余状态（含空）都报 unresolved。
- `validate_claim_map()`（:236-250）：仅终态；表头需 `status`；状态命中 `OPEN_STATUS_RE` 的每一行都报错，错误文案里直接带原状态字符串。
- `validate_manuscript()`（:253-265）：先 `collect_tex_text()` 与 `resolve_graphics()`，任一 `ContractError` → 一条 failure 并 return；再按**拼接后行号**逐行扫两个正则，报错截取该行前 140 字符。

### 3.6 模式选择与退出码（:439-476）

```text
root 不存在                         → print + return 2
mode = --mode or ("skill" if 根目录有 SKILL.md 且 references/ 是目录 else None)
mode == skill → validate_skill()
否则         → validate_paper(root, mode)
输出：Warnings: ... / Failures: ...
failures 非空 → "Research quality gate failed with N failure(s)." + return 1
否则          → "Research quality gate passed." + return 0
```

| 退出码 | 含义 | 常见触发 |
|---|---|---|
| 0 | 通过（可有 warning） | 合法契约 |
| 1 | 检查失败 | 缺工件、未核验引用、未登记数字、终态构建不合规 |
| 2 | 无法执行（目录不存在或内部 `ContractError`） | JSON 坏、编码坏、输入图缺文件、无表格 |

### 3.7 与总闸门配套的旁路检查器

| 脚本 | 关键断言 | 退出码 |
|---|---|---|
| `check_numeric_evidence.py`（:310-345） | 重算注册表 + 扫证据敏感区 | 1=未登记数字；2=契约错误 |
| `check_citations.py`（:13-43） | cite key ⊆ bib key 且 bib 无重复；unused 仅提示 | 1=缺失/重复；2=输入错误 |
| `check_citation_lock.py`（:14-43） | 离线 `validate_lock()`，默认 180 天 | 1=锁失败；2=契约错误 |
| `check_research_handoff.py`（:91-121） | handoff 结构与工件；`--require-unblocked` 才把 blocker 变失败 | 1=blocker（当要求）；2=契约错误 |
| `check_todos.py`（:71-115） | 跳过 10 个目录 / 14 种后缀；`MARKER_RE`（:42-45）；`--allow` 正则白名单 | 1=发现 marker；2=坏输入/坏正则 |
| `parse_build_log.py`（:33-63） | 7 类模式去重计数；errors/emergency/undefined cites/undefined refs/missing files 命中即 1 | 1=严重问题；0=仅 overfull/underfull |
| `camera_ready_check.py`（:21-52） | 7 项静态检查（marker/Limitations/Bibliography/Checklist/作者元数据/Acknowledgments/LLM disclosure） | 1=任一 fail；2=输入错误 |
| `fetch_template.py`（:36-106） | 模板 id 白名单、输出目录不存在、下载 SHA-256 比对、zip 路径穿越/symlink 防护 | 2=任一步失败 |
| `record_build.py`（:17-132） | 命令无 shell 操作符、PDF 不早于输入、哈希一致 | 2=失败；0=写回 state |

---

## 4. 修改 Prompt 规范或闸门规则时的不变量

1. **新增/改名 `references/*.md`**：同步 `SKILL.md` 路由表（`SKILL.md:15-27`）、`references/README.md` 加载地图、`docs/modules_reference.md` 文件表。
2. **新增脚本**：登记 `scripts/README.md`、`.github/workflows/quality.yml`、本文 §3.7、`docs/modules_reference.md:90-130`。
3. **改正则/终态条件**：必须补 `tests/test_quality_tools.py` 对应类（`QualityGateTests:142-279` 等），并确认 fail-closed 未退化。
4. **改 JSON schema**：同步 schema 文件、脚本常量、在线验证器、总闸门、文档示例五处；只改其中一处会让“摘要链”不一致。
5. **不要把过程指令写进手稿**：模板里的“写作要求/图位/请用户”只能留在 plan/spec/review 文件；`PROCESS_RE` 会在拼接后的手稿里抓它们。
6. **不要为了过闸门而删检查**：缺口正确动作是“补证据 / 弱化主张 / 记 blocker”（`SKILL.md:48-56`）。

---

## 5. 阅读路径建议

- 要写故事：§2.1 → §2.2 → `references/paper-story.md`。
- 要写章节：§2.2 → §2.3 → `references/section-writing.md`。
- 要做图：§2.7 → `references/figure-workflow.md` + `figure-spec.md`。
- 要交付投稿包：§2.8 → `references/submission-packaging.md` → §3 全部闸门。
- 要改检查器：§3 全部 → `tests/test_quality_tools.py` → [`evidence_chain_deep_dive.md`](evidence_chain_deep_dive.md)。
