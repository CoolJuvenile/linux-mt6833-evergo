# evergo 主线 Linux

Xiaomi Redmi Note 11T 5G / Redmi Note 11 5G(evergo,联发科 MT6833 / 天玑 810)的主线内核移植。

姊妹机型 evergreen(POCO M4 Pro 5G / Redmi Note 11S 5G)和 opal(Redmi Note 11E 5G)硬件高度相似,后续按面板和触屏差异适配即可。

## 硬件信息

- SoC:MT6833,4×Cortex-A76 + 4×Cortex-A55
- 内存 4/6/8 GB,eMMC + microSD(GPIO4 卡检测)
- 屏幕:DSI 视频模式 1080×2400,双供应商面板(复位 GPIO86、使能 GPIO136、背光使能 GPIO87、偏压 GPIO137/138)
- 触屏:Novatek NT36672C,SPI 接口(复位 GPIO13、中断 GPIO14)
- 线性马达:AW8697(i2c9 @ 0x5a,复位 GPIO132、中断 GPIO6)
- NFC:ST21NFC(i2c3 @ 0x08)
- 指纹:FPC(spi5,复位 GPIO17、中断 GPIO18)
- USB:musb USB2

## 当前进度

| 功能 | 状态 |
| --- | --- |
| UART0 控制台(921600) | 可用 |
| LK 帧缓冲显示 | 待验证(地址取自下游默认值,黑屏就调地址/尺寸) |
| USB 外设模式 + ttyGS0 串口 | 驱动已配齐,待实机验证 |
| eMMC | 实验性(HS200) |
| microSD | 实验性 |
| PMIC / 充电 / 按键 | 未开始(待确认 PMIC 型号后启用 pwrap) |
| 显示 / GPU / 触屏 / 音频 / 相机 | 未开始 |

## 编译

推送后 GitHub Actions 自动构建,产物在 Actions 页面(evergo-mainline-kernel,含 Image 和所有 mt6833 dtb)。

本地编译(WSL / Linux):

    make ARCH=arm64 LLVM=1 defconfig
    make ARCH=arm64 LLVM=1 -j$(nproc) Image dtbs

## 刷写

1. 下载 `Image` 和 `arch/arm64/boot/dts/mediatek/mt6833-xiaomi-evergo.dtb`
2. 参考原厂 boot.img 的参数,用 mkbootimg 打包(或走 pmbootstrap)
3. 先 `fastboot boot mainline.img` 试跑,不落盘

调试输出:UART 波特率 921600;USB 配好后插电脑看 `/dev/ttyGS0`。

## TODO

1. 实机验证控制台与帧缓冲地址
2. 确认 PMIC 型号(MT6357/MT6359),启用 pwrap
3. 调通 eMMC,挂根文件系统
4. mmsys/drm 显示通路 + 面板驱动
5. NT36672C 的 SPI 触屏驱动
6. USB host、SMP、温控
