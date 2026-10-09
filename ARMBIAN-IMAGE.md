# WF7000A Armbian 卡刷镜像（云端构建）

为 **WF7000A**（Rockpi-4B 类 / RK3399）构建可直接写入 TF 卡启动的 Armbian 镜像，
用于 **eMMC 上 DTB 损坏导致无法开机** 的救砖场景。

## 一、它和官方 rockpi-4b 镜像差在哪

| 项 | 官方 rockpi-4b 镜像 | 本镜像 |
|---|---|---|
| 板型 / 内核 / 发行版 | `rockpi-4b` / `current` / `trixie` | **相同**（板子原本跑的就是 `linux-u-boot-rockpi-4b-current` + Debian trixie） |
| `/boot/dtb*/rockchip/rk3399-wf7000a.dtb` | 无 | **有**（82590 B，sha256 `a7ce6ea01f92b903512170612ceeeb0c3d101aaac7236fdc5476038af41a3f86`；源 = 仓库根 `rk3399-wf7000a.dts`） |
| `armbianEnv.txt` 的 `fdtfile` | `rockchip/rk3399-rock-pi-4b.dtb` | `rockchip/rk3399-wf7000a.dtb` |
| 首启 | 交互式向导 | **无人值守**（预设 root/user/口令/时区/有线 DHCP） |

其余（内核包、u-boot、用户空间）**全部是 Armbian 官方构建产物，未做任何精简或改动**。

## 二、为什么注入放在「成品镜像」上做，而不是 Armbian 的 customize 钩子

读 `armbian/build` 源码得到的时序（`lib/functions/main/rootfs-image.sh`）：

```
customize_image()                      ← userpatches/customize-image.sh 在这里跑
  ...
post_debootstrap_tweaks()              ← /boot 引导脚本在这里收尾
  ...
create_image_from_sdcard_rootfs()      ← 到这里才生成 .img
```

`customize_image()` 跑在引导脚本收尾**之前**，改 `armbianEnv.txt` 会被覆盖。
所以 workflow 改成：**先用官方流程产出完整 .img，再 losetup 挂载、就地替换 DTB 与 fdtfile**。
时序从此与 Armbian 内部实现无关。

## 三、怎么用

### 1. 取镜像
Actions 跑完后，在 **Releases → `armbian-image-latest`** 下载
`wf7000a-armbian-trixie-rockpi4b.img.xz` 与同名 `.sha256`。

### 2. 校验
```bash
shasum -a 256 -c wf7000a-armbian-trixie-rockpi4b.img.xz.sha256
```

### 3. 写卡（**会抹掉 TF 卡，别选错盘**）
```bash
# macOS：先用 diskutil list 确认目标盘，例如 /dev/disk4
xz -dc wf7000a-armbian-trixie-rockpi4b.img.xz | sudo dd of=/dev/rdiskN bs=1m
diskutil eject /dev/diskN
```
或直接用 balenaEtcher / Raspberry Pi Imager 写 `.img.xz`。

### 4. 首次开机
- TF 卡插上，上电。RK3399 的 BootROM 在 eMMC 引导失败时会自动落到 SD/TF。
- 走的是 Armbian **无人值守首启**（`/root/.not_logged_in_yet` 已预设），
  以太网走 DHCP，无需接显示器和键盘。

### 5. 登录
```
用户：mobius     口令：wf7000a
root：root       口令：wf7000a
```
> ⚠️ **`wf7000a` 是一次性临时口令，本仓库是公开仓库，所以真实口令绝不能写进来。
> 首次登录后立刻执行：**
> ```bash
> passwd          # 改 root
> passwd mobius   # 改用户
> ```

### 6. 确认 DTB 生效
```bash
cat /proc/device-tree/model
# 期望 WF7000A RK3399 Board
sha256sum /boot/dtb/rockchip/rk3399-wf7000a.dtb
# 期望 a7ce6ea01f92b903512170612ceeeb0c3d101aaac7236fdc5476038af41a3f86
grep fdtfile /boot/armbianEnv.txt
```

## 四、救 eMMC 的路子（TF 起来之后）

TF 卡启动进系统后，挂载 eMMC 根分区，把能开机的 DTB 写回去：

