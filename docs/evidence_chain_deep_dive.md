# 证据链深度拆解：从实验日志到可审计论文

> 事实来源：本仓库源码；`路径:行号` 是本检出版本的定位锚点。本文只解释**已实现**的算法，不把规范文档中的意图写成脚本已经保证的事实。
>
> 与 [`architecture.md`](architecture.md) 的分工：那篇讲“有哪些机制”，这篇讲“每一行代码如何把证据绑定成一条链、链在哪里会断、怎样组织本地实验材料才能让论文质量最大化”。

## 0. 一页速览：四条互相独立的证据链

这个技能包没有“一个总证据数据库”。它由四条各自可断裂、又彼此嵌套的链组成：

```text
链 A 数字   原始 CSV/JSON/JSONL  ──selector+aggregate──>  numeric_evidence.json ──> 手稿数字
链 B 引用   citation_requests.json ──provider 元数据+摘要──> citation_lock.json ──> citation_verification.md ──> BibTeX/手稿
链 C 主张   Abstract/Introduction/代码/日志 ──extract_claims.py 启发式──> claim_evidence_map.md ──> 手稿措辞强度
链 D 构建   TeX 输入图+图形+BibTeX+外部模板 ──SHA-256──> paper_state.json.build ──> PDF
```

四条链的**共同性质**：

1. **严格模式（fail-closed）**：声明的输入读不到、解析不了、字段不认识、空文件，都会报错而不是跳过（`paper_contract.py:27-51,112-119`、`research_quality_gate.py:68-80`）。
2. **摘要而非信任**：链上每一跳都保存的是规范化后的摘要（SHA-256）与来源 URL/路径，而不是“模型说它是对的”。
3. **终态才强制闭环**：非终态草稿允许缺件；`stage ∈ {complete, submission-ready}` 时才把未核验、未构建、有 blocker 判为失败（`research_quality_gate.py:38,267-436`）。
4. **机器检查 ≠ 科学判断**：脚本证明“数字可从声明源文件重算、引用元数据一致、文件哈希没变”，**不证明**实验设计正确、引用真的支持句子、图看起来对（`docs/README.md:57-66`）。

---

## 1. 链 A：数字证据（numeric evidence v2）

### 1.1 绑定模型与源码锚点

| 概念 | 实现 | 位置 |
|---|---|---|
| schema 版本与白名单 | `SCHEMA_VERSION`, `ROOT_FIELDS`, `ENTRY_FIELDS`, `SELECTOR_FIELDS`, `SELECTOR_KINDS`, `AGGREGATES`, `REPRESENTATIONS` | `scripts/check_numeric_evidence.py:19-25` |
| “严格章节”词表 | `STRICT_SECTION_TERMS`（result/experiment/evaluation/ablation/analysis/finding/benchmark） | :26-34 |
| 章节 / 表格 / 数字 / 跳过区正则 | `SECTION_RE`, `TABLE_RE`, `NUMBER_RE`, `SKIP_RE` | :35-50 |
| 容忍度匹配 | `EvidenceValue.matches()` → `math.isclose(rel_tol=tolerance, abs_tol=tolerance)` | :52-59 |
| 发现结果对象 | `NumericFinding(value, line, section, context)` | :62-66 |
| 路径逃逸检查 | `_source_path()`：禁 `#` 片段、必须落在项目根内、必须是文件 | :69-82 |
| JSON Pointer 解析 | `_json_pointer()`：`~1`→`/`、`~0`→`~`、数组下标必须数字化 | :84-98 |
| 行过滤 | `_matches_where()`：所有键值按 `str()` **精确相等**比较 | :100-102 |
| 取值器 | `_selected_values()`：四种 kind，拒绝 bool / NaN / Inf / 空集 | :104-175 |
| 聚合器 | `_aggregate()`：`identity/mean/sample-std/population-std/sum/min/max/count` | :177-201 |
| 注册表加载 + 重算断言 | `load_registry()`：字段白名单→重算→`isclose`→生成待匹配值 | :203-258 |
| 扫描手稿 | `_mask_ranges()` + `find_unverified_numbers()` | :261-303 |
| CLI | `verify_project()` / `main()` | :305-345 |

### 1.2 注册表条目究竟承诺了什么

`numeric_evidence.json` 只有两个顶层字段：`schema_version` 和非空 `entries`（:19-20,203-214）。每个 entry 的字段语义如下：

