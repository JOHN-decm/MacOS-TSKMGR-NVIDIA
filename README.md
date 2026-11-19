# MacOS-TSKMGR-NVIDIA

[简体中文](README.md) · [English](README.en.md)

面向 **Intel CPU + NVIDIA GPU 黑苹果** 的 macOS 任务管理器。项目运行在 macOS 26.0 及以上，只构建和发布 `x86_64` 版本。

它采用接近 Windows 任务管理器的页面结构，同时使用 macOS 原生 SwiftUI、AppKit、IOKit、SMC 和系统统计接口读取本机进程、性能、磁盘、GPU 与散热数据。

> 当前工程是 Intel + NVIDIA 平台特化版本，不再以 Apple Silicon/ARM 作为运行目标。

## 这几轮完成了什么

### 从 ARM 版本切换为 Intel + NVIDIA 特化

本工程最初沿用了 Mac mini M4 版本的系统数据读取方式。经过本轮平台迁移，工程已经固定为：

- Intel CPU
- Intel 核显和 NVIDIA 独显组合
- macOS 26.0 最低版本
- 仅打包 `x86_64`
- 不再为 Apple Silicon 提供通用兼容路径

构建和打包流程也已经固定为 x86_64 Release 产物，应用图标、最低系统版本和运行时 Swift 库都由统一脚本处理。

### 进程页面

进程页保留了任务管理器式的两种显示模式：

- 简洁信息模式：快速查看应用列表
- 详细信息模式：显示 CPU、内存、磁盘、网络和电源使用情况
- 应用与后台进程分组
- 结束任务、运行新任务、打开资源监视器等操作
- 深色模式下重新调整分组标题颜色，保证文字可读性
- 后台进程图标使用缓存，避免刷新过程中出现卡顿

<img src="pics/截屏2026-11-19 20.16.09.png" alt="进程简洁信息模式" width="720">

<img src="pics/截屏2026-11-19 20.16.21.png" alt="进程详细信息模式" width="960">

### CPU 性能页

CPU 页面针对 Intel 处理器重新校准了核心和缓存数据：

- 显示整体 CPU 利用率、频率和逻辑处理器图表
- 支持逻辑核心逐核心利用率
- 显示插槽、内核、逻辑处理器、虚拟化状态
- L1、L2、L3 缓存读取增加 `hw.cachesize` 回退路径，适配黑苹果上部分 `hw.l3cachesize` 返回 0 的情况
- 性能页左侧所有项目使用统一的 CPU 风格选中高亮

<img src="pics/截屏2026-11-19 20.16.29.png" alt="Intel CPU 性能页" width="960">

### 磁盘和分区读取

磁盘页面是本轮改动较多的部分，重点解决了 EFI、APFS、多分区和 SATA 磁盘的显示问题：

- 过滤 Apple disk image / disk image media，不把系统镜像当作真实挂载磁盘
- 默认忽略 EFI/ESP 分区，只有被单独强制挂载时才保留
- APFS 合成盘尽量映射到最上层物理磁盘
- 多挂载点按照 `/`、`/Volumes/...` 分别读取可用容量
- “可用”字段中每个分区单独占一行，路径和容量允许完整换行
- 右侧显示硬件名称，而不是卷标
- 依据 `SolidState` 和设备信息显示 SSD/HDD，SATA 设备不会被一律判定为 SSD
- 磁盘温度支持 HID 传感器，并保留 `smartctl` 作为可选回退
- 移除详情页面中同步调用 `diskutil` 导致的 SwiftUI 崩溃路径

<img src="pics/截屏2026-11-19 20.16.37.png" alt="磁盘 0 和多分区可用容量" width="960">

<img src="pics/截屏2026-11-19 20.17.05.png" alt="磁盘 1 性能页" width="960">

### Intel 核显与 NVIDIA 独显

GPU 页面已经从 ARM/Apple GPU 读取逻辑切换为 Intel + NVIDIA 专用路径：

- 通过 IORegistry 和 IOAccelerator 读取 GPU 设备
- 区分 Intel 集成显卡和 NVIDIA 独立显卡
- 显示 GPU 型号、利用率、显存和共享显存
- Intel 核显核心显示为 `24 (EU)`
- NVIDIA GPU 核心显示为 `1024 (CUDA)`
- 过滤 `UInt64.max` 等无效值，避免共享显存出现溢出或错误的大数
- Intel 核显不再误判为拥有专用显存
- NVIDIA 独显的专用显存和共享显存分开显示
- GPU 详情页移除重复的“GPU 个数”项目

