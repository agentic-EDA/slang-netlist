# DSC 输入触发 addRvalue SIGSEGV

## 结论

2026-09-19 已在固定 DSC 源码快照复现：默认位级依赖分析触发 SIGSEGV，
shell 退出码 139，stdout/stderr 文件均为空，未生成 JSON envelope。
`--find` 同样崩溃，不限于特定 driver 查询；`-j 1` 仍崩溃。
增加 `--no-resolve-assign-bits` 后同一 driver 查询返回 0。

原始记录是崩溃复现报告，不是修复结论。2026-09-24 已确认空指针根因并缩减独立 SV，详见文末追加定位记录。
不能仅凭线程池栈认定为并发竞态，不能把关闭位级分析当作等价修复。

## 环境与来源

- OS：Linux 6.18.33.2-microsoft-standard-WSL2，x86_64 GNU/Linux。
- 实际运行二进制：`/home/zys/.local/bin/slang-netlist`。
- 版本：`slang-netlist version 0.12.0 (slang 11.0.414+6001e362f)`。
- 二进制 SHA-256：`a2be539d942a962473c48ba65e32e317870059bf5206829a0275d498cfccdf93`。
- 报告目标仓库 HEAD：`0866f5e`，开始时工作区干净；不能据此断言已安装二进制
  恰好由该提交构建，未核实构建溯源。
- 输入仓库：`/home/zys/Project/dsc-cmodel-rtl`。
- 输入固定提交：`baf6577f7d3db90bf9d4e0287c7b9372ebaa8987`，只使用 `rtl/ip/*.sv`。
- 本次临时快照：`/tmp/slang-crash-baf6577.1kvdet`。

复现期间另一会话正在修改 DSC 工作区。直接读取活动工作区曾得到
`mid_sel_q[2:0]` 对升序 unpacked array 的 reversed-range 编译错误并退出 1；
那是不同输入的编译失败，不是本报告的崩溃。以下步骤固定提交，不依赖活动工作区。

## 可重复命令

在 bash 执行，无需修改或切换原仓库工作区：

```bash
repro_dir=$(mktemp -d /tmp/slang-dsc-repro.XXXXXX)
git -C /home/zys/Project/dsc-cmodel-rtl archive \
    baf6577f7d3db90bf9d4e0287c7b9372ebaa8987 rtl/ip \
    | tar -x -C "$repro_dir"
/home/zys/.local/bin/slang-netlist "$repro_dir"/rtl/ip/*.sv \
    --top dsc_encoder --drivers dsc_encoder.u_lm.rec_wr_cpnt \
    --format json --max-results 200 --max-depth 64 \
    > "$repro_dir/stdout.json" 2> "$repro_dir/stderr.txt"
repro_rc=$?
echo "exit_code=$repro_rc"
wc -c "$repro_dir/stdout.json" "$repro_dir/stderr.txt"
```

本次结果：`exit_code=139`，两个文件均 0 字节。shell 自身可能另打印
`Segmentation fault`，它不在被测进程的 stderr 重定向文件中。

## 对照结果

所有对照使用同一固定快照、相同 top 和 JSON 输出限制。

| 查询 / 额外选项 | shell rc | 结果 |
|---|---:|---|
| `--drivers dsc_encoder.u_lm.rec_wr_cpnt` | 139 | SIGSEGV，无 JSON |
| 同上，增加 `-j 1` | 139 | SIGSEGV |
| `--find '**.rec_wr_cpnt'` | 139 | SIGSEGV |
| driver 查询增加 `--no-resolve-assign-bits` | 0 | JSON schema 1，complete=true，diagnostics=[] |

最后一项返回 1 条记录，signal 为 `dsc_encoder.u_lm.rec_wr_cpnt`，范围 [1:0]。
这仅证明该查询在禁用位级解析后能结束，不证明完整图正确或精度等价。
此次结果记录没有逐边 precision 字段，不能宣称取得精确跨模块依赖证明。

## GDB 栈

沙箱内 ptrace 被拒绝；经批准在沙箱外对固定快照运行 GDB，成功捕获 SIGSEGV。
未修改 GDB 配置，也未开放 auto-load safe-path。复现命令：

