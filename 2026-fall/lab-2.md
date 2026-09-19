# Lab 2：CPU 仿真

!!! info "开始之前"
    - **主题**：SoC 仿真集成与总线调试
    - **板卡**：无（纯 ModelSim 仿真）
    - **课时**：2 次实验课
    - **发布目录**：`lab2-ST/SoC_cv32e40p_ST/`
    - **完成标准**：ModelSim 打印 `RESULT: PASS`。**本实验不需要提交任何材料**

## 1. 实验目标

在此前的实验中，被测硬件是孤立的：testbench 直接驱动模块端口，观察波形即可判断对错。
真实芯片不是这样工作的——硬件挂在总线上，由 CPU 通过地址访问，而 CPU 执行的是编译好的程序。
在这种系统里，一个模块接错一根线、filelist 少写一个文件、存储的读延迟差一拍，
表现出来的现象都是"跑不起来"，而错误的表象与真正的原因之间往往隔得很远。

本实验提供一个结构完整、但被刻意破坏了四处的 SoC。任务是把它修复到能正常运行。

四处破坏分别对应 SoC 集成中四类典型问题：

| 任务 | 破坏点 | 对应的工程问题 |
|---|---|---|
| Task 1 | filelist 被删除 | 编译单元的组织与依赖顺序 |
| Task 2 | `bootram` 中没有引导指令 | 复位后的启动流程与机器码 |
| Task 3 | `axi2mem` 的端口未连接 | 总线桥接：协议侧与存储侧的信号对应 |
| Task 4 | `sram_ff` 的读写逻辑为空 | 行为级存储建模：同步读、字节使能、读延迟 |

四个任务全部修复后才会 PASS。任何一个未完成，现象都是"编译失败"或"TIMEOUT"。
因此本实验实际考察的是定位能力：在一组高度相似的失败现象中判断是哪一环断开。

Task 的编号顺序就是推荐的完成顺序，它沿着信号路径从 CPU 一路走到存储，
这样每完成一步都能单独验证。详见第 7 节。

## 2. 安装 ModelSim

在开始实验之前先装好仿真器。已经装过并且 `vsim` 可以在命令行直接调用的，可以跳过本节。

### 2.1 下载

| 项 | 内容 |
|---|---|
| 下载地址 | <https://disk.pku.edu.cn/link/AA2EBD4EF43A214E549727E1731FC03BA2> |
| 文件名 | `modelsim_se_2019.zip` |
| 链接有效期 | 2026-10-18 12:11 |

破解版安装步骤写在压缩包内的说明文件里，按其操作即可，安装路径为D:\modeltech64_2019.2。
本节只补充说明安装完成后**如何配置 PATH**，因为压缩包里的说明没有覆盖这一步，而后续所有命令都依赖它。

### 2.2 为什么要配 PATH

安装完成后，`vsim.exe`、`vlog.exe`、`vlib.exe` 这些程序位于安装目录的 `win64` 子目录下，
例如：

```
D:\modeltech64_2019.2\win64
```

如果不做任何配置，在终端里直接输入 `vsim` 会提示找不到命令，必须每次都写完整路径。
把这个 `win64` 目录加入 PATH 环境变量之后，就可以在任意目录直接输入 `vsim`。

### 2.3 配置步骤（Windows）

1. 先确认自己的 `win64` 目录在哪。打开安装目录，找到里面含有 `vsim.exe` 的那个
   `win64` 文件夹，在资源管理器地址栏复制它的完整路径。
   本文后续一律以 `D:\modeltech64_2019.2\win64` 为例，请替换成你自己的路径。

2. 按 `Win + R`，输入 `sysdm.cpl`，回车。

3. 切换到「高级」选项卡，点击右下角的「环境变量」。

4. 在上方的「用户变量」列表中选中 `Path`，点击「编辑」。
   （改用户变量即可，不需要管理员权限，也不会影响其他用户。）

5. 点击「新建」，把第 1 步复制的路径粘贴进去，然后一路点「确定」关闭所有窗口。

6. **关闭并重新打开终端**。环境变量的修改只对新开的终端生效，已经开着的窗口读不到。

### 2.4 验证

在新开的 PowerShell 里执行：

```powershell
vsim -version
```

看到类似下面的输出就说明配置成功：

```
Model Technology ModelSim SE-64 vsim 2019.2 Simulator
```

如果提示「无法将"vsim"项识别为 cmdlet、函数、脚本文件或可运行程序的名称」，
说明 PATH 没生效。按顺序检查：