```bash
lsblk
sudo mkdir -p /mnt/emmc
sudo mount /dev/mmcblk1p1 /mnt/emmc          # eMMC 盘号以 lsblk 为准
sudo cp /boot/dtb/rockchip/rk3399-wf7000a.dtb \
        /mnt/emmc/boot/dtb/rockchip/rk3399-wf7000a.dtb
sudo sed -i 's|^fdtfile=.*|fdtfile=rockchip/rk3399-wf7000a.dtb|' /mnt/emmc/boot/armbianEnv.txt
sync && sudo umount /mnt/emmc
```

再把启动顺序改回 eMMC（拔 TF 卡或 `armbian-install`）。**这条路不丢数据。**

## 五、仓库内文件

| 路径 | 作用 |
|---|---|
| `rk3399-wf7000a.dts` | **DTB 的源文件（唯一正确来源）**。与上游 `arch/arm64/boot/dts/rockchip/` 同构，可直接随树编译 |
| `.github/workflows/build-armbian-image.yml` | 构建 + 镜像手术 + 出 Release |
| `assets/rk3399-wf7000a.dtb.xz.b64` | 自定义 DTB（xz + base64，workflow 里按 `DTB_SHA` 校验） |
| `assets/rk3399-wf7000a.dtb.GOOD` | **vendor 已知能开机**的旧 DTB（84806 B / `221c1da4…`），仅作回退参照，CI 不使用 |
| `assets/armbian-firstrun-preset` | 无头首启预设，落到镜像 `/root/.not_logged_in_yet` |

自行编译（本机或 CI 均可）：
```bash
# 需要一个含 rk3399-sapphire.dtsi 的 6.18.y 内核树（如 ophub/linux-6.18.y）
clang -E -nostdinc -undef -D__DTS__ -x assembler-with-cpp \
      -I include -I arch/arm64/boot/dts/rockchip rk3399-wf7000a.dts -o /tmp/wf7000a.dts.pre
dtc -@ -b 0 -I dts -O dtb /tmp/wf7000a.dts.pre -o rk3399-wf7000a.dtb
sha256sum rk3399-wf7000a.dtb   # 期望 a7ce6ea0…
```

另有历史路线 `.github/workflows/build-dtb.yml`（编 6.1 树的旧 `rk3399-wf7000.dts`）与
`build_dtb.sh` / `rk3399-wf7000.dts` / `wf7000.dts`，均**已废弃**，与现役流程无关联，勿再使用。

## 六、失败排障

构建失败时 workflow 会把构建日志尾部写回仓库 `.build-logs/last-failure.log`，
可直接读该文件定位问题，不必爬 Actions 页面。

---

## 七、能不能用 U 盘引导？—— **不能**（三条硬证据）

结论先说：**把这个镜像写到 U 盘上、插到 WF7000A 上，是引导不起来的。**
不是镜像的问题（镜像本身格式上 SD / USB 通用），是 RK3399 的引导链决定的——
下面两个独立原因，任一个都足以否定。

**1. BootROM 根本不认 USB 存储设备。**
RK3399 的 SoC 内部 BootROM，启动介质顺序是**焊死的**：SPI → eMMC → SD 卡。
USB 不在这个列表里。Pine64 官方 wiki 在 "Different devices" 一节明确列出
`not directly bootable: NVMe, USB 3, WiFi`；Radxa 官方论坛对同款问题的答复是
`boot priorities are fixed in hardware: NVM → SDCARD → eMMC → USB`。

**2. USB 只能等 u-boot 起来后由 u-boot 枚举，而 u-boot 把 USB 排在最后。**
mainline `include/configs/rockchip-common.h` 原文（2026-10-09 取自 master）：

```c
#ifndef BOOT_TARGETS
#define BOOT_TARGETS	"mmc1 mmc0 nvme scsi usb pxe dhcp spi"
#endif
```

SD 是 `mmc1`、eMMC 是 `mmc0`，**USB 排第 5 位**。

**3. 这块板子的 eMMC 引导是好的**（坏的只有 `/boot` 里那份 DTB）。
所以 u-boot 一定会先找到 eMMC 上的 `/boot/boot.scr` → 加载 `Image` + 那份坏 DTB
→ 内核卡死。**它永远走不到 `usb0`**，插 U 盘等于没插。

### 还有一道更硬的坎：Armbian 的 u-boot 没编 USB 大容量存储驱动

