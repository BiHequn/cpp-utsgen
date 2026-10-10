---
name: unit-test
description: 单测治理总线（迁移增强版）。从环境初始化→诊断→MC/DC 用例矩阵驱动→内环自愈→覆盖度评估（行/分支硬门禁）→变异测试质量评估（软门禁）→报告的全流程编排。核心流程语言无关，语言细节经 Language Pack（`assets/langpacks/<lang>.yaml`）数据驱动，支持语言由 `langpack_loader.py list` 动态发现（当前：c/cpp）。
---

# Unit Test — 单测治理总线（devflow 风格编排入口）

本 skill 是一条**确定性阶段总线**：Orchestrator 依次扮演「环境工程师 → 诊断分析师 → 方案规划师 → 自愈 Coder → 覆盖度审计员 → 变异质量评估员 → 报告作者」等角色，把「一个源文件/一批变更文件」经过 S0–S7 八个阶段治理为「达标的单元测试 + 质量报告」。

与传统单测生成器不同，本总线的核心增强点：

- **MC/DC 用例矩阵驱动**：方案阶段先用 `mcdc_matrix.py` 从布尔判定表达式推导「独立影响真值表」，再据此生成用例，而非盲目堆断言。
- **门禁分层**：行覆盖率 / 分支覆盖率为**硬门禁**（不达标回炉自愈或失败），MC/DC 判定覆盖与变异分数（MSI）为**软门禁**（不达标告警放行，可配置升级为硬阻断）。⚠️ **关键纠偏**：MSI 软门禁≠「不达标就直接过」。当 `gates.mutation.heal_loop: true`（默认）且存在**存活变异体**时，MSI 不达标**必须先走 S6→S4 变异专项自愈闭环**（补强断言、回炉重跑，最多 `mutation_heal_max_rounds` 轮），轮次耗尽仍有存活才按软门禁告警放行。**未进入自愈而直接 `--complete stage_6` 属违规**，由 `check_mutation_heal.py` / `--complete stage_6` 强制拦截。
- **内环自愈**：由确定性状态机 `heal_controller.py` 精确判断「继续 / 停滞熔断 / 达到上限 / 成功」，避免 AI 在数值判断上飘忽。
- **不 clone、主动装工具**：S0 直接在当前工作区探测环境并按需安装缺失的测试/变异工具，不强制 `git clone`。

> 阶段编号沿用脚本内部键名 `stage_0`…`stage_7`（对外称 S0…S7），由 `stage_gate.py` 强约束顺序。

---

## 🚫 核心红线（RED LINE，不可逾越）

> **测试全流程禁止修改任何被测源代码。** 这是本总线的**最高优先级硬约束**，凌驾于覆盖率/分支/变异等一切门禁之上。

- **允许**：新建 / 修改**测试文件**（如 Go `xxx_test.go`、PHP `tests/`）及纯测试辅助资源。
- **禁止**：以任何理由改动被测**源代码**（业务实现文件），包括但不限于：
  - ❌ 为了让测试通过而修改源码逻辑、删改分支、放宽校验；
  - ❌ 为了提升覆盖率 / 分支覆盖率 / 变异分数（MSI）而"顺手"重构源码；
  - ❌ 新增/删除源码文件、注释掉源码语句、调整源码可见性以便测试。
- **例外**：仅当**用户显式授权**「本次允许改源码」时才可放行，且须在报告中记录授权来源与改动清单；否则一律视为违规。
- **强制器**：由 `scripts/guard/source_guard.py` 落地——**S1 `snapshot`** 记录被测源码指纹，**S4 每轮**与 **S7 收尾** 执行 `verify`；一旦发现源码被修改/删除/新增，**exit 2 立即阻断**，要求还原源码后重跑，绝不放行。
- 源码 vs 测试文件的区分**完全数据驱动**（取自 langpack `files.source_ext` / `test_file_pattern` / `test_dir_convention` / `exclude_dirs`），语言无关。

> 若覆盖率/分支/变异因源码「不可测设计」无法达标，正确做法是：在**测试侧**用依赖注入/Mock/参数注入等手段解决，或在报告中如实标注并按软门禁放行——**而非修改源码**。

