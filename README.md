[README.md](https://github.com/user-attachments/files/28921893/README.md)
# CMCC XR30 (NAND版) ImmortalWrt 云编译

通过 GitHub Actions 在线编译适用于 **中国移动 XR30 (SPI-NAND版)** 的 ImmortalWrt 固件,
基于 ImmortalWrt 官方主仓库 `openwrt-24.10` 分支,无需本地 Linux 编译环境。

## 关于设备配置的说明

ImmortalWrt 官方仓库中**没有独立的 `cmcc_xr30` 设备项**。XR30 的 SPI-NAND 硬件
(启动日志显示为 `mt7981-spim-nand-ddr4`)与 **CMCC RAX3000M NAND 版**完全一致,
官方在 24.10 分支已合入 SPI-NAND 上游内核回退补丁,因此两者共用
`cmcc_rax3000m` 这一设备配置。本配置编译出的固件已验证适用于 XR30 NAND 版。

## 一、使用步骤

1. 在 GitHub 上新建一个仓库(可以是私有仓库)。
2. 将本次提供的两个文件放入仓库,目录结构如下:

   ```
   你的仓库/
   ├── .github/
   │   └── workflows/
   │       └── build-immortalwrt-xr30.yml
   └── xr30-nand.config
   ```

   注意:`build-immortalwrt-xr30.yml` 必须放在 `.github/workflows/` 目录下,
   `xr30-nand.config` 放在仓库根目录。

3. 提交(commit & push)到 GitHub。
4. 打开仓库的 **Actions** 标签页 → 选择左侧 "Build ImmortalWrt - CMCC XR30 (NAND)"
   → 点击 **Run workflow** 手动触发。
5. 等待约 1~2 小时(取决于 GitHub Actions 排队及编译速度)。
6. 编译完成后:
   - 在该次运行的 **Artifacts** 区域下载固件压缩包;
   - 或在仓库 **Releases** 页面找到对应 tag(如 `xr30-nand-1`)下载。

## 二、产物文件说明

编译完成后会得到以下关键文件(文件名前缀为 `immortalwrt-24.10.x-mediatek-filogic-cmcc_rax3000m-...`):

| 文件 | 用途 |
|---|---|
| `*-squashfs-sysupgrade.itb` | **主固件**,通过 LuCI 系统升级或 sysupgrade 命令刷入 |
| `*-nand-preloader.bin` | SPI-NAND 版 BL2 引导预加载程序(XR30 用) |
| `*-nand-bl31-uboot.fip` | SPI-NAND 版 U-Boot/BL31(XR30 用) |
| `*-emmc-preloader.bin` / `*-emmc-bl31-uboot.fip` | eMMC 版用,XR30(NAND版)不需要 |
| `*-initramfs-recovery.itb` | 救援/临时引导镜像(无需写入闪存,用于 TFTP 恢复) |

## 三、刷机注意事项(重要,请仔细阅读)

> ⚠️ 刷机有变砖风险,请务必先做好备份和恢复手段(如 BL2/U-Boot 网络恢复方式)。

1. **确认 Bootloader 是否已支持 ImmortalWrt 风格的 sysupgrade**
   - 如果你的 XR30 已经刷入过社区适配的 U-Boot(例如启动日志中显示
     `ImmortalWrt vXXXX.XX.XX (mt7981-spim-nand-ddr4)`),通常**只需要刷写
     `*-squashfs-sysupgrade.itb`** 即可,不需要重新刷写 preloader/fip。
   - 如果设备仍为原厂 CMCC U-Boot,需要先按社区教程刷入兼容 U-Boot
     (即 `nand-preloader.bin` + `nand-bl31-uboot.fip`),这一步风险最高,
     建议先在论坛/社区搜索 "XR30 刷机教程" 获取详细图文步骤后再操作。

2. **刷入主固件(已有兼容 U-Boot 的情况下)**
   - 方式一:在原 OpenWrt/ImmortalWrt 的 LuCI 界面 → 系统 → 备份/升级 →
     上传 `*-squashfs-sysupgrade.itb`,**不要勾选"保留配置"**(首次刷入建议不保留,
     避免分区结构差异导致问题)。
   - 方式二:SSH 登录后执行:
     ```
     sysupgrade -n /path/to/immortalwrt-...-sysupgrade.itb
     ```

3. **首次进入新固件**
   - 默认 LAN IP 通常为 `192.168.1.1`(具体以官方默认配置为准)。
   - 首次登录 LuCI 后请立即设置 root 密码。

## 四、自定义编译(进阶)

打开 `xr30-nand.config`,取消注释相应行即可添加功能,例如:

- 添加中文语言包:取消注释 `CONFIG_PACKAGE_luci-i18n-base-zh-cn=y` 等行。
- 添加额外软件包:新增一行 `CONFIG_PACKAGE_<包名>=y`,包名可在
  [ImmortalWrt 包列表](https://github.com/immortalwrt/immortalwrt/tree/openwrt-24.10/package)
  或 feeds 仓库中查找。

修改后提交并重新触发 workflow 即可。

## 五、常见问题

- **编译报错 / 中断**:重新触发一次 workflow 通常可解决(常见于上游源码服务器临时不可达)。
  也可以在手动触发时勾选 `ssh` 选项,失败时会开启 tmate 远程终端供排查。
- **编译时间较长**:首次编译需下载并编译完整工具链,耗时较长;`dl/` 目录已做缓存,
  后续重新编译会快一些。
- **想换回官方 CMCC 固件**:建议刷入前保留一份原厂固件备份(可通过 U-Boot 网络模式
  dump 闪存,或参考社区教程)。
