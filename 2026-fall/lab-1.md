# Lab 1：AI 辅助的简单 NPU 设计

!!! info "开始之前"
    - **前置**：[Lab 0](lab-0.md)——Claude Code 环境、Icarus Verilog、波形工具都已装好
    - **工具**：Claude Code（或任意你顺手的 AI Agent）+ `iverilog` / `vvp` + GTKWave / Surfer
    - **本 Lab 不需要提交报告**，跑通验收 testbench 即为完成

## 实验目的

Lab 0 里你让 AI 写了一个 2 路 MAC 阵列，那是个「玩具」：没有控制、没有状态机，算完就完了。

本 Lab 你要在 AI 的辅助下，做一个**真正像加速器的东西**——一个 4×4 矩阵乘法计算核（下文统称 `simple_npu`）：

1. 它有**一对握手信号 `start / done`**：外部把矩阵装好、拉一下 `start`，它自己跑，算完举手说 `done`
2. 它有**状态机**，收到 `start` 之后自己跑 4 拍，跑完停在 DONE 等下一次
3. 它有 **16 个 PE 并行**工作，一次算出整个 4×4 结果矩阵，结果**驻留**在 PE 里等外部来取

完成本 Lab 后你将掌握：

- 矩阵乘加速器最基本的数据流组织方式（output-stationary + 并行 PE 阵列）
- 「计算核 / 总线壳」分层——加速器 IP 的标准组织方式：核只管算，怎么挂到总线上是另一层的事
- 用 AI Agent 完成一个**有明确硬约束**的 RTL 设计任务，并且**有能力验证 AI 写得对不对**

## 实验环境与实验包

### 环境

沿用 Lab 0，不需要装任何新东西：

```bash
iverilog -V
vvp -V
```

两条命令都能打印版本号即可（本讲义在 Icarus Verilog 12.0 上验证通过，11.0 同样可用）。

### 实验包

助教发布的 `lab1_release/` 解压后长这样：

```text
lab1_release/
├── README.md                  ← 快速开始 + 常用命令
├── lab1.md                    ← 本讲义（内容与本页相同）
├── rtl/
│   ├── simple_npu_core.sv     ← 骨架，你要填的目标
│   └── simple_npu_pe.sv       ← 骨架，你要填的目标
└── tb/
    └── tb_npu_check.sv        ← 验收 testbench（助教提供，禁止修改）
```

建议把它放到一个**不含中文和空格**的工作目录，例如：

**Windows（PowerShell）：**

```powershell
New-Item -ItemType Directory -Force D:\Projects\AdvanceChipLAB1
Set-Location D:\Projects\AdvanceChipLAB1
```

**macOS（Terminal）：**

```bash
mkdir -p ~/Projects/AdvanceChipLAB1
cd ~/Projects/AdvanceChipLAB1
```

把 `rtl/` 和 `tb/` 两个目录放进去，后续所有命令都在这个目录下执行。

## 背景知识（20 分钟扫读）

### 1. NPU 在算什么

神经网络推理的绝大部分计算量是**矩阵乘法**。所谓 NPU（Neural Processing Unit），核心就是一个专门快速做矩阵乘的电路。真实的 NPU 会做 128×128 甚至更大，还要处理定点量化、数据搬运、多层调度；本 Lab 把它砍到最小可理解的规模：

```text
C = A · B

C[i][j] = Σ(k=0..3) A[i][k] × B[k][j]        i, j, k ∈ {0,1,2,3}
```

A、B、C 都是 4×4，元素是 **unsigned 4-bit**（0~15）。

### 2. 从 Lab 0 的 MAC 阵列到 NPU

| Lab 0 做的 | Lab 1 怎么变 |
|---|---|
| 2 路独立 MAC，`acc <= acc + a*b` | **16 路**并行，每路负责一个 `C[i][j]` |
| 输入 `a0/b0/a1/b1` 各自一个端口 | **整个矩阵打包成一个端口**，靠**下标约定**区分 16 个元素 |
| 没有控制，`en` 拉高就一直累加 | **3 态状态机**：IDLE → RUN(4 拍) → DONE，`start` 触发、`done` 报告 |
| testbench 直接读 `acc0/acc1` | 16 个 `acc` 打包成 `out` 端口整体读出 |
| `rst` 同步复位、高有效 | `rst_ni` **异步复位、低有效**（注意反过来了！） |

**PE 内部还是 Lab 0 那个 1 级 MAC**——`acc <= acc + a*b`，一拍一次，不做流水线。所以你在 Lab 0 已经写过这个 Lab 最核心的那一行代码了，剩下的全是「怎么把它组织起来、怎么控制它」。

### 3. 数据流：output-stationary + 16 PE 并行

「谁负责算什么」有很多种分法，本 Lab 用最好理解的一种：

> **output-stationary（输出驻留）**：每个 PE 死死锁定一个输出元素 `C[i][j]`，从头算到尾不换人。

4×4 的输出矩阵有 16 个元素，所以放 **16 个 PE**，排成 4 行 4 列。`PE[i][j]` 只负责 `C[i][j]`。

那 4 个 cycle 里发生了什么？

