# Cpp-UTsgen — Agent 使用指引

C/C++ 单元测试治理工程。Agent 在本仓工作时遵循本指引；硬约束以 `AGENTS.md` 为准。

## 必读顺序（每次会话/任务开始前）

1. `AGENTS.md` — 执行纪律、提交纪律、红线（改源码禁令）、外部依赖默认 Mock。
2. `HANDOFF.md` — 当前会话交接与钉死事实（勿再现场猜测）。
3. `knowledge/general.md` — 推理层策略本体：分层 Mock 决策模型、Mock 兜底
   优先级链、依赖契约推导原则、四类场景覆盖、常见反模式。
4. 语言任务叠加 `knowledge/cpp.md`（或 `cpp_summary.md`）/ `knowledge/c.md`。

> 规则：结论只来自事实（源码、命令输出）；落原因表前必须穷尽 Mock 优先级链；
> 禁止以"依赖复杂/无接口"为由跳过可测试文件。

## 目录导航

| 路径 | 用途 |
|---|---|
| `stages/stage_0..7.md` | S0-S7 治理流程：环境探测 → 工作区/红线快照 → AST 诊断 → MC/DC 方案 → 自愈内环 → 覆盖率门禁 → 变异 → 报告。每阶段有强制产物与门禁校验命令。 |
| `assets/unit-test.defaults.yaml` | 门禁真源（line/branch/mcdc/mutation 层级与阈值、构建档位 gxx/msvc）。可被项目级 `unit-test.config.yaml` 覆盖。 |
| `assets/langpacks/*.yaml` | 语言包：知识文档定位、覆盖率格式、构建档位、变异工具映射。 |
| `assets/schemas/*.json` | coverage/mutation/final_report 等 schema（v2 锁定结构），产出必须过 `scripts/validate/validate_report.py`。 |
| `scripts/gate/stage_gate.py` | 阶段门控与门禁评估（阈值自动读配置，结果回写 SSOT）。 |
| `scripts/guard/source_guard.py` | 源码红线指纹快照/复核（S1/S4/S7；exit 2 = 红线被破立即停止）。 |
| `scripts/provider/` | MSVC 能力层：`msvc_provider.py`（compile+test+cover JSON 服务）、`vcxproj_extract.py`、`build_module_lib.py`、`vcxproj_test_gen.py`、`coverage_convert.py`（CodeCoverage XML → lcov+summary）。 |
| `scripts/report/` + `scripts/validate/` | final_report.json 唯一 SSOT 聚合器与渲染器（报告禁止手填）。 |
| `tests/fixtures/msvc-m3/` | M3 批次测试资产与配方（batch2-9 + mock-batch + coverage-final）。 |
| `docs/discussions/` | 关键决策账本（按日期一天一文件）。 |
| `knowledge/` | 推理层策略（Mock 优先级链/依赖契约推导/场景覆盖）。 |

## MSVC 批量模式操作要点（login_camp 实战配方）

1. **闭包组装**：基础闭包 FC（foundation/core 源文件直编）+ COREEX
   （security/config/crashsight）+ KNET（kl_network，含 oneupdns 独立 /Fo
   与 `/DONEUPDNS_USE_NGR_CURL`）。链接 LNK2019 时"缺什么加什么"，但外部
   资源类依赖（网络/文件/线程）**默认走 Mock 闭包**（见下）。
2. **Mock 闭包**：`tests/fixtures/msvc-m3/mock-batch/mock_kl_network_deps.cpp`
   —— curl 客户端/DNS resolver/DomainIPMapper 链接替换 +
   `AppConfigService::GetGlobalStorage` 空存储语义。适用于一切触达
   KLNetwork 构造链的 subject。配合 `gtest_main_xlog.cpp`（GCloudLogInit）。
3. **头文件内联工厂不可链接替换**（如 `KLNetworkFactory`）——在具体类方法层
   替换（探针已证工厂创建本身安全）。
4. **运行时 DLL**：bin 必须随附 `libcrypto-3-x64.dll`、`rpc_kit.dll`、
   `sqlcipher.dll`、`libcurl-x64.dll`（dumpbin /DEPENDENTS 实测口径）。
5. **签名**：构建后统一 EV 签名（keystore sign_tool.py，需提权）——Device
   Guard 对未签名 exe 是**间歇性**拦截，不能凭单次运行通过视为豁免。
6. **编译标志**：以 vcxproj AdditionalOptions 为权威（`/utf-8
   /Zc:threadSafeInit-` 等），统一放 `common_flags.rsp`；oneupdns 需追加
   `/DONEUPDNS_USE_NGR_CURL`。

## 覆盖率采集与口径

- 工具：Microsoft.CodeCoverage 17.9.0（`collect /verbose /output:<trace>
  <exe>` → `analyze /output:<xml> <trace>`）→ `coverage_convert.py` 出
  lcov + summary。covrun64/msdia140 必须部署到被测 exe 同目录。
- 口径：partial 行按未覆盖计（保守）；只统计 subject_file 命中主体；
  branch = na（VS 收集器无分支数据，禁止伪造）。
- 聚合账本：`tests/fixtures/msvc-m3/coverage-final/m3_coverage_summary.json`。

## 报告（stage_7 格式）

- `final_report.json`（schema v2，唯一 SSOT）+ `unit-test-report.md`
  （纯渲染视图，含 YAML front matter）。字段规范见 `REPORT_SCHEMA.md`。
- `source_bugs[]` 无论有无发现必须存在。
- 样例：`tests/fixtures/msvc-m3/final-report/`。

## 红线速记

- 不改生产源码；source_guard exit 2 立即停止。
- 不拿运行结果回填 oracle；不写恒真断言。
- 推不出 oracle 的符号落原因表，且必须先穷尽 Mock 优先级链并留证据。
- 未经用户明确要求不 commit / 不 push。
