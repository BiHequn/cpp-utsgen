# 报告规范

> 本文件是 `SKILL.md` 的报告规范附件，包含 `final_report.json` 的完整 JSON 结构、字段说明、status 枚举、评分标准和 BadCase 归档规范。
> 在阶段七生成报告时，通过 `read_file` 加载本文件获取详细参考。

---

## 播报参考

| 阶段 | 内容 | 耗时 |
|------|------|------|
| 0-初始化 | 语言检测+环境预装+**覆盖率工具检测与安装**+Rules扫描+依赖检查+**环境分析结果播报** | 1-3分钟 |
| 1-工作区 | 创建目录结构 | <30秒 |
| 1.5-扫描 | 全量分支扫描+文件过滤+优先级排序（仅 `SCAN_MODE=branch`） | 1-5分钟 |
| 2-诊断 | AST解析+依赖分析+测试发现 | 1-3分钟 |
| 3-方案 | Mock策略+用例规划 | 1-3分钟 |
| 4-内环 | Coder+Executor | 2-5分钟/轮 |
| 5-审计 | 代码质量审查 | 1-2分钟 |
| 6-增量 | 增量覆盖率最终验证（可选） | <1分钟 |
| 7-发布 | commit+push+报告+**全量分支扫描汇总**+增量覆盖率摘要+**质量门禁指标摘要**+清理 | 1-2分钟 |

---

## `final_report.json` 结构

治理结束后（无论成功或失败），生成 `<WORKSPACE>/artifacts/final_report.json`。

```json
{
  "task_id": "string",
  "source_file": "string",
  "language": "string",
  "status": "string",                // 见 status 枚举表
  "duration_seconds": 0,
  "coverage": {
    "baseline": 0.0, "final": 0.0, "delta": 0.0,
    "target": 0.0, "target_reached": false,
    "incremental": {                  // 仅 SCOPE=incremental 时存在
      "line_coverage": 0.0, "line_total": 0, "line_covered": 0, "line_missing": [],
      "function_coverage": 0.0, "function_total": 0, "function_covered": 0, "function_missing": [],
      "target_line_coverage": 100, "target_reached": false
    }
  },
  "iterations": { "inner_loop_rounds": 0, "outer_loop_rounds": 0, "total_heal_attempts": 0 },
  "test_file": { "path": "string", "is_new": true, "test_count": 0 },
  "quality": { "review_verdict": "string", "review_score": 0, "pardoned": false, "pardon_reason": null },
  "errors_encountered": [],           // 治理过程中遇到的错误
  "badcases": [],                     // 典型问题归档
  "batch_summary": {                  // 仅 SCAN_MODE=branch 时存在
    "total_files": 0, "success_files": 0, "failed_files": 0, "skipped_files": 0,
    "overall_coverage": { "baseline": 0.0, "final": 0.0, "delta": 0.0 },
    "total_test_cases": 0, "total_test_files": 0,
    "file_results": [{ "source_file": "string", "status": "string", "coverage": {}, "test_count": 0, "quality_gate_passed": true }]
  },
  "quality_gate": {                   // 各指标阈值详见 §约束-7
    "passed": true,
    "test_pass_rate": { "value": 0.0, "total": 0, "passed": 0, "failed": 0, "source_bug_failures": 0, "source_bug_details": [], "target": 100.0, "reached": true },
    "incremental_function_coverage": { "value": 0.0, "total": 0, "covered": 0, "missing": [], "target": 100.0, "reached": true },
    "incremental_line_coverage": { "value": 0.0, "total": 0, "covered": 0, "missing": [], "target": 100.0, "reached": false },
    "line_coverage": { "value": 0.0, "total": 0, "covered": 0, "missing": [], "target": 100.0, "reached": false }
  },
  "source_bugs": [],                   // 治理过程中发现的源码 bug 清单（见下方 §source_bugs 字段说明；无则空数组，禁止删除该字段）
  "created_at": "ISO8601"
}
```

## `quality_gate` 字段说明

| 模式 | 存在的字段 | `passed` 判定规则 |
|------|-----------|-----------------|
| 全量（`SCOPE=full`） | `test_pass_rate` + `line_coverage` | 两项 `reached` 的 AND |
| 增量（`SCOPE=incremental`） | `test_pass_rate` + `incremental_function_coverage` + `incremental_line_coverage` | 三项 `reached` 的 AND |
| 分支扫描（`SCAN_MODE=branch`） | 同上（按文件独立校验），整体 `passed` = 所有文件门禁的 AND | 任一文件未通过则整体 `false` |

