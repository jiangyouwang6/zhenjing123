# Redmi AX3000T OpenWrt 24.10 固件（MTK 闭源驱动 + 功率解锁）

基于 [immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase) 的
`rework-24.10` 分支（OpenWrt 24.10 / 内核 6.6）定制，为红米 AX3000T（`xiaomi_mi-router-ax3000t`）编译。

## 固件特性

| 项目 | 说明 |
|---|---|
| 内核 | 6.6（OpenWrt 24.10） |
| 无线驱动 | **MTK 闭源驱动 `kmod-mt_wifi` 7.6.7.2**（原厂驱动，信号/稳定性优于开源 mt76） |
| 无线配置 | `mtwifi-cfg`（OpenWrt 原生 LuCI 界面，支持 802.11k/v/r、弱信号剔除等） |
| 硬件加速 | `turboacc-mtk`（PPE/HNAT 硬件 NAT 加速，双 PPE 各 32K entry） |
| 硬件 QoS | `luci-app-eqos-mtk` |
| 功率解锁 | 固件内置 `/usr/bin/txpwr.sh`（mt7981_factory_txpwr_patch），一键解锁 EEPROM 功率上限 |
| 界面 | LuCI + 中文语言包 + argon 主题 |
| 已剔除 | passwall / ssr-plus 等科学上网插件（如需可自行在 config 中启用） |

## 目录结构

```
.
├── .github/workflows/build-openwrt.yml   # GitHub Actions 编译工作流
├── config/ax3000t.config                 # 定制后的编译配置
├── files/usr/bin/txpwr.sh                # 功率解锁工具（编译进固件 /usr/bin/txpwr.sh）
└── README.md
```

## 使用方法（云编译）

1. 在 GitHub 上新建一个仓库（Public 或 Private 均可），例如 `ax3000t-openwrt`
2. 把本目录下**所有文件**推送到该仓库的 `main` 分支（`.github`、`config`、`files`、`README.md`）
3. 推送后自动触发编译；也可以在仓库页面 **Actions → Build OpenWrt for Redmi AX3000T → Run workflow** 手动触发
4. 编译约 1.5~3 小时，完成后在 Actions 运行页面的 **Artifacts** 下载
   `redmi-ax3000t-openwrt-24.10`（zip 包）

> 首次编译需下载完整工具链，耗时较长属正常现象。二次编译会复用 `dl/` 下载缓存。

## 编译产物说明

zip 包解压后为 `bin/targets/mediatek/filogic/` 目录，其中与本设备相关的是：

| 文件 | 用途 |
|---|---|
| `xiaomi_mi-router-ax3000t-initramfs-factory.ubi` | **原厂 uboot 刷入用**（第一次刷机） |
| `xiaomi_mi-router-ax3000t-squashfs-sysupgrade.bin` | **stock 布局正式固件**（initramfs 内升级用） |
| `xiaomi_mi-router-ax3000t-ubootmod-initramfs-recovery.itb` | OpenWrt U-Boot 布局的 recovery 镜像 |
| `xiaomi_mi-router-ax3000t-ubootmod-squashfs-sysupgrade.itb` | OpenWrt U-Boot 布局正式固件 |
| `xiaomi_mi-router-ax3000t-ubootmod-preloader.bin` / `bl31-uboot.fip` | OpenWrt U-Boot（ubootmod 专用） |
| `sha256sums` | 校验文件 |

> 如果同时编译了 `-mtkuboot` 变体，多出的 `-mtkuboot-*` 文件为 MTK 原厂 U-Boot 布局，
> 供刷了 MTK U-Boot 的设备使用，一般用户可忽略。

## 刷机指南（stock 布局，推荐首次刷机）

> 以下为 OpenWrt 官方方法，中文版（RD03）与海外版（RD23）均适用。
> 刷机有变砖风险，请仔细阅读每一步。

### 1. 在原厂固件上开启 SSH

国行（RD03）原厂固件存在 API 漏洞可开启 SSH，对应版本：

| 原厂固件版本 | 利用接口 |
|---|---|
| 1.0.47 (CN) | `misystem/arn_switch` |
| 1.0.64 (CN) | `xqsystem/start_binding` |
| 1.0.84 (CN) | `xqsystem/start_binding` |

