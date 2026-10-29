# Lab 5：Signoff & Tapeout

!!! info "开始之前"
    - **主题**：版图完成与签核：CDL 转换、Virtuoso、Seal Ring、Dummy、DRC、LVS/ERC、天线与 Wire Bond PAD 检查、归档
    - **前置**：[Lab 4](lab-4.md)（完成步骤 34.6 的 Innovus 交付清单）
    - **环境**：xlab 服务器（Virtuoso IC618、Calibre 2022.3）
    - **课时**：2 次实验课
    - **完成标准**：按第 14 节的完成标准表逐项核对，并生成第 15 节的交付归档。**本实验不需要提交任何材料**

本文承接 [Lab 4：逻辑综合 & 物理实现](lab-4.md) 的 **34.6：确认 Innovus 交付清单**，从仓库原始 `layout/` 目录开始，完成 CDL 转换、Virtuoso 导入、Seal Ring 适配、DRC 修正、Dummy 生成、LVS/ERC、最终天线及 Wire Bond PAD 检查和文件归档。

阅读顺序：输入准备 → CDL → Virtuoso 与 Seal Ring → POC → Dummy 前 DRC → Dummy 与合并 → 最终 DRC → LVS/ERC → Antenna → Wire Bond PAD → 归档。

适用基线：TSMC180 1P6M，顶层 `my_soc_top`，Die 为 1800.4×1800.4 µm，SRAM 为 `RA1SHD_2048x32M8`，六信号 PAD、28 供电/POC PAD、34 个 `PAD60LU_OBV`。坐标单位均为µm。使用其他芯片尺寸、IO 库或金属厚度时，先重新确定相应参数。

所有操作由操作者逐项完成。本文写出的新文件均在相应步骤中创建，不需要拿到他人的已修改规则、接口文件、OA 库或 runset。示例环境为 Virtuoso IC618、Calibre 2022.3。

## 1. 确认输入、路径和原始目录

### 1.1 输入文件

确认已完成《Lab 4：逻辑综合 & 物理实现》步骤 34.6，在 `pnr/my_soc_top/outputs/manual_final/` 至少有：

| 文件 | 本阶段用途 |
| --- | --- |
| `my_soc_top.gds.gz` | 合并标准单元、SRAM、IO、Bond PAD 的原始完整 GDS |
| `my_soc_top_flat_lvs.v` | 含真实 PG 连接的 LVS 物理网表 |
| `my_soc_top.def` | 核对实例位置、方向与端口 |
| 层次 Verilog、SDC、SPEF、SDF、LEF | 最终交付及后续仿真使用 |

保留 Innovus 检查点和报告。本流程已在 Innovus 中完成电源端口和 IO 连接，发现缺项应回到《Lab 4》对应步骤修正并重新导出。

### 1.2 设置终端路径

打开 **Linux bash 终端**，先进入自己的项目根目录，再执行：

```bash
cd /你的实际路径/soc_cv32e40p_template_pnr
export SOC_ROOT="$(pwd)"
export PDK_ROOT=/home/almalinux/tsmc180
export CALIBRE_BIN=/opt/eda/Mentor/aok_cal_2022.3_17.8/bin
printf '%s\n' "$SOC_ROOT"
```

第一行须替换成自己的实际路径；工艺库和工具安装位置也以所在服务器为准。后续终端命令均使用上述变量；新开终端时要重新设置。

**GUI 表格中的 `<项目根目录>` 要替换为打印出的实际绝对路径，不能原样输入，也不要在 GUI 文件框里写 `$SOC_ROOT`。** 相对文件名按表中的 Run Directory 解释。

### 1.3 原始 layout 目录中已有的文件

```text
layout/
  Makefile
  rule/calibre.drc
  rule/calibre.lvs
  rule/ant.drc
  runset/                       自带的参考配置
  dummy/scripts/
    Dummy_Metal_Calibre_0.18um.215a
    Dummy_OD_PO_Calibre_0.18um.210a
    run_dummy.csh
  workspace/
    cds.lib、lib.defs、.cdsinit、display.drf、techfile等
```

其中自带的 runset 含有原作者的路径、其他顶层名、SRAM Box、虚连及单项属性覆盖，**本指南不加载这些自带 runset**。从空白 Calibre 配置开始，按下文填写。

在 Linux 终端创建工作目录，复制项目规则：

```bash
cd "$SOC_ROOT"
mkdir -p layout/workspace/drc_preDummy
mkdir -p layout/workspace/drc_postDummy
mkdir -p layout/workspace/lvs_postDummy
mkdir -p layout/workspace/ant_postDummy
mkdir -p layout/workspace/wb_postDummy
mkdir -p layout/dummy/manual_final
cp layout/rule/calibre.drc layout/rule/my_soc_top.drc
cp layout/rule/calibre.lvs layout/rule/my_soc_top_postDummy.lvs
cp layout/rule/ant.drc layout/rule/my_soc_top_postDummy.ant.drc
cp layout/dummy/scripts/Dummy_Metal_Calibre_0.18um.215a layout/dummy/manual_final/
cp layout/dummy/scripts/Dummy_OD_PO_Calibre_0.18um.210a layout/dummy/manual_final/
```

这些 `cp` 只在首次准备时执行一次，后续重跑直接使用已配置的副本，避免覆盖自己的修改。原始 `rule/calibre.*`、`rule/ant.drc` 和 `dummy/scripts/` 保留不改。

规则文件用文本编辑器打开，例如 `gedit 文件路径 &`，或用 `vim`。配置修改后保存。命令执行环境分别为 Linux 终端、Virtuoso CIW；不要在 Innovus 提示符中执行本指南的 bash 或 SKILL 表达式。

## 2. 从 Verilog 转换 CDL

### 2.1 自己建立 SRAM 接口声明

在 Linux 终端执行，创建只含接口的 Verilog 文件：

```bash
cd "$SOC_ROOT/pnr/my_soc_top/outputs/manual_final"
cat > RA1SHD_2048x32M8_lvs_ports.v <<'EOF'
// SRAM interface for V2LVS; no functional implementation.
module RA1SHD_2048x32M8 (Q, CLK, CEN, WEN, A, D, OEN, VDD, VSS);
output [31:0] Q;
input CLK, CEN, OEN;
input [3:0] WEN;
input [10:0] A;
input [31:0] D;
inout VDD, VSS;
endmodule
EOF
```

这是转换器读取的端口声明，SRAM 电路仍由原始 CDL 提供，后续 LVS 完整比较 SRAM。不要把此文件当成 SRAM 功能模型，也不要通过 SRAM Box 跳过比较。

### 2.2 运行 V2LVS

仍在上述 Linux 终端目录执行：

