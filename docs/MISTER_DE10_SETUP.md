# MiSTer / DE10-Nano 安装、更新、编译与烧录指南

> 本文档基于 `MiSTer-devel` 官方 GitHub 仓库与文档整理，面向第一次接触 `MiSTer` 和 `DE10-Nano` 的用户。  
> 更新时间：2026-03-10

---

## 一、这份文档解决什么问题

如果你问的是“怎样把这个 NeoGeo core 跑到 `DE10-Nano` 上”，通常需要区分 3 件事：

1. **先把 MiSTer 系统装到 SD 卡并让 `DE10-Nano` 启动起来**
2. **再更新 MiSTer 主程序、菜单和常用组件**
3. **最后编译或复制某个 core 的 `.rbf` 文件到 SD 卡中运行**

对于 `MiSTer` 平台，最常见的“烧录”其实是：

- 把 `Mr. Fusion` 镜像写入 SD 卡
- 用 `MiSTer` 的脚本更新系统和 core
- 把某个 FPGA core 的 `.rbf` 复制到 SD 卡

而不是把本仓库直接永久烧进 `DE10-Nano` 的 FPGA 配置 Flash。

如果你已经有可用的 `MiSTer` 系统，只想编译本仓库的 `NeoGeo` core，请直接看 `docs/BUILD.md`。

---

## 二、官方 GitHub 来源

本文档主要基于以下官方来源整理：

- `MiSTer-devel/Wiki_MiSTer`：MiSTer 官方说明站点源码  
  <https://github.com/MiSTer-devel/Wiki_MiSTer>
- `MiSTer-devel/Main_MiSTer`：MiSTer 主程序与 Wiki 入口  
  <https://github.com/MiSTer-devel/Main_MiSTer>
- `MiSTer-devel/mr-fusion`：推荐的首次安装方式  
  <https://github.com/MiSTer-devel/mr-fusion>
- `MiSTer-devel/Downloader_MiSTer`：官方推荐更新脚本  
  <https://github.com/MiSTer-devel/Downloader_MiSTer>
- `MiSTer-devel/Hardware_MiSTer`：官方硬件扩展板说明  
  <https://github.com/MiSTer-devel/Hardware_MiSTer>

---

## 三、首次安装 MiSTer 到 DE10-Nano（推荐：Mr. Fusion）

### 3.1 准备硬件

根据官方文档，基础运行通常至少需要：

- `DE10-Nano`
- `microSD` 卡（建议 `32GB` 或更大）
- 合适的 `5V` 电源
- HDMI 显示器与输入设备

如果你要运行较大的街机 / 主机 core，通常还需要额外硬件，例如：

- `SDRAM` 扩展板
- `I/O` Board
- USB Hub
- RTC / 模拟视频 / 数字音频等附加模块

> 具体硬件组合与兼容性，请参考 `Hardware_MiSTer` 与官方 Setup Guide。

### 3.2 使用 Mr. Fusion 写入 SD 卡

官方文档推荐新用户优先使用 `Mr. Fusion` 初始化 SD 卡。高层流程如下：

1. 从 `MiSTer-devel/mr-fusion` 获取最新发布镜像
2. 使用 `balenaEtcher`、`Raspberry Pi Imager`、`Win32 Disk Imager` 等工具把镜像写入 `microSD`
3. 将 SD 卡插入 `DE10-Nano`
4. 首次开机，等待 `Mr. Fusion` 自动展开并完成基础安装
5. 安装完成后按提示断电重启

完成这一步后，`DE10-Nano` 就具备基础 `MiSTer` 启动环境。

### 3.3 首次开机后的目标状态

首次成功启动后，你应该能看到：

- `MiSTer` OSD 菜单
- `Scripts` 菜单或脚本目录
- 可识别的 SD 卡文件系统

如果此时只是想体验 `MiSTer`，通常不需要手动编译任何核心。

---

## 四、更新 MiSTer 主程序与常用组件

### 4.1 官方推荐方式：Downloader

官方文档当前推荐使用 `Downloader` 脚本更新：

- `MiSTer` 主程序
- 菜单 / 字体 / cheats / filters
- 各类 core
- DB / 资源文件

仓库：<https://github.com/MiSTer-devel/Downloader_MiSTer>

### 4.2 基本使用方式

在 `MiSTer` 上首次联网后，常见流程是：

1. 打开 `Scripts` 菜单
2. 运行 `Downloader`
3. 选择需要更新的内容
4. 等待脚本自动下载和安装
5. 重启或重新加载对应 core

如果你只是普通玩家，这通常已经足够，不需要自己编译 `Main_MiSTer` 或大部分 core。

---

## 五、编译这个仓库的 NeoGeo core 并部署到 MiSTer

### 5.1 什么时候需要自己编译

只有在以下场景下，才建议你自己编译本仓库：

- 你修改了 `RTL`
- 你要验证某个分支或补丁
- 你需要自定义 bitstream

如果你只是想运行 NeoGeo core，优先使用仓库 `releases/` 下已经提供的 `.rbf`。

### 5.2 本仓库的编译入口

本仓库已经提供完整 `Quartus` 工程：

- 单 `SDRAM`：`NeoGeo.qpf`
- 双 `SDRAM`：`NeoGeo_DualSDR.qpf`

详细编译说明请直接看：`docs/BUILD.md`