- 终端是不是配置完之后新开的
- 粘贴进去的路径是不是 `win64` 目录本身，而不是它的上级目录
- 用 `where.exe vsim` 看系统到底在哪些目录里找过

### 2.5 不配 PATH 也可以

如果暂时配不好，也可以每次都用完整路径调用，效果完全一样：

```powershell
& D:\modeltech64_2019.2\win64\vsim.exe -c -do "do sim/scripts/run_sim.do"
```

本讲义后续的命令示例都写成了完整路径的形式。配好 PATH 之后，把
`& D:\modeltech64_2019.2\win64\vsim.exe` 这一段直接换成 `vsim` 即可。

### 2.6 关于 Git Bash

如果你用 Git Bash 而不是 PowerShell，它会继承 Windows 的 PATH，所以按 2.3 配置完同样有效。
用 `which vsim` 验证，应当输出类似 `/d/modeltech64_2019.2/win64/vsim` 的路径。

注意：PowerShell 里输入的 `bash` 有可能是 WSL 的 bash，而 WSL 是独立的 Linux 环境，
看不到 Windows 版 ModelSim。要用 Git Bash，请从开始菜单单独打开它。

## 3. 实验环境

| 项 | 说明 |
|---|---|
| 仿真器 | ModelSim / Questa（本文以 ModelSim 2019.2 为例，安装见第 2 节） |
| 板卡 | 无 |
| 编辑器 | 任意 |
| AI 助手 | 允许使用，注意事项见第 9 节 |
| 发布代码 | `lab2-ST/SoC_cv32e40p_ST/` |
| RISC-V 工具链 | 不需要安装，测试程序的机器码已随发布包提供 |

发布目录结构：

```
SoC_cv32e40p_ST/
├── cpu_cv32e40p/           CV32E40P RISC-V CPU（完整，不需修改）
├── soc/rtl/
│   ├── my_soc_top.sv       SoC 顶层        <- Task 3
│   ├── axi/                AXI xbar / adapter / axi2mem 等
│   ├── debug/              RISC-V debug module
│   ├── common_cells/       通用单元
│   ├── clk_rst/            时钟复位
│   ├── include/            地址空间常量 my_soc_pkg.sv
│   └── mem/
│       ├── bootram.sv      启动存储        <- Task 2
│       ├── my_mainmem.sv   主存包装
│       └── sram_ff.sv      行为级 SRAM     <- Task 4
└── sim/
    ├── filelists/          <- Task 1：此处应有 my_soc_tb.f
    ├── scripts/            自动化脚本
    ├── tb/
    │   ├── my_soc_tb.sv    顶层 testbench
    │   ├── main.c          测试程序源码（只读，供理解）
    │   ├── test_status.c/h PASS/FAIL 上报机制（只读）
    │   └── mainmem_init.hex 编译好的程序镜像
    └── README.md           仿真流程说明，建议先读一遍
```

需要修改的只有四处：`sim/filelists/`（新建文件）、`soc/rtl/my_soc_top.sv`、
`soc/rtl/mem/sram_ff.sv`、`soc/rtl/mem/bootram.sv`。

`sim/tb/` 和 `sim/scripts/` 下的文件不要修改。修改 testbench 使其直接打印 PASS
没有任何意义，也无法说明系统真的跑通了。

## 4. 背景知识

本节建议用 20 分钟扫读。其中 4.5、4.3、4.4 分别对应 Task 2、Task 3、Task 4，需要读懂。

### 4.1 SoC 结构

```
   ┌─────────────────────────┐                ┌─────────────────────────┐
   │        CV32E40P         │                │      Debug Module       │
   │      (RISC-V RV32)      │                │       (JTAG/DMI)        │
   ├────────────┬────────────┤                ├─────────────────────────┤
   │   instr    │    data    │                │         master          │
   └─────┬──────┴──────┬─────┘                └────────────┬────────────┘
         │             │                                   │
   ┌─────┴─────┐ ┌─────┴─────┐                       ┌─────┴─────┐
   │axi_adapter│ │axi_adapter│                       │axi_adapter│
   └─────┬─────┘ └─────┬─────┘                       └─────┬─────┘
         │             │                                   │
     slave[0]      slave[1]                             slave[2]
         │             │                                   │
         └─────────────┴──────────┬────────────────────────┘
                                  │
            ┌─────────────────────┴─────────────────────┐
            │                 axi_xbar                  │
            └─────┬───────────────┬───────────────┬─────┘
                  │               │               │
              master[0]       master[1]       master[2]
                  │               │               │
           ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
           │   axi2mem   │ │   axi2mem   │ │   axi2mem   │
           │ i_axi2boot  │ │ i_axi2sram  │ │i_dm_axi2mem │
           └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                  │               │               │
           ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
           │   bootram   │ │ my_mainmem  │ │   dm_top    │
           │ 0x0001_0000 │ │ 0x8000_0000 │ │ 0x0000_0000 │
           └─────────────┘ └──────┬──────┘ └─────────────┘
                                  │
                          ┌───────┴───────┐
                          │    sram_ff    │
                          │ 2048 x 32-bit │
                          └───────────────┘
```

