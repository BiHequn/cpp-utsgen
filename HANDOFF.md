# HANDOFF - 会话交接（2026-10-05）

> 新会话开场：先读本文件 → `AGENTS.md` → `knowledge/general.md`（+ 对应语言篇）。
> 本文件为唯一交接入口；历史决策细节在 `docs/discussions/`（按日期）。

## 1. 项目定位

| 仓库/路径 | 角色 |
|---|---|
| `C:\Users\hequnbi\workspace\new_workspace\Cpp-UTsgen`（develop） | **主战场**：治理流程、测试资产、账本、报告 SSOT |
| `C:\Users\hequnbi\workspace\new_workspace\cpp-utgen` | 工具供应商：MSVC 隔离构建、覆盖率、签名等能力 |
| `C:\Users\hequnbi\workspace\new_workspace\ngrdownloader` | 被测工程：login_camp（CMake 生成 MSVC，/MT，/utf-8），自有 67 源文件 |

## 1.5 最新进展：cpp-msvc-strict 通用 C++ 流水线（2026-10-08）

在旧 M3 战役之外，新增了与手工批处理隔离的 **cpp-msvc-strict** 通用流水线：

* 入口：`scripts/engine/cpp_msvc_strict_engine.py`。
* 输入：单个 `.vcxproj + config + platform`。
* 覆盖 S0-S7：build facts、闭包 resolver、provider_v2、true-oracle 生成、
  CodeCoverage、mutation 显式 `not_recorded`、final report。
* 关键能力：自动补闭包源、runtime DLL 探测/部署、vendor runtime mismatch
  修复、对象名防碰撞、测试/生产 C++ 标准分离。
* 不修改生产源码；旧 M3 provider/batch 脚本保持不变。

### 新真实验证

在 `upgrader.vcxproj + SpeedCounter.cpp` 上完成闭环：

* 自动闭包补入 `TimeCache.cpp`、`Http.cpp`。
* 3 cases × 2 runs 全 PASSED。
* `SpeedCounter.cpp` 行覆盖 `20/67 = 29.85%`。
* `branch = na`、`mutation = not_recorded`、`source_guard = clean`。
* final report `overall_passed = true`。

产物：`build/cpp-msvc-strict-upgrader-closure/`。

## 1.6 最新进展：Requirement / TestIntent / GTest mapping（2026-10-08）

在 cpp-msvc-strict 上补了 requirement → TestIntent → GTest mapping 的最小闭环。
当前 `develop = f8ff9b9`，**未 push**。

### 已实现

* `parse_ast.py` 支持 C/C++ pointer-return 函数、`static` 函数识别、
  函数内 `decisions[]` 提取，并修复 Windows 子进程 UTF-8 输出。
* 新增 requirement normalization：
  - `scripts/cpp/normalize_requirements.py`
  - `assets/schemas/cpp_normalized_requirements.schema.json`
* 新增 oracle resolver：
  - `scripts/cpp/resolve_oracles.py`
  - `assets/schemas/cpp_oracle_resolved_intents.schema.json`
  - 分辨 `true_oracle` / `relational_oracle` / `contract_oracle` /
    `already_covered` / `not_applicable` / `unresolved`。
* 新增 GTest renderer：
  - `scripts/cpp/render_gtest.py`
  - pending 项生成 `GTEST_SKIP()`，不生成 `SUCCEED()` 或恒真断言。
* provider 现在单独统计 `skipped_tests`，不再把 skipped 算成 passed。
* `validate_planned_cases.py` 会解析 GTest 源码中的：
  - `REQUIREMENT`
  - `STATUS`
  - `ORACLE_EVIDENCE`
  并只把真实 `EXPECT_/ASSERT_` 且 runtime PASSED 的测试计入覆盖。
* `.gitignore` 已加入 `/build/`。

### xxtea.c 最小验证

