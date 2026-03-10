# NeoGeo MiSTer FPGA 核心 — 详细补充文档

> 基于完整源码逐行分析，补充 ARCHITECTURE.md 中的细节

---

## 一、顶层模块 `neogeo.sv` 完整解剖（2258行）

### 1.1 时钟体系（详细）

```
CLK_50M (DE10-Nano 板载晶振)
    │
    ├──→ PLL ──→ CLK_96M (96MHz, SDRAM 控制器时钟)
    │         └→ CLK_48M (48MHz, 系统主时钟 = clk_sys)
    │
    └──→ PLL_CFG (可动态重配置，MVS/AES 切换时微调频率)
              MVS: 参数 2576980378
              AES: 参数 2865308404

CLK_48M 分频链（在 clocks.v 中实现）：
    CLK_48M ──÷2──→ CLK_EN_24M (24MHz 使能脉冲)
              ──÷2──→ CLK_68KCLK (12MHz, 68000 时钟)
              ──÷4──→ CLK_12M
              ──÷8──→ CLK_6MB (6MHz, 像素时钟)
              ──÷16──→ CLK_3M
              ──÷8──→ CLK_1HB

Z80 时钟：
    CLK_48M ──÷12──→ CLK_EN_4M_P/N (4MHz, Z80 时钟使能)
```

**关键设计：所有模块共用 CLK_48M，通过 `CLK_EN_*` 使能信号模拟不同频率。**
这避免了多时钟域带来的亚稳态问题。例如 68000 CPU 不是真的跑在 12MHz 时钟上，而是在 48MHz 时钟的每 4 个周期中，只在 `CLK_EN_68K_P` 为高的那个周期执行。

`clocks.v` 中的核心逻辑：
```verilog
// 68000 时钟使能：24MHz 下降沿时翻转 CLK_68KCLK
always @(posedge CLK or negedge nRESETP)
    if (!nRESETP) CLK_68KCLK <= 1'b0;
    else if (CLK_EN_24M_P) CLK_68KCLK <= ~CLK_68KCLK;

// 使能脉冲：仅在 CLK_68KCLK 即将翻转的那个 48MHz 周期为高
assign CLK_EN_68K_P = ~CLK_68KCLK & CLK_EN_24M_P;  // 上升沿使能
assign CLK_EN_68K_N =  CLK_68KCLK & CLK_EN_24M_P;  // 下降沿使能

// 3-bit 分频计数器，从 24MHz 产生 12M/6M/3M
always @(posedge CLK, negedge nRESETP)
    if (!nRESETP) CLK_DIV <= 3'b100;
    else if (CLK_EN_24M_N) CLK_DIV <= CLK_DIV + 1'b1;
```

### 1.2 复位逻辑

```verilog
// TRASH_ADDR 计数器：复位后从 0 计数到 32767
// 计数期间 nRESET 保持低电平，同时用 TRASH_ADDR 作为地址
// 清零所有工作 RAM（利用双端口 RAM 的 B 端口写入）
reg [14:0] TRASH_ADDR;
always @(posedge CLK_48M) begin
    nRESET <= &TRASH_ADDR;  // 计满后释放复位
    if(CLK_EN_24M_N && ~&TRASH_ADDR) TRASH_ADDR <= TRASH_ADDR + 1;
    if(status[0] | status[14] | buttons[1] | bk_loading | RESET)
        TRASH_ADDR <= 0;     // 触发复位
end
```
巧妙之处：复位期间同时清零 RAM，不需要额外的初始化逻辑。

### 1.3 HPS 通信（ROM 加载）

ROM 通过 HPS（ARM 侧）加载到 FPGA 的存储中：

```
HPS ──→ hps_io ──→ ioctl_wr / ioctl_addr / ioctl_dout / ioctl_index
                          │
                          ├── ioctl_index 决定写入目标：
                          │   0  → System ROM (BIOS)     → SDRAM
                          │   1  → LO ROM                → BRAM (dpram)
                          │   2  → SFIX ROM              → SDRAM
                          │   4  → P1 ROM 前半           → SDRAM
                          │   5  → P1 ROM 后半           → SDRAM
                          │   6  → P2 ROM                → SDRAM
                          │   8  → S1 ROM (Fix 图形)     → SDRAM
                          │   9  → M1 ROM (Z80 程序)     → DDR3
                          │   16+ → V ROMs (ADPCM 采样)  → DDR3
                          │   64+ → C ROMs (精灵图形)    → SDRAM
                          │   10  → MEMCP (内存拷贝命令)
                          │
                          └── ioctl_addr_offset 计算实际 SDRAM 写入地址
```

**SDRAM 地址偏移计算（关键映射逻辑）：**
```verilog
localparam SPROM_OFFSET   = 27'h0000000;  // BIOS
localparam SFIXROM_OFFSET = 27'h0020000;  // 系统 Fix
localparam S1ROM_OFFSET   = 27'h0080000;  // 游戏 Fix
localparam P1ROM_A_OFFSET = 27'h0200000;  // P1 前半
localparam P1ROM_B_OFFSET = 27'h0280000;  // P1 后半
localparam P2ROM_OFFSET   = 27'h0300000;  // P2
// C ROM 起始地址 = CROM_START（动态计算，紧跟 P ROM 后面）
```

**ROM 大小掩码自动检测：**
```verilog
// 每次写入时，用 OR 运算累积地址的最高有效位
if(ioctl_index >= INDEX_CROMS) CROM_MASK <= CROM_MASK | CROM_LOAD_ADDR;
// 最终 CROM_MASK 变成 2^n - 1 的掩码，用于地址环绕
```

### 1.4 存储架构全貌

```
┌───────────────────────────────────────────────────────┐
│                    SDRAM (32MB)                        │
│  运行在 96MHz，由 sdram_mux.sv 仲裁                    │
│                                                       │
│  0x0000000  System ROM / SFIX ROM  (256KB)            │
│  0x0080000  S1 ROM (Fix 图形)      (512KB)            │
│  0x0200000  P1 ROM (68k 程序)      (1MB+)             │
│  0x0300000  P2 ROM (程序扩展)      (可达 4MB)          │
│  0x0800000+ C ROM  (精灵图形)      (可达 24MB)         │
└───────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────┐
│                    DDR3 (通过 HPS)                      │
│  高延迟但大容量，由 ddram.sv 管理                       │
│                                                       │
│  V ROMs (ADPCM-A 采样)        (可达 32MB)              │
│  V ROMs (ADPCM-B 采样)        (可达 32MB)              │
│  M1 ROM (Z80 程序)            (最大 512KB)             │
└───────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────┐
│               FPGA 片内 BRAM (约 385KB)                │
│                                                       │
│  68k Work RAM     64KB (dpram, 分 WRAML + WRAMU)      │
│  Z80 RAM          2KB  (spram)                         │
│  Slow VRAM        64KB (spram, 精灵属性表)              │
│  Fast VRAM        4KB  (spram, 精灵 Y 坐标/大小)       │
│  Palette RAM      16KB (spram, 4096 色 × 2 bank)      │
│  LO ROM           64KB (dpram, 地址查找表)              │
│  Backup RAM       64KB (dpram, 存档)                   │
│  Memory Card      8KB  (dpram)                        │
└───────────────────────────────────────────────────────┘
```

---

## 二、68000 地址空间完整映射

由 `neo_c1.v`（地址解码）+ `neo_e0.v`（BIOS 地址重映射）实现：

```
68000 地址        选通信号              内容
─────────────────────────────────────────────────
$000000-$0FFFFF   nROMOE              P1 ROM (游戏程序，1MB)
$100000-$10FFFF   nWRL/nWRU           Work RAM (64KB)
$110000-$1FFFFF                       Work RAM 镜像
$200000-$2FFFFF   nPORTOE/nPORTWE     Port Zone (P2 ROM bankswitch 窗口)
$300000-$3FFFFF                       （通常未使用）
$400000-$7FFFFF   nPAL/nPAL_WE        Palette RAM (16KB，大量镜像)
$800000-$BFFFFF   nSROMOE             System ROM (BIOS, 128KB, 有镜像)
$C00000-$C0000F   nLSPOE/nLSPWE       LSPC 视频寄存器（8个寄存器）
$D00000-$D0FFFF   nSRAMOE/nSRAMWE     Backup RAM (64KB, MVS only)
$D10000-$D107FF   nCRDO/nCRDW         Memory Card (2KB)
$300001           nDIPRD0             手柄 P1 输入
$340001           nDIPRD1             手柄 P2 输入
$380001           nBITW0              系统状态写
$3A0001           nBITW1/nBITWD0      系统锁存/DIP 读
$3C0000-$3C000F   nSDZ80R/W           Z80 通信寄存器
```

