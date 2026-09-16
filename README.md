# ImmortalWrt 24.10.6 for H3C Magic NX30 Pro (NMBM)


仓库里有 **两套互不干扰的构建方案**，都已编出并发布为 prerelease：

| | 方案 A · 官方基座 | 方案 B · padavanonly 闭源基座 |
|---|---|---|
| 无线 | 开源 mt76（`mt7915e`） | **闭源 MTK `mt_wifi`** ＋ `warp` |
| NAT 卸载 | 软件 flow offload | **HNAT 硬件卸载**（`mtkhnat.ko`） |
| 成功运行 | run #5 `34866009485`（4h42m） | run #7 `35002558354`（217.7 min） |
| Release | **`build-5`**（4 个资产） | **`mtwifi-build-7`**（5 个资产） |
| sysupgrade 体积 | 32.3 MiB | 38.1 MiB |
| 内核模块 | 89 个 | 159 个 |
| 风险 | 低（走上游主线） | 中高（同型机首例，见下「风险」） |

**选哪个**：要稳就 A；要闭源无线与硬件 NAT 的性能就 B。两者可以都刷一遍对比，互不覆盖
（产物名、Release tag 都带区分前缀）。


## 仓库结构

```
.
├── .github/workflows/
│   ├── build-nx30pro.yml                     # 方案 A：官方基座
│   └── build-nx30pro-padavanonly.yml         # 方案 B：padavanonly 闭源基座
├── nx30pro-nmbm.config                       # 方案 A 的编译配置
├── nx30pro-padavanonly-mtwifi.config         # 方案 B 的编译配置
└── README.md
```

两个工作流**各自只监听自己的 config 与 workflow 文件**，所以改 A 的文件不会触发 B 的构建，反之亦然。
仓库里没有源码 —— 每次构建时由 CI 现场从上游 clone，因此不存在「fork 落后上游需要同步」的问题。
**不需要任何 secrets。**



## 两套方案的关键参数

### 方案 A · 官方基座

| 项目 | 值 |
|---|---|
| 源码 | `immortalwrt/immortalwrt` tag `v24.10.6` |
| 设备变体 | `h3c_magic-nx30-pro-nmbm` |
| 版本号 | `24.10.6-nx30pro` |
| opkg 源 | `https://downloads.immortalwrt.org/releases/24.10.6` |
| vermagic | `6.6.133-1-205b2f77dbd69879db70a4bdaa1c49fc` |

用 tag 而不是分支，是为了可复现：该 tag 的 `feeds.conf.default` 把四个 feed 全部锁到具体提交，
编译出来的包版本与官方发布版一致，而不是跟着分支漂移。

### 方案 B · padavanonly 闭源基座

| 项目 | 值 |
|---|---|
| 源码 | `padavanonly/immortalwrt-mt798x-6.6`，分支 `openwrt-24.10-6.6`，**pin commit** `ec9ef10efc65da1e6d1de4e2c043c0e13d08eed8` |
| 设备变体 | `h3c_magic-nx30-pro-nmbm`（同上） |
| 版本号 | `24.10.6-nx30pro-mtwifi` |
| opkg 源 | `https://downloads.immortalwrt.org/releases/24.10-SNAPSHOT` |
| vermagic | `c64bb1b016d06278a0c58cbac85529ba` |
| 闭源驱动模块 | `mt_wifi.ko` · `mtk_warp.ko` · `mtk_warp_proxy.ko` · `mtkhnat.ko` |

**该仓库没有任何 release tag，所以只能 pin commit**，而且 feeds 也必须自己 pin ——
上游 `feeds.conf.default` 写的是分支名 `openwrt-24.10`（滚动的），同一次配置今天编和下周编会拿到不同的包版本。
工作流里锁了 4 个 feed SHA，改 `SRC_COMMIT` 时必须一并核对。

> ⚠️ **feeds pin 只能写 `url^<40位sha>`，绝不能带 `;branch`。**
> `scripts/feeds` 里 `;(.*)` 是贪婪到行尾的，会把 `^sha` 一起吞进 branch 名，导致 `git clone --branch '…^sha'` 直接失败。

## 刷机

两套方案刷机方式完全一样。进 U-Boot 的 failsafe 界面刷 `*-nmbm-squashfs-factory.bin`：