- **源代码 bug 豁免**：`source_bug_failures` > 0 时，这些用例不计入 `failed` 和 `total`，`value`/`reached` 基于排除后数据计算
- 增量覆盖率计算失败时，`incremental` 字段设为 `null`，并在 `errors_encountered` 中记录原因

## `source_bugs` 字段说明（发现的源码 Bug）

> 单测治理过程本身就是发现源码缺陷的高效手段。本字段是**顶层独立数组**，用于集中展示治理中发现的**源码 bug**（区别于测试侧/环境侧问题）。**无论是否有发现，该字段都必须存在**（无发现则为 `[]`），禁止整段删除，以便上层编排/CI 可靠解析。

**捕获来源（`discovered_by`）**：

| 来源 | 触发情形 |
|------|---------|
| `test_failure` | 用例失败根因定位为源码逻辑/边界/异常缺陷（对应 `quality_gate.test_pass_rate.source_bug_failures`） |
| `mutation` | 存活变异体暴露真实逻辑错误（非单纯断言偏弱），经分析确认是源码 bug |
| `code_review` | S5 代码质量审查 / S7 收尾复核发现的源码缺陷 |
| `mcdc` | MC/DC 用例矩阵暴露真实分支/判定逻辑与预期不符 |
| `dependency_contract` | `prd.json` 的 `dependency_return_matrix[].business_defense = missing`（漏防御，潜在 bug） |

**`source_bugs[]` 每项结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 唯一标识，格式 `SB-<NNN>`（如 `SB-001`） |
| `title` | string | 简短描述 |
| `file` | string | 缺陷所在源码相对路径 |
| `line` | int | 缺陷所在行号（无精确行号填 `0`） |
| `severity` | string | 严重度：`high` / `medium` / `low` |
| `category` | string | 类别枚举：`logic`（逻辑错误）/ `boundary`（边界错误）/ `npe`（空指针/空值）/ `concurrency`（并发）/ `error_handling`（异常处理）/ `contract_violation`（依赖契约违背）/ `state_management`（状态管理）/ `other` |
| `discovered_by` | string | 来源，取上表枚举值 |
| `evidence` | string | 发现依据（失败用例名 / 存活变异算子+行号 / 审查点 / `dependency_return_matrix` 条目等） |
| `impact` | string | 业务影响说明 |
| `suggested_fix` | string | 建议修复方式 |
| `status` | string | 状态：`reported`（已上报）/ `confirmed`（用户确认）/ `fixed_by_user`（用户已修）/ `wont_fix`（暂不修复） |
| `related_test_failures` | int | 该 bug 直接导致的失败用例数（关联并核对 `quality_gate.test_pass_rate.source_bug_failures`） |

```json
{
  "source_bugs": [
    {
      "id": "SB-001",
      "title": "订单金额边界比较缺失，超阈值未拦截",
      "file": "pkg/service/order.go",
      "line": 42,
      "severity": "high",
      "category": "boundary",
      "discovered_by": "mutation",
      "evidence": "存活变异 CONDITIONALS_BOUNDARY @ order.go:42，>= 改为 > 测试仍通过，确为逻辑缺陷",
      "impact": "金额为阈值临界值时漏拦截，可能产生资损",
      "suggested_fix": "将 `amount >= limit` 改为正确的边界比较逻辑并补充临界用例",
      "status": "reported",
      "related_test_failures": 1
    }
  ]
}
```

> 与 `quality_gate.test_pass_rate.source_bug_details` 的关系：后者仅记录「导致用例失败、并据此豁免门禁」的源码 bug 子集；`source_bugs` 是**完整发现清单**（含未触发失败但经审查/变异/契约确认的真实缺陷），二者互补、不冲突。

## status 枚举与生成时机

| status | 说明 | 触发条件 |
|--------|------|---------|
| `success` | 覆盖率达标且评审通过 | Reviewer approve |
| `pardoned` | 覆盖率未达标但特赦通过 | Reviewer pardon |
| `stagnated` | 覆盖率停滞 | 内环 Delta ≈ 0 |
| `max_iterations` | 达到最大自愈轮数 | 超过最大自愈次数 |
| `failed` | 治理失败 | 不可恢复错误 |
| `partial_success` | 部分文件成功（仅 `SCAN_MODE=branch`） | 批量执行中部分文件成功、部分文件失败 |
| `timeout` | 超时熔断 | 超过时间限制 |

报告必须在治理流程的**最后一步**生成，无论成功或失败。

## review_score 评分标准

