# WF7000A 板级设备树（RK3399 / mainline Linux）

WF7000A 工控板的板级 DTS/DTB。SoC 为 Rockchip RK3399（与 Rock Pi 4B 同平台），运行 mainline Linux（Armbian）。本仓库只保留两个文件，职责严格分离：

| 文件 | 大小 | 出处 | 定位 |
|---|---|---|---|
| [`rk3399-wf7000a.dts`](rk3399-wf7000a.dts) | 5,332 B | 以 mainline 内核 upstream-6.18 同族板级 dts 为模板**重写** | **终版**，编译部署用这个 |
| [`wf7000.dts`](wf7000.dts) | 97,816 B | 原厂安卓固件 dtb 经 `dtc` 反编译得到 | **硬件真相存档**，仅作对照，**勿直接部署** |

> 一句话：`rk3399-wf7000a.dts` 是能跑的终版；`wf7000.dts` 是「这块板到底怎么接的」权威档案。

## 一、`wf7000.dts` 的出处

**它是怎么来的**：从设备原厂安卓固件中解包出 dtb，用设备树编译器反编译：

```bash
dtc -I dtb -O dts -o wf7000.dts <原厂固件解包出的.dtb>
```

**它是什么形态**：一个**扁平的完整反编译产物**——约 97 KB、上千行、大量数字 phandle，**不是干净的源码**，也不适合直接阅读维护。

**它的价值**：它是**硬件连线真相**的权威来源——引脚复用、供电（regulator）、时钟、以及**厂方自定义 `compatible` 节点**都如实记录在里面。终版 dts 的每一处硬件相关决策，都以它作对照基准。

**它的限制**：它描述的是**原厂安卓内核**（旧内核 + 厂方私有 `.ko` 驱动 + 旧版 binding）。把它直接拿去 mainline 编译**跑不起来**——它只是档案，不是部署物。

## 二、mainline dts（`rk3399-wf7000a.dts`）编写过程

### 2.1 核心认知：不存在「转换」，只有「重写」

Android vendor dts 和 Linux dtb 是同一种二进制格式。真正的移植工作不是「把 `wf7000.dts` 转换成 mainline」，而是：

> **以 mainline 同族板级 dts 为模板，照 mainline binding 重写，并把厂方自定义节点的语义补回来。**

`wf7000.dts` 在这条路线里**不是必要条件**——终版以 upstream-6.18 同族 dts 作编译依赖，反编译产物只作对照。

### 2.2 选对基线

基线 = **目标内核自带的同族板级 dts**：

- 在板子上 `ls /boot/dtb/rockchip/`，找 compatible 最接近的同族 dtb，用板子自带 `dtc` 反编译；或
- 直接取内核源码同族 `.dts` / `.dtsi`（本仓库用 `upstream-6.18`）。

### 2.3 三方对照

| 列 | 来源 | 作用 |
|---|---|---|
| A | 硬件真相：`wf7000.dts` 佐证 + 板子 dtc 反编译同族 dtb + 运行期实测 | 硬件到底怎么接 |
| B | mainline 模板 / 基线 dts | 照它写 binding |
| C | 实测裁判：上板跑，读 sysfs / dmesg | 谁对听谁的 |

### 2.4 关键实战坑（mainline 文档不写，逐个踩过）

1. **厂方自定义 `compatible` 节点**（`gh_gpio_sel`、`led_ctrl`、`rockchip,key` 等）：原厂由 `.ko` 驱动在启动时拉几根 GPIO。mainline 没这个驱动 → GPIO 悬空 → **整套外设断路**。典型症状是「软件层全绿但子系统不通」，必须把语义补回 mainline binding。
2. **gpio bank 地址以实测为准**：本板 `gpio3` 在 `@ff788000`，不是标准 `@ff7b0000`——不能照抄标准地址，要现场查 phandle。
3. **`CONFIG_GPIO_HOG` 没开时 `gpio-hog` 静默失效**：改用 `regulator-fixed` 节点替代。
4. **`of_dapm_routes` 失败打印像 warning，实际让整张声卡注册失败**（`Failed to add route ...` 是致命的）——别被日志级别骗了。
5. **内核升级会冲掉自定义 DTB 导致变砖**：装 `/etc/kernel/postinst.d/` 钩子 + `DPkg::Post-Invoke` 双兜底，跨 6.12 → 6.18 多版本实测有效。

### 2.5 编译与部署

```bash
# 依赖：upstream-6.18 内核源码（arch/arm64/boot/dts + include）
KERNEL_SRC=upstream-6.18

cpp -nostdinc \
    -I "$KERNEL_SRC/arch/arm64/boot/dts" \
    -I "$KERNEL_SRC/arch/arm64/boot/dts/rockchip" \
    -I "$KERNEL_SRC/include" \
    -undef -D__DTS__ rk3399-wf7000a.dts \
  | dtc -I dts -O dtb -o rk3399-wf7000a.dtb -

# 部署到板子
scp rk3399-wf7000a.dtb <board>:/boot/dtb/rockchip/rk3399-wf7000a.dtb
```

部署后务必装 postinst 钩子（见 2.4-5），否则内核升级会覆盖回厂方 dtb。

### 2.6 当前状态

- 已在 Armbian mainline 内核（6.x）上运行使用。
- 曾排查「刷新新 DTB 后 HDMI 黑屏」，通过对比 vendor DTB 与 upstream 同族 dts 定位根因（设备树问题，非硬件损坏）。

## 三、文件校验

| 文件 | 字节 | md5 |
|---|---|---|
| rk3399-wf7000a.dts | 5332 | `8260340c9efac852bf259a769dd614f1` |
| wf7000.dts | 97816 | `0dc3db657ee9ce83719a3bd622be5da7` |

---

*本仓库为 WF7000A 板级设备树档案：终版 dts 可直接编译部署；反编译产物仅作硬件真相对照。*