```bash
"$CALIBRE_BIN/v2lvs" \
  -v my_soc_top_flat_lvs.v \
  -lsr "$PDK_ROOT/tcb018gbwp7t_290a/TSMCHOME/digital/Back_End/spice/tcb018gbwp7t_270a/tcb018gbwp7t_270a.spi" \
  -lsr "$PDK_ROOT/tpd018nv_280a/TSMCHOME/digital/Back_End/spice/tpd018nv_270a/tpd018nv_1_2.spi" \
  -l RA1SHD_2048x32M8_lvs_ports.v \
  -s "$PDK_ROOT/tcb018gbwp7t_290a/TSMCHOME/digital/Back_End/spice/tcb018gbwp7t_270a/tcb018gbwp7t_270a.spi" \
  -s "$PDK_ROOT/tpd018nv_280a/TSMCHOME/digital/Back_End/spice/tpd018nv_270a/tpd018nv_1_2.spi" \
  -s "$SOC_ROOT/src/macro/RA1SHD_2048x32M8/RA1SHD_2048x32M8.cdl" \
  -s0 VSS -s1 VDD -o my_soc_top.cdl > v2lvs.log 2>&1
echo $?
```

`echo $?` 紧跟转换命令，预期 0。打开 `v2lvs.log`，确认转换完成且没有 ERROR，再检查 CDL 非空、三个 `.INCLUDE` 路径存在。

CDL 顶层应为 `my_soc_top`，保留六信号和四个 PG 接口；四供电网独立，Decap 和 SRAM 保留。SRAM 通过命名引脚映射 Q/D 各 32 位、WEN4 位、A11 位，不凭 CDL 总线列表顺序猜测连接。

若报 license 不足，加载课程指定的 Calibre 许可环境后重试，不改网表来绕过转换。Virtuoso 导入和 Seal Ring 编辑可先做，LVS 前必须有有效 CDL。

## 3. 准备 Virtuoso 库并导入原始 GDS

### 3.1 整理项目 cds.lib

在文本编辑器中打开自己的 `layout/workspace/cds.lib`。保留可用的 `basic`、`analogLib` 和 `tsmc18` 定义；注释掉指向他人不存在目录的活动 `DEFINE`。示例技术库定义为：

```text
DEFINE tsmc18 /home/almalinux/tsmc180/t018cmsp007k3_1_3a/tsmc18mm_1.8v_3.3v_T-018-CM-SP-007-K3_IC61_bsim4.5_pdk_v1.3a_20170119/PDK/tsmc18
DEFINE to_iohelpers /home/almalinux/tsmc180/io/T18_1P6M_CUP_IO
```

原文件中若有同一路径的 `DEFINE T18_1P6M_CUP_IO ...`，将其注释掉，只保留 `to_iohelpers`。源 Seal Ring 的内部实例引用这个库名；不要给同一目录同时注册两个名字。这里只改项目映射，不改工艺目录中的文件，也不删除库目录。

确认 `layout/workspace/.cdsinit` 中的 Calibre 加载路径实际存在。本仓库参考行：

```lisp
load("/opt/eda/Mentor/aok_cal_2022.3_17.8/lib/calibre.OA.skl")
```

### 3.2 从正确目录启动

在 Linux 终端：

```bash
cd "$SOC_ROOT/layout/workspace"
virtuoso &
```

从 `workspace` 启动，让工具读取这里的库映射和初始化文件。若已经启动后才修改库映射，可在 CIW 执行 `ddUpdateLibList()`；旧名称缓存导致 master 仍无法解析时，保存自己的设计、正常退出，再从此目录重启。

### 3.3 创建项目库并导入

在 CIW 选择 **File → New → Library**，创建 `my_soc_top_manual`，选择 **Attach to an existing technology library → tsmc18**。

然后 **File → Import → Stream**：

| 字段 | 填写 |
| --- | --- |
| Library | my_soc_top_manual |
| Stream File | `<项目根目录>/pnr/my_soc_top/outputs/manual_final/my_soc_top.gds.gz` |
| Top Cell | my_soc_top |
| Attach Technology Library | tsmc18 |

文件选择器看不到 `.gz` 时改为 All files。点击 Translate，检查导入日志无 Error；在 Library Manager 打开 `my_soc_top_manual/my_soc_top/layout`。

`XSTRM-284` 提示源 GDS 与技术库 DBU 不同，应核对是否发生坐标舍入；不要直接改共享技术库精度。未定义 LPP 或未映射图形警告要核对实际层映射，不以导入结束作为完整性证明。

### 3.4 每次打开版图都检查显示设置

在版图窗口 **Options → Display**：

1. Resolution 设为 **Very High**。
2. 勾选 **Highest Resolution for P&R Objects**。
3. Display Levels：Start=`0`、Stop=`31`。
4. X/Y Snap Spacing 均为 `0.005` µm。
5. Apply、OK；Layers 点击 AV，Objects 确认 Instances、Mosaics 及 Shapes 可见。
6. 鼠标回画布，按 **Shift+F** 展开显示、**F**适配全图。

切换到 Seal Ring、副本或子单元时同样检查。细密环结构显示为黑屏时，先查显示分辨率、层可见性及 master 是否解析。

## 4. 复制并适配 Seal Ring

### 4.1 复制源环

在 Library Manager 只读打开：

```text
to_iohelpers / tsmc_c018_sealring_1p6m / layout
```

确认能看见完整环。选中该 layout，**Edit → Copy**：目标 Library=`my_soc_top_manual`，Cell=`sealring_my_soc_top`，View=`layout`，**Copy Hierarchy 关闭**。以编辑模式打开新副本；后续只改此副本，edge/corner 仍引用源库。

在副本为当前版图窗口时，CIW 输入只读查询：

```lisp
setof(i geGetEditCellView()~>instances i~>cellName == "tsmc_c018_sealring_corner_1p6m")~>xy
setof(i geGetEditCellView()~>instances i~>cellName == "tsmc_c018_sealring_corner_1p6m")~>orient
```

本源环的原尺寸为：prBoundary1860×1990，外框 `(-15,-15)–(1875,2005)`，edge 沿边长度 1。四角及本项目目标值如下：

| 角 | 原 Origin | 目标 Origin | 保持 Orientation |
| --- | --- | --- | --- |
| 左下 | (-15,-15) | (-15,-15) | R0 |
| 右下 | (1875,-15) | (1816,-15) | R90 |
| 左上 | (-15,2005) | (-15,1816) | MX |
| 右上 | (1875,2005) | (1816,1816) | MYR90 |

只有实测源尺寸符合时才使用下述位移。保持 1 µm 节距，将参考边界适配为 1801×1801，横向缩短 59、纵向缩短 189；不缩放工艺图形。