```text
cycle 0（k=0）：所有 16 个 PE 同时做  acc += A[i][0] × B[0][j]
cycle 1（k=1）：所有 16 个 PE 同时做  acc += A[i][1] × B[1][j]
cycle 2（k=2）：所有 16 个 PE 同时做  acc += A[i][2] × B[2][j]
cycle 3（k=3）：所有 16 个 PE 同时做  acc += A[i][3] × B[3][j]
                                     ↑ 累加完毕，acc 就是 C[i][j]
```

**4 个 cycle 算完整个 4×4 矩阵乘**（共 64 次乘加）。

关键在于同一拍里数据是怎么广播的。第 k 拍：

- `PE[i][j]` 的 a 输入 = `A[i][k]`——只和**行号 i** 有关，所以 **A 的第 k 列被广播给 4 行**
- `PE[i][j]` 的 b 输入 = `B[k][j]`——只和**列号 j** 有关，所以 **B 的第 k 行被广播给 4 列**

```text
                 B[k][0]  B[k][1]  B[k][2]  B[k][3]
                    ↓        ↓        ↓        ↓
       A[0][k] → [PE00]   [PE01]   [PE02]   [PE03]
       A[1][k] → [PE10]   [PE11]   [PE12]   [PE13]
       A[2][k] → [PE20]   [PE21]   [PE22]   [PE23]
       A[3][k] → [PE30]   [PE31]   [PE32]   [PE33]
```

每拍只需要从 A 里取 4 个数、从 B 里取 4 个数，就能喂饱 16 个乘法器——这就是矩阵乘天生适合做加速器的原因：**计算量 O(n³)，数据量 O(n²)**。

「输出驻留」还有一层意思：**算完之后结果就躺在 PE 的累加寄存器里**，外面什么时候来取都行，取之前哪怕把输入矩阵换掉，结果也不会变。验收 tb 专门考这一点（见 §6）。

### 4. 列优先（column-major）打包与 packed 数组

一个 4×4 矩阵有 16 个元素，本 Lab 把它们**拼成一个端口**：

```systemverilog
input logic [15:0][3:0] act;    // 16 个 4-bit 元素，总共 64 bit
```

这叫 **packed 数组**：`act[5]` 就是第 5 个元素（4 bit），`act` 整体是一根 64 bit 的线。那第几个元素对应矩阵的哪个位置？本 Lab 约定 **列优先**：

```text
act[j*4 + i] = A[i][j]
```

也就是说 `act[0..3]` 是 A 的第 0 列（`A[0][0], A[1][0], A[2][0], A[3][0]`），`act[4..7]` 是第 1 列，以此类推。`wgt`（矩阵 B）和 `out`（矩阵 C）用同样的规则。

> 💡 **为什么强调这个**：这是本 Lab 最容易出错的地方。i 和 j 弄反，仿真会跑出一个「看起来很对但就是不等」的转置结果。验收 tb 的 TEST1 专门抓这个。

有了这个约定，硬件里查表就很直接（下标可以是变量，`act[k_cnt*4 + i]` 这种写法是合法的）：

```text
A[i][k] = act[k*4 + i]
B[k][j] = wgt[j*4 + k]
```

### 5. 核与壳：为什么这个 Lab 不做总线

真实世界里，加速器不会把 16 个矩阵元素做成 16 组端口拉到芯片外面——那样引脚会爆炸。工业界的做法是分两层：

```text
┌─────────────────────────────────────────────────┐
│ 壳（bus wrapper）                                │
│   地址译码 · ACT/WGT 寄存器组 · 读数据打一拍     │
│   CONTROL/STATUS 寄存器 · 挂 AXI / memory-like   │
│      ┌──────────────────────────────────┐        │
│      │ 核（simple_npu_core）  ← 本 Lab  │        │
│      │   start/done · 16 PE · FSM       │        │
│      └──────────────────────────────────┘        │
└─────────────────────────────────────────────────┘
```

- **核**只管算：矩阵从端口整体进来，`start` 一拉就跑，`done` 举手，结果从端口整体出去
- **壳**负责和外面通信：把 CPU 一个 word 一个 word 写进来的数据攒成整个矩阵，把核的结果按地址一个一个读出去

本 Lab 只做核，端口长这样：

| 信号 | 位宽 | 含义 |
|---|---|---|
| `clk` | 1 | 时钟 |
| `rst_ni` | 1 | 异步**低**有效复位（`rst_ni == 0` 才复位） |
| `start` | 1 | 单拍脉冲，拉高一拍启动一次 GEMM |
| `done` | 1 | 算完置 1，**一直保持**到下一次 `start` |
| `act` | `[15:0][3:0]` | 矩阵 A，列优先 |
| `wgt` | `[15:0][3:0]` | 矩阵 B，列优先 |
| `out` | `[15:0][9:0]` | 矩阵 C，列优先，每个元素 10 bit |

**本 Lab 里，扮演壳的是 testbench**——`tb_npu_check.sv` 会直接驱动这 7 个端口。壳本身（以及后面挂到 SoC 总线上）是后续 Lab 的内容，到时候你会把这个核**原封不动**地例化进去。

### 6. ⏱️ 关键时序：start / done / 结果驻留

这是本 Lab 最容易踩的坑，单独拎出来讲。一次 GEMM 的标准时序：

