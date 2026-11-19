# MacOS-TSKMGR-NVIDIA

[简体中文](README.md) · [English](README.en.md)

A macOS task manager specialized for **Intel CPU + NVIDIA GPU Hackintosh systems**. The project targets macOS 26.0 and later and builds and ships **x86_64 only**.

It follows a Windows Task Manager-style layout while using native macOS SwiftUI, AppKit, IOKit, SMC, and system-statistics interfaces to read process, performance, disk, GPU, and thermal data.

> This is an Intel + NVIDIA specialized target. Apple Silicon/ARM is not a runtime target for this project.

## What We Completed in This Development Round

### Migrated from the ARM version to Intel + NVIDIA

The project originally inherited data-reading paths developed for a Mac mini M4. The platform-specific migration now fixes the engineering target to:

- Intel CPU
- Intel integrated graphics and NVIDIA discrete graphics
- macOS 26.0 minimum deployment target
- x86_64 packages only
- no general Apple Silicon compatibility path

The Release build and packaging flow is also fixed to x86_64. The app icon, deployment target, and Swift runtime libraries are handled by the packaging script.

### Processes page

The Processes page keeps two Task Manager-style modes:

- Compact mode for a quick application list
- Detailed mode with CPU, memory, disk, network, and power columns
- Separate Applications and Background Processes groups
- End Task, Run New Task, and Resource Monitor actions
- Improved dark-mode contrast for group headings
- Cached process icons to prevent refresh-related stutter

<img src="pics/截屏2026-11-19 20.16.09.png" alt="Compact processes view" width="720">

<img src="pics/截屏2026-11-19 20.16.21.png" alt="Detailed processes view" width="960">

### CPU performance page

The CPU page was recalibrated for Intel processors:

- Overall CPU utilization, frequency, and logical-processor graphs
- Per-logical-core utilization
- Socket, core, logical processor, and virtualization information
- L1, L2, and L3 cache values
- `hw.cachesize` fallback when `hw.l3cachesize` returns zero on some Hackintosh systems
- A unified CPU-style selection highlight for every item in the Performance sidebar

<img src="pics/截屏2026-11-19 20.16.29.png" alt="Intel CPU performance page" width="960">

### Disk and partition handling

Disk handling received several changes for EFI, APFS, multi-partition, and SATA systems:

- Apple disk images and disk image media are filtered from the Performance page
- EFI/ESP partitions are excluded unless they are explicitly mounted
- APFS synthesized disks are mapped to their top-level physical disk where possible
- Available capacity is collected separately for `/` and `/Volumes/...` mount points
- Every mount point is displayed on its own line, with automatic wrapping for the full path and capacity
- The right side shows the hardware name instead of the volume label
- SSD/HDD type is determined from `SolidState` and device information; SATA devices are not assumed to be SSDs
- Disk temperature supports HID sensors with optional `smartctl` fallback
- The SwiftUI detail path no longer synchronously invokes `diskutil`, preventing the previous crash

<img src="pics/截屏2026-11-19 20.16.37.png" alt="Disk 0 with multiple mount points" width="960">

<img src="pics/截屏2026-11-19 20.17.05.png" alt="Disk 1 performance page" width="960">

### Intel integrated graphics and NVIDIA discrete graphics

The GPU page now uses Intel + NVIDIA-specific paths instead of Apple GPU logic:

- GPU devices are read through IORegistry and IOAccelerator
- Intel integrated graphics and NVIDIA discrete graphics are distinguished
- GPU model, utilization, memory, and shared memory are shown
- Intel GPU core count is displayed as `24 (EU)`
- NVIDIA GPU core count is displayed as `1024 (CUDA)`
- Invalid `UInt64.max` values are filtered to prevent shared-memory overflow
- Intel integrated graphics are no longer misclassified as having dedicated VRAM
- NVIDIA dedicated and shared memory are displayed separately
- The redundant GPU-count row was removed from GPU details

<img src="pics/截屏2026-11-19 20.17.10.png" alt="Intel UHD Graphics 630 integrated GPU" width="960">

<img src="pics/截屏2026-11-19 20.17.19.png" alt="NVIDIA Quadro T2000 discrete GPU" width="960">

### Cooling, temperatures, and fan speed

The Cooling page combines AppleSMC, VirtualSMC, HID, and IORegistry sensors:

- Overall CPU temperature
- Performance-core and efficiency-core temperatures when available
- GPU temperature
- Disk temperature
- Network adapter temperature
- Logic-board, system, SoC, and power-surface temperatures
- Battery-temperature fallback
- Separate fan readings when multiple fans are exposed
- “Unavailable” on the left and “Unavailable (OpenCore limitation)” on the right when a sensor is not exposed
- Fan values exposed through SMCSuperIO/VirtualSMC are preferred
- Approximately two seconds of sensor sampling cache to avoid excessive refresh work

Fan readings on a Hackintosh depend on OpenCore, VirtualSMC, and the motherboard Super I/O chip exposing the corresponding keys to macOS. When no key is registered, the UI labels the limitation instead of inventing an RPM value.

<img src="pics/截屏2026-11-19 20.17.32.png" alt="Cooling, temperature, and fan page" width="960">

## Pages and features

The app currently includes:

- Processes
- Performance
- App History
- Startup
- Users
- Details
- Services

The Performance page covers:

- CPU
- Memory
- Disks
- Network
- Intel integrated graphics
- NVIDIA discrete graphics
- Cooling

The window uses native macOS blur materials and layered backgrounds. It supports light and dark appearances, compact layout, graph visibility, refresh-rate settings, always-on-top mode, a network details panel, and copying the current performance details.

## Hackintosh sensor limitations

The app uses the macOS-native reading paths whenever possible, but hardware data on Hackintosh systems depends on OpenCore and driver exposure. The following may vary by machine:

- Fan RPM
- Logic-board and power temperatures
- Some NVIDIA IORegistry performance fields
- SATA/NVMe disk temperature
- Some VirtualSMC keys

The UI distinguishes missing sensor data from an OpenCore limitation and does not present failed reads as normal values.

## Technical implementation

The project is a native Xcode macOS application:

- SwiftUI for pages, charts, and layout
- AppKit for windows, menus, and system interactions
- IOKit for GPUs, disks, IORegistry devices, and sensors
- AppleSMC / VirtualSMC for temperature and fan keys
- `sysctl` for CPU, cache, and system information
- `host_statistics` for memory and system statistics
- `proc_pidinfo` and `proc_pid_rusage` for process data
- CoreWLAN for network information
- `launchctl` for services and startup items
- `CGWindowList` for application-window and process support

Sensor and external-command reads are kept out of synchronous SwiftUI View computations to avoid the AttributeGraph crash path previously triggered from the disk detail page.

## NPU / ANE

The Intel + NVIDIA target has no Apple Neural Engine, so NPU/ANE is not a runtime hardware capability in this project. The related code and notes are retained only as historical reference from the earlier Apple Silicon version.

See [NPU.md](NPU.md) and [THERMAL.md](THERMAL.md) for additional implementation notes.

## Build

Minimum system version: macOS 26.0  
Target architecture: `x86_64`

Release build:

```bash
xcodebuild -project MacOSTSKMGR.xcodeproj \
  -scheme MacOSTSKMGR \
  -configuration Release \
  -derivedDataPath /tmp/MacOSTSKMGR-derived \
  build CODE_SIGNING_ALLOWED=NO
```

Package the app:

```bash
mkdir -p dist/x86_64

./Tools/package_swift_runtime_app.sh \
  /tmp/MacOSTSKMGR-derived/Build/Products/Release/MacOSTSKMGR.app/Contents/MacOS/MacOSTSKMGR \
  MacOSTSKMGR-Latest-Test \
  com.linqin.MacOSTSKMGR \
  26.0 \
  Sources/MacOSTSKMGR/AppIcon.icns \
  dist/x86_64 \
  MacOSTSKMGR-Latest-Test.zip
```

The project currently has no Apple Developer signing setup. Test packages may require manual Gatekeeper approval on macOS.

## Positioning and disclaimer

This is an open-source macOS task manager experiment inspired by the Windows Task Manager interface. It is intended for learning, personal use, and hardware-data validation. It is not affiliated with, endorsed by, or authorized by Microsoft.

Some hardware-monitoring interfaces are private, semi-public, or driver-dependent. Different Intel systems, NVIDIA drivers, OpenCore configurations, and macOS releases may expose different fields, so sensor output should be evaluated on the actual machine.

Windows is a registered trademark of Microsoft Corporation.