**NEO-E0 BIOS 地址重映射：**
```verilog
// 通过 nVEC 信号控制 BIOS 向量区映射：
// nVEC=0 时：$000000-$0000FF 映射到 System ROM（读取中断向量）
// nVEC=1 时：$000000-$0000FF 映射到 P1 ROM（正常运行）
```

**nDTACK 调整逻辑（关键，解决 SDRAM 延迟问题）：**
```verilog
wire nDTACK_ADJ = 
    // ROM 区域：等 SDRAM 数据就绪后才返回 DTACK
    ~&{nSROMOE, nROMOE, nPORTOE, ~CD_EXT_RD, ~CD_TR_RD_FIX, ~CD_TR_RD_SPR}
        ? ~PROM_DATA_READY | nDTACK
    // PCM 写入：等 DDR3 写完成
    : (CD_TR_WR_PCM) ? ~ddram_dtack | nDTACK
    // PCM 读取：等 DDR3 读完成
    : (CD_TR_RD_PCM) ? ~ADPCMA_RD_DTACK | nDTACK
    // 其他：使用原始 nDTACK（由 NEO-C1 产生）
    : nDTACK;
```

---

## 三、SDRAM 多路复用器详解 (`sdram_mux.sv`)

### 3.1 访问请求源（按优先级）

| 优先级 | 请求源 | 触发条件 | 数据宽度 | 说明 |
|--------|--------|----------|----------|------|
| 最高 | ROM 加载 | `DL_EN & DL_WR` | 16-bit 写 | HPS 写入 ROM |
| 高 | 68k 程序 | nROMOE/nSROMOE/nPORTOE ↓ | 16-bit 读 | CPU 取指取数 |
| 中 | 精灵图形 | PCK1B 上升沿 | 4×16-bit burst | 16像素/次 |
| 中 | Fix 图形 | PCK2B 上升沿 | 16-bit 读 | 4像素/次 |
| 低 | DMA | DMA_RUNNING | 16-bit 读/写 | Neo CD 传输 |
| 最低 | 刷新 | REFRESH_EN | — | SDRAM 刷新 |

### 3.2 精灵图形 Burst 读取

```verilog
// C ROM 地址由 CROM_ADDR[26:0] 给出
// 一次 burst 读 4 个 16-bit word = 64-bit
// 包含一个 8 像素行的完整 4 位平面数据
//
// CR_DOUBLE[63:0] 存储 16 像素：
//   CR_DOUBLE[31:0]  = 前 8 像素 (CA4=0)
//   CR_DOUBLE[63:32] = 后 8 像素 (CA4=1)
//
// ZMC2 解码器根据 CA4 选择哪 8 像素送入
```

### 3.3 Fix 图形读取

```verilog
// S ROM 数据在加载时被重排：
// 原始：列优先存储（column 0 line0-7, column 1 line0-7, ...）
// 重排后：行优先存储（line 0 col0-3, line 1 col0-3, ...）
//
// 这样一次 16-bit SDRAM 读取就能得到一行的 4 个像素
// S2H1 信号选择高/低字节（前2像素 vs 后2像素）
```

### 3.4 双 SDRAM 支持

```verilog
// 根据 SDRAM 容量和地址决定走哪块：
wire sdr_pri_sel = (~sdram_addr[26] | sdr_pri_128_64[1]) 
                 & (~sdram_addr[25] | sdr_pri_128_64[0]);

// 128MB SDRAM: 全部走主 SDRAM
// 64MB + 64MB: 按地址高位分配
// 32MB + 32MB: 类似
```

---

## 四、视频系统深入分析

### 4.1 每帧渲染流程

```
一帧 = 264 扫描线 (NTSC) 或 312 (PAL)
每扫描线 = 384 像素时钟 (6MHz) = 64μs
可见区域 = 320×224 (NTSC) 或 320×256 (PAL)

每条扫描线时间线：
┌── HBlank 前期 (约 64 像素时钟) ─────────────────┐
│  Fast Cycle: 从 Fast VRAM 逐个读精灵 Y/Size     │
│  判断哪些精灵与当前扫描线相交                     │
│  记录可见精灵列表                                │
├── HBlank 后期 ──────────────────────────────────┤
│  Slow Cycle: 从 Slow VRAM 读可见精灵的属性       │
│  （tile 编号、调色板、水平位置、翻转标志）        │
│  SDRAM Burst: 读取精灵 C ROM 图形数据            │
│  ZMC2: 解码位平面为像素                          │
│  写入 Line Buffer                               │
├── 可见期间 (320 像素时钟) ───────────────────────┤
│  从 Line Buffer 读出上一行的精灵像素             │
│  实时从 SDRAM 读 Fix 层图形                      │
│  NEO-B1 合并精灵层 + Fix 层                      │
│  查 Palette RAM 得到 RGB                        │
│  输出像素                                       │
└── 下一行 ─────────────────────────────────────── ┘
```

### 4.2 LSPC2-A2 寄存器

```
地址偏移  名称              功能
$C00000   LSPC_ADDR         VRAM 地址寄存器
$C00002   LSPC_DATA         VRAM 数据端口（自动递增）
$C00004   LSPC_INCR         VRAM 地址递增值
$C00006   LSPC_MODE         模式寄存器
                             [15:3] = Timer 预设值
                             [2]    = Timer 中断使能
                             [1]    = Timer 重载模式
                             [0]    = Timer 停止
$C00008   LSPC_TIMERHIGH    Timer 高位预设
$C0000A   LSPC_TIMERLOW     Timer 低位预设
$C0000C   LSPC_IRQACK       中断确认
$C0000E   LSPC_TIMERMODE    Timer 控制
```

### 4.3 精灵属性表 (Slow VRAM)

```
每个精灵占 4 个 word（在不同的 VRAM bank 中）：

SCB1 ($0000-$6FFF, 共 1536 entries × 64 tiles):
  每个 tile 的图形编号（20-bit tile index）

SCB2 ($8000-$81FF):
  每个精灵的缩放参数
  [15:8] = 水平缩放
  [7:0]  = 垂直缩放

SCB3 ($8200-$83FF):
  [15:14] = 垂直翻转, 水平翻转
  [13:8]  = 调色板编号
  [5:0]   = 精灵链中 tile 数量（0=1 tile, 33=33 tiles）

SCB4 ($8400-$85FF):
  [15:7]  = X 坐标 (9-bit, 0-511)
  [6]     = Sticky bit（与前一精灵粘连）
```

### 4.4 NEO-B1 像素合并逻辑

```verilog
// 优先级从高到低：
// 1. Fix 层非透明像素 (FIXD != 0)
// 2. 精灵层非透明像素 (GAD/GBD != 0)
// 3. 背景色 (Palette 0, Color 0)
//
// 输出：12-bit 调色板地址 = {palette_index[5:0], color_index[3:0], bank}
```

### 4.5 颜色格式与 Dark Bit

```verilog
// Neo Geo 16-bit 颜色格式：
// Bit: 15  14  13  12  11-8  7-4   3-0
//       D   R0  G0  B0  R4-1  G4-1  B4-1
//
// D = Dark Bit（全局减暗 1 级）
// R0/G0/B0 = 各通道的 LSB（实现 6-bit 精度）
// R4-1/G4-1/B4-1 = 各通道的 MSB

wire [6:0] R6 = {1'b0, PAL_RAM_REG[11:8], PAL_RAM_REG[14], PAL_RAM_REG[11]} 
              - PAL_RAM_REG[15];  // 6-bit R 减去 Dark Bit
// R6[6] 是借位标志，如果为 1 说明结果为负 → 钳位到 0
wire [7:0] R8 = R6[6] ? 8'd0 : {R6[5:0], R6[4:3]};  // 扩展到 8-bit
```