### 4.2 调整四角及直边

先保存副本。逐个选中 corner，按 **Q**，Origin 改成上表目标值，保持原方向。左下不移动。

随后按以下顺序处理直边，每次操作前按 Esc 清除旧选择：

1. **删上下边右端多余 edge。** 只选 `tsmc_c018_seal_ring_edge_1p6m`，横边方向 R0/MX，Origin X 在 `[1806,1865)` 的 59 段/边，共 118 段。避开 corner 和边界。
2. **下移上横边。** 只选剩余的上边整排 edge，按 M、F3 打开 Move。取消勾选 **Treat as absolute point**，Delta X=`0`、Delta Y=`-189`，Apply 一次后 Esc。上边 edge Origin Y 应由 1990 变为 1801。
3. **删左右边上端多余 edge。** 每边删除 189 个 1 µm 段，保留到上端接合边界 1806。只选竖边 edge，不能删进已移动的上横边或 corner。可用 **Tools → Find/Replace**，筛选 Cell Name 为 edge、范围为 current cellView、区域模式 Enclosed Figures；区域覆盖 `(-20,1806)–(1880,1995)`，符合此源环时预期 378 段。图形选择数量不符先查清，勿直接 Delete。
4. **左移右竖边。** 只选剩余右边整排 edge，M、F3，取消 Treat as absolute point，Delta X=`-59`、Delta Y=`0`，Apply 一次后 Esc。左边、下边及已调整 corner 不移动。

按 Delete 执行删除；Find 窗口不要点 Replace All。无需在画布底部坐标栏输入位置；可手动框选并用 Q 核对对象，或使用下面的精确横边选择。

若横边难以框选，在 CIW 逐条输入，然后回画布按 Delete：

```lisp
geDeselectAllFig()
geSelectFigs(setof(i geGetEditCellView()~>instances i~>cellName == "tsmc_c018_seal_ring_edge_1p6m" && (i~>orient == "R0" || i~>orient == "MX") && xCoord(i~>xy) >= 1806 && xCoord(i~>xy) < 1865))
length(geGetSelSet())
```

上述源尺寸下应选中 118 个。1806 是沿边方向的接缝边界，不能理解为“所有 X 或 Y 大于 1806 的对象都删除”。

### 4.3 调整参考边界并插入顶层

选中副本的 prBoundary，按 Q，将矩形设为 `(0,0)–(1801,1801)`；若显示顶点列表，改成对应四个矩形顶点。仅改参考边界。

放大检查四角和接缝，确认无缺口、重叠、长尾。副本外框应为 `(-15,-15)–(1816,1816)`，上/右边径向边界约 1805.82–1816。保存副本。

打开 `my_soc_top_manual/my_soc_top/layout`，按 **I**，选择自己的 `sealring_my_soc_top/layout`，Orientation=`R0`。**先在画布任意位置放置一次，Esc 退出；选中新实例，按 Q，把 Origin 改为 `(-0.3,-0.3)`。**

最终环外框应为 `(-15.3,-15.3)–(1815.7,1815.7)`，本例参考边界相对 1800.4 die 居中。原环已含 15 µm 外扩，不能再把实例原点设成 `(-15,-15)`。保存芯片顶层。

!!! note "Core、Die 与 Seal Ring 外框：本例尺寸与你的上限"
    上面出现的坐标都是**本例**的数值，不要当成自己设计的固定尺寸。
    这里把几个"框"分开，方便你核对自己的版图：

    | 项目 | 本例尺寸 |
    | --- | --- |
    | Core（标准单元可用区域） | 约 1500.24 × 1500.24 µm |
    | 原始 Die（不含 Seal Ring） | 1800.4 × 1800.4 µm，即 1.8004 × 1.8004 mm，约 3.24144 mm² |
    | Seal Ring 的参考边界 `prBoundary` | 1801 × 1801 µm |
    | 加 Seal Ring 后的最终外框 | 1831 × 1831 µm，即 1.831 × 1.831 mm，约 3.35256 mm² |

    本例 Seal Ring 的尺寸是这样来的：

    - 环本身宽约 **15 µm**；实例原点放在 `(-0.3,-0.3)`，最终外框为 `(-15.3,-15.3)–(1815.7,1815.7)`
    - 因此相对原始 Die，**每一边向外增加 15.3 µm**，每维增加 30.6 µm：`1800.4 + 15.3 × 2 = 1831 µm`

    **你自己做设计时，请遵守以下约束：**

    - 原始 Die 尺寸**不超过 1950 × 1950 µm**；加 Seal Ring 后的最终外框**不超过 1980 × 1980 µm**
    - Seal Ring 的单元结构、环宽和工艺图形**不要改动**：只按本节的复制、按边裁剪和平移来适配你的 Die，不要自行缩放或重新设计环

## 5. 预先补 POC 标签

本 IO 源 SPI 含 `.GLOBAL POC ...`，LVS 源侧会出现 POC 端口；需要在版图较高层放置可被读取的 POC 文字。直接在 **Dummy 前的芯片顶层**补好，让后续合并自动带入。

在 `my_soc_top_manual/my_soc_top/layout` 中：

1. 找到底边 `i_pad_S_vddio`，master=`PVDD2CDG`。本布局 Origin=`(1100,0)`、R0。
2. 显示 **METAL3 drawing** 与 **METAL3 pin**。该 PAD 本地 POC 轨为 `(0,111.595)–(60,113.715)`；顶层对应 `(1100,111.595)–(1160,113.715)`。
3. 按 **L** 创建 Label：Mode=`Manual`，文字=`POC`，Label Layer/Purpose=`Select layer → METAL3 pin`，**取消 Attach**。
4. 先放置一次，Esc 退出，选中文字按 Q，将锚点位置改为 `(1130,113.215)`，保证在上述 M3 轨内部。
5. 保存顶层，不进入工艺库 PAD master 修改。

这里**不要求已有 METAL3 pin 矩形**。物理导体是 METAL3 drawing，文字用 METAL3 pin 层；不另画金属、不把 POC 接到 VDD/VDDPST。映射中 M3 drawing 为 28/0，METAL3 pin 文字为 42/0；LVS 使用文字锚点附着到 M3 导体。

若同学的 PAD 位置不同，先按 Q 读取实际 Origin 和方向，按变换后的 POC 轨放置，不能套用本坐标。文字可以伸出窄轨，锚点必须落在正确导体内。

## 6. 导出 Dummy 前 GDS

保存芯片顶层。在 CIW **File → Export → Stream**：

