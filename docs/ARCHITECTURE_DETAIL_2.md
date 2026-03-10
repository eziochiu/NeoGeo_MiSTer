# NeoGeo MiSTer FPGA 核心 — 深度分析文档

> 基于源码逐行分析，覆盖 LSPC2 视频控制器、CPU 集成、TTL 基础单元、SDRAM 控制器、顶层模块剩余部分

---

## 一、LSPC2 视频控制器完全解析 (`rtl/video/lspc2_a2.v`, 751行)

LSPC2-A2 是整个 Neo Geo 最复杂的自定义芯片，负责精灵评估、渲染调度、VRAM 仲裁和视频时序。

### 1.1 时钟分频 (`lspc2_clk.v`)

```
24MHz 系统时钟
  ├─ ÷2 → 12MHz (LSPC_12M)
  ├─ ÷4 → 6MHz  (LSPC_6M)
  ├─ ÷8 → 3MHz  (LSPC_3M)
  └─ ÷16 → 1.5MHz (LSPC_1_5M)

所有使能信号从 CLK_EN_24M_P 和 DIV_CNT 解码生成：
  LSPC_EN_12M_P/N  — 12MHz 正/负沿使能
  LSPC_EN_6M_P/N   — 6MHz 正/负沿使能
  LSPC_EN_3M       — 3MHz 使能
  LSPC_EN_1_5M_P/N — 1.5MHz 正/负沿使能
```

### 1.2 像素计数器 PIXELC 与光栅计数器 RASTERC

```
PIXELC[8:0] = {P15_Q[2:0], P50_Q[3:0], LSPC_1_5M, LSPC_3M}
  P50_Q: 4-bit 计数器，溢出后产生 P50_CO
  P15_Q: 4-bit 计数器，P50_CO 进位驱动，溢出后从 0x5 重载
  每扫描线总计 384 个像素时钟（在 6MHz 下）

RASTERC[8:0] = {J268_J269_Q[7:0], FLIP}
  J268_J269_Q: 8-bit 行计数器
  溢出重载值：
    NTSC (VMODE=0): 0xF8 → 共 263 扫描线
    PAL  (VMODE=1): 0xC8 → 共 311 扫描线
  FLIP: 每场翻转，用于隔行扫描

行切换条件：
  RASTER_CHG = ({P15_Q[2:0], P50_Q} == 7'b0111111) & LSPCE_EN_1_5M_N
  即 PIXELC 高位到达特定值时触发
```

### 1.3 像素时钟生成 (PCK1, PCK2, H, EVEN1)

```verilog
// PCK1/PCK2 在 CLK_EN_24M_N 边沿更新
PCK1 <= T160A_OUT   // 来自 slow_cycle 的时序信号
PCK2 <= T160B_OUT

// H 信号（水平翻转方向）
T172_Q <= SPR_TILE_HFLIP  // PCK1 下降沿锁存
H      <= T172_Q           // PCK2 下降沿锁存

// CA4（选择 CR_DOUBLE 的前/后 8 像素）
CA4 = T172_Q ^ LSPC_1_5M

// 使能信号生成
PCK1_EN_P = CLK_EN_24M_N & ~PCK1 &  T160A_OUT  // PCK1 上升沿
PCK1_EN_N = CLK_EN_24M_N &  PCK1 & ~T160A_OUT  // PCK1 下降沿
PCK2_EN_P = CLK_EN_24M_N & ~PCK2 &  T160B_OUT
PCK2_EN_N = CLK_EN_24M_N &  PCK2 & ~T160B_OUT

// EVEN1（ZMC2 奇偶像素选择）
// 复杂的奇偶逻辑，取决于 HSHRINK_OUT_A/B 和 EVEN_nODD 状态
EVEN1 = ~&{多个 NAND 门组合}  // 控制 ZMC2 输出哪一对像素
```

### 1.4 VRAM 寄存器接口 (`lspc_regs.v`)

```
CPU 地址映射（M68K_ADDR[3:1] 选择寄存器）：

地址偏移  寄存器            功能
0 ($C00000) REG_VRAMADDR    VRAM 地址寄存器（16-bit）
1 ($C00002) REG_VRAMRW      VRAM 数据端口（读/写触发自动递增）
2 ($C00004) REG_VRAMMOD     VRAM 地址自增步长
3 ($C00006) REG_LSPCMODE    模式寄存器
              [15:8] = AA_SPEED  自动动画速度
              [7]    = TIMER_MODE[2] 定时器溢出重载
              [6]    = TIMER_MODE[1] 扫描线触发重载
              [5]    = TIMER_MODE[0] 软件触发重载
              [4]    = TIMER_IRQ_EN  定时器中断使能
              [3]    = AA_DISABLE    禁用自动动画
4 ($C00008) TIMER_HIGH      定时器高 16 位
5 ($C0000A) TIMER_LOW       定时器低 16 位
6 ($C0000C) IRQ_ACK         中断确认（写入位 [2:0] 清除对应 IRQ）
7 ($C0000E) TIMER_STOP      定时器停止标志（bit 0）

CPU 读取复用：
  根据 M68K_ADDR[2:1] 和 REG_VRAMADDR[15] 选择：
  CPU_DATA_MUX = ADDR[2] ?
    (高地址 ? REG_LSPCMODE : REG_VRAMMOD)
    : (高地址 ? VRAM_HIGH_READ : VRAM_LOW_READ)

VRAM 自动递增：
  VRAM_ADDR_AUTOINC = REG_VRAMMOD + VRAM_ADDR
  写入 VRAM 数据端口后，地址自动加上步长
  步长可为 1、32、64 等任意值
```