```text
             T0    T1    T2    T3    T4    T5    T6    T7    T8
clk        __/‾\__/‾\__/‾\__/‾\__/‾\__/‾\__/‾\__/‾\__/‾\__
act / wgt  ==X============ 稳定，至少保持到 done ==============
start      ______/‾‾‾‾‾\____________________________________
state_q    IDLE  IDLE  RUN   RUN   RUN   RUN   DONE  DONE  DONE
k_cnt       0     0     0     1     2     3     0     0     0
pe_clr     ______/‾‾‾‾‾\____________________________________
pe_en      ____________/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\________________
acc         0     0     0    +k0   +k1   +k2   +k3 = C（驻留）
done       ______________________________________/‾‾‾‾‾‾‾‾‾‾‾‾ ← sticky
```

- `start` 在 T1 拉高一拍，DUT 在 T2 的上升沿采到它，进 RUN
- RUN 停 4 拍（T2~T5），`pe_en` 正好高 4 拍，`k_cnt` 走 0→1→2→3
- T6 进 DONE，`done` 拉高，**然后一直保持**，直到下一次 `start`
- `pe_clr` 在进 RUN **之前**那一拍拉高，把上一轮的 `acc` 清掉。图里它和 `start` 同拍，是因为参考写法把它写成组合的（`pe_clr = start && 不在 RUN`）。你也可以把 `start` 先寄存一拍再用，整张图往右平移一拍——**关键只有一条：`pe_clr` 必须在第一次累加之前生效，且 PE 里 `clr` 的优先级高于 `en`**

验收 tb 对这条时序查三件事，**每一件都是硬性规格**：

**(a) `done` 必须 sticky**。tb 看到 `done = 1` 之后会**再等 5 拍**，再检查一次 `done` 还是不是 1。如果你写成「进 DONE，下一拍无条件回 IDLE」，`done` 就只亮 1 拍，这里直接挂。

**(b) `start` 之后 `done` 必须先拉低**。连续跑两次 GEMM 时，第二次 `start` 之后 tb 要求先看到 `done = 0`，再看到 `done = 1`。如果上一轮的 `done` 一直挂着不掉，tb 一看 `done = 1` 就去读结果，读到的是上一轮的旧值——所以这种情况直接判错，不给你蒙混的机会。

**(c) 结果必须驻留在 PE 里**。`done` 拉高之后，tb 会**故意把 `act` 和 `wgt` 全部按位取反**，等 2 拍，然后才去读 `out`。

- `out` 直接接 PE 的累加寄存器 → 输入怎么变都不影响，读到正确值 ✅
- `out` 是从 `act/wgt` **组合算出来的**（AI 很爱这么写：`assign out[...] = act[...]*wgt[...] + ...`）→ 输入一翻，输出立刻跟着变，读到一堆错值 ❌
- DONE 期间 `pe_en` 没关、PE 还在继续累加 → 同样读到错值 ❌

> 🔍 这模拟的是真实场景：壳（或 CPU）完全可能在读走结果之前就开始装下一组数据。「结果驻留」不是风格问题，是 **output-stationary 这个数据流的定义**。不改 `act/wgt` 的 tb 是抓不到这个 bug 的，所以验收 tb 里那两句 `act <= ~act; wgt <= ~wgt;` 别去掉。

### 7. AI 辅助硬件设计

AI 写软件已经很强，写硬件还在快速进步。它的特点：

- ✅ 端口签名、`generate` 数组、状态机骨架，都能给出像模像样的一稿
- ✅ 解释一段 RTL、讲清时序、提示 testbench 怎么写，很好用
- ⚠️ 经常生成**「看着对、跑不通」**的 RTL：忘了复位、位宽错、把 `rst_ni` 当成高有效、把 16 个 PE 偷换成一个组合大乘法器
- ⚠️ 写出的 testbench 经常**覆盖不到边界**（比如从来不测「连续跑两次」）

所以本 Lab 的流程被设计成：**AI 出一稿 → 你读 → tb 证伪 → 波形定位 → 你告诉 AI 错在哪 → 迭代**。

本 Lab 想练的不是「你和 AI 聊了多少」，而是**你判断 AI 输出的能力**。

## 📋 实验规格

### 设计规格速览（**这张表可以整段复制给 AI 当 spec**）

