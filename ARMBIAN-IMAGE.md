# WF7000A Armbian 卡刷镜像（云端构建）

为 **WF7000A**（Rockpi-4B 类 / RK3399）构建可直接写入 TF 卡启动的 Armbian 镜像，
用于 **eMMC 上 DTB 损坏导致无法开机** 的救砖场景。

## 一、它和官方 rockpi-4b 镜像差在哪

| 项 | 官方 rockpi-4b 镜像 | 本镜像 |
|---|---|---|
| 板型 / 内核 / 发行版 | `rockpi-4b` / `current` / `trixie` | **相同**（板子原本跑的就是 `linux-u-boot-rockpi-4b-current` + Debian trixie） |
| `/boot/dtb*/rockchip/rk3399-wf7000a.dtb` | 无 | **有**（sha256 `221c1da472f4f5a221028a41d821068be9f9accf419ca8d5ee1a6f07cfb23482`） |
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
sha256sum /boot/dtb/rockchip/rk3399-wf7000a.dtb
# 期望 221c1da472f4f5a221028a41d821068be9f9accf419ca8d5ee1a6f07cfb23482
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
| `.github/workflows/build-armbian-image.yml` | 构建 + 镜像手术 + 出 Release |
| `assets/rk3399-wf7000a.dtb.xz.b64` | 自定义 DTB（xz + base64，workflow 里校验 sha256） |
| `assets/armbian-firstrun-preset` | 无头首启预设，落到镜像 `/root/.not_logged_in_yet` |

另有历史路线 `.github/workflows/build-dtb.yml`（只编 standalone DTB），与本流程互不影响。

## 六、失败排障

构建失败时 workflow 会把构建日志尾部写回仓库 `.build-logs/last-failure.log`，
可直接读该文件定位问题，不必爬 Actions 页面。
