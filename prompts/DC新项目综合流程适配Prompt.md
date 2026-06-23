# DC 新项目综合流程适配 Prompt

你是一名熟悉 Synopsys Design Compiler 的数字 IC 综合工程师。我要为一个新的 Verilog/SystemVerilog 设计建立一套清晰、可运行、便于维护的 DC 综合流程。

本任务只处理 DC 阶段。请先确认必要信息，再生成目录结构和脚本。默认采用**最小可用流程**，不要为了兼容未知需求加入空变量、备用分支、跨环境回退、重复检查，或同时生成普通 DC 与 Topographical 两套配置。

## 一、先确认必要信息

如果我已经提供了某项信息，不要重复询问。缺少会影响脚本正确性的内容时再提问。

### 1. 设计与 RTL

```text
顶层模块名：
RTL 语言：Verilog / SystemVerilog
RTL 文件列表及正确编译顺序：
时钟端口名：
复位端口名、有效电平、同步或异步：
是否包含 SRAM、ROM、IP 等宏单元：
是否需要 DesignWare：
```

不要读取 testbench。多文件 RTL 必须使用 Tcl `list` 保存，并保持依赖文件在前、顶层文件在后。

### 2. 综合目标与约束

```text
目标时钟周期或频率：
input delay：
output delay：
clock uncertainty：
clock transition：
clock source latency：
clock network latency：
输入 driving cell 及其输出 pin：
输出 load：
是否存在 false path、multicycle path 或异步时钟：
```

没有依据的约束值不得自行编造。如果某类约束不适用于当前设计，则不要生成对应的空变量和条件分支。

### 3. 库与运行模式

```text
标准单元库根目录及可用 .db：
当前综合使用的 PVT corner：
宏单元对应的 .db：
symbol library（如有）：
使用普通 dc_shell 还是 dc_shell -topo：
```

普通 DC 如果使用 wire load model，还需确认模型名称、所属 library 和 wire load mode。

Topo 模式还需确认：

```text
Milkyway reference library：
technology file：
TLUPlus max/min 文件：
technology-to-ITF map file：
Milkyway design library 输出目录：
```

### 4. 需要保留的输出

默认输出：

```text
mapped.v
mapped.ddc
mapped.sdc
timing / area / power / constraint 报告
完整控制台日志和命令日志
check_design / check_timing / compile 关键阶段日志
```

只有明确需要时才额外输出 unmapped DDC、SDF、SVF 或其他报告。

## 二、推荐目录结构

```text
project/
├── rtl/
└── dc/
    ├── scr/
    │   ├── lib_list.tcl
    │   ├── common_setup.tcl
    │   ├── dc_setup.tcl
    │   ├── sdc.tcl
    │   └── dc_run.tcl
    ├── syn/
    │   └── .synopsys_dc.setup
    ├── work/
    ├── alib/
    ├── log/
    ├── mw/                 # 仅 Topo 模式需要
    ├── out/
    └── rep/
```

各目录职责：

```text
rtl/    可综合 RTL
scr/    配置、约束和主流程脚本
syn/    dc_shell 启动目录
work/   analyze 产生的 WORK 设计库
alib/   DC 分析 .db 后产生的 ALIB 缓存
log/    完整会话、命令记录和关键阶段日志
mw/     Topo 模式的 Milkyway design library
out/    综合结果
rep/    综合报告
```

## 三、脚本职责与生成规则

### 1. lib_list.tcl：描述可用库资源

该文件只回答“库在哪里、有哪些可用资源”，不能决定当前设计使用哪个 PVT corner。

允许定义：

```tcl
set STD_DB_ROOT  "/path/to/std/db"
set SRAM_DB_ROOT "/path/to/sram/db"
```

只有在我提供了完整、可复用的多工艺角库清单时，才可以在这里建立中性的库索引，例如：

