# evergo 主线 Linux (mainline) 内核移植

为 **Xiaomi Redmi Note 11T 5G / Redmi Note 11 5G(代号 evergo,MT6833 / 天玑 810)**
移植主线 Linux 内核的工程。

姊妹机型 evergreen(POCO M4 Pro 5G / Redmi Note 11S 5G)与 opal(Redmi Note 11E 5G)
硬件高度相似(下游三机共用统一内核 "everpal"),本移植以 evergo 为首要目标,
其余两机预计仅需替换面板/触屏等板级差异。

## 移植思路

不重新发明轮子,分三层取材:

| 来源 | 用途 |
| --- | --- |
| [MT6833-Mainline/linux-mt6833](https://github.com/MT6833-Mainline/linux-mt6833) | **基座**。camellia(POCO M3 Pro 5G,同 SoC)的主线化树:v6.18-rc1+ 带来 mt6833 的 SoC dtsi、pinctrl/clk 驱动、camellia 板级 DTS |
| [Addster09/android_kernel_xiaomi_mt6833](https://github.com/Addster09/android_kernel_xiaomi_mt6833) 与 [Xiaomi-MT6833/kernel_xiaomi_evergo](https://github.com/Xiaomi-MT6833/kernel_xiaomi_evergo) | **硬件事实**。下游 evergo DTS:GPIO 分配、I2C/SPI 设备地址、面板/触屏型号 |
| [MT6895-Mainline](https://github.com/MT6895-Mainline)(kde-yyds 的 MT6895/xaga 主线化) | **工程组织方式**。分支命名(如 `7.2-mt6895-xiaomi-xaga`)、initramfs/rootfs/modem 配套仓库、系统侧(quirks、ModemManager)的做法,后续 evergo 起飞后照此建仓 |

本仓库(`patches/`)产出的是叠加在 linux-mt6833 master 之上的 **evergo 补丁序列**;
`linux/` 是完整组装树(linux-mt6833 master 整树导入为一个基线提交 + evergo 板级支持提交),
可直接推送为 GitHub 仓库并用内置 CI 构建。

## evergo 硬件要点(来自下游 DTS)

- SoC:MT6833(4×A55 + 4×A76?见 mt6833.dtsi 注),PMIC 待确认(MT6357 或 MT6359 + MT6360 伴侣芯片,中断 GPIO10)
- 内存:4/6/8 GB;eMMC + microSD(GPIO4 卡检测,上升沿有效)
- 面板:DSI 视频模式 1080×2400,双供应商 `k16a_36_02_0a_vdo` / `k16a_42_02_0b_vdo`
  (复位 GPIO86、使能 GPIO136、背光使能 GPIO87、偏压 GPIO137/138、TE GPIO84)
- 触屏:Novatek NT36672C,**SPI 接口**(复位 GPIO13、中断 GPIO14)——主线现有
  novatek 驱动仅支持 I2C,需要新写/移植 SPI 变体
- 线性马达:Awinic AW8697(i2c9 @ 0x5a,复位 GPIO132、中断 GPIO6)
- NFC:ST21NFC(i2c3 @ 0x08,复位 GPIO92、中断 GPIO5)
- 指纹:FPC(spi5,复位 GPIO17、中断 GPIO18)
- USB:musb(USB2,OTG)
- 摄像头(i2c 地址):主摄 0x5a(对焦 0x18)、超广 0x2b、景深 0x12、微距 0x71、前摄 0x7d

## 当前状态(诚实版)

以 camellia 主线化的成熟度为参照,evergo 处于**第一步 bring-up**:

| 功能 | 状态 |
| --- | --- |
| UART0 控制台(921600) | ✅ 预期可用(引脚 69/70 与 camellia 同为 SoC 固定复用) |
| lk 引导的 simple-framebuffer | ⚠️ 地址取自下游 atag 默认值(0x7e605000/0x1be0000),需实机验证 |
| USB2 musb(外设/OTG) | ⚠️ 上游树自带 "broken usb" workaround,host 模式不可靠 |
| microSD | ⚠️ 实验性(camellia 上 SD 也尚未修好) |
| eMMC(mmc0) | ❌ 未启用(下一步) |
| pwrap/PMIC | ❌ 未启用——先在实机确认 PMIC 型号(MT6357 vs MT6359)再加 |
| 显示 DSI / GPU / 触屏 / 音频 / 传感器 / 充电 | ❌ 未开始 |

在 camellia 上已知的问题同样会影响 evergo(同一基座):SD 卡、USB host、
以及 pmOS wiki 提到的"目前只在 CPU0 上启动"的 SMP 问题。

## 构建

**方法一:GitHub Actions(推荐,无需本地 Linux)**

把组装好的 `linux/` 推到你 fork 的仓库,`.github/workflows/build-kernel.yml`
会自动构建,产物在 Actions Artifacts(`Image` + `mt6833-xiaomi-*.dtb`)。

**方法二:本地(WSL/任意 Linux)**

```sh
sudo apt install bc bison flex libssl-dev libelf-dev dwarves clang lld llvm
make ARCH=arm64 LLVM=1 defconfig
make ARCH=arm64 LLVM=1 -j$(nproc) Image dtbs
# 产物: arch/arm64/boot/Image, arch/arm64/boot/dts/mediatek/mt6833-xiaomi-evergo.dtb
```

## 刷写与测试(evergo 已解锁 bootloader)

```sh
# 用下游 boot.img 的头部 + 主线 Image 打包(最简验证路径):
#   mkbootimg --kernel Image --dtb mt6833-xiaomi-evergo.dtb \
#            --base 0x40000000 --kernel_offset 0x40080000 ... -o mainline.img
# 或使用 postmarketOS 的 pmbootstrap(参考 xiaomi-camellia 的移植方式)。
fastboot boot mainline.img        # 不落盘,先试 UART 控制台
```

测试要点:UART TX/RX 在主板测试点(与 camellia 类似,可参考拆机图);
`earlycon` 已开启,上电即应出字。屏幕亮不亮取决于 lk framebuffer 地址是否猜对,
不影响 UART 调试。

## 路线图

1. **实机验证**:UART 控制台出字 → framebuffer 地址/尺寸修正 → fastboot boot 落盘
2. **eMMC(mmc0)**:启用节点 + 固定引脚,挂上根文件系统(rootfs 建议 UFS→eMMC 分区或 USB 网络 + initramfs)
3. **PMIC**:读 pwrap ID 确认 MT6357/MT6359 → 引入/编写对应 dtsi(上游已有 mt6359 基础)→ 供电、充电、按键
4. **显示**:mt6833 的 mmsys/disp 驱动上游化(参考 MT6895-Mainline 的 mmsys 提交)+ k16a 面板驱动
5. **触屏**:novatek SPI 驱动
6. **USB host 修复、SMP、USB 网络/adb**
7. **系统侧**:照 MT6895-Mainline 建 initramfs/rootfs/quirks 仓库,跑 postmarketOS/Ubuntu touch

## 目录

- `patches/` — 叠加在 linux-mt6833@master 之上的 evergo 补丁(git format-patch)
- `linux/` — 组装完成的完整内核树(含 CI)
- `references/` — 下游 evergo DTS 等参考材料
- `fetchdelta.py` / `ghget.py` — 本工程在受限网络下重建 linux-mt6833 内容用的工具

## 致谢

- [MT6833-Mainline](https://github.com/MT6833-Mainline)(evilMyQueen 等)的 camellia/mt6833 主线化
- [MT6895-Mainline](https://github.com/MT6895-Mainline)(kde-yyds)的 MT6895/xaga 主线化与系统配套
- [Addster09](https://github.com/Addster09) 与 xiaomi-mt6833-dev / Xiaomi-MT6833 的下游 everpal/evergo 内核