### 1.5 VRAM 写请求握手

```verilog
// 写请求 DFF：CPU 写 REG_VRAMRW → VRAM_WRITE_REQ 置 1
register D38(CLK, 1'b0, VRAM_WRITE_DONE, ~WR_VRAM_RW, 1'b1, VRAM_WRITE_REQ);

// 写完成检测：VRAM_WRITE_ACK 上升沿 → VRAM_WRITE_DONE 置 1
// VRAM_WRITE_DONE 复位 VRAM_WRITE_REQ

// 地址更新触发：
O108B_OUT = ~&{nCPU_WR_HIGH, nCPU_WR_LOW}
D112B_OUT = ~|{~WR_VRAM_ADDR, O108B_OUT}

// 地址源选择（M68K_ADDR[1] 控制）：
VRAM_ADDR_MUX = VRAM_ADDR_UPD_TYPE ? VRAM_ADDR_AUTOINC : REG_VRAMADDR
```

### 1.6 快速周期 — 精灵 Y 评估与活跃列表 (`fast_cycle.v`, 503行)

```
功能：逐个检查所有精灵的 Y 坐标，找出与当前扫描线相交的精灵

精灵 Y 匹配算法：
  PARSE_LOOKAHEAD = 当前光栅行 + 2（流水线延迟补偿）
  PARSE_ADD_Y = PARSE_LOOKAHEAD + 精灵Y坐标[7:0]
  N186_OUT = ~^{PARSE_ADD_Y[8], 精灵Y[8]}  // 溢出检测
  PARSE_ADD_SIZE = {N186_OUT, ~PARSE_ADD_Y[7:4]} + 精灵尺寸[4:0]
  PARSE_MATCH = PARSE_ADD_SIZE[5]  // 溢出 = 匹配！

Y 评估状态机（N98 4级移位寄存器）：
  N98_QA → 精灵属性读取（Y 坐标 + 尺寸）
  N98_QB → 数据锁存（缩放值）
  N98_QC → 数据锁存（Y/尺寸/链接）
  N98_QD → 活跃列表写入门控

活跃精灵列表：
  双端口 Fast VRAM 存储
  写地址 ACTIVE_WR_ADDR：匹配时递增（最大 96 个精灵/行）
  读地址 ACTIVE_RD_ADDR：PIXELC[6:0] 溢出时重载为 0
  满检测：ACTIVE_WR_ADDR[6:5] 全 1 → nACTIVE_FULL

  每扫描线最多 96 个可见精灵（硬件限制）

精灵参数流水线（42-bit 移位寄存器）：
  SR_SPR_PARAMS <= {SR_SPR_PARAMS[27:0], {SPR_CHAIN, XSHRINK, F[15:7]}}
  3 级流水：当前精灵 → 前一精灵 → 前前精灵
  最终级输出 PIPE_C（X 坐标 14-bit）和 HSHRINK（水平缩放 4-bit）

解析控制：
  PARSE_INDEX 遍历所有精灵（0~383）
  解析完成条件：PARSE_INDEX 特定位模式匹配（全部扫描完毕）
```

### 1.7 慢速周期 — Fix 图层与精灵属性 (`slow_cycle.v`, 206行)

```
功能：顺序访问 Slow VRAM，读取 Fix 图层和精灵属性

地址多路复用器（4 个源）：
  1. FIXMAP_ADDR = {4'b1110, O62_nQ, PIXEL_HPLUS, ~PIXEL_H8, RASTERC[7:3]}
     Fix 图层 tile map 地址
  2. SPRMAP_ADDR = {H57_Q, ACTIVE_RD, O185_Q, SPR_TILEMAP, K166_Q}
     精灵属性 map 地址
  3. VRAM_ADDR — CPU 访问地址
  4. 零（填充）

数据锁存时序（Q162_Q 移位寄存器驱动）：
  Q162_Q[0] → CPU 读取低字节
  Q162_Q[1] → CPU 读取低字节使能
  Q162_Q[3] → 精灵属性锁存使能

数据提取：
  FIX_MAP_READ[11:0]  → FIX_TILE（Fix tile 编号 12-bit）
  FIX_MAP_READ[15:12] → FIX_PAL（Fix 调色板 4-bit）
  SPR_TILE[19:0]      → 精灵 tile 编号 20-bit
  SPR_TILE_VFLIP      → 垂直翻转
  SPR_TILE_HFLIP      → 水平翻转
  SPR_AA_2, SPR_AA_3  → 自动动画位
  SPR_PAL             → 精灵调色板

BOE/BWE 控制：
  BOE → CPU 读使能（CLK_EN_24M_P 边沿）
  BWE → VRAM 写使能（LSPC_EN_12M_N 边沿）
```

### 1.8 定时器逻辑 (`lspc_timer.v`, 167行)

