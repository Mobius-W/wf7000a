# assets/

| 文件 | 说明 |
|---|---|
| `rk3399-wf7000a.dtb.xz.b64` | 注入镜像的 DTB 载荷（`xz -9` / CRC64 + base64 每 1000 字符折行）。CI 先解码、再按 workflow 里的 `DTB_SHA` 强校验，不匹配即刻失败 |
| `rk3399-wf7000a.dtb.GOOD` | **vendor 已知能开机**的旧 DTB（84806 B / sha256 `221c1da472f4f5a221028a41d821068be9f9accf419ca8d5ee1a6f07cfb23482`）。仅作回退参照，CI 不使用 |
| `armbian-firstrun-preset` | 无头首启预设，落到镜像 `/root/.not_logged_in_yet` |

## 当前 DTB 载荷

- 大小：82590 B
- sha256：`a7ce6ea01f92b903512170612ceeeb0c3d101aaac7236fdc5476038af41a3f86`
- 源文件：仓库根 `rk3399-wf7000a.dts`（与上游 `arch/arm64/boot/dts/rockchip/` 同构，可随树编译）

## 重新生成载荷

```bash
# 1) 先编出 rk3399-wf7000a.dtb（见根目录 ARMBIAN-IMAGE.md 第五节）
# 2) 打包成 CI 使用的载荷
xz -9 -c rk3399-wf7000a.dtb | base64 -w 1000 > assets/rk3399-wf7000a.dtb.xz.b64
# 3) 把同一份 sha256 填进 .github/workflows/build-armbian-image.yml 的 DTB_SHA
sha256sum rk3399-wf7000a.dtb
```
