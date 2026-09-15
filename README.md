# ImmortalWrt 24.10.6 for H3C Magic NX30 Pro (NMBM)

用 GitHub Actions 自编译 H3C Magic NX30 Pro 的 ImmortalWrt 固件，内置 PassWall，面向"科学上网 + 内网约 15 台设备"的场景。

> **已验证可用。** 最近一次成功构建（run `34866009485`）零错误零告警通过：三件套镜像 28 / 35 / 33 MB、内核模块 89 个、版本后缀 `-nx30pro` 均按预期产出，并自动发布为 prerelease `build-5`。

## 仓库结构

```
.
├── .github/
│   └── workflows/
│       └── build-nx30pro.yml      # 构建流程
└── nx30pro-nmbm.config            # 编译配置
```

仓库本身只有这两个文件，源码在每次构建时由 CI 现场从上游 clone，所以不存在"fork 落后上游需要同步"的问题。

## 怎么用

1. **把仓库设为 Public。**
   Actions 对公开仓库免费且不计量；私有仓库每月只有 2000 分钟，而单次编译约 2–3.5 小时。

2. **把两个文件放到位。**
   `nx30pro-nmbm.config` 放仓库根目录，`build-nx30pro.yml` 放 `.github/workflows/` 下。

3. **触发构建。**
   到 Actions 页面点 *Run workflow*；或者在改动 `nx30pro-nmbm.config` 后 push，会自动触发。

4. **取产物。**
   Actions 运行页的 Artifacts 区可以下载；同时会自动发一个 prerelease，后者不需要登录 GitHub 就能下载。

## 关键参数

| 项目 | 值 | 说明 |
|---|---|---|
| 源码 | `immortalwrt/immortalwrt` tag `v24.10.6` | 用 tag 而非分支：该 tag 的 `feeds.conf.default` 把四个 feed 锁到具体提交，构建可复现 |
| 设备变体 | `h3c_magic-nx30-pro-nmbm` | 结尾 `-nmbm` 不能省，另一个变体是 ubootmod 布局，需配套另一套 U-Boot |
| 内核 | 6.6.133 | |
| 代理 | PassWall 25.12.16（Xray + **Hysteria2** + SS-Rust 客户端 + geo 数据） | Hysteria2 由 `INCLUDE_Hysteria` 提供；该包名为 `hysteria`，但装的是 2.6.5（即 Hysteria2），passwall 里节点类型显示为 `hysteria2` |
| 组网 | **ZeroTier 1.14.1**（`zerotier` + `luci-app-zerotier` + 中文包） | 边际体积 3.47 MB（新增 7 个包，未压缩口径）；复用了已编入的 `kmod-tun`。界面在「VPN → ZeroTier」 |
| 产物 | `*-nmbm-squashfs-factory.bin`（刷机用）、`*-nmbm-squashfs-sysupgrade.bin`（后续升级用） | |

## 刷机

进 U-Boot 的 failsafe 界面刷 `*-nmbm-squashfs-factory.bin`：

1. 网线接 LAN 口，电脑固定 IP `192.168.1.100` / 掩码 `255.255.255.0`（U-Boot 不提供 DHCP）
2. 断电，按住 RESET 通电，持续 15 秒以上
3. 浏览器隐身窗口访问 `http://192.168.1.1`，上传 `factory.bin`

**不要刷 `-bl31-uboot.fip` 和 `-preloader.bin`。** 那是引导程序本身，写坏只能拆机接 TTL。

刷前请先备份 Factory 分区（存放 MAC 与无线校准参数）。NX30 Pro 上的分区号已按实机确认，**Factory 是 `mtd3`**（`mtd0` 是整片 NAND、`mtd1` 是 BL2、`mtd2` 是 u-boot-env）：

```sh
nanddump -f /tmp/factory.bin /dev/mtd3   # 或 dd if=/dev/mtd3 of=/tmp/factory.bin
ls -l /tmp/factory.bin                   # 必须是 2097152 字节（2 MiB）
```

详见随附的《刷机与首次配置手册》。

## 两个必须知道的机制

