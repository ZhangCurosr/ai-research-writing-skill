# 排错指南：证据链、LaTeX 与单步写作阻断

> 事实来源：源码与 `references/` 规范；`路径:行号` 为本检出版本锚点。
>
> 使用原则：**先定位失败的是哪一条链，再做最小修复**。绝不要通过删检查、改状态字符串、放宽 tolerance 或隐藏负结果来“让闸门变绿”。所有检查器都是 fail-closed：读不到输入就报错，而不是跳过（`paper_contract.py:27-51`）。
>
> 相关深度说明：[`evidence_chain_deep_dive.md`](evidence_chain_deep_dive.md)（算法）、[`prompts_and_gate_reference.md`](prompts_and_gate_reference.md)（规则全集）。

## 0. 三分钟定位流程

```bash
# 1) 确认当前模式与失败条目（根目录执行会推断为 skill 模式！）
python3 scripts/research_quality_gate.py /path/to/paper --mode full-paper

# 2) 若是 skill 包自身：显式 skill 模式
python3 scripts/research_quality_gate.py . --mode skill

# 3) 按失败信息分流，逐条单独复现
python3 scripts/check_numeric_evidence.py /path/to/paper
python3 scripts/check_citation_lock.py /path/to/paper
python3 scripts/check_citations.py paper/main.tex paper/references.bib
python3 scripts/parse_build_log.py paper/main.log

# 4) 复现后先读 paper_state.json 的 mode/stage/build.*，确认是否终态
cat paper_state.json
```

| 退出码 | 含义 | 第一反应 |
|---|---|---|
| 0 | 通过（可能有 warning） | 记录实际命令与输出进 `build_check.md` / `plan/progress.md` |
| 1 | 检查失败 | 按本文 §1–§3 找对应条目 |
| 2 | 无法执行（目录不存在 / JSON、编码、输入图坏） | 先修输入，不要改内容 |

`research_quality_gate.py:461-472` 的输出固定分 `Warnings:` / `Failures:` 两段，最后一行给出失败数；失败数 >0 返回 1。

---

## 1. 证据链断裂

### 1.1 数字证据失败（`Unverified manuscript numbers`）

**典型输出**（`check_numeric_evidence.py:336-340` / `research_quality_gate.py:354`）：

```text
Unverified numbers in evidence-sensitive manuscript regions:
  collected line 412 [Results]: 91.2 | We achieves 91.2% accuracy ...
Research quality gate failed with 1 failure(s).
```

排查顺序：

1. **确认“collected line”不是物理行号**。`verify_project()` 用 `collect_tex_text()`（`check_numeric_evidence.py:305-308`）把 `\input/\include` 按 DFS 顺序拼接（`paper_contract.py:59-81`）。先打印拼接顺序：
   ```bash
   python3 -c "from scripts.paper_contract import collect_tex_files; from pathlib import Path; print('\n'.join(map(str, collect_tex_files(Path('paper/main.tex')))))"
   ```
   （若在 `scripts/` 目录内执行，把 import 改成 `from paper_contract import ...`。）
2. **确认这个数字是否真的需要登记**。脚本只检查两类区域（`check_numeric_evidence.py:290-292`）：表格环境内，或当前章节标题含 `result/experiment/evaluation/ablation/analysis/finding/benchmark`（:26-34）。`Discussion` 段里的数字不会被这条规则扫到——但这不构成“可以写未验证数字”的理由。
3. **检查表示形式**。`representations:["raw"]` 只登记 `0.912`；正文写 `91.2%` 时需要在注册表加 `"percent"`（:250-257）。
4. **检查 aggregate 语义**。`identity` 要求恰好一个选中值（:177-186）；多行 CSV 忘了改 `mean` 时报错与“数字写错”看起来完全不同。
5. **检查 selector**。四种 kind 的必填字段与错误文案见 [`evidence_chain_deep_dive.md` §1.3](evidence_chain_deep_dive.md)。