| 字段 | 必填 | 代码行为 | 容易踩的坑 |
|---|---|---|---|
| `value` | 是 | 有限数；bool 被显式拒绝（:227-230） | 写 `"0.912"` 字符串会失败 |
| `source` | 是 | 项目根内相对路径；**不允许** `results.json#/score` 片段（:69-82） | v1 写法必须迁移成 `source` + `selector.pointer`（`references/numeric-evidence.md:36`） |
| `selector` | 是 | 只能是 `{kind,pointer,column,where}` 的子集（:104-113） | 加 `limit`、`row` 等自定义字段会直接失败 |
| `aggregate` | 否（默认 `identity`） | `identity` 要求**恰好一个**选中值（:177-186） | 多行 CSV 忘记改 `mean` 是最高频错误 |
| `tolerance` | 否（默认 `1e-6`） | 同时作为 `rel_tol` 与 `abs_tol`（:52-59,235-238） | 手写四舍五入的数字要显式放宽，例如 `0.0001` |
| `representations` | 否（默认 `["raw"]`） | `raw` 登记 `value`；`percent` 额外登记 `value*100`（:250-257） | 正文写 `91.2%` 但只登记 `raw:0.912` → 失败 |
| `note` | 否 | 不参与计算，纯人类说明 | 不要在这里塞另一个数字指望它被登记 |

示例（与 `references/numeric-evidence.md:5-20` 同构）：

```json
{
  "schema_version": "ai-research-writing/numeric-evidence-v2",
  "entries": [
    {
      "value": 0.912,
      "source": "experiments/raw_results.csv",
      "selector": {"kind": "csv", "column": "accuracy", "where": {"method": "ours", "seed": "42"}},
      "aggregate": "mean",
      "representations": ["raw", "percent"],
      "tolerance": 0.0001,
      "note": "Mean accuracy over 3 seeds of the final system"
    }
  ]
}
```

这段注册表的含义是：脚本会打开 CSV，筛出 `method=ours 且 seed=42` 的行，取 `accuracy` 列，算 `statistics.fmean`，要求结果与 `0.912` 在 `1e-4` 内一致；随后手稿中 `0.912` 与 `91.2` 都被视为“已登记数字”，`91.2%` 的 `%` 不参与数字正则。

### 1.3 取值与聚合的完整分支

`_selected_values()`（:104-175）按 `selector.kind` 分支，行为差异很大：

| kind | 必需字段 | 解析方式 | 典型失败信息 |
|---|---|---|---|
| `json-pointer` | `pointer`（必须以 `/` 开头）；源文件必须 `.json` | `json.loads` 后逐 token 下钻；标量包装成单元素列表，数组原样保留 | `JSON Pointer does not resolve`、`requires a .json source` |
| `csv` | `column`；可选 `where` | `csv.DictReader(delimiter=",")`；表头缺列即失败 | `column 'accuracy' is missing from ...` |
| `tsv` | `column`；可选 `where` | 同上，仅 delimiter 为 `\t` | 同上 |
| `jsonl` | `column`；可选 `where` | 逐行 `json.loads`，行号写进错误；每行必须是对象 | `Invalid JSONL at source:12` |

所有分支最终都要经过同一道数值净化（:157-175）：

```text
选出的值 → 拒绝 bool → float(x) → math.isfinite → list[float]
空列表直接 ContractError("selector matched no values")
```

`_aggregate()`（:177-201）的约束值得单独记住，因为它经常是“我明明登记了却失败”的根因：

- `identity`：选中值数量必须 **== 1**；
- `sample-std`：至少 2 个值（内部用 `statistics.stdev`）；
- `population-std`：`statistics.pstdev`，无最小数量限制；
- `sum`：`math.fsum`（不是内建 `sum`，避免浮点误差累积）；
- `min/max/count`：`count` 返回选中值个数的浮点形式；
- 任何其他字符串：`Numeric evidence aggregate must be one of ...`。

### 1.4 手稿扫描算法（逐阶段）

`find_unverified_numbers()`（:270-303）不是“把手稿数字和注册表 String 比较”，它是一个五阶段流水线：