```
32-bit 递增计数器（加载反转值，溢出等于"归零"）

TIMER_LOW[15:0]  — 在 LSPC_EN_6M_N 边沿递增
TIMER_HIGH[15:0] — TIMER_LOW 全 1 进位时递增

加载操作：
  nRELOAD=0 → TIMER_LOW <= ~REG_TIMERLOW, TIMER_HIGH <= ~REG_TIMERHIGH
  效果：计数从反转值递增到 0xFFFF_FFFF → 溢出

溢出检测：
  TIMER_CO = nTIMER_EN & &TIMER_LOW & &TIMER_HIGH

三种重载模式：
  Mode 0 (TIMER_MODE[0]): 软件触发
  Mode 1 (TIMER_MODE[1]): 扫描线触发（FLIP 边沿）
  Mode 2 (TIMER_MODE[2]): 溢出自动重载 + 触发 IRQ

  nRELOAD = ~|{RELOAD_MODE0, RELOAD_MODE1, RELOAD_MODE2}

定时器使能控制：
  受 RASTERC 位、VMODE、TIMER_STOP 共同控制
  nTIMER_EN 为低时暂停计数
```

### 1.9 P-Bus 输出复用

```
P-Bus 在不同时间携带不同数据（S183_Q_DELAYED 和 S171_nQ 控制）：

模式                P_OUT_MUX[23:16]          P_OUT_MUX[15:0]
精灵模式1(S183=0,S171=0)  SPR_PAL               X坐标/缩放
精灵模式2(S183=0,S171=1)  {tile[19:16],SPR_LINE}  {tile[15:0],AA}
Fix 模式1(S183=1,S171=0)  0x00                   {PIXELC,RASTERC,FIX_TILE}
Fix 模式2(S183=1,S171=1)  {0000,FIX_PAL}         0x0000

LO ROM 寻址：
  LO_ROM_ADDR = {YSHRINK, LO_LINE_MUX}
  LO ROM 返回垂直缩放后的实际行号

精灵 Y 计算：
  SPR_Y_LOOKAHEAD = {RASTERC[7:1], FLIP} + 1
  SPR_Y_ADD = SPR_Y_LOOKAHEAD + SPR_Y[7:0]
  SPR_CONTINUOUS = &{SPR_SIZE0, SPR_SIZE5, SPR_Y_SHRINK[8]}  // 连续精灵检测
```

### 1.10 行缓冲控制信号

```
行缓冲使用 NEO-B1 中的 4 块 RAM（RAMBR/BL/TR/TL），由 LSPC2 控制写入：

LD1/LD2 — 行缓冲地址重载
  通过 R50_Q、R69_Q、S55_Q 等触发器链生成
  LD1 控制 Bottom 缓冲对，LD2 控制 Top 缓冲对

SS1/SS2 — 缓冲选择/清除
  SS1 = ~|{S48_nQ, CHG_D}
  SS2 = ~|{nCHG_D, S48_nQ}

WE1-4 / CK1-4 — 写使能和地址时钟
  由 T31_P 和 U24_P 锁存器生成
  写使能受 DOTA/DOTB（像素是否不透明）门控
  不透明像素才写入行缓冲

CHG — 缓冲交换指示
  每条扫描线结束时翻转
  渲染写入一个 bank，显示读出另一个 bank
```

### 1.11 完整扫描线渲染周期

```
一条扫描线（384 个 6MHz 像素时钟 = 64μs）：

阶段1：快速周期（Fast Cycle）
  ├─ 每 1.5MHz 周期执行一次精灵 Y 评估
  ├─ T125A_OUT 驱动 N98 移位寄存器（4 级流水线）
  ├─ PARSE_INDEX 遍历精灵列表（0~383）
  ├─ Y 匹配的精灵写入活跃列表
  └─ 活跃列表满（96个）后停止

阶段2：慢速周期（Slow Cycle）+ 像素输出
  ├─ 从 Slow VRAM 读取活跃精灵属性（tile、调色板、翻转）
  ├─ 触发 SDRAM burst 读取精灵 C ROM 图形
  ├─ ZMC2 解码位平面 → 像素
  ├─ 写入行缓冲（上一行数据同时输出显示）
  ├─ Fix 层 tile 从 SDRAM 读取
  ├─ NEO-B1 合并精灵 + Fix 层
  └─ 查调色板 RAM → RGB 输出

阶段3：行切换
  ├─ RASTERC 递增
  ├─ ACTIVE_RD_ADDR 重载为 0
  ├─ 行缓冲 bank 交换（CHG 翻转）
  └─ PARSE_INDEX 重置，开始新行评估
```

---

## 二、CPU 集成详解

### 2.1 68000 CPU 封装 (`cpu_68k.v`, 84行)