| 分数 | 含义 |
|------|------|
| 9-10 | 优秀：深度断言、真实 Mock、边界覆盖充分 |
| 7-8 | 良好：核心路径覆盖、断言合理 |
| 5-6 | 及格：基本覆盖但存在浅层断言 |
| 1-4 | 不合格：虚假断言、无效 Mock |

## BadCase 归档

典型问题记录到 `artifacts/badcases.json`，每条包含 `category`、`description`、`context`、`resolution`、`timestamp` 字段。

**分类枚举**：`environment`（环境问题）、`mock_complexity`（Mock复杂度）、`code_unreachable`（代码不可达）、`plan_defect`（方案缺陷）、`tool_limitation`（工具限制）、`source_bug`（源代码bug）。

`errors_encountered` 记录治理过程中的错误，`badcases` 同时写入独立文件便于跨任务聚合。

---

## 数据文件规范

### `project_deps.json` 关键字段

| 字段 | 说明 |
|------|------|
| `language` | 项目语言 |
| `runtime_version` | 运行时版本 |
| `dependency_file` | 依赖声明文件路径 |
| `lock_file` | Lock 文件路径（如有） |
| `dependencies[]` | 依赖列表，每条含 `name` + `version` + `is_test` |
| `test_framework` | 测试框架信息，含 `declared` + `name` + `version` |
| `coverage_tool` | 覆盖率工具信息，含 `name` + `available` + `installed_by_skill` |
| `warnings[]` | 解析过程中的警告信息 |

### `prd.json` 中 `dependency_return_matrix[]` 字段规范（§约束-12，强制产出）

每个外部依赖调用对应一条记录，由方案架构师在阶段三 Step 3.1 输出。语言中立的字段结构如下：

```json
{
  "dependency_return_matrix": [
    {
      "dependency_call": "<依赖调用表达式，使用项目源代码语言的语法>",
      "evidence_chain": {
        "doc_annotation": "<PHPDoc/Javadoc/Go doc/Python docstring 中的返回类型与异常声明，含源码行号；无注释填 'none'>",
        "type_declaration": "<函数签名静态类型声明；无类型填 'none (untyped)'>",
        "all_returns": [
          "<file:line> return <expr> [in try|catch|if|else|loop block]",
          "<file:line> return <expr> ..."
        ],
        "callee_chain": [
          "→ <callee_name> (<file:line>) ├─ <返回形态摘要含来源行号>",
          "  → <更深层调用> (<file:line>) ..."
        ],
        "knowledge_table_hit": "<knowledge/<LANGUAGE>.md 速查表条目原文 | 'not in cheatsheet'>"
      },
      "trace_depth": 2,
      "reach_native_boundary": true,
      "trace_status": "complete | shallow",
      "trace_stop_reason": "<reach_native_boundary=false 时必填，说明为何停止追溯>",
      "return_space": ["<形态1>", "<形态2>", "<错误标记>"],
      "business_defense": "correct | missing | partial | n_a",
      "must_test_forms": ["<必测形态1>", "<必测形态2>"],
      "mapped_cases": ["<test_cases[].case_name 1>", "<test_cases[].case_name 2>"]
    }
  ]
}
```

跨语言示例（语义一致，语法因语言而异）：

| 语言 | `dependency_call` 示例 | `evidence_chain.callee_chain` 示例 | `return_space` 示例 |
|------|----------------------|-----------------------------------|--------------------|
| PHP | `$esClient->updateByQuery()` | `→ execWriteRequest (ESClient.php:489) ├─ catch returns false (line 528) ├─ try returns $result (line 506)` | `["array{updated:int}", "[]", "false"]` |
| Go | `userRepo.FindByID(ctx, id)` | `→ db.QueryRow (database/sql) ├─ ErrNoRows path` | `["*User", "(nil, ErrNotFound)", "(nil, error)"]` |
| Java | `userRepository.findById(id)` | `→ JpaRepository.findById ├─ Optional.empty when row not found` | `["Optional.of(user)", "Optional.empty()", "throws DataAccessException"]` |
| Python | `cache.get(key)` | `→ redis.client.execute_command ├─ ConnectionError on socket timeout` | `["bytes", "None", "raises ConnectionError"]` |
| TypeScript | `httpClient.fetch(url)` | `→ fetch (native) ├─ rejects on network error` | `["Response", "null", "rejects with NetworkError"]` |
| C | `fopen(path, mode)` | `native libc — no further callee` | `["FILE*", "NULL"]` |