<img src="pics/截屏2026-11-19 20.17.10.png" alt="Intel UHD Graphics 630 核显" width="960">

<img src="pics/截屏2026-11-19 20.17.19.png" alt="NVIDIA Quadro T2000 独显" width="960">

### 散热、温度和风扇

散热页面综合使用 AppleSMC、VirtualSMC、HID 和 IORegistry 传感器：

- CPU 整体温度
- 性能核心和能效核心温度（硬件提供时）
- GPU 温度
- 磁盘温度
- 网卡温度
- 主板、整机、SoC 和电源表面温度
- 电池温度回退
- 风扇转速和多个风扇分别显示
- 传感器不可用时，左侧显示“不可读取”，右侧显示“不可读取（OpenCore限制）”
- 通过 SMCSuperIO/VirtualSMC 暴露的风扇数据会优先读取
- 传感器采样增加约 2 秒缓存，避免高频刷新拖慢界面

黑苹果的风扇数据是否可用取决于 OpenCore、VirtualSMC 和主板 Super I/O 芯片是否把键值注册给 macOS。没有注册时，界面会明确标注限制，而不是显示伪造的 RPM。

<img src="pics/截屏2026-11-19 20.17.32.png" alt="散热、温度和风扇页面" width="960">

## 页面和功能

当前应用包含：

- 进程
- 性能
- 应用历史记录
- 启动
- 用户
- 详细信息
- 服务

性能页目前覆盖：

- CPU
- 内存
- 磁盘
- 网络
- Intel 核显
- NVIDIA 独显
- 散热

窗口使用原生 macOS 模糊材质和分层背景，支持深色/浅色模式、紧凑布局、图表显示开关、刷新频率设置、置顶显示、网络详情面板和复制当前性能详情。

## 黑苹果读取限制

本项目会优先使用 macOS 对应的系统读取方式，但黑苹果平台的硬件数据依赖 OpenCore 和驱动暴露情况。以下项目可能因机器不同而不可用：

- 风扇转速
- 主板和电源温度
- NVIDIA 部分 IORegistry 性能字段
- SATA/NVMe 磁盘温度
- 某些 VirtualSMC 键值

因此界面会区分“没有传感器数据”和“OpenCore 限制”，不把读取失败伪装成正常数值。

## 技术实现

工程使用 Xcode 原生 macOS 项目：

- SwiftUI：页面、图表和布局
- AppKit：窗口、菜单、系统交互
- IOKit：GPU、磁盘、IORegistry 设备和传感器
- AppleSMC / VirtualSMC：温度和风扇键值
- `sysctl`：CPU、缓存和系统基础信息
- `host_statistics`：内存和系统统计
- `proc_pidinfo`、`proc_pid_rusage`：进程数据
- CoreWLAN：网络信息
- `launchctl`：服务和启动项
- `CGWindowList`：应用窗口和进程辅助信息

传感器和外部命令读取不在 SwiftUI View 的计算属性中同步执行，避免详情页面触发 AttributeGraph 崩溃。

## NPU / ANE 说明

Intel + NVIDIA 目标平台没有 Apple Neural Engine，因此 NPU/ANE 不属于当前运行时硬件能力。相关代码和说明仅作为早期 Apple Silicon 版本的历史参考保留。

更多实现记录见 [NPU.md](NPU.md) 和 [THERMAL.md](THERMAL.md)。

## 构建

最低系统版本：macOS 26.0
目标架构：`x86_64`

当前工程未配置 Apple Developer 签名，测试包需要在 macOS 中手动允许 Gatekeeper。

## 项目定位与免责声明

本项目是一个开源的 macOS 任务管理器实验项目，界面灵感来自 Windows 任务管理器，仅用于学习、个人使用和硬件数据验证，与 Microsoft 没有任何关联，也未获得 Microsoft 的认可或授权。

部分硬件监控接口属于私有、半公开或依赖驱动实现的系统接口。不同 Intel 机型、NVIDIA 驱动、OpenCore 配置和 macOS 版本可能返回不同字段，传感器显示应以实际机器结果为准。

Windows 是 Microsoft Corporation 的注册商标。