1. **合并 TeX 输入图**：`verify_project()` 调用 `collect_tex_text(main_tex)`（:305-308）。`paper_contract.py:59-81` 递归解析 `\input/\include`，按 DFS 顺序拼接；因此报错里的 `collected line N` 是**拼接后文本的行号**，不是某个 `main.tex` 的物理行号。排查时要先跑 `python3 -c` 打印 `collect_tex_files()` 的顺序。
2. **屏蔽非数字区域**：`_mask_ranges()`（:261-268 / 调用于 :271）用 `SKIP_RE`（:41-50）把 `\cite{...}`、`\ref{...}`、`\url{}`、`\href{}{}`、`\includegraphics[]{}`、`\begin/\end{}` 等整段替换为**等长空格**，保持后续行号不漂移。
3. **识别表格范围**：`TABLE_RE`（:36-39）记录 `table*/tabular*/tabularx/longtable` 环境的 `(start, end)` 字符区间（:272）。表格里的数字**一律严格检查**，不管所在章节名。
4. **识别当前章节**：`SECTION_RE`（:35）收集 `\section/\subsection/\subsubsection` 的位置与标题，然后这些命令本身也被 mask 掉（:273）。扫描时按字符位置推进 `current_section`（:279-282）。
5. **逐个数字判定**：`NUMBER_RE`（:40）支持千分位、正负号、小数、科学计数法；两侧不能紧邻字母/下划线/反斜杠（避免匹配 `\le2` 中的 `2`）。对每个候选：
   - 若不在表格且当前章节标题不含 `STRICT_SECTION_TERMS` 任一关键词 → 跳过（:290-292）；
   - 若与任一 `EvidenceValue` 在 tolerance 内匹配 → 跳过（:293-294）；
   - 否则生成 `NumericFinding(value, line, section, context)`，其中 `line = 拼接文本中该数字所在行`，`context` 为该行压缩空白后的前 180 字符（:295-302）。

**推论（很多用户踩过）**：把结果写成 `We improve by 30%.` 并放在名为 `Discussion` 的小节、且不在表格里，脚本不会检查这个 `30`；但同样的句子放进 `Results` 小节就会失败。这不是 bug，而是“证据敏感区域”的设计取舍——不要利用它来藏数字，审稿人和 `references/reviewer-self-review.md` 的审计仍会看到。

### 1.5 数值链的失败信号与修复顺序

| 报错片段 | 根因 | 最小修复 |
|---|---|---|
| `numeric_evidence.json value X does not match computed source value Y` | 注册表 `value` 与源数据重算结果不一致 | 以源数据为准改 `value`，或修改 `selector.where/aggregate` 使重算语义正确；不要反向改源文件 |
| `selector matched no values in ...` | `where` 过严、列名大小写不符、JSONL 里 `seed` 是数字而 `where` 写字符串（比较用 `str()`，见 :100-102） | 用 `python3 -c` 打印源数据前几行，核对字段名与类型 |
| `identity aggregate requires exactly one selected value` | 多行结果用了默认聚合 | 改 `aggregate` 为 `mean`/`max` 等，或在 `where` 加 seed/run 维度 |
| `Unverified manuscript numbers: 91.2 in Results (collected line N)` | 正文用 percent 表示，注册表只有 raw | 在 `representations` 加 `"percent"`，或登记 `value: 91.2` 的另一条 |
| `Required text file is not valid UTF-8` | 日志/源数据非 UTF-8 | 转成 UTF-8 并重算；脚本不会猜测编码 |
| `Numeric evidence source escapes project root` | 使用 `../shared/results.csv` 或绝对路径 | 把数据复制/软链进项目内并登记项目内相对路径 |

修复后按“源文件 → 注册表 → 手稿”的顺序重跑：

```bash
python3 scripts/check_numeric_evidence.py /path/to/paper-project
python3 scripts/research_quality_gate.py /path/to/paper-project
```

---

## 2. 链 B：引用（citation requests → lock → verification table）

### 2.1 三维验证不可合并

`references/citation-workflow.md:3-9` 把验证拆成：①存在性与元数据；②BibTeX 完整性；③句级支持关系。`check_citations.py:13-43` **只做②**（外加输入图完整性），`verify_citations.py` 主要做①并记录③的声明，两者都不能单独证明“这句引用是对的”。

### 2.2 请求契约：逐字段解析

`load_requests()`（`citation_lock.py:30-88`）的断言顺序：

1. 顶层字段集合**恰好** `{schema_version, entries}`（:32-35）；
2. `schema_version == "ai-research-writing/citation-requests-v1"`（:15,36-37）；
3. `entries` 为非空列表（:38-40）；
4. 每个 entry 字段集合**恰好** `{key, identifier, claim_support}`（:44-49）；`key` 非空且全局唯一（:51-58）；
5. `identifier` 必须以 `doi:` 或 `arxiv:` 开头（大小写不敏感）（:59-66）；
6. `claim_support` 非空数组；每项的字段集合**恰好** `{claim, relation, status, evidence, source}`（:67-75）；`claim/status/evidence/source` 必须是非空字符串，`status` 这里指**句级支持状态**；
7. `relation ∈ {direct, background, contrast, software, partial, weak, metadata-only}`（:20,81-85）。