| 项 | 规格 |
|---|---|
| 矩阵尺寸 | **4 × 4**（A、B、C 都是） |
| 元素数据类型 | **unsigned 4-bit**（范围 0~15） |
| 单次乘积范围 | 0 ~ 225 |
| 累加器位宽 | **10-bit**（4 次累加最大 4×225 = 900 < 1024） |
| 计算式 | `C[i][j] = Σ_{k=0..3} A[i][k] · B[k][j]` |
| 数据流类型 | **output-stationary**：每个 PE 锁定一个 C[i][j]，不做 systolic |
| PE 数量 | **16 个并行**（4×4，用 generate 展开） |
| PE 内部 | **1 级 MAC**：`acc <= acc + a*b`，无流水线 |
| 单次 GEMM 周期 | **4 cycle**（RUN 停留 4 拍，每拍喂一个 k） |
| 状态机 | **3 态**：IDLE → RUN → DONE，标准三段式；DONE 收到 start 直接回 RUN |
| 接口 | 核接口：`clk / rst_ni / start / done / act / wgt / out` |
| 矩阵端口 | packed 数组：`act`、`wgt` 为 `[15:0][3:0]`，`out` 为 `[15:0][9:0]` |
| 数据打包 | **列优先**：`act[j*4+i] = A[i][j]`，`wgt`、`out` 同理 |
| 复位 | `rst_ni` **异步、低有效** |
| 启动 | `start` 单拍脉冲；RUN 期间再来的 `start` 忽略 |
| 完成 | `done = 1` 表示算完；RUN 期间必须为 0 |
| DONE 的保持 | **必须 sticky**：`done` 置 1 后要一直保持，直到下次 `start`。**不能是 1 拍脉冲** |
| 输入稳定性 | `act/wgt` 从 `start` 到 `done` 之间保持不变（tb 保证，你不用管）。**DONE 期间输入可能是任意垃圾值**，靠 `pe_en = 0` 隔离 |
| 输出驻留 | `done` 之后 `act/wgt` **可以任意改动**，`out` 必须不变（结果驻留在 PE 里） |

> ⚠️ **「3 态 IDLE→RUN→DONE」这行别按字面理解成「DONE 之后回 IDLE」**。如果你写成「进 DONE，下一拍无条件回 IDLE」，那 `done` 就只亮 **1 拍**，验收 tb 的 sticky 检查必挂。
>
> 最简单的做法：**DONE 只有 `start` 能离开**（DONE → RUN），`done` 就等于「当前在 DONE 状态」。IDLE 只在复位之后用一次。
>
> 你也可以让状态机回 IDLE、另外用一个寄存器保持 `done`（`start` 来了才清）——两种写法都能通过验收。但要记住：**「状态机的状态」和「给外面看的 done」可以是两件事**，不要直接把 `state_q == S_DONE` 当成一个 1 拍脉冲用。

### 🔒 端口契约（**一行都不能改**）

打开 `rtl/simple_npu_core.sv`，顶部已经写好：

```systemverilog
module simple_npu_core (
    input  logic             clk,       // 时钟
    input  logic             rst_ni,    // 异步低有效复位
    input  logic             start,     // 单拍脉冲：启动一次 GEMM
    output logic             done,      // sticky：算完后保持 1，直到下次 start
    input  logic [15:0][3:0] act,       // 矩阵 A，act[j*4+i] = A[i][j]
    input  logic [15:0][3:0] wgt,       // 矩阵 B，wgt[j*4+i] = B[i][j]
    output logic [15:0][9:0] out        // 矩阵 C，out[j*4+i] = C[i][j]
);
```

⚠️ **模块名 / 7 个端口的名字、位宽、方向，一个字都不要改**——`tb_npu_check.sv` 是**按名字连接**的，改了就编译不过。特别注意 `act/wgt/out` 是 **packed** 数组（`[15:0][3:0]`，维度写在名字前面），不要写成 unpacked 的 `[3:0] act [0:15]`——Icarus 连接 packed 和 unpacked 会直接崩溃，报错信息还很难看懂（见 §B-4 的 Icarus 坑列表）。

你要做的是在 `// NEED TO BE DONE` 处填实现。

`rtl/simple_npu_pe.sv` 的端口是**建议**不是约束，只有你自己的 core 会用到它，随便改；甚至你可以不拆这个子模块，全塞进 core 里（能跑就行，但不推荐）。

### 📐 打包约定（**下标规则完全冻结**）

| 端口 | 槽位 k | 内容 | 位宽 |
|---|---|---|---|
| `act[k]` | `k = j*4 + i`，k ∈ [0,15] | 矩阵 A 元素 `A[i][j]` | 4 bit |
| `wgt[k]` | `k = j*4 + i`，k ∈ [0,15] | 矩阵 B 元素 `B[i][j]` | 4 bit |
| `out[k]` | `k = j*4 + i`，k ∈ [0,15] | 结果 C 元素 `C[i][j]` | 10 bit |

```text
A[i][j] = act[j*4 + i]
B[i][j] = wgt[j*4 + i]
C[i][j] = out[j*4 + i]
```

> 💡 反过来读：给定槽位号 `k`，`k[1:0]` 是 i、`k[3:2]` 是 j。写 PE 阵列时用行号 `gi`、列号 `gj`：这个 PE 的输出接 `out[gj*4 + gi]`，**不是** `out[gi*4 + gj]`。

### 🔄 完整的一次交互流程

外部（testbench）会严格按这个顺序操作，你的硬件要能配合：

```text
1. 复位：rst_ni 拉低几拍再释放
2. 装 A：act[j*4+i] <= A[i][j]，16 个槽位同一拍赋值
3. 装 B：wgt[j*4+i] <= B[i][j]
4. 拉一拍 start
5. 等 done == 1（要求先看到 done == 0，再看到 done == 1）
6. 再等 5 拍，确认 done 还是 1（sticky 检查）
7. 把 act / wgt 全部取反，等 2 拍（结果驻留检查）
8. 从 out 取 16 个结果：C[i][j] = out[j*4+i]，逐个对答案
9. 回到第 2 步，再跑一次（不重新复位！）
```

