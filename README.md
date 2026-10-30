# FA608FM Hackintosh EFI (macOS 15 Sequoia / macOS 26 Tahoe)

华硕 ASUS FA608FM（AMD 平台）黑苹果 OpenCore EFI。已在 macOS 26.7.1 (Tahoe) 上跑通，Wi-Fi / 蓝牙 / 电源适配器 / 音频 / 触控板均正常。当前状态：**可正常安装/启动 macOS 15.6 和 macOS 26，电源适配器识别、Wi-Fi、蓝牙、音频、触控板、亮度键均正常**。

> ⚠️ **使用前必须生成 SMBIOS**：仓库内 `OC/config.plist` 的 `SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM` 是占位值。请用 GenSMBIOS 生成自己的四项，机型保持 `MacBookPro16,2`。
> 直接沿用别人的序列号会和对方的 Apple ID 服务冲突，也容易被判为无效机器。

## 硬件

| 部件 | 型号 | 状态 |
| --- | --- | --- |
| CPU | AMD Ryzen 9 9955HX（Family 26 / Zen 5，16C32T） | 正常（`machdep.xcpm.mode=0`） |
| 核显 | Radeon 890M（`1022:14d8`） | 未驱动 |
| 独显 | RTX 5060 Laptop（`10de:2d59`） | 由第三方栈 NVRM / NVKMS / NVAccel / NVRMFB 2.5.3 驱动，内屏走这里 |
| Wi-Fi | MediaTek MT7922（AMD RZ616，`14c3:0616`） | `AirportMtkWlan` 1.6.0，macOS 15/26 各自构建，5 GHz 正常 |
| 蓝牙 | MediaTek `0e8d:0616`（USB） | SimpleMtkBtFw + SimpleMtkBtPatcher，音频正常；Tahoe BLE HID 需 `-smbtlefix` |
| 有线网卡 | Realtek RTL8111 | RealtekRTL8111.kext，`en0` |
| 音频 | ALC 系列（AppleALC） | 内置扬声器 / 麦克风正常 |
| 电池 | 5650 mAh 设计容量 | SMCBatteryManager，读数正常 |

macOS 15.6 (24G84)，SMBIOS `MacBookPro16,2`。SIP 已关（自定义配置），FileVault 关闭。

## 目录

```
EFI/
├── BOOT/BOOTx64.efi          OpenCore 启动器
└── OC/
    ├── OpenCore.efi
    ├── config.plist
    ├── ACPI/                 6 个 SSDT（见下）
    ├── Drivers/              HfsPlus / OpenRuntime / ResetNvramEntry
    ├── Kexts/                22 个 kext（含 Tahoe / Sequoia 两套 Wi-Fi 构建）
    ├── Resources/            启动界面资源
    └── Tools/NullMothSafe.efi
```

## 相对原版 OpenCore 的改动

### 1. 电源适配器识别（本 EFI 的核心修复）

这台机器插上电源时 macOS 一直显示「电池供电」，`pmset -g ac` 报 `No adapter attached`。

根因：固件 DSDT 里 `\_SB.ACAD._PSR`（AC 存在检测）执行完没有返回值，内核日志 3 小时刷了 516 次
`ACPI Error: [\_SB.ACAD._PSR] ..., AE_AML_NO_RETURN_VALUE`，于是 `AppleACPIACAdapter` 实例数为 0，
电源页永远认为没插电，AC 侧的 `pmset` 策略也从不生效。

修复由两部分组成：

- `ACPI/SSDT-AC.aml`：在 `\_SB.ACAD` 下重建 `_PSR`，直接读 EC 里的 `ACIN` / `NOAC` 位
  （`OperationRegion(ERAM, SystemMemory, 0xFEDD8300, 256)` 里的 1-bit field），无返回值时兜底返回 1。
- `config.plist → ACPI → Patch`：DSDT 里 `_PSR` → `XPSR` 重命名（`Find 5f505352` / `Replace 58505352`，
  `TableSignature DSDT`，`Count 1`），让原生 `_PSR` 让位给 SSDT 里的同名方法。

验证：`ioreg -l | grep -c AppleACPIACAdapter` = 1，`pmset -g ac` 显示 60W，日志里 `AE_AML_NO_RETURN_VALUE` 归零。

### 2. 其他 ACPI

- `SSDT-EC.aml`：伪造 EC，供 VirtualSMC 使用
- `SSDT-PLUG-ALT.aml`：CPU 电源管理
- `SSDT-USB-Reset.aml` / `SSDT-USBX.aml`：USB 复位与供电参数
- `SSDT-XOSI.aml` + `config.plist` 的 `_OSI → XOSI` 重命名：让固件以为在跑 Windows

### 3. 引导参数

```
-vi2c-force-polling nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth \
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0 gfxrst=4 nvrmacpibl=1 -smbtlefix
```

- `-vi2c-force-polling`：VoodooI2C 触控板轮询
- `nvfb / nvaccel / nvfbheads / nvkmsnosmooth / gfxrst / nvrmacpibl`：RTX 5060 第三方驱动栈的参数
- `amfi_get_out_of_my_way=0x1 amfi=0x80`：允许加载未签名的显卡/无线驱动
- `-smbtlefix`：仅在 Tahoe 上启用 MT7922/RZ616 BLE HID readiness 修复

Wi-Fi 可选参数默认关闭：`-mtk160` 启用实验性 160 MHz，`-mtkhe` 启用实验性
802.11ax，`-mtkrxamsdu` 启用 RX A-MSDU 解封装。普通 WPA2/VHT 连接不需要这些参数。

