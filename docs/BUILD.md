# NeoGeo MiSTer FPGA 核心 — 编译与部署指南

> 面向首次接触本仓库的开发者或玩家，覆盖 `Quartus` 编译、`MiSTer/DE10-Nano` 部署，以及 `ROM/BIOS/romsets.xml` 的准备流程。

---

## 一、文档目标与适用范围

本仓库是一个运行在 `MiSTer` 平台上的 `Neo Geo` core，而不是面向裸 `DE10-Nano` 的独立 FPGA 工程。

它的常规使用方式是：

1. 使用 `Quartus` 编译出 FPGA 比特流；
2. 将生成的 `.rbf` 文件复制到 `MiSTer` 的 SD 卡；
3. 由 `MiSTer` 的 `HPS` 在运行时加载该 core；
4. 准备好 `ROM`、`BIOS` 和 `romsets.xml` 后在板上启动。

因此，本文档默认面向**已经安装好 MiSTer 系统的 `DE10-Nano`**。除临时调试外，不建议把本 core 当作独立工程直接“烧进 FPGA 配置 Flash”。

如果你还没有安装 `MiSTer` 系统，或者想先了解 `DE10-Nano` 上的 SD 卡写入、系统更新与常规部署流程，请先看 `docs/MISTER_DE10_SETUP.md`。

---

## 二、工程版本与工具要求

### 2.1 工程分支

本仓库提供两个 `Quartus` 工程入口：

- `NeoGeo.qpf` / `NeoGeo.qsf`
  - 单 `SDRAM` 版本
  - `PROJECT_REVISION`：`NeoGeo`
  - `LAST_QUARTUS_VERSION`：`17.0.2 Lite Edition`
- `NeoGeo_DualSDR.qpf` / `NeoGeo_DualSDR.qsf`
  - 双 `SDRAM` 版本
  - `PROJECT_REVISION`：`NeoGeo_DualSDR`
  - `LAST_QUARTUS_VERSION`：`17.0.2 Standard Edition`

如果你使用的是 `Quartus Lite`，请选择单 `SDRAM` 工程；双 `SDRAM` 工程需要 `Standard Edition`。

### 2.2 关键构建信息

- 目标器件：`5CSEBA6U23I7`
- FPGA 家族：`Cyclone V`
- 顶层实体：`sys_top`
- 输出目录：`output_files`
- 文件列表入口：`files.qip`

> 注意：不要在 `Quartus` IDE 中直接增删工程文件。`NeoGeo.qsf` 已明确提示，应通过修改 `files.qip` 维护源文件列表。

### 2.3 环境准备

开始前请确保：

- 已安装与工程匹配的 `Quartus Prime 17.0.2`
- 若需要 `JTAG` 临时下载，已安装 `Quartus Programmer`
- 若需要直接复制到板子，`MiSTer` 的 SD 卡或网络访问方式可用

---

## 三、编译步骤（GUI / CLI）

### 3.1 使用 Quartus GUI 编译

#### 单 SDRAM 版本

1. 打开 `NeoGeo.qpf`
2. 确认当前 revision 为 `NeoGeo`
3. 在 `Processing` 菜单中选择 `Start Compilation`
4. 等待综合、布局布线、汇编完成
5. 在 `output_files/` 中查看生成文件

#### 双 SDRAM 版本

1. 打开 `NeoGeo_DualSDR.qpf`
2. 确认当前 revision 为 `NeoGeo_DualSDR`
3. 在 `Processing` 菜单中选择 `Start Compilation`
4. 等待完整编译结束
5. 在 `output_files/` 中查看生成文件

### 3.2 使用命令行编译

在仓库根目录执行：

```bash
quartus_sh --flow compile NeoGeo -c NeoGeo
```

双 `SDRAM` 版本执行：

```bash
quartus_sh --flow compile NeoGeo_DualSDR -c NeoGeo_DualSDR
```

编译完成后，产物会出现在 `output_files/` 目录。通常你会看到与 revision 同名的输出文件，例如：

- `output_files/NeoGeo.sof`
- `output_files/NeoGeo.rbf`
- `output_files/NeoGeo_DualSDR.sof`
- `output_files/NeoGeo_DualSDR.rbf`

如果你后续改动了源码文件集合，请修改 `files.qip`，不要直接在 `Quartus` 工程界面中增删文件。

---

## 四、产物说明与 DE10-Nano 部署

### 4.1 `.rbf` 与 `.sof` 的区别

- `.rbf`
  - 面向 `MiSTer` 运行时加载
  - 由 `HPS` 在系统运行中装载到 FPGA
  - 是**正常使用本 core**时应部署的文件
- `.sof`
  - 面向 `JTAG` 临时下载调试
  - 断电后失效
  - 适合验证综合结果，不适合日常分发或长期使用

### 4.2 复制 `.rbf` 到 MiSTer SD 卡

将生成的 `.rbf` 文件复制到 `MiSTer` SD 卡中的核心目录。

仓库根 `README.md` 当前写的是 `Console` 或 `Arcade` 目录；实际很多 `MiSTer` 安装会使用带下划线的目录名，例如：

- `/_Console/`
- `/_Arcade/`

如果你的系统里看到的是不带下划线的 `Console` / `Arcade`，按现有目录结构放置即可。

部署完成后，在板子的 `OSD` 中选择对应的 `NeoGeo` core 即可启动。

### 4.3 使用 JTAG 临时下载 `.sof`