其中已经包含：

- `Quartus 17.0.2` 版本要求
- `GUI / CLI` 编译方法
- `.rbf` / `.sof` 区别
- ROM、BIOS、`romsets.xml` 准备方法

### 5.3 编译完成后如何“烧录”

对于 `MiSTer` core，常规部署方式不是把 `.sof` 长期固化进 FPGA，而是：

1. 取编译生成的 `.rbf`
2. 复制到 SD 卡中的 core 目录，例如：`/_Console/` 或 `/_Arcade/`
3. 将 ROM、BIOS、`romsets.xml` 放到 `games/NeoGeo` / `games/NeoGeo-CD`
4. 回到 `MiSTer` OSD 中选择该 core 启动

这就是绝大多数 `MiSTer` core 的实际“烧录/部署”流程。

---

## 六、JTAG 临时下载与“真正烧录”的区别

### 6.1 `.sof` 适合什么场景

`.sof` 适合：

- 开发调试
- 验证 FPGA 综合结果
- 临时通过 `USB-Blaster` 下载到板子里

特点是：

- 断电丢失
- 不适合作为日常分发文件
- 不等价于 MiSTer 运行时使用的 `.rbf`

### 6.2 `.rbf` 才是 MiSTer 常规运行路径

对于 `MiSTer` 而言，`HPS` 会在运行中把 `.rbf` 加载到 FPGA，因此：

- 普通用户应关注 `.rbf`
- 开发者在硬件验证时才会频繁用 `.sof`

### 6.3 为什么不建议把本 core 当独立工程去烧 Flash

因为这个 core 依赖 `MiSTer` 提供的系统环境与接口，例如：

- `HPS` 与 FPGA 通信
- OSD 菜单
- 文件加载
- SDRAM / BRAM / 通用基础设施

所以对于本仓库，最正确的路径是：

- `MiSTer` 先装好
- core 编译成 `.rbf`
- 复制到 SD 卡并由 `MiSTer` 加载

---

## 七、进阶：是否需要编译 Main_MiSTer 或整个 MiSTer 系统

### 7.1 大多数用户不需要

官方开发文档明确指出，完整 `MiSTer` 开发栈包括：

- Linux kernel
- U-Boot
- device tree
- Buildroot 用户态系统
- `Main_MiSTer` 主程序
- 各 FPGA core

但对于绝大多数用户或单 core 开发者：

- **不需要自己重编整个 MiSTer 系统**
- 一般只需要更新官方发布版，或编译你正在修改的那个 core

### 7.2 如果你要编译 Main_MiSTer

官方文档给出的简化流程是：

1. 安装 `arm-linux-gnueabihf` 交叉编译工具链
2. 克隆 `MiSTer-devel/Main_MiSTer`
3. 准备并编译第三方库（如 `libjpeg`、`libpng`、`freetype`）
4. 执行 `make` 生成 `MiSTer` 主程序
5. 将生成的二进制替换到板子上的 `/media/fat/MiSTer`

这一部分属于 `MiSTer` 平台开发，而不是本仓库 `NeoGeo` core 的常规使用流程。

如果你后续需要，我可以再基于 `Main_MiSTer` 官方文档单独补一份“编译主程序”的专题文档。

---

## 八、建议的实际操作顺序

如果你是第一次上手，建议按下面顺序做：

1. 用 `Mr. Fusion` 把 `MiSTer` 系统写入 SD 卡
2. 在 `DE10-Nano` 上完成首次启动
3. 运行 `Downloader` 更新系统与常用资源
4. 直接使用本仓库 `releases/` 下的 `NeoGeo_*.rbf` 做第一次验证
5. 确认 `games/NeoGeo`、BIOS、`romsets.xml` 工作正常
6. 只有在需要改代码时，再按 `docs/BUILD.md` 自己编译 core
7. 只有在硬件调试时，才使用 `JTAG + .sof`

---

## 九、常见问题

### 9.1 “烧录到 DE10-Nano”到底是指什么？

在 `MiSTer` 语境里，通常有两种含义：

- **把 MiSTer 系统镜像写入 SD 卡**：这是真正的新机初始化
- **把 core 的 `.rbf` 放进 SD 卡让 MiSTer 加载**：这是日常使用中的“部署”

而不是像传统 FPGA 裸工程那样，默认去做长期的配置 Flash 固化。

### 9.2 我是不是必须自己编译 NeoGeo core？

不是。大多数情况下你可以直接使用仓库 `releases/` 下已有的 `.rbf`。

### 9.3 我是不是必须自己编译 Main_MiSTer？

也不是。只有在你要参与 `MiSTer` 主程序开发时才需要。

### 9.4 第一次上电最推荐的路径是什么？

官方路线是：

- `Mr. Fusion` 初始化 SD 卡
- `Downloader` 更新系统
- 使用现成 core 验证
- 需要开发时再进入自编译流程

---

## 十、后续阅读

- 本仓库 core 编译与部署：`docs/BUILD.md`
- `romsets.xml` 说明：`xml.md`
- 购买版 ROM 的整理方式：`releases/README.md`
- MiSTer 官方 Setup Guide：<https://mister-devel.github.io/MkDocs_MiSTer/>
- MiSTer 主仓库：<https://github.com/MiSTer-devel/Main_MiSTer>