```bash
gdb -batch -ex 'set debuginfod enabled off' -ex run -ex 'bt 24' \
    --args /home/zys/.local/bin/slang-netlist "$repro_dir"/rtl/ip/*.sv \
    --top dsc_encoder --drivers dsc_encoder.u_lm.rec_wr_cpnt \
    --format json --max-results 200 --max-depth 64
```

观测：Thread 23 收到 SIGSEGV。以下保留符号调用链，省略 ASLR 地址和模板参数：

```text
#0  slang::netlist::NetlistBuilder::addRvalue(
      ast::EvalContext&, ast::ValueSymbol const&, ast::Expression const&,
      DriverBitRange, NetlistNode*, DependencyRole, DependencyPrecision)
#1  slang::netlist::DataFlowAnalysis::handleRvalue(...)
#2  slang::netlist::DataFlowAnalysis::driveRhsLspSegment(...)
#3  slang::netlist::DataFlowAnalysis::handle(ast::AssignmentExpression const&)
#4  slang::ast::Expression::visitExpression<...>(...)
#5  slang::ast::Statement::visit<...>(...)
#6  slang::analysis::AbstractFlowAnalysis<...>::visitStmt(ast::BlockStatement const&)
#7  slang::netlist::NetlistBuilder::handleProceduralBlock(...)
#8  BS::thread_pool<...>::worker(...)
#9  libstdc++.so.6
#10 start_thread
#11 __clone3
```

安装二进制栈没有给出 C++ 源码行号。GDB 自身返回 0 不等于被测工具成功，
本报告以 inferior 的 SIGSEGV 和直接运行 rc=139 为崩溃证据。

## 原始报告给维护者的下一步（定位进展见文末）

1. 使用带调试信息的构建复现同一快照，核对二进制构建提交。
2. 在 addRvalue 处检查 symbol、expression、range、目标节点及访问对象，
   找到实际触发的 SV 表达式，再缩减独立最小用例。
3. 优先检查 driveRhsLspSegment → handleRvalue → addRvalue 的位片段路径；
   这是调用栈和禁用位级分析对照支持的定位方向，不是已证实根因。
4. 将最小用例加入 CLI/库回归：默认模式应返回合法结果或受控错误，不能崩溃；
   同时验证单线程及默认配置。修复后恢复默认位级解析验证，不以禁用功能代替。

本次只新增报告，不修改工具源码、不修改 DSC RTL、不提交任何文件。


## 2026-09-24 根因定位

### 已确认的非法访问

在 HEAD `0866f5e` 的未修改源码上重新构建 Debug CLI：

```bash
cmake --build build/clang-debug --target slang-netlist -j 4
```

对相同 DSC 固定提交、`--top dsc_encoder --find '**.rec_wr_cpnt' -j 1
--format json` 运行 GDB，确认：

- 崩溃位置：`source/NetlistBuilder.cpp:279`，读取 `symbol.kind`。
- 上游位置：`source/DataFlowAnalysis.cpp:494`，
  `auto const &symbol = *path.rootSymbol();` 已解引用空指针；
  第 540 行把这个无效引用交给 `handleRvalue`。
- `path.rootExpr->kind == ExpressionKind::Call`。
- `path.fullExpr->kind == ExpressionKind::RangeSelect`。
- `path.lsp == nullptr`，`symbol` 地址为 `0x0`，当前段宽度为 4。
- `fullExpr` 的源码字节范围 `[6248, 6278)` 对应固定快照
  `rtl/ip/dsc_cfg.sv:141` 的 `get_field(O_VER_MINOR, 4)[3:0]`。
  该文件含中文注释，定位必须按 UTF-8 字节偏移，不能按字符偏移。

因此，这是确定的空指针解引用，与查询目标 `u_lm.rec_wr_cpnt` 无关。
图构建先遍历到配置模块中的函数返回值切片，尚未执行查询就已崩溃。
单线程复现不需要并发竞态作为解释。

### 为什么会生成非法 LSP