1. 网线接 LAN 口，电脑固定 IP `192.168.1.100` / 掩码 `255.255.255.0`（U-Boot 不提供 DHCP）
2. 断电，按住 RESET 通电，持续 15 秒以上
3. 浏览器隐身窗口访问 `http://192.168.1.1`，上传 `factory.bin`

**不要刷 `-bl31-uboot.fip` 和 `-preloader.bin`。** 那是引导程序本身，写坏只能拆机接 TTL。
分区表与 DTS 与官方逐字节一致 ⇒ **不需要换 U-Boot**。

**刷前请先备份 Factory 分区**（存放 MAC 与无线校准参数）。NX30 Pro 上的分区号已按实机确认，
**Factory 是 `mtd3`**（`mtd0` 是整片 NAND、`mtd1` 是 BL2、`mtd2` 是 u-boot-env）：

```sh
nanddump -f /tmp/factory.bin /dev/mtd3   # 或 dd if=/dev/mtd3 of=/tmp/factory.bin
ls -l /tmp/factory.bin                   # 必须是 2097152 字节（2 MiB）
```

后续升级用 `*-nmbm-squashfs-sysupgrade.bin`。首次开机头几分钟 CPU 占用偏高属正常（驱动初始化与固件加载）。

详见本工作区的《刷机与首次配置手册》（该文档未随仓库发布）。

## 必须知道的几个机制

这些是踩过坑之后固化下来的，**改动配置或工作流前值得先读一遍**。

### 1. 内核模块无法事后安装

内核的 vermagic 是三段拼出来的：`<内核版本>-<LINUX_RELEASE>-<配置哈希>`。
配置哈希的算法是 `grep '=[ym]' .config | LC_ALL=C sort | md5`（`include/kernel-defaults.mk` 第 130 行，写进 `.vermagic`），
再经 `include/kernel.mk` 读入 `LINUX_VERMAGIC`、由 `include/feeds.mk` 拼进 kmods 源路径。

**两套方案的 vermagic 都与官方发布版不同**（因为 `.config` 不同），所以官方 kmods 源里的包在依赖检查阶段就会被拒绝。
**凡是用得到的模块必须在编译时就选进去。** 用户态包（LuCI 插件、`xray-core` 这类）不依赖内核校验，`opkg` 随时可装。

### 2. Kconfig 对「写错的东西」一律静默

三类坑，方向各不相同：

- **没有 prompt 的符号，`.config` 里写什么都不算数。** 但还要分两问：① 我写的值生效吗 → 一律不能；
  ② 这行会出现吗 → 无 `default` ⇒ 不出现，**有 `default` ⇒ 照常出现并携带默认值**。
  （实例：`CCACHE` 在 `if DEVEL` 内；`STRIP_KERNEL_EXPORTS` 挂在 `depends on BROKEN` 上。）
  ⇒ 对无 default 的符号做「整行断言」必然误报。
- **★ 反方向：有 prompt 且 `default y` 的符号 —— 「不写」等于「打开」。** 这一条最阴险，
  因为配置里根本没有那一行，任何逐行检查都看不见它，只有值能露馅。
  ⇒ **以别人的 defconfig 为底稿必须整份继承（含末尾的 `# CONFIG_X is not set` 块），只能「增加」不能「摘取」。**
- **重复符号取最后一行。** `conf_read_simple()` 逐行无条件赋值、**不检测重复**。
  ⇒ **禁止对 `.config` 去重/排序/合并**；要覆盖先前设置就**就地改写原行**，不要「末尾追加」（会造出重复符号，日后人人读错）。

还有一条顺序敏感：`CONFIG_IMAGEOPT=y` **必须写在 `CONFIG_VERSIONOPT=y` 之前**
（`VERSIONOPT` 的 prompt 是 `bool "…" if IMAGEOPT`，而 `IMAGEOPT` 默认 n）。
否则自定义版本号静默失效、`VERSION_REPO` 回落到默认值，而**构建全绿** —— 问题被完美掩盖。
同理 `CONFIG_DEVEL=y` 必须在 `CONFIG_CCACHE=y` 之前。

### 3. 配置自检是这套流水线的核心，不是装饰

`make defconfig` 之后的**配置自检**。kconfig 对拼错的符号和不存在的包名一律静默丢弃、不报错 ——
没有这步校验，CI 会全绿通过，你刷完才发现固件里没有 PassWall。