---

## 执行清单

> 每阶段**进入前**用 `stage_gate.py --check stage_N` 校验前置，**完成后**用 `stage_gate.py --complete stage_N --outputs <产物>` 标记。禁止跳过、合并或乱序执行。

| 阶段 | 名称 | 强制产出 | 门控前置 |
|------|------|----------|----------|
| **S0** | 环境初始化（**不 clone** + 主动装工具） | `env_probe.json` | — |
| **S1** | 工作区初始化（含**源码红线指纹快照**） | `WORKSPACE/`（含 `metadata/` `logs/` `artifacts/` + `task_info.json` + `progress_status.json` + `source_guard.json`） | S0 completed |
| **S2** | 智能诊断（AST 解析 + 布尔判定提取 + 测试发现） | `project_graph.json` | S1 completed |
| **S3** | 方案规划（依赖/返回值矩阵 + **MC/DC 用例矩阵驱动**） | `prd.json`（含 `mcdc_matrix[]`） | S2 completed |
| **S4** | 内环自愈（Coder ↔ Executor 循环） | 测试文件 + `heal_state.json` | S3 completed |
| **S5** | 覆盖度评估（行/分支**硬门禁** + MC/DC **软门禁**） | `coverage_report.json` + `heal_advice.json` | S4 completed |
| **S6** | 变异测试评估（**软门禁**；存活变异体按 `heal_loop` 回 S4 补强重跑） | `mutation_report.json` | S5 completed |
| **S7** | 报告与产物 | `final_report.json`（含 `gate_results[]` 与 `source_bugs[]`）+ `unit-test-report.md`（含「发现的源码 Bug」章节） | S6 completed |

**门控标记示例：**

```bash
# 进入 S2 前校验
python3 scripts/gate/stage_gate.py --check stage_2 --state-file WORKSPACE/metadata/progress_status.json
# 完成 S2 后标记
python3 scripts/gate/stage_gate.py --complete stage_2 --outputs "project_graph.json" \
  --state-file WORKSPACE/metadata/progress_status.json
```

---

## 用户意图解析

启动时先解析下列参数，缺省则采用默认值：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `TARGET_PATH` | 当前工作区（cwd） | 目标项目根目录。**本总线不 clone**，默认目标代码已在当前工作区。 |
| `SOURCE_FILE` | 无（可从 `CHANGED_FILES` 推导） | 单文件治理入口，指定要生成/补齐单测的源文件。 |
| `CHANGED_FILES` | 空 | JSON 数组，如 `["pkg/handler/user.go","pkg/service/order.go"]`。**非空即触发增量模式**（`SCOPE=incremental`），仅对变更文件评估覆盖率与变异。 |
| `SCOPE` | `incremental`（有 `CHANGED_FILES`）/ `full` | `incremental`=仅评估变更；`full`=对 `SOURCE_FILE`/目标目录做全量评估。 |
| `SCAN_MODE` | `single` | 扫描粒度：`single`=单文件；`branch`=全量分支扫描（批量发现待治理源文件）。 |

**子 Agent 模式（生产路径）**：当本 skill 被 devflow `test-engineer` 通过 `use_skill("unit-test")` 调用时，须传入以下参数：

| 参数 | 说明 |
|------|------|
| `CHANGED_FILES` | JSON 数组，变更的 Go/PHP 源文件列表 |
| `TARGET_PATH` | cwd（项目根目录），代码已在工作区，无需 clone |
| `SCOPE` | 固定 `incremental` |
| `ARTIFACTS_DIR` | devflow 产物目录，S7 拷贝最终报告至此 |
| `skill_root` | `.codebuddy/skills/unit-test/`，供脚本定位 assets/ |
| `UNIT_TEST_GATES_OVERRIDE` | （可选）devflow 级门禁覆写，含 `line.level`/`line.threshold`、`branch.level`/`branch.threshold`、`mcdc.level`/`mcdc.threshold`、`mutation.level`/`mutation.threshold`/`mutation.max_rounds`。传入后**优先于** `unit-test.defaults.yaml` 内置默认值。|