```verilog
// 端口映射
module cpu_68k(
    CLK,              // 48MHz 主时钟
    CLK_EN_68K_P/N,   // 12MHz 正/负沿使能
    nRESET,            // 复位（来自看门狗 neo_b1.v）
    M68K_ADDR[23:1],   // 23-bit 地址总线（字地址）
    FX68K_DATAIN[15:0],  // 数据输入
    FX68K_DATAOUT[15:0], // 数据输出
    nLDS, nUDS,        // 字节选通
    nAS,               // 地址选通
    M68K_RW,           // 读(1)/写(0)
    IPL2, IPL1, IPL0,  // 中断优先级
    nDTACK,            // 数据确认（调整后的）
    nBG,               // 总线授权输出（DMA 用）
    nBR, nBGACK,       // 总线请求/确认
    FC2, FC1, FC0,     // 功能码（CPU 状态指示）
    SYSTEM_CDx         // 系统类型（影响 VPA 逻辑）
);

// 关键内部逻辑
EN_PHI1 = CLK_EN_68K_P    // fx68k 相位 1 使能
EN_PHI2 = CLK_EN_68K_N    // fx68k 相位 2 使能

// 复位同步化（在 EN_PHI2 相位采样）
always @(posedge CLK)
    if (EN_PHI2) reset <= ~nRESET;

// VPA 信号生成（中断确认检测）
VPAn = SYSTEM_CDx | nAS | ~&M68K_ADDR[23:4]
// VPA 有效条件：非 CD 模式 + 地址有效 + 地址高位全 1 ($FFFxxx)
// 这是 Neo Geo 的简化中断确认方式（不使用 FC 功能码）

// BERR 始终未触发（1'b1）
// IPL2 始终拉高（中断优先级 0-2 只用 IPL0 和 IPL1）
```

### 2.2 68000 数据总线复用 (`neogeo.sv` 1304-1320行)

```verilog
// 字节掩码处理
M68K_DATA_BYTE_MASK =
    (~|{nLDS, nUDS}) ? M68K_DATA :                    // 两个字节都有效
    (~nLDS)          ? {8'h00, M68K_DATA[7:0]} :       // 只有低字节
    (~nUDS)          ? {M68K_DATA[15:8], 8'h00} :      // 只有高字节
                       16'h0000;                        // 都无效

// 双向总线控制
M68K_DATA = M68K_RW ? 16'bz : FX68K_DATAOUT;  // 读时释放，写时驱动
FX68K_DATAIN = M68K_RW ? M68K_DATA_BYTE_MASK : 16'h0000;
```

### 2.3 Z80 CPU 封装 (`cpu_z80.v`, 59行)

```verilog
module cpu_z80(
    CLK,               // 48MHz 主时钟
    CLK4P_EN, CLK4N_EN, // 4MHz 正/负沿使能
    nRESET,
    SDD_IN[7:0],        // 数据输入
    SDD_OUT[7:0],       // 数据输出
    SDA[15:0],          // 16-bit 地址总线
    nIORQ, nMREQ,       // I/O 和内存请求
    nRD, nWR,           // 读/写
    nBUSRQ, nBUSAK,     // 总线仲裁
    nINT, nNMI, nWAIT   // 中断和等待
);

// MREQ 特殊处理（屏蔽刷新周期）
nMREQ = MREQ_n | ~RFSH_n
// 刷新周期时 RFSH_n=0 → nMREQ 被拉高 → 外部存储器不响应

// T80pa 核心使用 CEN_p/CEN_n 使能信号
// 每个 CEN_p 脉冲执行一个 T-state
// 有效指令时钟 = 4MHz
```

### 2.4 Z80 总线仲裁（Neo CD 模式）

```verilog
// neogeo.sv 1788-1792行
// Z80 总线可由三个主控设备驱动：

{ SDA, SDD_OUT } = ~CD_HAS_Z80_BUS
    ? { Z80_SDA, Z80_SDD_OUT }           // 正常模式：Z80 驱动
    : DMA_RUNNING
        ? { DMA_ADDR_OUT[16:1], DMA_DATA_OUT[7:0] }  // CD DMA 驱动
        : { M68K_ADDR[16:1], M68K_DATA[7:0] };       // 68k 直接驱动

{ nSDRD, nSDWR } = ~CD_HAS_Z80_BUS
    ? { Z80_nSDRD, Z80_nSDWR }
    : { ~CD_TR_RD_Z80, ~CD_TR_WR_Z80 };

// CD_HAS_Z80_BUS = CD_USE_Z80（由 cd.sv 控制）
// DMA_RUNNING = DMA 活跃标志
```

---

## 三、TTL 基础单元库 (`rtl/cells/`)

Neo Geo 原始硬件使用 TTL 逻辑芯片，FPGA 实现用 Verilog 模拟这些芯片。

### 3.1 单元一览

| 文件 | 原始芯片 | 功能 | 延迟 |
|------|---------|------|------|
| `bd3.v` | 74244/245 | 单向缓冲器 | 5ns |
| `fdm.v` | 7474/74175 | D 触发器（Q + nQ 输出） | 2ns |
| `fds.v` | 74595/166 | 4-bit 寄存器 | 1ns |
| `fds16bit.v` | — | 16-bit 寄存器（4×FDSCell） | 1ns |
| `lt4.v` | 74373/375 | 4-bit 透明锁存器 | 0 |
| `C43.v` | 74163 | 4-bit 同步计数器 | — |
| `register.v` | — | 通用寄存器（带 set/reset） | — |

### 3.2 BD3 — 缓冲驱动器

```verilog
module BD3(input INPT, output OUTPT);
    assign #5 OUTPT = INPT;  // 5ns 传播延迟
endmodule

// 用途：信号缓冲、添加可控延迟、隔离时钟域
```

### 3.3 FDM — D 触发器

```verilog
module FDM(input CK, D, output reg Q, output nQ);
    always @(posedge CK) Q <= #2 D;  // 上升沿采样，2ns 延迟
    assign nQ = ~Q;                   // 组合反转
endmodule

// 用途：状态机、中断标志存储、异步信号同步
```

