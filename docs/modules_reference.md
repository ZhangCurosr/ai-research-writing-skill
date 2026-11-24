# 模块参考：技能文件、Prompt 规范、检查脚本

本文是维护/扩展时的**符号与契约清单**。所有脚本均为 Python 3 stdlib-only（`scripts/README.md:3`），通过 `argparse` 暴露 CLI。

## 1. 入口与文件清单

### 1.1 入口文件

| 文件 | 关键内容 | 修改时联动 |
|---|---|---|
| `SKILL.md` | YAML frontmatter 仅 `name`/`description`（:1-4）；Mandate（:8-13）；Route 表（:15-27）；Execution Contract（:30-37）；Gates（:39-49）；Evidence Boundaries（:51-56） | `research_quality_gate.py:102-132` 会校验 frontmatter 和本地资源路径 |
| `README.md` / `README.zh-CN.md` | 用户向说明、安装、命令摘要 | `validate_skill()` 检查其中的本地 Markdown 链接（:124-132） |
| `.github/workflows/quality.yml` | CI：Python 3.10–3.13；unittest、py_compile、skill gate、example gate、citation/TODO 检查（:10-41） | 新增脚本后在此登记 |

### 1.2 `references/` 参考文件

| 文件 | 用途（依据） | 关键规范 |
|---|---|---|
| `README.md` | 路由地图；加载规则；先 map 再开多文件（:3, :44-50） | layers 而非编号 |
| `workflow.md` | 完整论文状态机 8 阶段（:5-69） | 两遍草稿顺序（:24-44） |
| `artifacts.md` | `paper_state.json` 契约（:5-37）、核心工件（:39-52）、最小集（:56-67） | 产物写在目标项目 |
| `task-management.md` | task packet（:5-19）、progress（:21-34）、completion（:36-44） | 终态禁止 “待验证” |
| `research-handoff.md` | handoff JSON（:9-37）、所有权边界（:39-43）、验收（:45-51） | 上游只导证据 |
| `paper-story.md` | 12 项 story（:5-21）、repo mining（:54-66）、四级 claim（:150-157）、模板（:159-195） | thesis/contribution/naming test |
| `abstract-introduction.md` | 高风险章节预写问题、Abstract 模板 | 每条强主张对应 claim map 行（:5-7） |
| `section-writing.md` | Method/Experiments/Related Work/Limitations 写作循环（:5-14） | agent 系统需分清推理与确定性执行（:32） |
| `literature-review.md` | 语料工件（:9-19）、inventory 模板（:21-35） | 不以记忆定位工作 |
| `citation-workflow.md` | 三维验证（:3-9）、来源路由（:11-21）、六步法（:36-45）、七种 relation（:47-59）、终态状态（:71-77） | OpenAlex 只作 cross-check（:30-34） |
| `citation-checklist.md` | 最终审计（:5-20）与最终命令（:35-44） | 强主张不得依赖 weak/metadata-only |
| `numeric-evidence.md` | v2 示例、selector/aggregate、v1 迁移（:3-36） | 未登记数字即失败 |
| `figure-workflow.md` | plan（:9-22）、后端二分（:24-45）、数据图标准（:62-90）、六段 prompt（:118-129）、LaTeX include（:131-157） | 生成图不是证据 |
| `figure-spec.md` | YAML spec（:9-34）、role taxonomy（:36-47）、caption contract（:97-104） | 每图一个 message |
| `figure-pattern-atlas.md` | Class 1 图的模式路由；不 vendored 上游图/脚本（:3-10） | 需要具体图形时再加载 |
| `style-presets.md` | 字号、色彩、venue 尺寸（:14-23） | 7pt 以下禁止 |
| `reviewer-guidelines.md` | evidence-first 六步（:12-19）、evidence notes（:21-36）、六维（:74-84）、venue 倾向（:108-132） | 模拟审稿需标注 |
| `reviewer-self-review.md` | 三镜头（:16-26）、六维（:28-40）、rejection-risk（:98-109）、claim/experiment 审计（:130-163） | 输出作者可决策的分析 |
| `venue-profiles.md` / `venue-templates.md` | venue 工作流（:7-15）、checklist 骨架（:5-32） | 活跃投稿需核对官方规则 |
| `submission-packaging.md` | build 工作流（:9-31）、静态检查（:32-41）、hygiene（:52-61）、readiness（:77-95）、最终回复（:108-116） | PDF 不等于可投稿包 |
| `example-bank.md` / `examples/index.md` | 紧凑模式库与具体样例索引 | 选定 pattern 后再打开具体样例 |