| 检查项 | 结果 |
|---|---|
| `configs/rock-pi-4-rk3399_defconfig`（u-boot master） | 有 `CONFIG_CMD_USB` / `CONFIG_USB_XHCI_HCD` / `CONFIG_USB_DWC3` / `CONFIG_USB_KEYBOARD` / `CONFIG_USB_HOST_ETHER`，**没有 `CONFIG_USB_STORAGE`** |
| `config USB_STORAGE` 定义（`drivers/usb/Kconfig:95`） | `bool "USB Mass Storage support"`，**无 `default y`** |
| 谁 select 它 | 全树只有 `board/tq/tqma6` / `board/intel/slimbootloader` / `arch/arm` 三个无关处 → **不会被隐式打开** |
| Armbian 是否补这一项 | `config/sources/families/include/rockchip64_common.inc` 与 `patch/u-boot/u-boot-rockchip64/` 均无 |

⇒ 即便 u-boot 真的轮到了 `usb0`，它也**读不了 U 盘**。

**唯一能让 U 盘方案成立的前提**：把 u-boot 换成带 `CONFIG_USB_STORAGE=y`、
且 `boot_targets` 把 usb 提前的版本（见下面路线 C）。

## 八、不用 TF 卡的四条路

| 路线 | 需要什么 | 丢数据？ | 说明 |
|---|---|---|---|
| **A. 串口（USB-TTL）** | 3.3V USB-TTL 线（TX/RX/GND），UART2 / `ttyS2`，1500000 8N1 | ❌ 不丢 | **最省事**。打断 u-boot 后改用发行版自带的 `rk3399-rock-pi-4b.dtb` 手动引导进系统，再修 eMMC 上那份坏 DTB。也能一秒钟看清到底卡在哪一层 |
| **B. maskrom + rkdeveloptool** | USB-C（OTG）数据线 + **按住 MASKROM 键上电**；Mac 上需装 rkdeveloptool | 整盘写=丢；**只写前 16MiB 引导区=不丢** | Radxa 官方流程 `rkdeveloptool ld` → `db` → `wl`。WF7000A 的 MASKROM/RESET 键位置需现场确认 |
| **C. 云端重编 u-boot（USB-first）** | 同 B（需要 maskrom 才能写引导区） | ❌ 不丢 | 用同一套 CI 产出 `idbloader.img` + `u-boot.itb`（`CONFIG_USB_STORAGE=y` + `boot_targets="usb0 mmc1 mmc0"`），写进前 16 MiB，之后 U 盘即可直接引导 |
| **D. 借一张 TF 卡** | 一张 TF 卡 + 读卡器 | ❌ 不丢 | 硬件优先级里 **SD 高于 eMMC**，插卡即引导，理论最短路径 |

### ⚠️ 一个必须先说清的判断

DTB 等价性验证的结论是 **新旧 DTB 功能完全等价**（节点 527 = 527、phandle 357 = 357、
637 处值差异 100% 归因于 phandle 重编号、`__symbols__` 仅多一个手工命名的 `fusb0_int`）。
**如果这个结论成立，那这次无法开机就未必是 DTB 造成的**，"把好 DTB 换回去"也未必救得活。

所以动手刷任何东西之前，**建议先接串口看一眼**（路线 A）：

- u-boot 有输出 → 卡在哪一步？
- 停在 `Starting kernel ...` 之后 → DTB / 内核参数问题
- 卡在挂载 rootfs → 文件系统损坏（那次硬断电的嫌疑很大）
- 完全没有输出 → 引导层真损坏

这一眼能省掉一整轮盲目刷机。

## 九、Release 资产

tag：`armbian-image-latest`（prerelease），每次构建覆盖同名资产。

| 资产 | 说明 |
|---|---|
| `wf7000a-armbian-trixie-rockpi4b.img.xz` | 解压后约 2544 MiB |
| `wf7000a-armbian-trixie-rockpi4b.img.xz.sha256` | 该次构建对应的校验值 |

> ⚠️ **sha256 不是固定值**：Armbian 构建非可复现（时间戳/包版本），每次跑出来的
> `.img.xz` 指纹都不同，**一律以同次产出的 `.sha256` 为准**，不要拿历史指纹比对。
>
> 当前 Release 上那份仍是**旧载荷**（DTB `221c1da4…`）构建的；本仓库已把载荷换成
> `a7ce6ea0…`，需要重跑一次 `build-armbian-image.yml` 才会覆盖成新版。

取回：
```bash
gh release download armbian-image-latest --repo Mobius-W/wf7000a --pattern "*.img.xz*" --dir .
```