1. `source/BitSliceList.cpp:63-70` 只按最外层表达式类型判断，
   将 `ElementSelect`、`RangeSelect`、`MemberAccess` 等统一交给 `pushLsp`。
2. `pushLsp` 第 264 行构造 `ValuePath`，没有检查 `path->lsp` 和
   `path->rootSymbol()`，就创建 `BitSliceSource::Kind::Lsp`。
3. 对 `get_field(...)[3:0]`，路径根是函数调用，不能解析成 `ValueSymbol`。
   所固定的 slang 依赖 `6001e362f` 在 `ValuePath.cpp:158-250` 明确允许这种情况：
   非命名根使 `lsp` 为空，`rootSymbol()` 返回空指针。
4. `driveRhsLspSegment` 却假定所有 LSP 来源都有合法根符号并直接解引用。
   `addRvalue` 只是最终发生内存读取的位置。
5. 关闭位级分析会进入旧遍历路径；`noteReference` 会检查 `path.lsp`
   和根符号是否为空，因此绕过这次非法解引用。

### 独立最小复现

```systemverilog
module top(input logic [7:0] a, output logic [3:0] y);
    function automatic logic [7:0] identity(input logic [7:0] x);
        return x;
    endfunction
    always_comb y = identity(a)[3:0];
endmodule
```

保存为 `minimal.sv`，执行：

```bash
build/clang-debug/tools/driver/slang-netlist minimal.sv \
    --top top --find top.y --format json -j 1
```

已安装 CLI 与本次重建 Debug CLI 的结果相同：默认线程及 `-j 1` 均为
SIGSEGV（Python subprocess 返回 `-11`，对应 shell 139），stdout/stderr
均为 0 字节；增加 `--no-resolve-assign-bits` 后退出 0。

### 与二次开发的关系

本地 Git 引用比较（未联网刷新远端）：

- `main = f473f49`，`main...HEAD` 为 `0 / 5`。
- `origin/main = 9b25e8b`，`origin/main...HEAD` 为 `0 / 4`。
- 后一组额外提交为 `42a82e0`、`f01b289` 和两个合并提交
  `6495d7b`、`0866f5e`；不是本地可见主线之外约 10 个提交。
- `BitSliceList.cpp` 和 `BitSlice.hpp` 相对主线没有差异。
- 主线的 `driveRhsLspSegment` 同样无条件解引用根符号；`git blame`
  将该语句追溯到 `cb753471`。最初的 LSP 分类逻辑可追溯到 `e938dc6`。
  两个提交都已经包含在本地 `origin/main` 历史中。
- 二次开发 `42a82e0` 增加了依赖角色、精度和动态选择器处理，但这次
  触发的是常量 `[3:0]`，且新增逻辑之前就已取得空根符号。

另外将 `origin/main` 的 `9b25e8b` 用 `git archive` 导出到
`/tmp/slang-dsc-investigation/main`，独立编译其全部 `source/*.cpp` 和 CLI，
链接现有 Debug 构建的 slang/fmt 静态库（两分支固定的依赖版本一致），
未使用二次开发的 netlist 对象文件。构建脚本保留于
`/tmp/slang-dsc-investigation/build-main.py`。

该主线 CLI 实测结果：

| 输入 | 默认线程 | `-j 1` | `--no-resolve-assign-bits` |
|---|---|---|---|
| 6 行最小 SV | SIGSEGV | SIGSEGV | 退出 0 |
| 固定 DSC 快照 | SIGSEGV | SIGSEGV | 退出 0 |

源码与主线二进制对照均说明这是继承的位级路径缺陷，不能归因为二次开发新增的动态选择器逻辑。
上述历史追溯不是对历史上每个提交做二分，不能据此宣称已经确定最早可运行的坏提交。

### 修复方向与验收要求

应在 `BitSliceList::pushLsp` 建立分类不变量：只有具有有效 LSP 和根符号的
路径才能生成 `Kind::Lsp`；否则将整个表达式回退为 `Opaque`，继续遍历其依赖。
不能只在 `addRvalue` 返回或直接丢弃该片段，那会掩盖崩溃并遗漏依赖。
共享分类入口同时服务过程赋值和端口连接，应统一处理。