主体：`source/login_camp/king/Libs/xxtea/xxtea.c`，scope = public API。

当前 minimal S3 采用 raw planned cases，不做 normalization 合并，因此：

```text
planned_case_count    = 68
test_intent_count     = 68
generated_gtest_count = 68
```

S4 provider 真实编译、链接、运行通过：

```text
provider_status = passed
runtime_total   = 142   # 2 runs × 71
runtime_passed  = 6     # 旧 M3 的 3 条真实测试 × 2 runs
runtime_skipped = 136   # 68 个 pending GTest × 2 runs
runtime_failed  = 0
```

requirement checker 结果符合预期：

```text
planned_cases = 68
pending       = 68
covered       = 0
coverage      = 0%
stage_4       = failed
```

也就是说：68 个 GTest 已能编译、链接、运行，但都是 pending scaffold，
没有真实 oracle，因此不能算 requirement coverage。

产物：

```text
build/xxtea-68-gtest-validation/
```

### 下一步

1. 从 68 条里选可推导 oracle 的 requirement，把 `GTEST_SKIP()` 替换成真实
   `EXPECT_/ASSERT_`，并保留 `REQUIREMENT` / `ORACLE_EVIDENCE`。
2. 重跑 S4，确认 checker 从 `0 covered` 递增。
3. 再把旧 M3 的 3 条 xxtea 测试映射到 normalized requirements。
4. `normalize_requirements.py` / `resolve_oracles.py` 已保留为后续通用能力，
   但当前 strict engine 默认先走 raw planned cases 最小链路。

## 1.7 当前交接状态（2026-10-09 上午）

> 本节是最新事实，优先级高于 §1.6 中已经过时的“下一步”描述。

### 已完成并验证

* xxtea resolver 链路已接进 strict engine 的 S3：

  ```text
  68 raw planned requirements
    -> 52 normalized requirements
    -> 52 resolved test intents
    -> 52 generated GTest functions
  ```

  产物：`build/xxtea-resolver-validation/`。S4 能真实编译、链接、运行，
  但 52 个 requirement GTest 仍是 `GTEST_SKIP()` scaffold，因此
  `covered_requirements = 0`，stage_4 按硬门禁失败。这是能力验证结果，
  不是链路崩溃。

* login 全仓规模估算已确认：

  ```text
  82 source files
  943 functions
  1593 decisions
  ~8881 planned requirements
      = 3496 scenario
      + 3758 MC/DC
      + 1627 dependency
  login_camp 67 文件子集约 6130 条
  ```

  该 8881 是基于 `project_graph.json` 的完整展开估算；当时没有落盘完整
  `cpp_requirement_plan.json`，不要把
  `build/login-requirement-estimate/cpp_requirement_plan.json` 当成 login
  全仓 plan——那个文件是 xxtea 重跑产物（68 条）。

* login 全仓闭包已打通。关键事实：
  - `build/lib/Debug/*.lib` 是 2026-09-23 的过期 shared lib，缺
    `RegeditUtil`、`KingFoundation::File` 等符号，不能用。
  - M3 已验证的链接配方在
    `C:\Users\hequnbi\workspace\new_workspace\ngrdownloader\build\source\login_camp\test_login_meta.json`
    和 `C:\ut\m3-module-lib\test_login.vcxproj`。
  - 正确闭包库：`C:\ut\m3-module-lib\{module,foundation_closure,core_closure,rpc_kit_closure,kl_network_closure,namedpipe_closure,proto_closure}.lib`。
  - 用该配方后 provider 探针于 2026-10-08 20:49:54 成功链接出
    2MB Debug exe；closure planner 于 21:17:03 写出：
    `cpp_closure_plan.json`、`cpp_link_diagnostics.json`、`cpp_oracle_plan.json`。

### 当前未完成 / 阻塞点

