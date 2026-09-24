# DSC 位级 drivers 查询恢复：执行交接计划

## 决策

**有条件接受空指针修复，继续处理 drivers 查询差异，作为独立里程碑。**

空指针修复符合已确认根因，可独立保留。原始任务的 driver 查询仍失败，因此只能声明
“崩溃已消除”，不能声明“原始查询已恢复”。这属于功能正确性缺口，应在宣称 DSC
默认位级分析可用前解决。不要为了这一问题撤销 Opaque 回退或关闭位级分析。

执行范围：在本仓库完成最小复现、首个差异定位、证据支持的局部修复和回归；
本计划不要求启动其他 agent，不授权修改 DSC 工作区、发布或推送。

## 当前基线及证据等级

- 仓库：`/home/zys/Project/slang-netlist`。
- HEAD：`0866f5e43a546ebae15696c717b7266178ed7e57`。
- 未提交的已审阅修改：`source/BitSliceList.cpp`、`tests/unit/BitSliceTests.cpp`、
  `tests/unit/BugTests.cpp`。必须保留这些改动，不能将 HEAD 误认为已含修复。
- `BitSliceList::pushLsp` 检查 `path->lsp` 和 `path->rootSymbol()`，无效时整段
  `pushOpaque(expr)`；新增分类及输入依赖可达性测试。该修复方向接受。
- 原崩溃及根因见 [崩溃报告](crash-dsc-add-rvalue-2026-09-19.md)。
- 本次审阅重新运行构建目标 `slang-netlist netlist_unittests`，成功且无需重编译；
  运行 `netlist_unittests '[BitSliceList],[Bugs]'`，33 项、161 断言通过。
  执行报告中的“38 项、168 断言”未按原过滤器独立复核，不应混为同一轮结果。

本次独立复现使用 Debug CLI、固定 DSC 快照及 `-j 1`：

| 模式 | 查询 | 退出码及关键结果 |
|---|---|---|
| 默认位级分析 | `--find dsc_encoder.u_lm.rec_wr_cpnt` | 0，port 节点，bounds `[0,1]` |
| 默认位级分析 | `--drivers dsc_encoder.u_lm.rec_wr_cpnt` | 6，`invalid_query`，`could not find signal` |
| 关闭位级分析 | 同一 find | 0，同一名称和范围的 port 节点 |
| 关闭位级分析 | 同一 drivers | 0，一项 `[0,1]`，driver 为该 port |

用户报告默认线程也完成了复现；执行验收时仍须补齐两种线程配置。
关闭位级分析返回的端口本身只是局部 driver 表示，不能据此宣称已证明上游跨模块依赖。

## 已知的首个查询分歧与待证假设

已确认查询入口分歧：

- `tools/driver/driver.cpp` 的 find 使用 `graph.findNodes`，查找节点名称。
- drivers 分支先调用 `graph.hasSignal(path)`，失败即抛出错误；尚未进入
  `getBitDrivers`。统一异常处理将此错误包装为 `invalid_query`。
- `source/NetlistGraph.cpp:190` 的 `hasSignal` 只检查图中边的
  `edge.symbol->hierarchicalPath`，不检查同名节点。
- `getBitDrivers` 同样从带该名称的边提取 source 与 bounds。

因此，已确认“存在端口节点，但没有带该信号名的边”。尚未确认第一处建图分歧，
也尚未判定是缺边、边符号标注错误，还是边索引承担了不适当的信号存在性语义。
**禁止把这些假设写成已证实根因。**

优先检查 `DataFlowAnalysis` 的位级读取路径、`PendingRvalueQueue`、
`PortConnectionHandler::drivePortSegment`、形式端口与内部符号映射，
再按证据检查 canonical instance 解析。当前端口实际连接位于固定快照
`dsc_encoder.sv` 的 `.rec_wr_cpnt(lm_rec_cpnt)`，声明在 `dsc_linemem.sv:38`。

## 执行步骤

### 1. 固定复现和可比输出

从固定提交创建快照，不读取或修改活动 DSC 工作区：

```bash
repro_dir=$(mktemp -d /tmp/slang-dsc-drivers.XXXXXX)
git -C /home/zys/Project/dsc-cmodel-rtl archive \
  baf6577f7d3db90bf9d4e0287c7b9372ebaa8987 rtl/ip \
  | tar -x -C "$repro_dir"
cmake --build build/clang-debug --target slang-netlist netlist_unittests -j 4
build/clang-debug/tools/driver/slang-netlist "$repro_dir"/rtl/ip/*.sv \
  --top dsc_encoder --drivers dsc_encoder.u_lm.rec_wr_cpnt \
  --format json --max-results 200 --max-depth 64 -j 1
```