```tcl
set STD_TT_DB [file join $STD_DB_ROOT <tt_library>.db]
set STD_SS_DB [file join $STD_DB_ROOT <ss_library>.db]
set STD_FF_DB [file join $STD_DB_ROOT <ff_library>.db]
```

如果我只提供了当前设计要使用的某个标准单元库或 SRAM 库，不要在 `lib_list.tcl` 中创建带有“当前工程选择”含义的变量；具体文件选择应放入 `common_setup.tcl`。

`lib_list.tcl` 中不得设置 `target_library`、`link_library`，也不得出现设计名、RTL 文件和时序约束。

### 2. common_setup.tcl：描述当前设计

该文件负责选择当前综合对象、RTL、工艺角和宏单元库，例如：

```tcl
set DESIGN_NAME <top_name>

set RTL_SOURCE_FILES [list \
    <dependency_1.v> \
    <top.v> \
]

set TARGET_LIBRARY_FILES [list \
    [file join $STD_DB_ROOT <current_corner_stdcell>.db] \
]

set ADDITIONAL_LINK_LIBRARY_FILES [list \
    [file join $SRAM_DB_ROOT <current_corner_sram>.db] \
]
```

职责边界：

```text
lib_list.tcl       提供库目录或完整的可用库索引
common_setup.tcl   选择当前设计真正使用的 corner、宏库和 RTL
dc_setup.tcl       把这些选择配置给 DC 内置变量
```

不要预先生成当前设计不存在的端口变量、库变量和可选功能开关。

### 3. dc_setup.tcl：配置 DC 工具环境

配置 `search_path`、`target_library` 和 `link_library`：

```tcl
set_app_var search_path [concat $search_path $ADDITIONAL_SEARCH_PATH]
set_app_var target_library $TARGET_LIBRARY_FILES
set_app_var link_library [concat "*" \
    $TARGET_LIBRARY_FILES \
    $ADDITIONAL_LINK_LIBRARY_FILES]
```

其中：

```text
target_library                 DC 可以用于逻辑映射的标准单元库
link_library                   解析网表中标准单元、宏单元和已引用设计
ADDITIONAL_LINK_LIBRARY_FILES  SRAM、ROM、IP 等只需链接而不参与普通逻辑映射的库
```

有 symbol library 时再设置 `symbol_library`。需要 DesignWare 时再加入 synthetic library。普通 DC 与 Topo 配置只能根据已确认的运行模式生成一种。

### 4. sdc.tcl：描述设计时序环境

只生成当前设计真实需要的约束，通常包括：

```text
create_clock
set_clock_uncertainty
set_clock_transition
set_clock_latency（有明确预算时）
set_input_delay / set_output_delay（存在对应数据端口时）
set_driving_cell / set_load（已知外部电气环境时）
复位 false path（异步复位且方法合理时）
时序例外（确有设计依据时）
```

必须遵守：

```text
时钟端口不能作为普通数据输入设置 input delay
异步复位不能作为普通同步数据路径处理
input delay 与 driving cell 描述的对象不同
output delay 与 output load 描述的对象不同
clock source latency 与 network latency 含义不同
约束对象必须检查是否存在，不能依靠空集合掩盖端口名错误
```

纯寄存器接口且没有可靠板级预算时，应先说明缺失信息，而不是创建大量值为空的 IO 约束变量。

### 5. dc_run.tcl：执行线性综合流程

主流程保持顺序清晰，并在关键检查和综合阶段保存日志：