如果你只是想快速验证 FPGA 编译结果，可以使用 `Quartus Programmer` 通过 `JTAG` 临时下发 `.sof`：

1. 将 `DE10-Nano` 通过 `USB-Blaster` 接到电脑
2. 打开 `Quartus Programmer`
3. `Hardware Setup` 中选择对应的下载器
4. `Auto Detect` 识别链路
5. 添加 `output_files/` 下生成的 `.sof`
6. 勾选 `Program/Configure` 并开始下载

这条路径适合临时硬件验证；若你要在 `MiSTer` 环境下实际使用本 core，仍应部署 `.rbf` 到 SD 卡。

### 4.4 关于“烧录进 DE10-Nano”的边界说明

如果你的意思是“让 `MiSTer` 正常运行这个核心”，正确方式是部署 `.rbf` 到 SD 卡并由 `MiSTer` 加载。

如果你的意思是“把 bitstream 固化到板上非易失存储中”，那已经超出本仓库的常规使用方式，也不属于这个 core 的推荐部署路径。对于本项目，优先采用 `.rbf + MiSTer` 的工作流。

---

## 五、ROM / BIOS / `romsets.xml` 准备

### 5.1 目录结构

常见目录结构如下：

```text
/media/fat/
├── _Console/ 或 _Arcade/
│   └── NeoGeo_*.rbf
└── games/
    ├── NeoGeo/
    │   ├── romsets.xml
    │   ├── 000-lo.lo
    │   ├── sfix.sfix
    │   ├── sp-s2.sp1
    │   ├── neo-epo.sp1
    │   ├── uni-bios.rom
    │   └── <各游戏目录或 zip>
    └── NeoGeo-CD/
        ├── top-sp1.bin
        ├── neocd.bin
        └── uni-bioscd.rom
```

### 5.2 BIOS 文件要求

`games/NeoGeo` 目录下需要准备以下 BIOS 文件：

- `000-lo.lo`
- `sfix.sfix`
- `sp-s2.sp1`（MVS）
- `neo-epo.sp1`（AES）
- `uni-bios.rom`

`games/NeoGeo-CD` 目录下需要准备以下 BIOS 文件：

- `top-sp1.bin`（CD）
- `neocd.bin`（CDZ）
- `uni-bioscd.rom`（CD / CDZ）

如果你拿到的 BIOS 文件名不同，需要先重命名到上述名称再使用。

### 5.3 `romsets.xml` 的用途

`romsets.xml` 用来告诉 core：

- 游戏目录或 zip 的名字是什么
- 每个 ROM 文件属于哪一类（`P` / `S` / `C` / `M` / `V`）
- 各文件要加载到什么偏移位置

仓库中已有两份现成配置：

- `releases/romsets.xml`
  - 主要面向 `Darksoft` ROM 包
- `releases/gog-romsets.xml`
  - 面向 `gog.com` 等购买版提取出的 ROM 文件
  - 使用前需要重命名为 `romsets.xml`

如果你使用的是自定义 `MAME` 解密 ROM 集，请参考 `xml.md` 自行编写或调整 `romsets.xml`。

### 5.4 zipped / unzipped ROM 的限制

- 使用未压缩 `Darksoft` ROM 集时：
  - 每个游戏必须放在自己的目录中
  - 目录名必须和 `XML` 中的 ROM set 名称一致
- 使用 zip 形式时：
  - zip 内部**不能再套目录**
- ROM 可以放在 `games/NeoGeo` 的子目录中进行整理，但 `XML` 引用关系必须保持正确

### 5.5 进阶参考

- `xml.md`：`romsets.xml` 的高级字段说明与示例
- `releases/README.md`：使用购买版 ROM（如 `gog`）时的提取与目录组织说明

---

## 六、常见问题

### 6.1 为什么 `Quartus Lite` 编不了双 SDRAM 版本？

因为 `NeoGeo_DualSDR.qsf` 指定的工程版本是 `17.0.2 Standard Edition`。如果你只有 `Lite Edition`，请选择单 `SDRAM` 工程 `NeoGeo.qpf`。

### 6.2 为什么不建议在 Quartus IDE 里直接增删文件？

因为工程文件已经明确写了警告：应通过 `files.qip` 统一维护文件列表。直接在 IDE 中修改，容易破坏仓库现有工程组织。

### 6.3 我已经生成 `.sof` 了，为什么还不能长期使用？

`.sof` 只是 `JTAG` 临时下载文件，适合调试。断电后配置会丢失。正常使用 `MiSTer` core 时，应将 `.rbf` 放到 SD 卡中由 `MiSTer` 加载。

### 6.4 核心编译通过了，为什么板上还是进不去游戏？

编译通过只说明 FPGA 逻辑能生成 bitstream，并不代表 ROM、BIOS、`romsets.xml` 已经准备正确。请优先检查：

- `.rbf` 是否放在正确的核心目录
- `games/NeoGeo` 与 `games/NeoGeo-CD` 是否存在
- BIOS 文件名是否完全匹配
- `romsets.xml` 是否与当前 ROM 集对应

### 6.5 我应该先看哪份文档？

- 想上板运行：先看本文档
- 想理解 `romsets.xml`：看 `xml.md`
- 想处理购买版 ROM：看 `releases/README.md`
- 想理解内部实现：看 `docs/ARCHITECTURE.md` 与 `docs/ARCHITECTURE_DETAIL.md`
