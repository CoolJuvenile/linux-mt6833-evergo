# Linux 6.18-rc1 for Xiaomi evergo

> 给小米 Redmi Note 11T 5G / Note 11 5G（`evergo` · 联发科 MT6833 / 天玑 810）做的**主线内核**移植。
> Mainline Linux port for Xiaomi evergo (MT6833), based on Linux 6.18-rc1.

![base](https://img.shields.io/badge/base-Linux%206.18--rc1-58A6FF?style=flat&labelColor=0D1117)
![arch](https://img.shields.io/badge/arch-arm64-58A6FF?style=flat&labelColor=0D1117)
![toolchain](https://img.shields.io/badge/toolchain-LLVM%3D1-58A6FF?style=flat&labelColor=0D1117)
![status](https://img.shields.io/badge/status-WIP-orange?style=flat&labelColor=0D1117)

姊妹机型 `evergreen`（POCO M4 Pro 5G / Redmi Note 11S 5G）与 `opal`（Redmi Note 11E 5G）硬件高度相似，后续按面板与触屏差异适配即可。

## 进度

| 功能 | 状态 |
|:--|:--|
| UART0 控制台（921600） | ✅ 可用 |
| LK 帧缓冲显示 | 🟡 待验证 |
| USB 外设模式 + ttyGS0 串口 | 🟡 驱动已配齐，待实机验证 |
| eMMC | 🟠 实验性（HS200） |
| microSD | 🟠 实验性 |
| PMIC / 充电 / 按键 | ⬜ 未开始 |
| 显示 / GPU / 触屏 / 音频 / 相机 | ⬜ 未开始 |

> 这是一个**进行中的移植**，不是能日常使用的系统。刷机前请确认自己会救砖。

## 构建

推送即构建：GitHub Actions 自动产出 `evergo-mainline-kernel`，内含 `Image` 与全部 `mt6833` dtb，到 Actions 页面下载即可。

本地编译（WSL / Linux）：

```bash
make ARCH=arm64 LLVM=1 defconfig
make ARCH=arm64 LLVM=1 -j$(nproc) Image dtbs
```

## 刷写

1. 取 `Image` 与 `arch/arm64/boot/dts/mediatek/mt6833-xiaomi-evergo.dtb`
2. 参考原厂 `boot.img` 的 cmdline 与地址，用 `mkbootimg` 打包（或走 pmbootstrap）
3. **先 `fastboot boot mainline.img` 试跑，不落盘**

调试：UART 波特率 `921600`；USB 配好后插电脑看 `/dev/ttyGS0`。

## 硬件

| 部件 | 型号 / 关键点 |
|:--|:--|
| SoC | MT6833 · 4×Cortex-A76 + 4×Cortex-A55 |
| 内存 | 4 / 6 / 8 GB |
| 存储 | eMMC + microSD（GPIO4 卡检测） |
| 屏幕 | DSI 视频模式 1080×2400 · 双供应商面板（复位 GPIO86、使能 GPIO136、背光使能 GPIO87、偏压 GPIO137/138） |
| 触屏 | Novatek NT36672C · SPI（复位 GPIO13、中断 GPIO14） |
| 线性马达 | AW8697 · i2c9 @ 0x5a（复位 GPIO132、中断 GPIO6） |
| NFC | ST21NFC · i2c3 @ 0x08 |
| 指纹 | FPC · spi5（复位 GPIO17、中断 GPIO18） |
| USB | musb USB2 |

## 路线图

- [ ] 实机验证控制台与帧缓冲地址
- [ ] 确认 PMIC 型号（MT6357 / MT6359），启用 pwrap
- [ ] 调通 eMMC，挂根文件系统
- [ ] mmsys / drm 显示通路 + 面板驱动
- [ ] NT36672C 的 SPI 触屏驱动
- [ ] USB host、SMP、温控

详细移植笔记与踩坑记录见 [EVERGO.md](EVERGO.md)。

## 许可

内核本身为 GPL-2.0，见 [COPYING](COPYING)。