**内核模块无法事后安装。** 内核的 vermagic 是三段拼出来的：`<内核版本>-<LINUX_RELEASE>-<配置哈希>`。配置哈希的算法是 `grep '=[ym]' .config | LC_ALL=C sort | md5`（`include/kernel-defaults.mk` 第 130 行，写进 `.vermagic`），再经 `include/kernel.mk` 第 49 行读入 `LINUX_VERMAGIC`、由 `include/feeds.mk` 第 41 行拼进 kmods 源路径。官方 24.10.6 是 `6.6.133-1-a8b93917f464536104594f27d870028d`；本配置实编固件实测 `6.6.133-1-205b2f77dbd69879db70a4bdaa1c49fc`——机制一样，**只差第三段**，因为多选了 `kmod-tun` 等模块。第三段不同，官方 kmods 源里的包在依赖检查阶段就会被拒绝。凡是用得到的模块必须在编译时就选进去。当前配置额外包含了 `kmod-tun`、`kmod-inet-diag`、`kmod-netlink-diag`、`kmod-wireguard`、`kmod-sched-cake` 作为余量。

**用户态包可以事后安装。** LuCI 插件、`xray-core`、`sing-box` 这类包不依赖内核校验，`opkg` 随时可装。

## 工作流里最关键的一步

`make defconfig` 之后的**配置自检**。kconfig 对拼错的符号和不存在的包名一律静默丢弃、不报错——没有这步校验，CI 会全绿通过，你刷完才发现固件里没有 PassWall。自检覆盖设备符号、PassWall 及其子选项（开/关双向断言）、ZeroTier 组网、内核模块、UPnP 后端、构建开关、opkg 源地址、feeds 就位情况八类，任一不符直接 fail 掉构建。此外产物阶段还会做**实证核对**：直接去 `bin/packages` 里找 `xray-core`、`hysteria`、`shadowsocks-rust-sslocal`、`zerotier`、`luci-app-zerotier` 的 ipk，并验证 `luci-app-passwall` 包内确实含 `util_hysteria2.lua`、`zerotier` 包内确实含 `zerotier-one` / `zerotier-fw4` / `etc/init.d/zerotier`——这防的是"依赖解析没把包拉进来"这类 `.config` 断言看不见的问题。

## ZeroTier 怎么用

刷完之后 ZeroTier 默认是**关闭**的（`option enabled '0'`），需要手动开：

1. 打开 LuCI → **VPN → ZeroTier → Configuration**，勾选 Enable（或在 ttyd 里执行下面两条命令）。
2. 在界面上加一条 Network，填入你的 16 位网络 ID；也可以直接 `zerotier-cli join <网络ID>`。
3. Interface info 页可以看到已加入的网络、分配的 IP 与 Peers 状态。

```sh
uci set zerotier.global.enabled='1'
uci set zerotier.global.copy_config_path='1'   # 见下方说明
uci add zerotier network
uci set zerotier.@network[-1].id='<16位网络ID>'
uci set zerotier.@network[-1].enabled='1'
uci commit zerotier
/etc/init.d/zerotier restart
```

三个容易踩的点：

- **`allow_default` 保持 `0`。** 这一项是"允许 ZeroTier 接管默认路由"。一旦置 1，对外流量可能整体改走 ZeroTier，与 passwall 的透明代理叠加后行为难以预测。
- **`copy_config_path` 建议置 `1`。** 这一项把 ZeroTier 的运行目录复制到内存而不是直接写闪存，可以明显减少 NAND 写入——ZeroTier 会频繁更新对等节点状态，长期直接写闪存不划算。
- **要对外连通，UDP 9993 得能进。** 它自带的 `/usr/bin/zerotier-fw4` 会往 `inet fw4` 表里注入规则（走的是 nftables，不需要 iptables），UCI 里的 `fw_allow_input` 控制的就是这个。默认已开。

**它和 passwall 不冲突。** passwall 的脚本里没有任何 ZeroTier 相关处理，透明代理只对你在 passwall 里选定的接口生效（通常是 `br-lan`），而 ZeroTier 的接口名是 `ztXXXXXXXX`，默认不在其中，所以 ZeroTier 流量不会被代理。反过来说，如果你希望 ZeroTier 的流量也走代理，需要主动把该接口加进 passwall——但那通常不是你想要的效果。

**体积账：边际 3.47 MB。** 这是用同一条闭包算法跑两遍再取差集算出来的——不含 ZeroTier 时 164 个包 / 101.7 MB，含时 171 个包 / 105.2 MB。新增的只有 7 个：`libstdcpp6`（2,060 KB，ZeroTier 是 C++ 写的）、`zerotier`（1,250 KB）、`libminiupnpc`、`libnatpmp1`、`libatomic1`、`luci-app-zerotier`、`luci-i18n-zerotier-zh-cn`。**`kmod-tun` 没有新增开销**——它在内核模块区已经预留了。有个反直觉的点：`miniupnpd-nftables` 并不依赖 `libminiupnpc` / `libnatpmp1`，别以为 UPnP 那里已经带了。

