# 工具模块参考

> 本文件是 `SKILL.md` 的工具参考附件，包含各工具脚本的详细用法、参数说明和输出格式。
> 在治理流程各阶段需要调用工具时，通过 `read_file` 加载本文件获取详细参考。

---

## 数据协议

各模块通过 `metadata/` 目录下的 JSON 文件传递结构化数据：

| 数据文件 | 写入者 | 读取者 | 说明 |
|---------|-------|-------|------|
| `task_info.json` | init_workspace.py | 所有角色 | 任务元信息、路径 |
| `env_probe.json` | S0 环境工程师 | 所有角色 | 语言/工具探测结果 |
| `project_graph.json` | S2 诊断分析师 | S3 方案架构师, S4 Coder | AST 分析结果、依赖图、decisions[] |
| `prd.json` | S3 方案架构师 | S4 Coder, S5 审计员 | MC/DC 矩阵 + 测试方案 + Mock 策略 |
| `test_plan.md` | S3 方案架构师 | S4 Coder | 可读测试计划 |
| `coverage_line.json` | extract_coverage.py | S4 Coder, S5 审计员 | 行覆盖率数据 |
| `coverage_report.json` | **build_coverage_report.py（S5）** | S6 变异评估员, S7 报告作者 | 三维覆盖率汇总（**schema_version=2 嵌套结构**，每维 status） |
| `mutation_report.json` | **build_mutation_report.py / refocus merge（S6）** | S7 报告作者 | 变异分数 MSI（**v2**，含 rounds[]/文件级双 MSI/msi_status） |
| `heal_state.json` | heal_controller.py | S4 Coder, S5 审计员, S6 变异评估员 | 内环状态机（含 coverage_history_clean / mutation_round / mutation_history） |
| `heal_advice.json` | build_heal_advice.py | S4 Coder（回炉时） | 未覆盖点 + 存活变异体补测建议（无 coverage 字段） |
| `final_report.json` | **build_final_report.py（S7）** | test-engineer（上层编排） | **唯一 SSOT**：全指标 + metrics_availability + 逐轮序列 + gate_results[] |
| `progress_status.json` | init_workspace.py / stage_gate.py | 所有角色 | 阶段进度状态、运行模式、断点续传信息 |

---

## 阶段门控控制器 (`scripts/gate/stage_gate.py`)

强制阶段门控校验脚本，确保各阶段按顺序执行，禁止跳过或合并阶段。

**初始化**：
```bash
python3 scripts/gate/stage_gate.py --init \
    --state-file <WORKSPACE>/metadata/progress_status.json \
    --runtime-mode <direct|sub_agent> \
    --skill-root <SKILL_ROOT> \
    --workspace-path <WORKSPACE>
```

**阶段前置检查**（exit code 0=通过, 1=阻断）：
```bash
python3 scripts/gate/stage_gate.py --check <stage_id> \
    --state-file <WORKSPACE>/metadata/progress_status.json
```

**标记阶段完成**：
```bash
python3 scripts/gate/stage_gate.py --complete <stage_id> \
    --outputs "file1.json,file2.json" \
    --state-file <WORKSPACE>/metadata/progress_status.json
```

**标记失败/跳过**（可选）：
```bash
python3 scripts/gate/stage_gate.py --complete <stage_id> \
    --outputs "file.json" --status failed \
    --state-file <WORKSPACE>/metadata/progress_status.json
```

**断点续传查询**：
```bash
python3 scripts/gate/stage_gate.py --resume \
    --state-file <WORKSPACE>/metadata/progress_status.json
```

**门禁评估（硬/软分层）**：
```bash
# 硬门禁：分支覆盖率（不传 --level/--threshold → 读配置 hard@100）
python3 scripts/gate/stage_gate.py --evaluate-gate branch --value 87.5 \
  --skill-root "${SKILL_ROOT}" --target-path "${TARGET_PATH}" \
  --state-file WORKSPACE/metadata/progress_status.json
# 未达标 exit 1（阻断）；结果回写 gate_results[] SSOT

# 软门禁：变异分数（读配置 soft@80）
python3 scripts/gate/stage_gate.py --evaluate-gate mutation --value 72 \
  --skill-root "${SKILL_ROOT}" --target-path "${TARGET_PATH}" \
  --state-file WORKSPACE/metadata/progress_status.json
# 恒 exit 0；未达标 passed=true, warning=true
```