- `init_workspace.py --runtime-mode sub_agent --skill-root <path>` 初始化工作区并记录运行模式。
- **门禁覆写应用**：若传入了 `UNIT_TEST_GATES_OVERRIDE`，S5/S6 每次调用 `stage_gate.py --evaluate-gate <metric>` 时**必须**追加 `--level <覆盖level>` 和 `--threshold <覆盖threshold>`，以 devflow 配置为准。mutation 自愈的 `mutation_heal_max_rounds` 也使用覆盖值。
- S7 完成后将 `final_report.json` + `unit-test-report.md` 拷贝至 `$ARTIFACTS_DIR/04-e2e/unit-test/`。

---

## 脚本模块清单

所有脚本位于 `scripts/`，以 `python3` 调用，输出 JSON 到 stdout（门禁类脚本用 exit code 表达阻断）。

| 脚本 | 阶段 | 职责与关键调用 |
|------|------|----------------|
| `scripts/langpack/langpack_loader.py` | **所有阶段（语言抽象层唯一入口）** | Language Pack 加载器，实现"核心流程语言无关、语言细节数据驱动"：`list`（发现已注册语言）、`detect --source-file <f>`（扩展名反查语言）、`resolve --lang X --key <dotted.path> [--var k=v]`（取真实命令/正则/格式并做 `{var}` 替换）、`validate --lang X`（schema 校验，失败 exit 1）。合并 `include: path#anchor`（如 `mutation` 引用 `mutation-tools.yaml#<lang>`）。**新增语言只需加 `assets/langpacks/<lang>.yaml` + `knowledge/<lang>.md`，脚本与 stage 零改动。** |
| `scripts/langpack/yaml_lite.py` | **（被 loader 依赖）** | 零依赖 YAML 子集解析器（嵌套 map / 块序列 / flow list&map / 标量 / 注释），供 loader 读取 langpack，不引入第三方依赖。 |
| `scripts/coverage/format_parsers/` | **S5/S6** | 覆盖率**格式家族**分支解析器（`go_coverprofile_block` / `clover` 等）+ `get_branch_parser` 注册表。覆盖率按 langpack `coverage.branch_format`/`line_format` 声明的格式分派，**与语言解耦**；未知格式报 "unsupported format（需新增 parser）"（spec 合理例外）。 |
| `scripts/guard/source_guard.py` | **S1（snapshot）/ S4 每轮 / S7（verify）** | **源码防篡改守卫（红线强制器）**：`snapshot --root <TARGET_PATH> --language <lang> --output <source_guard.json> [--paths <csv>]` 记录被测**源码文件**（数据驱动排除测试文件/`exclude_dirs`/工作区）的 SHA-256 指纹；`verify --snapshot <source_guard.json> [--json]` 复核指纹，发现源码被**修改/删除/新增**即判红线违规、输出违规清单并 **exit 2**（阻断）。区分源码 vs 测试文件完全取自 langpack `files.*`，语言无关。 |
| `scripts/gate/stage_gate.py` | **所有阶段** | 阶段门控与门禁评估。`--init` 初始化状态、`--check stage_N` 校验前置、`--complete stage_N --outputs <...>` 标记完成（**`stage_6` 完成时内置变异自愈契约校验，违约 exit 2 拒绝**）、`--resume` 求续跑点、`--evaluate-gate <metric> --value <v> [--level] [--threshold] [--survived <n>] --skill-root --target-path` 统一评估门禁（变异门禁额外输出 `heal_required` 信号）。**阈值/层级自动从 `unit-test.defaults.yaml`（被项目级 `unit-test.config.yaml` 覆盖）读取**，命令行仅作临时覆盖；评估结果**回写 `progress_status.json.gate_results[]` 作 S7 SSOT**（硬门禁不达标 exit 1）。 |
| `scripts/gate/check_mutation_heal.py` | **S6** | **变异自愈契约核验器**（把 S6→S4 回炉从散文约束变成机器强制）：读 `mutation_report.json` + `heal_state.json`，判定「`heal_loop=true` 且存活>0 且 MSI<阈值」时是否存在自愈义务、且 `mutation_final_status` 已落到 `success`/`max_mutation_rounds`。`exit 0`=满足/无义务/合法降级（可完成），`exit 2`=违约（必须回 S4 补强后重回 S6，禁止完成）。阈值/`heal_loop`/轮次上限全部取 effective 配置，绝不硬编码。 |
| `scripts/workspace/init_workspace.py` | **S1** | 在 `TARGET_PATH` 下创建 `workspace/{task_id}/{metadata,logs,artifacts}`，写 `task_info.json`，解析 `--skill-root` 与 `--runtime-mode direct\|sub_agent`，并初始化 `progress_status.json`。 |
| `scripts/ast/parse_ast.py` | **S2** | 解析目标源文件 AST，提取函数签名、依赖、返回值分支与**布尔判定表达式**（供 S3 MC/DC 使用），产出 `project_graph.json`（可配合 `ast/cache_manager.py` 缓存）。 |
| `scripts/discovery/detect_test_files.py` | **S2** | 发现既有测试文件与命名/目录约定，判断补齐 vs 新建，回填 `project_graph.json`。**扩展名映射与测试命名/目录约定（`test_file_pattern`/`test_dir_convention`）优先取自 langpack `files.*`**（builtin 回退未迁移语言）。 |
| `scripts/mcdc/mcdc_matrix.py` | **S3** | 从布尔判定表达式生成 MC/DC 独立影响真值表（**tree-sitter AST 驱动**，已移除正则切分与 eval）：`--expr "<expr>" --language <lang> --decision-id <id> [--pretty]`。**AST profile（运算符/grammar/包裹模板/定位策略/节点类型）完全取自 langpack `ast.*`，grammar 经 `importlib` 动态加载，语言无关。** 输出 `conditions` / `required_pairs` / `target_cases` / `truth_table`。`!=`/`!` 等作完整原子保留、运算符优先级由 AST 天然表达（`precedence_warning` 恒 null）；条件数 >12 降级为条件覆盖（`degraded=true`）。首次运行触发 tree-sitter venv 自举（与 `parse_ast.py` 共用 `scripts/.venv`）。 |
| `scripts/heal/heal_controller.py` | **S4/S5/S6** | 内环自愈状态机：`init`、`decide --coverage <v> --test-status <...>`（continue/stagnated/max_iterations/success）、`coder-failed`；**`mutation-decide --survived <n> [--msi] [--threshold] --max-rounds <n>`** 驱动 S6→S4 变异专项自愈闭环（返回 continue_mutation/max_mutation_rounds/mutation_success）。**轮次语义已修正**：`mutation_round` 表示「已发起的补强重跑轮次」，仅在 `continue_mutation` 时 +1（首轮评估与 success/max 分支不消耗），故 `--max-rounds N` 恰为「最多回 S4 补强 N 次」；且 `mutation_round`/`mutation_final_status` 已作为 `heal_state.json` 正式字段持久化，回 S4 后不会被主内环 `decide`/`coder-failed` 的全量写回抹除（修复此前轮次归零/错乱问题）。维护 `heal_state.json`。 |
| `scripts/heal/build_heal_advice.py` | **S5/S6** | **自愈建议聚合器**：读 `coverage_report.json` + `mutation_report.json`，聚合 `uncovered_lines` / `uncovered_branches`(含 approx) / `uncovered_mcdc_pairs` / `surviving_mutants` + 可读 `advice[]`，产出 `heal_advice.json` 供 S4 Coder 聚焦补测（`heal_controller.py` 仍只做数值决策）。 |
| `scripts/heal/assert_validator.py` | **S4** | 校验生成用例的断言有效性（防止空断言 / 恒真断言 / 无意义断言）。 |
| `scripts/heal/verify_report.py` | **S4/S5** | 汇总本轮自愈执行结果，形成可读校验报告，供 `heal_controller.py decide` 消费。 |
| `scripts/coverage/extract_coverage.py` | **S4/S5** | 从测试运行的覆盖率原生输出（Go cover profile / PHP clover 等）提取**行覆盖率**结构化数据。**优先取工具原生口径**：Go 用 coverprofile 语句计数（numStmt，等同 `go tool cover -func`）、PHP 取 Clover `<metrics>`，不再自行按行累加。 |
| `scripts/coverage/extract_branch_coverage.py` | **S4/S5** | 提取**分支覆盖率**结构化数据。PHP Clover `type=cond` 为真分支（`approx=false`）；**Go 无原生分支覆盖，以 block(count>0) 近似并显式标注 `approx=true` / `mode=go_block`**。 |
| `scripts/coverage/calc_incremental_coverage.py` | **S5** | 按 `CHANGED_FILES`/变更行计算**增量覆盖率**（行/分支），产出 `coverage_report.json` 增量口径。 |
| `scripts/mutation/run_mutation.py` | **S6** | 驱动变异工具：`--language <lang> --target-dir <dir> [--changed-files ...] [--timeout-sec 600] [--skill-root\|--tools-yaml] [--language-version <v>]`。**`--auto-install` 时内部调用 `resolve_tool_version.py` 按当前语言版本选兼容安装命令**（如 Go `1.23.3` → `gremlins@v0.5.1`，避免 `@latest` 因 runtime 过旧编译失败）**再补装**。**消费 `mutation-tools.yaml` 的 `primary.run` 模板做变量替换**（`{report_json}`/`{target_dir}`/`{changed_files}`/`{timeout_args}`，空 changed_files/timeout_args 丢弃对应 token；`{timeout_args}` 由 `timeout_args_template`（flag 模板）+ `gates.mutation.per_language_timeout.<lang>`（系数/超时值，可项目级覆盖）拼出，即"每变异体超时窗口"，用于防误判 TIMED OUT 压低 MSI），argv[0] 用解析到的真实二进制替换。工具缺失→`status=skipped`（软降级，含 `install_note`），超时→`status=timeout`，成功→产出报告文件路径 + `tool_version` + 实际生效的 `timeout_args`。 |
| `scripts/mutation/resolve_tool_version.py` | **S0/S6** | 版本选择器：探测（或接受 `--language-version`）语言运行时版本，挑可安装且兼容的工具版本，输出 `install_cmd`/`tool_version`/`resolved_by`/`discovery_source`/`note`。**默认开启自动发现**（`mutation-tools.yaml` 的 `primary.discovery.auto_discover=true`）：在线查 Go 官方代理（解析各版本 `go.mod` 的 `go` 指令）或 Packagist（解析各版本 `require.php` 约束），实时选「当前 runtime 能编译的最高版本」，无需手维护兼容表；发现结果缓存于 `assets/.tool_cache/`（默认 1 天）并在 CI 离线时回退到 `primary.versions` 静态快照。`--no-discover` 仅用静态表；`--sync` 把最新发现结果写回 `mutation-tools.yaml` 的 `versions` 块（刷新离线快照）。S0 据此生成安装命令，S6 的 `run_mutation.py --auto-install` 据此补装。 |
| `scripts/mutation/parse_mutation.py` | **S6** | 解析变异工具报告（支持 `gremlins_json` / `infection_json` / `cargo_mutants_json` / `mull_json` / `mutmut_sqlite` / `pit_xml`），统一按 `msi = killed / (total - not_covered) × 100` 计算 MSI（跨语言可比），产出统一结构（含 `not_covered` / `error` / `tool_native_msi`）。**新增：按 per-mutant 记录聚合 `per_file[]`，配合 `--source-file` + `--scope file` 输出 `file_level_msi`/`package_level_msi` 双字段（解决包级冒充文件级 #3），并输出 `msi_status`（present/skipped_no_mutants/not_recorded，解决 null/0/N-A 歧义 #5）**。**go-mutesting 报告暂无对应 parser，若降级到 go-mutesting，则 mutation 阶段记为 `skipped`（软门禁放行）。** |
| `scripts/mutation/refocus_mutation.py` | **S6（补强轮）** | **变异自愈聚焦复验加速器**，解决「补强后又全量重跑变异、耗时过长」：`emit-scope --report <mutation_report.json>` 从存活变异体提取去重文件 CSV（供 `run_mutation.py --changed-files` 收窄范围）；`merge --baseline-report <累计> --refocus-parsed <本轮> [--refocus-files <csv>] [--round N] [--healed <json>] --output <累计>` 按**文件级替换**合并（基于「补强只改测试不改源码 ⇒ 变异体集合稳定、已杀死不复活」不变式：`total` 恒定、`killed` 单调增、`survived` 单调减、MSI 全局准确，`--msi-formula auto` 从基线反推口径保持与首轮一致），并**追加 `rounds[]` 逐轮记录**（口径与 headline MSI 一致）。仅在 `gates.mutation.focused_reverify: true`（默认）且 `MUTATION_ROUND>=1` 时启用。 |
| `scripts/mutation/build_mutation_report.py` | **S6** | **mutation_report.json 确定性组装器（schema_version=2）**：把 `run_mutation.py`（status/tool/scope）与 `parse_mutation.py`（msi/killed/…/per_file/双 MSI）合并为锁定的 v2 结构，**每轮追加 `rounds[]` 逐轮结构化数据**（解决逐轮数据只在 Markdown 自由文本 #2），透传 `file_level_msi`/`package_level_msi`/`msi_status`。取代 S6 手工拼装。 |
| `scripts/report/build_coverage_report.py` | **S5** | **coverage_report.json 确定性组装器（schema_version=2）**：从 `extract_coverage`/`calc_incremental_coverage`/`extract_branch_coverage` + `heal_state` 组装嵌套结构，每维显式 `status`（present/absent/na），回填 `line.best`/`line.last`（解决重测波动 #6），取代手工拼装（解决 schema 漂移 #1）。 |
| `scripts/report/build_final_report.py` | **S7** | **final_report.json 唯一权威聚合器（SSOT，schema_version=2）**：聚合 coverage/mutation/gate_results/heal_state，必含行/分支/MC-DC/MSI 全部指标 + `scope` + 逐轮序列 + `metrics_availability`（present/absent/na/skipped_no_mutants/not_recorded），`overall_passed` 依红线+硬门禁计算（解决产物分层倒置 #4）。 |
| `scripts/report/render_report.py` | **S7** | **unit-test-report.md 渲染器（视图层）**：从 `final_report.json`（SSOT）**纯渲染** Markdown，依 `metrics_availability` 显式区分 `N/A(=100)` / `-（未记录）` / `0`（解决 null/0/N-A 歧义 #5），不再手填数据。 |
| `scripts/validate/validate_report.py` | **S5/S6/S7** | **产物 schema 校验器（零依赖）**：`--kind coverage_report\|mutation_report\|heal_advice\|final_report --file <json>` 按 `assets/schemas/<kind>.schema.json` 强制校验，通过 exit 0、schema 违规 exit 3、入参错误 exit 1。产出时强制调用，杜绝脏 schema 流出（解决 #1/#4）。 |