`SUPPORT_TERMINAL = {verified, software-doc}`（:21）是锁校验时对**每一项** claim_support 的要求。`LOCK_TERMINAL = {verified}`（:22）是锁记录本身的要求——注意 `software-doc` 在 verification 表里是终态（`research_quality_gate.py:215-223`），但在 `citation_lock.json` 里不是；因此软件/仓库类引用通常走 `citation_verification.md` 的 `software-doc` 行，而不进入强制锁的学术引用集合。

### 2.3 在线验证：provider 路由与降级

`verify_citations.main()`（`scripts/verify_citations.py:146-229`）对每个请求按如下流程执行：

```text
identifier
 ├─ doi: ──> Crossref crossref() (:57-77)
 │           ├─ 成功 → provider="crossref"
 │           └─ ProviderFailure(status="not-found") → DataCite datacite() (:79-96)
 │                                                      → provider="datacite"
 └─ arxiv: ─> arXiv Atom API arxiv() (:98-118) → provider="arxiv"

元数据三要素(title/year/authors)缺一 → incomplete-metadata (:175-177)
若设置 OPENALEX_API_KEY → openalex() (:120-139) 交叉核对
   comparable() (:141-144): 仅比较 casefold+去非字母数字的 title、year、doi
   不一致 → status="conflict"，error 记录冲突；OpenAlex 永不覆盖主记录
任一 claim_support.status ∉ SUPPORT_TERMINAL → status="metadata-only" (:182-184)
网络/解析异常 → status = not-found / rate-limited / provider-error / network-error ... (:185-191)
```

`fetch()`（:31-45）统一设置 `User-Agent: ai-research-writing-skill/1.0`（带 mailto），把 HTTP 404/429/其他错误码与网络异常映射为带 `status` 的 `ProviderFailure`：这是“失败也写进锁、绝不静默为空”的关键。**任何非 `verified` 状态都会让 CLI 返回 1**（:225-229），但锁文件仍会被写出，便于检查失败原因。

### 2.4 锁记录的摘要链

每条锁记录包含 12 个固定字段（顺序无关，集合必须完全一致，`citation_lock.py:107-118`）：

```text
key, identifier, status, provider, request_url, checked_at,
metadata, metadata_sha256, claim_support_sha256,
cross_check, cross_check_sha256, error
```

摘要全部由 `canonical_digest()`（:25-28）计算：`json.dumps(sort_keys=True, separators=(",", ":"), ensure_ascii=True)` 后 SHA-256。因此：

- `request_sha256 = canonical_digest(完整 requests 对象)`（`verify_citations.py:220`）；
- `metadata_sha256 = canonical_digest(本条 metadata)`（:215）；
- `claim_support_sha256 = canonical_digest(request 中本 key 的 claim_support)`（:216）；
- `cross_check_sha256 = canonical_digest(cross_check 对象或 None)`（:218）。

这形成一条有向链：**改请求 → 顶层摘要失效；改逐句支持 → 条目摘要失效；改 provider 元数据 → 元数据摘要失效**。任何一处“只改了 JSON 没重跑 verifier”都会被 `validate_lock()` 抓到。

### 2.5 离线锁校验的全部断言

`validate_lock()`（`citation_lock.py:91-178`）逐项检查：

| 类别 | 断言（行号） | 失败文案示例 |
|---|---|---|
| 参数 | `max_age_days > 0`（:94-95） | `max_age_days must be positive` |
| 请求 | 请求本身先通过 `load_requests()`（:96） | 见 §2.2 |
| 锁顶层 | 字段集合恰好 `{schema_version, request_sha256, records}`（:99-100） | `must contain only ...` |
| 版本 | `LOCK_SCHEMA`（:101-102） | `schema_version must be ...` |
| 新鲜度 | `request_sha256` 与请求重算摘要一致（:103-104） | `citation_lock.json is stale for citation_requests.json` |
| 记录 | `records` 为列表；每项字段集合固定；`key` 非空不重复（:106-124） | `invalid field set` / `empty or duplicated` |
| 对应关系 | 记录必须有请求；`identifier` 与请求相同（:125-131） | `lock record has no request` / `identifier changed` |
| 摘要 | metadata / claim_support / cross_check 三个摘要逐一重算比对（:132-137） | `metadata digest mismatch for KEY` |
| 终态 | `status == "verified"`；provider、request_url 非空且 error 为空（:138-145） | `citation is not verified` |
| 时间 | `checked_at` 可解析、带时区；不在未来；不超过 `max_age_days`（默认 180，`check_citation_lock.py:18`）（:146-160） | `lock is older than 180 days` |
| 句级 | 请求中每条 `claim_support.status ∈ SUPPORT_TERMINAL`（:161-163） | `claim support is not terminal` |
| 覆盖度 | 手稿 `\cite` key 与请求 key、锁记录 key 必须**双向相等**：无缺失、无多余（:165-177） | `cited keys missing from citation_lock.json` / `unexpected keys` |

