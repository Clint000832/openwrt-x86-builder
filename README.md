# OpenWrt x86_64 从零开始编译教程

本仓库记录了从零开始编译 OpenWrt x86_64 固件的完整流程，包含自定义配置和常用软件包选择，适合在 **PVE / ESXi / Proxmox / 物理机** 等 x86 平台上运行。

## 特性

- **目标平台**: x86_64 (generic)
- **内核版本**: Linux 6.6.139
- **根文件系统**: ext4 (GRUB EFI 引导)
- **开箱即用**: 预配 LAN IP 为 `192.168.110.1`
- **软件包数量**: ~246 个，涵盖基础路由、代理、防火墙等功能

### 主要集成软件包

| 类别 | 软件包 |
|------|--------|
| Web 管理 | luci-light, luci-ssl, luci-mod-admin-full |
| 代理 | openclash, nikki |
| 防火墙 | luci-app-firewall, luci-app-pbr |
| **FanchmWrt** | fwxd, kmod-fwx, luci-theme-fanchmwrt |
| 广告过滤 | luci-app-adguardhome |
| 网络工具 | curl, wget-ssl, tcpdump, iperf3 |
| 系统工具 | htop, bash, nano, luci-app-package-manager |
| 存储 | ext4, btrfs (内核支持) |

## 前置要求

- **操作系统**: Ubuntu 22.04 / Debian 12 / macOS (推荐 Linux)
- **磁盘空间**: 至少 40GB 可用空间
- **内存**: 至少 4GB (编译过程内存敏感)
- **网络**: 稳定的互联网连接 (需要下载大量源码包)

### 安装依赖

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  g++-multilib gettext git libncurses-dev libssl-dev python3 python3-distutils \
  python3-setuptools rsync swig unzip zlib1g-dev file wget

# macOS (Homebrew)
brew install coreutils diffutils findutils gawk gnu-getopt gnu-tar grep \
  make ncurses pkg-config python3 unzip
```

## 编译步骤

### 1. 获取 OpenWrt 源码

```bash
git clone https://git.openwrt.org/openwrt/openwrt.git --branch openwrt-24.10
cd openwrt
```

### 2. 配置 feeds

```bash
# 使用本仓库提供的 feeds.conf.default
cp feeds.conf.default feeds.conf

# 更新 feeds
./scripts/feeds update -a
./scripts/feeds install -a
```

**说明**: `feeds.conf.default` 已添加第三方 feed `kenzok8/openwrt-packages` 和 `fanchmwrt/fanchmwrt-packages`，分别包含 openclash、nikki 等常用包以及 FanchmWrt 依赖。

### 3. 配置编译选项

```bash
# 使用本仓库提供的 .config（推荐）
cp .config .config

# 或者从头开始配置
make menuconfig
```

**`.config` 关键配置项** (本仓库):

- Target System: `x86`
- Subtarget: `x86_64`
- Target Profile: `Generic`
- Root filesystem: `ext4` (512MB)
- Bootloader: `GRUB` + `GRUB EFI`
- Images: gzip 压缩

### 4. 添加自定义文件（可选）

本仓库包含 `files/` 目录，编译时会自动打包进固件：

```bash
# 目录结构
files/
└── etc/
    └── uci-defaults/
        └── 99-lan-ip    # 首次启动设置 LAN IP 为 192.168.110.1
```

使用方式：将自定义文件放在 `openwrt/files/` 下，保持与目标系统的目录结构一致即可。

### 5. 开始编译

```bash
# 下载所需的源代码包
make download -j$(nproc)

# 检查下载完整性
find dl -size -1024c -exec ls -l {} \;

# 编译固件（-j 后跟 CPU 核心数，推荐使用 1-2 倍核心数）
make -j$(nproc) || make -j1 V=s
```

> **提示**: 首次编译耗时较长，通常需要 30 分钟到 2 小时，取决于网络和 CPU 性能。
> 如果编译出错，使用 `make -j1 V=s` 查看详细日志定位问题。

### 6. 获取固件

编译完成后，固件位于 `bin/targets/x86/64/`：

```
openwrt-x86-64-generic-squashfs.img.gz    # squashfs 固件
openwrt-x86-64-generic-ext4.img.gz        # ext4 固件
openwrt-x86-64-generic-kernel.bin         # 内核
openwrt-x86-64-generic-rootfs.tar.gz      # 根文件系统
```

推荐使用 `ext4` 版本，方便后期扩容和数据维护。

## 安装

### PVE / Proxmox VE

1. 上传 `openwrt-x86-64-generic-ext4.img.gz` 到 PVE 节点
2. 解压并导入：
```bash
gunzip openwrt-x86-64-generic-ext4.img.gz
qm importdisk <VM_ID> openwrt-x86-64-generic-ext4.img <STORAGE_ID>
```
3. 创建虚拟机：取消勾选 CD-ROM，引导顺序设置为新导入的磁盘

### ESXi

1. 解压 `.img.gz` 获取 `.img` 文件
2. 使用 StarWind V2V Converter 转换为 VMDK
3. 创建虚拟机，选择转换后的 VMDK 作为磁盘

### 物理机 / 其他平台

使用 `dd` 或 Rufus / balenaEtcher 将 `.img.gz` 写入 U 盘或硬盘：

```bash
zcat openwrt-x86-64-generic-ext4.img.gz | dd of=/dev/sdX bs=1M
```

## 首次启动

1. 固件默认 LAN IP 为 **`192.168.110.1`** (在 `files/etc/uci-defaults/99-lan-ip` 中配置)
2. 将电脑网口连接到设备 LAN 口，设置静态 IP 为 `192.168.110.x`
3. 浏览器访问 `http://192.168.110.1`，进入 Luci 管理界面
4. 默认无密码，首次登录后请立即设置 root 密码