---

## 五、声音系统详细

### 5.1 YM2610 (jt10) 架构

```
Z80 写寄存器 ──→ jt12_mmr (寄存器映射)
                    ├──→ jt12_pg  (相位发生器) ─→ jt12_op (算子) ─→ FM 输出
                    ├──→ jt12_eg  (包络发生器) ─┘
                    ├──→ jt10_adpcm_drvA (ADPCM-A × 6ch) ──→ 采样输出
                    ├──→ jt10_adpcmb     (ADPCM-B × 1ch) ──→ 采样输出
                    └──→ jt49            (SSG/PSG × 3ch)  ──→ 方波输出
                              ↓
                         jt12_mixer (混音) ──→ snd_left / snd_right
```

### 5.2 ADPCM 数据读取（DDR3）

ADPCM 采样数据存储在 DDR3 中（因为太大放不进 SDRAM）：

```verilog
// ADPCM-A 读取触发：nSDROE 下降沿
if (ADPCMA_OE_SR == 2'b10) begin
    ADPCMA_READ_REQ <= ~ADPCMA_READ_REQ;  // 翻转请求信号触发读取
    ADPCMA_ADDR_LATCH <= {ADPCMA_BANK, ADPCMA_ADDR} & V1ROM_MASK;
    ADPCMA_ACK_COUNTER <= 8'd128;  // 必须在 128 个 96MHz 周期内返回
end

// ADPCM-B 读取触发：nSDPOE 下降沿
if (ADPCMB_OE_SR == 2'b10) begin
    ADPCMB_READ_REQ <= ~ADPCMB_READ_REQ;
    ADPCMB_ADDR_LATCH <= {~use_pcm, ADPCMB_ADDR & ADPCMB_MASK};
    ADPCMB_ACK_COUNTER <= 11'd1580;  // 更宽裕的时间窗口
end
```

**DDR3 读写仲裁 (ddram.sv)：**
```
3 个读端口 + 1 个写端口：
  读 1: ADPCM-A 采样数据
  读 2: ADPCM-B 采样数据
  读 3: Z80 M1 ROM 程序
  写:   ROM 加载 / CD PCM 写入
```

### 5.3 YM2610 时钟使能

```verilog
// YM2610 在 4MHz 下运行，但需要等待 ADPCM 数据就绪
// adpcm_en 只在 ADPCM-A 和 ADPCM-B 数据都就绪时为高
reg adpcm_en;
always @(posedge CLK_48M) begin
    reg en;
    en <= ADPCMA_DATA_READY & ADPCMB_DATA_READY;
    adpcm_en <= en;
end

jt10 YM2610(
    .clk(CLK_48M), 
    .cen((CLK_EN_4M_P | CLK_EN_4M_N) & adpcm_en),  // 数据没准备好就暂停
    ...
);
```

### 5.4 最终音频输出

```verilog
// YM2610 输出 + CD 音轨混合
wire [16:0] snd_mix_l = $signed(snd_left) + $signed(CD_AUDIO_L);
wire [16:0] snd_mix_r = $signed(snd_right) + $signed(CD_AUDIO_R);
assign AUDIO_L = snd_mix_l[16:1];  // 截取高 16 位
assign AUDIO_R = snd_mix_r[16:1];

// 各声道可独立开关（通过 OSD 菜单）：
// snd_enable = ~status[28:25]  → FM/ADPCMA/ADPCMB/PSG
// ch_enable  = ~status[62:57]  → ADPCMA 通道 1-6
```

---

## 六、I/O 系统完整信号流

### 6.1 NEO-C1 (`neo_c1.v`) — 主地址解码器

```
输入：
  M68K_ADDR[21:17] — 地址高位（决定大区域）
  A22Z, A23Z       — 地址最高位（由 NEO-E0 处理）
  nLDS, nUDS, RW, nAS — 68000 控制信号

输出（active low 选通信号）：
  ROM 区域:    nROMOEL, nROMOEU, nSROMOEL, nSROMOEU
  Port 区域:   nPORTOEL, nPORTOEU, nPORTWEL, nPORTWEU
  Work RAM:    nWRL, nWRU, nWWL, nWWU
  Backup RAM:  nSRAMOEL, nSRAMOEU, nSRAMWEL, nSRAMWEU
  LSPC:        nLSPOE, nLSPWE
  Palette:     nPAL, nPAL_WE
  Memory Card: nCRDO, nCRDW, nCRDC
  手柄/DIP:    nDIPRD0, nDIPRD1
  系统控制:    nBITW0, nBITW1, nBITWD0
  Z80 通信:    nSDZ80R, nSDZ80W, nSDZ80CLR
  nDTACK:      数据确认（考虑等待周期）
```

### 6.2 68000 ↔ Z80 通信桥

```
                  NEO-C1                    NEO-D0
68000 ──写──→ nSDZ80W ──→ 通信寄存器 ──→ Z80 可读 (nSDZ80R)
68000 ←─读──  SDD_RD_C1 ←── 通信寄存器 ←── Z80 写 (nSDW)

68000 ──→ nSDZ80CLR ──→ 清除 Z80 NMI ──→ Z80_nNMI 拉高
68000 ──→ NMI 触发  ──→ Z80_nNMI 拉低 ──→ Z80 响应 NMI

典型流程（播放声音）：
1. 68000 写命令字节到 $3C0000
2. 68000 触发 Z80 NMI
3. Z80 NMI 中断处理：读取命令字节
4. Z80 根据命令操作 YM2610 寄存器
5. YM2610 开始播放
```

### 6.3 系统锁存 (`syslatch.v`)

```verilog
// 68000 写入 $3A0001 时，根据 ADDR[4:1] 设置不同的系统标志：
// ADDR[4:1]=0: SHADOW   — 画面变暗（模拟 RGB 减半）
// ADDR[4:1]=1: nVEC     — 中断向量映射切换
// ADDR[4:1]=2: nCARDWEN — 记忆卡写使能
// ADDR[4:1]=3: CARDWENB — 记忆卡写使能（双重保护）
// ADDR[4:1]=4: nREGEN   — 寄存器使能
// ADDR[4:1]=5: nSYSTEM  — 系统模式（MVS only）
// ADDR[4:1]=6: nSRAMWEN — 备份 RAM 写使能（注意取反，见 MVS 原理图 page 3）
// ADDR[4:1]=7: PALBNK   — 调色板 bank 选择（0/1，双 bank 切换）

// 内部用 8-bit 寄存器 SLATCH[7:0] 存储
// 复位时：nBITW1=1 → 全部清零；nBITW1=0 → Demux 模式
// 运行时：nBITW1=0 → Latch 模式，M68K_ADDR[4] 作为数据位写入
```

### 6.4 看门狗定时器 (`watchdog.v`)

```
功能：如果 68000 停止响应（死机），自动触发系统复位

核心机制：
  - 4-bit 递增计数器 WDCNT[3:0]
  - WDCLK 上升沿驱动计数
  - 当 WDCNT[3]=1 时，nRESET 和 nHALT 被拉低
  - 游戏必须定期写 $300001 来"喂狗"（清零计数器）

喂狗地址解码：
  $300001 = nLDS=0, RW=1, A23=A22=0, ADDR[21:20]=11, ADDR[19:17]=000
  → WDRESET 信号有效，WDCNT 清零

复位行为：
  nRESET = nRST & ~WDCNT[3]  （开漏输出，外部 4.7kΩ 上拉）
  nHALT = nRESET              （同步，均为开漏）
  复位约持续 8 个 WDCLK 周期，释放再经 8 个周期稳定
```

### 6.5 实时时钟 (`upd4990.v`)