| 字段 | 说明 |
|------|------|
| `dependency_call` | 依赖调用表达式（含调用者，使用项目源代码语言的语法） |
| `evidence_chain` | **§约束-12 强制产出，5 个 key 缺一不可**。每项缺失视为推导不完整，阶段五审计将一票否决（`return_path_incomplete` / `trace_too_shallow` / `dependency_evidence_missing`）。详见下表。 |
| `trace_depth` | 整数，已追溯到第几层调用（直接函数体=1，1 层子方法=2，依此类推） |
| `reach_native_boundary` | bool，是否追到 SDK/框架/语言原生函数边界。`false` 时 `trace_stop_reason` 必填 |
| `trace_status` | `complete`（已追到原生边界） / `shallow`（停留在中间层，必须给 `trace_stop_reason`） |
| `return_space` | 真实返回类型空间数组（从 `evidence_chain` 推导，与业务代码无关） |
| `business_defense` | 业务对各形态的防御情况，枚举：`correct`（防御正确）/ `missing`（漏防御，潜在 bug）/ `partial`（部分防御）/ `n_a`（无须防御） |
| `must_test_forms` | 必测形态枚举数组，对应 `return_space` 中需覆盖的所有真实可达形态 |
| `mapped_cases` | 每个必测形态对应的 `test_cases[].case_name` 列表（一一对应） |

#### `evidence_chain` 5 个 key 详解

| key | 类型 | 说明 | 缺失/不规范的处置 |
|-----|------|------|----------------|
| `doc_annotation` | string | PHPDoc `@return`/`@throws`、Javadoc、Go doc、Python docstring 中的返回类型与异常声明（含源码行号）。无注释填 `"none"` | 缺失 → `dependency_evidence_missing` 警告 |
| `type_declaration` | string | 函数签名静态类型声明（PHP `: ?array`、Go `(*User, error)`、TS `Promise<T \| null>`）。无类型填 `"none (untyped)"` | 缺失 → 同上 |
| `all_returns` | string[] | 函数体内**所有** `return` 语句的列表（含源码行号），每条标注其所在控制块（try/catch/if/else/loop） | 缺失或 < 实际 return 数 → 一票否决（`return_path_incomplete`） |
| `callee_chain` | string[] | 函数体内调用的其它自定义方法/函数列表，每个 callee 必须递归追到 native/SDK 边界并标注其返回形态（含源码行号）。函数体确实无内部调用时填 `["no internal call"]` | 函数体内有自定义方法调用但 `callee_chain` 为空或停留在第一层 → 一票否决（`trace_too_shallow`） |
| `knowledge_table_hit` | string | 是否命中 `knowledge/<LANGUAGE>.md` 速查表对应条目。命中填速查表条目原文，未命中填 `"not in cheatsheet"` | 命中但形态未在 `return_space` 中体现 → 一票否决（`cheatsheet_ignored`） |

> ⚠️ **禁止**通过业务代码中的"存在性/类型/非空判断"等防御性表达式反推 `return_space`，必须从 `evidence_chain` **正向推导**。各语言对应的防御性表达式示例详见 `knowledge/general.md` 「依赖契约推导原则」。
>
> ⚠️ **禁止**只读依赖方法的"直接函数体"就结束推导。典型反模式：`updateByQuery()` 函数体只有 `return $result;`，但其调用的 `execWriteRequest()` 内部 `catch{ return false; }` 才是 false 形态的真实来源。`callee_chain` 必须穿透这一层。参见 `knowledge/php.md` BadCase #54。

### `prd.json` 中 `cpp_build_config` 字段规范

仅 `LANGUAGE` 为 `c` 或 `cpp` 时必须包含：

```json
{
  "cpp_build_config": {
    "source_file": "被测源文件路径（如 src/user.c）",
    "include_headers": [
      "被测源文件对应的真实头文件路径（如 include/user.h）"
    ],
    "compile_command_template": "gcc -fprofile-arcs -ftest-coverage -I<include_dir> -o <test_binary> <test_file> <source_file> (C 项目用 gcc，C++ 项目用 g++)",
    "header_resolution_strategy": "has_header | include_source | extern_declare"
  }
}
```

- `source_file`：被测源文件相对路径
- `include_headers[]`：真实头文件路径（无头文件时填被测源文件路径）
- `compile_command_template`：编译命令模板（必须同时含测试文件和被测源文件）
- `header_resolution_strategy`：`has_header`/`include_source`/`extern_declare`

---

## 迁移增强版新增数据结构