读图要点：

- 上半部分是三个 **master**（发起请求的一方）：CPU 的取指口（instr）、CPU 的访存口（data）、
  以及 debug module。每个 master 经过一个 `axi_adapter` 转成 AXI 事务，接到 xbar 的
  `slave[0..2]` 端口上。
- 下半部分是三个 **slave**（响应请求的一方）：bootram、主存、debug module 的从口。
  它们挂在 xbar 的 `master[0..2]` 端口上，每一路都经过一个 `axi2mem` 把 AXI 转成
  类存储器信号。**Task 3 要修的就是中间那一路 `i_axi2sram` 与 `my_mainmem` 之间的连线。**
- `axi_xbar` 根据请求地址决定把它路由到哪个 slave。
- debug module 在图上出现了两次：它既是 master（通过 `slave[2]` 主动访问存储，
  用于调试时读写内存），也是 slave（通过 `master[2]` 被 CPU 访问，用于读写调试寄存器）。
  本实验不涉及调试功能，这一路可以先不管。
- 主存的实际存储体是 `sram_ff`，`my_mainmem` 只是一层包装，负责把字节地址转成字地址。

地址空间定义在 `soc/rtl/include/my_soc_pkg.sv`：

| 区间 | 目标 | 说明 |
|---|---|---|
| `0x0000_0000 – 0x0000_0FFF` | debug module | 调试用，本实验不涉及 |
| `0x0001_0000 – 0x0001_FFFF` | bootram | CPU 复位后从这里取第一条指令 |
| `0x8000_0000 – 0x8FFF_FFFF` | 主存 | 实际只实现 8 KB，测试程序运行在这里 |

落在这三段之外的访问会被 xbar 判给 `axi_err_slv`，返回 AXI 错误响应。

### 4.2 AXI：Master、Slave 与 Xbar

AXI（Advanced eXtensible Interface）是一种总线协议。本实验不要求掌握协议细节，
但需要建立三个概念：

- **Master** 发起读写请求，请求中包含操作类型（读或写）、地址、写数据。
  本 SoC 有 3 个 master：CPU 取指口、CPU 访存口、debug module。
- **Slave** 响应请求，把读出的数据返回给对应的 master。
  本 SoC 有 3 个 slave：bootram、主存、debug module 的从口。
- **Xbar（Crossbar）** 是多 master 与多 slave 之间的交换矩阵。每个 slave 被分配一段地址，
  xbar 根据请求地址路由到正确的 slave。每个请求带一个 ID，用于分辨响应应返回给哪个 master。

需要注意 `my_soc_top.sv` 中两组 `AXI_BUS` 接口数组的命名方向：它们是站在 xbar 自身的角度命名的。
`slave[]` 表示"xbar 作为 slave 去连接 CPU 那些 master"，`master[]` 表示"xbar 作为 master
去连接存储那些 slave"。因此 `slave[0..2]` 实际连的是 CPU 和 debug module，
`master[0..2]` 连的才是 bootram 和主存。

另外，CPU 的取指和访存是两个独立端口，但主存只有一块且是单端口的，
两者的访问在 xbar 内部由 `rr_arb_tree` 轮询仲裁、在时间上串行化。

### 4.3 axi2mem：从 AXI 到类存储器端口

AXI 的握手信号很多（`ar`/`aw`/`w`/`r`/`b` 五个通道，每个通道有 valid/ready/addr/len/size/burst 等），
让一块 SRAM 直接实现 AXI 从机并不现实。因此中间加了一个桥接模块 `axi2mem`。

它左侧接 AXI，右侧输出一组极简的类存储器信号：

```
         ┌─────────────────┐
         │                 │──> req_o    本拍有访问请求
 AXI ───>│    axi2mem      │──> we_o     1 = 写，0 = 读
         │                 │──> addr_o   字节地址（32-bit）
         │                 │──> be_o     字节使能（4-bit，写时哪几个字节有效）
         │                 │──> data_o   要写进存储的数据
         │                 │<── data_i   存储读出的数据
         └─────────────────┘
```