| 报错 | 根因 | 修复 |
|---|---|---|
| `Numeric evidence value X does not match computed source value Y` | 注册值与重算值不一致 | 以源数据为准改 `value` 或改 `where/aggregate`；不要改原始日志迁就正文 |
| `JSON Pointer does not resolve` | pointer 未以 `/` 开头、数组越界、键名错 | 用 `python3 -m json.tool` 打开源文件核对层级；`~1`/`~0` 转义 |
| `column 'accuracy' is missing from ...` | 表头名不符或分隔符错（csv vs tsv） | 打印 `reader.fieldnames`；确认 `kind` |
| `Invalid JSONL at source:12` | 原始日志有非 JSON 行（进度条、warning） | 清洗成纯 JSONL，且每行是对象 |
| `selector produced a boolean/non-number/non-finite value` | 选中列含 `true`/字符串/NaN | 修 selector 或源数据；脚本不做类型猜测 |
| `numeric_evidence.json contains unknown fields` | 自定义字段 | 只保留 `schema_version` 与 `entries` 白名单字段（:19-25） |
| `Numeric evidence source escapes project root` | 用了 `../` 或绝对路径 | 把数据复制进论文项目并登记项目内相对路径 |

**不要做的事**：把数字从 `Results` 移到 `Discussion` 以绕过扫描；把 tolerance 改成 `1`；把手稿数字改成“约 90%”以免被 NUMBER_RE 命中。这些都在规避证据，而不是补证据。

### 1.2 引用链断裂

#### (a) 总闸门阶段

| 报错（来源） | 含义 | 修复 |
|---|---|---|
| `Cited keys lack a verified citation record: X`（`research_quality_gate.py:228-229`） | 手稿 `\cite{X}` 在 verification 表里没有终态记录 | 在 `citation_verification.md` 加 `Key=X, Status=verified/software-doc/repository-verified` 行；或删掉未被支持的引用 |
| `Citation verification has unresolved records: X (metadata-only)`（:224-226） | 表里留了非终态状态 | 补元数据与逐句支持后改成终态；终态不允许 `metadata-only/conflict/needs-access/placeholder` |
| `citation_verification.md table requires Key and Status columns`（:210-214） | 表头不符 | 表头必须含 `key` 与 `status`（解析器会转小写，见 :82-99）；样例见 `examples/paper-about-ai-research-writing-skill/citation_verification.md:3-9` |
| `No Markdown table found in required artifact`（:99） | 没有 `|---|` 分隔行 | 按 `| Key | ... | Status |` + `|---|---|` 格式重建表 |
| `Missing BibTeX keys: X` / `Duplicate BibTeX keys: X`（:363-366） | `\cite{X}` 无 bib 条目 / bib 中重复 | 从主来源重新导出 BibTeX；保持稳定小写 key（`citation-workflow.md:88-94`） |
| `Terminal scholarly citations require citation_requests.json and citation_lock.json`（:373-378） | 有 `verified` 学术引用但缺请求/锁 | 补两份 JSON 并重跑 `verify_citations.py` |
| `Citation lock is stale for citation_requests.json`（`citation_lock.py:103-104`） | 改了请求未重新核验 | 重跑 `verify_citations.py`；不要手改 `request_sha256` |
| `citation lock is older than 180 days`（:157-159） | 锁过期（默认 `check_citation_lock.py:18`） | 重新联网核验；确需更严格可用 `--max-age-days` |

#### (b) 在线验证阶段

`scripts/verify_citations.py:222-229` 会逐条打印 `STATUS: key via provider`，非 `verified` 返回 1，但**锁文件仍写出**，可直接查看 `error` 字段。

