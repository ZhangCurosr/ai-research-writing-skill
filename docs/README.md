# AI Research Writing Skill — AI 维护者入口

> 本文面向零上下文 AI（或新的人类维护者）。目标：读完本套文档后，能够在**不猜测**的前提下维护、扩展和路由这个技能包。
>
> 事实来源：本仓库源码。文中 `路径:行号` 是本检出版本的定位锚点；修改源码后请同步更新这些锚点。

## 1. 它是什么

`ai-research-writing-skill` **不是一个运行时 Python 应用**，而是一个给 coding agent 使用的**声明式技能包（agent skill）**：把 ML/AI 研究仓库、实验日志、笔记和投稿模板，转化为可审计的 LaTeX 论文与投稿包。

它的核心命题写在 [`SKILL.md:8-13`](../SKILL.md)：

> Treat paper writing as claim-evidence engineering, not prose generation.

也就是说：**论文写作被建模为「主张—证据—验证」的工程链路**，而不是让模型凭记忆生成流畅段落。

仓库根目录的 [`SKILL.md`](../SKILL.md) 是**唯一规范入口**（README.md:163-174）。其余目录都是按需加载的支撑材料：

| 目录/文件 | 角色 | 是否运行时执行 |
|---|---|---|
| `SKILL.md` | 入口契约：mandate、路由表、Execution Contract、Gates | 由 agent 读取 |
| `references/` | 分层知识：工作流、产物契约、写作、引用、图、投稿 | 由 agent 按需读取 |
| `scripts/` | 确定性检查器（**Python 标准库**，无第三方依赖） | 由 agent/人类命令行调用 |
| `templates/` | 官方模板的审计清单（不 vendored 作者套件） | `fetch_template.py` 读取 |
| `examples/` | 用本技能写出的自指论文完整样例 | 供参照、CI 验证 |
| `tests/` | 质量工具的 unittest 套件 | CI / 本地回归 |

参考文件之间的**加载地图**在 [`references/README.md:5-22`](../references/README.md)，而不是靠数字编号。新增/移动参考文件时必须同步更新该地图和 `SKILL.md` 的路由表。

## 2. 设计哲学

### 2.1 主张—证据工程（claim-evidence engineering）

- 每个实质性主张必须能追到仓库文件、实验产物、笔记或已核验引用（`SKILL.md:8-13`）。
- 精确数字必须能追到数据、日志、表格、notebook 或脚本；定量论文维护 `numeric_evidence.json` v2 注册表（`SKILL.md:39-42`）。
- 负面、混合、缺种子、不完整的结果被当作**范围约束**，不得隐藏（`SKILL.md:51-56`）。
- 证据不足时：**弱化主张、删除主张、记为 blocker**；不得留下看似完成的表述。

### 2.2 渐进式披露（progressive disclosure）

`SKILL.md:15-27` 明确规定「选择覆盖请求的最小路由，只读列出的参考」。例如：

| 用户意图 | 首读文件 |
|---|---|
| 完整论文 / 大改 | `references/workflow.md`, `references/artifacts.md` |
| 上游研究系统交接 | `references/research-handoff.md`, `references/artifacts.md` |
| story / 贡献边界 | `references/paper-story.md` |
| 章节写作 | `references/README.md` → 对应章节参考 |
| 图与表 | `references/figure-workflow.md`, `references/figure-spec.md` |
| 引用查找/修复 | `references/citation-workflow.md` |
| 审稿 | `references/reviewer-guidelines.md` |
| 投稿 / camera-ready | `references/submission-packaging.md`, `references/citation-checklist.md` |
| 确定性检查 | `scripts/README.md` |

这是**上下文预算设计**：不要一次性加载整个 `references/` 树（`references/README.md:44-50`）。

### 2.3 语义决策与确定性执行分离

agent 负责 story、措辞、引用关系判断；**脚本负责机械契约**：

- `paper_contract.py` 严格解析 LaTeX 输入图、图形、citation key、JSON、哈希；
- `check_numeric_evidence.py` 从 CSV/JSON/JSONL 重新计算数字；
- `citation_lock.py` + `verify_citations.py` 做元数据与摘要锁定；
- `research_quality_gate.py` 做模式感知（mode-aware）的总闸门。

脚本**只执行机械检查**，不替代科学判断、语义引用审查、官方投稿规则或真实 LaTeX 构建（`scripts/README.md:23-24`）。

### 2.4 人工判断保留在环内

- 科学正确性、图像目视检查、审稿风险、最终投稿决定由人负责（`SKILL.md:11-13`）。
- 审稿模拟必须标明是模拟，不得伪造真实审稿人意见（`references/reviewer-guidelines.md:5-10`）。
- 高分风险未解决的条目可以**阻止 submission-ready**，但不阻止诚实地完成一份审稿报告（`references/reviewer-self-review.md:98-109`）。

### 2.5 产物边界

**论文产物属于目标论文项目，不属于技能仓库**（`references/artifacts.md:3`、`references/README.md:50`）。维护技能时不要在 `references/` 下生成 `paper_story.md`；只有在修改技能本身时才改技能仓库。

## 3. 五道不可跳过的闸门