1. login 全仓 requirement plan 还没有落盘。
   - 2026-10-09 09:46 手动重启
     `scripts/cpp/build_requirement_plan.py`，随后为交接主动停止。
   - `build/login-full-chain-run/artifacts/tests/generated_requirements_test.cpp`
     不存在，因此**当前没有 login 全仓 TEST 数**。
   - `build/login-full-chain-run/metadata/engine_result.json` 是 20:04 的旧
     `smoke_inputs_missing` 失败结果，不代表 21:17 之后的真实进度。

2. 统一构建 graph mode 已实现、已验证、已提交（commit `faff515`）。
   - 新增：
     - `scripts/cpp/build_graph.py`
     - `assets/schemas/cpp_build_graph.schema.json`
     - `tests/test_cpp_build_graph.py`
     - `tests/test_cpp_build_graph_stage4.py`
     - `tests/test_provider_v2_graph_mode.py`
   - 修改：
     - `scripts/engine/cpp_msvc_strict_engine.py`
     - `scripts/provider/msvc_provider_v2.py`
     - `assets/schemas/provider_request_v2.schema.json`
     - `scripts/validate/validate_report.py`
  - 2026-10-09 上午回归结论（事实）：
    - `test_cpp_build_graph.py` / `test_cpp_build_graph_stage4.py` /
      `test_oracle_resolver.py` / `test_provider_v2_graph_mode.py`
      沙箱外全部通过；provider graph-mode 单跑 2/2 OK（31.7s）。
    - 旧 `MSB4184` 根因已定位：是执行沙箱对
      `C:\Users\hequnbi\AppData\Local\Microsoft SDKs` 的读取限制
      （shell 里 `icacls`/`Get-ChildItem` 同样被拒，且与 build 目录
      位置无关），不是 provider 代码缺陷。graph-mode 测试必须在
      沙箱外运行。
    - `test_provider_v2_graph_mode.py` 的诊断缺陷已修：provider 失败
      信息改为 provider_result_v2 的 `reasons[]` + 阶段日志
      （`cmake_configure.log` / `cmake_build.log` / `ctest.log`，
      存在才读尾部），不再拼接 provider stdout/stderr，也不再无条件
      读 `ctest.log`。
    - 批量运行出现过一次 ctest `BAD_COMMAND` / `Process not started`：
      configure/build 均成功、api.exe 已生成，为 Device Guard 间歇
      拦截未签名 exe 的已知风险（§3 事实 8）；单发重跑即过（16.6s OK）。

### 新会话建议动作

1. graph-mode 回归必须在沙箱外运行（MSBuild 需读取
   `%LOCALAPPDATA%\Microsoft SDKs`，沙箱内必现 `MSB4184`）；
   ctest 出现 `BAD_COMMAND` 属 Device Guard 间歇拦截未签名 exe
   （§3 事实 8），单发重跑确认即可，不要先改 graph 逻辑。
2. 继续 login 全仓时不要重跑 S0-S2/closure：
   - 复用 `build/login-full-chain-run/metadata/project_graph.json`。
   - 复用 21:17 的 `cpp_closure_plan.json`。
   - 请求文件：`build/login-full-chain-request.json`（已带 82 个 sources
     和 M3 closure `link_libs`）。
   - 先跑 `build_requirement_plan.py` 落盘完整 plan，再依次跑
     `normalize_requirements.py`、`resolve_oracles.py`、`render_gtest.py`。
3. 最终 TEST 数以
   `build/login-full-chain-run/artifacts/tests/generated_requirements_test.cpp`
   中 `TEST(` 计数为准；没有该文件前不要报数。

## 1.8 当前交接状态（2026-10-10 下午）

> 本节是最新事实，优先级高于 §1.6 和 §1.7 中已经过时的下一步描述。

### 权威范围和口径