| 状态 | 触发代码 | 含义与动作 |
|---|---|---|
| `not-found` | Crossref 404 → DataCite 404（`verify_citations.py:57-96`） | identifier 可能有误；先到 DOI 官网核对，不要换成搜索结果链接 |
| `rate-limited` | HTTP 429（:37-39） | 稍后重试，带 `--mailto`；记录失败而不是当作“没有文献” |
| `network-error` / `provider-error` | 网络异常 / 其他 HTTP 码 / 解析失败（:40-45,185-191） | 检查代理与证书；`fetch()` 只认 JSON/Atom 结构 |
| `incomplete-metadata` | 缺 title/year/authors（:175-177） | 主 provider 数据不全；换更权威的 identifier 或第二个 provider |
| `conflict` | OpenAlex 与主记录的规范化 `title/year/doi` 不一致（:141-144,178-181） | 并列展示两条记录，按主来源裁定；OpenAlex 永不覆盖主记录（`citation-workflow.md:30-34`） |
| `metadata-only` | 任一 `claim_support.status ∉ {verified, software-doc}`（:21,182-184） | 打开论文读原文，写清 `claim/relation/evidence/source` 后再核验 |

**摘要链失败的统一修法**：`metadata digest mismatch / claim-support digest mismatch / cross-check digest mismatch`（`citation_lock.py:132-137`）说明“只改了 JSON 的一部分”。直接重跑 `verify_citations.py`，它会用 `canonical_digest()` 重算四个摘要（:25-28，`verify_citations.py:215-220`）。

**覆盖度失败**（`citation_lock.py:165-177`）：

```text
cited keys missing from citation_requests.json: X
requested keys missing from citation_lock.json: Y
citation_lock.json has unexpected keys: Z
```

三者是**双向集合相等**检查：手稿引用、请求、锁记录必须完全对齐。删掉手稿里不再用的引用后，记得同时清理请求与锁并重新核验，否则 `unexpected keys` 会一直存在。

### 1.3 claim map / handoff 断链

- `Claim/evidence map has unresolved statuses: needs evidence`（`research_quality_gate.py:240-245`）：终态时状态列不能命中 `OPEN_STATUS_RE`（:54）。修复：补 `Evidence`（文件:行号 / 实验 ID / 已核验引用）并把状态改成 `Supported`；若证据不足就**弱化或删除该主张**，不要只改状态词。
- `claim_evidence_map.md table requires a Status column`（:238-241）：表头需含 `Status`；`markdown_rows()` 只认带 `|---|` 分隔线的首张表（:82-99）。
- `Required artifact is missing/empty: ...`（:68-80）：基线工件清单见 §3.4 M 步；空文件也算缺失。
- `A terminal stage cannot contain unresolved blockers: ...`（:318-319）：blocker 非空却写了 `stage=complete/submission-ready`。保留 blocker 并把 stage 退回 `drafting` 等非终态值，或在确实解决后清空 blocker。
- `plan/progress.md still describes verification as pending.`（:334-336）：终态进度文件不能出现 `verification planned/to be recorded/not yet run/待验证/待运行/尚未验证`（:50-53）。改成“已执行 + 命令 + 结果”。
- `research_handoff.json ... escapes project root` / `does not exist` / `is empty`（`check_research_handoff.py:36-49`）：artifacts 路径必须项目内相对、存在、非空；`quantitative=true` 时 `experiment_inventory` 与 `numeric_evidence` 是硬性要求（:70-79）。
- `research handoff is structurally valid with declared blockers`（:110-116）：这是**报告**而非失败；只有加 `--require-unblocked` 才返回 1。正式全文写作前必须使用该参数（`SKILL.md:33-35`）。

### 1.4 构建摘要失效