## 2. Prompt 规范（本技能是“规范文本”，不是模板引擎）

**重要事实**：仓库中没有 Python prompt 渲染模块。所谓 “Prompt 规范” 是写在 `references/*.md` 里的结构化指令；agent 读取后自行生成文本。维护时应保持以下**语义契约**一致。

### 2.1 Claim 强度（`references/paper-story.md:150-157`）

| 级别 | 含义 | 允许位置 |
|---|---|---|
| strong | 直接由实验/形式分析支持 | Abstract / Introduction |
| careful | 部分支持、混合或依赖假设 | 限定性措辞 |
| aspirational | 动机或未来方向 | 不要放进结果性文字 |
| forbidden | 与证据矛盾或无支持 | 必须删除 |

关系标签（引用）：`direct/background/contrast/software/partial/weak/metadata-only`（`citation-workflow.md:47-59`）。强技术/比较型主张需要 `direct`、`contrast` 或 `software`。

### 2.2 Thesis / Contribution / Naming Tests

- Thesis 模板（`paper-story.md:70-74`）：`We introduce [artifact] for [task] that enables [capability] by [insight], and we show [evidence].` 若退化成模块列表，故事尚未收敛。
- Contribution 四问：什么新、为何不平凡、技术优势、证据（:78-87）。
- Naming：避免泛化名、避免暗示统一模型、贴近任务/机制/表示（:89-97）。

### 2.3 图表 Prompt 六段结构（`figure-workflow.md:118-129`）

1. Framing：venue、章节、读者 takeaway、语气；
2. Visual style：视觉风格；
3. Color palette：精确 hex 与语义；
4. Layout：区域、面板、阅读顺序；
5. Connections：每条箭头、反馈路径、虚线失败路径；
6. Constraints：不得幻觉标签、小字、logo、假数字或不支持的主张。

Caption 四步（:169-183）：说明内容 → 如何读（metric direction）→ takeaway → 必要 caveat。

### 2.4 审稿证据笔记（`reviewer-guidelines.md:21-36`）

每条 concern 一个 ID：`ID / affected claim`, `Location`, `Already supplied / checked`, `Observation and evidence`, `Interpretation or uncertainty (fact/inference/needs clarification)`, `Impact and severity`, `Question for authors`, `Minimum evidence to resolve`。

## 3. JSON 契约（schema 文件与代码双重校验）

| Schema | 常量/校验器 | 关键要求 |
|---|---|---|
| `references/paper-state.schema.json` | `research_quality_gate.py:30-37,267-436` | mode ∈ draft/full-paper/submission；终态要求 build passed；未知字段禁止 |
| `references/numeric-evidence.schema.json` | `check_numeric_evidence.py:19-25,203-258` | `numeric-evidence-v2`；selector kind；aggregate 枚举；recompute 一致 |
| `references/research-handoff.schema.json` | `check_research_handoff.py:14-88` | 基础三工件；quantitative 时必需 experiment_inventory + numeric_evidence |
| `references/citation-requests.schema.json` | `citation_lock.py:15-19,30-88` | 唯一 key；doi/arxiv 前缀；非空 claim_support |
| `references/citation-lock.schema.json` | `citation_lock.py:91-178` | 请求/元数据/支持/交叉核验摘要；终态 verified；checked_at 不过期 |
| `templates/manifest.json` | `research_quality_gate.py:140-164`, `fetch_template.py:43-65` | venue/official_page/license_status/notes；archive_url 与 sha 成对；无 vendored 目录 |

