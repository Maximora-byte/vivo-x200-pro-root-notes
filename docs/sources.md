# 来源、版本与归属

本文主证据是 2026-11-12—13 当次会话的本地采集与回退记录。下列是对应第三方项目 / 文档，不表示作者或维护者为本设备背书。

| 来源 | 本轮使用或调查的版本 / 固定点 | 用途 |
| --- | --- | --- |
| [CyberMeowfia](https://github.com/theVakhovskeIsTaken/CyberMeowfia/tree/c87b8664c8d910a68c132dcb940696a95ea8f527) | c87b8664… | 初期同内核 target 候选；上游代码为 Apache-2.0，未在本仓库复制 |
| [GhostLock Windows 包装分支](https://github.com/xiaohj233/ghostlock-x200-root) | 当次查验分支 | 记录兼容差异，不把同内核源码 target 当成 Windows 设备模块 |
| [BakaSU / 原 ReSukiSU](https://github.com/Baka-SU/BakaSU) | 35171 / v4.2.0-rc3 / 239e1e88… | 管理器与 LKM 版本核对 |
| [NeoZygisk](https://github.com/JingMatrix/NeoZygisk/releases/tag/v2.4) | 2.4 / 289 | 原提供者与最终回退版本 |
| [Vector](https://github.com/JingMatrix/Vector/releases/tag/v2.2) | 2.2 / 3080 / API102 | Xposed 基础框架 |
| [GKD](https://github.com/gkd-kit/gkd/releases/tag/v1.13.1) | 1.13.1 / 101 | 无障碍规则客户端 |
| [Lin-arm 规则库](https://github.com/Lin-arm/GKD_subscription/tree/396e8e6f6a1cbf16237a0ebabfc811b358d2c142) | 396e8e6f… / v632 | 在线规则来源；本仓库不镜像整个第三方规则库 |
| [AIsouler 旧规则库](https://github.com/AIsouler/GKD_subscription) | 当次已停止维护 / 归档 | 早期本地筛选来源，后改在线维护分支 |
| [Native Detector](https://github.com/reveny/Android-Native-Root-Detector/releases/tag/v7.7.0) | 7.7.0 | 本地默认与加强检测；未改动检测器 |
| [NyaZygisk](https://github.com/Nk-Be/NyaZygisk/releases/tag/v2.5) | 2.5 / 351 / c873f24 | 已试验、重复失败并移除 |
| [Zygisk Hide](https://github.com/kgr-online/Zygisk-Hide/releases/tag/3.1.0) | 3.1.0 | 下载和脚本检查；未安装实测 |
| [Zygisk Assistant](https://github.com/snake-4/Zygisk-Assistant/releases/tag/v2.1.4) | 2.1.4 | 候选调查，未安装 |
| [AOSP 动态分区](https://source.android.com/docs/core/ota/dynamic_partitions/implement) | Android 官方文档 | super 的概念边界 |
| [vivo 社区动作权限说明](https://bbs.vivo.com.cn/newbbs/thread/39027863) | 社区帖子，非本机验收替代品 | 查找系统原生入口；最终以本机界面计数核验 |

没有发布第三方 APK、ZIP、固件、完整规则库或漏洞实现；它们各自的许可证、商标与分发条件不由本仓库许可证替代。原创文字和脱敏摘要采用 CC BY-NC-SA 4.0；设备结构 / 偏移是版本限定的事实记录，不承诺适配与运行成功。