保存各模式 stdout、stderr、退出码与构建版本。使用源码路径、节点类型、范围、
边角色和精度比较结果，不以并发分配的 node ID 或 artifact_id 判断等价。

### 2. 缩减并定位建图的第一处差异

- 从 `dsc_linemem` 对 rec_wr_cpnt 的读取方式出发，缩减为不依赖 DSC 文件的 SV。
  保留触发差异所需的动态索引、位选择、内部使用方式和实例连接；不要先假设
  所有端口连接都有问题，也不要将函数返回值切片强行保留在第二个复现中。
- 对比默认/关闭位级分析的端口入边、出边、edge.symbol、bounds、role、precision。
  分别检查形式端口名、内部变量名、父层实际信号 `lm_rec_cpnt`。
- 跟踪对应引用经过切片分类、段驱动、pending 入队/解析到最终边的过程，给出
  最早丢失或改名的位置及原因。临时诊断输出不得进入最终产品输出。
- 查阅 `getBitDrivers` 文档及既有 Driver/Port/Instance 测试，明确局部驱动者语义。
  比较二次开发与本地主线相关代码；新问题的归属必须独立判断，不能沿用空指针
  问题“来自主线”的结论。

交付证据至少包含：最小 SV、失败命令、两模式结构差异、首个分歧的 C++ 位置，
以及由 SV 语义推导的预期 driver 和范围。

### 3. 根据证据做最小修复

- 若确实漏掉引用/边或符号映射，应修复其产生点，保留正确范围及依赖角色。
- 若图的连接本身正确，而信号存在性与驱动查询语义不一致，应明确区分
  “不存在的信号”和“存在但无 driver 的信号”，并同时校验查询的数据来源。
- 不能只把 `hasSignal` 改成检查节点来让 invalid_query 消失；那可能仅将问题变成
  空结果。不能凭端口名称制造伪 driver、自环或全宽依赖来通过测试。
- 默认模式必须保留位级能力；不能自动切到 legacy、复制 legacy JSON、吞掉异常，
  或把所有依赖精度标成 Exact。范围可能比 legacy 更细，不要求 JSON 文本相同。
- 保持当前 JSON schema 和正常退出码约定；真实不存在的名称仍须返回受控错误。
- 修改局限于相关建图/查询代码、必要回归测试及本报告；新增代码注释使用中文。

如果证据足以支持局部修复，可完成步骤 3 和 4 后一次性交接，无需中途申请确认。
若发现必须重定义公共 driver 语义、扩大序列化格式或进行广泛架构重构，完成诊断后
先交接证据和选项，暂停该扩大部分，不猜测实现。

## 回归和验收门槛

1. 新增最小 SV 的库测试与 CLI 测试：必须断言正确 driver 类型/身份、覆盖范围，
   以及输入依赖可达；仅断言“不崩溃”或“JSON 可解析”不够。
2. 原始 DSC drivers 查询在默认位级分析、`-j 1` 和默认线程均退出 0，合法 JSON，
   `command=drivers`、diagnostics 为空、`summary.complete=true`，有效 driver
   覆盖目标 `[1:0]`；逐位覆盖及身份符合所收集的 SV/图证据，无伪边。
3. 显式位范围查询（按当前 CLI 支持的语法）、整信号查询、find 的名称/范围一致。
   若修复涉及存在性，补上未驱动信号和真正不存在信号的区分测试。
4. 对新最小例与 DSC 同时保留关闭位级分析的对照结果；比较语义而非条目数或 ID。
5. 空指针最小例及已有函数返回值切片、端口连接回归仍通过。
6. 运行相关 BitSlice/Bugs/BitDrivers/Port/Instance 测试及 CLI driver 测试；
   最终运行本地已配置的常规 `ctest --test-dir build/clang-debug --output-on-failure`。
   无需下载外部大型设计。失败须注明是否既存、复现命令及影响，不伪报全绿。
7. 如改到图的持久化或索引重建，补充序列化往返验证；未涉及则不扩大范围。
8. 更新崩溃报告，明确原崩溃修复和新查询修复分别完成了什么；核对 diff，保留用户改动。

## 停止条件与交接格式

达到上述门槛后交接，不继续扩展其他查询功能。若证据不足或验收失败，交接阻塞点、
最小复现和已排除假设，不能将仅有合法 JSON 的结果标为成功。

最终报告包含：

- 基线 HEAD、最终 commit（若尚未提交须明确写出）、工作区状态及改动清单；
- 首个建图分歧、修复依据和最小 SV；
- 实际执行的测试过滤器、测试/断言总数、DSC 四组模式的退出码及结构化结果摘要；
- 查询返回 driver 的身份、范围、与上游输入的关系和剩余精度限制；
- 是否达到验收门槛、仍有何限制。

本计划仅落盘交接，不自动派发执行或批准下一阶段。