## 4. 脚本参考（CLI、函数、退出码）

### 4.1 共享库 `scripts/paper_contract.py`（无 CLI）

| 名称 | 作用 |
|---|---|
| `ContractError` | 声明输入无法读取/解析时抛出（:23-24） |
| `read_utf8()` / `strip_comments()` | UTF-8 严格读取；去掉 TeX `%` 注释（:27-51） |
| `resolve_tex()` / `collect_tex_files()` | 递归解析 `\input/\include`，缺文件即报错（:54-77） |
| `collect_tex_text()` | 拼接全部 TeX 文本（:80-81） |
| `resolve_graphics()` | 解析 `\includegraphics`，尝试后缀 `""`, `.pdf`, `.png`, `.jpg`, `.jpeg`, `.eps`, `.svg`（:13, :84-95） |
| `collect_citation_keys()` / `collect_bib_keys()` | 收集 citation key 与 bib 条目 key（:98-109） |
| `load_json_object()` | 严格 JSON 对象解析（:112-119） |
| `paper_input_files()` | TeX + 图形 + bibliography 去重排序（:122-128） |
| `digest_files()` / `digest_file()` | 相对路径+字节的 SHA-256（:131-160） |

正则常量：`INPUT_RE`, `GRAPHICS_RE`, `CITE_RE`, `BIB_RE`, `GRAPHIC_SUFFIXES`（:12-20）。

### 4.2 检查器

| 脚本 | CLI | 关键函数 | 输出/退出码 |
|---|---|---|---|
| `extract_claims.py` | `FILES... [--sections abstract,introduction] [--all-sentences]` | `strip_comments`, `collect_macros`, `apply_macros`, `latex_to_text`, `extract_target_text`, `split_sentences`, `escape_md`, `main`（:34-132） | Markdown claim 表到 stdout；无匹配打印提示；0 |
| `check_citations.py` | `TEX BIB` | `main`（:13-43） | 重复/缺失 key → 1；正常 0；输入错误 2 |
| `check_numeric_evidence.py` | `PROJECT [--main PATH] [--registry numeric_evidence.json]` | `EvidenceValue.matches`, `NumericFinding`, `_source_path`, `_json_pointer`, `_matches_where`, `_selected_values`, `_aggregate`, `load_registry`, `_mask_ranges`, `find_unverified_numbers`, `verify_project`, `main`（:51-345） | 未登记数字 → 1；契约错误 2；通过 0 |
| `check_research_handoff.py` | `PROJECT [--handoff research_handoff.json] [--require-unblocked]` | `_artifact_path`, `validate_handoff`, `main`（:36-121） | blocker + require-unblocked → 1；契约错误 2；通过 0 |
| `check_citation_lock.py` | `PROJECT [--main PATH] [--max-age-days 180]` | `main`（:14-47） | 锁失败 → 1；契约错误 2；通过 0 |
| `verify_citations.py` | `PROJECT [--mailto EMAIL] [--openalex-api-key KEY] [--timeout 20.0]` | `ProviderFailure`, `fetch`, `_year_from_crossref`, `crossref`, `datacite`, `arxiv`, `openalex`, `comparable`, `main`（:25-229） | 写 `citation_lock.json`；任一非 verified → 1 |
| `check_todos.py` | `PATHS... [--allow REGEX ...]` | `iter_files`, `read_text`, `main`（:48-119） | 发现 marker → 1；缺输入/坏正则/读取错误 2；通过 0 |
| `parse_build_log.py` | `LOG_FILE [--max-lines 20]` | `unique`, `main`（:22-63） | errors/emergency/undefined cites/refs/missing files 命中 → 1 |
| `camera_ready_check.py` | `MAIN_TEX [--checklist FILE] [--llm-disclosure FILE]` | `has_section`, `main`（:17-56） | 7 项静态检查任一 fail → 1；输入错误 2 |
| `fetch_template.py` | `[TEMPLATE_ID] [--output DIR] [--list]` | `safe_members`, `main`（:22-110） | 哈希不符/路径不安全/无 pinned 包 → 2；成功 0 |
| `record_build.py` | `PROJECT [--run] [--timeout 300.0]` | `main`（:17-132） | 构建失败或哈希矛盾 → 2；成功写回 state 并 0 |
| `research_quality_gate.py` | `PROJECT [--mode draft|full-paper|submission|skill]` | `validate_skill`, `validate_paper`, `validate_manuscript`, `validate_claim_map`, `validate_verification_table`, `markdown_rows`, `external_build_records`, `main`（:58-476） | failures → 1；目录不存在 2；pass 0 |
| `make_latex_table.py` | `CSV [--output FILE] [--caption] [--label] [--columns] [--precision] [--bold none\|max\|min] [--bold-columns]` | `latex_escape`, `to_float`, `format_value`, `choose_bold_values`, `build_table`, `main`（:23-124） | 表格（stdout 或文件）；无表头直接退出 |

