# S20P AX6000：冻结内核 ABI + 全量 kmod 构建说明

面向 `mediatek/filogic` 的 `clx_s20p`（S20P AX6000）。目的：把内核 ABI 一次性
冻结，编出**全部 kmod**，之后在设备上按需 `opkg install` 与运行内核完全匹配的
内核模块，而不必再重编内核、也不会再出现
`.gnu.linkonce.this_module section size must match the kernel's built struct module size`
这类“模块与内核不同源”的错误。

参考 release：<https://github.com/SmartRouterZone/CLX-S20P/releases/tag/20260910>

## ABI 指纹（本次构建）

| 项 | 值 |
|---|---|
| LINUX_VERSION | `6.6.151` |
| LINUX_RELEASE | `1` |
| LINUX_VERMAGIC | `cb59282b7bd81e8628c2c0d16dce9602` |
| kmods feed 路径 | `targets/mediatek/filogic/kmods/6.6.151-1-cb59282b7bd81e8628c2c0d16dce9602` |
| kernel ipk | `kernel_6.6.151~cb59282b7bd81e8628c2c0d16dce9602-r1_aarch64_cortex-a53.ipk` |

`LINUX_VERMAGIC` = 内核 `.config.set` 中所有 `=y/=m` 行排序后的 md5；
每个 kmod 的 `Depends` 都是 `kernel (=6.6.151~cb59282b…-r1)`。

## feeds 锁定

写入仓库根目录的 `feeds.conf`（该文件名被 `.gitignore` 忽略，需手动创建）：

```
src-git packages https://github.com/immortalwrt/packages.git^89c025e84343847b0daedbc9d4289794642c878d
src-git luci https://github.com/immortalwrt/luci.git^80ed8a44f244627c5a9f808d09f4c3d8f985b764
src-git routing https://github.com/openwrt/routing.git^00619bc7bc60d8b67ecc490121e45298b122cd6e
src-git telephony https://github.com/openwrt/telephony.git^92892fa285360b8981f62bf4e0a097e6449e7e33
```

## 构建

```sh
cp defconfig/mt7986-ax6000-allkmods.config .config
# 本机 curl 访问 github 系域名 TLS 失败，wget/git 正常，故改用 wget 下载源码
printf '\nCONFIG_DOWNLOAD_TOOL_CUSTOM="wget"\n' >> .config
./scripts/feeds update -a && ./scripts/feeds install -a
make defconfig
make -j8 download
CCACHE_DIR=$PWD/.ccache make -j6
```

产物：

- 固件：`bin/targets/mediatek/filogic/s20p-ax6000-<version>-mediatek-filogic-clx_s20p-squashfs-sysupgrade.bin`
  （同目录还有 `-initramfs-kernel.bin`、`.manifest`、`sha256sums`）
- 全部 ipk：`bin/targets/mediatek/filogic/packages/*.ipk`（含全部 kmod）

## 本仓库为 ALL_KMODS 所做的改动

1. **`defconfig/mt7986-ax6000-allkmods.config`**
   以 release `20260910` 的 `config.buildinfo` 为基线，加 `CONFIG_ALL_KMODS=y`。
   ALL_KMODS 让每个 kmod 的 `CONFIG_*=m` 都写进内核配置，ABI 因此固定；这些模块
   只编成 ipk，不进固件（只有显式 `=y` 的才进 rootfs）。

2. **`include/kernel-defaults.mk`**
   在内核配置合并完成后补一次 `olddefconfig`。ALL_KMODS 会引入原本没人编到的
   Kconfig 新符号（如 `IXGBE_IPSEC`），`make syncconfig` 在非 tty 环境下会
   交互式提问并直接失败。

3. **`target/linux/mediatek/filogic/config-6.6`**
   增加 `CONFIG_PTP_1588_CLOCK=y`。否则 `kmod-ptp` 把它设成 `m`，而内建的
   `mtk_eth_ptp.o` 调用 `ptp_clock_register()`，链接 vmlinux 时报
   `undefined reference`。

4. **`target/linux/mediatek/patches-6.6/999-1722-net-phy-2p5g-eee-backport-fix-missed-drivers.patch`**
   补齐 `999-1715`（`eee_broken_modes` 由 u32 改成 linkmode 位图）漏改的驱动：
   `drivers/net/phy/micrel.c`、`drivers/net/ethernet/realtek/r8169_main.c`、
   `drivers/net/usb/lan78xx.c`、
   `drivers/net/ethernet/microchip/lan743x_ethtool.c`。
   这些驱动原本不在编译范围，只有开 ALL_KMODS 才会暴露。

5. **`package/kernel/{r8101,r8125,r8126,r8127,r8152,r8168}/patches/010-ethtool-keee-for-backported-eee-api.patch`**
   Realtek 厂商驱动用 `#if LINUX_VERSION_CODE >= KERNEL_VERSION(6,9,0)` 判断
   `struct ethtool_keee` 新 API；本内核是 6.6 + backport，会误选旧 API。
   patch 把阈值改为 `6.6.0`。

## 两处仍需手工处理（不在本仓库跟踪范围内）

- **`feeds/packages/libs/libxcrypt/Makefile`**：GCC 14 + `_FORTIFY_SOURCE` 下
  libxcrypt 自带的 `-Werror` 与 fortify 的 `stdio.h` 冲突
  （`-Werror=format-nonliteral`）。加一行：

  ```
  TARGET_CFLAGS += -Wno-error=format-nonliteral
  ```

  `feeds/` 是独立的 git clone，被 `.gitignore` 忽略，改动不会随本仓库分发。

- **构建环境**：本机 `curl` 访问 github.com / raw.githubusercontent.com 等
  TLS 握手失败（`wget`、`git` 正常），所以下载工具切到 wget；另外 sandbox 下
  `ccache` 需要 `CCACHE_DIR` 指向可写目录（否则 sqlite3 的 configure 会报
  "Compiler does not work"）。

## 设备端按需安装 kmod

把 `bin/targets/mediatek/filogic/packages/` 与
`bin/targets/mediatek/filogic/kmods/6.6.151-1-cb59282b7bd81e8628c2c0d16dce9602/`
（都要带 `Packages` / `Packages.gz` 索引）发布到 HTTP，然后在设备上：

```sh
cat >> /etc/opkg/customfeeds.conf <<'EOF'
src/gz local_core http://<host>/targets/mediatek/filogic/packages
src/gz local_kmods http://<host>/targets/mediatek/filogic/kmods/6.6.151-1-cb59282b7bd81e8628c2c0d16dce9602
EOF
opkg update
opkg install kmod-fs-nfs-common
```

不需要 `--force-depends`。注意：

- **必须刷同一构建出的固件**，三者（固件 / packages / kmods）是同一 ABI 的一套。
- 不要再混装官方源（downloads.immortalwrt.org）的 kmod，其 vermagic 与本定制内核不同。
- 任何改动内核 `.config`（增删 `=y/=m` 符号、内核版本 bump、target config 变更）
  都会产生新 ABI，必须重编并重刷，旧 ipk 全部作废。
