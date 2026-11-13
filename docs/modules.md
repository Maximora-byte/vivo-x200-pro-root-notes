# NeoZygisk / Vector：安装与激活分开验收

## 名称和职责

- ReSukiSU 是当前临时 Root 的管理 / 用户空间体系，对应内核驱动版本也需要匹配。
- NeoZygisk 是 Zygote 注入与 Zygisk API 提供者，不等于一个 Xposed 功能模块。
- Vector 提供 Xposed 兼容框架；具体功能模块还需要另装、启用和设置作用域。

本轮最终保留 NeoZygisk v2.4 build289、Vector v2.2 build3080 / API102，没有安装具体 Xposed 功能模块。

## 安装阶段

从官方 GitHub Releases 取得 ZIP，核对发布摘要和手机端副本，ZIP 自检通过，先备份原 /data/adb。安装器成功时文件仍位于 `modules_update`，Zygote 与 system_server PID 没变，因此记录为“安装成功、尚待激活”。

NeoZygisk 对 ReSukiSU 35171 输出超过其充分测试版本范围的警告；没有把安装器未中止当成兼容性保证。

## 临时 Root 的激活方式

操作者明确选择模拟软重启，而不是整机重启。通过 ReSukiSU 的 `soft-reboot` 重建 Android 框架后：

- 内核启动标识不变，uptime 连续；
- Zygote、system_server PID 更新，`sys.boot_completed=1`；
- ADB 仍为 uid=0，SELinux Enforcing；
- 提供者报告 monitor tracing / zygote injected / daemon running；
- Vector CLI IPC 返回成功，API102，日志确认系统服务注入；
- 模块清单 enabled=true / update=false；
- 后续回退时又核验管理器显示“已激活”，不能只依赖 CLI。

**模拟软重启仍会关闭应用，不是无感设置。** 当前 Root 是临时 LKM，不能把普通整机重启当作同一操作。本文只描述已获同意的这次操作，没有建议陌生版本直接重放命令。

## 留下的诊断信息

初次激活日志包含两次 init-group ioctl EPERM，以及反射 / Trace / SELinux 探测诊断。之后基础注入成功，并未关闭 Enforcing 来抹平日志。这只能证明当次基础框架可工作，不能证明每个 LSPosed 模块或长期稳定性。

## 后来隐藏试验里的重要反例

NyaZygisk 替换期间，模拟软重启留下旧 Root 守护进程，新旧 Vector 并存时新进程崩溃。随后出现过“CLI status 成功，但管理器仍异常”。清理遗留进程并干净启动后管理器才激活。

因此验收应同时覆盖进程唯一性、注入、真实 IPC、管理器和目标功能；单个绿色状态不够。试验中的额外 core 只在保留诊断日志后清理，没有清应用数据。
