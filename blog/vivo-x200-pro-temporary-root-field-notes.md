---
title: "vivo X200 Pro 临时 Root 完整实录：watchdog、系统备份、GKD 与 Zygisk 隐藏回退"
published: 2026-11-13
description: "记录同一台 X200 Pro 从 ADB、GhostLock 配置和 DirtyFrag VR watchdog，到 super 续传、NeoZygisk/Vector、GKD 及 NyaZygisk 对照回退的全过程，保留失败、证据和未验证边界。"
tags: ["Android", "vivo X200 Pro", "Root", "KernelSU", "LSPosed", "GKD", "故障排查", "A/B 测试"]
category: 设备实践
draft: false
---

记录日期：2026-11-12—2026-11-13，时区 Asia/Shanghai。

这次工作从连接手机和生成 GhostLock 配置开始，后来扩展到 watchdog 崩溃分析、当前系统镜像提取、完整系统与 Root 备份、NeoZygisk / Vector 激活、系统去广告，以及 Zygisk 隐藏方案的对照和回退。

最终保留的是 **ReSukiSU 临时 Root + NeoZygisk 2.4 + Vector 2.2**。GhostLock 配置完成了镜像核验和导入，但没有执行该配置的提权；NyaZygisk 的加强检测结果没有稳定复现，因此已回退。GKD 确认配置和服务就绪，没有把未遇到广告当成跳过成功。

本文由设备操作者与 Codex 协作整理，依据当次会话与本地留存收据。操作者负责重新运行提权和设备授权；后续采集、工具操作和多数检查通过 ADB 完成。公开材料已去掉设备唯一标识、联系人、通知、个人应用范围和生产连接资料；原始镜像与完整日志留在本地。

完整机型为 vivo X200 Pro / V2405A / PD2405M / MT6991，采集时为 Android 15 / OriginOS 5，构建 `PD2405B_A_15.0.27.0.W10.V000L1`，B 槽内核 `6.6.57-android15-8-ge9c3d7352454-abogki428077543-4k`。这些细节限定本文范围，不构成同系列其他设备的适配声明。