> 本章节为「单测治理总线（迁移增强版）」新增的数据结构规范，覆盖 MC/DC 用例矩阵、覆盖率报告、变异测试报告与最终门禁汇总四类 JSON。各结构均由对应脚本产出，供阶段间流转与最终报告汇总使用。

### 1. `prd.json` 的 `mcdc_matrix[]`

来源：`scripts/mcdc/mcdc_matrix.py` 对被测源文件中每个布尔判定（decision）分析后的输出汇总。方案阶段将其写入 `prd.json` 的 `mcdc_matrix[]` 数组，作为用例规划的驱动依据（每个 `required_pair` 对应一组必测用例）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `decision_id` | string | 判定唯一标识（如 `dec_1`），同一文件内唯一 |
| `expr` | string | 判定表达式原文（归一化后的布尔表达式，如 `a && (b \|\| c)`） |
| `language` | string | 判定所在源代码语言（`go` / `php` / ...） |
| `conditions[]` | string[] | 判定中所有原子条件（去重后的叶子条件列表，如 `["a", "b", "c"]`） |
| `required_pairs[]` | object[] | MC/DC 独立影响对，每项含：`condition`（被验证的条件）/ `case_true`（该条件为真的取值向量）/ `case_false`（该条件为假的取值向量）/ `outcome_true`（case_true 对应的判定结果）/ `outcome_false`（case_false 对应的判定结果）。二者仅在目标 `condition` 上取值不同，且判定结果相反，从而证明该条件对结果的独立影响 |
| `target_cases[]` | string[] | 覆盖本判定所有 `required_pairs` 所需的**最小**测试向量集合（去重后；典型 n+1、最坏 2n，**非** 2^n 完整真值表），用于映射到 `test_cases[].case_name` |
| `precedence_warning` | string（可选） | 当源语言存在低优先级关键字运算符（如 PHP 的 `and/or/xor/not`）与 `&&/\|\|` 无括号混用、归一化后优先级可能与原意不一致时填入告警说明；无风险时可省略 |

```json
{
  "mcdc_matrix": [
    {
      "decision_id": "dec_1",
      "expr": "a && (b || c)",
      "language": "go",
      "conditions": ["a", "b", "c"],
      "required_pairs": [
        {
          "condition": "a",
          "case_true":  { "a": true,  "b": true, "c": false },
          "case_false": { "a": false, "b": true, "c": false },
          "outcome_true": true,
          "outcome_false": false
        },
        {
          "condition": "b",
          "case_true":  { "a": true, "b": true,  "c": false },
          "case_false": { "a": true, "b": false, "c": false },
          "outcome_true": true,
          "outcome_false": false
        },
        {
          "condition": "c",
          "case_true":  { "a": true, "b": false, "c": true },
          "case_false": { "a": true, "b": false, "c": false },
          "outcome_true": true,
          "outcome_false": false
        }
      ],
      "target_cases": [
        "a=true,b=true,c=false",
        "a=false,b=true,c=false",
        "a=true,b=false,c=true",
        "a=true,b=false,c=false"
      ],
      "precedence_warning": null
    }
  ]
}
```

### 2. `coverage_report.json`（schema_version 2）

> ⚠️ **schema 已锁定为 v2（嵌套结构）**：废弃旧的顶层平面字段 `coverage_percentage`，统一为
> `line` / `branch` / `mcdc` 三段嵌套，每段显式携带 `status`（`present`/`absent`/`na`）。由
> `scripts/report/build_coverage_report.py` **确定性组装**（不再手工拼装），并由
> `scripts/validate/validate_report.py --kind coverage_report` 强制校验，fail 阻断。

统一的覆盖率报告结构。`line` 段来自 `extract_coverage.py`（或增量口径 `calc_incremental_coverage.py`）；`branch` 段来自 `extract_branch_coverage.py`（Go 用 coverprofile block 近似分支 `approx=true`/`mode=go_block`，PHP 用 Clover cond 真分支 `approx=false`）；`mcdc` 段由 S5 统计。