### 3.4 C43 — 4-bit 同步计数器

```verilog
module C43(CK, D[3:0], nL, EN, CI, nCL, Q[3:0], CO);
    always @(posedge CK, posedge CL)
        if (CL)        Q <= 0;       // 异步清零（最高优先级）
        else if (!nL)  Q <= D;       // 同步加载
        else if (EN&CI) Q <= Q + 1;  // 使能计数
    assign CO = &{Q, CI};             // 进位输出 = 全 1 且 CI=1

// 级联示例（12-bit 计数器）：
// C43 cnt0(CLK, D[3:0],  nL, EN, CI,  nCL, Q[3:0],  CO0);
// C43 cnt1(CLK, D[7:4],  nL, EN, CO0, nCL, Q[7:4],  CO1);
// C43 cnt2(CLK, D[11:8], nL, EN, CO1, nCL, Q[11:8], CO2);
```

### 3.5 LT4 — 透明锁存器

```verilog
module LT4(input nG, input [3:0] D, output reg [3:0] P, output [3:0] N);
    always @(*) P = (!nG) ? D : P;  // nG=0 透明，nG=1 保持
    assign N = ~P;
endmodule

// 与 DFF 的区别：LT4 是电平触发（组合逻辑），DFF 是边沿触发
// 用途：跨时钟域数据保持、复用器控制锁存
```

### 3.6 register — 通用寄存器

```verilog
module register #(WIDTH=1) (clock, s, r, c, d, q);
    // 优先级：reset(r) > set(s) > clock(c)
    always @(*) begin
        if (r)      q = 0;              // 异步复位
        else if (s) q = {WIDTH{1'b1}};   // 异步置位
        else        q = val_reg;         // 保持
    end
    always @(posedge clock) begin
        if (~c_d & c & ~r & ~s)          // c 上升沿且无异步控制
            val_reg <= d;
    end
endmodule

// 用途：中断标志（可独立 set/reset）、控制信号锁存
```

---

## 四、精灵像素解码器 (`zmc2_dot.v`, 62行)

```
功能：将 SDRAM burst 读取的 32-bit 位平面数据转换为 4-bit 像素索引

输入：
  CR[31:0] — 4 字节 × 8 位平面，来自 CR_DOUBLE（经 CA4 选择前/后 8 像素）
  EVEN     — 奇偶行选择
  LOAD     — 加载移位寄存器
  H        — 水平翻转方向

内部 32-bit 移位寄存器 SR：
  LOAD=1 时：SR <= CR（并行加载）
  H=1 时（左移）：
    SR <= {SR[29:24],2'b00, SR[21:16],2'b00, SR[13:8],2'b00, SR[5:0],2'b00}
  H=0 时（右移）：
    SR <= {2'b00,SR[31:26], 2'b00,SR[23:18], 2'b00,SR[15:10], 2'b00,SR[7:2]}

像素提取（组合逻辑，根据 EVEN 和 H 选择 SR 位）：
  {EVEN,H}=00: GAD={SR[24],SR[16],SR[8],SR[0]}, GBD={SR[25],SR[17],SR[9],SR[1]}
  {EVEN,H}=01: GAD={SR[31],SR[23],SR[15],SR[7]}, GBD={SR[30],SR[22],SR[14],SR[6]}
  {EVEN,H}=10: GAD 和 GBD 互换
  {EVEN,H}=11: GAD 和 GBD 互换

  GAD[3:0] → 像素 A 的 4-bit 颜色索引
  GBD[3:0] → 像素 B 的 4-bit 颜色索引

不透明标志：
  DOTA = |GAD  （GAD 非零 = 不透明）
  DOTB = |GBD
```

---

## 五、NEO-B1 像素合并完整逻辑 (`neo_b1.v`, 147行)

```
功能：合并精灵层和 Fix 层，输出调色板地址

Fix 层处理：
  FIXD[7:0] 在 CLK_EN_1HB 锁存
  S1H1 选择高/低 nibble → FIX_COLOR[3:0]
  FIX_OPAQUE = |FIX_COLOR & EN_FIX

4 块行缓冲 RAM 由 LSPC2 写入：
  RAMBR(CK[0], WE[0], LD1, SS1, GAD) — Bottom-Right，精灵 A
  RAMBL(CK[1], WE[1], LD1, SS1, GBD) — Bottom-Left，精灵 B
  RAMTR(CK[2], WE[2], LD2, SS2, GAD) — Top-Right，精灵 A
  RAMTL(CK[3], WE[3], LD2, SS2, GBD) — Top-Left，精灵 B

输出选择：
  MUX_BA = {TMS0, S1H1}
  00 → RAMBR，01 → RAMBL，10 → RAMTR，11 → RAMTL

调色板地址优先级（PA[11:0] 输出）：
  1. CPU 访问（最高）— 地址 $400000-$7FFFFF 时 nCPU_ACCESS=0
     PA = M68K_ADDR_L[12:1]
  2. 消隐 — CHBL=1 时
     PA = 12'h000（黑色）
  3. Fix 层 — FIX_OPAQUE=1 时
     PA = {4'b0000, FIX_PAL_REG[3:0], FIX_COLOR[3:0]}
  4. 精灵层（最低）
     PA = RAM_MUX_OUT[11:0]（来自行缓冲）

看门狗集成：
  watchdog WD 实例嵌入 NEO-B1 内部
  监控 CPU 活动，超时触发 nRESET/nHALT
```