（口径说明：上句的 164 → 171 个包、101.7 → 105.2 MB 是**显式包闭包**口径——算边际必须这么算，两边同一条算法各跑一遍再相减。它不代表镜像里的全部包：target 的 `Default-Packages` 还会**无条件**带入 93 个包 / 12.25 MB（`dnsmasq-full`、`firewall4`、`wpad-openssl`、`luci-light`、`ubi-utils`、`nftables-json`、`procd-ujail`、`default-settings-chn` 等），并集才是真机装机集，约 264 个包 / 117.5 MB。详见方案报告第 05 节。）

## 版本号为什么不生效：`IMAGEOPT` 是隐藏前提

如果你只写 `CONFIG_VERSIONOPT=y`、`CONFIG_VERSION_NUMBER="24.10.6-nx30pro"`，`make defconfig` **不会报错**，但版本号也不会生效。原因是一条三层嵌套的 prompt 依赖：

1. 版本菜单不在主仓库的 `config/` 目录里（那只有 6 个文件），而是由包元数据脚本生成——`scripts/package-metadata.pl` 的 `gen_package_config()` 会打印 `menuconfig IMAGEOPT`（`default n`）并 `source "package/*/image-config.in"`，汇总进 `tmp/.config-package.in`。
2. `package/base-files/image-config.in` L153-154 才是真正的定义：
   ```
   menuconfig VERSIONOPT
       bool "Version configuration options" if IMAGEOPT
       default n
   if VERSIONOPT
       config VERSION_DIST / VERSION_NUMBER / VERSION_REPO / VERSION_FILENAMES ...
   endif
   ```
3. `IMAGEOPT` 默认是 **n**。kconfig 里**没有 prompt 的符号，`.config` 写什么都不算数**——`VERSIONOPT` 因此没有 prompt，`VERSION_FILENAMES`/`VERSION_NUMBER`/`VERSION_REPO` 全在 `if VERSIONOPT` 块内、一并失去可见性，被静默丢弃。

后果有三层：`VERSION_NUMBER` 的自定义值失效；`VERSION_REPO` 回落到 `version.mk` 的 tag 默认值；而那个默认值恰好也是官方地址，于是**问题被完美掩盖**——构建全绿，只有产物名里少了 `nx30pro` 字样。

**修法就是加一行 `CONFIG_IMAGEOPT=y`**（写在 `CONFIG_VERSIONOPT=y` 之前）。官方 `config.buildinfo` 第 252 行的 `CONFIG_IMAGEOPT=y` 可作印证。工作流里已加硬断言：`CONFIG_VERSION_NUMBER` 不等于 `"24.10.6-nx30pro"` 就直接 fail。

顺带一提，`CONFIG_STRIP_KERNEL_EXPORTS` **不能写**——它的 prompt 挂着 `depends on BROKEN`（`BROKEN` 默认 n），属于"无 prompt 符号"，写 `=y` 或 `is not set` 都不会出现在 `.config` 里。这个坑曾经让整条流水线卡死过一次。

## 刷完之后会自己运行的一次性脚本

`default-settings-chn` **一定会进固件**——它由 target 的 `Default-Packages` 无条件带入（官方 `profiles.json` 的 `default_packages` 共 42 项，它就在里面），任何 `INCLUDE_*` 开关都管不到，也不体现在任何包的依赖里。它装的 `/etc/uci-defaults/99-default-settings-chinese` 在**首次开机**执行两件事：

1. **把时区设好**：`system.timezone=CST-8`、`zonename=Asia/Shanghai`，NTP 换成 `ntp.tencent.com` / `ntp1.aliyun.com` / `ntp.ntsc.ac.cn` / `cn.ntp.org.cn`。所以手册里的"设时区"其实只是复核。
2. **改写 `opkg` 源**：`sed -i.bak "s,https://downloads.immortalwrt.org,$opkg_mirror,g" /etc/opkg/distfeeds.conf`，`opkg_mirror` 缺省为第三方镜像 `https://mirrors.vsean.net/openwrt`，原文件留在 `distfeeds.conf.bak`。