---

## 配置加载

- **默认配置**：读取 `assets/unit-test.defaults.yaml`。
- **项目级覆盖**：若目标项目根（`TARGET_PATH`）下存在 `unit-test.config.yaml`，则以其内容**逐项覆盖**默认值（合并策略：项目级优先）。
- **变异工具映射**：语言→变异工具由 `assets/mutation-tools.yaml` 提供（Go 主 `gremlins` / 备 `go-mutesting`；PHP 主 `infection`；Java 主 `pit`；Python 主 `mutmut`；Rust 主 `cargo-mutants`；C/C++ 主 `mull`），S0 装工具、S6 跑变异均以此为准。**安装命令按当前语言版本选兼容版本，且默认自动发现、无需手维护兼容表**（仅 `go` / `php` 实现了在线版本发现）：`primary.discovery.auto_discover=true` 时，`resolve_tool_version.py` 在线查 Go 官方代理（各版本 `go.mod` 的 `go` 指令）或 Packagist（各版本 `require.php` 约束），据 `go/php` 版本（自动探测或 `--language-version`）选「当前 runtime 能编译的最高版本」，将 `install_template` 的 `{version}` 替换为选中版本；发现结果写入 `assets/.tool_cache/`（默认 1 天有效，CI 离线时回退到 `primary.versions` 静态快照）。其余语言（java/python/rust/c/cpp）未实现在线版本发现，一律走固定 `install` 命令（fixed_install），工具缺失时按 soft 门禁合法降级为 `skipped`。无兼容命中回退 `default_install`（`@latest`）并告警。维护者可用 `resolve_tool_version.py --sync --language <go|php>` 把最新发现结果写回 `versions` 块。⚠️ Go 禁用 `@latest`——新版本（如 gremlins `v0.6.0` 需 go≥1.25）在旧 runtime 会编译失败，必须走版本兼容解析。