---

## 六、NEO-C1 地址解码完整逻辑 (`neo_c1.v`, 162行)

```
输入：M68K_ADDR[21:17], A22Z, A23Z, nLDS, nUDS, RW, nAS

区域解码（使用地址高位组合逻辑）：

  区域              地址范围            激活条件
  nROM_ZONE       $000000-$0FFFFF   A23=0, A22=0, A21=0, A20=0
  nWRAM_ZONE      $100000-$1FFFFF   A23=0, A22=0, A21=0, A20=1
  nPORT_ZONE      $200000-$2FFFFF   A23=0, A22=0, A21=1, A20=0
  nIO_ZONE        $300000-$3FFFFF   A23=0, A22=0, A21=1, A20=1
  nPAL_ZONE       $400000-$7FFFFF   A23=0, A22=1
  nSROM_ZONE      $800000-$BFFFFF   A23=1, A22=0
  nLSPC_ZONE      $C00000-$CFFFFF   基于 nIO_ZONE 子解码

I/O 子区域（$300000-$3FFFFF 内 M68K_ADDR[19:17] 进一步解码）：
  nCTRL1_ZONE     $300000  P1 手柄读取
  nICOM_ZONE      $320000  68k↔Z80 通信
  nCTRL2_ZONE     $340000  P2 手柄读取
  nSTATUSB_ZONE   $380000  系统状态读取
  nBITW0          $380001  系统输出写
  nBITW1          $3A0001  系统锁存写
  nLSPC_ZONE      $3C0000  LSPC 视频寄存器

输出使能规则：
  读使能 = ~RW | nStrobe | nZone   （读操作 + 有效选通 + 在区域内）
  写使能 = RW | nStrobe | nZone    （写操作 + 有效选通 + 在区域内）

子模块：
  c1_regs    — 68k↔Z80 通信寄存器
  c1_wait    — 等待状态生成器
  c1_inputs  — P1/P2 手柄输入复用
```

### 6.1 等待状态生成 (`c1_wait.v`)

```
功能：根据访问类型插入等待周期（延迟 nDTACK 响应）

nDTACK = nAS | WAIT_MUX

WAIT_MUX 真值表：
  访问类型           nROMWAIT   nPWAIT    等待周期    说明
  ROM                0          X         4           慢速 ROM
  ROM                1          X         0           快速 ROM
  PORT               X          01        4           标准等待
  PORT               X          10        3           减少等待
  CARD               X          X         4           记忆卡
  其他               X          X         0           无等待

等待计数器：
  WAIT_CNT[2:0] 在 nAS=0 时递减
  nAS=1 时重载为 5
  WAIT_MUX 在 WAIT_CNT 达到阈值时变 0 → nDTACK 有效
```

---

## 七、SDRAM 控制器完整分析

### 7.1 SDRAM 多路复用器状态 (`sdram_mux.sv`, 299行)

```
请求源优先级（非传统状态机，基于请求标志 + 组合优先级）：

优先级  请求源              触发条件                数据宽度
1(最高) ROM 加载           DL_WR & DL_EN           16-bit 写
2       68k ROM 读         nROMOE/nSROMOE/nPORTOE↓ 16-bit 读
3       精灵 C-ROM Burst   PCK1B↑ & SPR_EN         4×16-bit burst
4       Fix S-ROM 单次     PCK2B↑ & FIX_EN         16-bit 读
5       CD DMA 传输        CD_TR_RUN               16-bit 读/写
6(最低) SDRAM 刷新         refresh_cnt 溢出        —

请求握手：
  每个源有 *_REQ（请求标志）和 *_RUN（运行标志）
  SDRAM_READY 上升沿表示物理控制器准备就绪
  下降沿表示数据已可用
```

### 7.2 精灵 Burst 读取流程

```verilog
// 触发：PCK1B 上升沿
if (REQ_CROM_RD | CROM_RD_RUN) begin
    CROM_RD_RUN <= 1;
    SDRAM_RD <= 1;
    SDRAM_ADDR <= CROM_ADDR[26:1];  // C ROM 地址
end

// 数据收集（4 word burst）：
cr_shift[1:0] 计数 3→2→1→0
  Word 0: CR_DOUBLE[63:48] ← SDRAM_DOUT
  Word 1: CR_DOUBLE[47:32] ← SDRAM_DOUT
  Word 2: CR_DOUBLE[31:16] ← SDRAM_DOUT
  Word 3: CR_DOUBLE[15:0]  ← SDRAM_DOUT

// 输出 64-bit：
  CR_DOUBLE[31:0]  = 前 8 像素 (CA4=0)
  CR_DOUBLE[63:32] = 后 8 像素 (CA4=1)
```

### 7.3 68k ROM 读取地址计算