注意 `data_o` 与 `data_i` 的 `o`/`i` 后缀是站在 `axi2mem` 自身角度命名的。
`data_o` 是 `axi2mem` 输出给存储的写数据，`data_i` 是存储输入给它的读数据。
连接到存储一侧时，分别对应存储的 `wdata_i` 和 `rdata_o`。

`my_soc_top.sv` 中已有一个连接完整的样板：`i_axi2boot`（bootram 那一路）。
Task 3 需要照此把 `i_axi2sram` 连接起来。

### 4.4 行为级 SRAM 与读延迟

工艺库中的 SRAM 宏是工艺相关的，不便于 RTL 仿真，因此这里用一个 reg 数组建模：

```systemverilog
reg [DataWidth-1:0] memory [0:(1<<AddrWidth)-1];
```

它必须满足三条契约，否则与 `axi2mem` 的时序对不上：

1. **同步读，下一拍有效。** 请求出现在 cycle T，读数据要到 cycle T+1 才出现在 `data_o` 上。
   实现方式是读出后先打一拍寄存器：`data_out_reg <= memory[addr_i];`。
   若写成组合读（`assign data_o = memory[addr_i];`），仿真中看似更快，
   但与 `axi2mem` 的握手对不上，会读到错位的数据。
2. **字节使能写。** `wen_i[3:0]` 每一位控制一个字节。CPU 执行 `sb` / `sh` 指令时
   只写 1 个或 2 个字节，若忽略 `wen_i` 而整字写入，会破坏相邻字节。
3. **单端口。** 读写共用同一个 `addr_i`，同一拍只有一个地址。

关于地址位宽：`axi2mem` 输出的 `addr_o` 是字节地址，而 `memory[]` 按字索引。
`my_mainmem.sv` 中已完成转换（`addr_i[ADDR_W+1:2]`，丢弃最低 2 位），不需要重复处理。

### 4.5 CPU 的启动流程

CV32E40P 复位后从 `0x0001_0000` 取第一条指令，也就是 bootram 的第 0 个字。
而测试程序装载在主存 `0x8000_0000`（由 `$readmemh` 加载 `mainmem_init.hex`）。
因此 bootram 中必须放一段引导代码，把 PC 跳转到主存：

```asm
lui  t0, 0x80000     # t0 = 0x80000000
addi t0, t0, 0       # 微调低 12 位（此处为 0）
jr   t0              # PC <- t0，跳入主存
```

`bootram.sv` 用 `mem_q[0..15]` 这 16 个字模拟该存储，复位时把机器码直接写入。
Task 2 需要把这三条指令翻译成机器码填入。

`mem_q[3..15]` 必须保持为 0。CPU 的取指带预取，会顺带读到后面几个字；
若那里是 `X`，CPU 会因取到非法指令而卡死或进入异常循环。

汇编转机器码可使用在线工具：<https://riscvasm.lucasteske.dev>

### 4.6 filelist

仿真器需要知道编译哪些文件、按什么顺序编译。filelist 就是这个清单，
`vlog -f <filelist>` 按行读取。

两条硬性规则：

1. **顺序**：被依赖的文件在前。对 SystemVerilog 而言：
   - `package`（`*_pkg.sv`）必须在所有 import 它的文件之前
   - `interface`（本工程是 `axi_intf.sv`）必须在使用它的模块之前
   - testbench 放在最后
   - 普通模块之间 ModelSim 不强制顺序（它会做两遍扫描），但包和接口必须在前