### 4.3 `citation_lock.py` 共享常量与函数

`REQUEST_SCHEMA`, `LOCK_SCHEMA`, `REQUEST_FIELDS`, `ENTRY_FIELDS`, `SUPPORT_FIELDS`, `RELATIONS`, `SUPPORT_TERMINAL`, `LOCK_TERMINAL`（:15-22）；`canonical_digest()`, `load_requests()`, `validate_lock()`（:25-178）。修改这些常量意味着同时改变在线验证、离线检查与总闸门行为。

### 4.4 `citation_verification.md` 的隐式契约

虽然是 Markdown，但被 `markdown_rows()` 解析（`research_quality_gate.py:82-99`）：第一列分隔线必须是 `|---|` 风格，列名小写后包含 `key` 与 `status`，行单元格数必须与表头一致。改表头或加列前先看 `validate_verification_table()`（:205-233）。

## 5. 测试清单（`tests/test_quality_tools.py`）

| 测试类 | 覆盖 |
|---|---|
| `QualityGateTests`（:142-279） | 合法 full-paper 契约、终态 progress、缺失 TeX input、引用未核验、源码变更使 build 摘要失效、未知 state 字段、PDF 早于源、submission-ready 需 executed attestation、构建执行记录、不可读文本、numeric registry 强制 |
| `NumericEvidenceTests`（:281-402） | raw/percent、未登记数字、缺失源、JSON pointer、CSV mean、v1 迁移失败 |
| `CitationLockTests`（:404-484） | request 摘要陈旧、非终态 support、Crossref 归一化、网络错误写成非终态锁 |
| `ResearchHandoffTests`（:486-570） | quantitative handoff、缺 numeric_evidence、`--require-unblocked` |
| `CameraReadyTests`（:572-591） | 仅文字提到 checklist 不算 checklist 工件 |
| `TemplateAuditTests`（:593-627） | schema 是合法 JSON、9 个 venue 且无 vendored 目录、归档路径穿越被拒 |

`PaperProject`（:37-140）是测试内构造临时论文项目的 helper。新增脚本契约时优先考虑在此扩展，而不是只加 smoke test。

## 6. 维护检查清单

- [ ] 新增/改名参考文件 → 更新 `SKILL.md` 路由表 + `references/README.md` 地图 + 本文表格。
- [ ] 新增脚本 → 更新 `scripts/README.md`、本文 §4、`.github/workflows/quality.yml`。
- [ ] 改 schema → 同步 JSON schema、脚本常量、测试、文档示例。
- [ ] 改闸门正则/终态条件 → 补对应的 `QualityGateTests` 用例，确认 fail-closed 未退化。
- [ ] 运行 `python3 -m unittest discover -s tests -v && python3 scripts/research_quality_gate.py . --mode skill`。