| 报错 | 触发 | 最小修复 |
|---|---|---|
| `Build input digest is stale; rebuild the PDF and record the new digest.`（`research_quality_gate.py:431-432`） | 输入图/外部输入摘要与 state 不符 | 重跑真实构建，再 `record_build.py <project>`（或 `--run` 一次完成） |
| `Recorded PDF digest does not match the current PDF.`（:433-434） | PDF 被覆盖/手工替换 | 重新登记；先确认 PDF 确由声明命令产出 |
| `Recorded build log digest does not match the current log.`（:410-411） | 日志被编辑 | 重新 `--run`，不要手改日志 |
| `PDF is older than one or more build inputs; run the declared build command before recording`（`record_build.py:101-107`） | 源文件 mtime 晚于 PDF | 先编译再登记 |
| `A terminal stage requires build.status = passed.`（:398-399） | 构建仍未通过 | 修编译错误后 `--run` |
| `submission-ready requires record_build.py --run attestation.`（:400-401） | `attestation=artifact-only` | 必须真实执行一次声明命令 |
| `Executed build attestation requires exit_code = 0.`（:403-404） | 编译失败却登记了 executed | 看日志，修 TeX 错误，重跑 |

输入摘要覆盖 `\input/\include` 展开后的全部 TeX、`\includegraphics` 解析出的图形、BibTeX、外部模板记录，以及 executed 模式下的 `{command, tool_version}`（`paper_contract.py:122-153`、`record_build.py:112-119`）。因此**只改一个标点也会让摘要失效**，这是预期行为。

---

## 2. LaTeX 编译与格式不兼容

### 2.1 模板获取与 venue 样式

| 症状 | 根因 | 修复 |
|---|---|---|
| `No pinned archive is configured for X. Download it manually from ...`（`fetch_template.py:60-65`） | manifest 中该项无 `archive_url/sha256` | 从官方页面手工下载，并把来源与哈希在 `build.external_inputs` 中声明 |
| `Template archive hash mismatch ... The upstream package changed`（:74-81） | 上游 zip 变了 | 人工审计新包内容，确认许可与差异后再更新 `templates/manifest.json`；不要盲目改哈希 |
| `Archive contains a symbolic link` / `Unsafe archive path`（:22-33） | zip 含 symlink、绝对路径或 `..` | 换官方原包；这是安全拒绝，不是可绕过的警告 |
| `Output path already exists; refusing to merge or overwrite`（:53-55） | 目标目录已存在 | 换新目录或先人工移走旧目录；脚本刻意不合并 |
| `Vendored template directories are not allowed`（`research_quality_gate.py:163-164`） | 把模板复制进技能仓库 `templates/` | 用 `fetch_template.py --output <目标项目>/.venue-template` |

格式兼容的规范来源：

- `references/style-presets.md:14-23`：NeurIPS/ICLR 单栏 5.5in；ICML 3.25/6.75in；ACL 3.3/6.8in；AAAI 3.3/7.0in；跨栏图用 `figure*`。
- `references/figure-workflow.md:62-90`：数据图 PDF/SVG 优先，栅格 ≥300 DPI，字号 ≥7pt（`style-presets.md:5-13`），色盲安全色板。
- `examples/paper-about-ai-research-writing-skill/build_check.md:5-14`：真实样例用 `tectonic -Z search-path=.venue-template main.tex`，外部模板地址/哈希登记在 `examples/paper-about-ai-research-writing-skill/paper_state.json:20-25`。

### 2.2 输入图与路径

`paper_contract.py` 的严格解析会在**编译之前**拦住这些问题：

| 报错 | 行号 | 说明 |
|---|---|---|
| `Missing LaTeX input declared by <path>: <child>` | :59-77 | `\input/\include` 指向不存在文件；无扩展名时自动补 `.tex`（:54-57） |
| `Missing graphic declared by <tex>: <raw>` | :84-95 | `\includegraphics` 找不到；依次尝试 `""`, `.pdf`, `.png`, `.jpg`, `.jpeg`, `.eps`, `.svg`（:20） |
| `Required text file is not valid UTF-8` | :27-37 | 非 UTF-8 的 TeX/bib/JSON；转码 |
| `Required file does not exist: <path>` | :27-29,154-159 | 声明的主 tex、bib、PDF 或日志缺失 |
| `Invalid JSON in <path>: line N, column M` | :112-119 | `paper_state.json` / 注册表 / 请求 / 锁文件坏 JSON |

