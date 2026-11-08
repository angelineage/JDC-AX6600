# AX6600 定制固件 · GitHub Actions 云编译

> 面向 [![Build](https://img.shields.io/github/actions/workflow/status/angelineage/JDC-AX6600/QCA-ALL.yml?branch=main&label=Build)](https://github.com/angelineage/JDC-AX6600/actions/workflows/QCA-ALL.yml)**JDCloud RE-CS-02（京东云无线宝雅典娜 AX6600）** 的 ImmortalWrt 定制固件。基于 `qualcommax/ipq60xx`、Linux 6.18 LTS，启用 Qualcomm PPE 硬件卸载与 NSS Wi-Fi 加速，集成 SONiC FullCone NAT1。本仓库只负责**构建配置与自动化**，源码来自上游 ImmortalWrt 分支。

[![Stars](https://img.shields.io/github/stars/angelineage/JDC-AX6600?style=flat&logo=github&label=Stars)](https://github.com/angelineage/JDC-AX6600/stargazers)
[![Build](https://github.com/angelineage/JDC-AX6600/actions/workflows/QCA-ALL.yml/badge.svg)](https://github.com/angelineage/JDC-AX6600/actions/workflows/QCA-ALL.yml)
[![Release](https://img.shields.io/github/v/release/angelineage/JDC-AX6600?label=Latest)](https://github.com/angelineage/JDC-AX6600/releases)
[![License](https://img.shields.io/badge/build-config-MIT-blue.svg)](#免责声明)

---

## 项目现状速览

| 项 | 当前值 |
|---|---|
| 目标设备 | JDCloud RE-CS-02（京东云无线宝雅典娜 AX6600），`qualcommax/ipq60xx` / `jdcloud_re-cs-02` |
| 内核 | Linux 6.18 LTS（随上游 `owrt` 分支更新，非固定小版本） |
| 构建源码 | [ones20250/immortalwrt_ipq](https://github.com/ones20250/immortalwrt_ipq) 的 `owrt` 分支 |
| 发布版本 | **仅 PLUS 一个版本**，不再同时构建 PURE |
| 触发方式 | **仅手动**：Actions → `QCA-ALL` → Run workflow；无定时、无 push/workflow_run 自动触发 |
| 产物命名 | `IPQ60XX-WIFI-YES-PLUS-angelineage-athena-<编译日期>-<编译时间>` 统一前缀 |
| 默认管理地址 | `192.168.10.1`，主机名 `OWRT`，SSID `OWRT` |
| 加速架构 | PPE（Linux Flowtable 硬件卸载）+ NSS Wi-Fi 驱动 + firewall4 SONiC FullCone NAT1 |

### 2026-11 起的变更

- **只发 PLUS**：`QCA-ALL.yml` 的 matrix 固定为 `PROFILE: [PLUS]`。历史 Release 中仍有 PURE 版，属旧构建，不再更新。
- **不再自动编译**：`Auto-Clean` / `Cache-Clean` 的 `schedule` 已注释，`QCA-ALL` 移除了 `workflow_run`。清理旧 Release 不会连带触发编译；提交代码也不会触发编译。
- **统一产物命名**：Release tag、固件本体、`Config-*.txt`、`Packages-*.txt` 全部使用同一个 `WRT_TAG` 前缀，详见[固件命名](#固件命名与附件说明)。
- **编译期注入默认 fstab**：新增 `Files/etc/config/fstab`，用于消除启动阶段的 `block: fstab: Entry not found` 告警，详见[已知启动日志](#已知启动日志与处理)。

---

## 快速开始

### 下载

**[👉 前往 Releases 下载最新 PLUS 固件](https://github.com/angelineage/JDC-AX6600/releases)**

每个 Release 只含一个版本的产物；列表最上方即最新构建。刷机救砖所需的 U-Boot、分区表与图文流程见 [刷机救砖教程](Docs/刷机救砖教程.md)。

> ⭐ 固件对你有用的话，顺手点个 Star，是对持续维护最好的支持。

---

## 固件命名与附件说明

自 2026-11 起，所有产物统一由 `WRT_TAG` 生成：

```
WRT_TAG = <WRT_CONFIG>-<WRT_PROFILE>-angelineage-athena-<编译日期>-<编译时间>
```

- `WRT_CONFIG`：固定 `IPQ60XX-WIFI-YES`
- `WRT_PROFILE`：固定 `PLUS`
- `angelineage-athena`：仓库标识 + 板型标识（取代旧的 `ones20250-owrt` 源码名/分支名）
- `编译日期-编译时间`：`WRT_DATE`，格式 `YY.MM.DD-HH.MM.SS`（`TZ=UTC-8`，即北京时间），取**编译开始**时间，用于对应上游源码提交

示例（假设 2026-11-08 10:30:00 开始编译）：

```
IPQ60XX-WIFI-YES-PLUS-angelineage-athena-26.11.08-10.30.00
```

| 文件 | 命名形式 | 说明 |
|---|---|---|
| Release tag / 标题 | `IPQ60XX-WIFI-YES-PLUS-angelineage-athena-26.11.08-10.30.00` | 与产物前缀完全一致，便于按时间排序回溯 |
| 固件本体 | `<WRT_TAG>-ipq60xx-jdcloud_re-cs-02-squashfs-<factory\|sysupgrade>.bin` | `factory` 从原厂固件首次刷入；`sysupgrade` 在 OpenWrt/ImmortalWrt 内升级 |
| 配置快照 | `Config-<WRT_TAG>.txt` | 本次编译完整 `.config`，便于复现或二次定制 |
| 插件来源 | `Packages-<WRT_TAG>.txt` | 外部插件（PassWall / OpenClash 等）的仓库、分支与 commit |
| manifest | `<WRT_TAG>-...manifest` | 编译产出的包清单 |
| 校验文件 | `sha256sums.txt` | 刷机前请务必校验，方法见[刷机前必读](#刷机前必读) |

> 固件本体在统一前缀后仍保留平台/设备/镜像类型后缀——`factory` 与 `sysupgrade` 必须可区分，否则容易刷错。

---

## 构建机制

### 工作流

| 工作流 | 触发 | 作用 |
|---|---|---|
| `QCA-ALL.yml` | 仅 `workflow_dispatch` | **唯一发布入口**。matrix 固定单组合，调用 `WRT-CORE.yml` |
| `WRT-CORE.yml` | `workflow_call` | 编译核心：拉源码、打补丁、注入自定义文件、生成配置、编译、打包、发 Release |
| `WRT-TEST.yml` | 仅 `workflow_dispatch` | 配置验证，可选 `TEST=true` 只生成最终 `.config` 不出固件 |
| `Auto-Clean.yml` | 仅 `workflow_dispatch` | 手动清理旧 Release / workflow run（定时已关闭） |
| `Cache-Clean.yml` | 仅 `workflow_dispatch` | 手动清理 Actions 缓存 |

### 单版本构建

- `WRT_PROFILE` 默认 `PLUS`。PLUS 的额外配置来自 `Config/GENERAL_AX6600_PLUS.txt`，追加在通用配置之后按 kconfig 规则覆盖。
- LuCI 插件本体由 `Scripts/Packages.sh` 克隆到 `package/`（优先级高于 feeds）；依赖包（xray、sing-box 等）由 `passwall_packages` feed 提供，避免同名包双重定义。
- `WRT-TEST.yml` 仍保留 `PURE` 选项，仅用于**配置对比验证**，不会产出或发布 PURE 固件。
- 工具链 / ccache 缓存键由源码提交、目标平台与配置名决定，不含 profile 标识；单版本构建不会让缓存翻倍。

### 编译期补丁与自定义文件

| 内容 | 位置 | 作用 |
|---|---|---|
| QCA8081 时钟修复 | `Patches/qualcommax-qca8081-clock-fix.patch` | 绑定 PPE 端口 RX/TX 父时钟、补齐 IPQ6018 SGMII 分频表；CI 有后置断言，缺失即终止构建。详见 [Patches/README.md](Patches/README.md) |
| 默认 fstab | `Files/etc/config/fstab` | 由 `WRT-CORE.yml` 的 `Install Custom RootFS Files` 步骤复制进编译树 `files/`，随镜像写入 ROM |
| 自定义设置 | `Scripts/Settings.sh` | 主机名/IP/SSID、LuCI 版本标识、内存水位线等 |

> `Files/` 遵循 OpenWrt buildroot 的 `files/` 机制：内容原样并入 rootfs。**保留配置升级时，设备 overlay 中的 `/etc/config/fstab` 优先**，不会被仓库默认值覆盖。

### 自定义编译

修改 `Config/*.txt` 后，到 Actions 手动 Run `QCA-ALL`；`PACKAGE` 输入可临时追加插件（多个用 `\n` 分隔），`TEST=true` 时只输出配置不编译。

---

## 项目结构

| 路径 | 用途 |
|---|---|
| `.github/workflows/` | 5 个工作流，发布入口 `QCA-ALL.yml`、核心 `WRT-CORE.yml` |
| `.github/scripts/` | 构建辅助：QCA8081 补丁、宿主工具穿透、Rust 预编译、构建报告 |
| `Config/` | `.config` 片段：`GENERAL_AX6600*.txt`（通用 / PLUS 增量）、`IPQ60XX-WIFI-YES.txt`（平台设备） |
| `Files/` | 编译期注入 rootfs 的默认文件（当前为 `etc/config/fstab`） |
| `Patches/` | QCA8081 时钟修复补丁与说明 |
| `Scripts/` | 插件拉取、自定义设置、Docker 备份恢复助手 |
| `Docs/` | 保留配置升级与 Docker 备份指南、刷机救砖教程 |
| `tests/` | 配置断言、补丁白名单、升级一致性等回归测试 |

---

## 已知启动日志与处理

刷机后 `logread` 可能看到以下两类启动信息，均为无害，无需处理固件功能：

### 1. `block: unable to load configuration (fstab: Entry not found)` — 已在构建期消除

preinit 阶段 overlay 尚未挂载，`fstools/block` 会在 ROM 中查找 `/etc/config/fstab`；若 ROM 里没有这个文件，就报 `Entry not found` 与 `no usable configuration`，随后 `extroot: not configured`（正常）。它不影响 extroot、挂载或系统启动。

本仓库已通过 `Files/etc/config/fstab` 在编译期注入一份只含 `config global` 的默认配置来消除该告警。**不写死任何分区 UUID**（每台设备不同）；需要挂载数据分区时用 `block detect > /etc/config/fstab` 生成，或保留配置升级沿用已有配置。

### 2. `procd: Could not find firmware .../firmware-2.bin` — 保留，属预期

`ath11k` 驱动会先探测可选的合并固件镜像 `firmware-2.bin`（上游 Linux 为 MHI 设备引入的机制），不存在时自动回退到 `amss.bin` / `m3.bin` / `board-2.bin`。procd 记录的是"可选文件未找到"，**不是固件加载失败**。

判断依据：

- `dmesg` 中 ath11k 正常打印 `fw_version / fw_build_id`，`phy0-ap0`、`phy1-ap0` 均 UP，客户端可正常关联并跑满协商速率；
- 上游 linux-firmware 的 `IPQ6018/hw1.0`、`QCN9074/hw1.0` 目录本就不提供 `firmware-2.bin`，缺失是设计预期而非打包遗漏。

> ⚠️ **不要手工创建空的或随便下载的 `firmware-2.bin`**：驱动会真的加载它，格式/版本不匹配可能导致无线完全起不来。这条日志只能接受或在日志侧过滤，不能靠"补文件"解决。

若希望主日志干净，可在设备侧部署一个只做展示层过滤的采集脚本（全量原始日志仍保留）。参考做法：

```sh
# 仅过滤展示，不修改系统日志、不丢原始记录
logread -f | while IFS= read -r line; do
  case "$line" in
    *BEACON-*) continue ;;
    *'Could not find firmware'*'firmware-2.bin'*) continue ;;
  esac
  printf '%s\n' "$line" >> /overlay/ax6600-diag/logs/system.log
done
```

---

## 刷机指南

> [!IMPORTANT]
> - **分区红线**：PLUS 包约 160MB，升级前必须确认目标 `rootfs` 分区已扩容（>200MB）。原厂 60MB 分区**绝对不能**刷入 PLUS；不要用整机 eMMC 容量误判。
> - **Docker 数据保护**：`sysupgrade` 默认重置内部 overlay，**保留配置 ≠ 保留整个 overlay**。若 Docker 数据在内部 overlay，升级前必须完整导出；在独立分区的，升级后先核验挂载再启动 Docker。
> - **平滑升级**：可保留网络/Wi-Fi/DHCP/防火墙配置，升级后只精准清理旧版 NSS / ECM 遗留节（如 `firewall.qcanssecm`），不要误删当前 `nss-wifi` 配置。
> - 配置杂乱或重度魔改过，建议备份后 `sysupgrade -n` 不保留配置升级，再按需导入。

### 刷机前必读

1. **备份**：SSH 登录设备，备份 U-Boot 与分区表
   ```bash
   dd if=/dev/mtd0 of=/tmp/uboot.bin
   dd if=/dev/mtd1 of=/tmp/partition_table.bin
   ```
2. **校验完整性**：把 `sha256sums.txt` 与固件放同一目录
   ```bash
   sha256sum -c sha256sums.txt --ignore-missing   # Linux / macOS
   ```
   ```powershell
   certutil -hashfile 固件文件名.bin SHA256      # Windows，与 .txt 中对应行比对
   ```
3. **准备恢复手段**：TTL 串口、原厂固件备份、不死 U-Boot。

### 刷机步骤

1. 旧固件后台启用 SSH；2. 备份分区（SSH 或 TTL）；3. 刷入不死 U-Boot 与双分区 GPT 表；4. 创建 storage 分区；5. 从 Releases 下载固件（首刷 `factory`、升级 `sysupgrade`），经 U-Boot Web 界面刷入。

完整图文流程见 [刷机救砖教程](Docs/刷机救砖教程.md)；升级与 Docker 数据细节见 [保留配置升级与 Docker 备份恢复指南](Docs/AX6600保留配置升级与Docker备份恢复指南.md)。

### U-Boot 仓库

| 平台 | 仓库 | 说明 |
|---|---|---|
| eMMC | [chenxin527/uboot-ipq60xx-emmc-build](https://github.com/chenxin527/uboot-ipq60xx-emmc-build) | eMMC 设备专用 |
| NOR Flash | [chenxin527/uboot-ipq60xx-nor-build](https://github.com/chenxin527/uboot-ipq60xx-nor-build) | NOR 设备专用 |

> U-Boot 必须与硬件存储类型匹配，选错会导致无法启动。

| 风险 | 症状 | 预防 |
|---|---|---|
| U-Boot 错误 | 无法启动 | 使用不死 U-Boot，备份原件 |
| 分区错误 | 系统无法识别 | 使用正确 GPT 分区表 |
| 刷机中断电 | 变砖 | 稳定电源 / UPS |
| 固件损坏 | 开机无响应 | 先校验 SHA256 |

---

## WiFi 实际配置

雅典娜硬件为三频，但**当前构建默认启用两个射频**，第三射频（QCN9074 / HE160 / `AirPort`）在 `/etc/config/wireless` 中默认 `disabled=1`，需要时可自行启用：

| 射频 | 接口 | 信道 | 带宽 | 发射功率 | 说明 |
|---|---|---|---|---|---|
| 2.4G | `phy1-ap0` | 11 | HE20 | 20 dBm | IoT 与低速设备，20MHz 抗干扰优先 |
| 5G | `phy0-ap0` | 149 | HE80 | 22 dBm | 非 DFS，US 下功率上限较高 |
| 第三射频 | `radio2` | 36 | HE160 | 24 dBm | 默认禁用，`ssid=AirPort` |

通用项：地区 `US`、加密 `sae-mixed`（WPA3/WPA2 混合，CCMP）、2.4G/5G 同名 SSID `Air`。

> 实测提示：5GHz HE80 单客户端 TCP 吞吐约 400–500 Mbps 属正常区间，主要受信号强度与 MCS 波动影响；`radio0` 硬件不支持 160MHz（`neither 160 nor 80+80`），不要按 160MHz 预期调参。

---

## PLUS 版使用提示

- **AdGuard Home 首次启用**：默认关闭。启用后访问 `http://路由器IP:3000` 走安装向导，DNS 监听端口填 `5553`（`53` 已被 dnsmasq 占用）；完成后在 LuCI「网络 → DHCP/DNS」的「DNS 转发」填 `127.0.0.1#5553` 并勾选「忽略解析文件」。
- **代理插件二选一**：OpenClash 与 PassWall2 都接管 DNS + 透明代理，同时启用会互相抢占，轻则分流错乱、重则断网。只启用一个，切换前先停用当前插件。
- **Docker 存储去向**：默认写入系统分区容易撑爆。建议先用「分区扩容（partexp）」把空闲 eMMC 挂为独立分区，再把 Docker 数据根指向该分区。
- **依赖外部插件仓库**：上游调整包名或依赖时，可能需要同步更新 `Config/GENERAL_AX6600_PLUS.txt`。

---

## PPE + NSS Wi-Fi 加速架构

PPE 与 NSS 同时启用、各管一段，启用 PPE 不等于移除全部 NSS 组件。

| 路径 | 组件 | 作用 |
|---|---|---|
| PPE 硬件卸载 | `qca_ppe`、Linux Flowtable | 加速受支持的有线转发、路由 / NAT |
| NSS Wi-Fi 加速 | `nss-wifi-drv`、ath11k、NSS 固件 | 加速受支持的 Wi-Fi 数据路径 |

不再使用旧版 `kmod-qca-nss-*` / ECM 组件，但**仍依赖闭源 NSS Wi-Fi 固件**，不能描述为"完全移除闭源微码"。实际卸载范围取决于上游版本、无线设备、协议与网络配置，不能由开关已启用推断所有流量均已卸载。

### 检查运行状态

```sh
uci show firewall | grep flow_offloading
cat /sys/kernel/debug/3a000000.ppe/active_flows
cat /sys/kernel/debug/3a000000.ppe/offload

uci show nss-wifi
cat /sys/module/ath11k/parameters/nss_offload
cat /sys/kernel/debug/39000000.nss/boot
cat /sys/kernel/debug/39000000.nss/flows
```

`nss_offload=Y` 且 NSS `boot=1` 表示加速已启用且核心已启动；还需结合已安装流与持续增长的收发计数判断实际转发。PPE 的 `ingress_not_switch_port` 等拒绝计数只说明该次请求不满足卸载条件，**不等同于丢包**，也不能单独用来判断 NSS 加速失败。

---

## 固件特性对比

| 特性 | 原厂固件 | 本项目固件 |
|---|---|---|
| OpenWrt / 内核 | 不支持 | ✅ ImmortalWrt / Linux 6.18 LTS / firewall4 |
| 硬件加速 | 原厂闭源 NSS 架构 | ✅ PPE + Flowtable 硬件卸载，NSS Wi-Fi 加速受支持的无线路径 |
| NAT 模式 | 普通锥形 NAT | ✅ SONiC FullCone NAT1（集成于 firewall4） |
| 自定义插件 | ❌ | ✅ Docker / PassWall2 / OpenClash 开箱即用 |
| 构建方式 | — | ✅ GitHub Actions **手动触发**云编译，宿主工具穿透与缓存优化 |
| 上游跟进 | — | ✅ 每次编译时拉取 `owrt` 分支最新提交（非每日自动构建） |
| 开源透明 | ❌ 闭源 | ✅ 构建脚本与主机侧驱动可审查；NSS Wi-Fi 仍依赖闭源固件 |

---

## 免责声明

刷机有风险，操作需谨慎。本项目固件仅供学习与研究使用，请确认设备型号匹配并提前备份数据。因刷机造成的设备损坏或数据丢失，作者不承担任何责任。