**每变异体超时窗口（`per_language_timeout`）**：变异工具靠"重跑测试套件"杀死变异体，必须判断变异体是跑挂（`KILLED`）还是真卡死（`TIMED_OUT`）。窗口 = 基线耗时 × 系数（或固定秒/毫秒）。窗口太小会把"慢但本应 `KILLED`"的变异体误判成 `TIMED_OUT`，使 MSI 虚低（典型如 Go `gremlins` 内置系数仅 5）。本总线在 `mutation-tools.yaml` 用 `timeout_args_template` 声明各工具 flag、在 `gates.mutation.per_language_timeout.<lang>` 给足安全默认值（Go `coefficient:10` / Rust `coefficient:10`+`minimum_test_timeout:60` / Java `factor:2`+`constant:10000` / PHP `seconds:30` / C·C++ `ms:120000`；Python mutmut 无系数概念不设），可按项目级 `unit-test.config.yaml` 覆盖。若 `parse_mutation` 报告 `timeout_warning=true`（TIMED OUT 占比 >20% 或 ≥ KILLED 数），即疑似窗口过小误判，应调大对应系数后重跑。

**关键配置项：**

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `gates.coverage.line` | `hard@100`，`scope: incremental` | 行覆盖率**硬门禁**，增量口径 100%。 |
| `gates.coverage.branch` | `hard@100`，`scope: incremental`，`measure: always` | 分支覆盖率**硬门禁**，增量口径 100%；`measure: always` 表示每轮默认产出分支覆盖并纳入报告（解决 branch 空缺显示 `-` #5/#6）。 |
| `gates.coverage.mcdc` | `soft@100` | MC/DC 判定对覆盖**软门禁**，建议 100%。 |
| `gates.mutation` | `soft`，`msi_threshold: 80`，`incremental_only: true`，`enforce_file_level_msi: true`，`timeout_sec: 600`，`per_language_timeout`（见下），`heal_loop: true`，`mutation_heal_max_rounds: 3`，`focused_reverify: true` | 变异分数**软门禁**，建议 MSI≥80%，默认仅跑增量、单文件 600s 熔断；`enforce_file_level_msi: true` 时单文件治理门禁一律取 `file_level_msi`（禁止包级冒充文件级 #3）；`per_language_timeout` 为**每变异体超时窗口**（防误判 TIMED OUT 压低 MSI，见下）；`heal_loop` 开启**存活变异体回 S4 内环**补强并重跑，`mutation_heal_max_rounds` 为变异专项自愈轮次上限（防死循环）；`focused_reverify` 开启时补强轮**仅对存活变异体所在文件聚焦复验**并合并累计报告（大幅提速），关闭则每轮全量重跑。 |
| `tools.auto_install` | `true` | S0 是否自动安装缺失工具。 |
| `languages` | 动态发现 | **支持语言不再硬编码**：由 `scripts/langpack/langpack_loader.py list` 从 `assets/langpacks/*.yaml` 动态发现（当前 c/cpp）。新增语言 = 加 `langpacks/<lang>.yaml` + `knowledge/<lang>.md`（+ 按需装 tree-sitter grammar），核心脚本与 stage 零改动。 |

