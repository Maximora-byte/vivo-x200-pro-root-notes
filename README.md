# vivo X200 Pro 临时 Root 实录

从 ADB 连接、GhostLock 配置、DirtyFrag VR watchdog、系统备份，到 NeoZygisk / Vector、GKD 和 Root 隐藏对照实验的完整记录。

**记录时间：2026-11-12—2026-11-13（Asia/Shanghai）。这是单台设备的经验档案，不是官方适配声明或“一键 Root 成功教程”。**

## 最终结论

| 项目 | 这次实际确认到哪里 |
| --- | --- |
| 设备 | vivo X200 Pro，V2405A / PD2405M，MT6991，Android 15 / OriginOS 5 |
| 采集系统 | `PD2405B_A_15.0.27.0.W10.V000L1`，当前 B 槽 |
| 内核 | `6.6.57-android15-8-ge9c3d7352454-abogki428077543-4k` |
| Root 来源 | 设备操作者运行月虹 v1.6.8 的 DirtyFrag VR；后续会话验证到真实 Root |
| GhostLock | 完成当前镜像提取、参数交叉核验和导入；**没有执行本配置的 GhostLock 提权测试** |
| watchdog | 定位了 atomic / 页表锁 / RCU stall 错误链；VR 回调是嫌疑，未完成符号映射定责 |
| 备份 | 108 份分区或小型 UFS 镜像完成；super 长流失败后改分块续传并全量校验 |
| LSPosed 基础 | NeoZygisk 2.4 + Vector 2.2 / API 102 激活；没有验证具体功能模块 |
| 去广告 | 快应用与多项推荐关闭；GKD 在线订阅配置、服务及规则加载核验通过，未取得真实广告跳过命中证据 |
| 摇一摇 | 53 个应用设为“仅开屏时禁止”，范围是每次启动前 6 秒 |
| 隐藏试验 | NyaZygisk 2.5 曾通过加强检测，但重复未稳定复现；**最终回退 NeoZygisk 2.4** |
| 最后状态 | Root 保留、SELinux Enforcing、Vector 已激活、GKD 正常；未写底层分区、未整机重启 |

## 阅读顺序

1. [完整实录](docs/full-record.md)——博客同源正文，覆盖全部阶段。
2. [时间线与证据分级](docs/timeline.md)——早期结论与最终状态分开。
3. [watchdog 分析](docs/watchdog.md)——两次崩溃不能合并归因。
4. [GhostLock 配置演进](docs/ghostlock.md)——同内核字符串、可编译、镜像核验、运行成功是四回事。
5. [备份与断点续传](docs/backup.md)——super、GPT、两槽与设备唯一数据的边界。
6. [NeoZygisk / Vector 安装与激活](docs/modules.md)。
7. [系统推荐、快应用、摇一摇与 GKD](docs/ad-cleaning.md)。
8. [Root 隐藏 A/B 与回退](docs/hiding-trial.md)。
9. [只读验证命令与复核清单](docs/verification.md)。
10. [公开材料和隐私边界](docs/evidence-and-privacy.md)。

机器可读摘要在 [evidence/](evidence/)，最终镜像核验过但未执行提权的配置在 [profiles/](profiles/)。

博客版本：<https://blog.maximoraverse.org/posts/vivo-x200-pro-temporary-root-field-notes/>。

## 如何使用这份记录

先核对完整系统构建、内核、槽位和镜像，而不是只看“X200 系列”或 Android 大版本。不同版本的偏移、模块、Root 管理器和激活时机不能混用。本文没有把降低系统版本、执行漏洞或刷回备份写成推荐操作；已有事实与未验证部分在各章单列。

资料由设备操作者与 Codex 协作整理，依据当次会话和本地留存的采集、校验与回退记录。Root 再次运行由操作者完成；后续文件采集、配置和多数工具检查通过 ADB 完成。全文经脱敏，不发布生产服务器访问资料、设备唯一标识、完整 bugreport、私有授权表或设备镜像。

原创文档采用 [CC BY-NC-SA 4.0](LICENSE)。第三方项目及数据来源见 [来源与归属](docs/sources.md)。