**stage_id 枚举**：`stage_0` / `stage_1` / `stage_2` / `stage_3` / `stage_4` / `stage_5` / `stage_6` / `stage_7`

**`progress_status.json` 结构**：
```json
{
  "runtime_mode": "direct|sub_agent",
  "skill_root": "/absolute/path",
  "workspace_path": "/absolute/path",
  "task_id": "task_...",
  "stages": {
    "stage_0": {"stage_id": "stage_0", "status": "completed", "timestamp": "...", "outputs": ["env_probe.json"]}
  },
  "gate_results": []
}
```

**status 枚举**：`completed` / `failed` / `skipped`

**门禁 metric 枚举**：`line` / `branch` / `mcdc` / `mutation`

---

## AST 分析工具 (`scripts/ast/parse_ast.py`)

基于 Tree-sitter 的多语言 AST 解析器（支持 Go/PHP/Python/Java/C/C++），零语言分支设计（规则外部化在 `scripts/ast/queries/*.scm`）。

```bash
python3 scripts/ast/parse_ast.py --file <源文件路径> [--output json|text] [--language auto] [--skill-root <SKILL_ROOT>]
```

**输出字段**：`functions[]`（名称、行范围、参数、返回值、external_calls、complexity、is_exported）、`imports[]`、`structs[]`、`complexity_total`、`decisions[]`（布尔判定表达式，供 S3 MC/DC 矩阵使用）

**附属工具**：`cache_manager.py`（AST 缓存管理）、`queries/*.scm`（各语言查询规则）

**注意**：脚本首次运行会自动创建 `.venv` 并安装依赖（tree-sitter 系列），对用户系统零侵入。**必须调用本脚本执行 AST 分析**，依赖缺失时禁止降级为手动分析，必须解决依赖问题后重试。仅在脚本执行超时、崩溃（segfault）或输出格式解析失败时才允许降级为手动分析。

---

## MC/DC 矩阵生成器 (`scripts/mcdc/mcdc_matrix.py`)

从布尔判定表达式生成 MC/DC 独立影响真值表（tree-sitter AST 驱动）。

```bash
python3 scripts/mcdc/mcdc_matrix.py \
    --expr "<表达式>" --language <go|php> --decision-id <ID> \
    [--pretty]
```

**输出字段**：`conditions[]` / `required_pairs[]` / `target_cases[]` / `truth_table[]` / `precedence_warning`（AST 驱动后恒为 null）

**降级规则**：条件数 > 12 时降级为条件覆盖（`degraded=true`），避免 2^n 真值表爆炸。

---

## 覆盖率提取工具

### extract_coverage.py（行覆盖率）

从测试执行结果中提取行覆盖率数据。**优先取工具原生口径**：Go 用 coverprofile 语句计数（numStmt）、PHP 取 Clover `<metrics>`。

```bash
python3 scripts/coverage/extract_coverage.py \
    --language <LANGUAGE> --coverage-file <覆盖率文件> --source-file <源文件> \
    [--skill-root <SKILL_ROOT>] [--pretty]
```

**输出字段**：`coverage_percentage`、`total_lines`、`covered_lines`、`missing_lines[]`

### extract_branch_coverage.py（分支覆盖率）

提取分支覆盖率。PHP Clover `type=cond` 为真分支（`approx=false`）；Go 以 coverprofile block(count>0) 近似并标注 `approx=true` / `mode=go_block`。

```bash
python3 scripts/coverage/extract_branch_coverage.py \
    --language <LANGUAGE> --coverage-file <覆盖率文件> --source-file <源文件> \
    [--pretty]
```

**输出字段**：`branch_coverage`、`total_branches`、`covered_branches`、`uncovered_branches[]`（含 `approx` 标记）

### calc_incremental_coverage.py（增量覆盖率）