```
casez ({CD_RD_SDRAM_SIG, ~nROMOE, ~nPORTOE})
  3'b1zz: SDRAM_ADDR <= CD_REMAP_TR_ADDR[24:1]     // CD 模式重映射
  3'b01z: SDRAM_ADDR <= {5'b0_0010, M68K_ADDR[19:1]}  // P1 ROM @$0200000
  3'b001: SDRAM_ADDR <= P2ROM_OFFSET + P2ROM_ADDR      // P2 ROM @$0300000+
  default:
    SYSTEM_CDx ? {6'b0_0000_0, M68K_ADDR[18:1]}       // System ROM (CD)
              : {8'b0_0000_000, M68K_ADDR[16:1]}       // System ROM (卡带)
```

### 7.4 SDRAM 物理控制器 (`sdram.sv`, 308行)

```
时序状态机（简化为 6 个关键状态）：

STATE_STARTUP → 初始化序列：
  PRECHARGE all → AUTO_REFRESH ×2 → LOAD_MODE
  Mode 寄存器 = {3'b000, 1'b1(无写burst), 2'b00, CAS=2, 1'b0(顺序), 3'b010(burst=4)}

STATE_IDLE → 等待请求：
  收到 rd/wr 请求 → STATE_WAIT
  收到 refresh 请求 → STATE_RFSH

STATE_WAIT → ACTIVATE（激活行）：
  发送 ACTIVE 命令 + 行地址

STATE_RW → READ 或 WRITE：
  发送 READ/WRITE 命令 + 列地址
  READ: 设置 data_ready_delay[CAS_LATENCY]（2 周期后数据就绪）
  WRITE: 立即 ready=1

STATE_IDLE_4..1 → 等待 tRP/tRAS：
  填充等待周期满足 SDRAM 时序要求

CAS 延迟流水线：
  data_ready_delay[2:0] 移位寄存器
  CMD_READ → delay[2]=1 → 移位2次 → delay[0]=1 → ready=1 + 锁存数据

SDRAM 物理引脚：
  SDRAM_A[12:0]  → 行/列地址（13-bit）
  SDRAM_BA[1:0]  → Bank 选择（4 个 bank）
  SDRAM_DQ[15:0] → 双向数据（16-bit）
  SDRAM_DQML/H   → 字节掩码
  SDRAM_nCS/nRAS/nCAS/nWE → 命令编码
```

### 7.5 双 SDRAM 地址分配

```verilog
// 根据容量标志计算选择位
sdr_pri_128_64 <= {~sdram_sz[14] & &sdram_sz[1:0],
                   ~sdram_sz[14] & sdram_sz[1]};

// 地址路由
sdr_pri_sel = (~sdram_addr[26] | sdr_pri_128_64[1])
            & (~sdram_addr[25] | sdr_pri_128_64[0]);

// sdr_pri_sel=1 → 主 SDRAM，=0 → 副 SDRAM
sdram_dout = sdr_pri_sel ? sdram1_dout : sdram2_dout;
sdram_ready = sdram2_ready & sdram1_ready;  // 两者都就绪

// 分配策略：
  128MB 单片：全走主 SDRAM（sdr_pri_sel 始终为 1）
  64+64MB：addr[26]=0 走主，=1 走副
  32+32MB：addr[25]=0 走主，=1 走副
```

---

## 八、neogeo.sv 顶层模块剩余分析

### 8.1 PLL 动态重配置（330-410行）

```
PLL 输出：50MHz → 96MHz (CLK_96M) + 48MHz (CLK_48M)

MVS/AES 切换时动态调频：
  MVS 参数: 2576980378 → 编码特定分数乘法因子
  AES 参数: 2865308404 → 略有不同的频率

重配置状态机（3-bit state）：
  State 1: cfg_address=0, 写入复位命令
  State 5: cfg_address=7, 写入 MVS 或 AES 参数
  State 7: cfg_address=2, 写入生效命令

  使用 cfg_waitrequest 等待 PLL 重配置完成
  sys_mvs 信号变化时触发（双级同步防止亚稳态）
```

### 8.2 ROM 加载完整流程（550-1069行）

```
ROM 索引常量：
  INDEX_SPROM   = 0    System ROM (BIOS)        → SDRAM $0000000
  INDEX_LOROM   = 1    LO ROM (查找表)          → BRAM (dpram)
  INDEX_SFIXROM = 2    SFIX ROM (系统字库)      → SDRAM $0020000
  INDEX_P1ROM_A = 4    P1 ROM 前半              → SDRAM $0200000
  INDEX_P1ROM_B = 5    P1 ROM 后半              → SDRAM $0280000
  INDEX_P2ROM   = 6    P2 ROM                   → SDRAM $0300000+
  INDEX_S1ROM   = 8    S1 ROM (Fix 图形)        → SDRAM $0080000
  INDEX_M1ROM   = 9    M1 ROM (Z80 程序)        → DDR3
  INDEX_VROMS   = 16+  V ROMs (ADPCM 采样)     → DDR3
  INDEX_CROMS   = 64+  C ROMs (精灵图形)        → SDRAM

ROM 大小掩码自动检测：
  加载时用 OR 累积所有写入地址 → 得到 2^n-1 掩码
  CROM_MASK |= CROM_LOAD_ADDR
  V1ROM_MASK |= VROM_LOAD_ADDR
  运行时地址 AND 掩码 → 实现地址环绕

CROM_START 自动计算：
  跟踪 P ROM 加载的最高地址
  C ROM 从 P ROM 之后开始
  默认值 = 3（$0600000）

精灵数据地址重排：
  CROM_LOAD_ADDR = {ioctl_addr[25:0], 1'b0}
                 + {(ioctl_index[7:1]-INDEX_CROMS[7:1]), 18'h0, ioctl_index[0], 1'b0}
  效果：奇偶 ROM 文件（C1/C2, C3/C4...）交织存储到连续地址
```

