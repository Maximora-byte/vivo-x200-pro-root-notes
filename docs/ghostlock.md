# GhostLock 配置：从源码候选到实机镜像核验

## 一开始没有对应完整固件

系统已经变为 15.0.27.0，原先其他构建 / b57 的数据不能沿用。查找本地降级包、boot 镜像和降级工具缓存没有取得可核验的完整固件，因此先做了明确标记 `verified=false` 的上游源码 target 候选。

来源是 CyberMeowfia 固定提交 `c87b8664c8d910a68c132dcb940696a95ea8f527` 的 PD2405 target。完整内核字符串一致，但上游用户空间构建标签与本机不同。NDK r28 / AArch64 / API 35 的语法检查和共享库链接成功，Windows 包装分支却仍缺 `SLIDE_INIT_CRED_OFF`，且缺少它额外要求的窗口数据。

**因此“能编译”没有被写成“Windows 一键工具已支持”。** 也没有把保留未验证标记的候选扔进自动匹配目录。上游实现和 Windows 包装流程的行为与停止阶段并不相同。

## 获得 Root 后，改用当前 B 槽镜像

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

[配置](../profiles/PD2405-6.6.57-from-device.conf)和[核验摘要](../evidence/ghostlock-validation.json)保留在仓库。它们只对应文中精确版本与镜像，不能按机型系列、内核主版本或 Android 版本通用套用。

## 导入结果与停止边界

通过 GhostLock 的导入入口加载后，App 私有目录保存的配置 SHA256 与电脑文件一致；缺失参数提示消失。公开仓库的文本副本统一为 LF 换行，因此另列 public_copy_sha256；原始导入收据中的 configuration_sha256 仍指原电脑 / 手机文件，不能混用这两个文件哈希。界面又提出 Shizuku 前置要求，但本轮没有据此继续执行 GhostLock。

完成的是：**提取 → 参数核验 → 导入和读回**。没有执行本配置的 GhostLock 提权，也没有开启强制攻击测试。当前真实 Root 来自之前的 DirtyFrag VR，不能拿它给 GhostLock 配置背书。

仓库不复制第三方 `root.c`、二进制 payload 或完整固件，源码依据以固定提交链接保留，第三方授权见[来源](sources.md)。