```bash
python3 scripts/coverage/calc_incremental_coverage.py \
    --coverage-file <覆盖率文件> --diff-functions '<JSON数组>' \
    --language <LANGUAGE> --output <输出JSON路径> [--skill-root <SKILL_ROOT>]
```

按 `CHANGED_FILES`/变更行计算增量覆盖率（行/分支），产出 `coverage_report.json` 增量口径。

---

## 自愈控制器 (`scripts/heal/heal_controller.py`)

确定性的内环循环控制器，负责轮次管理、停滞检测、熔断判断。

**初始化**：
```bash
python3 scripts/heal/heal_controller.py init \
    --state-file <WORKSPACE>/metadata/heal_state.json \
    --target-coverage <目标> --max-iterations 30
```
可选参数：`--min-coverage-delta 1.0`、`--low-coverage-threshold 50.0`、`--low-coverage-min-delta 2.0`、`--stagnation-tolerance 3`、`--initial-coverage 0.0`

**每轮决策**：
```bash
python3 scripts/heal/heal_controller.py decide \
    --state-file <state_file> --coverage <值> --test-status <success|failed|error|timeout|oom> \
    [--progress-file <progress_status.json>] [--skill-root <SKILL_ROOT>]
```

**Coder 失败时**：
```bash
python3 scripts/heal/heal_controller.py coder-failed --state-file <state_file>
```

**变异专项自愈决策**（S6→S4 闭环）：
```bash
python3 scripts/heal/heal_controller.py mutation-decide \
    --state-file <state_file> --survived <N> --msi <分值> --threshold <阈值> --max-rounds <上限>
```

**decision 枚举**：

| 值 | 含义 | 动作 |
|---|---|---|
| `continue` | 继续内环 | 根据 `mode` 调用 Coder |
| `success` | 覆盖率达标+测试全通过 | 进入 S5 |
| `stagnated` | 覆盖率停滞 | 进入 S5（硬门禁裁决） |
| `max_iterations` | 达到最大轮次 | 进入 S5（硬门禁裁决） |
| `continue_mutation` | 变异未达标 | 回 S4 补强断言 |
| `max_mutation_rounds` | 变异轮次耗尽 | 软门禁放行→S7 |
| `mutation_success` | 变异达标 | 进入 S7 |

**mode 枚举**（仅 decision=continue 时有意义）：

| 值 | 含义 |
|---|---|
| `error_fix` | 测试失败/编译错误 |
| `heal` | 测试通过但覆盖率不足 |

---

## 断言验证器 (`scripts/heal/assert_validator.py`)

验证测试代码中断言质量，检测虚假断言模式（空断言、恒真断言、无意义断言）。

```bash
python3 scripts/heal/assert_validator.py \
    --test-file <测试文件> --language <LANGUAGE> \
    --output <输出JSON路径> --skill-root <SKILL_ROOT>
```

---

## 自愈建议构建器 (`scripts/heal/build_heal_advice.py`)

聚合未覆盖清单为结构化自愈建议。

```bash
python3 scripts/heal/build_heal_advice.py \
    --coverage-report <coverage_report.json> \
    [--mutation-report <mutation_report.json>] \
    --source-file <源文件> --scope <incremental|full> --round <轮次> \
    --output <heal_advice.json>
```

**输出字段**：`uncovered_lines[]` / `uncovered_branches[]` / `uncovered_mcdc_pairs[]` / `surviving_mutants[]` / `advice[]`（可读补测指引）

---

## 报告验证器 (`scripts/heal/verify_report.py`)

汇总本轮自愈执行结果，形成可读校验报告。

```bash
python3 scripts/heal/verify_report.py \
    --report-file <execution_report.json> --write-back [--skill-root <SKILL_ROOT>]
```

---

## 变异测试工具集 (`scripts/mutation/`)

### run_mutation.py

驱动变异测试工具，支持缺失补装、超时熔断与降级。