手稿 key 来自 `paper_contract.py:14-18,98-106` 的 `CITE_RE`：`\cite/\citep/\citet/\citealp/\citealt/\citeauthor/\citeyear/\parencite/\textcite/\autocite/\footcite/\supercite/\nocite`，支持星号与两个可选方括号参数，逗号分隔的多个 key 会被拆开。

### 2.6 verification 表与总闸门的耦合

`research_quality_gate.py:82-99` 的 `markdown_rows()` 实现一个**很小的 Markdown 表格解析器**：找到第一行以 `|` 开头且下一行匹配 `^\s*\|?(?:\s*:?-{3,}:?\s*\|)+\s*$` 的位置；表头去 `|`、去空白、转小写；后续连续 `|` 行且单元格数等于表头数才被收录；找不到表直接 `ContractError`。

`validate_verification_table()`（:205-233）在此基础上：

- 表头必须含 `key` 与 `status`（:210-214）；
- key 单元格去掉反引号（:219）；
- `status ∈ {verified, software-doc, repository-verified}` 记入 verified；仅 `verified` 记入 `scholarly`（:220-223）；
- 其他状态（含空状态）全部作为 unresolved 报错（:224-226）；
- `cited - verified` 报错（:227-229）：**手稿里每个被引 key 都要有终态记录**；
- 只有 `scholarly` 非空时才要求 `citation_requests.json` + `citation_lock.json` 存在并跑 `validate_lock()`（:230-233）——这就是纯软件引用项目可以没有锁文件的原因。

样例表的真实形状见 `examples/paper-about-ai-research-writing-skill/citation_verification.md:3-9`（表头为 `Key | Source | Used for | Status`，状态 `software-doc`）。

---

## 3. 链 C：主张—证据表与 `extract_claims.py`

### 3.1 启发式提取器做了什么

`scripts/extract_claims.py` 是**bootstrap 工具**，不是验证器；文件头 docstring 明确说“human/agent must still verify evidence and revise unsupported claims”（:1-5）。

| 阶段 | 函数/正则 | 行号 | 关键行为 |
|---|---|---|---|
| 去注释 | `strip_comments()` | :34-48 | 识别转义 `\%`；逐行截断未转义 `%` |
| 收集宏 | `MACRO_RE` / `collect_macros()` | :30,50-52 | 只匹配一层花括号的 `\newcommand{\X}{...}` / `\renewcommand` |
| 展开宏 | `apply_macros()` | :54-58 | `re.sub(r"\\X\b", value, text)`；不递归展开 |
| 去图表噪声 | `FLOAT_RE` / `latex_to_text()` | :31,60-71 | 整个 `figure/table` 环境替换为空格；`\cite*`→`[CITE]`；`\ref{x}`→`x`；generic 命令保留首个花括号内容；`~`→空格；压缩空白 |
| 选区 | `SECTION_RE` / `extract_target_text()` | :26-29,73-90 | 默认扫 `abstract,introduction`；section 标题经 `latex_to_text().lower()` 后与 `--sections` 比较；未命中任何区块时退回整份文件（:88-89） |
| 分句 | `split_sentences()` | :92-96 | `(?<=[.!?])\s+(?=[A-Z(])`，丢弃长度 ≤20 字符的片段 |
| 主张判定 | `CLAIM_RE` | :15-24 | `we propose/introduce/...`、`our method/...`、`outperform/improve/...`、`first/novel/sota/robust/general/...` 等；`--all-sentences` 可绕过 |
| 输出 | `main()` | :102-136 | 打印 7 列表：`ID | Source | Section | Claim | Evidence | Status | Revision`；新行固定 `|  | needs evidence |  |`；表内 `|` 被转义（:98-99） |

推荐用法（先看候选，再人工补证）：

```bash
python3 scripts/extract_claims.py paper/main.tex --sections abstract,introduction > /tmp/claims.md
# 人工把 /tmp/claims.md 中与论文相关的行合并进 claim_evidence_map.md，
# 为每个 ID 填 Evidence（文件:行号 / 实验 ID / 已核验引用）与 Status。
```

**已知的语义边界**：`CLAIM_RE` 会漏掉不含触发词的隐含主张（例如纯描述性句子里的因果暗示），也会命中“we show in prior work”这类非本文贡献；宏展开不是递归的；分句规则假设句点后跟大写字母或 `(`，对缩写与 CJK 标点不友好。这些都不是失败，而是要求人工复核的理由。