| 字段 | 填写 |
| --- | --- |
| Run Directory | `<项目根目录>/layout/workspace` |
| Library | my_soc_top_manual |
| Top Cell | my_soc_top |
| View | layout |
| Stream File | my_soc_top_preDummy.gds |
| Technology Library | tsmc18 |
| Layer Map | 下列 tsmc18.layermap 的实际绝对路径 |

参考 Layer Map：

```text
/home/almalinux/tsmc180/t018cmsp007k3_1_3a/tsmc18mm_1.8v_3.3v_T-018-CM-SP-007-K3_IC61_bsim4.5_pdk_v1.3a_20170119/PDK/tsmc18/tsmc18.layermap
```

导出完整层次，保留 Seal Ring 和原生库引用，不只导出顶层 shape，不额外 flatten。Translate 完成后看日志，确认无 Error 且输出文件更新。Technology Library 只是指定工艺，不需要把整个 tsmc18 库作为版图一起导出。

Run Directory 为上述 workspace 时，Stream File 只填文件名即可。若报 XSTRM-15，检查日志中是否把 `home/...` 误写为缺少开头 `/` 的相对路径，导致路径重复拼接，先修正路径。

## 7. Dummy 前 DRC 和 CO.R.4 修正

### 7.1 检查 DRC 项目副本

打开第 1 节复制的 `layout/rule/my_soc_top.drc`，确认与本基线一致：

| 开关 | 本基线 |
| --- | --- |
| FULL_CHIP、WITH_SEALRING、SmallSealringCorner | 开启 |
| 1.8V、3.3V | 开启 |
| THICK_20K、THICK_40K | 关闭 |

保留原有其他检查和数值，不全局关闭 ESD/LUP 等规则。这里 `#DEFINE 名称` 表示开启，行首 `//` 表示注释。输入 GDS、Top 和输出路径通过下一步 GUI 设置；规则中原有 GDSFILENAME/TOPCELLNAME 占位由 Calibre 配置覆盖。

### 7.2 配置 nmDRC

在 Virtuoso 版图窗口选择 **Calibre → Run nmDRC**，从空白配置填写：

| 页面／字段 | 填写 |
| --- | --- |
| Rules / Rules File | `<项目根目录>/layout/rule/my_soc_top.drc` |
| Rules / Run Directory | `<项目根目录>/layout/workspace/drc_preDummy` |
| Rules / Check Selection / Recipe | Checks selected in the rules file |
| Inputs / Format | GDSII |
| Inputs / Layout File | `<项目根目录>/layout/workspace/my_soc_top_preDummy.gds` |
| Inputs / Top | my_soc_top |
| Export from Layout Viewer | 取消勾选 |
| Results Database | my_soc_top_preDummy.drc.results，ASCII |
| Summary Report | my_soc_top_preDummy.drc.summary |

Check Selection 在左侧 **Rules** 页内。Run Control 可按服务器资源启用多核和 Hyper；点击 **Run DRC**，等待正常完成。GUI 没有 Maximum Results 等字段时无需寻找，实际输出是否截断以报告为准。

保存 GUI 配置用 **File/Files → Save Runset As**，写到 `layout/runset/my_soc_top_preDummy.drc.runset`。**runset 不能保存成规则文件名**；把 runset 覆盖到 `.drc` 里会报 SYN1 unrecognized operation。

### 7.3 查看结果

RVE 中展开具体规则，点击标记缩放到版图，阅读规则说明及报告坐标。`4-vertex polygon` 只是标记形状，`Not Waived` 表示未豁免结果，均不是 DRC 错误类别。

Dummy 前可能有 M2.R.1/M5.R.1/M6.R.1 金属密度不足、DOD.R.1 缺少 Dummy OD。这些留到第 8 节生成填充后复查，不能此时当作最终通过。

本实验的教学口径单独排除 **DRM、ESD、MOM、LUP** 四类；保留原始报告，其他规则必须处理。此口径不等于工艺正式豁免，不把原始非零报告写成 0 违规。

### 7.4 修正 CO.R.4

若出现本 IO 库的 CO.R.4，显示 **CONT、PIMP、NIMP、POLY1、RPDUMMY**，查看接触孔是否伸出 PIMP 覆盖范围。规则中的 RPDMY 在 Virtuoso 实际层名为 `RPDUMMY`，不是 RPO。

1. 先备份自己的 OA 版图或导出一份修改前 GDS。
2. 在 RVE 定位后，到自己的导入库中进入报错的 IO 子单元；File → Make Editable 启用编辑。必须确认窗口所属库是 `my_soc_top_manual`，不编辑共享 PDK 库。
3. 选择 **PIMP drawing**，按 R 画矩形扩展原 PIMP 区域，覆盖伸出的 CONT，接成原区域的连续延伸；下边与原 PIMP 矩形上沿衔接、右边与相邻 NIMP 边沿对齐。不得越过相邻器件边界盲目扩大。
4. 保存修改过的子单元，回到芯片顶层保存；按第 6 节覆盖导出 `my_soc_top_preDummy.gds`，重跑本节 DRC。
5. 确认 CO.R.4 归零，且未产生新的 implant 间距、器件几何等违规。

同一 master 的修改影响其全部实例，因此既要检查原标记消失，也要检查新增违规。参考轮 CO.R.4 为 1 个层次标记、展开 8 处；自己的数量以实际报告为准。

## 8. 配置并生成 Dummy Fill

使用第 1 节已经复制到 `layout/dummy/manual_final/` 的两份规则。本文逐条运行生成与合并命令，不调用原始 `make dummy` 或 `run_dummy.csh`，避免其中的原始路径和默认参数直接覆盖项目状态。

### 8.1 修改两份规则的输入

在文本编辑器中，分别找到并替换两份文件中的输入行：

```text
LAYOUT PATH "<项目根目录>/layout/workspace/my_soc_top_preDummy.gds"
LAYOUT PRIMARY "my_soc_top"
```

实际文件中必须写真实绝对路径。原值是 `../../cv32e40p_wrapper.gds` 和 `cv32e40p_wrapper`，不要保留。

输出保持原值：金属规则 `DM.gds`/`DM.sum`，OD/PO 规则 `DOD.gds`/`DOD.sum`；输出到执行命令时的目录。

### 8.2 金属 Dummy 参数

打开 `Dummy_Metal_Calibre_0.18um.215a` 项目副本：

| 项目 | 操作 |
| --- | --- |
| WithSealring | 保持开启，输入已包含环 |
| FILL_M1 至 FILL_M6 | 保持开启 |
| MIXED_SCHEME | 保持开启，层/数据类型沿用原规则 |
| THICK_40K_M6 | 把原来的 `#DEFINE THICK_40K_M6` 改为 `//#DEFINE THICK_40K_M6` |
| 其他 THICK_M6、THICK_30K_M6 | 保持关闭，与普通顶层金属一致 |
| 普通 M6 分支的 dmW_M6、dmL_M6 | 两项均由 0.6 改为 0.8 |