```bash
# 全量
python3 scripts/mutation/run_mutation.py \
    --language <go|php> --target-dir <目录> \
    --report-json <输出路径> --timeout-sec 600 \
    [--skill-root <SKILL_ROOT>]

# 增量（传变更文件 CSV）
python3 scripts/mutation/run_mutation.py \
    --language <LANGUAGE> --target-dir <目录> \
    --changed-files "pkg/a.go,pkg/b.go" \
    --report-json <输出路径> --timeout-sec 600 \
    --skill-root <SKILL_ROOT> --auto-install
```

**status 枚举**：`completed` / `completed_with_warnings` / `skipped` / `timeout`

### parse_mutation.py

解析变异工具报告，计算 MSI，并按 per-mutant 记录聚合 `per_file[]`。

```bash
python3 scripts/mutation/parse_mutation.py \
    --format <gremlins_json|infection_json|...> \
    --report-file <变异报告路径> \
    --scope file --source-file <被测源文件> \
    --threshold 80 [--pretty]
```

**新增字段**：`per_file`（按文件聚合）/ `file_level_msi` / `package_level_msi`（`--scope file --source-file` 时）/ `msi_status`（`present` / `skipped_no_mutants`=无可变异点 / `not_recorded`=真实缺口）。scope=file 时 `msi` 即文件级 MSI。

### build_mutation_report.py（mutation_report.json 组装器，schema_version=2）

把 run/parse 结果组装为 v2 结构并逐轮追加 `rounds[]`。

```bash
python3 scripts/mutation/build_mutation_report.py \
    --parsed-report <mutation_parsed.json> --run-status <run_mutation 输出> \
    --heal-state <heal_state.json> --round <N> --scope file --source-file <源文件> \
    --output <mutation_report.json>
```

### resolve_tool_version.py

按语言运行时版本选择兼容的变异工具版本（默认在线自动发现）。

```bash
python3 scripts/mutation/resolve_tool_version.py \
    --language <go|php> [--skill-root <SKILL_ROOT>] [--no-discover]
```

### refocus_mutation.py

变异自愈聚焦复验加速器：提取存活变异体文件列表 → 收窄范围重跑 → 合并累计报告。

```bash
# 提取存活文件范围
python3 scripts/mutation/refocus_mutation.py emit-scope --report <mutation_report.json>

# 合并聚焦复验结果（追加 rounds[]）
python3 scripts/mutation/refocus_mutation.py merge \
    --baseline-report <累计报告> --refocus-parsed <本轮解析结果> \
    --refocus-files <文件CSV> --round <N> --healed '["L294"]' --output <累计报告>
```

---

## 产物组装与校验（schema_version=2 SSOT）

> 关键产物统一由**确定性组装器**产出、由**校验器**强制校验，取代 AI 手工拼装（根治 schema 漂移）。

### build_coverage_report.py（S5）

从 `extract_coverage` / `calc_incremental_coverage` / `extract_branch_coverage` + `heal_state`
组装 `coverage_report.json`（v2 嵌套结构，每维 `status`，回填 `line.best`/`last`）。

```bash
python3 scripts/report/build_coverage_report.py \
    --source-file <源文件> --scope <file|package|incremental> \
    --incremental-report <incremental_coverage.json> --line-report <coverage_line.json> \
    --branch-report <coverage_branch.json> --heal-state <heal_state.json> \
    --mcdc-coverage <值|na> --mcdc-applicable <true|false> \
    --output <coverage_report.json>
```

### build_final_report.py（S7，唯一 SSOT）

聚合 coverage/mutation/gate_results/heal_state → `final_report.json`（含全指标 + `metrics_availability` + 逐轮序列）。

```bash
python3 scripts/report/build_final_report.py \
    --task-id <ID> --language <lang> --scope <scope> \
    --coverage-report <...> --mutation-report <...> \
    --progress-status <progress_status.json> --heal-state <heal_state.json> \
    --source-guard <source_guard_final.json> --test-files '<JSON数组>' \
    --output <final_report.json>
```

### render_report.py（S7，视图层）

从 `final_report.json`（SSOT）**纯渲染** `unit-test-report.md`，依 `metrics_availability` 区分 `N/A(=100)` / `-` / `0`。

```bash
python3 scripts/report/render_report.py \
    --final-report <final_report.json> --run-mode <direct|sub_agent> \
    --output <unit-test-report.md>
```