自检分两类：

- **正向断言**（`chk_on` / `chk_off` / `chk_eq` / `chk_absent`）：验证「我要的东西真的生效了」。
- **★ 反向体检**：拿**源码树里自带的作者 defconfig** 当基准，找出「作者显式关闭、而我们这里被打开或压根没表态」的符号。
  判据分三档 —— `CONFIG_MTK_*`/内核类**值不同就 fail**；`CONFIG_PACKAGE_*` **一字未提才 fail**
  （写了 `=y` 属「有意偏离」，只列不判）；`CONFIG_TARGET_*` 排除（作者一份配置产出 19 个机型，我们只编一个）。

> **判据要判在「危险」上，不是判在「不同」上。真正的危险是「沉默」—— 我一个字没写，让默认值顶上来。**

产物阶段还有**实证核对**：不看 `.config` 里选了什么，只看编译产物里到底有没有 ——
直接去镜像里查内核模块与固件文件、去 `bin/packages` 里找 ipk 并解包验证内部文件。
这防的是「依赖解析没把包拉进来」这类 `.config` 断言看不见的问题。

### 4. 产物交付与自检是解耦的

**自检红掉会让运行变红，但不该让编好的固件消失。** 这是踩过坑之后特意做的设计，
改动工作流时**不要把它们重新串起来**。

| 步骤 | 条件 | 语义 |
|---|---|---|
| `Build firmware` | 无 `if`（隐式 `success()`） | 前序任一失败就不编 |
| `Verify artifacts and report sizes` | 无 `if`（隐式 `success()`） | 构建成功才校验；发现问题 `exit 1` |
| `Upload firmware` | `always() && steps.build.outcome == 'success'` | **自检红了也照传**；构建没成功过才不传 |
| `Publish release` | `always() && build==success && verify==success && !PR` | **不看 Upload 的结论** —— 两条交付路径互为冗余 |

`artifact` =「编出来就有」的兜底；`Release` =「过了全部自检」的承诺。两者语义不同，不该用同一个门槛。

另外：`kmods-built.txt` 与 `config.custom` 的生成被放在自检**断言之前**，否则断言一 `exit 1`，这两个文件永远不会存在。

> **GitHub Actions 的坑**：步骤不写 `if` 时才有隐式 `success()`；一旦条件里写了
> `success()` / `always()` / `failure()` / `cancelled()` 之一，隐式那条就失效，条件完全以写的为准。
> 所以写 `always()` 时必须自己把「构建成功过」补上。

> **当前状态**：方案 B 的工作流已按上表实现，并在 run #6 上实测过（自检红、artifact 照传 114 MB）。
> 方案 A 的工作流目前仍是旧时序 —— 一旦自检失败，产物不会上传，改造版本尚未推送上去。

### 5. 失败诊断必须能被匿名读到

公开仓库里，**job 日志匿名是读不到的**（`actions/jobs/{id}/logs` 返回 403，job 页面 HTML 只有框架、没有日志正文）。
所以：**任何「要人去翻日志才知道」的输出 = 没有输出。**

工作流里因此把长时间步骤的输出全部落盘（`make … > build.log 2>&1`），失败时把 grep 出的错误行 ＋ 日志尾部
发成 `::error::` —— 它会变成 check-run 的 annotation，**公开仓库可匿名读**。

## 排错

**构建红了先看注解，不要先翻日志。** Actions 运行页顶部 / 步骤旁的报错就是注解内容。

| 现象 | 大概率原因 |
|---|---|
| 第 9 步「配置自检」失败 | 配置里写了本树不存在的符号，或值与期望不符 —— 注解里会给出「期望 vs 实际」 |
| 第 9 步反向体检报「未表态」 | 某个 `default y` 的符号你没写、被默认打开了。若确实要它，显式写 `=y`；否则显式写 `# … is not set` |
| 第 11 步编译失败 | 注解里带错误行与日志尾部。常见是配置开关组合导致某源码引用了本树不存在的宏 |
| 第 12 步产物自检失败 | 镜像内容与预期不符。**先判断是「东西真缺」还是「判据前提错了」** —— 见下 |
| 日志里出现 `(共 0 个)` 这类零结果 | 几乎总是**路径基准错**，先查 `pwd` |