| 字段 | 类型 | 说明 |
|------|------|------|
| `schema_version` | int | 固定 `2` |
| `source_file` | string | 被测源文件 |
| `scope` | enum | `file` / `package` / `incremental` |
| `line.coverage_percentage` | number\|null | 行覆盖率百分比（`null`+`status=absent` 表示未测量） |
| `line.total_statements` / `line.covered_statements` | int | 语句总数 / 已覆盖语句数（工具原生口径） |
| `line.uncovered_lines[]` | array | 未覆盖行号/行对象列表 |
| `line.best` / `line.last` | number | 逐轮 clean 序列的**历史最优** / **最近值**（门禁取 best，规避重测波动回落 #6） |
| `line.status` | enum | `present` / `absent` / `na` |
| `branch.branch_coverage` | number\|null | 分支覆盖率百分比 |
| `branch.total_branches` / `branch.covered_branches` | int | 分支总数 / 已覆盖分支数 |
| `branch.uncovered_branches[]` | array | 未覆盖分支列表（Go block 每项带 `approx`） |
| `branch.approx` / `branch.mode` | bool / string | 是否近似 / 近似模式（Go=`go_block`） |
| `branch.status` | enum | `present` / `absent` / `na` |
| `mcdc.mcdc_coverage` | number\|null | MC/DC 覆盖率；**不适用时为 `null` 且 `status=na`** |
| `mcdc.applicable` | bool | 该文件是否存在布尔判定（`false`→`na`，等价 100%，非缺数据） |
| `mcdc.uncovered_mcdc_pairs[]` | array | 未覆盖的独立影响对 |
| `mcdc.status` | enum | `present` / `absent` / `na` |

```json
{
  "schema_version": 2,
  "source_file": "pkg/service/order.go",
  "scope": "incremental",
  "line":   { "coverage_percentage": 100.0, "total_statements": 22, "covered_statements": 22, "uncovered_lines": [], "status": "present", "best": 100.0, "last": 98.41 },
  "branch": { "branch_coverage": 87.5, "total_branches": 16, "covered_branches": 14, "uncovered_branches": [{ "start_line": 22, "end_line": 24, "approx": true }], "approx": true, "mode": "go_block", "status": "present" },
  "mcdc":   { "mcdc_coverage": null, "applicable": false, "reason": "no boolean decisions", "uncovered_mcdc_pairs": [], "status": "na" }
}
```

> 数据缺口语义（解决 #5）：`status=na` 表示**不适用**（等价 100%，如无布尔判定的 MC/DC）；`status=absent` 才是**真实未测量**（渲染为 `-`）。下游据此渲染 `N/A(=100)` 而非一律 `-`。

### 3. `mutation_report.json`（schema_version 2）

> ⚠️ **schema 已锁定为 v2**：新增 `rounds[]` **逐轮结构化数据**（解决自愈闭环逐轮 MSI 演进只在
> Markdown 自由文本、无法程序化消费 #2）、`file_level_msi`/`package_level_msi` **双 MSI**（解决包级
> 冒充文件级 #3）、`msi_status` **显式状态**（解决 null/0/N-A 歧义 #5）。由
> `scripts/mutation/build_mutation_report.py` 组装（首轮）/ `refocus_mutation.py merge` 追加（补强轮），
> 并由 `validate_report.py --kind mutation_report` 校验。

变异测试报告。`msi`（Mutation Score Indicator）为核心质量指标，统一口径 `killed / (total - not_covered) × 100`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `schema_version` | int | 固定 `2` |
| `status` | enum | `completed` / `completed_with_warnings` / `skipped` / `timeout` |
| `scope` | enum | `file` / `package` / `incremental` |
| `source_file` | string\|null | 目标源文件（scope=file 时聚合文件级 MSI 的依据） |
| `msi` | number\|null | **生效 MSI**：scope=file 时为文件级；无可变异点时 `null` |
| `msi_status` | enum | `present` / `skipped_no_mutants`（无可变异点，等价 100%）/ `not_recorded`（真实缺口） |
| `file_level_msi` | number\|null | 文件级 MSI（按 `source_file` 从 per-mutant 聚合） |
| `package_level_msi` | number\|null | 包级 MSI（全部变异体聚合，仅作对照） |
| `tool_native_msi` | number\|null | 工具自身报告的 MSI（仅审计对照，不参与门禁） |
| `killed`/`survived`/`timeout`/`error`/`not_covered`/`total` | int | 各状态变异体数（包级全局口径） |
| `rounds[]` | array | **逐轮结构化数据**，每项含 `round`/`mutants_total`/`killed`/`survived`/`not_covered`/`msi`/`healed_this_round[]`；口径与 headline MSI 一致（scope=file 时为文件级） |
| `mutation_rounds_executed` | int | 已发起的补强重跑轮次 |
| `mutation_final_status` | string\|null | `success` / `max_mutation_rounds` / `null` |
| `per_file` | object | `{文件路径: {killed,survived,timeout,error,not_covered,total,msi}}` 按文件聚合 |
| `survivors[]` | array | 存活变异体明细（`operator`/`file`/`line`/`suggestion`） |
| `threshold` / `meets_threshold` | number / bool | MSI 阈值 / 是否达标 |