* 当前分母是 `build/login-oracle-scope/normalized.json` 的
  **4932 normalized requirements**，对应 54 个 subject 文件。
  `scope.json` 是排除项权威：vendor 14 和 WeGame/Oversea 14 不算分母；
  被排除图节点仍保留为 closure dependencies。
  `KingLoginWeGameManager.cpp` 和 `FontAssist.cpp` 在范围内。
* 旧 full-login 的 **6746** 是 `build/login-full-chain-run` 的结构展开
  scaffold：11095 raw cases 去重后 6746，无 oracle contract，全部
  `GTEST_SKIP`，断言数为 0。它只用于解释旧链路，不能作为当前进度或覆盖率
  分母。
* 当前合并账本是 `build/oracle-contracts-all-merge.json`。机械分类：

  ```text
  A directly executable            =   19
  B contract needs harness         =  526
  C removable denominator          =   78
  D bad plan record                =  598
  E likely harness-support TODO    = 1329
  F semantic TODO                  = 2372
  G conflict                       =    8
  H unreviewed                     =    2
  total                            = 4932
  ```

  剔除 C+D 后，有效分母是 **4256**。这些数字是 normalized requirement
  记录数，不是编译/运行/行覆盖结果。

### B+E harness 池

`B+E = 526 + 1329 = 1855` 是最大尝试池，不是“全部已推导成功”的承诺。
其中 B 已有真实 oracle 契约；E 只是失败原因显示卡点像 harness/注入/隔离。
按执行机制聚合：

```text
M1 injection / public route        =  944
M2 runtime isolation/determinism   =  451
   fresh process 209, serial/global 79,
   log/telemetry 117, allocation 27, time/random 19
M3 IPC / network runtime           =  145
M4 header + link/mock replacement  =  204
M5 unknown                         =  111
```

下一优先级不是继续逐条推 F，而是实现 M4 -> M2 -> M3 -> M1 的统一 harness
能力，随后重跑 merge/classification。M5 只补分类，不先手写。

### 代码能力与验证

* `scripts/cpp/derive_oracle_contracts.py` 支持字面契约推导和
  source-review pack 合并；会校验 requirement ID、语义 evidence、
  execution requirements 和 include 路径。非空 execution requirements 会被
  隔离，不自动算成 executable。
* `resolve_oracles.py` 现在透传 contract includes；
  `render_gtest.py` 支持 contract includes、按源文件输出目录、
  `--executable-only` 和跨文件全局唯一 TEST 名。
* strict engine 的 S4 现在默认 legacy；只有 request 明确
  `build_mode: graph` 才生成/使用 graph。graph mode 会把 forced includes
  投影成 CMake `/FI`。
* provider v2 legacy/graph 都支持签名后执行；legacy 编译增加
  `/bigobj`、独立 `compile.pdb`、`/INCREMENTAL:NO`、流式日志、elapsed time、
  进程树超时终止。`source_vcxproj` 可生成统一测试 vcxproj。
* 这些能力不等于 login 全仓 S4 已成功。此前 link 内存耗尽/未完整退出仍是
  停止事实；`build/current-real-tests/` 只有 19 个直接可执行 GTest 的静态
  渲染产物，未编译。
* 2026-10-10 回归：oracle derivation、oracle batch rendering、provider
  build execution、signing、requirement planning、graph stage 4 和 provider
  graph mode 共 **28 tests passed**（沙箱外）。沙箱内 graph/signing 会因
  `MSB4184` 和 keystore 权限误报失败。

### 新会话动作

1. 先读 `AGENTS.md`、`knowledge/general.md`、`knowledge/cpp.md` 和
   `utgen-project-pipeline` skill。
2. 以 §1.8 的 4932/4256 口径为准；不要把 build/ 下的 review pack 当成
   已提交 SSOT，也不要把 6746 当分母。
3. 先实现 M4（header/link mock replacement）和 M2（runtime isolation），
   用现有 19 + 526 验证；不要直接 resume 6746 scaffold 编译。