最后一项具体搜索下面两行，只改普通分支的这两行：

```text
VARIABLE dmW_M6  0.8
VARIABLE dmL_M6  0.8
```

文件中其他条件分支还有 3.0 或 4.5 的同名参数，不能全部替换。保留其余间距、中/大填充尺寸、输出层和密度参数。

修改原因：本基线 DRC 的 M6.A.1 要求面积至少 0.562 µm²，普通默认 0.6×0.6=0.36 会报错；0.8×0.8=0.64 满足该面积下限，仍需通过最终间距和密度检查。参考轮此问题落在 SRAM 子层次，是新加 Dummy，不是原 SRAM 功能金属损坏。

### 8.3 OD/PO 参数

打开 `Dummy_OD_PO_Calibre_0.18um.210a` 副本，保持 **WithSealring、ADD_DPO_ALSO、MIXED_SCHEME** 开启；其余参数沿用原值，不开启 REVISE_DOD/REVISE_DPO。保存两份规则。

### 8.4 分别运行并检查

Linux 终端逐条执行：

```bash
cd "$SOC_ROOT/layout/dummy/manual_final"
"$CALIBRE_BIN/calibre" -hyper -turbo -hier -drc Dummy_Metal_Calibre_0.18um.215a > METAL.log 2>&1
echo $?
```

退出码 0 后打开 METAL.log、DM.sum，确认正常结束、无 ERROR/FATAL 且 DM.gds 非空。再执行：

```bash
"$CALIBRE_BIN/calibre" -hyper -turbo -hier -drc Dummy_OD_PO_Calibre_0.18um.210a > OD_PO.log 2>&1
echo $?
```

同样检查 OD_PO.log、DOD.sum、DOD.gds。这些生成报告中的 RESULTS 表示输出的填充图形，不是完整芯片 DRC 违规数。

### 8.5 合并完整芯片和填充

仍在 `layout/dummy/manual_final`：

```bash
"$CALIBRE_BIN/calibredrv" -a layout filemerge \
  -in ../../workspace/my_soc_top_preDummy.gds \
  -in DOD.gds -in DM.gds \
  -rename -createtop my_soc_top_postDummy \
  -out ../../workspace/my_soc_top_postDummy.gds > merge.log 2>&1
echo $?
```

确认退出码 0、merge.log 成功结束、最终 GDS 非空。输出是新增顶层 `my_soc_top_postDummy` 包住原芯片及填充；后续 Layout 的 Top 必须跟着改。不要把 DM.gds 或 DOD.gds 单独当成最终芯片。

若重新生成 Dummy，合并输入始终使用干净的 preDummy GDS，不能拿上一次 postDummy 再叠一轮。覆盖 DM 或最终 GDS 前先保留需要的旧版本。

## 9. 导入最终 GDS 并复查 DRC

在 CIW 新建项目库 `my_soc_top_final`，Attach 到 `tsmc18`。File → Import → Stream：Library=`my_soc_top_final`，Stream File 为 `<项目根目录>/layout/workspace/my_soc_top_postDummy.gds`，Top Cell=`my_soc_top_postDummy`。Translate 后打开该顶层 layout，按第 3.4 节设置显示。

检查原芯片、Seal Ring 和 Dummy 都存在；不要误打开 `my_soc_top` 子层次或狭长的 SRAM WORD_ROWS 后就认为顶层不完整。

在 nmDRC 沿用第 7 节设置，仅修改：

| 字段 | Dummy 后值 |
| --- | --- |
| Run Directory | `<项目根目录>/layout/workspace/drc_postDummy` |
| Layout File | `<项目根目录>/layout/workspace/my_soc_top_postDummy.gds` |
| Top | my_soc_top_postDummy |
| Results Database | my_soc_top_postDummy.drc.results |
| Summary Report | my_soc_top_postDummy.drc.summary |

确认 Rules File 仍为自己的 `my_soc_top.drc`、Export from Layout Viewer 关闭，运行 DRC。保存新 runset 到 `layout/runset/my_soc_top_postDummy.drc.runset`。

验收：M2.R.1/M5.R.1/M6.R.1、DOD.R.1、CO.R.4、M6.A.1 及 SR.S.1 归零；按课程口径排除四类后无其余违规。参考轮 523 项检查，原始非零 1036 层次/6206 展开，仅用于核对，不要求自己的计数与之相等。

若 M6.A.1 落在 SRAM 内，先查 DM.gds 中对应形状及第 8.2 节参数；需重新生成金属 Dummy、用原 preDummy 重合并，OD/PO 输入未变时无需重跑 DOD 生成。不要通过修改 SRAM 电路来消除填充错误。

## 10. 从原始规则建立 LVS 配置

### 10.1 修改项目规则副本

打开第 1 节由原始 `calibre.lvs` 复制出的 `layout/rule/my_soc_top_postDummy.lvs`。逐项搜索修改：

| 原语句 | 替换为 |
| --- | --- |
| `LAYOUT PATH "ts_allcelllvs.gds"` | `LAYOUT PATH "<项目根目录>/layout/workspace/my_soc_top_postDummy.gds"` |
| `LAYOUT PRIMARY "ts_allcelllvs"` | `LAYOUT PRIMARY "my_soc_top_postDummy"` |
| `SOURCE PATH "ts_allcelllvs.cdl"` | `SOURCE PATH "<项目根目录>/pnr/my_soc_top/outputs/manual_final/my_soc_top.cdl"` |
| `SOURCE PRIMARY "ts_allcelllvs"` | `SOURCE PRIMARY "my_soc_top"` |
| `TEXT DEPTH PRIMARY` | `TEXT DEPTH 1` |
| `PORT DEPTH PRIMARY` | `PORT DEPTH 1` |
| `VIRTUAL CONNECT COLON YES` | `VIRTUAL CONNECT COLON NO` |
| `LVS REDUCE SPLIT GATES NO` | `LVS REDUCE SPLIT GATES YES` |

Top 是 GUI 字段名，PRIMARY 是规则语言的关键字，不要把规则代码中的 PRIMARY 改成 TOP。

TEXT/PORT DEPTH 设 1，用于读取合并后外壳及下一层原芯片的文字端口。关闭 COLON 虚连，避免按名字代替真实连接。Split Gate 允许归约信号 PAD 内部等效的分拆 MOS 网络，不是忽略缺失器件。

### 10.2 设置本例电阻比较容差

在同一个副本中搜索 **`TRACE PROPERTY R(PR) R R`**。原文件有三条，末尾分别为 2、2、0，全部改为：