---

## 门禁分层

本总线将质量门禁分为两层，统一由 `stage_gate.py --evaluate-gate` 评估。**阈值与层级默认从 `unit-test.defaults.yaml`（被项目级 `unit-test.config.yaml` 覆盖）自动读取**，命令行 `--level`/`--threshold` 仅作临时覆盖；每次评估结果**自动回写 `progress_status.json.gate_results[]` 作为 S7 单一数据源（SSOT）**。硬门禁不达标返回 **exit 1**（阻断），软门禁不达标仅告警（`passed=true`、`warning=true`）：

| 门禁 | 层级 | 阈值 | 不达标处理 |
|------|------|------|-----------|
| **源码红线 source_guard** | **红线（最高）** | 源码 0 改动 | ⛔ 检出源码被修改/删除/新增即 **exit 2 立即阻断**，凌驾所有其他门禁：必须还原源码、仅在测试侧解决问题后重跑；除非用户显式授权改源码。 |
| 行覆盖率 line | **硬** | 增量 100% | ⛔ 回 **S4 自愈**（携 `heal_advice.json`）重试；自愈耗尽仍不达标→阶段失败。**严禁通过改源码达标（触红线）。** |
| 分支覆盖率 branch | **硬** | 增量 100% | ⛔ 回 **S4 自愈**重试；耗尽仍不达标→阶段失败。**Go 为 block 近似（`approx=true`）**，PHP Clover cond 为真分支。 |
| MC/DC 判定覆盖 mcdc | 软 | 100% | ⚠️ 告警放行；可将 `gates.coverage.mcdc.level` 改为 `hard` 升级为阻断。 |
| 变异分数 MSI | 软 | ≥80%（**取 effective 配置 `msi_threshold`，可能为 100 等，禁止硬编码**） | ⚠️ 软门禁≠直接放行。阈值与层级从 `unit-test.defaults.yaml`（被项目级 `unit-test.config.yaml` 覆盖）读取。**`gates.mutation.heal_loop: true`（默认）且存在存活变异体时，MSI 不达标必须回 S4 补强并重跑**（`mutation_heal_max_rounds` 上限，耗尽软放行；`focused_reverify: true` 时补强轮仅聚焦复验存活文件以提速）。`--evaluate-gate mutation` 会额外输出 `heal_required` 信号：`true` 即**必须进入 S6→S4 自愈闭环，禁止直接完成**；该契约由 `check_mutation_heal.py` 及 `--complete stage_6` 强制校验（违约 exit 2）。工具缺失/超时按 `skipped`/`timeout` 记录不阻断；可将 `gates.mutation.level` 改为 `hard` 升级为阻断。 |