### 4. 关键 Quirks

- `Kernel → Quirks → DisableIoMapper = true`：SimpleMtkWlan / NVRM 都不支持 AppleVTD
- `Misc → Security → SecureBootModel = Disabled`、`Vault = Optional`
- `AMFIPass.kext` 配合上面两个 amfi 参数

### 5. kext 启用情况

- `AMDRyzenCPUPowerManagement.kext` / `SMCAMDProcessor.kext`：**已从本 EFI 移除**。本机是 Zen 5（family 0x1A），旧版 0.7.2f1 与 DrogaBox fork 3.34.0 两种情况实测都会让内置盘概率性起不来（加了就黑屏复位、去掉就一次点亮，2026-10-23 验证），因此不再启用，也不放在 `Kexts/` 里。
- `VoodooPS2Controller.kext/Contents/PlugIns/VoodooInput.kext`：与 VoodooI2C 的手势输入重复，禁用

### 6. Wi-Fi 驱动（SimpleMtkWlan 1.6.0）

仓库包含 [e5u/SimpleMtkWlan v1.6.0](https://github.com/e5u/SimpleMtkWlan/releases/tag/v1.6.0)
的两套官方 Release 构建，不能混用 IO80211 ABI：

- `AirportMtkWlan-Tahoe.kext`：仅加载于 Darwin 25（macOS 26 Tahoe）。
- `AirportMtkWlan-Sequoia.kext`：仅加载于 Darwin 24.2–24.99（macOS 15.2+）。

两套 kext 由 `Kernel → Add` 的 `MinKernel` / `MaxKernel` 自动选择。1.6.0 包含
MT7922 PCIe `14c3:0616` 识别、MT7922 固件选择、TX 速率修复、DMA/RX 改进、
WCL roaming lock 和扩展主机测试。VHT160、802.11ax 和 RX A-MSDU 是可选实验功能；
WPA3-only (SAE) 与 AWDL 当前不支持。

### 7. 蓝牙驱动（SimpleMtkBtFw 1.1.0）

`SimpleMtkBtFw.kext` 保持官方 1.1.0 固件加载逻辑不变。`SimpleMtkBtPatcher.kext`
已加入 [SimpleMtkBtFw PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3)
的 Tahoe MT7922/RZ616 BLE HID readiness 修复，并通过 `-smbtlefix` 按需启用。

该补丁针对 Tahoe 在 LE Remote Features 后等待 PHY/Data Length readiness 事件、
导致 BLE HID 配对停住的问题；其他 macOS 版本和普通蓝牙音频路径不启用此逻辑。

## 已知问题

- **睡眠 / 关机卡死**：`pmset sleepnow` 后 14 秒 darkwake 超时，`Sleep/Wakes since boot` 恒为 0；
  关机时会留下 `shutdown_stall` 报告。嫌疑是 NVRM 栈没有 PCI 休眠回调（内屏由独显驱动，卸不掉）。
  已排除 UURemote 断言、tcpkeepalive、womp、powernap、外置卷、USB 键鼠。
- **Wi-Fi 系统统计与 WCL 兼容性仍需持续验证**：1.6.0 已加入驱动侧计数和
  roaming lock，但 macOS 系统流量统计、少数 WCL ioctl 与 `networksetup`
  查询仍可能不完整（不影响正常关联和数据传输）。
- **传感器读不到**：`powermetrics --samplers smc` 只有 `SMC version: Unknown`。AMD 传感器 kext（`SMCAMDProcessor` / `AMDRyzenCPUPowerManagement`，0.7.2f1 与 3.34.0 均试过）会导致开机概率性黑屏复位，已全部移除；GPU 温度/功耗与风扇转速在 macOS 下没有可用读法。

- **关机变自动重启 / 点「重新启动」经常开不开（2026-10-21 定位）**：macOS 关机流程会在电源状态转换处卡死（/Library/Logs/DiagnosticReports/shutdown_stall_* 每次关机一份），卡到超时后系统被强制复位、主板重新上电，表现为「点了关机，机器自己又开机」。这次复位不等于断电，dGPU/PCIe 仍是脏状态，所以紧随其后的启动常在 15%、launchd 交接等不同位置黑屏复位，且不产生 panic 文件。
  **用法建议**：需要重启时点「关机」——它会自己重启且一次点亮（相当于完成冷启动）；真的卡死不动时长按电源 10 秒断电再开。根因在 NV 显卡驱动栈缺 PCI stop/suspend 回调（与睡眠失败同源），属驱动级问题。

## 安装

1. 挂载 EFI 分区：`sudo diskutil mount disk0s1`
2. 把本仓库的 `EFI/` 整个复制进去（与原有的 `EFI/Microsoft` 共存，不要删 Windows 的启动文件）
3. 用 `OpenCore Configurator` 或 `ocvalidate` 校验 `config.plist`
4. 重启进 BIOS 把该分区加到启动顺序，或按 F8 手动选 `UEFI OS: <磁盘名>`

> 想恢复 Windows 直启：把 `EFI/Boot/bootx64.efi` 换成 `bootmgfw.efi` 的副本，或在 BIOS 里选 `Windows Boot Manager`。

## 致谢

OpenCore / Lilu / VirtualSMC / WhateverGreen / AppleALC / VoodooI2C / VoodooPS2（Acidanthera 及作者们）、
[e5u/SimpleMtkWlan](https://github.com/e5u/SimpleMtkWlan)、
[flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw)、
[SimpleMtkBtFw PR #3 / Iussch](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3)、
RealtekRTL8111（Mieze）、NVRM 显卡驱动栈（nullmoth）。