```text
TRACE PROPERTY R(PR) R R 3.48
```

三个条件分支仍保留，运行时激活相应分支。只改这三条 PR 阻值容差，不改其他属性、RVPR、器件提取公式或源 SPI/CDL，不为了此设置开启 ACCURACY 或 RC_DECK。

本例 IO 电阻的版图提取值相对源值系统性偏高约 3.47%，例如 248.8 Ω对应 257.44 Ω；源值来自 IO SPI，并非 Innovus 布线寄生。3.48%是此教学实例采用的比较容差，不是自动适用于其他工艺/设计的标准，也不代表修复了模型系列差异。其他同学若出现不同偏差，应先查模型和实际几何，不能持续放大容差求通过。

### 10.3 配置 nmLVS

在最终版图窗口 **Calibre → Run nmLVS**。从空白配置开始，填写：

| 页面／字段 | 填写 |
| --- | --- |
| Rules / Rules File | `<项目根目录>/layout/rule/my_soc_top_postDummy.lvs` |
| Rules / Run Directory | `<项目根目录>/layout/workspace/lvs_postDummy` |
| Inputs / Run | Hierarchical |
| Inputs / Step | Layout vs Netlist |
| Layout Format | GDSII |
| Layout File | `<项目根目录>/layout/workspace/my_soc_top_postDummy.gds` |
| Layout 区域 / Top | my_soc_top_postDummy |
| Export from Layout Viewer | 取消勾选 |
| Inputs / Layout Netlist | my_soc_top_postDummy.layout.sp |
| Source/Schematic Format | SPICE |
| **SPICE_FILE** | `<项目根目录>/pnr/my_soc_top/outputs/manual_final/my_soc_top.cdl` |
| Source/Schematic 区域 / **Top** | my_soc_top |
| Export from Source/Schematic Viewer | 取消勾选 |
| Outputs / LVS Report | my_soc_top_postDummy.lvs.report |
| SVDB Directory | svdb，保留 QUERY |

**Layout Netlist 在 Inputs 中**，是本次提取网表的保存文件；不是拿它代替 SPICE_FILE 输入。Step 使用 Layout vs Netlist，不能选 Netlist vs Netlist 去比较旧提取结果。

在左侧 **Options → Gate Reduction** 勾选 **Reduce split gates**。Options 没显示时，用 **Settings → Show Pages → Options** 打开。Classic 界面对应 Setup → LVS Options → Gates。

在 **Options → Trace Property：取消勾选，下面表格留空，不用填写 3.48%。** Classic 界面对应 Properties 中的 Use settings from rules。不要加载自带 runset 中的单项 PR 表格；此前这种配置会把完整 605 条属性检查缩减成 1 条。

Hcells/Box 不新增 SRAM Box，Virtual Connect 不新增虚连。保持 VDD/VDDPST 为 POWER、VSS/VSSPST 为 GROUND；本规则已经包含这些名称，不合并四个供电网。

ERC 页：**LVS Execute ERC=Yes**，Check Selection Recipe=`Checks selected in the rules file`，结果库 `my_soc_top_postDummy.erc.results`，摘要 `calibre_erc.sum`。若 ERC 页未显示，用 Settings → Show Pages 显示。

保存配置到 `layout/runset/my_soc_top_postDummy.lvs.runset`，不要覆盖 `.lvs` 规则。点击 **Run LVS**。

### 10.4 查看与判断

打开 `my_soc_top_postDummy.lvs.report`，检查 **OVERALL COMPARISON RESULTS = CORRECT**，再检查 LVS PARAMETERS：

- `LVS REDUCE SPLIT GATES YES`。
- PR 属性语句为 `TRACE PROPERTY r(pr) r r 3.48`。
- 完整属性列表仍保留，本基线参考 605 条。
- 没有 SRAM Box 或用虚连掩盖断路。

检查本次生成的 `_my_soc_top_postDummy.lvs_` 是否引用正确 GDS/CDL 及规则；它是工具自动生成的 wrapper，用于核对，不手工改它。

也可在 Linux 终端核对实际报告中的属性条目（下面用的是 `rg`，即 ripgrep；服务器上没有安装时，改用等价的 `grep -c` / `grep -n -i`，其余参数不变）：

```bash
rg -c '^[[:space:]]*TRACE PROPERTY[[:space:]]' \
  "$SOC_ROOT/layout/workspace/lvs_postDummy/my_soc_top_postDummy.lvs.report"
rg -n -i 'TRACE PROPERTY[[:space:]]+r\(pr\)|LVS REDUCE SPLIT GATES' \
  "$SOC_ROOT/layout/workspace/lvs_postDummy/my_soc_top_postDummy.lvs.report"
```

本基线应看到 605 条、PR3.48 及 Split Gates YES。若只有 1 条，先检查是否误勾 GUI Trace Property；不要以此时的 CORRECT 替代完整属性验收。

参考轮端口 11/11、网络 142757/142757、器件 216153/216153 匹配。CDL 原顶层声明六信号加四 PG 共 10 个，IO 的 GLOBAL POC 使源端口比较出现第 11 个，不需要手工把 POC 硬塞进功能 RTL。

常见问题：

| 现象 | 检查／处理 |
| --- | --- |
| specify schematic cell | Source/Schematic 区域 Top 填 my_soc_top，取消 Export from Source/Schematic Viewer |
| 缺少 POC 端口 | 查第 5 节标签是否在正确层次、锚点在 M3 导体内，以及导出的实际 GDS 和 TEXT/PORT DEPTH |
| 六个信号 PAD 出现成组器件/网络差异 | 核对 Split Gate YES 实际生效，再看真实连接，不先删器件 |
| 只剩 PR 阻值属性差异 | 核对三处 3.48、GUI Trace Property 未勾选及本次实际生效参数 |
| 缺少 cell 定义 | 查 CDL 的 include 路径及对应库内容，不能用 Box 直接跳过 |

若必须在 Dummy 后补标签：编辑 `my_soc_top_final/my_soc_top_postDummy/layout`，保存并以 Top Cell=`my_soc_top_postDummy`、Stream File=`my_soc_top_postDummy.gds` 重新导出，再复跑。把同样标签同步到 preDummy 源 OA，否则以后重合并会丢失该修正。仅增加标签不需要重新生成 Dummy。

## 11. ERC 查看与处理

ERC 是 Electrical Rule Check，检查井/衬底偏置及可疑电气连接。LVS CORRECT 不代表 ERC 也无结果。

在 RVE 中选择 **View → ERC → ERC Results**，展开规则，选中标记查看位置和层次。此基线参考残项：