⚠️ **第 9 步是重点**：验收 tb 会**连续跑好几次 GEMM 而不复位**。这意味着新一轮 `start` 时，PE 里上一轮留下的累加值**必须被清掉**，否则第二次的结果会是两次的和。这是本 Lab 最经典的 bug（症状见 §Part C 的对照表）。

⚠️ **第 7 步是新增的坑**：`out` 必须接 PE 的累加寄存器，不能从 `act/wgt` 组合算出来（见 §背景知识 6）。

### 🧠 推荐的模块组织（不强制）

```text
simple_npu_core.sv
 ├─ IDLE/RUN/DONE 三段式 FSM + k 计数器
 ├─ pe_clr / pe_en / done 三个控制信号
 └─ simple_npu_pe.sv × 16（generate 展开，acc 直接接到 out 的对应槽位）
```

比原来想象的少？对——**因为没有总线**。核本身就这么点东西，难点全在「控制对不对、下标对不对、位宽够不够」。

> 💡 **命名建议**：讲义后面（§B-5 波形、Prompt 3）默认你用这套名字——状态 `state_q`（IDLE=0 / RUN=1 / DONE=2）、计数器 `k_cnt`、控制 `pe_clr` / `pe_en`、generate 块 `g_row` / `g_col`、PE 实例 `i_pe`。不强制，但换了名字的话，看波形时记得对应替换。

## 🛠️ 上机步骤

### 🔵 Part A · 跑通空骨架，看到「预期失败」（约 20 min）

在写任何逻辑之前，先确认**编译流程和验收 tb 是通的**。在工作目录下执行：

```bash
iverilog -g2012 -s tb_npu_check -o check.vvp rtl/simple_npu_pe.sv rtl/simple_npu_core.sv tb/tb_npu_check.sv
vvp check.vvp
```

| 参数 | 含义 |
|---|---|
| `-g2012` | 启用 SystemVerilog 2012 语法 |
| `-s tb_npu_check` | 指定顶层**模块名**（不是文件名） |
| `-o check.vvp` | 编译输出 |
| 后面三个 `.sv` | 两个设计文件 + 验收 tb |

你应该看到：

```text
VCD info: dumpfile npu_check.vcd opened for output.
[CHECK] reset released @ cycle 6
[CHECK] TEST1 GEMM1 FAIL: TIMEOUT waiting for done=1 (polled 200 times)
        -> 检查 FSM：start 有没有被捕获、是不是卡在 RUN、DONE->RUN 的路径有没有写。
[CHECK] TEST1 FAIL  (1 error(s))
...
--------------------------------------------------
[CHECK] FAIL: 6 error(s)
--------------------------------------------------
tb/tb_npu_check.sv:254: $finish called at 12180000 (1ps)
```

（最后那行 `$finish called at ...` 是 vvp 自己打的，不是错误。）

✋ **这就是预期结果**。骨架里只有端口没有逻辑，`done` 永远是 0，所以 tb 轮询 200 拍后报 TIMEOUT。

**这一步的意义**：证明你的 iverilog 环境、命令行、验收 tb 都正常，接下来只差你的实现。

顺便花 10 分钟**读一遍 `tb/tb_npu_check.sv`**，特别是 `run_gemm` 这个 task——它定义了「外部会怎么访问你」，是你实现时序时最准确的参照。

### 🟡 Part B · AI 协作实现（约 120 min）

#### B-1 · 启动 Claude Code

在工作目录下启动（沿用 Lab 0 的配置）：

```bash
claude
```

#### B-2 · 四个示范 prompt

下面 4 个 prompt 仅供参考，**鼓励你按自己的理解改写**。

**Prompt 1 · 先问整体设计（不要急着让它写代码）**

````text
我要用 Verilog/SystemVerilog 写一个 4x4 矩阵乘计算核 simple_npu_core，仿真工具是
Icarus Verilog。规格如下：

[把 §设计规格速览 整张表粘进来]
[把 §端口契约 和 §打包约定 也粘进来]

先不要写代码。请你回答：
- 你建议拆成几个文件？每个文件负责什么？为什么这么拆？
- 16 个 PE 在第 k 拍分别应该读到 act 和 wgt 的哪个下标？
  请把下标推导过程写出来（列优先打包 act[j*4+i] = A[i][j]）。
- 每个 PE 的输出应该接到 out 的哪个槽位？
- out 是直接接 PE 的累加寄存器，还是从 act/wgt 组合算出来？
  如果 done 之后 act/wgt 变了，out 会不会跟着变？规格允许吗？
- done 信号什么时候置 1、什么时候清 0？「状态机在 DONE 状态」和
  「done 输出为 1」是不是同一件事？
````

> 💡 **为什么先问设计再写代码**：AI 直接生成的代码你很难判断对错，但它的**设计推导过程**你能看懂、能挑错。特别是「16 个 PE 的下标推导」那一问，如果它推错了，后面代码一定错——在这一步抓出来，比调 100 行代码便宜得多。

**Prompt 2 · PE**

````text
请写 simple_npu_pe.sv，一个 1 级 MAC 的 PE：

端口：
  input  logic        clk, rst_ni, clr, en;
  input  logic [3:0]  a, b;
  output logic [9:0]  acc;

