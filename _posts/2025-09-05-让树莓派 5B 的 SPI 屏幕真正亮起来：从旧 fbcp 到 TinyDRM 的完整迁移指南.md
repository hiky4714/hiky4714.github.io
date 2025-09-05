---
layout:     post
title:      让树莓派 5B 的 SPI 屏幕真正亮起来：从旧 fbcp 到 TinyDRM 的完整迁移指南
date:       2025-09-05
header-img: img/post-bg-desk.jpg
catalog: true
---

> 你是否也遇到：树莓派 5B 接 SPI 屏幕只亮背光却不显示任何内容？这其实是因为 **fbcp** 依赖的旧显示栈在新硬件上已被移除。本文用一键可复制的方式，带你从「白屏」到「流畅播放 1080P」。

---

## 1 发生了什么：为什么 Pi4 的配置在 Pi5 上失效？

- **核心变动**：Pi5 彻底移除了 **legacy display stack**，靠 `DispmanX` 工作的 `fbcp`、`fbcp-ili9341` 无法调用 VideoCore，会报  
  ```
  * failed to open vchiq instance
  ```
- **新架构**：只有通过 **DRM/KMS** 或 **TinyDRM** 驱动注册的 SPI 屏幕，才能被现代桌面识别为独立显示器，支持硬件加速旋转、窗口拖动、甚至 OpenGL。

---

## 2 三步为 SPI 小屏装上「官方驱动」

> 以下命令默认你使用 **Raspberry Pi OS 64 位 Bookworm（2023-12-05 以后版本）**。

### 2.1 断开 HDMI（可选）
当你只有 SPI 屏幕时，可移除 HDMI，系统会自动选中下一个可用 DRM 设备。

### 2.2 写入固件与设备树覆盖层

以 1.3″ 240×240、ST7789W 驱动为例——该驱动同级兼容 Joy-IT SBC-LCD01 与 Waveshare SKU19650。

```bash
# 1. 下载固件并复制
sudo wget https://github.com/raspberrypi/firmware/raw/master/boot/overlays/mipi-dbi-spi.dtbo -O /boot/firmware/overlays/mipi-dbi-spi.dtbo
sudo wget https://forums.raspberrypi.com/download/file.php?id=63123 -O /lib/firmware/wavesku19650.bin

# 2. 编辑 /boot/firmware/config.txt
cat <<'EOF' | sudo tee -a /boot/firmware/config.txt

# ST7789 1.3" SPI display
dtoverlay=mipi-dbi-spi,speed=32000000
dtparam=compatible=wavesku19650\0panel-mipi-dbi-spi
dtparam=write-only,cpha,cpol
dtparam=width=240,height=240,width-mm=23,height-mm=23
# 按实际引脚分配，以下为官方参考
dtparam=reset-gpio=27,dc-gpio=25,backlight-gpio=18
EOF
```

### 2.3 接线（→ 40pin GPIO）

| 功能 | 图标引脚 | BCM 编号 |
|---|---|---|
| BL   | 12 | 18 |
| DC   | 22 | 25 |
| RST  | 13 | 27 |
| DIN  | 19 | 10  (MOSI) |
| CLK  | 23 | 11  (SCLK) |
| CS   | 24 | 8   (CE0) |
| GND  | 20 | — |
| VCC  | 17 | — |

完成后 `sudo reboot`，会看到「桌面扩展」自动出现；若仍空白，请核查 **供电 ≥1 A** 及 **固件日期 ≥2023-12-05**。

---

## 3 验证：你的 SPI 屏幕已变成 `/dev/dri/card2`

```bash
# 查看是否注册成功
$ dmesg | grep mipi-dbi
# 预期出现 panel-mipi-dbi-spi, 240x240

# 列出显卡
$ ls /dev/dri/
# card0 = HDMI，card1 = VC4，card2 = SPI
```

---

## 4 把视频「丢」到 SPI 屏幕

在仅远程 ssh 的场景，常用以下两组命令切换输出目标：

```bash
# 确认 card2（SPI）上有 framebuffer
export DISPLAY=:0
export WAYLAND_DISPLAY=wayland-0

# 以 DRM 后端全屏播放
cvlc --vout=drm --fullscreen /opt/videos/test.mp4 &
# 若想应对 90° 旋转，再追加
# --transform-type=90
```

播放时你会惊讶：1080P 片源也能保持 25–30 fps，虽然颜色阶梯略显粗糙，但远超旧 fbcp 方案。

---

## 5 常见问题与解答（FAQ）

**Q1：为什么我已按教程连接，但 dmesg 无 mipi-dbi 日志？**  
A：大概率停留在 32 位系统。Pi 5 + TinyDRM 目前仅官方维护 64 位驱动，务必重装 64 位镜像。

**Q2：我想保留 HDMI，并自动扩展而非复制桌面，可以么？**  
A：可以。SPI 与 HDMI 是两张**独立** DRM 卡。进入「首选项 → 屏幕配置」即可拖拽排列。复制模式需自己写 composite-wlroots，目前社区实验阶段。

**Q3：Wayfire 会不会把 SPI 屏作为主屏导致无桌面？**  
A：若 HDMI 断开，SPI 会晋升为主屏；系统会正常显示登录界面。后续插入 HDMI 需手动「屏幕配置」切回。

**Q4：旧项目必须用 `/dev/fb0` 怎么办？**  
A：TinyDRM 已自带 **fbdev emulation**，加载后仍为 `/dev/fb0`，故兼容最老的应用。

---

## 6 扩展阅读 & 未来规划

- 官方正整合 dtoverlay，未来会提供 **一张覆盖层 + panel 参数** 的方式，减少重复编译。  
- 已确认支持及待测列表（社区计划）：ST7735、ILI9341、SSD1306、GC9A01。

👉 [快速掌握最新 SPI 屏兼容列表，一键不再踩坑](https://okxdog.com/)

---

## 7 小结：仅需记住的三句话

1. **Pi5 无旧 Stack，停止 fbcp**。  
2. **TinyDRM = DRM + 固件文件**，加载即正常桌面。  
3. **SPI 屏幕与 HDMI 独立**，可扩展、可旋转、硬件加速。

👉 [现在就动手，十分钟让树莓派 5B 的 SPI 屏变生产力副屏！](https://okxdog.com/)