```tcl
set_app_var sh_continue_on_error false

set LOG_DIR ../log
set REP_DIR ../rep
set OUT_DIR ../out
file mkdir $LOG_DIR
file mkdir $REP_DIR
file mkdir $OUT_DIR

remove_design -all
analyze -format <verilog_or_sverilog> -library WORK $RTL_SOURCE_FILES
elaborate $DESIGN_NAME
current_design $DESIGN_NAME
link

redirect -tee -file $LOG_DIR/check_design.log {
    check_design
}

source ../scr/sdc.tcl

redirect -tee -file $LOG_DIR/check_timing.log {
    check_timing
    report_clock
}

redirect -tee -file $LOG_DIR/compile.log {
    compile_ultra
}

if {![get_attribute [current_design] is_mapped]} {
    error "综合后仍包含未映射逻辑，禁止写出正式网表"
}

redirect -file $REP_DIR/${DESIGN_NAME}_constraint.rpt {
    report_constraint -all_violators
}
redirect -file $REP_DIR/${DESIGN_NAME}_timing.rpt { report_timing }
redirect -file $REP_DIR/${DESIGN_NAME}_area.rpt   { report_area }
redirect -file $REP_DIR/${DESIGN_NAME}_power.rpt  { report_power }
redirect -file $REP_DIR/${DESIGN_NAME}_qor.rpt    { report_qor }

write -hierarchy -format ddc \
    -output $OUT_DIR/${DESIGN_NAME}.mapped.ddc
write -hierarchy -format verilog \
    -output $OUT_DIR/${DESIGN_NAME}.mapped.v
write_sdc $OUT_DIR/${DESIGN_NAME}.mapped.sdc
```

日志只保留最关键的三份阶段文件：

```text
check_design.log    设计结构、连接和引用问题
check_timing.log    时钟及约束完整性问题
compile.log         优化、库分析和工艺映射问题
```

不要为 analyze、elaborate 和每个 write 命令机械生成独立日志；这些过程统一保留在完整控制台日志中。

必须设置 `sh_continue_on_error=false`，并在 `compile_ultra` 后检查当前设计的 `is_mapped` 属性。综合失败或仍有未映射逻辑时必须停止，不能继续生成名称为 `mapped.v` 的无效网表。

只有明确要求时才加入：

```text
set_svf
unmapped DDC
write_sdf
编译前后重复报告
GTECH/DesignWare 残留检查
大量 max/min path 报告
自动删除并重建 WORK
错误后自动尝试其他配置
```

不要在 `dc_run.tcl` 中重复 source 已由 `.synopsys_dc.setup` 加载的配置文件。

### 6. .synopsys_dc.setup：统一启动入口

从 `dc/syn/` 启动 `dc_shell`，由该文件完成工作目录初始化和配置加载：

```tcl
history keep 200

file mkdir ../work
file mkdir ../alib
file mkdir ../log
file mkdir ../out
file mkdir ../rep

set_app_var sh_output_log_file  ../log/dc_console.log
set_app_var sh_command_log_file ../log/command.log
set_app_var alib_library_analysis_path ../alib
define_design_lib WORK -path ../work

source ../scr/lib_list.tcl
source ../scr/common_setup.tcl
source ../scr/dc_setup.tcl
```

日志含义：

```text
dc_console.log    几乎完整的控制台输出，用于寻找第一条 Error/Fatal
command.log       初始变量和实际执行命令，用于复现工具状态
```

只保留真正有用的交互 alias。配置加载只能选择一个入口，不要再让 `dc_run.tcl` 编写相同的 fallback source 逻辑。

## 四、输出时的要求

获得必要信息后，请按以下顺序回答：

1. 给出最终目录结构和文件放置说明。
2. 输出六个完整脚本。
3. 简要解释每个脚本的输入、职责和与下一阶段的衔接。
4. 给出从 `dc/syn/` 启动和执行综合的命令。
5. 列出预期输出，并说明如何从日志判断流程是否成功。
6. 说明日志和报告的区别，并给出故障定位时的阅读顺序。

脚本要求：

```text
使用适量中文注释，不逐行机械解释
路径和文件列表使用 Tcl list/file join 等结构化写法
不硬编码尚未确认的信息
不生成当前设计用不到的配置
不同时保留多套互斥实现
不通过复杂防御代码掩盖配置错误
发现错误时先定位完整日志中的第一条根因，不追逐后续连锁报错
```

解释内容与脚本分开。先保证脚本短、直观、可运行，再说明关键命令的作用。