要求：
- always_ff @(posedge clk or negedge rst_ni)，异步低有效复位
- 优先级：!rst_ni -> acc<=0；clr -> acc<=0；en -> acc <= acc + a*b；否则保持
- 直接写 acc <= acc + (a*b)，并解释为什么这样不会被截断到 8 bit
  （提示：赋值语句的上下文宽度）
- 如果你想拆出中间信号（例如 wire prod = a*b），请说明它的位宽该给多少、为什么
- 每个分支加注释说明它对应什么场景
````

**Prompt 3 · 状态机**

````text
请写 simple_npu_core 里的 IDLE/RUN/DONE 三段式状态机部分：

输入（模块端口）：
  - start：外部给的单拍脉冲
  - clk, rst_ni

这段逻辑要产生的信号（除 done 外都是模块内部信号，不是端口，端口一个都不能加）：
  - state_q
  - k_cnt[1:0]：RUN 期间 0->1->2->3
  - pe_clr：新一轮 RUN 的第一拍累加之前拉高，清掉 PE 累加器
  - pe_en：RUN 期间持续拉高，正好 4 拍
  - done：算完置 1

约束：
- 三段式：状态寄存 / 次态组合 / 输出组合，分三个 always 块写
- 外部会**连续跑多次 GEMM 而不复位**，所以从 DONE 回到 RUN 的路径必须
  也能把累加器清零，请特别说明你是怎么保证的
- **done 必须是 sticky 的**：置 1 之后要一直保持，直到下一次 start 才清。
  请说明你打算怎么保持它，以及这和「状态机什么时候离开 DONE 状态」
  是不是同一件事
- RUN 期间如果又来了一个 start，忽略它

写完请画一个时序表格：从 start 拉高那一拍开始，逐拍列出
state_q / k_cnt / pe_clr / pe_en / done 的值，一直列到下一轮 start。
````

> 💡 最后那句「画时序表格」很重要。**表格比代码好检查**——你可以直接数一数 `pe_en` 是不是正好 4 拍、`pe_clr` 是不是在第一次累加之前，然后和 §背景知识 6 的那张图对一对。

**Prompt 4 · 调试求助**

跑挂了之后这样问，把**现象**说清楚，不要只说「不对」：

````text
我跑 tb_npu_check.sv，TEST1 全 PASS，但 TEST2 和 TEST3 全 FAIL，报错是：

[CHECK] TEST2 GEMM1 MISMATCH: FAIL_I=0 FAIL_J=0 HW_VAL=251 REF_VAL=1
[CHECK] TEST2 GEMM2 MISMATCH: FAIL_I=0 FAIL_J=0 HW_VAL=395 REF_VAL=144

读出来的数都比预期大很多，而且看不出简单的倍数关系。
已知这个 tb 从头到尾不复位，会连续跑多次 GEMM。
我的 FSM 代码是：[贴代码]
我的 PE 代码是：[贴代码]

请给我 3 个最可能的怀疑点，按可能性排序，并说明我应该在波形里看哪个信号来确认。
````

#### B-3 · 逐行读懂 AI 写的代码

⚠️ **不要不读就保存**。本 Lab 常见的 AI 翻车点，逐条对照检查：

| 检查项 | 怎么看 |
|---|---|
| `rst_ni` 被当成高有效用了 | 找 `if (rst_ni)` ——应该是 `if (!rst_ni)` |
| 复位写成同步的 | 敏感列表应该是 `@(posedge clk or negedge rst_ni)` |
| 端口位宽 / 维度被改了 | `act`、`wgt` 必须是 `[15:0][3:0]`，`out` 必须是 `[15:0][9:0]`，维度在名字**前面** |
| `acc` 只给了 8 bit | 必须 ≥ 10 bit，否则全 15 的用例会溢出 |
| **`out` 是从 `act/wgt` 组合算出来的** | 找 `assign out` ——`out` 的每个槽位应该直接接某个 PE 的 `acc`，中间不该有乘法 |
| **只有一个乘法器 + 4 层 for 循环** | AI 有时会把 16 个 PE 偷换成「一个大 always 块算完整个矩阵」。要有 16 个 `simple_npu_pe` 实例 |
| PE 的 a / b 下标推导错 | `a = act[k*4+i]`，`b = wgt[j*4+k]`，别弄反 |
| PE 接 `out` 时 i / j 弄反 | 行 `gi` 列 `gj` 的 PE 接 `out[gj*4+gi]`，不是 `out[gi*4+gj]` |
| `pe_clr` 只覆盖了 IDLE→RUN | DONE→RUN 那条路径也要清零 |
| `pe_en` 在 DONE 期间没关 | 应该只在 RUN 状态为 1，否则结果一直在变 |
| **`done` 只亮 1 拍** | `done` 必须 sticky，不能直接把「进了 DONE 状态」当脉冲用 |

**看不懂就追问**，让 AI 解释到你懂为止。这是本 Lab 的核心训练内容。

#### B-4 · 写自己的 testbench 自测

**先用自己的 tb 跑通，再去碰验收 tb。**自己的 tb 更小、报错更直接，调试效率高得多。

新建 `tb/tb_simple_npu.sv`，让 AI 帮你写，但你要确认前 6 步都覆盖到：