**统一评估调用示例（阈值/层级读配置，结果回写 SSOT）：**

```bash
# 硬门禁：分支覆盖率（不传 --level/--threshold → 读配置 hard@100）
python3 scripts/gate/stage_gate.py --evaluate-gate branch --value 87.5 \
  --skill-root "${SKILL_ROOT}" --target-path "${TARGET_PATH}" \
  --state-file WORKSPACE/metadata/progress_status.json
# 未达标 exit 1（阻断，触发回 S4）；结果回写 gate_results[]

# 软门禁：变异分数（读配置 soft@80，阈值取 msi_threshold，绝不硬编码）
python3 scripts/gate/stage_gate.py --evaluate-gate mutation --value 72 --survived 7 \
  --skill-root "${SKILL_ROOT}" --target-path "${TARGET_PATH}" \
  --state-file WORKSPACE/metadata/progress_status.json
# 输出含 heal_required：true → 必须进入 S6→S4 自愈闭环（Step 4.5/4.6），禁止直接完成
#                         false/none → 无存活或达标，可正常推进
```

---

## 分步执行约束

为保证进度可见、可审计、可续跑，执行时遵守以下约束：

- **先播报后执行**：每进入一个阶段，**先**在正文输出进度播报（当前阶段号/名称/将执行的动作/预期产物），**再**调用对应工具脚本。
- **禁止静默连跑**：⛔ 禁止在无播报的情况下连续执行多个重型步骤（AST 解析、自愈循环、覆盖率评估、变异测试等），每个重型步骤前必须有可见播报。
- **门控贴合**：进入前 `--check`、完成后 `--complete`，异常时用 `--resume` 求续跑点，保证阶段状态与产物一致。
- **源码红线贴合**：S1 必须 `source_guard.py snapshot`；S4 每轮生成/修改测试后、`decide` 前必须 `source_guard.py verify`；S7 收尾再 `verify` 一次并写入报告。任一 `verify` 返回 exit 2 即停止推进，先还原源码再重跑，**禁止带着源码改动继续任何后续阶段**。
- **子 Agent 一致性**：sub_agent 模式下同样遵守上述播报、门控与**源码红线**约束，并在阶段切换时输出结构化完成标记，便于上层编排消费。