```json
{
  "schema_version": 2,
  "status": "completed",
  "tool": "gremlins",
  "scope": "file",
  "source_file": "api_client.go",
  "msi": 92.59,
  "msi_status": "present",
  "file_level_msi": 92.59,
  "package_level_msi": 88.30,
  "killed": 75, "survived": 6, "timeout": 0, "error": 0, "not_covered": 31, "total": 112,
  "mutation_rounds_executed": 1,
  "mutation_final_status": "success",
  "rounds": [
    { "round": 0, "mutants_total": 112, "killed": 70, "survived": 10, "not_covered": 32, "msi": 87.65, "healed_this_round": [] },
    { "round": 1, "mutants_total": 112, "killed": 75, "survived": 6,  "not_covered": 31, "msi": 92.59, "healed_this_round": ["L294", "L297"] }
  ],
  "per_file": { "api_client.go": { "killed": 75, "survived": 6, "timeout": 0, "error": 0, "not_covered": 31, "total": 112, "msi": 92.59 } },
  "survivors": [ { "operator": "CONDITIONALS_BOUNDARY", "file": "api_client.go", "line": 42, "suggestion": "补充 age == 18 的边界用例" } ],
  "threshold": 80.0,
  "meets_threshold": true
}
```

工具缺失/超时降级：`{"schema_version":2,"status":"skipped","msi":null,"msi_status":"not_recorded","rounds":[], ...}`。

#### `heal_advice.json`（schema_version 2）字段口径

> 固定字段：`uncovered_lines` / `uncovered_branches` / `uncovered_mcdc_pairs` / `surviving_mutants` + 可读 `advice[]`。
> ⚠️ **无任何 `coverage` 字段**（如 `coverage.branch.value`）——下游禁止读取不存在的 coverage 路径，否则会得到 `-`（误当缺数据）。校验：`validate_report.py --kind heal_advice`。

### 4. `final_report.json` 的 `gate_results[]`

最终门禁汇总数组，由 `scripts/gate/stage_gate.py --evaluate-gate` 综合 `coverage_report.json` 与 `mutation_report.json` 后写入 `final_report.json`。每条对应一个门禁指标的评估结果，区分硬门禁（不达标即失败）与软门禁（不达标仅告警）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `metric` | string | 指标类型枚举：`line`（行覆盖）/ `branch`（分支覆盖）/ `mcdc`（MC/DC 覆盖）/ `mutation`（变异得分 MSI） |
| `level` | string | 门禁级别：`hard`（硬门禁，`passed=false` 则整体失败）/ `soft`（软门禁，不达标仅置 `warning`） |
| `value` | number | 该指标的实测值（百分比或分数） |
| `threshold` | number | 该指标的门禁阈值 |
| `passed` | bool | `value >= threshold` 是否成立 |
| `warning` | bool | 软门禁未达标时为 `true`，否则（硬门禁或已达标）为 `false`。**告警文案由脚本输出的 `message` 字段承载。** |

```json
{
  "gate_results": [
    { "metric": "line",     "level": "hard", "value": 87.5, "threshold": 80.0, "passed": true,  "warning": false },
    { "metric": "branch",   "level": "hard", "value": 75.0, "threshold": 70.0, "passed": true,  "warning": false },
    { "metric": "mcdc",     "level": "soft", "value": 83.3, "threshold": 90.0, "passed": false, "warning": true },
    { "metric": "mutation", "level": "soft", "value": 78.6, "threshold": 80.0, "passed": false, "warning": true }
  ]
}
```

### 6. `final_report.json`（schema_version 2，唯一 SSOT）

> ⚠️ **产物分层归位（解决 #4）**：`final_report.json` 是 S7 **单一权威聚合**，必含行/分支/MC-DC/MSI
> 全部指标 + 各自 `scope` + 各自逐轮序列（`coverage_history` / `mutation_history`）+
> `metrics_availability`（每项是否有值）。`unit-test-report.md` 由 `scripts/report/render_report.py`
> 从本文件**纯渲染**生成，**不再手填**；下游机器消费**只读本 JSON**，彻底消除"Markdown 主源 / JSON 回退"
> 的脆弱链路。由 `scripts/report/build_final_report.py` 组装、`validate_report.py --kind final_report` 校验。