### validate_report.py（S5/S6/S7 强制校验）

按 `assets/schemas/<kind>.schema.json` 校验产物，schema 违规 exit 3 阻断。

```bash
python3 scripts/validate/validate_report.py \
    --kind <coverage_report|mutation_report|heal_advice|final_report> \
    --file <json> --skill-root <SKILL_ROOT>
```

---

## 测试发现工具 (`scripts/discovery/detect_test_files.py`)

多语言测试文件检测与路径推荐。

```bash
# 为源文件推荐测试路径
python3 scripts/discovery/detect_test_files.py \
    --project-root <项目根目录> --source-file <源文件路径> --recommend \
    [--skill-root <SKILL_ROOT>]

# 扫描整个项目测试结构
python3 scripts/discovery/detect_test_files.py \
    --project-root <项目根目录> --scan --language <LANGUAGE> \
    [--skill-root <SKILL_ROOT>]
```

**输出字段**：`--recommend` 输出 `recommended_test_file`、`existing_test_files[]`、`mode: append|create`。

---

## 工作区初始化 (`scripts/workspace/init_workspace.py`)

在目标项目根目录下创建标准 WORKSPACE 骨架。

```bash
# 单文件模式（默认）
python3 scripts/workspace/init_workspace.py \
    --project-root <项目绝对路径> --source-file <源文件相对路径> \
    --target-coverage <目标覆盖率> [--target-function <函数名>] \
    [--skill-root <SKILL_ROOT>] [--runtime-mode direct|sub_agent]

# 全量分支扫描模式
python3 scripts/workspace/init_workspace.py \
    --project-root <项目绝对路径> --batch-mode \
    --target-coverage <目标覆盖率> \
    [--skill-root <SKILL_ROOT>] [--runtime-mode direct|sub_agent]
```

**参数说明**：
- `--skill-root`：Skill 根目录绝对路径。sub_agent 模式下由 test-engineer 传入
- `--runtime-mode`：`direct` / `sub_agent`。不指定时若传入 `--skill-root` 则自动判定为 `sub_agent`
- `--batch-mode`：全量分支扫描模式

**输出字段**：`task_id`、`project_root`、`workspace`（含 metadata/、logs/、artifacts/ 子目录）、`skill_root`、`runtime_mode`

**附加行为**：初始化时自动创建 `progress_status.json`，写入 `runtime_mode`、`skill_root`、`workspace_path` 等元信息。幂等：workspace 已存在时清理重建。

---

## 环境检测参考

### 语言与依赖声明文件

| 语言 | 判定特征 | 包管理器 | 覆盖率工具 |
|------|---------|---------|-----------|
| Go | `go.mod` | `go mod` | `go test -cover`（内置） |
| PHP | `composer.json` | `composer` | `phpunit` + `pcov` 或 `xdebug` |
| Python | `requirements.txt` 等 | `pip` | `pytest-cov` |
| Java | `pom.xml` | Maven | JaCoCo |
| C/C++ | `CMakeLists.txt` | CMake | `gcov` + `lcov` |

### 变异工具映射

| 语言 | 主工具 | 备用工具 | 解析格式 |
|------|--------|---------|---------|
| Go | gremlins | go-mutesting | `gremlins_json` |
| PHP | infection | — | `infection_json` |

详见 `assets/mutation-tools.yaml`。安装命令按当前语言运行时版本选择兼容版本（`resolve_tool_version.py` 自动发现），禁止盲目 `@latest`。

### 覆盖率工具安装

| 语言 | 安装命令 | 备注 |
|------|---------|------|
| Go | —（内置） | `go test -coverprofile` |
| PHP | `pecl install pcov` | pcov 优先，其次 xdebug |

### 处理规则

- 已安装 → ✅ 已就绪
- 缺失 + `auto_install: true` → 🔧 自动安装
- 安装失败 → ❌ 记录到 `tools_missing[]`（不阻断流程，S6 降级）
- `auto_install: false` → 仅探测不安装，缺失工具全部记入 `tools_missing[]`