4. M5 的 111 条先补分类，M1 的 944 条按同一注入/route 机制批量处理。
5. 所有 review pack 结论在跑编译/运行前都只是 static semantic review，
   不是 runtime coverage。

## 2. 当前状态（M3 已收口）

**总账：67 = 52 完成 (77.6%) + 1 原因表 + 14 范围外 + 0 未开始**

| 分类 | 总数 | 完成 | 原因表 | 范围外 |
|---|---|---|---|---|
| leaf | 7 | 6 | 1（KLGlobalSharedData，空 TU） | 0 |
| internal_lib | 50 | 38 | 0 | 12 |
| protobuf | 6 | 4 | 0 | 2 |
| windows_api | 4 | 4 | 0 | 0 |

* true-oracle 用例 **138** 全 PASSED（真编译真链接，签名后连跑验证）。
* 真实行覆盖率 **1896/6103 = 31.1%**（48 主体有数据；partial=未覆盖，branch=na）。
* 14 范围外 = WeGame/Oversea 文件（用户决策剔除，不计分母）。
* 唯一原因表：KLGlobalSharedData.cpp（空翻译单元）。原 9 个原因表中 7 个已由
  mock 闭包救回转完成，Ping.cpp 复审后补测转完成。

**批次账本**：M1 leaf 5 → batch2×4(24 用例) → batch3×7(22) → batch4×3(8) →
batch5×3(5) → batch6×8(17) → batch7×6(22) → batch8 Ping(3) → batch9×7(19)
→ mock 主路径×1(9)。**所有提交已推送**（origin/develop = ad62fa1）。

## 3. 钉死事实（勿再现场猜测，完整版见 docs/discussions/）

1. **CMake 静态库空壳**：CMake 生成的 .lib 只含 PCH obj；真实符号靠直编
   源文件闭包（FC/COREEX/KNET 配方）。
2. **KLNetwork ctor 链间歇挂死/SEH**（探针矩阵实证）：根因在 resolver
   配置/DomainIPMapper 装配（vendor/私有边界）。规避 = **mock 闭包**
   （见 §4），已让原 9 个原因表中的 7 个复活。
3. **GCloudCore XLog 未初始化链 SEH**：CampMacro 的 XLogInfo 路径在
   未调 `ABase::GCloudLogInit()` 时必崩；`gtest_main_xlog.cpp` 已内置调用。
4. **AppConfigService::GetGlobalStorage/GetUserStorage**（真 sqlite+XLog）
   是残余 flake 源，mock 闭包中已链接替换为空存储语义。
5. **编码标志权威**：vcxproj AdditionalOptions（/utf-8 /Zc:threadSafeInit-）；
   oneupdns 需追加 `/DONEUPDNS_USE_NGR_CURL`（oneupdns_shared.vcxproj 权威）。
6. **oneupdns aes.cpp/hex.cpp 必须单独 /Fo**：与 foundation Aes.obj 在
   大小写不敏感 FS 上基名碰撞。
7. **运行时 DLL 清单**（dumpbin /DEPENDENTS 实测）：libcrypto-3-x64、
   rpc_kit、sqlcipher、libcurl-x64 —— bin 必须随附。
8. **Device Guard 间歇拦截未签名 exe**：跑批前统一 EV 签名
   （keystore sign_tool.py，需提权），不能凭单次通过视为豁免。
9. **KLNetworkFactory 是头文件内联工厂**：不可链接替换；但工厂创建本身
   安全（探针 5/5），mock 在具体类方法层替换即可。
10. **KLProtobuf 三件套覆盖率不可采集**：多 TU 构建 PDB 源映射不完整
    （analyzer 不产出主体行），测试本体全绿；待解决映射问题后补采。

## 4. Mock 闭包（核心资产，KLNetwork 系通用）

位置：`tests/fixtures/msvc-m3/mock-batch/mock_kl_network_deps.cpp`（+.h）
+ `mock_appconfig_service.cpp`（含 loginguard 链接所需全部 stub）。