**注意**：命令 `pdflatex main.tex` 能编译而 `record_build.py --run` 失败，通常是因为 `build.command` 在 `main_tex.parent` 下执行（`record_build.py:52`），而示例命令用了 `-Z search-path=.venue-template` 的相对路径。检查当前工作目录与模板目录是否随主 tex 一起存在。

### 2.3 构建命令与日志

`record_build.py` 对 `build.command` 的安全约束（:45-49）：

- 用 `shlex.split()` 解析，必须是**单个可执行 argv**；
- 拒绝 `|`、`||`、`&&`、`;`、`<`、`>`、`>>` 作为 token；
- 需要管道/多步构建时，写一个构建脚本（如 `scripts/build_paper.sh`）并把命令设为 `bash scripts/build_paper.sh`，同时把脚本本身纳入输入图/外部输入声明；
- 默认超时 300 秒（`--timeout` 可调，:21,55-56）；`--version` 探测最多 20 秒（:83）。

`parse_build_log.py:11-20` 识别 7 类日志信号，输出 `# Build Log Summary` + 每类计数与去重条目（:44-59）：

| 类别 | 正则 | 是否导致退出码 1 |
|---|---|---|
| `errors` | `^! .+` | 是 |
| `emergency` | `Emergency stop` / `Fatal error occurred` | 是 |
| `undefined_citations` | `Citation ... undefined` | 是 |
| `undefined_references` | `Reference ... undefined` | 是 |
| `missing_files` | `LaTeX Error: File 'X' not found` | 是 |
| `overfull` / `underfull` | `Overfull/Underfull \hbox ...` | 否（默认 `--max-lines 20` 只截断显示） |

`undefined_citations` 通常是 bib 条目缺失或编译次数不够（bibtex 流程需要 pdflatex→bibtex→pdflatex×2，`submission-packaging.md:17-30`）；`undefined_references` 是 `\ref`/`\label` 不匹配。两者都应先修源文件再重建，**不要手工编辑 PDF 或日志**。

### 2.4 手稿里的占位标记与过程指令

`validate_manuscript()`（`research_quality_gate.py:253-265`）在拼接后的 TeX 上扫两类正则：

- `UNRESOLVED_RE`（:39-43）：`TODO/FIXME/TBD/XXX/PLACEHOLDER/CITATION NEEDED/FILL LATER` 与 `待回填/待替换/待补充/公式占位/算法占位`；
- `PROCESS_RE`（:44-49）：`write naturally/avoid ai/replace later/user should/this section is a template/.../figure position/table position` 与 `写作要求/修改要求/请用户/用户替换/回填模板/讨论提示/图位/表位/实验目的`。

报错格式：`Manuscript contains unresolved marker near collected line N: <前140字符>`。这类失败**没有豁免机制**：把图位说明、写作要求移到 `figures/figure_plan.md` 或 `plan/task-packets/`，正文只留实义内容。

### 2.5 camera-ready 静态检查七项

`camera_ready_check.py:38-52` 逐项输出 pass/fail：

1. `No unresolved markers` — `TODO_RE` 不命中；
2. `Limitations section present` — `\section{...Limitations...}`；
3. `Bibliography command present` — `\bibliography{` 或 `\printbibliography`；
4. `Venue checklist artifact present` — 真实 Checklist 章节或 `--checklist FILE` 指向存在文件（:36）；
5. `Non-anonymous author metadata present` — `\author{...}` / `\icmlauthor{...}` 且不是 `Anonymous`（:43）；
6. `Acknowledgments section present` — 或有 venue 例外说明；
7. `Declared LLM disclosure file resolves` — 传了 `--llm-disclosure` 就必须存在（:37）。

常见误区：`tests/test_quality_tools.py:572-591` 明确验证“正文里写了 checklist 字样”不算 checklist 工件；必须提供文件。

---

## 3. 单步写作幻觉阻断排查