### 3.2 状态词表与终态判定

- `paper-story.md:150-157` 定义四级主张强度：`strong / careful / aspirational / forbidden`。
- `citation-workflow.md:47-59` 定义七种引用关系；强主张只接受 `direct/contrast/software`。
- `research_quality_gate.py:54` 的 `OPEN_STATUS_RE = needs evidence|pending|unknown|unverified|placeholder|待核验|待补充`。
- `validate_claim_map()`（:236-250）**仅在终态**运行：要求表里有 `Status` 列（大小写不敏感，依赖 `markdown_rows()` 的小写化），凡状态命中 `OPEN_STATUS_RE` 就报 `Claim/evidence map has unresolved statuses`。

也就是说：`needs evidence` 作为**草稿中间态**完全合法；它只有在 `stage=complete/submission-ready` 时才阻塞。正确做法是补证据或弱化措辞，而不是把字符串改成 `Supported` 了事。

---

## 4. 链 D：上游交接与构建证明

### 4.1 handoff 是证据链的入口守卫

`check_research_handoff.py:14-33` 定义白名单：根字段 7 个；artifacts 允许 7 种；基础必需 `{project_inventory, analysis, decision}`，`quantitative=true` 时再加 `{experiment_inventory, numeric_evidence}`（:70-79）。`_artifact_path()`（:36-49）逐项要求：非空字符串、落在项目根内、文件存在、非空；文本后缀还要通过 UTF-8 读取；`numeric_evidence` 会直接调用 `load_registry()` 重算（:86-87）。

`blockers` 数组默认只**报告**；加 `--require-unblocked` 才导致退出码 1（:91-121）。这与 `SKILL.md:33-35` 的 Execution Contract 对应：正式全文写作前必须先过无 blocker 的 handoff。

### 4.2 构建摘要的实际算法

`paper_contract.py:122-153` 定义输入图：

```text
paper_input_files() =
    collect_tex_files(main_tex)      # \input/\include DFS 展开
  + resolve_graphics(main_tex)       # \includegraphics，后缀候选 "", .pdf, .png, .jpg, .jpeg, .eps, .svg
  + bibliography
  → 去重 → 按项目内相对路径排序

digest_files() = SHA-256( 对每个 path:
      relative.as_posix().encode() + b"\0" + file_bytes + b"\0"
   然后对每个 sorted(external_records):
      b"external\0" + record.encode() + b"\0" )
```

因此**修改文件名、移动文件、改外部模板 URL/哈希、换 TeX 工具版本**（executed attestation 时把 `{command, tool_version}` 作为额外 external record 加进摘要，`record_build.py:112-119`）都会使 `input_sha256` 失效。这是刻意设计：它证明“这份 PDF 对应这套确切输入与工具声明”。

`record_build.py:17-132` 的两条路径：

- `--run`：用 `shlex.split` 解析 `build.command`，**拒绝 shell 操作符**（`|`, `||`, `&&`, `;`, `<`, `>`, `>>`，:45-49），在 `main_tex.parent` 里 `subprocess.run(..., timeout=300)`；写日志（stdout/stderr 分节）、`log_sha256`、`exit_code`；失败时先把 `status=failed` 写回 state 再抛错（:70-76）；成功再探测 `<cmd> --version` 写入 `tool_version`，设 `attestation=executed` 与 UTC `ran_at`（:78-88）。
- 无 `--run`：只设 `attestation=artifact-only`，**不验证命令真的跑过**（:89-96）。`research_quality_gate.py:400-401` 因此禁止 `submission-ready` 使用 artifact-only。

还有一道时间检查（:98-107）：任何输入文件的 `st_mtime_ns` 晚于 PDF 就拒绝登记，防止“先写哈希后编译”。

### 4.3 总闸门对构建链的断言顺序

`validate_paper()` 的 build 部分（`research_quality_gate.py:385-436`）：

1. `build` 必须是 dict 且字段集合 ⊆ `BUILD_FIELDS`（:33-36,385-391）；
2. `status ∈ {not-run, blocked, failed, passed}`、`command` 非空（:392-395）；
3. `external_inputs` 每项恰好 `{name, source, archive_sha256}`，hash 必须 64 位小写 hex（:37,175-203,396）；
4. 终态要求 `status == passed`；`submission-ready` 额外要求 `attestation == executed`（:397-401）；
5. `executed` 要求 `exit_code == 0`、非空 log 路径、log 文件存在且 `log_sha256` 与当前文件一致（:402-411）；
6. `pdf` 非空存在；最后重算 `input_sha256` 与 `pdf_sha256` 并与 state 比对（:412-436）。