```systemverilog
// 1. 时钟 + 复位
// 2. 把 A/B 按列优先打包到 act/wgt：act[j*4+i] <= A[i][j]
// 3. 拉一拍 start
// 4. 等 done 拉高
// 5. 从 out 取结果：C[i][j] = out[j*4+i]，和手算预期逐个对比
// 6. 至少跑 2 组数据，中间不复位（推荐第一组用单位阵：A = I 时预期 C = B，最好验证）
//    ⚠️ B 要用「非对称」矩阵（B[i][j] != B[j][i]），否则 i/j 弄反了也看不出来
// 7.（可选）验收 tb 没测但规格里有的：RUN 期间再塞一个 start，结果应该不受影响
```

跑：

```bash
iverilog -g2012 -s tb_simple_npu -o npu_tb.vvp rtl/simple_npu_pe.sv rtl/simple_npu_core.sv tb/tb_simple_npu.sv
vvp npu_tb.vvp
```

> ⚠️ **Icarus Verilog 的几个坑**，AI 写代码时经常踩（Icarus 12.0 实测）：
> - 不支持给 task 传 unpacked 二维数组 → 把矩阵放模块级变量共享，task 只传标量
> - 不支持用 `break` 跳出循环 → 用 `while` 加标志位，或者 `disable`
> - 端口写成 unpacked 数组（`[3:0] act [0:15]`）→ iverilog 直接崩溃，报 `assert: ... failed assertion ...`。照骨架写 packed `[15:0][3:0]`
> - `unique case` / `priority case` → 提示 `sorry: Case unique/unique0 qualities are ignored.`。**这只是警告**，逻辑和普通 `case` 一样，`.vvp` 照样生成。想让输出干净就把 `unique` 去掉
> - 总之：别一看到 `sorry:` 就以为代码坏了，**看它到底有没有生成 `.vvp`**——生成了就是能跑

#### B-5 · 看波形

```bash
gtkwave npu_check.vcd
```