这里的“阻断”不是运行时拦住模型，而是指：**当你想用一个写作步骤直接产出终稿时，证据链的某一条断言让它无法被调用为完成**。理解这一点能避免“换个提示词再试一次”的无效循环。

### 3.1 幻觉类型与对应闸门

| 幻觉形态 | 表现 | 拦截它的代码/规范 | 正确动作 |
|---|---|---|---|
| 编造数字 | 正文出现源数据里没有的 `+12.3%` | `check_numeric_evidence.py:270-303` + 总闸门 :344-356 | 从不可变日志生成 selector entry，或删掉该数字 |
| 编造引用 | `\cite{author2024}` 无 bib / 无锁 | `research_quality_gate.py:357-383`、`citation_lock.py:91-178` | 用 DOI/arXiv 主来源重新核验；查不到就删引用并弱化句子 |
| 引用“_topic 相关但不支撑句子_” | 元数据真实、`relation=background` 却用于强主张 | 规范层 `citation-workflow.md:47-59`；脚本只检查 `status` 终态（非语义） | 把 `claim_support.relation` 改成 `direct/contrast/software` 并填具体 `evidence`，否则改写句子 |
| 生成图被当成证据 | 概念图里出现“实验结果”数字 | `figure-workflow.md:28-35,57`、`figure-spec.md:57` | 数字图改确定性生成；概念图只表达流程/思想 |
| 过程指令泄漏 | 正文出现“图位/请用户替换/写作要求” | `PROCESS_RE`（`research_quality_gate.py:44-49`） | 移到 plan/spec 文件 |
| 未完成的框架冒充完成 | `needs evidence` / `待验证` 留在终态 | `OPEN_STATUS_RE` :54、`STALE_PROGRESS_RE` :50-53 | 补证据或退回非终态 stage |
| 整段凭空写作 | 没有研究上下文却生成“带引用”报告 | 规范层 `SKILL.md:8-13`；本仓库没有运行时 LLM 守卫 | 先完成证据盘点与 claim map，再进入写作 |
| 过早优化措辞 | 先写 Abstract，实验后反复对不上 | `workflow.md:24-44`（先证据承载核心、最后重写 Abstract） | 按两遍顺序重做，保留旧稿在 `plan/` 便于审计 |

> 跨仓库参照：在 `gpt-researcher` 中，运行时有一个类似的“空上下文弃答”守卫——gpt-researcher 仓库的 gpt_researcher/skills/writer.py 第 79-88 行 在没有研究材料时直接返回拒答文本。本技能包是声明式 skill，**没有**这个 Python 守卫；它的等价物是“证据不足就弱化/删除主张并阻断终态”的人工/agent 流程（`SKILL.md:48-56`）。不要误以为本地 `scripts/` 会在写作时实时拦截幻觉。

### 3.2 单步写作被阻断时的诊断树

```text
质量门失败
├─ Unverified manuscript numbers → 数字链：定位 collected line → 补 registry entry / 改措辞
├─ Cited keys lack verified record → 引用链：补 verification 行或更新 requests+lock
├─ Claim/evidence map unresolved → 主张链：补 Evidence 或弱化 claim（禁止只改 Status 字符串）
├─ Manuscript contains unresolved marker/process instruction → 清洁度：移出正文
├─ Required artifact missing/empty → 先产出工件（plan/task-packets、build_check 等）
├─ blockers 非空 / progress 仍写待验证 → 退回非终态 stage，记录真实风险
└─ Build digest stale / attestation 不合规 → 真实重建并重登记
```

### 3.3 让“一个写作步骤”更安全的操作约束

来自 `workflow.md:35` 与 `abstract-introduction.md:5-7`、`reviewer-self-review.md:26` 的可执行约束：