`tests/test_quality_tools.py:180,202,212,224` 分别对应用例：源码变更使摘要失效、PDF 早于源码、`submission-ready` 需要 executed、执行构建会留下日志。

---

## 5. 端到端反虚构工作流（可直接照跑）

下面的顺序把四条链接成一条流水线。每一步都要求“先有证据，后有文字”。

```bash
# 0) 若上游系统交接：先验证证据，再动论文
python3 scripts/check_research_handoff.py /path/to/handoff --require-unblocked

# 1) 声明契约（references/artifacts.md:9-28），尽早暴露缺件
python3 scripts/research_quality_gate.py /path/to/paper

# 2) 证据盘点：project_inventory.md / experiment_inventory.md（paper-story.md:22-40）
#    每条实验给一个稳定 ID，例如 E03-S42-R2

# 3) 从 Abstract/Introduction 抽出主张候选，人工补 Evidence/Status
python3 scripts/extract_claims.py paper/main.tex > /tmp/claims.md

# 4) 数值：从不可变原始日志生成 numeric_evidence.json v2
python3 scripts/check_numeric_evidence.py /path/to/paper

# 5) 引用：先 requests（逐句 claim_support），再在线核验，再离线校验
python3 scripts/verify_citations.py /path/to/paper --mailto you@example.org
python3 scripts/check_citation_lock.py /path/to/paper
python3 scripts/check_citations.py paper/main.tex paper/references.bib

# 6) 写作：先证据承载核心，后框架章节（workflow.md:24-44）
#    一个 numbered block 一次 agent 调用，只允许读 story + claim map + 证据 + 已完成章节

# 7) 静态卫生
python3 scripts/check_todos.py paper plan
python3 scripts/parse_build_log.py paper/main.log

# 8) 真实构建并登记（submission-ready 必须）
python3 scripts/record_build.py /path/to/paper --run

# 9) 总闸门 + 审稿自查
python3 scripts/research_quality_gate.py /path/to/paper
```

**状态机语义**：缺口出现时按 `SKILL.md:48-56` 处理——补证据 → 否则弱化主张 → 否则删除主张 → 否则记录 blocker 并保持非终态。绝不通过编辑正则、改状态字符串、放宽 tolerance 来“让检查变绿”。

---

## 6. 如何组织本地实验日志与代码，才能写出高质量论文

这一节是**工程建议**，不是脚本强制要求；它对应 `paper-story.md:54-66` 的 Repo Mining Checklist 与 `literature-review.md:9-19` 的语料工件思路。

### 6.1 推荐的论文项目布局

```text
paper-project/
├── paper_state.json                 # 机器状态：mode/stage/blockers/build
├── plan/
│   ├── task-packets/write-core.md   # 范围、允许编辑文件、验收命令
│   └── progress.md                  # 只写已完成事实，终态禁止 "待验证"
├── evidence/
│   ├── project_inventory.md         # 仓库地图：方法证据/实验证据/缺口
│   ├── experiment_inventory.md      # 每个实验：设置、指标、seeds、caveats
│   └── run_index.jsonl              # 每次运行的稳定索引（见 6.2）
├── experiments/
│   ├── raw/2026-07-16_E03_S42_R2.jsonl    # 不可变原始日志
│   ├── derived/summary.csv                # 由脚本从 raw 生成
│   └── scripts/aggregate.py               # 生成 derived 的唯一入口
├── numeric_evidence.json            # 只引用 derived 或 raw，selector 可重算
├── literature/
│   ├── paper_inventory.md
│   ├── related_work_matrix.md
│   ├── positioning.md
│   ├── notes/                       # close work / baseline 逐篇笔记
│   └── papers/                      # 仅在许可允许时保存 PDF
├── figures/
│   ├── figure_plan.md               # 每图 class/role/message/backend/fallback
│   ├── figure_specs.yaml            # figure-spec.md:9-34 的 YAML
│   └── *_prompt.md / *_plot.py      # 生成图的来源
├── paper/
│   ├── main.tex                     # \input 子文件；所有图路径项目内相对
│   └── references.bib
├── citation_requests.json
├── citation_lock.json
├── citation_verification.md
├── claim_evidence_map.md
├── paper_story.md
├── build_check.md
└── submission_readiness.md          # submission 模式必需
```

### 6.2 让日志可被“selector 化”的最小约定

数值链的输入质量决定了整篇论文的可靠性上限。建议每条运行记录至少包含：