### 8.3 音频混合输出（2045-2071行）

```verilog
// ADPCM 数据就绪门控
adpcm_en <= ADPCMA_DATA_READY & ADPCMB_DATA_READY;

// YM2610 实例化
jt10 YM2610(
    .cen((CLK_EN_4M_P | CLK_EN_4M_N) & adpcm_en),  // 数据未就绪则暂停
    .snd_enable(~status[28:25]),   // FM/ADPCMA/ADPCMB/PSG 独立开关
    .ch_enable(~status[62:57]),    // ADPCMA 6 通道独立开关
    ...
);

// 最终混音
snd_mix_l = $signed(snd_left) + $signed(CD_AUDIO_L);
snd_mix_r = $signed(snd_right) + $signed(CD_AUDIO_R);
AUDIO_L = snd_mix_l[16:1];  // 17→16 bit（截取高 16 位）
AUDIO_R = snd_mix_r[16:1];
AUDIO_MIX = status[6:5];    // OSD 立体声混合模式
```

### 8.4 VRAM 实例化

```
Fast VRAM:  spram #(11,16) — 2K×16-bit，精灵 Y 坐标/尺寸
Slow VRAM:  spram #(15,16) — 32K×16-bit，精灵属性表
Palette RAM: spram #(13,16) — 8K×16-bit，{PALBNK, PAL_RAM_ADDR} 寻址
Z80 RAM:    dpram #(11) — 2KB，Z80 工作 RAM
LO ROM:     dpram #(16) — 64KB，地址查找表
```

### 8.5 nDTACK 调整逻辑

```verilog
// 根据访问类型选择不同的 DTACK 源
nDTACK_ADJ =
    // ROM 区域：等 SDRAM 数据就绪
    ~&{nSROMOE, nROMOE, nPORTOE, ~CD_EXT_RD, ...}
        ? ~PROM_DATA_READY | nDTACK
    // PCM 写入：等 DDR3 完成
    : (CD_TR_WR_PCM) ? ~ddram_dtack | nDTACK
    // PCM 读取：等 DDR3 完成
    : (CD_TR_RD_PCM) ? ~ADPCMA_RD_DTACK | nDTACK
    // 其他：使用 NEO-C1 原始 nDTACK
    : nDTACK;

// 效果：ROM/SDRAM 访问需要等数据就绪，片内 RAM 立即响应
```

---

## 九、BRAM 封装

### 9.1 dpram — 双端口 RAM

```verilog
module dpram #(ADDRWIDTH=8, DATAWIDTH=8) (
    clock_a, address_a, data_a, wren_a, q_a,  // Port A
    clock_b, address_b, data_b, wren_b, q_b   // Port B
);
// Altera MegaCore 包装
// 操作模式：BIDIR_DUAL_PORT
// 读期间写：返回新数据（NEW_DATA_NO_NBE_READ）
// 无输出寄存器（组合读取）
```

### 9.2 spram — 单端口 RAM

```verilog
module spram #(ADDRWIDTH=8, DATAWIDTH=8) (
    clock, address, data, wren, q
);
// 内部使用 dpram，两个端口绑定相同地址
```

---

## 十、信号命名约定

```
前缀/后缀      含义                          示例
n*             Active Low（低有效）           nRESET, nAS, nDTACK
*_EN           使能信号                       CLK_EN_68K_P
*_P / *_N      正沿/负沿                      CLK_EN_24M_P
*_D / *_d      延迟一拍（用于边沿检测）       nAS_d
*_REG          寄存器输出                     PAL_RAM_REG
*_MASK         地址掩码                       CROM_MASK
*_ADDR         地址信号                       M68K_ADDR
*_DATA         数据信号                       M68K_DATA
*_OUT          模块输出                       SDD_OUT
*_IN           模块输入                       SDD_IN
*_WR           写使能/写请求                  DL_WR
*_RD           读使能/读请求                  SDRAM_RD
*_RUN          操作进行中                     CROM_RD_RUN
*_REQ          请求标志                       VRAM_WRITE_REQ
*_DONE         操作完成                       VRAM_WRITE_DONE
*_READY        数据就绪                       PROM_DATA_READY
*_ZONE         地址区域选择                   nROM_ZONE
CLK_EN_*       时钟使能                       CLK_EN_4M_P

信号有效电平速查：
  0 = 有效的信号：nRESET, nAS, nLDS, nUDS, nDTACK,
                   nROMOE, nPORTOE, nSROMOE, nBITW0/1,
                   nSDZ80R/W/CLR, nCRDO/W/C, nPAL_WE
  1 = 有效的信号：M68K_RW(=1读), CLK_EN_*, DOTA/DOTB,
                   PROM_DATA_READY, DL_WR, SDRAM_RD/WR
```

---

> 本文档与 ARCHITECTURE.md（高层概览）和 ARCHITECTURE_DETAIL.md（子系统分析）配合阅读，
> 三份文档共同覆盖 NeoGeo MiSTer FPGA 核心的所有 RTL 模块和实现细节