| 闸门 | 约束 | 代码/规范位置 |
|---|---|---|
| Evidence | 数字可重算；注册表 v2；未登记数字即失败 | `scripts/check_numeric_evidence.py:19-48,203-307` |
| Story | 明确 thesis/gap/contribution 后才写长文 | `references/workflow.md:18-22`, `references/paper-story.md` |
| Literature | Related Work 前完成文献定位 | `references/workflow.md:21`, `references/literature-review.md` |
| Citation | 学术引用需 `citation_requests.json` + 新鲜 `citation_lock.json`；逐句支持关系 | `scripts/citation_lock.py:15-22,91-178` |
| Figures | 概念图走图像生成；数值图/表格必须确定性生成 | `references/figure-workflow.md:24-45` |
| Review | 高风险意见必须解决或记为 blocker | `SKILL.md:46`, `references/reviewer-self-review.md` |
| Build | 投稿前真实编译并记录命令、日志、版本、退出码、哈希 | `scripts/record_build.py:38-121` |
| Completion | 必需产物与检查通过前不得说 “done” | `SKILL.md:58`, `references/task-management.md:36-44` |

闸门失败时的正确动作：**弱化主张 / 补证据 / 记录 blocker**，而不是绕过检查。

## 4. 状态与构建边界

机器状态 `paper_state.json` 是全链路的单一事实源（`references/artifacts.md:5-37`、`references/paper-state.schema.json`）：

- `mode`: `draft | full-paper | submission`（质量门内部另有 `skill` 模式，见 `research_quality_gate.py:31-32`）；
- `stage`: 自由字符串，但 `complete` / `submission-ready` 是终态（`research_quality_gate.py:38`）；
- `blockers`: 非空数组会阻止终态（`research_quality_gate.py:315-319`）；
- `build`: 状态、命令、PDF、外部输入、输入/PDF 哈希、attestation 等。

**注意**：脚本是严格模式——未知字段、缺失声明输入、空文件、非 UTF-8 文本都会报错，而不是静默跳过（`paper_contract.py:27-35,112-119`）。维护扩展时必须保持这个 fail-closed 属性。

## 5. 配套阅读顺序（给零上下文 AI）

1. [`docs/architecture.md`](architecture.md)：证据链绑定、状态机、防幻觉机制。
2. [`docs/pipeline_and_dataflow.md`](pipeline_and_dataflow.md)：从输入到投稿包的阶段与数据流。
3. [`docs/modules_reference.md`](modules_reference.md)：参考文件、Prompt 规范、脚本 API。
4. [`docs/execution_guide.md`](execution_guide.md)：在各类 AI 编码助手中的操作方式、命令与排障。

### 5.1 源码级深度参考（新增）

| 文档 | 回答什么问题 | 何时加载 |
|---|---|---|
| [`docs/evidence_chain_deep_dive.md`](evidence_chain_deep_dive.md) | 数字 / 引用 / 主张 / 构建四条证据链的逐行算法、正则、摘要链；如何组织实验日志与代码以提高论文质量 | 要建立或修复证据链、设计本地实验目录时 |
| [`docs/prompts_and_gate_reference.md`](prompts_and_gate_reference.md) | `references/` 全部内置模板与占位符清单；`research_quality_gate.py` 全部检查规则、断言顺序与退出码 | 要写故事/章节/图/审稿文本，或维护闸门规则时 |
| [`docs/troubleshooting.md`](troubleshooting.md) | 证据链断裂、LaTeX 编译与格式不兼容、单步写作幻觉阻断的报错→根因→修复流程 | 任一检查器失败、或写作流程被闸门阻断时 |

这三篇是**扩充**而非替代：先读上面的 4 篇建立全貌，再按需深入；`architecture.md` 讲机制，`evidence_chain_deep_dive.md` 讲实现细节。

## 6. 仓库事实（便于核验）

- 许可证：MIT（[`LICENSE`](../LICENSE)）。
- CI：`.github/workflows/quality.yml:10-41` 在 Python 3.10–3.13 上执行 `unittest discover`、`py_compile`、技能包质量门、样例质量门、citation 与 TODO 检查。
- 脚本全部只用 Python 标准库（`scripts/README.md:3`）；没有 `requirements.txt`。
- 模板清单 `templates/manifest.json` 记录官方来源、SHA-256、许可证审计状态；当前 pinned 下载覆盖 ICML 2026、ICLR 2026、CVPR 2026、ICCV 2025、ACL rolling style（`templates/README.md:19-22`）。
- 自指样例：`examples/paper-about-ai-research-writing-skill/`，其 `paper_state.json` 展示了 `complete` 阶段的 filled-in 契约。

## 7. 深度参考的维护约定

- 深度文档中的 `路径:行号` 锚点必须与当前检出一致；改脚本后先 grep 新行号再更新三篇文档。
- 三篇深度文档只引用**已存在**的文件与函数；规范中的意图若未落到代码，要明确标注为“规范层/人工判断”，不得写成脚本保证。
- 新增脚本或正则白名单时，同步更新 [`prompts_and_gate_reference.md`](prompts_and_gate_reference.md) §3.7 与 [`troubleshooting.md`](troubleshooting.md) §4，并补 `tests/test_quality_tools.py` 用例。