2. **头文件目录**用 `+incdir+<dir>` 声明。本工程的 `` `include `` 走 `soc/rtl/include/`，
   因此 filelist 需要包含 `+incdir+soc/rtl/include`。

路径相对于仿真器的工作目录（即 `SoC_cv32e40p_ST/`），不是相对于 filelist 自身。

### 4.7 PASS / FAIL 的判定机制

本实验与以往实验的一个显著差别：判定结果不是由 testbench 直接计算的，而是由 CPU 运行程序产生的。

`sim/tb/main.c` 执行一些整数运算和内存读写，最后调用
`publish_test_status()`（见 `sim/tb/test_status.c`），依次写入三个全局变量：

| 变量 | 值 |
|---|---|
| `g_test_status_fail_count` | 失败计数 |
| `g_test_status_magic` | `0x600DCAFE` = PASS，`0xDEADC0DE` = FAIL |
| `g_test_status_done_magic` | `0xC0DEDEAD` = 程序执行完毕 |

这些变量位于 `.bss`，地址由链接器分配，更换程序重新编译后会变化。因此 `my_soc_tb.sv`
不检查固定地址，而是监听主存的写端口：观察到向主存写入这几个 magic 值即判定结果。

这意味着看到 PASS 就说明 CPU 完整执行了一个真实程序——取指、访存、栈操作、函数调用均正常。
反过来，TIMEOUT 说明 CPU 没有执行到终点：可能卡在取指，可能卡在访存，也可能根本没有启动。
Task 2、3、4 的失败现象都是 TIMEOUT，这正是本实验的难点所在。

## 5. 仿真流程

完整说明见 `sim/README.md`，本节只列常用操作。所有命令都在 `SoC_cv32e40p_ST/` 目录下执行。

### 5.1 运行仿真

PowerShell / cmd：

```powershell
cd <你的路径>\lab2-ST\SoC_cv32e40p_ST
& D:\modeltech64_2019.2\win64\vsim.exe -c -do "do sim/scripts/run_sim.do"
```

Git Bash：

```bash
cd <你的路径>/lab2-ST/SoC_cv32e40p_ST
bash sim/scripts/run_sim.sh
```

脚本会依次完成：建立 `work` 库、用 `vlog` 编译整个 filelist、用 `vsim` 运行到结束、
打印 `RESULT: PASS / FAIL / TIMEOUT`。完整日志保存在 `sim/out/sim.log`。

运行结束后默认不退出 ModelSim，设计仍然加载着，便于继续 `examine` 或查看波形。
输入 `quit -f` 退出。

### 5.2 查看波形

去掉 `-c` 即可进入图形界面：

```powershell
& D:\modeltech64_2019.2\win64\vsim.exe -do "do sim/scripts/run_sim.do"
```

GUI 模式下脚本会在 `run -all` 之前自动执行 `log -r /*`，把全部信号记入 WLF。
这一点很关键：ModelSim 只绘制运行期间被记录过的信号，仿真结束后再 `add wave`
添加的信号只会显示 "No Data"。

### 5.3 清理

修改 RTL 后如果编译行为异常，先清理再全量重新编译：

```powershell
.\sim\scripts\clean.ps1
```
```bash
./sim/scripts/clean.sh
```

会删除 `work/`、`transcript`、`modelsim.ini`、`*.wlf`、`sim/out/`，源码不受影响。

### 5.4 手动分步执行

排查编译问题时可以拆开执行，与脚本所做的事情一致：

```tcl
vlib work
vmap work work
vlog -sv -f sim/filelists/my_soc_tb.f
vsim -suppress 12110 -novopt -onfinish stop work.my_soc_tb
run -all
examine -radix decimal /my_soc_tb/exit_code
```

- `-novopt` 用于绕开 ModelSim 2019.2 在 `lzc.sv` 上已知的 vopt 问题，
  `-suppress 12110` 屏蔽 `-novopt` 的弃用警告
- `-onfinish stop` 让 `$finish` 只是停下而不退出 vsim，这样才能读取 `exit_code`

## 6. 实验任务

### Task 1 · 重建 filelist

**现象**：直接运行脚本会看到

```
ERROR: 找不到 filelist 'sim/filelists/my_soc_tb.f'
```

**要求**：新建 `sim/filelists/my_soc_tb.f`，把编译所需的全部文件按正确顺序列入。

**提示**：

1. 需要包含 `+incdir+soc/rtl/include`
2. 所需文件分布在四处：`cpu_cv32e40p/include/`、`cpu_cv32e40p/rtl/`、
   `soc/rtl/**`、`sim/tb/my_soc_tb.sv`
3. 所有 `*_pkg.sv` 放在最前。本工程涉及的包有：
   `cv32e40p_pkg`、`cv32e40p_apu_core_pkg`、`cv32e40p_fpu_pkg`、
   `my_soc_pkg`、`axi_pkg`、`cf_math_pkg`、`dm_pkg`
4. `axi_intf.sv` 定义的是 interface，也要放在使用它的模块之前
5. `sim/tb/my_soc_tb.sv` 放在最后一行
6. `cpu_cv32e40p/vendor/` 下的内容不需要编译（浮点单元等可选组件，本配置未启用）
7. `soc/rtl/mem/bootrom.sv` 也不需要——真正接入 SoC 的是 `bootram.sv`

**自检**：`vlog` 能够编译通过，没有 `Cannot resolve module` 一类的错误。
编译通过不代表能运行，但可以确认 Task 1 完成。

查找文件可用 `find soc cpu_cv32e40p -name "*.sv"`（Git Bash）或
`Get-ChildItem -Recurse -Filter *.sv`（PowerShell）。不要把找到的文件全部加入，
加入不该编译的文件会导致重复定义或语法错误。

### Task 2 · 编写 bootram 的引导指令

**位置**：`soc/rtl/mem/bootram.sv`

**现象**：编译通过，仿真 TIMEOUT。

**原理**：CPU 复位后从 `0x0001_0000` 取指，取到的全是 0。
`0x00000000` 不是合法的 RISC-V 指令，CPU 会进入异常处理，无法到达主存。

**要求**：把 4.5 节的三条汇编指令翻译成机器码，填入 `mem_q[0]`、`mem_q[1]`、`mem_q[2]`。

**提示**：

- 可用 <https://riscvasm.lucasteske.dev> 将汇编转为十六进制机器码
- 也可按 RISC-V 编码手算：`lui` 是 U 型（opcode `0110111`），
  `addi` 和 `jalr` 是 I 型（opcode `0010011` / `1100111`）。
  `jr t0` 是 `jalr x0, 0(t0)` 的伪指令写法
- `mem_q[0]` 是 CPU 取的第一条指令
- 代码中已有一个把 `mem_q[0..15]` 全部清零的循环，三条指令写在该循环之后
  （非阻塞赋值中后写的生效）

**自检**：

在 GUI 中观察 `/my_soc_tb/dut/instr_addr`（取指地址）和
`/my_soc_tb/dut/instr_rdata`（CPU 取到的指令）。
正确时取指地址应从 `0x00010000` 开始，取到所填的三条指令，
随后跳转到 `0x80000000` 继续取指。

这一步的自检不依赖后面两个 Task：跳转本身只取决于 bootram 里的指令，
即使主存那一路还没通，`instr_addr` 也会如实显示跳到了 `0x80000000`
（只是取回来的指令会是 `X`）。

### Task 3 · 连接 axi2mem 到主存

**位置**：`soc/rtl/my_soc_top.sv` 中的 `i_axi2sram` 例化。

**现象**：编译通过（未连接的端口在 SystemVerilog 中是合法的），但仿真 TIMEOUT。

**原理**：`i_axi2sram` 右侧的 6 个类存储器端口全部未连接，
因此 `sram_req` 等信号没有驱动源，在仿真中为 `X`。
主存收不到有效请求，CPU 取指拿不到数据，程序无法运行。

**要求**：把 `i_axi2sram` 的 6 个端口连接到主存信号。
上方 `my_mainmem i_mainmem (...)` 已经例化完成，使用的信号名为
`sram_req` / `sram_we` / `sram_addr` / `sram_be` / `sram_wdata` / `sram_rdata`。

**提示**：

- 同一文件中的 `i_axi2boot`（bootram 那一路）是已连接完整的样板，可参照其结构
- 但注意两处差异：bootram 那一路的 `.be_o ()` 是有意空置的（bootram 不支持字节使能），
  主存这一路的 `be_o` 必须连接，否则 `sb` / `sh` 指令会写错
- 再次确认 4.3 节中 `data_o` / `data_i` 的方向

**自检**：在波形中观察 `/my_soc_tb/dut/sram_req`。
连接正确时，它应在 CPU 跳进主存之后开始出现高电平脉冲（CPU 正在取指）；
若仍为 `X` 或恒为 `0`，说明未连接或连接错误。

注意这一步的自检**需要 Task 2 已经完成**。如果 bootram 还没写引导指令，
CPU 会停在 `0x0001_0000` 附近的异常循环里，根本不会访问 `0x8000_0000`，
`sram_req` 自然不会有任何脉冲——此时看到没有脉冲，不能说明 Task 3 做错了。

### Task 4 · 补完行为级 SRAM

**位置**：`soc/rtl/mem/sram_ff.sv`

**现象**：编译通过，仿真 TIMEOUT。

**原理**：`memory` 数组和 `data_out_reg` 都已声明，`$readmemh` 也已把程序装入，
但没有任何逻辑把 `memory` 的内容送到 `data_o`，也没有写入路径。
CPU 取指始终读到 `X`。

**要求**：编写一个 `always @(posedge clk_i)` 块，实现：

1. `req_i` 为高时，把 `memory[addr_i]` 读出并打一拍送入 `data_out_reg`
2. 按 `wen_i[3:0]` 的四位分别写入 `memory[addr_i]` 的四个字节

**提示**：

- 参照 4.4 节的三条契约
- 字节使能写的写法：`if (wen_i[0]) memory[addr_i][7:0] <= data_i[7:0];`，四个字节各一句
- 读和写在同一个 `if (req_i)` 分支内完成，不要写成 if-else——
  同一拍既可能要返回读数据，也可能要写入
- 使用非阻塞赋值 `<=`

**自检**：

这是最后一块。前三个 Task 都正确时，补完本 Task 就应当直接跑出 `RESULT: PASS`。

如果仍然 TIMEOUT，先确认程序确实装进了主存：运行仿真后（或在 GUI 中 `run -all` 后），
菜单 `View → Memory List`，双击 `/my_soc_tb/dut/i_mainmem/i_mainmem/memory`，
应能看到与 `sim/tb/mainmem_init.hex` 一致的内容——
第 1 行 `00002117` 对应地址 `0x80000000`，第 2 行对应 `0x80000004`，依此类推。

注意：能看到正确内容只说明 `$readmemh` 成功（这部分未被破坏），
不代表读写逻辑正确。接着看 `/my_soc_tb/dut/instr_rdata`，
确认 CPU 取回的指令与主存内容一致、且不是 `X`。

## 7. 推进顺序与分步自检

Task 2、3、4 的失败现象完全相同（都是 TIMEOUT），因此无法通过"跑一次看报错"来判断问题所在。
第 6 节的四个 Task **已经按推荐顺序编号**，Step N 就是 Task N，照着顺序做即可：

```
Step 1 (Task 1)  编写 filelist         目标：vlog 编译通过
Step 2 (Task 2)  编写 bootram 引导指令  目标：instr_addr 从 0x00010000 跳到 0x80000000
Step 3 (Task 3)  连接 axi2mem           目标：sram_req 出现高电平脉冲
Step 4 (Task 4)  补完 sram_ff 读写逻辑  目标：RESULT: PASS
```

这个顺序不是随意排的，它沿着信号路径逐段往下推进：
**CPU 能否启动 → 请求能否传到存储 → 存储能否正确响应**。

按这个顺序做的好处是**每一步都能单独验证**。反过来，如果打乱顺序，自检就会失效：

- 先做 Task 3（连 axi2mem）而 Task 2 没做，`sram_req` 不会有脉冲——
  因为 CPU 还没跳进主存，压根没发出请求。此时你无法判断连线是对是错。
- 先做 Task 4（补 sram_ff）而 Task 3 没做，`req_i` 一直是 `X`，
  读写逻辑根本不会被触发，同样无法验证。

因此如果中途卡住，先回头确认前一步的自检是否真的通过了，再往下排查。

### 关键观察信号

| 想确认什么 | 观察信号 |
|---|---|
| CPU 在取哪条指令 | `/my_soc_tb/dut/instr_addr`、`/my_soc_tb/dut/instr_rdata` |
| 主存是否收到请求 | `/my_soc_tb/dut/sram_req`、`sram_we`、`sram_addr` |
| 主存返回了什么 | `/my_soc_tb/dut/sram_rdata` |
| bootram 那一路 | `/my_soc_tb/dut/boot_req`、`boot_addr`、`boot_rdata` |
| testbench 是否抓到 magic | `/my_soc_tb/seen_pass`、`seen_fail`、`exit_code` |
| 已运行多少周期 | `/my_soc_tb/cycle_count` |

添加波形（GUI 模式下脚本已自动执行 `log -r /*`，因此随时可以添加）：

```tcl
add wave -r /my_soc_tb/dut/sram_*
add wave /my_soc_tb/dut/instr_addr
add wave /my_soc_tb/dut/instr_rdata
```

## 8. 完成标志

```
==== vlog: 编译 sim/filelists/my_soc_tb.f ====
...
[85000] reset released, watching SRAM write port for test_status magics
[XXXXXX] TEST_STATUS_PASS written to 0x800001XX
[XXXXXX] TEST_STATUS_DONE written to 0x800001XX
[XXXXXX] PASS: test program finished cleanly after NNNN cycles
==== RESULT: PASS ====
```

看到 `RESULT: PASS` 即完成本实验。

## 9. 关于使用 AI

本实验允许使用 AI 辅助，但需要了解它在这类任务上的实际表现：

- 擅长：解释一段 RTL 的功能、解释 AXI 概念、把汇编转成机器码、
  编写字节使能这类模板化代码
- 不擅长：跨文件的集成问题。"为什么我的仿真 TIMEOUT"这类问题，
  AI 看不到波形、也不知道你改了哪些文件，通常只会给出一组听起来合理但没有针对性的猜测
- 常见错误：把 `rst_ni`（低有效）当作 `rst`、位宽写错、把同步读写成组合读、
  非阻塞赋值与阻塞赋值混用

有效的做法是把问题收敛到具体信号再提问，例如：

> 我在 ModelSim 中看到 `sram_req` 一直是 X。这个信号在 my_soc_top.sv 中声明为
> `logic sram_req;`，应由 axi2mem 的 req_o 驱动。X 说明什么？应该从哪个方向排查？

而不是：

> 我的 SoC 仿真 TIMEOUT 了，怎么办？

AI 给出的 RTL 建议需要自己判断是否正确，尤其是时序相关的部分。

## 10. 思考题

以下问题不需要提交，供自行检验对本实验的理解程度。

1. CPU 的取指口和访存口是两个独立的 AXI master，但主存只有一块单端口 SRAM。
   它们是如何共享这块存储的？带来什么性能代价？这个 SoC 中有没有缓解措施？

2. `sram_ff` 的读被要求"下一拍有效"。如果写成组合读
   （`assign data_o = memory[addr_i];`），会发生什么？
   这种错误在波形上有什么特征？为什么它比"完全不工作"更难排查？

3. Task 3 中如果把 `be_o` 空置不接（像 bootram 那一路那样），仿真还能 PASS 吗？
   为什么？提示：查看 `main.c` 中有没有小于一个字的写操作。

4. `bootram.sv` 中如果不把 `mem_q[3..15]` 清零会发生什么？
   为什么 CPU 会去读第 3 个字之后的内容——它在第 2 个字就已经跳走了？

5. testbench 判定 PASS 的方式是监听主存写端口上的 magic 值，而不是检查某个固定地址的内容。
   这两种做法各有什么优劣？更换测试程序重新编译后，哪一种不需要修改 testbench？

## 11. 常见问题

### Q1 `vlog` 报 `Cannot resolve module 'xxx'`

filelist 遗漏了文件。根据报错中的模块名，在 `soc/rtl/` 和 `cpu_cv32e40p/rtl/` 下
找到对应的 `.sv` 加入。

### Q2 `vlog` 报 `Package 'xxx_pkg' not found` 或 `Unknown type`

包的顺序不对。所有 `*_pkg.sv` 和 `axi_intf.sv` 必须排在使用它们的文件之前。

### Q3 `vlog` 报某个文件的 `` `include `` 找不到头文件

filelist 中缺少 `+incdir+soc/rtl/include`。

### Q4 编译通过，但 `vsim` 报 `Instantiation of 'my_soc_top' failed`

多半是 `my_soc_top.sv` 中改出了语法问题，或 filelist 遗漏了某个被例化的模块。
查看 `sim/out/sim.log` 中 `vlog` 阶段有没有 warning。

### Q5 一直 TIMEOUT，但四个 Task 都已完成

按第 7 节的信号表逐项确认。最常见的三种错误：

- `axi2mem` 的 `data_o` / `data_i` 接反（现象：取指拿到的是无意义的数据）
- `sram_ff` 写成了组合读（现象：数据比预期早一拍，CPU 执行紊乱）
- bootram 的机器码算错（现象：`instr_addr` 跳到了一个异常的地址）

### Q6 修改了 RTL，但仿真结果没有变化

`work` 库中仍是旧的编译结果。先清理再全量重新编译：

```powershell
.\sim\scripts\clean.ps1
& D:\modeltech64_2019.2\win64\vsim.exe -c -do "do sim/scripts/run_sim.do"
```

### Q7 波形窗口中信号显示 "No Data"

在 `run -all` 之后才执行了 `add wave`。ModelSim 只绘制运行期间记录过的信号。
使用 GUI 模式运行脚本（去掉 `-c`），脚本会在 run 之前自动执行 `log -r /*`。

### Q8 PowerShell 中 `bash sim/scripts/run_sim.sh` 提示找不到 vsim

PowerShell 中的 `bash` 可能是 WSL 的 bash，看不到 Windows 版 ModelSim。
改用 Git Bash，或直接使用 `.do` 方式（5.1 节）。

## 12. 延伸阅读

- `soc/rtl/axi/` —— AXI4 互联的完整实现（`axi_xbar`、`axi_demux`、`axi_mux`、`axi_adapter`）
- `cpu_cv32e40p/rtl/` —— PULP 开源 RISC-V 核 CV32E40P 的完整源码。
  注意它没有 cache，`cv32e40p_prefetch_buffer.sv` 中只有一个深度为 2 的 FIFO
- `soc/rtl/debug/` —— 遵循 RISC-V Debug Spec 0.13.2 的调试模块