（macOS 或没装 GTKWave 的同学，用 [Surfer 网页版](https://app.surfer-project.org/) 拖入 `.vcd` 即可。）

重点看这几个信号：

| 信号路径 | 看什么 |
|---|---|
| `start / done` | `done` 是不是在 `start` 之后先掉再起、起了之后一直保持 |
| `dut.state_q` | IDLE(0) → RUN(1) → DONE(2) 的转换时刻 |
| `dut.k_cnt` | RUN 期间是不是老老实实 0→1→2→3 |
| `dut.pe_en` | 是不是**正好 4 拍**，不多不少 |
| `dut.pe_clr` | 是不是在第一次累加**之前**拉高 |
| `dut.g_row[0].g_col[0].i_pe.acc` | `C[0][0]` 一拍一拍累加上去的过程；`done` 之后 tb 翻转 `act/wgt` 时它**不该动** |
| `act / wgt / out` | 整根向量。`act` 用十六进制显示，从右往左每个 hex 位就是 `act[0]、act[1]…` |

把 `acc` 的显示格式设成**无符号十进制**（GTKWave 里右键 → `Data Format → Decimal`），你应该能看到它在 4 拍里逐步累加到最终值。

> 💡 `act / wgt / out` 是 packed 数组，在波形里是一根 64 / 64 / 160 bit 的向量，**能看到**。`act` 和 `wgt` 每个元素正好 4 bit = 1 个 hex 位，直接读；`out` 每个元素 10 bit 不对齐，想看单个结果就看对应 PE 的 `acc`。如果你自己在内部用了 unpacked 数组（`logic [3:0] foo [0:15]`），那种 Icarus 默认不 dump 进 VCD，看不到是正常的。

#### B-6 · 回头看一眼你和 AI 的对话（可选）

做完之后，花 5 分钟翻一下对话记录，找出 **1~2 个 AI 给错了、你自己判断出来的地方**——比如它推错了下标、把 `done` 写成了脉冲、把 16 个 PE 偷换成了组合大乘法器。

这不用交给任何人。但它是本 Lab 真正想训练的东西：**能看出 AI 错在哪、能追问澄清、能在 AI 的输出和你的硬件知识冲突时坚持自己**。后面的 SoC Lab 规模更大，AI 翻车的机会更多，这个习惯到时候会救你。

### 🔴 Part C · 跑验收 testbench（约 30 min）

自测通过后，跑助教的验收 tb：

```bash
iverilog -g2012 -s tb_npu_check -o check.vvp rtl/simple_npu_pe.sv rtl/simple_npu_core.sv tb/tb_npu_check.sv
vvp check.vvp
```

三组测试逐级加难：

| 测试 | 内容 | 专门考什么 |
|---|---|---|
| **TEST1** | 单次 GEMM，dense 混合矩阵 | 基本功能、列优先打包对不对、结果驻留 |
| **TEST2** | 连续 2 次 GEMM（单位阵×pattern，然后 dense×dense） | DONE→ 下一轮的重启路径、`done` 有没有随 `start` 清掉 |
| **TEST3** | 连续 3 次（全 15×全 15 **跑两遍**，再单位阵×单位阵） | **累加器清零** + 10 bit 最大值 900 边界 |

> 💡 TEST3 的设计很阴险：同样的全 15 数据**连跑两次**，正确结果两次都是 900。如果你的累加器没清零，第二次会读出 1800 截断后的值，一抓一个准。

**全部通过的标志**：

```text
[CHECK] reset released @ cycle 6
[CHECK] TEST1 PASS  (14 cycles)
[CHECK] TEST2 PASS  (28 cycles)
[CHECK] TEST3 PASS  (42 cycles)
--------------------------------------------------
[CHECK] ALL PASS  (total 90 cycles)
--------------------------------------------------
```

对照一下你的 cycle 数：

| 测试 | 参考值 | 说明 |
|---|---|---|
| TEST1 | 14 | 每次 GEMM 和参考值差 1~2 拍是正常的（比如 `start` 先寄存一拍再用） |
| TEST2 | 28 | **差出十几拍**就要怀疑 RUN 状态多停留了 |
| TEST3 | 42 | |

**失败了怎么办**：**先看报错文字和「哪几组挂了」**，这两条信息基本能直接定位（下表每一行都是在参考实现上人工注入 bug 实测出来的）：

| 现象 | 最可能的原因 |
|---|---|
| 全部 `TIMEOUT waiting for done=1` | `done` 从来没起来：`start` 没被采到 / 卡在 RUN（看 `k_cnt` 有没有递增）/ 骨架里 `assign done = 1'b0` 没删 |
| `done 只亮了一下就掉了（不是 sticky）` | 写了 `S_DONE: state_nxt = S_IDLE;`（无条件回 IDLE）。DONE 只能由 `start` 离开 |
| TEST1 过，TEST2/3 报 `done 在 start 之后从来没有拉低过` | 上一轮 `done` 没被新的 `start` 清掉：FSM 缺 DONE→RUN 路径，或 `done` 寄存器没在 `start` 时清零 |
| **TEST1 过，TEST2/3 MISMATCH，HW_VAL 是一堆没规律的大数** | **累加器没清零**（本 Lab 头号 bug）。`pe_clr` 漏了 DONE→RUN 那条路径，或 PE 里没接 `clr`。别指望读数是正确值的 2 倍——残留是跨 TEST 累积的还会回绕 |
| TEST1/2 MISMATCH，TEST3 过 | **i/j 弄反了（转置）**。TEST3 全是对称阵所以看不出来。查 `a = act[k*4+i]`、`b = wgt[j*4+k]`、`.acc(out[gj*4+gi])` |
| 只有 TEST3 MISMATCH | **累加器位宽不够**。TEST1/2 的 C 最大 336/282，TEST3 是 900——你多半给了 9 bit。8 bit 的话三组全挂。编译时 `warning: Port 7 (acc) ... expects 9 bits, got 10` 就是在提醒你 |
| 三组全 MISMATCH，数值没规律，但下标确定没反 | **结果没有驻留**：`out` 是从 `act/wgt` 组合算的（tb 读之前把它们取反了），或 `pe_en` 在 DONE 期间没关。波形上看 `done=1` 之后 `i_pe.acc` 是不是还在动 |

然后：

1. `gtkwave npu_check.vcd` 看波形，按 §B-5 的信号清单查
2. 把**现象**（不是「不对」）描述给 AI，让它给 3 个怀疑点

看到 `ALL PASS`，本 Lab 就完成了。**不需要提交任何东西**——把 `rtl/` 里的两个文件留好，SoC Lab 会直接用到。

## 🤔 思考题（不用交，建议想一想）

这些问题没有标准答案，也没人检查。但它们每一个都指向后面 SoC Lab 会真正碰到的问题，现在想清楚，到时候少踩坑。

1. **本 Lab 的 `done` 是「保持到下次 `start`」。** 将来把核挂到总线上之后，CPU 是通过读一个 STATUS 寄存器来看 `done` 的，那时还有另一种常见做法：「STATUS 被读一次就把 `done` 清零」。两种各有什么优劣？（提示：想想如果有两个程序同时轮询同一个 NPU 会发生什么；再想想「清零」这件事应该由核做还是由壳做）

2. **如果 `out` 不是接 PE 的累加寄存器，而是从 `act/wgt` 用 64 个乘法器组合直接算出来（AI 很可能第一稿就这么写），功能上看起来也对。** 验收 tb 是怎么抓它的？这种写法和「16 个 PE 跑 4 拍」相比，在面积、功耗、以及「结果什么时候有效」上有什么区别？

3. **如果外部在 NPU 还在 RUN 状态时又拉了一次 `start`，或者改了 `act`，会发生什么？** 你的设计对此有保护吗？需要保护吗？

4. **本 Lab 的 4 个 cycle 里，16 个乘法器每拍都在满负荷工作。如果矩阵变成 8×8 但你的 PE 阵列还是只有 16 个，你会怎么改？** 需要几个 cycle？（不用写代码，说清思路即可）

5. **AI 给你写的 RTL 里，哪一段你最不放心？为什么？你在 testbench 里专门测了它吗，怎么测的？**

6. **（为 SoC Lab 预热）如果要把这个核挂到 CPU 的总线上，让 C 程序通过读写地址来用它，「壳」里至少需要哪些东西？** 列个清单就行：CPU 一次只能写一个 32 bit word，`act` 却要 64 bit 一起给；CPU 怎么发 `start`、怎么看 `done`、怎么把 `out` 一个一个读走。