```
模拟 μPD4990 日历时钟芯片，MVS 用于显示当前时间

输入：
  rtc[64:0] — MiSTer 框架提供的 BCD 格式时间
  CS, OE    — 片选和输出使能（在 Neo Geo 中始终为低）
  DATA_CLK  — 串行时钟
  DATA_IN   — 串行数据输入
  STROBE    — 命令锁存

输出：
  DATA_OUT  — 串行数据输出（48-bit 移位寄存器）
  TP        — 测试脉冲输出（可选频率）

时间数据格式（48-bit 移位寄存器）：
  {Year[7:0], Month_hex[3:0], 0, Weekday[2:0], Day[7:0], 00, Hour-Min-Sec[21:0]}

命令集（4-bit CMD_REG，STROBE 上升沿锁存）：
  0x0: Hold      — 冻结移位
  0x1: Shift     — 允许 DATA_CLK 移位
  0x2: Set time  — 设置时间（未实现）
  0x3: Read time — 加载 TIME_DATA 到移位寄存器
  0x4-0x7: TP 频率选择（64Hz / 256Hz / 2048Hz / 4096Hz / 间隔定时器）
  0xC: 重置间隔标志
  0xD: 启动秒定时器
  0xE-0xF: 停止秒定时器

频率分频链：
  12MHz → DIV9(÷366) → 32768Hz → DIV15 → 64Hz → DIV6 → 2Hz（半秒计数器）
```

### 6.6 NEO-G0 总线复用器 (`neo_g0.v`)

```
功能：控制 68000 数据总线对记忆卡和调色板 RAM 的访问

真值表：
  G0  G1  DIR    操作               M68K_DATA    WE
  0   0   0     同时写（不应发生）   Hi-Z          0
  0   0   1     读记忆卡            CDD数据       1
  0   1   0     写记忆卡            Hi-Z          1
  0   1   1     读记忆卡            CDD数据       1
  1   0   0     写调色板            Hi-Z          0
  1   0   1     读调色板            PC数据        1
  1   1   x     空闲               Hi-Z          1

核心逻辑（仅 3 行 assign）：
  READ_DATA = G0 ? PC : CDD           // G0 选择数据源
  M68K_DATA = (~(G0&G1) & DIR) ? READ_DATA : Hi-Z
  WE = G1 | DIR
```

### 6.7 Z80 ROM 地址转换 (`zmc.v`)

```
功能：实现 Z80 64KB 地址空间对大容量 ROM 的 bankswitching

4 个滑动窗口：
  Z80地址范围      窗口寄存器     位宽   默认值   实际ROM范围
  $F000-$F7FF      WINDOW_0      8-bit   0x1E    全部替换 MA[18:11]
  $E000-$EFFF      WINDOW_1      7-bit   0x0E    替换 MA[18:12]，保留 SDA[11]
  $C000-$DFFF      WINDOW_2      6-bit   0x06    替换 MA[18:13]，保留 SDA[12:11]
  $8000-$BFFF      WINDOW_3      5-bit   0x02    替换 MA[18:14]，保留 SDA[13:11]
  $0000-$7FFF      直通                          SDA_U[15:11] 直接输出

窗口更新：
  Z80 写 I/O 端口 → nSDRD0 上升沿 → SDA_L[1:0] 选择窗口 → 加载 SDA_U 高位
```

### 6.8 PCM 声音地址复用器 (`pcm.v`)

```
功能：复用 ADPCM-A 和 ADPCM-B 两个声音通道的 ROM 地址

ADPCM-A（ROM 通道，SDRAD 总线）：
  nSDRMPX 上升沿 → 锁存低字节 RALATCH[9:0]
  SDRMPX 上升沿  → 锁存高字节 RALATCH[23:10]

ADPCM-B（PCM 通道，SDPAD 总线）：
  nSDPMPX 上升沿 → 锁存低字节 PALATCH[11:0]
  SDPMPX 上升沿  → 锁存高字节 PALATCH[23:12]

地址输出选择：
  A[23:0] = nSDPOE ? RALATCH : PALATCH

数据回读：
  SDRAD = nSDROE ? Hi-Z : RDLATCH    // 从 DDR3 返回的 ADPCM-A 数据
  SDPAD = nSDPOE ? Hi-Z : PDLATCH    // 从 DDR3 返回的 ADPCM-B 数据
```

### 6.9 Z80 总线控制器 (`z80ctrl.v`)

```
功能：Z80 外设片选和 NMI 中断逻辑

I/O 端口地址解码（SDA_L[3:2]）：
  端口范围    SDA_L[3:2]  设备
  $00-$03     00          nSDZ80R/W/CLR（与 NEO-C1 通信）
  $04-$07     01          n2610RD/WR（YM2610 声音芯片）
  $08-$0B     10          nSDRD0（ROM bankswitch，见 zmc.v）
  $0C-$0F     11          nSDRD1（M1 ROM 扩展）

NMI 中断逻辑：
  1. Z80 写 I/O 端口 $08 → nNMI_SET 触发
  2. SDA_L[4] 决定 NMI 使能（nNMI_EN）
  3. 写端口 $0C 且 nNMI_EN=1 → nZ80NMI 拉低
  4. nSDZ80R 上升沿（68000 读通信寄存器）→ 释放 NMI

内存片选：
  nSDROM  = SDA_U[15:11] 全高（Z80 ROM 区域）
  nZRAMCS = ~nSDROM（Z80 RAM 位于 $F800-$FFFF）
  nSDMRD  = nMREQ | nSDRD
  nSDMWR  = nMREQ | nSDWR
```

---

## 七、保护芯片详解

### 7.1 NEO-SMA 加密 Bankswitch (`neo_sma.sv`)

```
功能：游戏特定的 P2 ROM bankswitch + LFSR 随机数发生器
适用游戏：KOF'99, Garou, Garou Hidden, Metal Slug 3, KOF 2000

架构：
  P2_ADDR[23:0] = bank[23:0] + {M68K_ADDR, 1'b0}
  当 TYPE >= 3 时启用

三类特殊地址：
  BANK_ADDR — 写入此地址更新 bank 寄存器
  ID_ADDR   — 读取返回固定 ID（$7F223 → 返回 0x9A37）
  RNG_ADDR  — 读取返回伪随机数

游戏特定配置：
  TYPE  游戏          BANK_ADDR   RNG_ADDR1   RNG_ADDR2   有效段数
  3     KOF'99       $7FFF8      $7FFFC      $7FFFD      32
  4     Garou        $7FFE0      $7FFE6      $7FFF8      51
  5     Garou Hidden $7FFE0      $7FFE6      $7FFF8      51
  6     Metal Slug 3 $7FFF2      无          无          44
  7     KOF 2000     $7FFF6      $7FFEC      $7FFED      36

Bankswitch 工作流程：
  1. nPORTWE 下降沿 + 地址匹配 BANK_ADDR
  2. 从 M68K_DATA 中提取 6-bit 索引（位顺序因游戏而异）
     KOF'99:  {DATA[5], DATA[12], DATA[10], DATA[8], DATA[6], DATA[14]}
     Garou:   {DATA[12], DATA[14], DATA[6], DATA[7], DATA[9], DATA[5]}
     mslug3:  {DATA[9], DATA[3], DATA[6], DATA[15], DATA[12], DATA[14]}
     kof2000: {DATA[5], DATA[10], DATA[3], DATA[7], DATA[14], DATA[15]}
  3. 用索引查找 64 项查找表 → 得到 24-bit bank 基址

LFSR 随机数发生器：
  16-bit Galois LFSR，抽头位 {2,3,5,6,7,11,12,15}
  种子：0x2345（复位后首次读取时加载）
  每次 nPORTOE 下降沿移位一次
```

### 7.2 NEO-PVC 数据加密 (`neo_pvc.v`)