1. **一次调用只写一个 numbered block**：输入 = paper story + claim map + 该块相关证据 + 已完成前置章节；不允许顺手改写无关章节。
2. **先写强主张的证据列，再写句子**：如果 claim map 的 Evidence 为空，先停止写作，回到实验日志/文献。
3. **不要边审边改**：`reviewer-self-review.md:26` 明确要求先记录带定位的具体发现、去重、分级，再决定是编辑、补实验、补引用、写 limitation、删主张还是记 blocker。
4. **措辞强度与证据强度一致**：`shows`（直接证据）/ `suggests`（有限证据）/ `is consistent with`（非因果）（`section-writing.md:47`）；`strong/careful/aspirational/forbidden` 四级（`paper-story.md:150-157`）。
5. **每步后跑一次对应检查器**：写完核心章节跑数字检查；补完引用跑 `check_citations.py + check_citation_lock.py`；任何实质改动后都重跑总闸门（`workflow.md:65`）。
6. **保留 human-in-the-loop**：科学正确性、图像目视检查、审稿风险和投稿决定不交给脚本或模型（`SKILL.md:11-13`）。

### 3.4 什么时候可以宣布完成

按 `references/task-management.md:36-44` 与 `SKILL.md:58`：

- task packet 的范围与必需产物逐项核对；
- claim 支持、引用契合、手稿清洁度与审稿风险已检查；
- 声明的验证命令全部实际执行，并**看过输出**；
- `paper_state.json` 已更新、构建已登记、总闸门通过；
- 每个剩余风险要么是显式 blocker（此时不能是终态），要么是已记录的 limitations。

`SKILL.md:60-66` 的最终报告必须包含：实际改动的文件、跑过的检查、剩余科学风险。只说“写好了”而没有任何命令输出，不构成完成。

---

## 4. 应急参考：常用命令与文件对应关系

| 目的 | 命令 | 产出/读取 |
|---|---|---|
| 总闸门（论文） | `python3 scripts/research_quality_gate.py <project> --mode full-paper` | 退出码 0/1/2 |
| 总闸门（技能包） | `python3 scripts/research_quality_gate.py . --mode skill` | 检查 SKILL.md/README/scripts/templates |
| 数字 | `python3 scripts/check_numeric_evidence.py <project>` | 未登记数字列表 |
| 引用在线 | `python3 scripts/verify_citations.py <project> --mailto ...` | 重写 `citation_lock.json` |
| 引用离线 | `python3 scripts/check_citation_lock.py <project>` | 锁失败列表 |
| bib key | `python3 scripts/check_citations.py paper/main.tex paper/references.bib` | 缺失/重复 key |
| 主张候选 | `python3 scripts/extract_claims.py paper/main.tex` | Markdown 表到 stdout |
| 构建 | `python3 scripts/record_build.py <project> --run` | 日志/版本/退出码/哈希写回 state |
| 日志摘要 | `python3 scripts/parse_build_log.py paper/main.log` | 7 类问题计数 |
| 投稿静态检查 | `python3 scripts/camera_ready_check.py paper/main.tex --checklist ...` | 7 项 pass/fail 表 |
| 模板 | `python3 scripts/fetch_template.py --list` / `... icml2026 --output ...` | 官方包下载与校验 |

---

## 5. 锚点速查

| 主题 | 文件:行号 |
|---|---|
| 总闸门常量与正则 | `scripts/research_quality_gate.py:30-55` |
| skill 模式检查 | :102-164 |
| 论文模式决策树 | :267-436 |
| 数字重算与扫描 | `scripts/check_numeric_evidence.py:104-303` |
| 引用锁校验 | `scripts/citation_lock.py:91-178` |
| 在线 provider 状态 | `scripts/verify_citations.py:31-45,146-229` |
| 输入图与哈希 | `scripts/paper_contract.py:27-160` |
| 构建登记 | `scripts/record_build.py:17-132` |
| 模板安全下载 | `scripts/fetch_template.py:22-106` |
| 相机就绪七项 | `scripts/camera_ready_check.py:21-52` |
| 回归用例 | `tests/test_quality_tools.py:142-627` |