实测这个镜像**可用**：24.10.6 的 `packages/aarch64_cortex-a53/base/Packages.gz`、`targets/mediatek/filogic/packages/Packages.gz`、以及对应 vermagic 的 `targets/mediatek/filogic/kmods/6.6.133-1-a8b93917f464536104594f27d870028d/Packages.gz` 三个都返回 200，所以 `opkg update` 不会因为这次改写失效，国内线路反而更快。不想指向第三方就刷完执行一行：

```sh
sed -i 's,mirrors\.vsean\.net/openwrt,downloads.immortalwrt.org,' /etc/opkg/distfeeds.conf
```

注意 `CONFIG_VERSION_REPO` 仍然要写对：`distfeeds.conf` 是构建时由它拼接出来的，脚本只是替换域名，路径部分原样保留。

## 版本耦合红线：`xray-core` 与 `luci-app-passwall` 必须同代

Xray-core 自 v26.1.23（2026-01-23）起用 `pinnedPeerCertSha256`（pcs）替代 `allowInsecure`，v26.1.31 起含该字段直接拒绝启动，v26.2.6 放宽为"UTC 2026-06-01 前告警放行"，v26.9.9 又收回为无条件报错。同时期 passwall 也把生成的 TLS 段落整体换掉了。

**本方案的 pin 组合不受影响**：feed 里的 `xray-core` 是 `25.2.21`、`luci-app-passwall` 是 `25.12.16`，都早于变更起点，"跳过证书验证"开关照常有效。**不要为此改动任何配置。**

但换组件时有一条红线：

| 组合 | xray-core | luci-app-passwall | 结果 |
|---|---|---|---|
| 现状 | 25.2.21 | 25.12.16 | 可用，跳过验证有效 |
| 成对升级 | 26.x+ | main | 可用；开关消失，自签改为填 pcs |
| **只换内核** | 26.x+ | 25.12.16 | **危险**：勾"跳过证书验证"→ 整个 Xray 实例拒绝启动，全部节点一起断 |
| 只换 passwall | 25.2.21 | main | pcs/vcn 被静默忽略（旧内核不认），自签节点连不上 |

两个底层机制解释了上表，改配置前值得记住：

- **`allowInsecure` 只在取值为 `true` 时触发报错**，字段至今仍留在结构体里。passwall 的写法是 `(…) and true or false`，这个键永远会被写出来，默认 `false`——所以"升级后没全断"和"勾一下全断"是同一个原因。
- **Xray 用标准库 `encoding/json` 解码，没有 `DisallowUnknownFields()`**，未知字段被静默忽略。所以字段改名是**静默失效**而非报错，排查时不会有任何线索。

顺带一处既有偏差：`25.2.21` 的 `TLSConfig` 里没有 ECH 字段（2026 年才加入），而 passwall `25.12.16` 会写 `echConfigList` / `echForceQuery`——**界面上的 ECH 选项在当前 pin 组合下静默失效**，配了不生效但不报错。`tcpMptcp`、`dialerProxy` 则都存在。

详见随附的《Xray allowInsecure 弃用影响评估》。

## 配置改动的注意事项

- 改动代理组件（`INCLUDE_*` 系列）前，先确认该选项名在锁定 feed 提交下的 Makefile 里真实存在。
- **Hysteria2 的三个名字不一样，别搞混**：开关是 `INCLUDE_Hysteria`，被拉入的包名是 `hysteria`（版本 2.6.5，即 Hysteria2，包里不含一代），passwall 界面上的节点类型是 `hysteria2`。改动前对一下这一条，比改完再排查快得多。
- **ZeroTier 的开关名字母大小写与那个 `Z` 不能改**：包名是全小写的 `zerotier`，但两个子开关是 **`CONFIG_ZEROTIER_ENABLE_DEBUG` / `CONFIG_ZEROTIER_ENABLE_SELFTEST`**——**大写且不带 `PACKAGE_` 前缀**。这是它容易在自检里被漏掉的原因。
- 选了带 `INCLUDE_` 的开关就等于选了对应的二进制包，注意体积与常驻内存（例如 `INCLUDE_SingBox` 会把 41 MB 的 sing-box 拉进来，且常驻内存显著高于 Xray）。
- 别改 `CONFIG_VERSION_REPO`：`opkg` 的源地址由它拼接（`include/feeds.mk` 里的 `src/gz %d_core %U/...`），改错会让刷完之后的 `opkg update` 指向不存在的路径。
- 别打开 `BUILDBOT`、`ALL_KMODS`、`BUILD_LOG`、`KERNEL_DEBUG_INFO`：这些是官方 buildbot 专用开关，会让编译时间与磁盘占用大幅上升。