```
功能：ROM bankswitch + 内部 RAM + 颜色处理寄存器
适用游戏：KOF 2003, SvC Chaos 等后期 MVS 游戏

地址解码：
  RAM_ACC  = M68K_ADDR[19:13] 全 1 → 访问内部 4KB 双端口 RAM
  PORT_ACC = M68K_ADDR[19:5] 全 1  → 访问端口寄存器
  PORT_NO  = M68K_ADDR[4:1]        → 16 个端口寄存器

内部 RAM（4KB = 2×2K words）：
  双端口 dpram，分 RAML（低字节）和 RAMU（高字节）
  Port A: 68000 读写
  Port B: 复位时自动清零（CLR_ADDR 自增）

端口寄存器映射：
  PORT  读返回           写操作
  0     —                 颜色 R/G/B + 饱和度（UR[4:0], UG[0], UB[0], US）
  1     {3'b0,UG,3'b0,UB}  颜色 G/B 高位
  2     {7'b0,US,3'b0,UR}  —
  4-5   —                 调色板颜色配置（PCOL 寄存器）
  6     PCOL              —
  8     {bank[7:0],0xA0}  bank 地址低字节
  9     bank[23:8]        bank 地址高/中字节

P2_ADDR = bank + {M68K_ADDR, 1'b0}

数据总线优先级：
  PORT_RD ? PORT_DO : RAM_ACC ? RAM_DO : PROM_DATA
```

### 7.3 NEO-CMC 视频域 Bankswitch (`neo_cmc.v`)

```
功能：在像素时钟域内动态切换图形 bank（不同于 SMA/PVC 的 CPU 域）
适用游戏：Metal Slug 4+ 等后期游戏

两种模式（TYPE[1:0]）：

TYPE[0]=1 — 行级 Banking：
  触发条件：ADDR[10:8]=7 且 PBUS[14:12]=0
  内部状态：32 项 map 表（每项 4-bit: {valid[1:0], bank[1:0]}）
  5-bit line 计数器逐行推进
  当 map[line] 有效且未跳过时 → BANK <= map[line][1:0]

  map 表更新：
    偶数 word 地址 {ADDR[6],ADDR[0],ADDR[10:8]}=5 时
    高半部分（$7580-$75BE）写入 valid/bank 位
    低半部分（$7500-$753E）写入跳过标志

TYPE[1]=1 — 调色板级 Banking：
  触发条件：ADDR[10:8]=5 且 PBUS[14:12] 全高
  80-bit banks 寄存器存储解码后的 bank 值
  BANK <= banks[{ADDR[10:5],1'b0} +:2]

特殊初始化：ADDR=$7E2 且 PBUS[14:12]=0 时重置（skip=0, line=0, BANK=1）
TYPE=0 时 BANK 强制为 0
```

### 7.4 游戏与保护芯片对照表

```
游戏                 保护芯片     TYPE/特征
─────────────────────────────────────────────
KOF'99               NEO-SMA      TYPE=3, 32段, 有RNG
Garou                NEO-SMA      TYPE=4, 51段
Garou Hidden         NEO-SMA      TYPE=5, 51段
Metal Slug 3         NEO-SMA      TYPE=6, 44段, 无RNG
KOF 2000             NEO-SMA      TYPE=7, 36段
KOF 2003             NEO-PVC      内部4KB RAM, 颜色加密
SvC Chaos            NEO-PVC      同上
Metal Slug 4+        NEO-CMC      TYPE[0]=1, 行级banking
其他后期游戏         NEO-CMC      TYPE[1]=1, 调色板级banking
多数早期游戏         无            标准 bankswitch (BNK[2:0])
```

---

## 八、CD 子系统详解 (`rtl/cd/`)

### 8.1 CD 系统顶层 (`cd.sv`, 811行)

```
模块名：cd_sys
功能：Neo CD 全部功能的中央协调器

子模块实例化：
  cd_drive DRIVE  — CD 驱动模拟
  lc8951  LC8951  — CD 解码器芯片
  cdda    CDDA    — CD 音轨播放

关键寄存器：
  REG_FF0002[15:0] — IRQ 控制标志
  REG_FF0004[11:0] — 中断使能
  DMA_ADDR_A[23:0] — DMA 源地址
  DMA_ADDR_B[23:0] — DMA 目标地址
  DMA_VALUE[31:0]  — DMA 填充值
  DMA_COUNT[31:0]  — DMA 传输计数
  DMA_MICROCODE[9] — 10-word 微码表
```

**DMA 状态机（14 态）：**
```
IDLE → BR（请求总线）→ WAIT_BG（等待授权）→ WAIT_AS（等待CPU释放）
  → START → 根据 DMA_MODE 选择：
    CACHE_RD / CACHE_RD_L  — 从 LC8951 缓存读取
    RD / RD_DONE           — 内存读取
    WR / WR_DONE           — 内存写入
  → DONE（计数递减，回到 START 或返回 IDLE）
```

**5 种 DMA 模式（由微码第 0 word 决定）：**
```
微码值         模式                     说明
$FF89/$FFC5    COPY_CD_WORD             LC8951 → DMA_ADDR_A (word)
$FC2D          COPY_CD_BYTE             LC8951 → DMA_ADDR_A (奇字节)
$FE6D/$FE3D    COPY_WORD                ADDR_A → ADDR_B (word)
$F2DD/$E2DD    COPY_BYTE                ADDR_A → ADDR_B (奇字节)
$FFCD/$FFDD    FILL_VALUE               用 DMA_VALUE 填充内存
```

**CD 寄存器映射（$FF00xx-$FF01xx）：**
```
地址      功能
FF0002    IRQ 控制（CD 通信/解码器中断使能位）
FF0004    中断使能（VBlank/Timer IRQ）
FF0061    DMA 启动（bit 6 触发微码执行）
FF0064-73 DMA 参数（ADDR_A, ADDR_B, VALUE, COUNT）
FF007E-8E DMA 微码表（10 words）
FF0105    上传区域选择
FF0111    精灵层使能
FF0115    Fix 层使能
FF0119    视频输出使能
FF0121-29 映射区域（SPR/PCM/Z80/FIX → 上传目标）
FF0141-49 取消映射
FF0163    CDD 通信输出
FF0165    HOCK 通信选通
FF016F    上传使能（同时关闭看门狗）
FF0181-83 驱动/Z80 复位控制
FF01A1-A3 精灵/PCM bank 选择
```

**扇区时序：**
```
CD_SPEED    每秒扇区数    周期数（96MHz时钟）
1X          75           MCLK/75  ≈ 1,288,951
2X          150          MCLK/150 ≈ 644,475
3X          225          MCLK/225 ≈ 429,650
4X          300          MCLK/300 ≈ 322,238

双 bank 扇区缓存：
  SECTOR_FILLED[2] — 两个缓存 bank 的填充状态
  CACHE_RD_BANK / CACHE_WR_BANK — 读/写 bank 选择
  CD_DATA_WR_READY = ~SECTOR_FILLED[~CACHE_RD_BANK]
```

### 8.2 CD 音轨播放 (`cdda.v`)

```
双循环缓冲架构：
  缓冲大小 = 2 × SECTOR_SIZE = 2 × 588 = 1176 个 32-bit word
  SECTOR_SIZE = 2352 bytes × 8 / 32 = 588 samples

数据流：
  DIN[15:0] → 写入缓冲 → LRCK 交替 → AUDIO_L[15:0] / AUDIO_R[15:0]

流控：
  WRITE_READY = (FILLED_COUNT ≤ BUFFER_AMOUNT - SECTOR_SIZE)
  维护独立的读/写指针和 FILLED_COUNT
```

### 8.3 CD 驱动模拟 (`drive.v`)

```
模拟 Sony CDD MCU 4-bit 串行接口

协议参数：
  比特率：clk_sys / 192 ≈ 250kHz
  帧格式：10 words × 4 nibbles = 40 nibbles/帧
  状态帧：40-bit（10 × 4-bit nibble）
  命令帧：40-bit（10 × 4-bit nibble）

2-bit 状态机：
  State 0 → 数据上总线，等待 HOCK 上升沿
  State 1 → HOCK 握手同步
  State 2 → 数据时钟输出

中断：
  CDD_nIRQ 以 64Hz 频率触发（~3906 个 250kHz 周期）
  脉冲宽度约 50% 占空比
  COMMAND_SEND 在帧结束时触发，通知 HPS 处理命令
```

### 8.4 LC8951 CD 解码器 (`lc8951.v`)