GitHub 档案与机器可读证据：[Maximora-byte/vivo-x200-pro-root-notes](https://github.com/Maximora-byte/vivo-x200-pro-root-notes)。这是一份包含失败记录的实录，没有自动刷写或自动提权入口。

## 1. 时间线与证据层级

所有时间均为北京时间。内核日志中的秒数另指该次开机后的单调时间；不能把 DropBox 收录时间当成故障指令执行时刻。

| 阶段 | 动作和结果 | 证据层级 / 限制 |
| --- | --- | --- |
| 11-12，连接准备 | Windows 识别手机，但 ADB 设备列表为空；操作者解锁、允许 USB 调试并重插 | USB 枚举不等于 ADB 已授权 |
| 11-12，版本变化 | 初期线索涉及另一系统和 b57 内核，后续实际系统为 15.0.27.0 / e9c | 旧构建的经验没有迁移为本构建的运行证据 |
| 11-12 23:37 左右 | 按上游同完整内核 target 建源码候选；NDK 语法与共享库链接通过 | `verified=false`；未核对本机镜像；Windows 包装分支编译仍缺 `SLIDE_INIT_CRED_OFF` |
| 11-12 23:56 / 23:58 收录 | 系统保留两次不同故障记录 | 前一次调度 NULL 异常与后一次 DirtyFrag VR atomic/RCU 错误链分开分析 |
| 11-13 00:16 左右 | 核对月虹 1.6.8 / DirtyFrag VR，开启去除 VR 探针并保留运行内核 BTF 自动偏移 | 此时只改设置，诊断报告的 `runtime_fix_verified=false` 是当时状态 |
| 其后 | 操作者再次运行；后续验证 Root，开启 ADB Root 后开展只读分区采集 | 后续 Root 可用，不能反推最初设置已被单独证实为唯一修复原因 |
| 11-13 00:32 左右 | 用当前 B 槽 boot / vendor_boot 与运行 BTF 完成 GhostLock 参数核验及导入 | 15 个 task_struct 字段、10 项符号/槽位核对；未执行 GhostLock |
| 11-13 至 10:33 | 扩展系统备份；super 经断流、重连和断点续传后完整校验 | 108 份镜像；不含照片、聊天、个人文档及 userdata；未执行恢复 |
| 10:50 | NeoZygisk 2.4 / Vector 2.2 安装器成功，目录处于待激活 | 安装成功不等于 Zygote 注入 |
| 11:00 左右 | 经授权执行一次 ReSukiSU 模拟软重启；核对 Root、Enforcing、PID、注入与 IPC | 内核启动标识不变；具体 Xposed 功能模块数量为 0 |
| 11:42—12:03 | 关闭快应用和系统推广，安装 GKD，导入仅开屏规则，开启普通无障碍和保活设置 | 处理了 Switch 语义、导入数组格式、导入后启用状态和 UI 转储干扰 |
| 12:05—12:21 左右 | 53 个应用开屏动作权限限制；GKD 扩为在线订阅 | 本地筛选版保留关闭；在线 v632，8 类启用，敏感规则和应用排除 |
| 12:27—12:44 左右 | Native Detector 默认正常，加强检测 Zygisk 1 项；固定金融类逐应用隔离 | 启动抽查没有看到 Root 弹窗，未验证登录、支付、交易 |
| 12:50—13:14 左右 | NyaZygisk 2.5 替换、复测、清理残留、原版对照、重复失败、最终回退 | 本段共 5 次模拟软重启，无整机重启；原版二进制哈希和配置复核完成 |

### 四类证据不要混写

- **实测确认**：当次设备读回、哈希、IPC、界面状态或原始日志直接支持。
- **候选推断**：例如 VR 回调关联、管理器进程与检测变化的关系，仍有未排除变量。
- **上游自述**：项目说明支持某系统或隐藏技术，不等于本机运行通过。
- **未验证**：GhostLock 提权、备份恢复、银行完整流程、长期稳定性和服务端完整性认证。

最终状态以最后一轮回退验收为准。早期“待激活”“尚无 Root”“候选修复未运行”等历史记录保留其时间语境，不应被偷偷改写成从头到尾成功。

## 2. DirtyFrag VR 与 watchdog

### 版本与触发信息

设备操作者确认最近一次使用的是月虹的 **DirtyFrag VR**。当时安装包为月虹 v1.6.8（versionCode 168），管理器为 ReSukiSU v4.2.0-rc3 / 35171，内核日志中的驱动也为对应版本，LKM 模式。进程名 `libksud.so` 的归属判断结合了管理器初始化和应用 UID 映射，而不是只凭文件名。

### 最近一次清晰的错误链

见 [脱敏摘录](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/evidence/watchdog-excerpt.txt)。大约在开机第 69 秒，日志出现 DirtyFrag 进入、管理器初始化、SELinux 恢复 Enforcing，紧接着发生：

```text
BUG: scheduling while atomic
syscall_trace_exit
  -> <未符号化回调 A/B>
  -> do_exit
  -> mmput / __mmput
  -> exit_mmap / unmap_vmas / unmap_page_range
  -> __pte_offset_map_lock
  -> _raw_spin_lock
```

之后出现 RCU stall、多 CPU 停滞，最终被 watchdog 复位。问题重点已经进入管理器接管、回调与退出清理路径；不能简单写成“初始提权完全没进入”。反过来，模块初始化成功也不代表稳定 Root。

VR / 反提权回调是首要嫌疑，但两个运行时回调地址没有被映射到精确 `vr.ko` 符号。因此不能把“VR 已被证实为唯一根因”写进结论，也没有证据说换一个管理器一定能修好。

### 前一次是另一条路径

前一次保留记录中有 `0xa8` NULL 指针访问，PC 为 `rt_mutex_setprio+0x2a0/0x4e8`，进程名 `id`，涉及 `rt_mutex_adjust_prio_chain`、`__sched_setscheduler` 和 `sched_setattr`。它与后一次页表锁 / RCU stall 记录分别保留，没有合并成同一个已经证实的原因。

### 实际采取的候选处理

月虹 DirtyFrag VR 页面上：

- 将“激活时去除 VR 探针”由关闭改为开启；
- 保留“从当前内核 BTF 自动获取偏移”；
- 没有手工搬用别的内核偏移，没有关闭 watchdog。

当时的采集报告只记录设置变化，没有执行漏洞。安装包引擎中的运行内核 BTF、偏移解析失败、探针去除失败时在管理器启动前退出等字符串，支持它包含这条候选路径，但不证明该路径已经执行成功。

操作者后来再次运行，后续会话确实拿到 Root，并完成备份和框架工作。这个后续观察比“只改了开关”更强，但仍缺少同条件多次 A/B、每次运行终端和两个回调的精确符号归属；所以本文把它称为**候选规避后观察到 Root 恢复**，没有发布“watchdog 永久修复补丁”。

对应管理器源码线索：[固定提交的 syscall_hook_manager.c](https://github.com/Baka-SU/BakaSU/blob/239e1e8871b8fcd51a6e5b3002e0ba522fdd99fb/kernel/hook/syscall_hook_manager.c)。其中的 sys_enter 注册逻辑不能直接替代本机 sys_exit 两个地址的符号证据。

完整 pstore、DropBox、recovery last_kmsg 和 bugreport 仅在本地保存；公开仓库只给最小错误链。

## 3. GhostLock 配置的三个阶段

### 一开始没有对应完整固件

系统已经变为 15.0.27.0，原先其他构建 / b57 的数据不能沿用。查找本地降级包、boot 镜像和降级工具缓存没有取得可核验的完整固件，因此先做了明确标记 `verified=false` 的上游源码 target 候选。

来源是 CyberMeowfia 固定提交 `c87b8664c8d910a68c132dcb940696a95ea8f527` 的 PD2405 target。完整内核字符串一致，但上游用户空间构建标签与本机不同。NDK r28 / AArch64 / API 35 的语法检查和共享库链接成功，Windows 包装分支却仍缺 `SLIDE_INIT_CRED_OFF`，且缺少它额外要求的窗口数据。

**因此“能编译”没有被写成“Windows 一键工具已支持”。** 也没有把保留未验证标记的候选扔进自动匹配目录。上游实现和 Windows 包装流程的行为与停止阶段并不相同。

### 获得 Root 后，改用当前 B 槽镜像

读出 `boot_b`、`vendor_boot_b` 后，手机端与电脑端 SHA256 对比一致。镜像提取 BTF 与运行内核 BTF 字节一致，再核验结构、符号和物理布局。

| 核对项 | 本次结果 |
| --- | --- |
| task_struct 字段 | 15 项 |
| 符号 / 槽位偏移 | 10 项 |
| 镜像 / 运行 BTF | 完全一致 |
| init_cred | 补齐为 `0x20e0678` |
| waiter_shift | 根据本镜像 pselect / futex 栈帧推导为 `-2` |
| logger 槽位 | 修正为 IPv4 槽位 `0x20c21b8`，不能只用表起点 `0x20c21b0` |
| tracepoint.funcs | BTF 核对 `0x48`；没有擅自加入 App 不支持的字段 |
| 6.x 不使用的引用字段 | 合法保留 null，与漏填必填字段区分 |

[配置](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/profiles/PD2405-6.6.57-from-device.conf)和[核验摘要](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/evidence/ghostlock-validation.json)保留在仓库。它们只对应文中精确版本与镜像，不能按机型系列、内核主版本或 Android 版本通用套用。

### 导入结果与停止边界

通过 GhostLock 的导入入口加载后，App 私有目录保存的配置 SHA256 与电脑文件一致；缺失参数提示消失。公开仓库的文本副本统一为 LF 换行，因此另列 public_copy_sha256；原始导入收据中的 configuration_sha256 仍指原电脑 / 手机文件，不能混用这两个文件哈希。界面又提出 Shizuku 前置要求，但本轮没有据此继续执行 GhostLock。

完成的是：**提取 → 参数核验 → 导入和读回**。没有执行本配置的 GhostLock 提权，也没有开启强制攻击测试。当前真实 Root 来自之前的 DirtyFrag VR，不能拿它给 GhostLock 配置背书。

仓库不复制第三方 `root.c`、二进制 payload 或完整固件，源码依据以固定提交链接保留，第三方授权见[来源](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/docs/sources.md)。

## 4. 系统备份与 super 续传

### 范围先确定

操作者选择先做系统与 Root 相关备份，不包含照片、聊天、个人文档，也没有把 userdata 纳入本次备份。需要保留的系统恢复材料含设备唯一校准数据，因此“没有照片”不代表“这些镜像都适合公开”。

本轮最终摘要记录 **108 份分区或小型 UFS 设备镜像**：初始 boot_b / vendor_boot_b 加 106 份额外镜像，其中包括完整 super；BTF、GPT 头等辅助文件不计入这个数字。

还保留两槽启动链、init_boot、dtbo、vbmeta、基带、NVRAM / NVDATA / persist、校准材料、Root 工具和应用数据、ADB 配置、内存布局、模块与 BTF。运行中的应用数据归档不等于关机一致性快照；A 槽可能保留另一版本，不能因为目录名写当前 B 槽构建就把 A 槽也叫相同系统。

### super 是什么

super 是 Android 动态分区容器，承载 system、vendor、product 等逻辑分区；它不是照片或聊天所在的 userdata，也不能等同于完整手机镜像。参见 [AOSP 动态分区说明](https://source.android.com/docs/core/ota/dynamic_partitions/implement)。

### 失败和调整

1. 早期直接 gzip 长流经过 USB 传到电脑，发生截断；损坏流没有计为有效备份。
2. 后来先在手机本地用 1 MiB 块读取并压缩 super，得到可核验的完整源文件和源哈希。
3. 电脑通过 ADB 分块传回；设备或 USB 读取中断时保留已完成块，从断点继续。
4. 大约 15.2% 和 59% 阶段都出现连接问题，操作者重新授权、插拔，并表示会换线或接口。**没有证据把所有中断单一归因于线材，也没有因此诊断存储介质损坏。**
5. 最后做 gzip 完整性、完整解压长度及手机源数据和电脑解压数据 SHA256 比对。

### 完成收据

| 项目 | 记录值 |
| --- | --- |
| 完成 | 2026-11-13 10:33 左右 |
| super 压缩字节 | 15,922,730,197 |
| 解压字节 | 18,790,481,920 |
| 解压 SHA256 | `22b4eef263c0db976d9807dbc26eb4de79d991eda9433b03702752873505a973` |
| 压缩 SHA256 | `e39e35971e7be97f444a40567864c7006994e8b36f1be6d7110941fa14510a61` |
| 主备 GPT | 4096 字节扇区下的头与分区项 CRC 核验通过 |
| 工具 APK | 5 份，仅本地留存 |
| 电脑归档总字节 | 16,583,944,789 |
| 恢复演练 | 未做 |

成功收据之后，才告知可以断线。单凭压缩文件存在、进度到 100% 或设备仍被 Windows 识别，都不够证明备份完整。

### 不公开的部分

所有分区镜像、校准数据、Root 应用私有数据和授权表留在本地。跨设备恢复 NVRAM、persist 或加密相关数据不是本文的使用场景；本文也没有执行刷回备份。公开收据只证明当次读取和校验到哪里，不证明任意恢复方案安全。

## 5. NeoZygisk 与 Vector 安装、激活

### 名称和职责

- ReSukiSU 是当前临时 Root 的管理 / 用户空间体系，对应内核驱动版本也需要匹配。
- NeoZygisk 是 Zygote 注入与 Zygisk API 提供者，不等于一个 Xposed 功能模块。
- Vector 提供 Xposed 兼容框架；具体功能模块还需要另装、启用和设置作用域。

本轮最终保留 NeoZygisk v2.4 build289、Vector v2.2 build3080 / API102，没有安装具体 Xposed 功能模块。

### 安装阶段

从官方 GitHub Releases 取得 ZIP，核对发布摘要和手机端副本，ZIP 自检通过，先备份原 /data/adb。安装器成功时文件仍位于 `modules_update`，Zygote 与 system_server PID 没变，因此记录为“安装成功、尚待激活”。

NeoZygisk 对 ReSukiSU 35171 输出超过其充分测试版本范围的警告；没有把安装器未中止当成兼容性保证。

### 临时 Root 的激活方式

操作者明确选择模拟软重启，而不是整机重启。通过 ReSukiSU 的 `soft-reboot` 重建 Android 框架后：

- 内核启动标识不变，uptime 连续；
- Zygote、system_server PID 更新，`sys.boot_completed=1`；
- ADB 仍为 uid=0，SELinux Enforcing；
- 提供者报告 monitor tracing / zygote injected / daemon running；
- Vector CLI IPC 返回成功，API102，日志确认系统服务注入；
- 模块清单 enabled=true / update=false；
- 后续回退时又核验管理器显示“已激活”，不能只依赖 CLI。

**模拟软重启仍会关闭应用，不是无感设置。** 当前 Root 是临时 LKM，不能把普通整机重启当作同一操作。本文只描述已获同意的这次操作，没有建议陌生版本直接重放命令。

### 留下的诊断信息

初次激活日志包含两次 init-group ioctl EPERM，以及反射 / Trace / SELinux 探测诊断。之后基础注入成功，并未关闭 Enforcing 来抹平日志。这只能证明当次基础框架可工作，不能证明每个 LSPosed 模块或长期稳定性。

### 后来隐藏试验里的重要反例

NyaZygisk 替换期间，模拟软重启留下旧 Root 守护进程，新旧 Vector 并存时新进程崩溃。随后出现过“CLI status 成功，但管理器仍异常”。清理遗留进程并干净启动后管理器才激活。

因此验收应同时覆盖进程唯一性、注入、真实 IPC、管理器和目标功能；单个绿色状态不够。试验中的额外 core 只在保留诊断日志后清理，没有清应用数据。

## 6. 去广告、摇一摇与 GKD

### 可恢复的系统设置

快应用使用本机服务开关关闭并停止进程，保留安装包。系统广告与隐私中的个性化广告、个性化推荐、智能推荐服务关闭；全局搜索的广告个性化、推荐词、卡片和热搜推广关闭；应用商店的开屏、悬浮球、限时推广、顶部轮播及栏目、个性化推荐关闭。

浏览器原有极简首页、已关闭的个性化服务和通知栏总开关予以保留，另关闭资讯推荐和积分任务弹窗。没有清浏览器数据，也没有为改子开关先把原本关闭的通知总开关重新打开。功能更新、下载完成等项目与营销推广区分处理。

关闭个性化推荐不等于广告数量归零。设置入口和活动名称与系统版本有关，本次没有复制仅适用于 OriginOS 6 / 7 的隐藏开关。

### 摇一摇 / 扭一扭

权限管理 → 访问设备动作与方向 → 全部仅开屏时禁止。界面读回：53 个应用全部落入这个类别，允许 0、永久禁止 0。

限制是**启动前 6 秒**，之后保留正常传感器用途；不等于永久禁用陀螺仪，也不能保证 6 秒之后的广告完全无法触发。没有全局断掉传感器或修改驱动。

### GKD 配置演进

1. 安装官方 GKD v1.13.1 / versionCode101，校验 APK。
2. 首次使用声明由操作者在手机上阅读并决定接受；不是规则自动点同意。
3. 最初筛为本地开屏专用：1 组全局、7 个应用专用组；排除金融、认证、权限、管理器等流程，并去掉坐标猜测。
4. 用户要求增加可用规则后，从 Lin-arm 当天规则形成过 22 应用 / 82 组的本地筛选候选。
5. 用户随后选择在线自动更新：接入 id667 / v632，本地候选关闭保留，避免重复点击。
6. 最终在线库当时有 1003 应用、2508 应用组、3 全局组；本机实际加载 31 应用、119 应用组。**库总量不等于全部都启用。**

在线来源：<https://gkd-subscription-667.pages.dev/gkd.json5>，GitHub：[Lin-arm/GKD_subscription](https://github.com/Lin-arm/GKD_subscription)。没有用即将停用的旧域名。旧 AIsouler 仓库已停止维护 / 归档；这也是更换来源的原因。

最终按操作者要求开启广告和普通骚扰类：开屏、局部、全屏、分段广告及青少年模式提示、更新、评价、通知提示。权限提示、功能类、其他保留关闭；43 个敏感应用和 33 个涉及协议 / 授权 / 支付等规则组设置排除。规则不是通用的“自动同意 / 自动登录 / 自动安装”工具。

自动更新频率为每天。新装应用如果库中已有适配可被匹配，没有适配仍要等维护者补充。后续线上规则会变化，不能保证未经重新审查的新增行为永远无误触。

### 实际配置与踩坑

- 普通无障碍即可；没有给 GKD Root、Shizuku、写安全设置、DUMP、伴生服务或触摸优化权限。
- 开启常驻通知、自启动、忽略电池优化；默认无障碍保活浮层保留。
- 某些 vivo 页面同时暴露外层 Switch 行和内层真实开关，外层 checked=false 不能代表实际值；曾短暂误开后纠正，改用语义与叶节点核验。
- GKD 备份导入要求选择器单值正规化为数组；首次导入失败的日志定位到 excludeMatches 字段，修正格式后成功。
- 导入成功不保证订阅已启用；既有本地订阅可能保留原来的关闭状态，需要核对 UI 与落库 enable。
- ADB uiautomator 转储会短暂抑制其他无障碍服务，检查本身可能造成 GKD 重连；后续采用截图 / 日志验收，故障标记出现时恢复同一已授权服务。

### 验证到哪里

最终确认普通无障碍运行、订阅启用、类别 / 排除项落库、每天更新配置，以及测试应用规则加载。测试中没有取得真实广告跳过 action_log；其中一个测试应用只出现启动画面，另一个停在首次声明且没有被代点同意。

所以交付是**配置和服务就绪**，不是“所有广告实机跳过成功”。完整本地订阅和个人应用配置不公开搬运；[公开类别快照](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/evidence/gkd-policy.json)只保留通用策略和计数。

## 7. Root 隐藏对照与最终回退

### 原环境与测试器

最终前的原环境是 ReSukiSU 35171 临时 Root、NeoZygisk 2.4、Vector 2.2。Native Detector 7.7.0 官方 APK 经摘要校验，检测器不授 Root。

默认检测显示环境正常，但实验检测与自定义 ROM 检测打开并冷启动后，出现 `Detected Zygisk (1)`，详情只有 `No info given`。不能用默认绿页替代加强模式。

金融类应用固定为自定义 UMOUNT、不授 Root；原来本就显示默认 UMOUNT，因此把它固定下来，不等于已经提高了隐藏强度。授权表在后续框架试验前后保持一致。

多款应用只做启动抽查，未见 Root 弹窗；部分停在登录或首次协议页面，没有代为接受协议、登录、支付或交易。也没验证 Play Integrity 或平台服务端风控。

### 为什么试 NyaZygisk

NyaZygisk 是 NeoZygisk 的较小规模社区分支，不是这里所谓“最主流、已经证明更安全”的替代品。v2.5 发布说明列出映射 / CFI / TracerPid 处理和临时 Root、LKM late-load、模拟软重启支持，值得针对本机验证，但只是上游自述。

校验官方 v2.5 build351 ZIP，先保存原模块、配置与官方 Neo 回退包。用户同意模拟软重启及必要回退后，替换提供者；没有让两个提供者同时注入，没有补内核 SuSFS。当前内核的 SuSFS 查询本就返回 Unsupported。

### 完整观察矩阵

| 条件 | 结果 | 能下什么结论 |
| --- | --- | --- |
| Neo + 工作中的 Vector，加强检测 | Zygisk 1 项 | 原问题存在 |
| Nya 首次启动，Vector 异常 | 环境正常 | 无效改善证据：不能牺牲 Vector 后宣布成功 |
| 清理残留后 Nya + Vector 激活，管理器进程存在 | Zygisk 1 项 | 单看提供者状态或 CLI 不够 |
| Nya + Vector 后台，结束管理器 UI 进程 | 某轮环境正常 | 只说明该轮通过，尚非稳定结论 |
| 切回 Neo，同样保留 Vector、结束管理器 UI | Zygisk 1 项 | 有同条件对照，但仍需重复 |
| 再装 Nya、干净激活、Vector 确认激活、结束管理器 UI、再次冷启动检测 | 又是 Zygisk 1 项 | **没有稳定复现改善，否决候选** |

管理器进程 / 时序可能影响观测，但不是已经定位的唯一原因；后一次反例否定了“关闭管理器就一定解决”的说法。公开记录保留中途的正常与最后失败，而不挑一张绿图宣传。

### 激活过程里的兼容性问题

第一轮软重启保留了旧的 Root 守护进程，同时新 Vector 守护进程发生崩溃。CLI 一度连接拒绝；仅恢复 CLI 后，管理器仍显示异常。清理旧进程并干净激活后，管理器才显示已激活。

重启服务时还遇到普通 Android unshare 不支持参数的问题，使用模块原本依赖的 KernelSU BusyBox 执行环境恢复。没有因此放宽全局 SELinux 或修改系统二进制。

这些处理是当次现场的诊断与恢复，**不是可无条件复制的杀进程清单**。PID 会变，不能重放记录中的 PID；需要识别实际提供者、守护进程、模块阶段、作用域和锁。

### 最终回退收据

本段共 5 次模拟软重启：首次激活、干净激活、原版对照、候选重复验证、最终回退。全程内核启动标识不变，无整机重启。

最终恢复 NeoZygisk v2.4 build289，运行二进制逐项与官方 ZIP 比对；Vector v2.2 管理器显示已激活、CLI API102 成功；GKD Bound service 正常，Crashed services 为空；金融应用授权表字节不变。功能配置记录的顺序变化，但 ID / 值集合一致，不写成“原始文件字节没变”。

Nya 未保留为当前运行方案；试验生成的 core 在保存崩溃日志后清理。最终结论只是**本次候选没有在这台设备上稳定解决问题**，不能推断所有不改分区的方案都不可能有效。

摘要见 [trial-results.json](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/evidence/trial-results.json)，来源见 [sources.md](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/docs/sources.md)。

## 8. 复核方法与验证清单

本轮使用 Android SDK platform-tools 37.0.1 和独立 ADB server 端口 5038；示例命令按读者实际环境调整 `-P`。Windows 枚举与 ADB 授权是不同层，不能把换连接方式直接当成唯一根因证据。

以下是观察命令，不包含提权、刷写、激活或自动修复流程。示例需要已有 ADB；某些文件还需要已经获得的 Root 权限。ADB 端口 / 路径依本机实际环境调整。

```sh
adb devices -l
adb shell getprop ro.product.model
adb shell getprop ro.build.display.id
adb shell uname -r
adb shell getprop ro.boot.slot_suffix
adb shell id
adb shell getenforce
adb shell getprop ro.boot.flash.locked
adb shell getprop ro.boot.vbmeta.device_state
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop sys.boot_completed
```

`id` 描述的是当前调用上下文。ADB 是 shell 不代表受控 Root 应用必然无 Root；ADB 是 root 也不代表所有应用都被授权。管理器 UI、应用配置、真实调用与 SELinux 状态分别看。

比较框架激活前后，可在本地读取 `/proc/sys/kernel/random/boot_id` 与 uptime，判断内核是否重新启动，但不应把真实标识贴进公开报告。模拟软重启的验证还要看 Zygote 和 system_server 是否更新，不能只看 sys.boot_completed。

已有这些模块时，可只读查看：

```sh
adb shell cat /data/adb/modules/zygisksu/module.prop
adb shell cat /data/adb/neozygisk/module.prop
adb shell /data/adb/lspd/cli --json status
adb shell dumpsys accessibility
```

路径是本轮版本的记录，不是不同实现通用接口。`dumpsys accessibility` 只读；不要把它和可能暂停其他无障碍服务的 `uiautomator dump` 混为一谈。

### 验收表

- [ ] 完整系统、内核、槽位和镜像一致性都核对过。
- [ ] 候选、已导入、已安装、已激活、功能工作分别记录。
- [ ] 崩溃记录保留时间、开机单调时间、来源与归因边界。
- [ ] 备份通过长度、格式完整性和源 / 目标哈希，而不只是传输进度。
- [ ] 隐藏测试开关一致，检测器没有被授 Root，冷启动过程与共存模块状态可复核。
- [ ] 新方案重复失败时能够退回已知配置，且回退后的实际运行状态重新验证。
- [ ] 图中没有联系人、通知正文、账号、序列号、真实启动标识和目录凭据。
- [ ] 未测试的登录、支付、交易、云端完整性认证、恢复和长期稳定性没有写成通过。

## 9. 公开证据与隐私边界

这里的完整指覆盖全部阶段、失败、错误假设、候选处理、后续证据和回退，不是把个人手机和生产服务器的原始数据全部上网。

### 公开

- 精确机型 / 构建 / 内核与工具版本；
- 去标识的时间线、错误链、状态矩阵和摘要；
- 官方发布包校验值、当前 B 槽公共系统镜像的核验摘要；
- 未执行提权、已导入的 GhostLock 版本限定配置；
- 通用 GKD 类别策略与计数，而不是个人应用列表；
- 只读验证方法、来源固定提交与最终回退状态。

### 只在本地

完整 bugreport、pstore / DropBox 全文、设备序列号 / 启动标识、联系人及通知截图、金融应用个人页面、全部软件清单、Root allowlist / 应用私有数据、NVRAM / NVDATA / persist / 校准、磁盘镜像和生产服务器连接资料。

这些材料里即使没有照片，也可能含设备唯一数据或个人信息。仓库 `.gitignore` 同时阻止常见镜像、备份、日志、APK 和凭据文件进入提交，但忽略规则不是隐私审查的替代品。

### 证据索引

| ID | 覆盖阶段 | 留存材料 | 公开形式 |
| --- | --- | --- | --- |
| E01 | ADB 与版本 | 设备采集、初期候选与构建收据 | 设备表与时间线 |
| E02 | watchdog | 完整 bugreport / pstore / DropBox / recovery、版本与界面 | 最小错误链、版本和推断边界 |
| E03 | 镜像配置 | boot / vendor_boot / BTF、提取器与导入收据 | 配置与去标识核验 JSON |
| E04 | 系统备份 | 108 镜像、GPT 与 SHA256 清单、续传记录 | 汇总与完整性收据 |
| E05 | 框架安装激活 | 安装日志、前后状态、IPC、注入与界面 | 版本 / 状态 JSON 与章节 |
| E06 | 去广告 | 原 / 新设置、UI XML、GKD 安装 / 导入 / 库配置 | 改动类别和验证限制 |
| E07 | Root 隐藏 | 加强模式截图、逐应用配置、启动抽查 | 脱敏矩阵，不披露个人应用范围 |
| E08 | Nya 对照回退 | 安装 / 控制 / 崩溃日志、二进制哈希、反例截图与恢复收据 | trial-results.json 与完整叙述 |

早期报告记录当时不知道的事，后续摘要记录后来验证到的事；存在时间差不等于可以删掉早期失败。本文也没有根据历史内存把另一版本的 Root 失败写成本构建已验证结果。

## 10. 最终保留什么

当前保留原版 NeoZygisk 2.4、Vector 2.2、金融应用逐应用隔离与 GKD 在线规则。Root 仍是临时 LKM，SELinux Enforcing；没有写底层分区，也没有整机重启。Nya 试验段使用的 5 次模拟软重启已列入记录，初始框架激活还有单独一次，不能把它们描述成“整个过程没有任何应用中断”。

本次没有证明 GhostLock 提权、系统镜像恢复、所有广告跳过、所有 LSPosed 模块兼容，或银行登录、支付、交易与服务端完整性认证通过。一次编译、一张绿色检测图或一个成功的 CLI 响应，都不能替代这些独立验收。

[版本与上游来源](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/docs/sources.md) · [机器可读试验摘要](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/evidence/trial-results.json) · [系统备份收据](https://github.com/Maximora-byte/vivo-x200-pro-root-notes/blob/main/evidence/backup-receipt.json)。