| 字段 | 说明 |
|------|------|
| `schema_version` | 固定 `2` |
| `summary` | `line_coverage`（取 best）/`branch_coverage`/`mcdc_coverage`/`mutation_msi`/`source_guard`（clean/violation）/`overall_passed` |
| `metrics_availability` | `{line,branch,mcdc,msi}` 各项取值枚举：`present`/`absent`/`na`/`skipped_no_mutants`/`not_recorded`——下游据此渲染 `N/A(=100)` / `-` / `0`，不再猜测 |
| `metrics` | 各指标详情：`value`/`scope`/`status` + line 的 `best`/`last`/`history`、branch 的 `approx`/`mode`、msi 的 `file_level_msi`/`package_level_msi`/`history`(=rounds[]) |
| `gate_results[]` | 直接采用 `progress_status.json.gate_results[]`（SSOT，见 §4） |
| `coverage_history` / `mutation_history` | 行/变异逐轮序列（来源统一，来自 `heal_state.json`） |
| `source_bugs[]` | 见 §source_bugs（无发现为 `[]`，禁止删除） |

```json
{
  "schema_version": 2,
  "task_id": "task_x", "source_file": "api_client.go", "language": "go", "scope": "incremental",
  "summary": { "line_coverage": 100.0, "branch_coverage": 87.5, "mcdc_coverage": null, "mutation_msi": 92.59, "source_guard": "clean", "overall_passed": false },
  "metrics_availability": { "line": "present", "branch": "present", "mcdc": "na", "msi": "present" },
  "metrics": {
    "line":   { "value": 100.0, "best": 100.0, "last": 98.41, "scope": "incremental", "status": "present", "history": [98.41, 100.0] },
    "branch": { "value": 87.5, "scope": "incremental", "status": "present", "approx": true, "mode": "go_block" },
    "mcdc":   { "value": null, "status": "na", "applicable": false },
    "msi":    { "value": 92.59, "scope": "file", "status": "present", "file_level_msi": 92.59, "package_level_msi": 88.30, "history": [] }
  },
  "gate_results": [],
  "coverage_history": [98.41, 100.0],
  "mutation_history": [],
  "test_files": ["api_client_test.go"],
  "heal_iterations": 4,
  "source_bugs": [],
  "warnings": ["mutation msi 75.0 < 80 (soft)"],
  "created_at": "2025-02-26T10:15:30Z"
}
```

### 5. `source_guard.json` / `source_guard_verify.json`（源码红线）

源码防篡改守卫产物，落实「测试全流程禁止修改任何源代码」红线。`source_guard.json`（S1 快照）由 `scripts/guard/source_guard.py snapshot` 产出；`source_guard_verify.json`（S4 每轮 / S7 收尾复核）由 `... verify --json` 产出。**红线为最高优先级：`verify` 检出违规即 exit 2 阻断，`final_report.json.summary.overall_passed` 一票置 `false`。**

**`source_guard.json`（快照）字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `kind` | string | 固定 `source_guard_snapshot` |
| `root` | string | 被测项目根目录（TARGET_PATH） |
| `language` | string | 语言键（决定源码/测试文件区分规则） |
| `scope` | string | `tree`（全量源码树）/ `paths`（仅显式路径，增量） |
| `count` | int | 受保护源码文件数 |
| `digests` | object | `{相对路径: sha256}`，受保护**源码**文件指纹（已排除测试文件/`exclude_dirs`/工作区） |

**`source_guard_verify.json`（复核）字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `status` | string | `clean`（源码零改动）/ `violation`（红线被破） |
| `kind` | string | 固定 `source_guard_verify` |
| `baseline_count` / `current_count` | int | 基线 / 当前受保护源码文件数 |
| `violations.modified[]` | string[] | 被**修改**的源码文件相对路径 |
| `violations.deleted[]` | string[] | 被**删除**的源码文件相对路径 |
| `violations.added_source[]` | string[] | 被**新增**的源码文件相对路径 |
| `snapshot_file` | string | 对应快照路径 |

```json
{
  "status": "violation",
  "kind": "source_guard_verify",
  "root": "/path/to/project",
  "language": "go",
  "scope": "tree",
  "baseline_count": 12,
  "current_count": 13,
  "violations": {
    "modified": ["pkg/service/order.go"],
    "deleted": [],
    "added_source": ["pkg/service/helper.go"]
  },
  "snapshot_file": "workspace/<task>/metadata/source_guard.json"
}
```

> `final_report.json.summary.source_guard` 取 `clean` / `violation`；`violation` 时（除非用户显式授权改源码）`overall_passed=false`，并在 `unit-test-report.md` 追加「红线违规」章节列出 `violations`。