```
功能：模拟 Sanyo LC8951 CD-ROM 解码器芯片

寄存器集（16 个，通过 RS 位选择地址/数据模式）：

读寄存器：
  0  SBOUT    — 命令输出（未使用）
  1  IFSTAT   — 接口状态
       Bit 7: ~CMDI_FLAG（命令中断）
       Bit 6: ~DTEI_FLAG（传输结束中断）
       Bit 5: ~DECI_FLAG（解码/扇区就绪中断）
  2-3 DBC     — 数据字节计数器（12-bit）
  4-7 HEAD    — 扇区头（MSF/模式信息，4 bytes）

写寄存器：
  1  IFCTRL   — 接口控制（中断使能位）
  2-3 DBC     — 传输字节计数
  6  DTTRG    — DMA 触发
  7  DTACK    — 清除传输 IRQ
  10 CTRL0    — 解码器使能（bit 7 = DECEN）
  11 CTRL1    — 模式控制

中断逻辑：
  SECTOR_READY 上升沿 → DECI_FLAG 置位（若 DECEN=1）
  DMA_RUNNING 下降沿 → 释放 nDTBSY
  CDC_nIRQ = ~(CMDI&CMDIEN | DTEI&DTEIEN | DECI&DECIEN)
  中断超时：21,600 × (1/48MHz) ≈ 0.45ms 脉冲宽度
```

### 8.5 HPS 扩展接口 (`hps_ext.v`)

```
功能：通过 EXT_BUS 与 ARM 侧交换 CD 数据

EXT_BUS 信号映射：
  [15:0]  — 数据输出 (io_dout)
  [31:16] — 数据输入 (io_din)
  [32]    — 输出使能 (dout_en)
  [33]    — 选通 (io_strobe)
  [34]    — 使能 (io_enable)

命令接口：
  CD_GET (0x34) — 读取 CD 数据/状态
  CD_SET (0x35) — 写入 CD 命令

数据传输类型：
  send_data_type=0: CDDA 就绪状态
  send_data_type=1: CD 数据就绪状态
```

---

## 九、视频输出流水线完整分析

### 9.1 同步信号生成 (`videosync.v`)

```
像素计数器 PIXELC[8:0]：
  格式：{P15_Q[2:0], P50_Q[3:0], LSPC_1_5M, LSPC_3M}
  P50_Q: 4-bit 计数器，从 14 到 15 翻转（在 24MHz 下）
  P15_Q: 4-bit 计数器，溢出时从 0x5 重载
  总计：每扫描线 384 个像素时钟

光栅计数器 RASTERC[8:0]：
  格式：{J268_J269_Q[7:0], FLIP}
  J268_J269_Q: 8-bit 计数器，从 0x00 到 0xFF
  溢出重载值：
    NTSC (VMODE=0): 0xF8 → 总共 263 扫描线（0xFF-0xF8+1+255+1=263）
    PAL  (VMODE=1): 0xC8 → 总共 311 扫描线
  FLIP: 每场翻转一次（隔行扫描跟踪）

HSYNC 生成：
  通过 4 级移位寄存器链：R15→T116→S122→S116
  HSYNC = ~(S116_Q & ~T116_REG[1])
  产生每行约 1 像素宽的同步脉冲

VSYNC 生成：
  NTSC: VSYNC = RASTERC[8]
  PAL:  VSYNC = BLANK_PAL（光栅行 3-7 + FLIP=0 时的窗口匹配）

消隐信号：
  BNK  = ~(PAL ? RASTERC[8] : BLANK_NTSC)  — 垂直消隐
  CHBL = ~R15_REG[3]                         — 水平消隐

可见区域：
  NTSC: 320×224 像素 @ 59.185 Hz
  PAL:  320×256 像素 @ 50.008 Hz
```

### 9.2 行缓冲 (`linebuffer.v`)

```
架构：4 块双端口 RAM 实现精灵双缓冲

  RAMBR (Bottom-Right)  ┐
  RAMBL (Bottom-Left)   ┤ 每块 256×12-bit
  RAMTR (Top-Right)     ┤ 两个 bank：Top / Bottom
  RAMTL (Top-Left)      ┘

数据格式（12-bit per pixel）：
  [11:4] = 精灵调色板编号（8-bit，256 个调色板）
  [3:0]  = 颜色索引（4-bit，每调色板 16 色）

双缓冲切换：
  MUX_BA = {TMS0, S1H1} 选择输出 bank
  一个 bank 被 LSPC2 写入新数据（渲染中）
  另一个 bank 被 NEO-B1 读取（显示中）

写入路径：
  CLEARING=1 时：强制 color=0xF（背景色），清除行缓冲
  正常写入：COLOR_GATED = 精灵解码后的颜色数据

读取路径：
  CK 上升沿锁存数据 → DATA_OUT
  地址由 ADDR_COUNTER 自增或 ADDR_LOAD 重载
```

### 9.3 水平缩放 (`hshrink.v`)

```
功能：实现精灵的水平方向缩放（0-100%）

SHRINK[3:0] 编码缩放因子
  4 个查找表（U193_P, T196_P, U243_P, U226_P）产生 4-bit 移位模式
  双移位寄存器链模拟原始 FS2 芯片

工作方式：
  LOAD（L=0）：并行加载缩放模式
  SHIFT（L=1）：串行移位，CK_EN 使能
  OUTA/OUTB：两个抽头输出，决定像素是否输出或跳过

效果：根据 SHRINK 值条件性地跳过像素，实现水平压缩
```

### 9.4 视频中断控制 (`irq.v`)

```
3 个独立中断源：
  B56_Q → Reset IRQ     — ACK[0] 确认清除
  B52_Q → Timer IRQ     — ACK[1] 确认，TIMER_IRQ_EN 门控
  C52_Q → VBlank IRQ    — ACK[2] 确认，VBL_IRQ_EN 门控

中断优先级编码输出 IPL[1:0]：
  IRQ状态    IPL     68000 中断级
  xx1        100     Level 4（Reset IRQ，最高）
  x10        101     Level 5（Timer IRQ）
  100        110     Level 6（VBlank IRQ）
  000        111     无中断

IPL0 = ~(~B32_Q[0] | B32_Q[2])
IPL1 = ~(~B32_Q[1] | ~B32_Q[0])

CD 模式特殊处理：
  CD_VBLANK_IRQ_EN / CD_TIMER_IRQ_EN 由 cd.sv 的 REG_FF0004 控制
  卡带模式下始终使能
```

### 9.5 自动动画计数器 (`autoanim.v`)

```
功能：自动循环精灵 tile 编号，实现动画效果（如火焰、水波等）

定时器：
  TIMER_CNT[7:0]：从 ~AA_SPEED[7:4:0] 计数到 0xFF
  AA_SPEED 控制频率（0=最慢, 0xF=最快）
  RASTER8 上升沿驱动（约每 2 帧一次触发）

输出：
  AA_COUNT[2:0]：3-bit 动画索引
  TIMER_CNT 溢出时递增
  LSPC2 用此值自动替换特定 tile 编号的低位
```

### 9.6 RGB 输出管线 (`neogeo.sv` 2150-2257行)