* knowledge/cpp.md 优先级链第 4 级的 MSVC 落地：测试构建提供符号定义 +
  排除真实现 .cpp（curl 客户端/DNS resolver/DomainIPMapper/
  AppConfigService 存储链）。
* mock client 的 AsyncPost **默认只记录不触发回调**——绕开持锁同步回调
  风险，并使"请求落进 sink"成为确定性断言；`asyncAutoFire=true` 时用
  `syncHandler` canned 响应同步触发（驱动重试链）。
* 配套 `gtest_main_xlog.cpp`（GCloudLogInit）。效果实证：
  KLNetwork 3.6%→71.4%（315/441，主路径 mock 测试）。

## 5. 剩余工作（按性价比）

1. **复活与新完成 12+1 文件的真实覆盖率已采集**（batch89 目录）；
   如需更深覆盖，对 request 类 Login/Run 主流程写 mock 驱动用例
   （stub client 返回 canned 响应，驱动 DidLoginSuccess 等逻辑分支）。
2. **MC/DC 与变异（Mull）未接入**——报告中已标 not_recorded；Mull 需
   clang+LLVM19 工具链（见 assets/mutation-tools.yaml 注记）。
3. **源码侧初始化注入点**（KLNetwork ctor 拆分/惰性化）可根治 SB-001
   死锁与 vendor SEH——需用户授权改生产源码。
4. 范围外 14 文件如需纳回，同款配方直接可用。

## 6. 关键路径速查

| 资产 | 路径 |
|---|---|
| 最终报告（stage_7 格式） | `tests/fixtures/msvc-m3/final-report/{final_report.json, unit-test-report.md}` |
| 覆盖率聚合账本 | `tests/fixtures/msvc-m3/coverage-final/m3_coverage_summary.json` |
| 覆盖率采集脚本 | `tests/fixtures/msvc-m3/coverage-final/{run_coverage.ps1, collect_batch89.ps1, aggregate_final_report.py}` |
| mock 闭包 | `tests/fixtures/msvc-m3/mock-batch/`（FC+MOCK+COREEX 配方见 build_m3_mock.bat） |
| 批次资产 | `tests/fixtures/msvc-m3/batch2..9/` |
| module.lib | `C:\ut\m3-module-lib\module.lib`（82 obj） |
| CodeCoverage 17.9.0 | `%LOCALAPPDATA%\cpp-utgen\codecoverage\17.9.0\amd64\` |
| 签名工具 | `C:\Users\hequnbi\keystore\sign_tool.py`（keystore 目录下运行，需提权） |
| 决策账本 | `docs/discussions/2026-10-03/04/05-*.md` |

## 7. 工作区注意

* 2026-10-09 本轮改动已按序提交（均未 push，develop 领先 origin 6 commits）：
  - `43f9245` chore: untrack VS IDE browsing artifacts
    （`.vs/` 已加入 `.gitignore` 并 untrack）。
  - `faff515` feat(cpp): add graph build mode to provider v2 pipeline
    （graph mode 功能闭环 + 诊断修复）。
  - `8852dd7` test(cpp): cover all oracle resolution families。
  - 本条目（HANDOFF 更新）为最后一个 commit。
* 2026-10-10 交接前新增提交（未 push）：
  - `aa4b995` feat(cpp): restore legacy S4 and add unified batch build。
  - `be64313` feat(cpp): merge reviewed oracle contracts safely。
  - 本条 HANDOFF/§1.8 更新是当前会话的最后一个 commit。
* `build/` 已在 `.gitignore`，`login-full-chain-*`、`provider-graph-debug`
  等验证产物不进 git；交接只依赖其中的 metadata/artifacts 事实。
* AGENTS.md 两条强制规则仍有效：**仓内方案优先**（先查 knowledge/scripts/
  再动手）与 **先读 knowledge/ 再动手**。