后续修复应覆盖函数返回值的位选、范围选择、过程赋值及端口连接，并验证输入依赖
仍被保留；对固定 DSC 快照用默认位级分析重跑单线程和默认线程。
本次定位未修改仓库 C++ 源码或 DSC RTL，也未提交文件。

## 2026-09-24 修复验证

空指针修复在 `BitSliceList::pushLsp` 拒绝无有效 LSP 或根符号的路径，并将整个
函数返回值切片作为 Opaque 表达式继续遍历。过程赋值和输入端口连接回归均确认
函数参数的输入依赖仍可达。

原始 DSC `--drivers dsc_encoder.u_lm.rec_wr_cpnt` 查询的后续失败是另一处位级路径
缺边。缩减后的独立输入如下，`sink.idx` 对应 DSC 中被用于动态写地址的
`rec_wr_cpnt`：

```systemverilog
module sink(input logic clk, input logic we, input logic [1:0] idx,
            input logic [7:0] data);
  logic [7:0] mem [0:3];
  always_ff @(posedge clk)
    if (we) mem[int'(idx)] <= data;
endmodule
module top(input logic clk, input logic we, input logic [1:0] sel,
           input logic [7:0] data);
  sink u(.clk(clk), .we(we), .idx(sel), .data(data));
endmodule
```

修复前，默认位级模式的 `--find top.u.idx` 返回范围 `[1:0]` 的输入端口，
但 `--drivers top.u.idx` 退出 6、报告 `invalid_query`；关闭位级分析时
`--drivers` 返回该端口作为 `[1:0]` 的局部 driver。DOT 对比显示第一处建图
差异是缺少从 `idx` 端口到写赋值节点的 `idx[1:0]` 边。位置在
`DataFlowAnalysis::driveLhsLspSegment`：原位级处理仅记录左值的写入，未遍历
左值路径中的动态选择器；legacy 遍历会读取该选择器。该结论不依赖
`hasSignal` 的节点存在性语义，也不需要修改查询入口。

修复后，位级路径以 `Address`、`Exact` 记录 `idx[1:0]` 选择器依赖；动态写入
的目标范围使用静态前缀的完整 `[31:0]`，而非误缩为 `[7:0]`。动态左值存在时，
右值整体遍历，避免拼接右值的后一个片段覆盖前一个片段的驱动记录；独立回归
确认 `{a,b}` 两项均能到达 `mem`。静态左值继续按位级片段处理。

固定 DSC 快照的原始 drivers 查询，默认位级分析及关闭位级分析、各自 `-j 1`
和默认线程四组均退出 0，JSON 的 `command=drivers`、`diagnostics=[]`、
`summary.complete=true`，返回同一个 `[1:0]` 输入端口
`dsc_encoder.u_lm.rec_wr_cpnt` 作为局部 driver。单比特 `[0]` 查询裁剪为
`[0:0]`；find 返回相同名称及 `[1:0]`；fan-in 包含父层
`dsc_encoder.lm_rec_cpnt`。该局部 driver 查询不证明更远上游每条依赖的位级
精度；动态写入地址的目标范围是保守范围。

## 2026-09-24 修复审阅与后续决策

工作区已加入空指针检查及 Opaque 回退，并有函数返回值位选/范围选择的分类、
过程赋值和端口连接测试。此次审阅构建成功，`[BitSliceList],[Bugs]` 测试
33 项、161 个断言通过。当前改动尚未提交。

固定 DSC 快照的单线程查询独立复核：默认位级模式的 find 找到 `[1:0]` 端口，
但 drivers 返回退出码 6，错误消息为 `could not find signal`；关闭位级分析后
两项查询均退出 0。CLI 的 drivers 存在性检查依赖带信号名的边，find 检查节点，
两者数据来源不同。第一处建图分歧仍待定位。

决策：有条件接受崩溃修复，原始查询恢复作为独立里程碑继续处理。
执行范围、诊断步骤和验收门槛见
[DSC 位级 drivers 查询恢复计划](plan-dsc-bit-drivers-2026-09-24.md)。
