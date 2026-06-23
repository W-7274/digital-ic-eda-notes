# EDA流程学习笔记

## 0. 这份笔记解决什么问题

数字 IC 的 EDA 流程看起来像一堆脚本和文件夹，但本质上是一条数据逐步变具体的链路：

```text
RTL 行为描述
  -> 门级网表
  -> 带物理位置和连线的后布局设计
  -> 带真实寄生参数的时序分析
  -> 带延迟标注的门级仿真
```

也可以理解为：

```text
前端关心：电路功能是否对，能不能综合成门级逻辑
后端关心：门级逻辑放到芯片上以后，面积、电源、时钟、布线、时序是否合理
签核关心：在更准确的库和寄生参数下，时序是否真的满足
后仿关心：加上真实延迟后，功能是否还能正确
```

脚本的作用不是“神秘地生成结果”，而是把工具操作固定下来：

```text
设置路径和库
读入设计
读入约束
执行当前阶段的核心操作
检查结果
输出给下一阶段使用的文件
```

所以学习 EDA 流程时，不要只背脚本名字，而要抓住三个问题：

```text
这个阶段读什么？
这个阶段做什么？
这个阶段输出什么，下一阶段怎么用？
```

## 1. 总流程概览

一个教学版数字 IC 工程通常可以分为以下阶段：

```text
rtl/                  编写或存放正式设计 RTL
vcs_pre/              RTL 前仿真，验证功能
dc/                   逻辑综合，把 RTL 变成门级网表
icc/                  布局布线，把门级网表变成物理版图
pt/                   静态时序分析，检查后布局时序
vcs_icc/              后仿真，验证带 SDF 延迟的门级电路功能
```

整体数据流如下：

```text
RTL(.v)
  |
  | VCS 前仿真：检查 RTL 功能
  v
功能正确的 RTL
  |
  | DC 综合：RTL -> 标准单元门级网表
  v
mapped.v / mapped.ddc / mapped.sdc / reports
  |
  | ICC 布局布线：门级网表 -> 物理实现
  v
后布局网表 / SDF / SPEF / GDS / reports
  |
  | PT 静态时序分析：后布局网表 + SPEF + SDC
  v
更准确的 timing reports / SDF
  |
  | VCS 后仿真：后布局网表 + 标准单元模型 + SDF
  v
后仿真日志和波形
```

## 2. 推荐文件夹规划

### 2.1 顶层目录

一个较清晰的工程可以这样组织：

```text
project/
├── rtl/
│   ├── design_a/
│   └── design_b/
├── vcs_pre/
├── dc/
├── icc/
├── pt/
├── vcs_icc/
└── document/
```

各目录的职责：

```text
rtl/       正式设计 RTL，只放可综合设计文件，不放 testbench
vcs_pre/   RTL 前仿真环境，可以放 testbench、filelist、Makefile
dc/        逻辑综合环境，包含脚本、输出和报告
icc/       后端物理实现环境，包含 setup、run、out、report
pt/        静态时序分析环境
vcs_icc/   后仿真环境
document/  实验截图、报告、笔记
```

### 2.2 为什么 rtl 和 vcs_pre 里可能都有 RTL

有些教学工程会在 `vcs_pre/` 下放一份前仿 RTL，在 `rtl/` 下再放一份综合 RTL。更规范的做法是：

```text
rtl/        放唯一正式 RTL
vcs_pre/    file_list.f 指向 rtl/ 里的设计文件
```

如果为了教学或快速实验复制了一份 RTL，也要保证：

```text
前仿 RTL 和综合 RTL 内容一致
testbench 不进入 dc 综合
```

### 2.3 每个阶段都应该分 out 和 report

建议所有阶段都区分：

```text
scr/ 或 scripts/     脚本
run/ 或 work/        工具运行目录
out/                 给下一阶段使用的正式输出
rep/ 或 rpts/        报告
log/                 日志
```

这样做的好处是：

```text
脚本不会和输出混在一起
下一阶段知道从哪里拿文件
报告和可交付文件不会混淆
清理中间文件时不容易误删脚本
```

## 3. 脚本分工的一般规律

EDA 脚本多，但分工通常有规律。

### 3.1 公共路径和库配置脚本

常见文件名：

```text
MPW180_lib_list.tcl
common_setup.tcl
```

作用：

```text
定义库根目录
定义标准单元库路径
定义 IO 库路径
定义 tech file / TLUPlus / Milkyway reference library
定义工程顶层设计名 DESIGN_NAME
定义输入输出目录
```

这类脚本的核心思想是：**把容易变化的路径和设计名集中管理**。

例如：

```tcl
set MPW180 "/home/wzs/SMIC/smic180_lib_list"
set MPW180_path "$MPW180/std"
set MPW180_dbpath "$MPW180_path/liberty"
set MPW180_mw "$MPW180_path/milkyway"
```

后面不要到处写绝对路径，而是用变量拼接：

```tcl
set MPW180_db_ss_125c "$MPW180_dbpath/scc018ug_hd_rvt_ss_v1p62_125c_basic.db"
```

这样库位置从学校服务器换到本机时，只需要改开头的根路径。

### 3.2 MPW180_lib_list.tcl 和 common_setup.tcl 为什么看起来重复

这两个文件都会出现“库”和“路径”，所以初学时容易觉得它们重复。更准确的分工是：

```text
MPW180_lib_list.tcl  = 全部工艺库路径清单
common_setup.tcl     = 工程配置：选择本工程使用哪些库，并定义工程自己的路径和设计变量
```

`MPW180_lib_list.tcl` 负责回答：

```text
SMIC180 这套库放在哪里？
标准单元 .db 在哪里？
symbol 库在哪里？
Milkyway reference library 在哪里？
tech file 在哪里？
TLUPlus 和 map file 在哪里？
```

它更像一个“库目录表”。只要工艺库位置和目录结构不变，一般都不用改它。

`common_setup.tcl` 负责回答：

```text
工程顶层设计叫什么？
工程 RTL 去哪里找？
综合时选择哪个 .db 作为 target library？
本工程的 Milkyway design library 叫什么？
当前工具流程要引用哪些 reference library、tech file 和 TLUPlus？
```

所以 `common_setup.tcl` 不是重新定义一整套库，而是从 `MPW180_lib_list.tcl` 定义好的库变量里挑选本工程要用的部分。

可以用下面的关系理解：

```text
MPW180_lib_list.tcl
    先定义“有哪些库、库在哪里”

common_setup.tcl
    使用 MPW180_lib_list.tcl 中已经定义好的库路径变量
    决定“本工程用哪些库、RTL 在哪里、顶层设计名是什么”
```

如果工程约定 `.synopsys_dc.setup` 是 DC 启动入口，更清晰的 source 关系是：

```text
.synopsys_dc.setup
    source MPW180_lib_list.tcl
    source common_setup.tcl
    source dc_setup.tcl
```

这样启动文件统一装配 DC 环境：先加载库路径变量，再加载工程配置，最后把配置设置给 DC。

维护原则可以这样理解：

```text
库位置、库目录结构变化
    主要影响 MPW180_lib_list.tcl

工程顶层、RTL 目录、约束对象变化
    主要影响 common_setup.tcl、运行脚本和 SDC
```

例如库位置从：

```text
/home/publib/digital/smic180_lib_list
-> /home/wzs/SMIC/smic180_lib_list
```

这种情况主要调整：

```text
MPW180_lib_list.tcl
```

如果更换工艺，例如从 SMIC180 换到另一套工艺，则需要重新准备对应工艺的库清单，`common_setup.tcl` 中引用的库变量也要对应调整。

### 3.3 工具环境 setup 脚本

常见文件名：

```text
dc_setup.tcl
icc_setup.tcl
pt_setup.tcl
.synopsys_dc.setup
```

作用：

```text
source 公共库路径脚本
设置 search_path
设置 target_library / link_library
设置工具启动时需要的变量
设置输入文件名
创建或打开设计数据库
```

它们不是主要执行流程，而是让工具进入正确环境。

### 3.4 约束脚本

常见文件名：

```text
sdc.tcl
```

作用：

```text
定义时钟
定义输入输出延迟
定义时钟不确定度
定义 false path / multicycle path
定义负载和驱动能力
```

SDC 的本质是告诉综合和时序工具：

```text
这个电路将在什么时钟下工作
外部输入什么时候到达
输出需要什么时候稳定
哪些路径不需要按普通同步路径检查
```

没有约束，工具不知道要优化到什么目标。

### 3.5 主运行脚本

常见文件名：

```text
dc_run.tcl
init_design_icc.tcl
design_run.tcl
sts.tcl
sdf_gen.tcl
```

作用：

```text
读入设计
执行当前阶段核心流程
检查结果
生成报告
输出给下一阶段使用的文件
```

主运行脚本应该尽量使用变量：

```tcl
read_verilog $rtl_path/${DESIGN_NAME}_pred.v
write_sdf ../out/${DESIGN_NAME}.sdf
```

避免写死：

```tcl
read_verilog fixed_top_pred.v
write_sdf fixed_top.sdf
```

因为硬编码会让脚本和某一个具体设计绑定，降低复用性。

## 4. VCS_pre：RTL 前仿真

### 4.1 这一阶段在验证什么

VCS 前仿真验证的是 **RTL 功能是否正确**。

它不关心：

```text
标准单元延迟
布线延迟
版图面积
功耗
GDS
```

它只关心：

```text
输入激励给进去后，RTL 输出是否符合预期
复位是否正确
时钟采样是否正确
边界条件是否正确
```

如果前仿真都不通过，后面综合和布局布线没有意义。

### 4.2 典型文件

```text
vcs_pre/
├── Makefile
├── file_list.f
├── design/
│   ├── top.v
│   └── top_tb.v
├── simv
├── sim.log
├── *.vcd / *.vpd
├── *.vdb
├── csrc/
├── simv.daidir/
└── DVEfiles/
```

### 4.3 file_list.f 的作用

`file_list.f` 告诉 VCS 要编译哪些 Verilog 文件。

前仿真一般包括：

```text
设计 RTL
testbench
package、interface 等公共定义
include 文件搜索路径
编译期宏定义
```

例如：

```text
./design/top.v
./design/top_tb.v
```

注意：

```text
前仿真要读 testbench
DC 综合不能读 testbench
```

filelist 不只是文件名清单，还承担编译顺序和预处理配置管理。一般按照下面顺序组织：

```text
package、宏和interface等公共定义
→ 底层RTL模块
→ RTL顶层模块
→ testbench
```

如果后面的文件使用了前面尚未编译的 package 或 interface，VCS 可能报告类型或定义不存在。因此，多文件工程应明确管理顺序，不要依赖目录遍历的偶然结果。

#### 编译前预处理：宏定义与include目录

VCS正式解析 Verilog/SystemVerilog 语法前，会先执行预处理。预处理主要处理：

```text
`include     把头文件内容插入当前位置
`define      定义宏
`ifdef       根据宏是否存在决定某段代码是否参与编译
```

假设目录为：

```text
project/
├── rtl/
│   ├── include/
│   │   └── des_config.vh
│   └── des_round.sv
└── vcs_pre/
    ├── tb/
    │   └── des_round_tb.sv
    └── file_list.f
```

头文件 `des_config.vh` 定义公共编译期常量：

```verilog
`define HALF_WIDTH      32
`define ROUND_KEY_WIDTH 48
```

RTL 通过 `` `include`` 使用该文件：

```systemverilog
`include "des_config.vh"