```
流水线：PAL_RAM → 颜色重建 → Shadow 处理 → video_cleaner → video_mixer → VGA 输出

第1步：调色板 RAM 读取
  PAL_RAM_ADDR 由 NEO-B1 输出（优先级：CPU > 消隐 > Fix > 精灵）
  {PALBNK, PAL_RAM_ADDR} 寻址 8K×16 调色板 RAM
  CLK_EN_6MB 锁存 PAL_RAM_REG

  消隐/窗口门控：
    pxcnt 范围 [7, 311) 内有效（304 像素可见区域）
    status[16]=0 时强制全屏输出（忽略消隐）
    VIDEO_EN 门控：CD 模式由 CD_VIDEO_EN 控制

第2步：16-bit → 6-bit 颜色重建
  Neo Geo 16-bit 颜色格式：
    Bit 15: Dark Bit (D)       Bits 11-8: R[4:1]
    Bit 14: R[0] (LSB)        Bits 7-4:  G[4:1]
    Bit 13: G[0] (LSB)        Bits 3-0:  B[4:1]
    Bit 12: B[0] (LSB)

  R6 = {0, R[4:1], R[0], R[4]} - D    // 6-bit + 借位检测
  G6 = {0, G[4:1], G[0], G[4]} - D    // LSB 重复到最低位
  B6 = {0, B[4:1], B[0], B[4]} - D    // Dark Bit 实现减暗

  借位检测：R6[6]=1 → 结果为负 → 钳位到 0

第3步：6-bit → 8-bit 扩展
  R8 = R6[6] ? 0 : {R6[5:0], R6[4:3]}   // 低2位填充 bit[4:3]
  G8 = G6[6] ? 0 : {G6[5:0], G6[4:3]}
  B8 = B6[6] ? 0 : {B6[5:0], B6[4:3]}

第4步：Shadow 效果
  SHADOW 信号来自 syslatch.v（$3A0001 bit 0）
  ~SHADOW ? R8 : {1'b0, R8[7:1]}    // 右移1位 = 亮度减半

第5步：VSync 重建
  原始 VSYNC 几乎等于 VBlank，不够精确
  重新计算：统计 VBlank 行数 → 中点位置 → 生成 3 行宽的 VSYNC 脉冲

第6步：video_cleaner → video_mixer → 最终输出
  video_cleaner：清理同步/消隐时序
  video_mixer：
    LINE_LENGTH = 320
    支持 scandoubler（扫描线倍频）
    HQ2x 插值（scale==1 时启用）
    CRT 扫描线效果（scale=2/3/4 → 25%/50%/75% 暗度）
    VGA_SL = scale ? scale[1:0] - 1 : 0
```

---

## 十、控制器输入与 Combo 按钮

### 10.1 输入系统架构

```
MiSTer 框架
  │
  ├── hps_io → joystick_0[15:0]  (P1 主手柄)
  │          → joystick_1[15:0]  (P2 主手柄)
  │          → joystick_2[15:0]  (P1 副/combo)
  │          → joystick_3[15:0]  (P2 副/combo)
  │          → spinner_0/1       (旋转编码器)
  │          → ps2_mouse         (鼠标)
  │          → ps2_key           (键盘)
  │
  ├── joy_0/joy_1 选择逻辑 (status[43:44] 控制)
  │
  └── NEO-C1 → P1_IN / P2_IN → 68000 读取 $300001 / $340001

手柄位映射（CONF_STR 定义）：
  "J1,A,B,C,D,Start,Select,Coin,ABC,A+B,C+D;"
  joystick_*[0]  = A         joystick_*[6]  = Coin
  joystick_*[1]  = B         joystick_*[7]  = ABC (A+B+C)
  joystick_*[2]  = C         joystick_*[8]  = A+B
  joystick_*[3]  = D         joystick_*[9]  = C+D
  joystick_*[4]  = Start     joystick_*[10] = (手柄ID扩展)
  joystick_*[5]  = Select    joystick_*[11] = (额外按键)

名称映射 → 实际游戏手柄：
  jn: A→A, B→B, C→X, D→Y, Start→Start, Select→Select, ABC→L, A+B→R, C+D→L2
  jp: A→B, B→A, C→D, D→C (Neo CD 控制器布局)
```

### 10.2 Combo 按钮实现

```verilog
// 三种输入模式由 status[43:44] 选择：

// 模式 A：多分接器模式（status[43]=1）
// P1_OUT[0] 切换主/副手柄
wire [11:0] joy_0a = P1_OUT[0]
    ? (joystick_2[11:0] | {P1_OUT[2],5'b00000})   // 副手柄
    : (joystick_0[11:0] | {P1_OUT[2],4'b0000});   // 主手柄

// 模式 B：Combo 按钮模式（status[44]=1）
// P1_OUT[0] 和 P1_OUT[1] 分别门控主/副手柄，结果 OR 合并
wire [11:0] joy_0b = ({12{P1_OUT[0]}} & joystick_0[11:0])
                   | ({12{P1_OUT[1]}} & joystick_2[11:0])
                   | {P1_OUT[2], 9'd0};

// 最终选择
wire [11:0] joy_0 = status[43] ? joy_0a   // 多分接器
                  : status[44] ? joy_0b   // Combo
                  :              joystick_0;  // 标准

// P1_IN 送入 NEO-C1（取反，因为 Neo Geo 输入为低有效）：
// {Select|Coin, 方向/按钮...}
// joystick_0[12:13] 映射为额外方向扩展
// joy_0[11] 在非扩展 RAM 模式下复制到 3 个方向位
```

### 10.3 多人模式支持

```
OSD 设置 "Multitap"（status[43:44]）：
  No         — 标准 2P 模式
  NEO-FTC1B  — 4 人适配器（status[43]=1）
  NeoTris    — 俄罗斯方块 4 人变体

NEO-FTC1B 工作原理：
  游戏通过 P1_OUT[0] 切换玩家（0=P1/P3, 1=P2/P4）
  读取 joy_0a 时，P1_OUT[0]=0 返回主手柄，=1 返回副手柄
```

### 10.4 旋转编码器与鼠标

```verilog
// 输入模式选择（status[42:41]）：
//   00/01: 自动检测（检测到旋转器/鼠标活动则切换）
//   10: 强制旋转器
//   11: 鼠标（Irritating Maze 专用）

// 旋转器累加：
always @(posedge clk_sys) begin
    if(old_sp0 ^ spinner_0[8]) sp0 <= sp0 - spinner_0[6:0];
    if(old_ms ^ ps2_mouse[24]) sp0 <= sp0 - ps2_mouse[14:8]; // 鼠标也累加到 sp0
end

// 自动切换逻辑：
if(旋转器/鼠标有活动) use_sp <= 1;
if(摇杆方向键按下)    use_sp <= 0;  // 摇杆覆盖

// 鼠标模式（use_mouse = &status[42:41]）：
// ms_xy 寄存器由 68000 写 nBITW0 切换 X/Y 轴
// ms_pos = ms_xy ? ms_y : ms_x
// ms_btn = {00, 左键, 右键, 0000}
```

### 10.5 DIP 开关

```
OSD → status 位 → neo_f0.v：
  DIPSW = {~status[9], ~status[8], 5'b11111, ~status[7]}

status[7]:  [DIP] Settings — 服务模式开关
status[8]:  [DIP] Freeplay — 免费游戏
status[9]:  [DIP] Freeze   — 画面冻结（调试用）
```

---

## 十一、存档系统

### 11.1 备份 RAM (`backup.v`)

```
容量：64KB（2 × 32KB dpram，分 SRAML + SRAMU）
地址：68000 地址 $D00000-$D0FFFF

双端口访问：
  Port A (CLK):     68000 读写
    地址：M68K_ADDR[15:1]
    写使能：nBWL = nSRAMWEL | nSRAMWEN_G（低字节）
           nBWU = nSRAMWEU | nSRAMWEN_G（高字节）
    nSRAMWEN_G 需要 syslatch 的 nSRAMWEN 使能

  Port B (clk_sys): HPS 读写（存档保存/加载）
    地址：sram_addr[14:0]
    写使能：sram_wr
    数据：sd_buff_dout ↔ sd_buff_din_sram
```

### 11.2 记忆卡 (`memcard.v`)

```
容量：
  卡带模式：2KB（每个游戏独立）— CDA_MASKED = {2'b00, CDA[10:1]}
  CD 模式：  8KB（所有游戏共享）— CDA_MASKED = CDA[12:1]

实现：2 × dpram（MEMCARDL + MEMCARDU），每块 4K×8-bit
  CDA[0] 选择高/低字节 RAM
  HPS 侧以 16-bit word 访问

68000 地址：$D10000-$D107FF
写保护：nWP 始终为 0（从不写保护）
```

### 11.3 存档文件格式

```
卡带游戏存档：
  偏移        内容              大小
  $000000     Backup RAM        64KB
  $010000     Memory Card       2KB

CD 系统存档：
  偏移        内容              大小
  $000000     Memory Card       8KB

OSD 操作：
  "RL,Reload Memory Card"    — 从 SD 卡重新加载
  "RC,Save Memory Card"      — 保存到 SD 卡
  "Autosave" (status[24])    — 自动保存开关
```