## 自定义配置说明

### 修改 LAN IP

编辑 `files/etc/uci-defaults/99-lan-ip`，修改目标 IP 地址。

### 添加更多软件包

```bash
make menuconfig
# 选中所需软件包（按 Y 键）
# 保存退出后重新编译
```

软件包搜索技巧：在 menuconfig 中按 `/` 键输入关键词搜索，记下路径后按数字键直接跳转。

### 保持配置跨版本更新

```bash
# 备份当前配置
cp .config .config.bak

# 更新源码
git pull
./scripts/feeds update -a
./scripts/feeds install -a

# 在原配置基础上增量更新
make oldconfig
```

## 目录结构

```
/
├── .config              # 编译配置（本仓库提供参考）
├── feeds.conf.default   # feeds 配置
├── files/               # 自定义配置文件
│   └── etc/uci-defaults/99-lan-ip
├── openwrt/             # OpenWrt 完整构建目录（本地）
│   └── package/fcm/     # FanchmWrt 集成组件
│       ├── fwx/         #   内核 DPI 模块
│       ├── fwxd/        #   用户态守护进程
│       ├── libfwx_common/   # 共享库
│       └── luci-theme-fanchmwrt/  # LuCI 主题
└── README.md            # 本文件
```

> `openwrt/` 完整构建目录在本地，未纳入版本控制。使用本教程请自行 `git clone` 源码。

## 参考链接

- [OpenWrt 官方文档](https://openwrt.org/docs/start)
- [OpenWrt 24.10 发布说明](https://openwrt.org/releases/24.10/notes)
- [kenzok8/openwrt-packages](https://github.com/kenzok8/openwrt-packages)
- [OpenClash](https://github.com/vernesong/OpenClash)
- [nikki](https://github.com/nikkinikki-org/OpenWrt-nikki)
- [FanchmWrt](https://github.com/fanchmwrt/fanchmwrt) — 集成 FanchmWrt 防火墙/DPI 功能

## FanchmWrt 集成说明

本仓库集成了 [FanchmWrt](https://github.com/fanchmwrt/fanchmwrt) 的核心组件，提供企业级防火墙和应用识别（DPI）能力：

### 集成组件

| 组件 | 说明 |
|------|------|
| `kmod-fwx` | 内核 Netfilter 模块，深度包检测（DPI） |
| `fwxd` | 用户态守护进程，应用识别、MAC 过滤、流量记录 |
| `libfwx_common` | 共享库，提供 UCI 配置接口 |
| `luci-theme-fanchmwrt` | LuCI 主题，支持明暗两套图标 |

### 启用方式

`package/fcm/` 目录下的组件已直接放入本地构建树，无需额外 feed。编译时通过 `.config` 的以下选项控制：

```bash
CONFIG_PACKAGE_kmod-fwx=y          # 内核 DPI 模块
CONFIG_PACKAGE_fwxd=y              # 用户态守护进程
CONFIG_PACKAGE_libfwx_common=y      # 共享库
CONFIG_PACKAGE_luci-theme-fanchmwrt=y  # LuCI 主题
```

### 依赖

固件内置的 FanchmWrt 依赖（已在 `.config` 中启用）：
- libsqlite3、libmosquitto-ssl、libcurl、libubus、libubox、libuci、libjson-c、libblobmsg-json、libuci-lua、lm-sensors

### 运行时配置

`fwxd` 守护进程提供以下功能：
- **应用过滤**（DPI）：通过 `feature.cfg` 特征库识别数千款应用（社交、视频、游戏、购物等）
- **MAC 过滤**：黑/白名单策略
- **流量记录**：记录网络连接详情
- **Ubus 接口**：供 LuCI 和 RPCD 调用

配置文件位于 `/etc/config/` 下（`fwx`、`appfilter`、`macfilter` 等），首次启动时的 UCI 默认值在 `files/etc/uci-defaults/100_fwx` 中设置。

### 调整或关闭

如果不需要 FanchmWrt 组件，可以：

```bash
make menuconfig
# 取消勾选以下项目：
#   Kernel modules > Netfilter Extensions > kmod-fwx
#   Base system > fwxd
#   Base system > libfwx_common
#   LuCI > Themes > luci-theme-fanchmwrt
# 保存退出
make -j$(nproc)
```