⚠️ **两个反直觉的坑**：

- **「搜不到 ≠ 不存在」。** 判某个包在不在，只能按「**包名 → `define KernelPackage/` 或 `define Package/`**」全树搜，
  而且**用「搜不到」当证据之前，必须先拿一个「必然搜得到」的词做对照** —— 否则你分不清是「真没有」还是「工具失效」。
  （GitHub 的 code search API 对这个仓库就是失效的。）
- **自检失败不一定是固件的问题，也可能是判据的前提错了。** 本项目有一次判据把 `mt7981_wo.bin` 当成硬性必需，
  而 padavanonly 转闭源时早已把配套的固件包删掉 —— 结果把一次**完全成功的编译**判成了失败。
  ⇒ 写自检逻辑时，**前提本身也要有证据**（去读文件内容、看 commit diff），并且**先用坏样本离线实测**，
  否则你分不清「没问题」和「没生效」。

## 刷完之后会自动运行的一次性脚本

`default-settings-chn` **一定会进固件** —— 它由 target 的 `Default-Packages` 无条件带入，
任何 `INCLUDE_*` 开关都管不到，也不体现在任何包的依赖里。它装的
`/etc/uci-defaults/99-default-settings-chinese` 在**首次开机**执行两件事：

1. **设好时区**：`system.timezone=CST-8`、`zonename=Asia/Shanghai`，NTP 换成国内源。
2. **改写 `opkg` 源**：把 `https://downloads.immortalwrt.org` 替换成第三方镜像
   `https://mirrors.vsean.net/openwrt`，原文件留在 `distfeeds.conf.bak`。

替换只动域名、路径原样保留，所以最终地址取决于该方案构建时写的 `VERSION_REPO`：
方案 A 是 `…/releases/24.10.6`，方案 B 是 `…/releases/24.10-SNAPSHOT`。

**刷完后建议先执行一次 `opkg update` 确认能用。** 若该镜像没有对应路径，改回官方源：

```sh
sed -i 's,mirrors\.vsean\.net/openwrt,downloads.immortalwrt.org,' /etc/opkg/distfeeds.conf
opkg update
```

## ZeroTier 怎么用

刷完之后 ZeroTier 默认是**关闭**的（`option enabled '0'`），需要手动开：

1. 打开 LuCI → **VPN → ZeroTier → Configuration**，勾选 Enable；或在 ttyd 里执行下面两条命令。
2. 加一条 Network，填入你的 16 位网络 ID；也可以直接 `zerotier-cli join <网络ID>`。
3. Interface info 页可以看到已加入的网络、分配的 IP 与 Peers 状态。

```sh
uci set zerotier.global.enabled='1'
uci set zerotier.global.copy_config_path='1'   # 见下
uci add zerotier network
uci set zerotier.@network[-1].id='<16位网络ID>'
uci set zerotier.@network[-1].enabled='1'
uci commit zerotier
/etc/init.d/zerotier restart
```

三个容易踩的点：

- **`allow_default` 保持 `0`。** 这一项是「允许 ZeroTier 接管默认路由」。一旦置 1，对外流量可能整体改走 ZeroTier，
  与 passwall 的透明代理叠加后行为难以预测。
- **`copy_config_path` 建议置 `1`。** 把 ZeroTier 的运行目录复制到内存而不是直接写闪存 —— ZeroTier 会频繁更新
  对等节点状态，长期直接写闪存不划算。
- **要对外连通，UDP 9993 得能进。** 它自带的 `/usr/bin/zerotier-fw4` 会往 `inet fw4` 表里注入规则（走 nftables，
  不需要 iptables），UCI 里的 `fw_allow_input` 控制的就是这个，默认已开。

**它和 passwall 不冲突。** passwall 的脚本里没有任何 ZeroTier 相关处理，透明代理只对选定的接口生效（通常是 `br-lan`），
而 ZeroTier 的接口名是 `ztXXXXXXXX`，默认不在其中。反过来，若希望 ZeroTier 流量也走代理，需要主动把该接口加进 passwall。

## 版本耦合红线：`xray-core` 与 `luci-app-passwall` 必须同代

Xray-core 自 v26.1.23 起用 `pinnedPeerCertSha256`（pcs）替代 `allowInsecure`，
v26.1.31 起含该字段直接拒绝启动，v26.2.6 放宽为「UTC 2026-06-01 前告警放行」，v26.9.9 又收回为无条件报错。
同时期 passwall 也把生成的 TLS 段落整体换掉了。