---

## 十二、构建系统与工程配置

### 12.1 Quartus 工程

```
目标器件：Cyclone V (DE10-Nano)
顶层实体：sys_top（MiSTer 框架，自动包装 neogeo.sv）

工程文件：
  NeoGeo.qpf      — 工程定义文件
  NeoGeo.qsf      — 主配置（引脚、编译选项、文件列表）
  NeoGeo.sdc      — 时序约束
  files.qip        — 文件清单（层级包含子 QIP）

编译输出：NeoGeo_*.rbf（FPGA 比特流，约 3.6-3.8 MB）
```

如需实际操作编译、部署到 `MiSTer/DE10-Nano`，以及准备 `ROM`、`BIOS` 和 `romsets.xml`，请参见 `docs/BUILD.md`。

### 12.2 文件组织 (`files.qip`)

```
层级 QIP 包含：
  rtl/cpu/FX68K/fx68k.qip    — 68000 CPU 核心
  rtl/cpu/T80/T80.qip        — Z80 CPU 核心
  rtl/video/video.qip        — 视频系统所有模块
  rtl/cd/cd.qip              — Neo CD 子系统
  rtl/io/io.qip              — I/O 控制器
  rtl/cells/cells.qip        — TTL 模拟单元（FD/FS/C43 等）
  rtl/mem/bram.qip           — BRAM 定义
  rtl/mem/dram.qip           — SDRAM 控制器
  rtl/jt12/target/quartus/jt10.qip — YM2610
  rtl/jt49/syn/quartus/jt49.qip    — PSG
  neogeo.sv                  — 核心顶层（2257 行）
```

### 12.3 时序约束 (`NeoGeo.sdc`)

```
derive_pll_clocks              — 自动推导 PLL 输出时钟
derive_clock_uncertainty       — 自动计算时钟不确定性

// Z80 多周期路径（运行在更慢的 4MHz 使能下）
set_multicycle_path -from {emu|Z80CPU|*} -setup 2
set_multicycle_path -from {emu|Z80CPU|*} -hold 1

// PLL 配置为异步路径（MVS/AES 切换时动态重配置）
set_false_path -from {emu|cfg[*]}
```

### 12.4 双 SDRAM 变体

```
NeoGeo_DualSDR.qsf — 支持双 SDRAM 配置
  主 SDRAM：P ROM + S ROM + System ROM（低地址空间）
  副 SDRAM：C ROM 扩展（高地址空间）

地址分配逻辑（sdram_mux.sv）：
  sdr_pri_sel = (~sdram_addr[26] | sdr_pri_128_64[1])
              & (~sdram_addr[25] | sdr_pri_128_64[0]);

  128MB 单片：全部走主 SDRAM
  64MB + 64MB：按地址高位 [26:25] 分配
  32MB + 32MB：类似
```

---

## 十三、跨模块设计模式总结

### 13.1 同步设计原则

```
核心规则：所有模块共用 CLK_48M（或 CLK_96M），通过 CLK_EN_* 使能信号模拟不同频率

优点：
  - 避免跨时钟域亚稳态问题
  - 所有信号在同一时钟沿采样，无需 CDC 同步器
  - 简化时序约束

示例：68000 不是真的跑 12MHz，而是在 48MHz 的每 4 个周期中，
      仅 CLK_EN_68K_P=1 的那个周期执行逻辑
```

### 13.2 双缓冲策略

```
三处使用双缓冲：
  1. 行缓冲（linebuffer.v）：Top/Bottom bank
     → 一行渲染精灵，另一行输出显示，每行切换
  2. CD 扇区缓存（cd.sv）：SECTOR_FILLED[2]
     → 一个 bank 接收新数据，另一个被 DMA 读取
  3. 调色板 bank（syslatch PALBNK）：
     → 游戏可以在不可见的 bank 准备新调色板，VBlank 时切换
```

### 13.3 优先级编码模式

```
调色板地址输出（NEO-B1）：
  CPU 访问 > 消隐 > Fix 层 > 精灵层 > 背景色

中断优先级（irq.v）：
  Reset (Level 4) > Timer (Level 5) > VBlank (Level 6)

SDRAM 访问仲裁（sdram_mux.sv）：
  ROM 加载 > 68k 程序 > 精灵图形 > Fix 图形 > DMA > 刷新

DMA 总线仲裁（cd.sv）：
  请求 nBR → 等待 nBG 授权 → 等待 nAS 高 → 开始传输
```

### 13.4 微码驱动 DMA

```
CD 系统的 DMA 操作由 10-word 微码表控制：
  游戏 BIOS 将微码写入 $FF007E-$FF008E
  cd.sv 解码微码第 0 word → 确定 DMA 模式
  无需微控制器，纯硬件状态机执行

这种设计允许：
  - BIOS 灵活定义新的传输模式
  - 不同 CD BIOS 版本可以有不同的 DMA 行为
  - 硬件状态机保证确定性时序
```

### 13.5 完整信号流总图

```
┌─ HPS (ARM) ──────────────────────────────────────────────┐
│  ROM加载 / OSD配置 / 手柄输入 / CD数据 / 存档读写         │
└────┬──────────┬──────────┬───────────┬──────────┬────────┘
     │ hps_io   │ hps_ext  │ ioctl     │ sd_buff  │ joystick
     ↓          ↓          ↓           ↓          ↓
┌────────────────────────────────────────────────────────────┐
│                    neogeo.sv (顶层)                        │
│                                                            │
│  ┌─ SDRAM_MUX ─┐  ┌─ DDR3 ─┐  ┌─ BRAM ──────────────┐   │
│  │ P/S/C ROM   │  │ V ROM  │  │ Work RAM  (64KB)    │   │
│  │ System ROM  │  │ M1 ROM │  │ Z80 RAM   (2KB)     │   │
│  └──┬───┬───┬──┘  └───┬────┘  │ Slow VRAM (64KB)    │   │
│     │   │   │          │       │ Fast VRAM (4KB)     │   │
│     │   │   │          │       │ Palette   (16KB)    │   │
│     │   │   │          │       │ LO ROM    (64KB)    │   │
│     ↓   ↓   ↓          ↓       │ Backup    (64KB)    │   │
│  ┌────┐ ┌────┐  ┌──────────┐  │ MemCard   (2-8KB)   │   │
│  │68K │ │LSPC│  │  YM2610  │  └──────────────────────┘   │
│  │CPU │ │ 2  │  │  (jt10)  │                              │
│  └─┬──┘ └─┬──┘  └────┬─────┘                              │
│    │      │           │                                    │
│    │  ┌───┴────┐      │      ┌── 保护芯片 ──┐             │
│    ├→ │NEO-B1  │──→RGB│      │ SMA/PVC/CMC │             │
│    │  │+palette│  输出 │      └──────────────┘             │
│    │  └────────┘      │                                    │
│    │                  │      ┌── CD 子系统 ──┐             │
│    ├→ NEO-C1 (地址解码)│      │ cd_sys       │             │
│    ├→ NEO-D0 (bankswitch)    │ lc8951       │             │
│    ├→ NEO-E0 (地址混洗)│      │ cdda         │             │
│    ├→ NEO-F0 (I/O复用) │      │ cd_drive     │             │
│    ├→ syslatch        │      └──────────────┘             │
│    └→ watchdog        │                                    │
│                       ↓                                    │
│              ┌─ video_cleaner ─┐                           │
│              │ video_mixer     │──→ HDMI/VGA 输出          │
│              └─────────────────┘                           │
│                                                            │
│              ┌─ audio_out ─────┐                           │
│              │ FM + ADPCM + PSG│──→ I2S/SPDIF 输出        │
│              │ + CDDA          │                           │
│              └─────────────────┘                           │
└────────────────────────────────────────────────────────────┘
```

---

> 本文档基于 NeoGeo_MiSTer 仓库源码逐行分析补充，覆盖所有 RTL 模块