| 规则 | 层次数（展开数） | 已知来源 |
| --- | --- | --- |
| mnpg | 4（832） | IO 原库的跨电源轨保护 MOS |
| floating.nxwell_float | 1（72） | PVSS1CDG 内保护二极管的井连接 |
| npvss49 | 1（72） | 同一原库保护二极管结构 |
| floating.psub_float | 1（1） | IO 的 PSUB2 分隔带之外、包含 Seal Ring 的外圈提取衬底 |

这组计数用于识别本例，不表示可忽略所有同名错误。新增核心区浮井、实际供电开路等须修复。原库保护电路的端点不得为消标记随意改接。

**不要跨 Seal Ring 装配隔离带画金属接 VSS。** 曾经四边加宽金属使 psub 标记消失，但引入四处真实 SR.S.1；该规则要求 5 µm 装配隔离禁布。把环改接 VSS、缩窄线或换金属层都不能绕过禁布要求，也不能靠虚假标签假装接地。

外圈 psub 标记的几何形成机制已定位，源环单元与工艺样例一致，但尚无适用于本项目的正式豁免或已验证修正。本次教学流程将其及原库项留待评审；课程验收时保留具体结果，并向负责人确认验收口径。**不关闭 ERC、不写 ERC Clean，也不把暂缓处理说成已修复。** 可继续以下独立天线与 WB 检查。

## 12. 最终 GDS 独立天线检查

使用第 1 节从原始 `ant.drc` 复制出的 `layout/rule/my_soc_top_postDummy.ant.drc`。编辑其输入输出为：

```text
LAYOUT PATH "<项目根目录>/layout/workspace/my_soc_top_postDummy.gds"
LAYOUT PRIMARY "my_soc_top_postDummy"
DRC RESULTS DATABASE "<项目根目录>/layout/workspace/ant_postDummy/my_soc_top_postDummy.ant.results" ASCII
DRC SUMMARY REPORT "<项目根目录>/layout/workspace/ant_postDummy/my_soc_top_postDummy.ant.summary"
```

填写真实绝对路径，其余工艺选项沿用原 ant 规则；本基线 THICK_20K/40K 均关闭，不额外添加 DRC SELECT/UNSELECT。

在 nmDRC 填写：

| 字段 | 填写 |
| --- | --- |
| Rules File | `<项目根目录>/layout/rule/my_soc_top_postDummy.ant.drc` |
| Run Directory | `<项目根目录>/layout/workspace/ant_postDummy` |
| Rules / Check Selection Recipe | Checks selected in the rules file |
| Layout File / Top | 最终 postDummy GDS / my_soc_top_postDummy |
| Export from Layout Viewer | 取消勾选 |
| Results Database | my_soc_top_postDummy.ant.results，ASCII |
| Summary Report | my_soc_top_postDummy.ant.summary |

运行，保存 runset 到 `layout/runset/my_soc_top_postDummy.ant.runset`。读取摘要，确认正确规则与最终输入、完整层次、检查非空、天线违规 0；本基线参考 62 项/0 违规。Innovus 中 Dummy 前的天线 0 不能替代这次检查。

若出现天线违规，按具体网络与天线规则定位，回到可维护的设计源修正，再重新生成受影响输出及最终验证；不能通过关闭天线规则取得 0。

## 13. 从工艺原包取得 Wire Bond PAD 规则

原始 `layout/rule/` 没有 WB 规则，需要同学自行从服务器工艺归档取得。

### 13.1 提取项目规则

在 Linux 终端执行：

```bash
cd "$SOC_ROOT"
mkdir -p layout/rule/wb_reference
unzip -p "$PDK_ROOT/tsmc180/Foundry Design Documents/t000cldr002c1_1_9_1a.zip" \
  DRC_Calibre_AL_WB_FC_v19_1a1.tar.gz \
  > layout/rule/wb_reference/DRC_Calibre_AL_WB_FC_v19_1a1.tar.gz
echo $?
```

确认退出码 0、tar.gz 非空，再执行：

```bash
tar -xOf layout/rule/wb_reference/DRC_Calibre_AL_WB_FC_v19_1a1.tar.gz \
  wire_bond/0.18um/C18_WIRE_BOND_6M.19_1a1 \
  > layout/rule/my_soc_top_postDummy.wb.drc
echo $?
```

确认文件头是 `C18_WIRE_BOND_6M.19_1a1` 规则。若工艺包路径不存在，先取得课程使用的同版 Wire Bond 包，不用 ant.drc 或普通几何 DRC 替代。

### 13.2 修改本例选项及路径

打开提取出的项目文件：

```text
#DEFINE PITCH_55_SINGLE
//#DEFINE PITCH_60_STAGGER
#DEFINE OBV_CUP
```

PITCH_55_SINGLE 原来是注释，需启用；60_STAGGER 原来开启，需关闭；OBV_CUP 原来注释，需启用。其他 pitch、TRL_CUP、TRL_DFS、OBV_DFS、MIX 保持关闭。

本例 34 个 `PAD60LU_OBV` 四边单排，同边最小节距 100 µm，CB 开窗 53×66 µm，匹配此单排类别和 OBV CUP 结构。55 是规则类别的最低节距，不是把实际间距改成 55。不同 PAD 型号或排列要重新确定选项。

输入输出改为：

```text
LAYOUT PATH "<项目根目录>/layout/workspace/my_soc_top_postDummy.gds"
LAYOUT PRIMARY "my_soc_top_postDummy"
DRC RESULTS DATABASE "<项目根目录>/layout/workspace/wb_postDummy/my_soc_top_postDummy.wb.results" ASCII
DRC SUMMARY REPORT "<项目根目录>/layout/workspace/wb_postDummy/my_soc_top_postDummy.wb.summary"
```

填写真实绝对路径，保留 `VARIABLE SRwidth 15.0` 及其余规则数值、原 EXCL 处理，不新增豁免。

### 13.3 运行并验收

在 nmDRC 把 Rules File 切换到该 WB 副本、Run Directory 切到 `<项目根目录>/layout/workspace/wb_postDummy`，Layout File 和 Top 仍是最终 postDummy；Export from Layout Viewer 关闭，Recipe 为 Checks selected in the rules file。

输出栏也改成 `my_soc_top_postDummy.wb.results` 和 `my_soc_top_postDummy.wb.summary`，避免沿用天线报告名；Run DRC，保存 runset 到 `layout/runset/my_soc_top_postDummy.wb.runset`。

本基线参考 64 项检查/0 违规，包括 CUPVIAT.DN.1_。PAD 库依据旧版 WB 规则，其 release note 中的旧版已知豁免不能自动套到新版 deck；如命中应核对具体规则及几何。自动 WB 检查之外的线径、探针/键合工艺和未编码人工规则另行确认。