module des_round (
    input  logic [`HALF_WIDTH-1:0]      data_in,
    input  logic [`ROUND_KEY_WIDTH-1:0] round_key,
    output logic [`HALF_WIDTH-1:0]      data_out
);

    always_comb begin
        data_out = data_in ^ round_key[31:0];
    end

`ifdef ENABLE_TRACE
    always_comb begin
        $display("data_in=%h data_out=%h", data_in, data_out);
    end
`endif

endmodule
```

对应 `file_list.f`：

```text
+incdir+../rtl/include
+define+ENABLE_TRACE

../rtl/des_round.sv
./tb/des_round_tb.sv
```

这里发生了两件不同的事。

```text
+incdir+../rtl/include
    给预处理器增加头文件搜索路径
    当代码遇到 `include "des_config.vh" 时，到该目录寻找文件

+define+ENABLE_TRACE
    从编译命令定义ENABLE_TRACE宏
    因此 `ifdef ENABLE_TRACE 中的调试代码参与编译
```

如果删除 `+define+ENABLE_TRACE`，`` `ifdef ENABLE_TRACE`` 中的调试 `$display` 不会进入本次编译，但主体功能逻辑不受影响。

需要注意：

```text
+incdir+只增加搜索路径，不会自动编译目录中的所有文件
普通RTL、package和interface仍应在filelist中明确列出
.vh/.svh通常作为头文件被`include，不应重复当作独立模块随意编译
```

宏本质上是语法解析前的文本替换，没有类型和模块作用域。例如：

```verilog
logic [`HALF_WIDTH-1:0] data;
```

预处理后近似变为：

```verilog
logic [31:0] data;
```

因此宏更适合：

```text
条件编译
全局编译开关
仿真调试代码开关
跨文件公共文本常量
```

模块位宽、流水级数等具有设计语义的配置，通常优先使用有作用域的 `parameter`。不要为了方便把所有设计参数都写成全局宏，也要避免因 `` `ifdef`` 造成仿真代码与综合代码行为不一致。

### 4.4 Makefile 的作用

Makefile 不是 VCS 自动识别的文件，而是 Linux 下 `make` 工具读取的命令封装。

它的核心作用是：

```text
把长命令封装成短命令
统一编译、仿真、看波形、看覆盖率和清理流程
减少手动输入长命令造成的错误
```

常见目标：

```text
make compile 编译
make sim     运行仿真
make debug   打开 DVE/Verdi 查看波形
make cov     打开覆盖率数据库
make clean   清理中间文件
```

一个较完整的 VCS 前仿真 Makefile 可以分成两部分：

```text
变量区：定义输出名、编译命令、仿真命令、覆盖率选项
目标区：定义 make compile / make sim / make debug / make cov / make clean
```

#### .PHONY

```makefile
.PHONY: compile sim clean debug cov
```

`phony` 不是缩写，意思是“假的、伪的”。`.PHONY` 表示这些 target 是伪目标，不是真实文件名。

例如：

```text
clean 不是一个文件
clean 是一个命令入口
```

这样即使目录里真的存在一个叫 `clean` 的文件，执行 `make clean` 时也会运行 `clean:` 后面的清理命令。

#### OUTPUT

```makefile
OUTPUT = simv
```

`OUTPUT` 用来统一管理 VCS 编译后生成的仿真可执行文件名。

后面：

```makefile
-o ${OUTPUT}
./${OUTPUT}
```

会展开为：

```text
-o simv
./simv
```

这样输出文件名只需要在一个地方维护。

#### VCS 编译命令

```makefile
VCS_HOME ?= /home/wzs/synopsys_tools/VCS_2018/vcs/O-2018.09-SP2
VCS_BIN = bash ${VCS_HOME}/bin/vcs
LD_FLAGS = -LDFLAGS "-Wl,--no-as-needed -Wl,--copy-dt-needed-entries"

VCS = ${VCS_BIN} -full64 -sverilog +v2k \
      -f file_list.f \
      -timescale=1ns/1ns \
      -l com.log \
      -o simv \
      ${LD_FLAGS}
```

学习时要理解：

```text
- VCS_HOME            VCS 安装目录
- VCS_BIN             VCS 启动命令
-f file_list.f        从 filelist 读文件
-timescale=1ns/1ns    设置仿真时间单位和精度
-l com.log            保存编译日志
-LDFLAGS             传给系统链接器的兼容参数
-o simv               指定仿真可执行文件名
+v2k                  打开 Verilog-2001 相关语法支持
```

这些选项的含义：

```text
vcs
    调用 VCS 编译器。VCS 会把 RTL/testbench 编译、展开、链接，生成仿真可执行文件。

VCS_HOME
    VCS 的安装目录。使用 `?=` 定义时，如果外部环境已经定义了 VCS_HOME，就使用外部环境；
    如果没有定义，就使用 Makefile 中给出的默认路径。

VCS_BIN
    实际调用 VCS 的命令。
    有些本地环境中，终端里的 `vcs` 是 alias，例如 `bash .../bin/vcs`。
    Makefile 不会展开 shell alias，所以需要在 Makefile 中显式写出真正的调用方式。

-full64
    使用 64 位模式。适合现代服务器环境和较大设计。

-sverilog
    按 SystemVerilog/Verilog 语法编译。即使代码主要是 Verilog，加上也通常没问题。

+v2k
    打开 Verilog-2001 相关语法支持。对于较老版本 VCS 和一些教学工程，保留该选项更稳妥。

-f file_list.f
    从 filelist 读取要编译的设计文件和 testbench。

-timescale=1ns/1ns
    设置默认时间单位和时间精度。如果 testbench 中写 #5，在该设置下表示 5 ns。

-l com.log
    保存编译日志。语法错误、warning、top module 等信息会写入该文件。

-LDFLAGS
    把后面的参数传给 Linux 系统链接器。
    某些老版本 VCS 在较新的 Linux/WSL 环境下链接运行库时，可能出现大量 undefined reference。
    `-Wl,--no-as-needed -Wl,--copy-dt-needed-entries` 用来保留 VCS 运行库的间接依赖，解决这类链接兼容问题。

-o ${OUTPUT}
    指定生成的仿真可执行文件名。
```

在 Ubuntu/WSL 中，终端里手动输入的 `vcs` 可能是 alias，例如：

```text
vcs='bash /home/wzs/synopsys_tools/VCS_2018/vcs/O-2018.09-SP2/bin/vcs'
```

但 Makefile 不会展开 alias。
如果 Makefile 直接执行 VCS 脚本，老版本 VCS 可能因为 `/bin/sh` 与 `dash` 的兼容问题报错：

```text
/bin/sh: 0: Illegal option -h
```

因此在 Makefile 中显式使用 bash 调用 VCS 脚本：

```makefile
VCS_BIN = bash ${VCS_HOME}/bin/vcs
```

这样 Makefile 中的 VCS 调用方式就和终端 alias 保持一致。

如果希望默认流程就生成覆盖率数据，可以把 `${CM}`、`${CM_NAME}`、`${CM_DIR}` 放进编译命令和仿真命令。
这样执行 `make compile` 和 `make sim` 后，就会生成 `${OUTPUT}.vdb` 覆盖率数据库。

#### SIM 仿真命令

```makefile
SIM = ./${OUTPUT} -l sim.log
```

如果 `OUTPUT = simv`，展开后就是：

```text
./simv -l sim.log
```

含义：

```text
运行 VCS 生成的仿真可执行文件
把仿真运行日志写入 sim.log
```

编译日志和仿真日志要区分：

```text
com.log    VCS 编译日志
sim.log    仿真运行日志，包含 testbench 的 $display 输出
```

#### target

Makefile 的基本语法是：

```makefile
目标名:
	命令
```

例如：

```makefile
compile:
	${VCS}

sim:
	${SIM}
```

执行：

```text
make compile
```

就是运行 `${VCS}` 编译命令。

执行：

```text
make sim
```

就是运行 `${SIM}` 仿真命令。

注意：target 下面的命令前面必须是 Tab，不是普通空格。

#### debug

```makefile
WAVE = top.vcd

debug:
	dve -full64 -vpd ${WAVE} &
```

作用：

```text
打开 DVE 图形界面查看波形
```

`&` 是 Linux shell 的后台运行符号。DVE 是 GUI 程序，加 `&` 后终端不会被 DVE 占住。

需要注意：

```text
debug 目标中的波形文件名要和实际生成的波形文件一致
如果 testbench 中写的是 $dumpfile("top.vcd")，Makefile 中就可以写 WAVE = top.vcd
如果实际生成的是 top.vpd，Makefile 中就可以写 WAVE = top.vpd
```

#### clean

```makefile
clean:
	rm -rf ./csrc *.daidir *.log *.vpd *.vdb simv* *.key *.race.out*
```

作用：

```text
清理 VCS/DVE 自动生成的中间文件、日志、波形和覆盖率数据库
```

常见清理对象：

```text
csrc/          VCS 编译中间文件
*.daidir       VCS 仿真数据库目录
*.log          编译/仿真日志
*.vpd          VPD 波形文件
*.vdb          覆盖率数据库
simv*          仿真可执行文件及相关文件
*.key          VCS/DVE 交互记录
```

`rm -rf` 比较危险，clean 里只应该写工具自动生成的文件，不应写 RTL、testbench、filelist 或 Makefile。

### 4.5 VCS 代码覆盖率

覆盖率用于回答：

```text
testbench 到底覆盖了多少代码？
哪些行没执行？
哪些分支没走到？
哪些信号没有翻转？
```

在 VCS 中，常用覆盖率选项是：

```makefile
CM = -cm line+cond+fsm+branch+tgl
CM_NAME = -cm_name ${OUTPUT}
CM_DIR = -cm_dir ./${OUTPUT}.vdb
```

#### -cm

`cm` 可以理解为 `coverage metrics`，即覆盖率指标。

```text
line      行覆盖率
cond      条件覆盖率
fsm       状态机覆盖率
branch    分支覆盖率
tgl       toggle 翻转覆盖率
```

#### -cm_name

```makefile
CM_NAME = -cm_name ${OUTPUT}
```

作用：

```text
给本次覆盖率 run 起一个名字
```

它不会生成一个外部文件，也不会覆盖仿真可执行文件。

例如：

```text
-cm_name simv
```

只是在覆盖率数据库内部把本次运行命名为 `simv`。

#### -cm_dir

```makefile
CM_DIR = -cm_dir ./${OUTPUT}.vdb
```

作用：

```text
指定覆盖率数据库保存目录
```

如果 `OUTPUT = simv`，会生成：

```text
simv.vdb/
```

`.vdb` 通常是一个目录，不是普通文本文件。

#### 为什么编译和仿真都要加覆盖率选项

覆盖率需要两个动作：

```text
编译时插桩
运行时收集
```

因此编译命令中要加：

```makefile
VCS = vcs ... \
	${CM} \
	${CM_NAME} \
	${CM_DIR} \
	...
```

表示：

```text
生成带覆盖率采集能力的 simv
```

仿真命令中也要加：

```makefile
SIM = ./${OUTPUT} \
	${CM} \
	${CM_NAME} \
	${CM_DIR} \
	-l sim.log
```

表示：

```text
运行仿真时把覆盖率结果写入 vdb
```

所以：

```text
simv        是仿真可执行文件
simv.vdb/   是覆盖率数据库
top.vcd     是波形文件
```

三者不是同一种东西。

#### cov 目标

```makefile
cov:
	dve -full64 -covdir ${OUTPUT}.vdb &
```

作用：

```text
打开 DVE 覆盖率界面，查看 vdb 数据库
```

覆盖率完整运行流程：

```text
make clean
make compile
make sim
make cov
```

#### 自定义覆盖率可视化

DVE 可以查看覆盖率，但图形界面不一定适合截图或写报告。
如果只是想快速得到一个更清楚的覆盖率摘要，可以从 `.vdb` 中读取覆盖率运行数据，生成自己的 Markdown 表格或 SVG 图。

需要先明确一点：

```text
simv.vdb 不是普通文本文件
它是 Synopsys 的覆盖率数据库目录
正式覆盖率报告应优先使用 DVE 或 URG
```

但在 VCS 生成的 `.vdb` 目录里，通常可以找到一部分结构化运行数据，例如：

```text
simv.vdb/snps/coverage/db/testdata/simv/
├── line.verilog.data.xml
├── branch.verilog.data.xml
├── tgl.verilog.data.xml
├── cond.verilog.data.xml
└── fsm.verilog.data.xml
```

这些文件虽然扩展名是 `.xml`，但很多实际是 gzip 压缩后的 XML。
解压后可以看到类似内容：

```xml
<instance_data name="top_tb" value="101111111" />
<instance_data name="top_tb.u_top" value="111111101" />
```

其中：

```text
name     当前覆盖率数据对应的实例层级
value    一串 0/1 覆盖率命中位
1        该覆盖点命中
0        该覆盖点未命中
```

因此可以做一个轻量统计：

```text
命中数 = value 中 1 的数量
总数   = value 字符串长度
覆盖率 = 命中数 / 总数
```

例如：

```text
value = 111011
命中数 = 5
总数 = 6
覆盖率 = 83.33%
```

在 Makefile 中可以增加一个自定义目标：

```makefile
COV_MD = coverage_summary.md
COV_SVG = coverage_summary.svg

cov_plot:
	python3 scripts/vdb_cov_summary.py ${OUTPUT}.vdb ${COV_MD} ${COV_SVG}
```

运行：

```text
make compile
make sim
make cov_plot
```

可以生成：

```text
coverage_summary.md    覆盖率表格
coverage_summary.svg   覆盖率柱状图
```

这种方法适合：

```text
快速查看覆盖率趋势
生成课程报告截图
对 line/branch/toggle 等基础指标做直观展示
```

但它不是完整的覆盖率签核方法，因为它没有完整处理：

```text
exclude 规则
覆盖率合并
复杂层级权重
工具内部的完整覆盖率语义
```

所以更准确的关系是：

```text
DVE/URG       官方覆盖率分析
自定义脚本    轻量可视化和学习辅助
```

### 4.6 testbench 的作用

testbench 是仿真环境，不是硬件本体。

它通常负责：

```text
产生时钟
产生复位
给输入激励
等待输出稳定
比较输出和期望值
打印 PASS/FAIL
dump 波形
```

自检 testbench 比只看波形更可靠，因为它能直接告诉你哪些 case 失败。

### 4.7 在 testbench 中生成 VCD 波形

VCD 的全称是：

```text
Value Change Dump
```

中文可以理解为：

```text
信号值变化记录文件
```

VCD 记录的是仿真过程中信号随时间的变化，比如：

```text
clk 什么时候翻转
rst 什么时候释放
输入什么时候变化
输出什么时候更新
```

在 testbench 中常见写法：

```verilog
initial begin
    $dumpfile("top.vcd");
    $dumpvars(0, top_tb);
end
```

这段必须写在 testbench 中，通常写在一个 `initial begin ... end` 里。

#### initial begin 的含义

```verilog
initial begin
    ...
end
```

表示：

```text
仿真开始时执行一次
```

它常用于 testbench 中：

```text
初始化信号
产生测试激励
设置波形 dump
控制仿真结束
```

注意：`initial` 一般用于仿真环境，不是综合电路的主体写法。

#### $dumpfile

```verilog
$dumpfile("top.vcd");
```

作用：

```text
指定 VCD 波形文件名
```

#### $dumpvars

```verilog
$dumpvars(0, top_tb);
```

作用：

```text
指定要记录哪些层级的信号
```

其中：

```text
0       表示递归记录 top_tb 下面所有层级的信号
top_tb  表示从 testbench 顶层开始 dump
```

如果写：

```verilog
$dumpvars(1, top_tb);
```

通常表示只记录 `top_tb` 当前层，不继续深入子模块。

所以：

```verilog
$dumpvars(0, top_tb);
```

适合调试，因为它会记录 testbench 和 DUT 内部信号。

代价是：

```text
波形文件更大
仿真可能略慢
```

#### dump 的意思

`dump` 在仿真里可以理解为：

```text
导出 / 记录 / 转储
```

`dump waveform` 就是把仿真内部的信号变化保存到外部波形文件中。

### 4.8 VCD、VPD、VDB 的区别

```text
VCD = Value Change Dump
VPD = VCS/Vera Proprietary Dump
VDB = Verification Database
```

可以这样理解：

```text
VCD
    通用文本波形格式，记录信号值变化，很多工具都能打开，但文件可能较大。

VPD
    Synopsys 常用的波形格式，常用于 VCS/DVE，查看效率通常比 VCD 更好。

VDB
    覆盖率数据库，保存 line/cond/branch/fsm/toggle 等覆盖率结果。
```

如果用 DVE 打开 VCD，有时工具会自动生成：

```text
top.vcd.vpd
```

它可以理解为：

```text
由 top.vcd 转换或缓存得到的 VPD 波形文件
```

这个文件不是源码，也不是核心设计文件，可以删除。下次 DVE 打开 VCD 时可能重新生成。

### 4.9 前仿真输出

典型输出：

```text
com.log        编译日志
sim.log        仿真运行日志
simv           VCS 生成的仿真可执行文件
csrc/          VCS 编译中间文件
simv.daidir/   VCS 仿真数据库
*.vcd/*.vpd    波形文件
*.vdb          覆盖率数据库
DVEfiles/      DVE 图形界面会话和日志目录
```

其中真正需要关注的是：

```text
sim.log 里 PASS/FAIL 是否正确
波形中关键信号是否符合预期
覆盖率是否达到预期
```

其中：

```text
csrc/
    VCS 默认生成的编译中间目录，不需要在 Makefile 中显式指定。

simv.daidir/
    VCS 默认生成的仿真数据库目录。

DVEfiles/
    DVE 图形界面自动生成的工作目录，保存 GUI 日志、历史命令、session 等。
```

这些都属于工具生成物，可以通过 clean 清理；真正需要长期保留的是源码、testbench、filelist、Makefile 和必要的报告/截图。

## 5. DC：逻辑综合

### 5.1 DC 在整个流程中的位置

DC 的任务是把 RTL 变成标准单元门级网表。

从书本知识看，它完成的是：

```text
行为级/RTL 描述
  -> 组合逻辑和时序逻辑结构
  -> 布尔逻辑优化
  -> 工艺库标准单元映射
```

RTL 中的：

```verilog
always @(posedge clk)
if (en) q <= d;
```

综合后会变成：

```text
触发器 + 组合逻辑 + 标准单元连接
```

### 5.2 DC 主要输入

```text
RTL 设计文件
标准单元 .db 逻辑库
SDC 约束
库路径和工具 setup 脚本
可选：物理库、TLUPlus、tech file，用于 topographical 综合
```

### 5.3 DC 典型目录

```text
dc/
├── scr/
│   ├── MPW180_lib_list.tcl
│   ├── common_setup.tcl
│   ├── dc_setup.tcl
│   ├── sdc.tcl
│   └── dc_run.tcl
├── syn/
├── work/
├── alib/
├── mw/
├── out/
└── rep/
```

建议理解：

```text
scr/   放脚本
syn/   运行 dc_shell 的工作目录
work/  放 analyze 后的 WORK 设计工作库文件
alib/  放 DC 对标准单元库分析后的 ALIB 缓存
mw/    放当前设计的 Milkyway design library
out/   放给 ICC/PT/后仿使用的正式输出
rep/   放 timing/area/power 等报告
```

### 5.4 DC 启动链和脚本职责

DC 流程不要只看 `dc_run.tcl`。很多变量和别名是在工具启动时由 `.synopsys_dc.setup` 自动读入的。

如果在 `dc/syn/` 目录启动 DC，常见执行链是：

```text
dc_shell / dc_shell -topo
  -> 自动读取 .synopsys_dc.setup
       -> file mkdir ../alib
       -> file mkdir ../work
       -> set_app_var alib_library_analysis_path ../alib
       -> define_design_lib WORK
       -> source MPW180_lib_list.tcl
       -> source common_setup.tcl
       -> source dc_setup.tcl
  -> 手动或脚本 source ../scr/dc_run.tcl
```

各文件职责可以这样分：

```text
.synopsys_dc.setup
    DC 启动环境。
    定义 alias、ALIB 缓存目录、WORK 工作库、自动 source setup 文件。

MPW180_lib_list.tcl
    工艺库路径清单。
    只回答“库在哪里、库文件叫什么”。

common_setup.tcl
    当前工程配置。
    选择顶层设计、RTL 搜索路径、目标 .db、Milkyway/topo 相关变量。

dc_setup.tcl
    把 common_setup.tcl 里整理好的变量真正设置给 DC。
    包括 search_path、target_library、link_library、Milkyway、TLUPlus。

sdc.tcl
    设计约束。
    描述时钟、输入输出时序、复位例外、驱动和负载。

dc_run.tcl
    主流程脚本。
    读 RTL、展开设计、link、读约束、综合、报告、输出。
```

这里有两类“库”容易混淆：

```text
WORK design library
    当前设计工作库，由 define_design_lib 创建。
    analyze 后的 RTL 中间结果会放进去。

ALIB cache
    标准单元 .db 库分析缓存，由 alib_library_analysis_path 指定。
    主要用于加快后续库读取，不是综合交付物。

target/link library
    工艺标准单元 .db 库。
    DC 用它把 RTL 映射成真实标准单元。
```

例如：

```tcl
file mkdir ../work
define_design_lib WORK -path ../work
analyze -format verilog -lib work {top.v}
```

表示：

```text
先定义当前设计工作库 WORK
再把 top.v 分析进 work 库
```

把 WORK 和 ALIB 单独放到 `dc/work/`、`dc/alib/` 的主要目的是避免污染 `dc/syn/` 运行目录，让中间文件、缓存、报告和正式输出职责分离。

### 5.5 工艺库和 DC 库设置

`MPW180_lib_list.tcl` 是库路径索引表，通常定义：

```text
标准单元库根目录
.db 逻辑库路径
symbol 库路径
tech file 路径
TLUPlus 路径
Milkyway reference library 路径
```

例如：

```tcl
set MPW180 "/home/wzs/SMIC/smic180_lib_list"
set MPW180_path "$MPW180/std"
set MPW180_dbpath "$MPW180_path/liberty"
set MPW180_db_ss_125c "$MPW180_dbpath/scc018ug_hd_rvt_ss_v1p62_125c_basic.db"
```

这样写的目的：

```text
库位置改变时，只改根路径
脚本其它地方继续使用变量
避免到处硬编码绝对路径
```

`common_setup.tcl` 则从这些库变量里选择当前设计要用的部分：

```tcl
set DESIGN_NAME top
set ADDITIONAL_SEARCH_PATH "... ../../rtl/design"
set TARGET_LIBRARY_FILES "$MPW180_db_ss_125c"
set SYMBOL_LIBRARY_FILES "$MPW180_sdb"
```

然后 `dc_setup.tcl` 把这些普通 Tcl 变量设置给 DC 内置变量：

```tcl
set search_path "$ADDITIONAL_SEARCH_PATH"
set target_library "$TARGET_LIBRARY_FILES"
set link_library "$TARGET_LIBRARY_FILES"
set symbol_library "$SYMBOL_LIBRARY_FILES"
```

含义：

```text
search_path
    DC 查找 RTL、脚本和库文件的路径。

target_library
    综合映射目标库。DC 只能把 RTL 映射成这里允许使用的标准单元。

link_library
    解析引用使用的库。用于 link 当前设计、子模块和标准单元。
    更完整的工程里常见写法是 "* $target_library"，其中 * 表示 DC memory 中已有设计。

symbol_library
    图形界面画原理图符号用，对综合结果本身不关键。
```

### 5.6 Topographical 设置、Milkyway 和 TLUPlus

普通 DC 逻辑综合主要依赖：

```text
RTL
.db 逻辑库
SDC 约束
```

但如果使用：

```text
dc_shell -topo
```

就进入 DC Topographical / 物理感知综合流程。它会提前使用部分后端物理信息，让综合阶段的线延迟估计更接近布局布线结果。

这时 `common_setup.tcl` 里会定义：

```tcl
set MW_DESIGN_LIB ../mw/${DESIGN_NAME}_LIB
set MW_REFERENCE_LIB_DIRS $MPW180_mw
set TECH_FILE "$MPW180_tfpath/scc018u_hd_6m_1tma1.tf"

set TLUPLUS_MAX_FILE $MPW180_TLU_CMAX
set TLUPLUS_MIN_FILE $MPW180_TLU_CMIN
set TLUPLUS_TYP_FILE $MPW180_TLU_TYP
set MAP_FILE $MPW180_map
```

这些文件的职责：

```text
Milkyway reference library
    标准单元物理参考库，描述 cell 尺寸、pin 位置、blockage 等。

Milkyway design library
    当前设计自己的物理数据库，保存本设计的物理信息。

technology file
    工艺技术文件，描述金属层、via、单位、site、物理规则等。

TLUPlus
    金属互连 RC 查找表，用于估算线电阻、电容和线延迟。

TLUPlus map file
    把 technology file 中的层名映射到 TLUPlus 文件中的层名。
```

`dc_setup.tcl` 中一般先创建/打开 Milkyway design library：

```tcl
set mw_reference_library "$MW_REFERENCE_LIB_DIRS"
set mw_design_library "$MW_DESIGN_LIB"

file mkdir [file dirname $mw_design_library]

if {![file isdirectory $mw_design_library]} {
    create_mw_lib -technology $TECH_FILE \
                  -mw_reference_library $mw_reference_library \
                  $mw_design_library
}

open_mw_lib $mw_design_library
```

这里把 Milkyway design library 放到 `dc/mw/`，是为了避免在 `dc/syn/` 运行目录下生成 `*_LIB` 这类物理数据库目录。

这里不是把标准单元库复制一份，而是创建当前设计自己的 Milkyway 数据库，并绑定：

```text
tech file
    工艺层和物理规则

Milkyway reference library
    可用标准单元的物理形状和 pin 信息
```

打开 Milkyway design library 后，再设置 TLUPlus：

```tcl
set_tlu_plus_files -max_tluplus $TLUPLUS_MAX_FILE \
                   -min_tluplus $TLUPLUS_MIN_FILE \
                   -tech2itf_map $MAP_FILE
```

然后可以检查：

```tcl
check_tlu_plus_files
```

注意：

```text
create_mw_lib 不使用 TLUPlus
TLUPlus 是打开 Milkyway design library 后设置的 RC 模型
typ TLUPlus 常作为典型角保留，很多基础 set_tlu_plus_files 流程主要使用 max/min
```

如果实际只运行普通 `dc_shell`，没有使用 `dc_shell -topo`，这些 Milkyway/TLUPlus 设置对主综合结果不是核心；它们更像教学模板里保留的物理感知综合配置。

### 5.7 普通 DC 的导线负载模型

如果不用 topo 模式，DC 没有真实 placement/routing 信息，线延迟通常通过 wire load model 估算。

导线负载模型常见命令：

```tcl
set_wire_load_model -name <wire_load_model_name> -library <library_name>
set_wire_load_mode top
```

含义：

```text
set_wire_load_model
    指定使用哪个 wire load model 来估算连线电容和连线延迟。

set_wire_load_mode top
    层级设计统一使用顶层 wire load model。
```

这类设置通常放在：

```text
dc_setup.tcl
    作为库和工具环境的一部分

或 dc_run.tcl 中 source sdc.tcl 之前
    作为综合前的环境设置
```

示例写法：

```tcl
# 普通 DC 非 topo 流程中使用，具体 model 名称需要从 .lib/.db 中确认。
set_wire_load_model -name smic18_wl10 -library scc018ug_hd_rvt_ss_v1p62_125c_basic
set_wire_load_mode top
```

如果使用 topo：

```text
不要再重点依赖 wire load model
应使用 Milkyway + tech file + TLUPlus 进行物理感知线延迟估算
```

所以两种思路是：

```text
普通 DC
    .db + wire load model

DC topo
    .db + Milkyway + tech file + TLUPlus
```

### 5.8 SDC 约束的书写逻辑

SDC = Synopsys Design Constraints。它不是附属文件，而是综合和 STA 的核心输入。

书本上的静态时序分析本质是：

```text
数据到达时间 <= 数据要求时间
```

DC 要完成这个判断，至少需要知道：

```text
时钟周期是多少？
时钟有多少不确定性？
输入信号什么时候到达模块？
输出信号要给外部留下多少时间？
输入由多强的外部单元驱动？
输出后面接了多大负载？
哪些路径不应该按普通数据路径检查？
```

#### 时钟约束

最核心的是：

```tcl
set per 20.00
create_clock -period $per [get_ports clk]
```

如果目标是 200MHz：

```text
period = 1000 / 200 = 5ns
```

所以写：

```tcl
set per 5.00
create_clock -period $per [get_ports clk]
```

如果希望 DC 阶段给后端留余量，也可以把 DC 目标设得更紧，比如后端目标 200MHz，DC 先按 220MHz 或 240MHz 综合。

#### 时钟非理想因素

教学脚本常见：

```tcl
set_clock_latency -source -max [expr {$per*0.1}] $CLOCK1
set_clock_latency -max [expr {$per*0.1}] $CLOCK1
set_clock_uncertainty -setup [expr {$per*0.05+0.04+0.03+0.03}] $CLOCK1
set_clock_transition -max [expr {$per*0.08}] $CLOCK1
```

含义：

```text
set_clock_latency -source
    时钟源到设计 clk 端口之前的延迟，表示设计外部时钟延迟。

set_clock_latency
    设计 clk 端口到内部寄存器 clock pin 的时钟网络延迟。

set_clock_uncertainty
    setup 检查预留余量，覆盖 skew、jitter、margin 等不确定因素。

set_clock_transition
    时钟边沿转换时间，影响 cell delay 和时序分析。
```

这些值应该来自实际时钟预算、工艺建议、后端 CTS 目标或工程经验。
教学模板按比例设置可以帮助理解命令，但不代表真实工程约束一定合理。

#### 路径分组

```tcl
group_path -name INPUT -from [all_inputs]
group_path -name OUTPUT -to [all_outputs]
group_path -name INOUT -from [all_inputs] -to [all_outputs]
```

它们用于让报告和优化更清晰：

```text
INPUT
    从输入端口出发的路径，例如 input -> register。

OUTPUT
    到输出端口结束的路径，例如 register -> output。

INOUT
    输入端口直接经过组合逻辑到输出端口的路径，即 input -> output。
    这里的 INOUT 不是 Verilog 的 inout 端口。
```

#### 输入延迟和输出延迟

输入延迟：

```tcl
set all_in_exp_clk [remove_from_collection [all_inputs] [get_ports "clk rst"]]
set_input_delay -max [expr {$per*0.4}] -clock $CLOCK1 $all_in_exp_clk
```

含义：

```text
set_input_delay 描述输入信号“什么时候到达模块输入端口”。
它表示外部电路已经消耗的时间。
```

输出延迟：

```tcl
set_output_delay -max [expr {$per*0.4}] -clock $CLOCK1 [all_outputs]
```

含义：

```text
set_output_delay 描述输出信号要给外部接收电路预留多少时间。
```

注意要排除 `clk`：

```text
clk 是时钟端口，不是普通数据输入
不能对 clk 设置普通 input delay
```

复位也常常要单独处理：

```text
rst/rst_n 通常是异步控制信号
不应简单当作普通同步数据输入来约束
```

#### 复位路径例外

例如：

```tcl
set_false_path -from [get_ports rst]
```

含义：

```text
不要把 rst 当成普通数据路径做 setup/hold 分析。
```

这不是说复位不重要，而是复位通常有自己的时序要求，例如 recovery/removal。
课程流程里常先把异步 reset 从普通数据路径中排除，避免 DC 把 reset 路径当作普通输入数据路径优化。

#### 输入驱动和输出负载

```tcl
set_driving_cell -lib_cell INHDV1 -pin ZN -library scc018ug_hd_rvt_ss_v1p62_125c_basic \
     $all_in_exp_clk
set_load 1 [all_outputs]
```

`set_input_delay` 描述“输入什么时候到”，而 `set_driving_cell` 描述“输入由谁驱动、边沿多快”。

两者区别：

```text
set_input_delay
    时序预算约束，影响 input port 到寄存器路径剩余可用时间。

set_driving_cell
    电气环境约束，影响输入 slew，进而影响第一级门和后续逻辑的 delay 估算。

set_load
    电气环境约束，描述输出端后面接了多大电容负载。
```

对 input -> register 路径，可以近似理解为：

```text
input_delay
+ 输入端口到寄存器 D 端的内部组合逻辑延迟
+ setup time
<= clock period - uncertainty
```

`set_driving_cell` 会影响“输入端口到寄存器 D 端的内部组合逻辑延迟”，尤其是第一级 cell 的 delay。

#### SDC 书写检查表

写 SDC 时建议按这个顺序检查：

```text
1. create_clock 是否覆盖所有真实时钟？
2. clk 是否从普通 input delay 中排除？
3. rst/rst_n 是否按复位信号处理，而不是普通数据输入？
4. input_delay/output_delay 是否符合上下游接口预算？
5. clock_uncertainty/transition 是否有合理来源？
6. set_driving_cell 和 set_load 是否能反映边界电气环境？
7. 是否存在 input -> output 组合路径，需要单独关注？
8. check_timing 是否还有 unconstrained path？
```

### 5.9 dc_run.tcl 的执行逻辑

DC 主流程可以按下面理解：

```text
remove_design
set_svf
analyze
elaborate
link
check_design
write unmap ddc
source sdc.tcl
check_timing
compile_ultra
report
write outputs
```

下面逐步解释。

理解该脚本时，可以先忽略具体参数，把每一段看成一个明确的物理状态转换：

| 脚本阶段 | 输入状态 | 核心变化 | 典型检查/产物 |
|---|---|---|---|
| 工作副本 | `data_setup` | 复制出可修改的`floorplan` CEL | 保留干净初始化起点 |
| Floorplan | 无芯片边界 | 建立core、row、利用率和IO边界 | `block_shape` CEL |
| 物理准备 | 已有row | 限制布线层，加入end-cap和tap cell | 工艺基础结构完整 |
| Power Planning | 无供电几何 | 建立ring、strap和标准单元电源轨 | `verify_pg_nets`、`ffinish` CEL |
| Placement | 单元无坐标 | 放置、合法化并优化标准单元 | 拥塞/时序报告、`placemented` CEL |
| CTS | 理想时钟 | 插入时钟缓冲器并建立真实时钟树 | clock tree、latency、skew报告 |
| Post-CTS | 时钟树已建立 | 根据真实时钟到达时间修复hold等问题 | 更新后的时序状态 |
| Routing | 只有连接关系 | 生成时钟和信号的金属线、via | 布线后时序/功耗/约束报告 |
| Post-Route | 已有真实走线 | 修复DRC和hold，检查开路、antenna、LVS | route、PG、LVS检查结果 |
| Chip Finish | 标准单元间有空隙 | 插入filler并保存最终CEL | `finish`/`allfinish` CEL |
| Handoff | 最终物理数据库 | 导出GDS、网表、SDC、SDF和SPEF | 交给PT、后仿和版图交付 |

脚本之所以较长，是因为它不只是连续调用几个优化命令。每个阶段都应形成闭环：

```text
继承上一阶段检查点
→ 设置本阶段规则
→ 执行物理变换
→ 报告时序/拥塞/DRC/PG等结果
→ 保存新的Milkyway CEL
→ 下一阶段继续
```

阶段检查点的价值是可回退、可比较、可重复。例如floorplan参数不合理时，可以重新从`data_setup`复制，而不必重新导入网表和SDC。

#### remove_design

```tcl
remove_design -design
```

清空当前 DC 内存中的旧设计。

为什么要做：

```text
避免上一次综合留下的设计污染当前运行
```

#### set_svf

```tcl
set_svf ../out/${DESIGN_NAME}.svf
```

SVF 用于形式验证，记录综合前后等价验证需要的信息。
把它输出到 `../out/`，可以和 mapped.ddc、mapped.v、mapped.sdc 等综合产物统一管理。

学习阶段可以知道：

```text
它不是给 ICC 的主输入
主要服务于 formal verification
```

#### analyze

```tcl
analyze -format verilog -lib work { top.v }
```

作用：

```text
读取 HDL 源文件
检查语法
把设计单元存入 work 库
```

注意：`analyze` 只是分析 RTL，还没有真正建立顶层设计。

#### elaborate

```tcl
elaborate $DESIGN_NAME
```

作用：

```text
根据顶层名展开设计层级
解析 parameter/generate
建立寄存器、组合逻辑、模块例化关系
形成 DC 内部的设计结构
```

从 Verilog 语言角度看，`elaborate` 就是把代码描述变成工具可操作的电路层级。

#### link

```tcl
link
```

作用：

```text
解析当前工程引用的子模块
解析库单元
检查有没有 unresolved reference
```

如果顶层里例化了某个模块，但 filelist 没读这个模块，`link` 就会报错。

#### check_design

```tcl
check_design
```

作用：

```text
检查设计结构问题
例如未连接端口、多驱动、组合环、未解析引用等
```

它是综合前的体检。

#### write unmap ddc

```tcl
write -hierarchy -f ddc -out ../out/${DESIGN_NAME}_unmap.ddc
```

保存未综合映射前的 DC 数据库。

作用：

```text
保留 elaboration 后的设计状态
方便调试或重新进入 DC 查看
```

#### source sdc.tcl

```tcl
source sdc.tcl
```

加载约束。

没有约束时，工具不知道应该优化到什么频率，也不知道输入输出环境。

#### check_timing

```tcl
check_timing
```

作用：

```text
检查时钟是否定义
检查输入输出是否约束
检查是否有 unconstrained path
检查 timing setup 是否完整
```

它不是看最终时序好坏，而是看时序分析环境是否完整。

#### compile_ultra

```tcl
compile_ultra
```

这是 DC 综合优化的核心命令。

它做的事包括：

```text
逻辑优化
门级映射
时序优化
面积优化
必要时复制逻辑、调整门大小、优化关键路径
```

综合完成后，RTL 已经变成由标准单元组成的门级网表。

#### report

```tcl
report_constraint -significant_digits 4 -all > ../rep/rep_constraints
report_timing > ../rep/${DESIGN_NAME}_timing
report_area   > ../rep/${DESIGN_NAME}_area
report_power  > ../rep/${DESIGN_NAME}_power
```

报告的作用：

```text
constraint 看整体约束违例，例如 setup、transition、capacitance、fanout 等
timing     看具体关键路径、data arrival time、required time 和 slack
area       看标准单元面积
power      看功耗估计
```

`report_constraint` 和 `report_timing` 的侧重点不同：

```text
report_constraint
    像总检查表，回答“当前设计违反了哪些约束？”

report_timing
    像路径分析表，回答“最差路径从哪里到哪里，为什么 slack 是这个值？”
```

初学者重点看：

```text
slack 是否为正
关键路径从哪里到哪里
面积是否异常
有没有 unconstrained path
有没有 max_transition / max_capacitance 这类设计规则违例
```

#### write outputs

```tcl
write -hierarchy -format ddc -output ../out/$DESIGN_NAME.mapped.ddc
write -hierarchy -format verilog -output ../out/$DESIGN_NAME.mapped.v
write_sdc ../out/$DESIGN_NAME.mapped.sdc
write_sdf ../out/$DESIGN_NAME.mapped.sdf
```

这些文件是 DC 交给后续流程的结果。

## 6. DC 输出文件详解

### 6.1 mapped.v

综合后的门级网表。

它描述：

```text
用了哪些标准单元
这些标准单元之间怎么连接
寄存器和组合逻辑如何组成电路
```

ICC 会读它进行布局布线。

### 6.2 mapped.ddc

DC 内部数据库格式。

作用：

```text
保存 DC 里的设计、约束、综合状态
比 Verilog 网表保留更多工具内部信息
可用于 DC/ICC 继续读取
```

可以理解为 DC 的工程存档。

### 6.3 mapped.sdc

综合后的约束文件。

ICC 会继续使用它，因为后端也需要知道：

```text
时钟周期
输入输出延迟
false path
时钟不确定度
```

### 6.4 mapped.sdf

综合阶段估计延迟文件。

一般可用于门级仿真，但不如后布局 SDF 准确。

原因：

```text
DC 阶段还没有真实布局布线
互连延迟只是估计
```

### 6.5 timing / area / power report

这些报告不是给工具继续读的，而是给人检查结果用的。

重点：

```text
timing：看关键路径和 slack
area：看标准单元面积
power：看动态功耗和静态功耗估计
```

## 7. ICC：布局布线

### 7.1 ICC 在整个流程中的位置

DC 只知道门和连接关系，ICC 要把这些门真正放到芯片版图中。

ICC 做的是物理实现：

```text
确定芯片区域
放置标准单元
建立电源网络
建立时钟树
完成信号布线
提取寄生参数
导出版图和后仿文件
```

从书本知识看，它把逻辑网表变成物理版图。

### 7.2 ICC 的两类 Milkyway 库

Milkyway 这个词容易混淆，因为有两类。

#### Milkyway reference library

这是工艺库/标准单元物理参考库，通常由库厂或 foundry 提供。

它是只读的，包含：

```text
标准单元尺寸
pin 位置
routing blockage
metal layer 信息
物理抽象
```

脚本中常见变量：

```tcl
set MW_REFERENCE_LIB_DIRS ...
```

#### Milkyway design library

这是当前设计的物理数据库，由 ICC 创建和修改。

脚本中常见变量：

```tcl
set MW_DESIGN_LIB ${DESIGN_NAME}_LIB
```

它保存脚本执行后的物理结果：

```text
floorplan
power ring/strap
placement
CTS
routing
不同阶段保存的 mw cell
```

可以理解为：

```text
reference library = 标准单元物理参考库，只读
design library    = 当前设计实现结果，可写
```

ICC 工作时：

```text
读 reference library
写 design library
```

不是把你的设计结果写回 foundry 库。

### 7.3 ICC 典型目录

```text
icc/
├── rm_setup/
│   ├── common_setup.tcl
│   ├── icc_setup.tcl
│   └── lcrm_setup.tcl
├── scr/
│   ├── init_design_icc.tcl
│   ├── common_optimization_setting_icc.tcl
│   └── design_run.tcl
├── run/
├── work/
├── out/
├── rpts/
└── rep/
```

建议理解：

```text
rm_setup/   公共环境、库、输入文件变量
scr/        具体执行脚本
run/        启动 ICC 的目录
work/       工具运行工作区
out/        正式输出给 PT/后仿/版图检查
rpts/       timing/constraint/power 报告
rep/        PG/LVS 等检查报告
```

### 7.4 ICC common_setup.tcl

`common_setup.tcl` 不执行布局布线。它先把后续脚本需要的设计名、输入、库和工艺文件按用途分类保存为 Tcl 变量，再由 `icc_setup.tcl` 和初始化脚本真正应用。

它回答的是：

```text
顶层设计叫什么？
DC 输出在哪里？
逻辑库、物理库和工艺文件在哪里？
最大/最小延迟库如何配对？
电源网络、单元电源引脚和布线层如何命名？
```

#### 路径为什么要按用途分开

`ADDITIONAL_SEARCH_PATH` 不是“所有路径的总表”，它最终会加入 ICC 的 `search_path`，用于按文件名查找 `.db`、门级网表和 SDC：

```tcl
set ADDITIONAL_SEARCH_PATH [join "
    ${DESIGN_REF_PATH}/liberty
    ${ICC_INPUTS_PATH}
"]
```

`MW_REFERENCE_LIB_DIRS` 则会传给 `create_mw_lib -mw_reference_library`，明确指定当前 Milkyway design library 引用哪些物理库：

```tcl
set MW_REFERENCE_LIB_DIRS [join "
    ${DESIGN_REF_PATH}/milkyway
    ${IO_REF_PATH}/apollo/IO_LIBRARY
"]
```

二者都只是在 `common_setup.tcl` 中定义变量，但后续用途不同：

```text
ADDITIONAL_SEARCH_PATH
    → search_path
    → 回答“普通输入文件可能在哪里”

MW_REFERENCE_LIB_DIRS
    → create_mw_lib -mw_reference_library
    → 回答“当前物理设计引用哪些只读物理库”
```

把 Milkyway 目录加入 `search_path` 并不会自动建立物理参考关系。反过来，也不能把包含 Liberty 和 DC 输出目录的混合路径列表直接当成 Milkyway reference library。

#### 逻辑库的三个角色

ICC 与 DC 一样需要标准单元逻辑库，但 ICC 还要进行物理优化、插入缓冲器和 CTS，因此必须清楚区分：

```tcl
set TARGET_LIBRARY_FILES     "..."
set ADDITIONAL_LINK_LIB_FILES "..."
```

随后 `icc_setup.tcl` 会设置：

```tcl
set_app_var target_library "$TARGET_LIBRARY_FILES"
set_app_var link_library "* $TARGET_LIBRARY_FILES $ADDITIONAL_LINK_LIB_FILES"
```

含义如下：

```text
target_library
    ICC 优化时允许主动选择和插入的标准单元库

link_library
    解析门级网表中已有实例所需的全部逻辑库

ADDITIONAL_LINK_LIB_FILES
    SRAM、ROM、PLL、硬宏或第三方 IP 等只需链接的额外 .db
```

如果设计只有标准单元，没有额外存储器和硬宏，`ADDITIONAL_LINK_LIB_FILES` 为空是正常的，此时 `link_library` 仍然包含 `TARGET_LIBRARY_FILES`。

#### 最大与最小延迟库

后端必须同时关注 setup 和 hold：

```text
SS、低电压、高温慢角
    单元延时较大，主要用于 setup 分析

FF、高电压、低温快角
    单元延时较小，主要用于 hold 分析
```

旧版 RM 脚本通常通过 `MIN_LIBRARY_FILES` 保存“最大延迟库 最小延迟库”配对：

```tcl
set MIN_LIBRARY_FILES [join "
    slow_corner_basic.db
    fast_corner_basic.db
"]
```

`icc_setup.tcl` 再建立对应关系：

```tcl
set_min_library $max_library -min_version $min_library
```

变量名虽然是 `MIN_LIBRARY_FILES`，内容却不能只写一个快角库。每一对还应采用相同的模型类型，例如 `basic` 对 `basic`，不能把 `basic` 和 `ccs` 随意配成一对。

仅把 SS、FF 文件列入目标库，只表示工具能够找到这些库。完整 OCV/MCMM 流程还需要通过 operating condition 或 scenario 明确每种分析使用哪个逻辑角和 RC 角。

#### 电源命名与布线配置

```tcl
set MW_POWER_NET   "VDD"
set MW_POWER_PORT  "VDD"
set MW_GROUND_NET  "VSS"
set MW_GROUND_PORT "VSS"
```

`net` 是芯片内部的连接网络，`port/pin` 是设计边界或标准单元边界上的连接点。二者可以同名，也可以不同：

```text
单元电源引脚 VDD
        ↓
内部电源网络 VDD_CORE
        ↓
顶层电源端口或电源 PAD
```

路由层上下限用于约束普通信号可使用的金属层范围。变量本身不会自动生效，必须在后续脚本中传给相应的布线层设置命令。电源和时钟也可以通过各自命令使用不同的层范围。

多电压变量中的 power domain、voltage area 和 `VDD1/VDD2` 等只是 RM 模板预留项。电源域名称为空时，不会因为定义了这些字符串就自动生成多电压设计。

### 7.5 init_design_icc.tcl

这个脚本把前一节定义的配置真正应用到工具中，完成从“DC 文件”到“可进行物理实现的 Milkyway 设计”的转换。

学习这个脚本时，重点不是先记住每条 Tcl 命令，而是理解为什么DC网表还不能直接拿去布局布线。DC门级网表只保存逻辑结构，ICC初始化需要把其他来源的信息补齐：

| 信息来源 | 告诉ICC什么 | 初始化后的用途 |
|---|---|---|
| DC门级网表 | 使用了哪些标准单元，端口、实例和net如何连接 | 建立设计拓扑 |
| `.db`逻辑库 | 单元逻辑功能、cell delay、transition、capacitance和功耗 | 链接网表并计算单元时序 |
| Milkyway物理参考库 | 单元宽高、pin坐标、blockage和物理抽象 | 使标准单元成为可放置、可布线的物理对象 |
| Technology File | 金属层、via、布线方向和物理设计规则 | 规定版图实现必须遵守的工艺规则 |
| TLUPlus与Map | 不同布线条件下的单位长度R/C及层名对应关系 | 为placement估算和routing后提取互连RC准备模型 |
| SDC | 时钟、输入输出延时及其他时序约束 | 指导placement、CTS和routing优化 |
| 电源配置 | power net与标准单元PG pin的对应关系 | 为后续power planning建立逻辑连接基础 |

这里不存在第二次标准单元映射：

```text
DC已经决定“用哪些标准单元”
ICC初始化负责“让这些实例同时拥有逻辑、电气和物理含义”
ICC后续流程再决定“这些单元放在哪里、金属线怎样连接”
```

单元延迟来自`.db`逻辑库。Milkyway物理库不会提供cell delay；Technology File和TLUPlus主要服务物理规则及互连RC。初始化阶段可以在零互连模式下报告cell delay，但此时还没有任何真实走线延时。

从输入和输出看，这个脚本可以概括为：

```text
输入：门级网表 + 逻辑库 + 物理库 + 工艺规则 + RC模型 + SDC
处理：导入、关联、检查、建立物理实现上下文
输出：data_setup Milkyway CEL
```

这里的`data_setup`可以理解为“物理实现所需数据已经准备完成”的阶段名，不是“布局布线完成”。它只是说明后端开始前需要的数据和关系已经准备好。

整体顺序是：

```text
加载流程环境
→ 创建 Milkyway design library
→ 导入 DC 门级网表
→ 建立最大/最小 RC 模型
→ 连接电源和地
→ 读取并检查 SDC
→ 建立布局前时序参考结果
→ 处理 rst/clk 的 ideal network 属性
→ 保存 data_setup 检查点
```

#### 三类辅助脚本的分工

初始化脚本通常先加载：

```text
lcrm_setup.tcl
    Synopsys LCRM 运行框架，提供日志、目录、指标和 Lynx 兼容函数

icc_setup.tcl
    把 common_setup.tcl 中的变量设置给 search_path、target/link library 和 RM 流程变量

common_optimization_setting_icc.tcl
    设置延时计算算法、CPU 核数、dont_use、面积和功耗等后续优化策略
```

其中 LCRM 和通用优化脚本主要属于工具运行框架，不是 floorplan、CTS、routing 等设计步骤。通常理解其职责即可，不需要逐行修改。

#### 创建物理设计数据库

```tcl
create_mw_lib ${DESIGN_NAME}_LIB -open \
    -tech $TECH_FILE \
    -mw_reference_library $MW_REFERENCE_LIB_DIRS
```

这一步创建当前设计可写的 Milkyway design library，并绑定：

```text
TECH_FILE
    金属层、通孔、单位及物理设计规则

MW_REFERENCE_LIB_DIRS
    标准单元和 IO 的只读物理视图
```

#### 导入已经映射的门级网表

```tcl
read_verilog -top $DESIGN_NAME $ICC_IN_VERILOG_NETLIST_FILE
```

DC 输出网表已经把 RTL 映射为标准单元。ICC 的 `read_verilog` 不会重新综合或重新映射，而是把 Verilog 文本解析为内部数据库对象：

```text
顶层设计和端口
标准单元实例
网络
实例引脚之间的连接关系
```

门级网表只说明“用了什么单元、怎样连接”，不包含单元尺寸、pin 坐标、放置位置和金属走线。ICC 在导入时结合 `.db` 逻辑视图和 Milkyway 物理视图，建立后续布局布线可操作的设计对象。

`-top` 用于选择从哪个 Verilog 模块建立完整层次；`current_design` 则设置后续命令的当前操作上下文。二者语义不同，即使 `read_verilog -top` 通常已经选中顶层，显式执行 `current_design` 仍能使流程上下文更清楚。

```tcl
uniquify_fp_mw_cel
```

如果同一个层次化模块被实例化多次，该命令为每个实例建立独立的层次化物理 CEL，避免多个可编辑实例共享同一个 master。展平的小设计中它可能不产生变化，但通用层次化流程需要保留。

```tcl
save_mw_cel -as $DESIGN_NAME
```

`read_verilog` 先在工具内存中建立设计，`save_mw_cel` 再把当前状态写入 Milkyway design library，形成可重新打开的初始检查点。后续的 `data_setup`、`placement`、`CTS`、`routing` 也可以分别保存为不同 CEL。

#### 配置最大和最小互连 RC

```tcl
set_tlu_plus_file \
    -max_tluplus $TLUPLUS_MAX_FILE \
    -min_tluplus $TLUPLUS_MIN_FILE \
    -tech2itf $MAP_FILE
```

逻辑角描述标准单元本身的快慢，TLUPlus 角描述金属互连 RC 的大小。`MAP_FILE` 把 Technology File 中的层名映射到 TLUPlus 使用的层名。

这里必须区分“RC模型”和“具体线网RC”：

```text
TLUPlus RC模型
    工艺提供的查找表，说明不同金属层、线宽、线距下单位长度导线大约有多少R和C

具体线网RC
    某一条net根据实际长度、所用金属层、线宽、邻近导线和via数量计算出的寄生参数
```

执行 `set_tlu_plus_file` 只完成第一项。初始化阶段没有单元坐标和金属走线，工具还不知道某条net有多长、走哪一层、经过多少via，因此不可能完成真实RC提取。

RC信息会随物理实现逐渐具体：

```text
data_setup阶段
    只绑定TLUPlus和层映射文件，没有具体线网RC

placement阶段
    已有单元位置，可根据虚拟走线或线长模型估算RC

global/detailed routing阶段
    已确定走线层、路径、线宽和via，可得到更准确的RC

extract_rc阶段
    从完成的布线几何中提取寄生电阻和电容，并可写出SPEF供PT分析
```

```text
慢单元库 + 偏大 RC
    常用于更保守的 setup 检查

快单元库 + 偏小 RC
    常用于更保守的 hold 检查
```

#### 建立电源连接

```tcl
derive_pg_connection \
    -power_net $MW_POWER_NET -power_pin $MW_POWER_PORT \
    -ground_net $MW_GROUND_NET -ground_pin $MW_GROUND_PORT
```

该命令把标准单元的电源/地 pin 关联到芯片内部的 VDD/VSS 网络。它建立的是电源连接关系，后续 power planning 才会真正创建 ring、strap 和标准单元电源轨连接。

`check_mv_design` 用于检查电源网络、电源域和电源引脚的一致性；即使是单电压设计，也可以用它发现基本的 PG 连接问题。

#### 读取约束后的检查顺序

```tcl
read_sdc $ICC_IN_SDC_FILE
```

读取 SDC 后，不应马上假定约束全部有效。综合可能改变对象名称、删除逻辑或改变层次，因此应按以下顺序检查：

```text
check_timing
    时序分析环境是否完整，例如无时钟寄存器、未约束路径和组合环

report_timing_requirements
    是否存在 false path、multicycle、max/min delay 等路径级特殊要求

report_disable_timing
    哪些单元内部 pin-to-pin 时序弧被用户、常量传播或打断组合环关闭

report_case_analysis
    哪些模式控制端口或引脚被固定为 0/1

report_constraint
    哪些 setup/hold、transition、capacitance、fanout 等约束满足或违例

report_timing
    具体路径的 arrival time、required time、单元/线网延时和 slack
```

这里最容易混淆的是：

```text
timing requirement
    针对完整起点到终点路径的特殊规则

disable timing
    从时序图中关闭某个单元内部的时序弧
```

#### CTS前的时钟报告

```tcl
report_clock
report_clock -skew
```

`report_clock` 检查时钟名称、周期、波形、来源及 generated/propagated 属性。此时还没有构建真实时钟树，`report_clock -skew` 主要显示 SDC 中设置的 latency、uncertainty 等时钟网络约束，不是 CTS 后测得的实际物理 skew。

CTS 完成后应使用：

```tcl
report_clock_timing -type latency
report_clock_timing -type skew
report_clock_timing -type transition
```

#### 为什么先做零互连延时报告

```tcl
set_zero_interconnect_delay_mode true
report_constraint -all
report_timing
set_zero_interconnect_delay_mode false
```

开启零互连延时后，工具暂时令线网延时为 0，只观察标准单元延时和约束本身。这份结果是布局布线前的参考结果：

```text
零互连下已经违例
    逻辑级数或单元速度本身可能无法满足周期

零互连下满足、布线后违例
    新增问题更可能来自线长、拥塞、RC 或时钟树
```

报告完成后必须恢复为 `false`，否则后续物理优化会忽略互连延时，结果不可信。“时序基线”不是签核结果，只是用于阶段比较和定位问题的初始参考。

#### rst和clk为什么在不同时间取消ideal

复位和时钟都可能是高扇出网络，但实现策略不同：

```text
rst
    通常没有独立的树综合阶段
    初始化结束时取消 ideal
    让 placement 能检查扇出、插入缓冲并真实布线

clk
    有专门的 CTS 阶段
    placement 前仍按理想时钟处理
    到 CTS 开始前再 remove_ideal_network
    由 clock_opt 专门建立时钟缓冲树
```

`remove_ideal_network` 只取消理想网络属性，不会删除端口、网络或逻辑功能。

最后：

```tcl
save_mw_cel -as data_setup
```

`data_setup` 保存的是已经完成以下工作的物理设计起点：

```text
门级网表已导入ICC数据库
逻辑库和Milkyway物理参考库已关联
TLUPlus与层映射文件已绑定
标准单元电源pin与电源net的逻辑关系已建立
SDC已读取并完成基础约束检查
rst已准备交给后续物理优化真实处理
```

它明确不包含：

```text
floorplan结果
标准单元坐标
时钟树
信号金属走线
基于真实布线提取的寄生RC
```

下一阶段不会直接修改这个检查点，而是：

```tcl
copy_mw_cel -from data_setup -to floorplan
open_mw_cel floorplan
```

这样 `data_setup` 始终保留为干净的初始化起点。floorplan或后续步骤出现问题时，可以重新复制它继续运行，而不需要再次导入网表、绑定工艺模型和读取SDC。

### 7.6 design_run.tcl 主流程

ICC 主流程通常包括：

```text
floorplan
power planning
placement
CTS
post-CTS optimization
routing
focal optimization
verification
output
```

下面逐步解释。

#### floorplan

作用：

```text
定义 die/core 区域
定义 row
定义 IO 到 core 的距离
定义 pin 约束
给后续标准单元摆放提供物理边界
```

常见命令：

```tcl
initialize_rectilinear_block
create_floorplan
set_fp_pin_constraints
```

如果设计很小，core 过大会浪费面积；如果 core 过小，布线拥塞、时序和 DRC 可能变差。

#### floorplan后的物理准备

floorplan只建立可用区域，还需要补充工艺实现必须具备的基础结构：

```tcl
set_ignored_layer -max_routing_layer METAL4
add_end_cap ...
add_tap_cell_array ...
```

对应作用：

```text
routing layer限制
    规定普通信号允许使用的最高金属层，避免无计划占用高层资源

end-cap cell
    封闭标准单元row边界，满足well、implant和边界设计规则

tap cell
    周期性连接衬底/阱到VDD/VSS，降低闩锁风险并满足最大tap间距要求
```

这些cell不实现RTL逻辑，却是标准单元版图能够通过工艺检查的必要结构。

#### power planning

作用：

```text
创建 VDD/VSS ring
创建 power strap
连接标准单元电源地
检查电源网络
```

常见命令：

```tcl
create_rectilinear_rings
create_power_straps
derive_pg_connection
verify_pg_nets
preroute_standard_cells
```

电源网络要先做，因为标准单元摆放和布线都需要电源地基础。

这里还要区分两种连接：

```text
derive_pg_connection
    建立“哪个PG pin属于哪个power net”的逻辑关系

ring/strap/preroute_standard_cells
    在版图中真正生成电源金属几何并连接标准单元电源轨
```

`verify_pg_nets`检查的是物理电源网络连通性，比初始化阶段的`check_mv_design`更接近版图实现检查。

#### placement

作用：

```text
把标准单元放到 row 中
尽量优化线长、拥塞和时序
保证单元合法摆放
```

常见命令：

```tcl
create_placement
legalize_placement
place_opt
report_congestion
report_timing
```

placement 后已经可以看到标准单元大概位置，但信号线还没有完成详细布线。

此时工具可根据单元坐标和虚拟走线估算线长与RC，所以placement后的时序比`data_setup`零互连参考结果更接近真实情况，但仍不是最终布线时序。

#### CTS

CTS 是 Clock Tree Synthesis，时钟树综合。

作用：

```text
把理想时钟变成真实时钟网络
插入 clock buffer/inverter
控制 skew 和 latency
让各寄存器时钟到达时间合理
```

常见命令：

```tcl
remove_ideal_network [get_ports clk]
set_clock_tree_options
set_clock_tree_references
clock_opt -only_cts
report_clock_tree
report_clock_timing -type skew
```

为什么需要 CTS：

```text
DC 阶段常把 clock 当成理想网络
真实芯片里时钟也要走金属线，也有延迟和偏斜
CTS 就是构建真实时钟网络
```

#### post-CTS optimization

CTS 后时钟延迟变真实了，时序会变化。

因此需要：

```text
修 setup
修 hold
调整 buffer
优化路径
```

常见命令：

```tcl
set_fix_hold [all_clocks]
clock_opt -no_clock_route -fix_hold_all_clock
```

#### routing

作用：

```text
完成时钟网和信号网布线
生成真实金属线和 via
```

常见命令：

```tcl
route_zrt_group -all_clock_nets
route_zrt_auto
verify_zrt_route
```

布线完成后，设计才真正有了金属连接图形。

#### post-route优化与物理验证

布线把抽象连接变成真实几何，也会暴露此前无法准确看到的问题：

```text
线网过长导致setup/hold恶化
线间距或via不满足DRC
网络开路或短路
长金属线引发antenna风险
电源网络不连续
版图连接与逻辑网表不一致
```

脚本使用：

```tcl
focal_opt ...
verify_zrt_route ...
verify_pg_net ...
verify_lvs ...
```

`focal_opt`负责增量修复，`verify_zrt_route`检查布线质量，`verify_pg_net`检查供电网络，`verify_lvs`检查版图连接关系与网表是否一致。它们分别解决不同问题，不能由一个`report_timing`替代。

#### chip finish与filler

```tcl
insert_stdcell_filler ...
```

placement完成后，标准单元之间可能存在空隙。filler不参与逻辑功能，但用于保持well、implant和电源轨连续，并满足部分版图规则。插入后保存`finish`和`allfinish`等最终CEL，形成版图交付起点。

#### extract_rc

```tcl
extract_rc -coupling_cap
write_parasitics -output ../out/${DESIGN_NAME}.spef -format SPEF
```

作用：

```text
从布线结果提取电阻电容寄生参数
输出 SPEF 给 PT 做更准确的时序分析
```

为什么要提取 RC：

```text
后端真实连线会带来延迟
高频设计中连线延迟可能非常重要
PT 需要 SPEF 才能准确分析后布局时序
```

因此，TLUPlus和`extract_rc`不是同一件事：

```text
set_tlu_plus_file
    在初始化阶段提供“如何计算RC”的工艺模型

extract_rc
    在布线完成后把实际线长、金属层、via和耦合关系代入模型，得到每条net的具体RC
```

### 7.7 ICC 输出文件详解

#### 后布局网表

```text
icc/out/<DESIGN_NAME>_pred.v
```

包含后端插入或优化后的门级网表。

#### 后仿网表

```text
icc/out/<DESIGN_NAME>_nophy.v
```

通常是不含 physical-only cells 的网表，用于 VCS 后仿真。

physical-only cells 例如 filler、tap cell 等主要服务物理实现，不参与逻辑功能仿真。

#### 后布局 SDC

```text
icc/out/<DESIGN_NAME>_pred.sdc
```

后端输出的约束，给 PT 继续使用。

#### SDF

```text
icc/out/<DESIGN_NAME>.sdf
```

标准延迟格式，用于门级后仿真延迟标注。

SDF 描述：

```text
单元延迟
互连延迟
setup/hold 检查延迟
```

#### SPEF

```text
icc/out/<DESIGN_NAME>.spef
```

标准寄生参数格式。

SPEF 描述：

```text
每条 net 的电阻
每条 net 的电容
耦合电容
```

PT 读 SPEF 后能做更准确的后布局时序分析。

#### GDS

```text
icc/out/<DESIGN_NAME>.gds
```

GDS 是最终版图文件，描述芯片在硅片上的几何图形。

它描述：

```text
每层金属怎么画
通孔在哪里
标准单元摆在哪里
pin 在哪里
```

生成 GDS 需要：

```text
当前设计的 Milkyway design library
标准单元 GDS
GDS layer map 文件
```

其中：

```text
标准单元 GDS：提供标准单元内部版图形状
layer map：把 ICC 内部层名映射到 GDS layer/datatype 编号
```

真正输出的是：

```tcl
../out/${DESIGN_NAME}.gds
```

库里的 GDS 是参考输入，不是输出位置。

## 8. PT：静态时序分析

### 8.1 PT 为什么在 ICC 后还要做

ICC 自己也能报告时序，但 PT 是更常用的 signoff STA 工具。

PT 的作用：

```text
独立读取后布局网表
读取标准单元时序库
读取 SDC 约束
读取 SPEF 寄生参数
重新分析所有时序路径
检查约束是否完整
生成更可信的时序报告
```

为什么不能只看 DC：

```text
DC 没有真实布局布线
DC 的线延迟主要是估计
ICC 后真实连线和时钟树会改变时序
```

为什么还要看 PT：

```text
PT 是独立时序签核工具
可以避免只依赖实现工具自己的分析
更接近 tape-out 前的时序检查方式
```

### 8.2 PT 输入

```text
ICC 输出的后布局网表
ICC 输出的 SDC
ICC 输出的 SPEF
标准单元 .db timing library
PT 脚本
```

### 8.3 PT 典型目录

```text
pt/
├── scr/
│   ├── sts.tcl
│   └── sdf_gen.tcl
├── run/
└── out/
```

### 8.4 sts.tcl

典型流程：

```tcl
source library setup
set search_path
set link_library
read_db
read_verilog
link_design
current_design
read_parasitics
read_sdc
check_timing
update_timing
report_timing / report_qor / report_constraint
```

逐步解释：

```text
read_db              读取标准单元时序库
read_verilog          读取后布局门级网表
link_design           解析网表里的标准单元和层级引用
current_design        指定当前分析顶层
read_parasitics       读取 SPEF 寄生参数
read_sdc              读取时序约束
check_timing          检查约束和时序环境
update_timing         更新时序计算
report_*              输出时序和 QoR 报告
```

### 8.5 sdf_gen.tcl

SDF 生成脚本通常和 STA 脚本读同样的东西：

```text
后布局网表
标准单元库
SPEF
SDC
```

然后输出：

```text
pt/out/<DESIGN_NAME>.sdf
```

这个 SDF 可以给 VCS 后仿真使用。

### 8.6 setup 和 hold 应该看哪个库

基本原则：

```text
setup 看慢角 worst case
hold 看快角 best case
```

原因：

```text
setup 关心数据能不能在下一个时钟沿前到达，路径越慢越危险
hold 关心数据会不会在当前时钟沿后太快变化，路径越快越危险
```

教学脚本里可能只读一个 ss 慢角库，主要看 setup；完整项目通常会做多工艺角、多电压、多温度分析。

## 9. VCS_icc：版图后仿真

### 9.1 后仿真验证什么

后仿真验证的是：

```text
综合和布局布线后的门级网表
在标注 SDF 延迟后
功能是否仍正确
```

它不是重新综合，也不是重新布局。

### 9.2 后仿真输入

```text
testbench
ICC 输出的后仿网表 <DESIGN_NAME>_nophy.v
标准单元 Verilog 模型
SDF 延迟文件
file_list.f
Makefile
```

这里最容易混淆的是：后仿也会读 `.v` 文件，但这个 `.v` 通常不是 RTL。

例如：

```text
../icc/out/mul8_top_nophy.v
```

这是 ICC 输出的 Verilog 门级网表。它的后缀也是 `.v`，但内容已经不是 RTL 中的算法描述，而是由标准单元实例连接出来的结构网表。

可以这样区分：

```text
RTL .v
    描述“这个电路行为上应该怎么工作”
    常见 always、assign、case、for 等语句

门级网表 .v
    描述“这个电路由哪些标准单元连接而成”
    常见 INV、NAND、DFF、AOI 等标准单元实例
```

`nophy` 一般表示：

```text
no physical-only cells
```

也就是去掉 filler、tap cell 等只服务物理版图、不参与逻辑功能的单元。仿真关心逻辑功能和延迟，这类 physical-only cell 通常不需要进入后仿网表。

### 9.3 file_list.f

后仿 filelist 一般包括：

```text
testbench.v
../icc/out/<DESIGN_NAME>_nophy.v
/path/to/std/verilog/scc018ug_hd_rvt.v
```

为什么要读标准单元 Verilog：

```text
后布局网表里已经不是 RTL always 语句
而是大量标准单元实例
VCS 需要知道这些标准单元的仿真模型
```

例如门级网表里可能出现：

```verilog
INVD1 U12 (...);
ND2D1 U25 (...);
DFCNQD1 U31 (...);
```

这些模块不是 Verilog 内建模块，VCS 不会天然认识它们。标准单元 Verilog 模型文件的作用就是告诉仿真器：

```text
每种标准单元的逻辑功能是什么
触发器、锁存器、复位单元如何工作
是否包含 specify block 和时序检查
```

所以后仿 filelist 的核心关系是：

```text
testbench
    负责产生激励、检查结果、标注 SDF

<DESIGN_NAME>_nophy.v
    负责提供布局布线后的门级电路结构

标准单元 Verilog 模型
    负责提供门级网表中每个标准单元的仿真行为
```

### 9.4 SDF 标注

testbench 里常见：

```verilog
$sdf_annotate("../icc/out/top.sdf", u_top,,,
              "MINIMUM", "1:1:1", "FROM_MTM");
```

作用：

```text
把 SDF 延迟标注到门级实例上
让仿真考虑门延迟和连线延迟
```

完整一些看，这句话常见格式是：

```verilog
$sdf_annotate(
    sdf_file,
    instance,
    config_file,
    log_file,
    mtm_spec,
    scale_factors,
    scale_type
);
```

例如：

```verilog
initial begin
    $sdf_annotate("../icc/out/mul8_top.sdf",
                  u_mul8_top,
                  ,,
                  "MINIMUM",
                  "1:1:1",
                  "FROM_MTM");
end
```

逐项理解：

```text
"../icc/out/mul8_top.sdf"
    SDF 文件路径，记录布局布线后的 cell delay、net delay 和 timing check。

u_mul8_top
    反标对象，也就是 testbench 中例化的 DUT 实例名。
    SDF 延迟会加载到这个实例内部。

第三、第四个参数为空
    分别是 config file 和 log file。
    教学工程里通常不额外指定，所以留空。

"MINIMUM"
    选择 SDF 中的最小延迟。

"1:1:1"
    延迟缩放比例，格式是 min:typ:max。
    1:1:1 表示不缩放。

"FROM_MTM"
    表示从 SDF 的 min/typ/max 三元组中取值。
```

`"MINIMUM"` 和 `"FROM_MTM"` 的关系：

```text
FROM_MTM = 从 SDF 的 min/typ/max 三档延迟里选
MINIMUM  = 具体选择最小延迟那一档
```

如果 SDF 中某个延迟写成：

```text
(0.12:0.18:0.25)
```

那么：

```text
MINIMUM  -> 0.12
TYPICAL  -> 0.18
MAXIMUM  -> 0.25
```

因此这句 SDF 标注可以翻译成：

```text
在仿真开始时，把 mul8_top.sdf 中的最小延迟信息，
按照 1:1:1 的比例标注到 testbench 里的 u_mul8_top 实例上。
```

注意：

```text
SDF 标注对象必须是设计实例名
SDF 文件必须和当前网表对应
检查输出时要避开时钟边沿，留出延迟稳定时间
```

如果实例名不匹配，SDF 可能标不上；如果 SDF 和网表不是同一次后端输出，延迟路径也可能对不上。

### 9.5 +nospecify 要慎用

有些 Makefile 里会看到：

```makefile
simulate:
	./${OUTPUT} -l sim.log +nospecify
```

`+nospecify` 的作用是忽略标准单元模型中的 `specify` block。这样可以减少某些时序检查带来的干扰，但它也会让一部分路径延迟或时序检查不生效。

因此：

```text
快速看功能
    可以临时使用 +nospecify

严格后仿
    不建议随便加 +nospecify
```

课程截图如果是为了证明后仿结果，一般优先使用正常 `sim` 入口，而不是带 `+nospecify` 的简化入口。

### 9.6 后仿真输出

```text
compile log
simulation log
waveform
PASS/FAIL 自检结果
```

如果后仿不通过，可能原因包括：

```text
SDF 和网表不匹配
testbench 检查时间太靠近时钟边沿
时序真的违例
复位或初始化方式不适合门级仿真
标准单元模型路径错误
```

## 10. 常见文件类型速查

| 文件 | 阶段 | 作用 |
| --- | --- | --- |
| `.v` RTL | 前仿真/DC | 可综合设计源码 |
| testbench `.v` | VCS | 仿真激励和自检，不参与综合 |
| `file_list.f` | VCS | 编译文件列表 |
| `Makefile` | VCS/工程 | 封装编译、仿真、清理命令 |
| `.tcl` | DC/ICC/PT | 工具命令脚本 |
| `.db` | DC/ICC/PT | 标准单元时序、面积、功耗库 |
| `.sdc` | DC/ICC/PT | 时序约束 |
| `.ddc` | DC | DC 内部数据库 |
| mapped netlist `.v` | DC -> ICC | 综合后门级网表 |
| Milkyway reference library | ICC | 标准单元物理参考库，只读 |
| Milkyway design library | ICC | 当前设计物理数据库，可写 |
| tech file `.tf` | ICC | 工艺层和物理规则信息 |
| TLUPlus | ICC | RC 寄生参数模型 |
| SPEF | ICC -> PT | 布线寄生参数 |
| SDF | DC/ICC/PT -> VCS | 延迟标注文件 |
| GDS | ICC | 最终版图几何文件 |
| report | DC/ICC/PT | 给人看的分析结果 |

## 11. 建立或维护工程时的检查顺序

### 11.1 RTL 和前仿真

```text
rtl/ 是否放了正式 RTL
vcs_pre/file_list.f 是否读入 RTL 和 testbench
testbench 实例化模块名是否正确
端口名是否匹配
测试向量是否覆盖关键功能
```

### 11.2 DC

```text
MPW180_lib_list.tcl 库根路径是否正确
common_setup.tcl 的 DESIGN_NAME 是否正确
common_setup.tcl 的 RTL 搜索路径是否正确
dc_run.tcl 的 analyze 文件列表是否完整
sdc.tcl 的 clk/rst/input/output 端口是否和 RTL 一致
report_timing 是否有正 slack
out/ 是否生成 mapped.v、mapped.ddc、mapped.sdc、mapped.sdf
```

### 11.3 ICC

```text
common_setup.tcl 的 DESIGN_NAME 是否正确
ICC_INPUTS_PATH 是否指向 dc/out
init_design_icc.tcl 是否能读到 mapped 网表和 SDC
floorplan 面积是否适合当前设计规模
是否有 clk；如果没有时钟，CTS 流程要调整
输出文件名是否使用 DESIGN_NAME
out/ 是否生成 pred.v、nophy.v、sdf、spef、gds
```

### 11.4 PT

```text
read_verilog 是否读 <DESIGN_NAME>_pred.v
read_parasitics 是否读 <DESIGN_NAME>.spef
read_sdc 是否读 <DESIGN_NAME>_pred.sdc
report_timing 是否有正 slack
write_sdf 是否输出正确设计名
```

### 11.5 后仿真

```text
file_list.f 是否读后仿 testbench
file_list.f 是否读 <DESIGN_NAME>_nophy.v
标准单元 Verilog 模型路径是否正确
$sdf_annotate 路径是否正确
$sdf_annotate 实例名是否正确
自检结果是否全部 PASS
```

## 12. 脚本编写规范

### 12.1 设计名统一用变量管理

推荐：

```tcl
set DESIGN_NAME top
write_sdf ../out/${DESIGN_NAME}.sdf
```

避免：

```tcl
write_sdf ../out/fixed_top.sdf
```

原因：

```text
脚本不和某一个固定顶层绑定
减少硬编码导致的维护错误
```

### 12.2 库路径集中管理

推荐：

```tcl
set MPW180 "/home/wzs/SMIC/smic180_lib_list"
set MPW180_path "$MPW180/std"
```

避免在多个脚本里散落：

```text
/home/publib/digital/smic180_lib_list/...
/home/lijiaxiang/work/digital/smic180_lib_list/...
```

### 12.3 输入输出分清

每个阶段都要知道：

```text
输入来自哪里
输出写到哪里
报告写到哪里
```

例如：

```text
DC 输入：rtl/
DC 输出：dc/out/
ICC 输入：dc/out/
ICC 输出：icc/out/
PT 输入：icc/out/
后仿输入：icc/out/ + 标准单元 verilog + SDF
```

### 12.4 testbench 不进入综合

testbench 包含：

```text
initial
$display
$stop
$dumpvars
仿真延迟 #
```

这些不属于真实硬件，不能进入 DC 综合。

### 12.5 脚本要可复用

好脚本应该做到：

```text
工程顶层、RTL 文件列表、SDC 端口集中管理
库位置变化主要改库根路径
输出文件自动跟着 DESIGN_NAME 改
报告路径固定清晰
```

## 13. 从一个设计跑完整流程时应该理解什么

以一个普通同步数字电路为例，完整流程不是简单“跑脚本”，而是逐步回答这些问题：

### 13.1 RTL 是否正确

看：

```text
VCS 自检是否 PASS
波形是否符合预期
复位、使能、边界输入是否正确
```

### 13.2 RTL 能否综合

看：

```text
DC analyze/elaborate/link 是否报错
check_design 是否有严重问题
compile_ultra 是否完成
mapped.v 是否生成
```

### 13.3 时序是否满足

看：

```text
DC report_timing
ICC report_timing
PT report_timing
```

理解：

```text
DC 时序较理想或估计
ICC/PT 时序考虑布局布线和寄生参数，更接近真实情况
```

### 13.4 物理实现是否合理

看：

```text
floorplan 面积是否合适
placement 是否拥塞
power network 是否通过检查
routing 是否有 DRC/antenna/open/short 问题
GDS 是否输出
```

### 13.5 后仿是否通过

看：

```text
SDF annotation 是否成功
门级网表是否能编译
自检是否 PASS
波形是否和前仿一致
```

## 14. 常见误区

### 14.1 “GDS 是不是输出回标准单元库？”

不是。

标准单元库里的 GDS 是输入参考，当前设计的 GDS 输出到工程 `out/`。

```text
读取标准单元 GDS + layer map + 当前 Milkyway design library
  -> 输出当前设计 GDS
```

### 14.2 “Milkyway 是不是脚本？”

不是。

```text
Tcl 脚本 = 操作步骤
Milkyway design library = ICC 执行脚本后保存的物理设计数据库
```

### 14.3 “DC 和 PT 都看 timing，为什么还要 PT？”

DC 在综合阶段看时序，主要用于优化 RTL 到门级网表。

PT 在后布局阶段看时序，读入 SPEF，分析更接近真实物理实现的时序。

### 14.4 “SDF 和 SPEF 都和延迟有关，有什么区别？”

```text
SPEF：寄生 RC 参数，给 STA 工具计算延迟
SDF：已经形成的延迟信息，给仿真器标注延迟
```

### 14.5 “filelist 和 Tcl 都是文件列表吗？”

不是。

```text
file_list.f：通常给 VCS，列出编译哪些 Verilog
Tcl 脚本：给 DC/ICC/PT，执行工具命令
```

## 15. 最小复习路线

如果时间有限，按下面顺序复习：

```text
1. 看总流程：RTL -> DC -> ICC -> PT -> 后仿
2. 看每阶段输入输出
3. 看 DC 的 analyze/elaborate/link/compile/report/write
4. 看 ICC 的 floorplan/power/placement/CTS/routing/output
5. 看 PT 的 read_verilog/read_spef/read_sdc/report_timing
6. 看 VCS 后仿的 filelist + 标准单元模型 + SDF 标注
7. 最后看文件类型速查表
```

真正理解后，看到一个脚本应该能快速判断：

```text
它属于哪个阶段
它在设置环境、读输入、执行流程、生成报告还是输出结果
它读的文件来自哪里
它输出的文件给谁用
```