具体操作（以 1.0.84 为例，浏览器访问路由器 IP 后打开）：

```
http://192.168.31.1/cgi-bin/luci/;stok=<登录后stok>/api/xqsystem/start_binding
```

返回 `{"code":0}` 后，用以下命令开启 telnet/ssh：

```
telnet 192.168.31.1   # 用户名 root，密码为空（部分版本需要先执行 start_binding 再 telnet）
```

若 telnet 不通，可尝试先降级到 1.0.47 / 1.0.64 再操作（小米官网可下载历史固件，
或使用官方"小米路由器修复工具"）。

### 2. 刷入 initramfs

把 `xiaomi_mi-router-ax3000t-initramfs-factory.ubi` 上传到路由器 `/tmp`：

```
scp xiaomi_mi-router-ax3000t-initramfs-factory.ubi root@192.168.31.1:/tmp/
```

SSH 登录后执行（**先确认当前系统槽位**）：

```
cat /proc/cmdline    # 看 firmware=0 还是 firmware=1
```

- 若 `firmware=0`：
  ```
  ubiformat /dev/mtd9 -y -f /tmp/xiaomi_mi-router-ax3000t-initramfs-factory.ubi
  ```
- 若 `firmware=1`：
  ```
  ubiformat /dev/mtd8 -y -f /tmp/xiaomi_mi-router-ax3000t-initramfs-factory.ubi
  ```

然后设置启动标志并重启：

```
nvram set boot_wait=on
nvram set uart_en=1
nvram set flag_boot_rootfs=1
nvram set flag_last_success=1
nvram set flag_boot_success=1
nvram set flag_try_sys1_failed=0
nvram set flag_try_sys2_failed=0
nvram commit
reboot
```

重启后路由器即为 OpenWrt（initramfs 临时系统），默认地址 `192.168.1.1`。

### 3. 升级为正式固件

在 OpenWrt 临时系统里执行（或 LuCI → 系统 → 备份/升级）：

```
sysupgrade -n /tmp/xiaomi_mi-router-ax3000t-squashfs-sysupgrade.bin
```

升级完成后即为正式固件，默认地址 `192.168.1.1`，用户名 `root`，无密码。

## 功率解锁（提升无线发射功率）

固件内置 `txpwr.sh`（修改 Factory 分区 EEPROM 中的发射功率字段，不影响 MAC/校准数据）。
内置了针对 AX3000T 的预设：2.4G 28~29dBm、5G 28dBm。

SSH 登录后：

```
# 查看当前各频段功率字段
txpwr.sh

# 应用 AX3000T 高功率预设（2g/5g/6g 全部）
txpwr.sh -p ax3000t -b all

# 只解锁 5G
txpwr.sh -p ax3000t -b 5g
```

脚本会生成 `/tmp/factory_dump.bin`（修改后的副本）并显示修改前后对比，
**不会自动写回 MTD**，确认无误后按脚本提示手动写回（见脚本输出）。

> ⚠️ 警告：
> - 修改功率会增大发热与功放压力，可能缩短寿命或导致不稳定，请自行评估风险
> - 中国法规限制 5G 室内 EIRP ≤ 23dBm，解锁后仅建议在允许的环境中测试使用
> - 操作前务必备份原 Factory 分区：`dd if=/dev/mtdX of=/tmp/factory_backup.bin`
> - 若出现异常，写回备份即可恢复

## 注意事项

- 不要在开启 `turboacc-mtk`（hwnat）时同时启用主线的 flow-offload 加速
- 无线加密推荐 WPA2-PSK/WPA3-PSK 以获得最佳兼容性
- 2.4G 默认已关闭 MU-MIMO（兼容老旧智能家居设备）
- 中继（APCLI）扫描期间，对应频段设备可能短暂掉线，属正常现象
- 官方 24.10 起已修复"重启 6 次后配置重置"的原厂 uboot 逻辑问题

## 版权与致谢

- [immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase)（chasey-dev）
- [immortalwrt-mt798x](https://github.com/hanwckf/immortalwrt-mt798x)（hanwckf）
- [mt7981_factory_txpwr_patch](https://github.com/4n0n4/mt7981_factory_txpwr_patch)（4n0n4）