```json
{"run_id":"E03-S42-R2","exp_id":"E03","seed":42,"repeat":2,
 "method":"ours","dataset":"bench-v2","split":"test",
 "metric":"accuracy","value":0.912,"ts_utc":"2026-07-16T09:12:33Z",
 "git_commit":"1f2a9c…","config_hash":"sha256:…","artifact":"experiments/raw/2026-07-16_E03_S42_R2.jsonl"}
```

为什么这样设计：

- `run_id` 稳定且唯一 → claim map 的 Evidence 列可以直接写 `E03-S42-R2`；
- 维度字段（`seed/method/dataset/split`）是**扁平顶层键** → `selector.where` 可以精确筛选，不需要嵌套 JSON Pointer；
- `git_commit`/`config_hash` → 让“哪版代码跑出来的”可追，回应 `reviewer-guidelines.md:82` 的可复现性检查；
- 原始日志**只追加不可变**；派生表格必须由 `experiments/scripts/aggregate.py` 生成 → 与 `figure-workflow.md:62-75` 的“源数据 + 可重生成脚本”一致；
- 负结果、混合结果、缺 seed 的运行**同样入库**，在 `experiment_inventory.md` 里记为范围约束（`SKILL.md:51-56`）；不要因为它们“不好看”而从 raw 里删掉。

### 6.3 从日志到论文的五条映射规则

| 论文元素 | 来源要求 | 对应脚本/规范 |
|---|---|---|
| 每个精确数字 | 必须有一个 `numeric_evidence.json` entry，selector 指向不可变源 | `check_numeric_evidence.py:203-258` |
| 每个强主张 | claim map 一行，Evidence 写 `E##-S##-R#` 或已核验 citation key | `paper-story.md:150-157`, `abstract-introduction.md:5-7` |
| 每个 close work | `literature/notes/` 有笔记后才可写进 Related Work | `literature-review.md:115-129` |
| 每张数据图/表 | 有 `figure_spec` + 源数据 + 生成脚本；图不产生新数字 | `figure-workflow.md:28-35`, `figure-spec.md:49-57` |
| 每个构建声明 | 命令、日志、版本、退出码、哈希一起登记 | `record_build.py:17-132` |

### 6.4 写作者最容易犯的五个错误

1. **在 LaTeX 里手改数字**：正文 `0.913`、注册表 `0.912` → 数值链断。数字只能从派生源复制。
2. **把 snippet 当证据**：引用只有 OpenAlex 搜索结果、没有 DOI/arXiv 主记录 → `metadata-only`，终态失败。
3. **把 `needs evidence` 留到最后一刻**：终态会扫描状态列；临时占位符必须当天清掉。
4. **用图像生成做结果图**：脚本抓不到图中数字，审稿人也无法核验；结果图必须确定性生成。
5. **先写完 Abstract 再补实验**：`workflow.md:24-44` 明确要求先写证据承载核心、最后重写标题和摘要；否则 Abstract 里的强主张会反复与证据冲突。

---

## 7. 边界、已知限制与维护不变量

- 脚本不会读论文语义：`validate_manuscript()` 只看 `UNRESOLVED_RE`/`PROCESS_RE` 与 TeX 输入图（`research_quality_gate.py:39-49,253-265`）。
- `check_citations.py` 的 `unused`（:29,41-42）只是提示，不影响退出码。
- `validate_verification_table()` 信任 `Status` 字符串，不重新联网核验；联网核验发生在 `verify_citations.py`，且锁会过期（默认 180 天）。
- `artifact-only` 构建可以通过 `complete`，但不能通过 `submission-ready`（:400-401）。
- `mode=skill` 与论文模式互斥：模式由 `--mode` 或“根目录有 `SKILL.md` 且存在 `references/`”推断（`research_quality_gate.py:450-452`）。
- 新增 selector kind / aggregate / relation / status 时，必须同时改：JSON schema、脚本常量、在线验证器、总闸门、测试、文档示例（`docs/modules_reference.md:145-151` 的维护清单）。

---

## 8. 锚点速查

| 主题 | 首选文件:行号 |
|---|---|
| 数值注册表与扫描 | `scripts/check_numeric_evidence.py:19-50,203-345` |
| 严格输入解析 | `scripts/paper_contract.py:12-160` |
| 引用请求/锁契约 | `scripts/citation_lock.py:15-178` |
| 在线 provider 验证 | `scripts/verify_citations.py:25-229` |
| 主张提取启发式 | `scripts/extract_claims.py:15-136` |
| handoff 校验 | `scripts/check_research_handoff.py:14-121` |
| 构建登记 | `scripts/record_build.py:17-132` |
| 总闸门 | `scripts/research_quality_gate.py:30-476` |
| 断言用例 | `tests/test_quality_tools.py:142-627` |