## 14. 最终复核和完成标准

所有最终检查应对应 **同一个最终 GDS**。最后一次版图修改和导出完成后，依次重跑几何 DRC、LVS/ERC、Antenna 和 WB；确认每次实际 wrapper 中的输入路径、Top 和 INCLUDE 正确。已有结果只要仍对应相同输入即可使用，改过版图不能沿用修改前结果。

可以在 Linux 终端记录输入哈希，再进行检查：

```bash
cd "$SOC_ROOT"
sha256sum layout/workspace/my_soc_top_postDummy.gds \
  pnr/my_soc_top/outputs/manual_final/my_soc_top.cdl \
  > layout/workspace/final_inputs.sha256
```

检查全部结束后：

```bash
cd "$SOC_ROOT"
sha256sum -c layout/workspace/final_inputs.sha256
```

都为 OK 说明记录后输入内容未变；还须核对各报告实际读的是这些文件。若输入变过，更新哈希并重跑受影响检查。

| 项目 | 完成条件 |
| --- | --- |
| 几何 DRC | 课程四类排除后无其余违规，原始结果及例外单独保留 |
| LVS | CORRECT，完整属性检查、3.48%教学 PR 容差和 Split Gate 实际生效 |
| ERC | 查看并记录；真实错误修正，本例原库/外圈残项保留待评审，不冒充 Clean |
| Antenna | 最终 GDS 独立检查 0 违规 |
| Wire Bond PAD | 正确规则及 PAD 选项下独立检查 0 违规 |
| 文件一致性 | 最终 GDS、CDL、配置和报告对应，保存日志和输入哈希 |

参考轮各项通过数不能代替自己的报告。工具正常结束、退出码 0、RVE 显示笑脸，都要结合对应检查和完整报告判断。

到这里可完成本课程的版图制作与已列明的教学验证。正式投片还需处理 ERC 验收口径、工艺模型适配、Dummy 后寄生/STA、IR/EM 及封装人工规则；本指南不把这些未执行项目列为通过。

## 15. 自己生成交付归档

准备一份本次验收说明，写明设计名、最终 GDS 和 CDL 的 Top、各报告路径和结论、DRC 四类实验排除、PR3.48%容差、ERC 残项及其他限制，不复制他人的“通过”状态。

从项目根目录执行下面命令，把**自己本轮生成**的文件打包：

```bash
cd "$SOC_ROOT"
export SOC_DELIVERY="$SOC_ROOT/deliverables/my_soc_top_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$SOC_DELIVERY"
tar -czf "$SOC_DELIVERY/design_and_reports.tar.gz" \
  pnr/my_soc_top/outputs/manual_final \
  pnr/my_soc_top/reports/postFiller \
  pnr/my_soc_top/my_soc_top_postFiller_antenna.rpt \
  pnr/my_soc_top/my_soc_top_postFiller_drc.rpt \
  pnr/my_soc_top/my_soc_top_postFiller_pg.rpt \
  pnr/my_soc_top/my_soc_top_postFiller_place.rpt \
  pnr/my_soc_top/my_soc_top_postFiller_signal.rpt \
  config \
  layout/workspace/my_soc_top_preDummy.gds \
  layout/workspace/my_soc_top_postDummy.gds \
  layout/workspace/final_inputs.sha256 \
  layout/workspace/drc_preDummy \
  layout/workspace/drc_postDummy \
  layout/workspace/lvs_postDummy \
  layout/workspace/ant_postDummy \
  layout/workspace/wb_postDummy \
  layout/rule/my_soc_top.drc \
  layout/rule/my_soc_top_postDummy.lvs \
  layout/rule/my_soc_top_postDummy.ant.drc \
  layout/rule/my_soc_top_postDummy.wb.drc \
  layout/dummy/manual_final \
  layout/runset/my_soc_top_preDummy.drc.runset \
  layout/runset/my_soc_top_postDummy.drc.runset \
  layout/runset/my_soc_top_postDummy.lvs.runset \
  layout/runset/my_soc_top_postDummy.ant.runset \
  layout/runset/my_soc_top_postDummy.wb.runset \
echo $?
```

退出码应为 0；若提示文件不存在，补齐对应产物或按自己的实际名称修正清单，不能忽略打包失败。把自己写的验收说明另存入 `$SOC_DELIVERY`。

校验压缩包并生成校验值：

```bash
gzip -t "$SOC_DELIVERY/design_and_reports.tar.gz"
echo $?
cd "$SOC_DELIVERY"
sha256sum design_and_reports.tar.gz > SHA256SUMS
sha256sum -c SHA256SUMS
```

保留活动 OA 库及 Innovus 检查点和配套 `.enc.dat`，上述包不包含完整可恢复工作环境。CDL 仍引用原工艺 SPI/SRAM 库绝对路径，迁移时须提供相应库环境并更新工作副本路径。

最终物理版图是 **`my_soc_top_postDummy.gds`**；Innovus 导出的 `my_soc_top.gds.gz` 没有后来加入的 Seal Ring/Dummy。已有 SPEF/SDF 和时序报告属于 Innovus 阶段，不表示做过 Dummy 后重新提取。

## 附录：常用 GUI 动作与复跑提示

| 动作 | 操作 |
| --- | --- |
| 编辑模式 | File → Make Editable |
| 查看／改属性 | 选中对象后 Q |
| 实例精确放置 | I 放一次、Esc、Q 改 Origin |
| 矩形／路径／标签 | R / P / L，先确认当前 Layer/Purpose |
| 相对定长移动 | M、F3，取消 Treat as absolute point，填 Delta，Apply 一次 |
| 退出命令 | Esc |
| 完整层次显示 | 显示设置后 Shift+F、F |
| 保存 GUI 配置 | Save Runset As，写入 layout/runset，不覆盖规则 |
| 查 ERC | RVE：View → ERC → ERC Results |

补线前先确认导体所属网络及跨层 via。不要沿用手工补 M6 供电线的做法，本流程已由 Innovus 完成四边 sroute；不要因为 LEF 抽象的 VDDPST/VSSPST no routing 提示就擅自合并电源。最终原生 GDS 连通由完整 LVS 验证。

运行结果找不到时，先检查 GUI 实际 Run Directory 和输出文件名。GUI 可能保留上次目录或覆盖规则中的输出名；以实际 wrapper 及 summary 判断，不能因预期文件名不存在就宣布通过或失败。

本文与《Lab 4：逻辑综合 & 物理实现》配套使用；最终天线 / WB 等完整检查要求可对照[课程 Template 文档](https://pku-pacific-lab-team.github.io/Tape-out_template_doc/)。上述四类 DRC 教学排除不能扩大成 ERC 全部豁免。