**当前 pin 组合不受影响**：feed 里的 `xray-core` 是 `25.2.21`、`luci-app-passwall` 是 `25.12.16`，
都早于变更起点，「跳过证书验证」开关照常有效。**不要为此改动任何配置。**

但换组件时有一条红线：

| 组合 | xray-core | luci-app-passwall | 结果 |
|---|---|---|---|
| 现状 | 25.2.21 | 25.12.16 | 可用，跳过验证有效 |
| 成对升级 | 26.x+ | main | 可用；开关消失，自签改为填 pcs |
| **只换内核** | 26.x+ | 25.12.16 | **危险**：勾「跳过证书验证」→ 整个 Xray 实例拒绝启动，全部节点一起断 |
| 只换 passwall | 25.2.21 | main | pcs/vcn 被静默忽略（旧内核不认），自签节点连不上 |

两个底层机制解释了上表：

- **`allowInsecure` 只在取值为 `true` 时触发报错**，字段至今仍留在结构体里。passwall 的写法是
  `(…) and true or false`，这个键永远会被写出来，默认 `false` —— 所以「升级后没全断」和「勾一下全断」是同一个原因。
- **Xray 用标准库 `encoding/json` 解码，没有 `DisallowUnknownFields()`**，未知字段被静默忽略。
  所以字段改名是**静默失效**而非报错，排查时不会有任何线索。

顺带一处既有偏差：`25.2.21` 的 `TLSConfig` 里没有 ECH 字段，而 passwall `25.12.16` 会写
`echConfigList` / `echForceQuery` —— **界面上的 ECH 选项在当前 pin 组合下静默失效**，配了不生效但不报错。

详见本工作区的《Xray allowInsecure 弃用影响评估》（该文档未随仓库发布）。

## 配置改动的注意事项

- 改动代理组件（`INCLUDE_*` 系列）前，先确认该选项名在锁定 feed 提交下的 Makefile 里**真实存在**。
- **Hysteria2 的三个名字不一样，别搞混**：开关是 `INCLUDE_Hysteria`，被拉入的包名是 `hysteria`
  （版本 2.6.5，即 Hysteria2），passwall 界面上的节点类型是 `hysteria2`。改动前对一下这一条，比改完再排查快得多。
- **ZeroTier 的开关名字母大小写与那个 `Z` 不能改**：包名是全小写的 `zerotier`，但两个子开关是
  **`CONFIG_ZEROTIER_ENABLE_DEBUG` / `CONFIG_ZEROTIER_ENABLE_SELFTEST`** —— **大写且不带 `PACKAGE_` 前缀**。
- 别改 `CONFIG_VERSION_REPO`：`opkg` 的源地址由它拼接，改错会让刷完之后的 `opkg update` 指向不存在的路径。
- 别打开 `BUILDBOT`、`ALL_KMODS`、`BUILD_LOG`、`KERNEL_DEBUG_INFO`：官方 buildbot 专用开关，
  会让编译时间与磁盘占用大幅上升。
- 方案 B 的配置是**从上游作者的 `defconfig/mt7981-ax3000.config` 整份继承**后再增改的。
  再动它时请记住上面第 2 条 —— **只能「增加」，不要「摘取」**。

## 风险与已知限制

| 编号 | 风险 | 当前证据 |
|---|---|---|
| R1 | 开源/闭源驱动争抢同一块 WMAC ⇒ 无线行为不可预测（**仅方案 B**） | **镜像侧已排除**：`/mt7915e.ko` 不在镜像里、`package/kernel/mt76` 整包不在源码树里；闭源四模块全在。**运行期行为仍需刷机验证** |
| R6 | 同型机无先例 —— 闭源方案在 NX30 Pro NMBM 上是首例（**仅方案 B**） | 无先例可依，只能实机验证 |
| — | 两套方案的 vermagic 都与官方不同 | 无法用 `opkg` 补装内核模块；用户态包正常 |
| — | 两个 Release 都是 `prerelease=true` | GitHub 不会显示为 Latest |
| — | Release 资产里没有 `sha256sums` | 只在 artifact 里；要刷机前校验可从 artifact 取 |
