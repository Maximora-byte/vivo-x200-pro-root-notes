# DirtyFrag VR watchdog：先区分故障，再谈修复

## 版本与触发信息

设备操作者确认最近一次使用的是月虹的 **DirtyFrag VR**。当时安装包为月虹 v1.6.8（versionCode 168），管理器为 ReSukiSU v4.2.0-rc3 / 35171，内核日志中的驱动也为对应版本，LKM 模式。进程名 `libksud.so` 的归属判断结合了管理器初始化和应用 UID 映射，而不是只凭文件名。

## 最近一次清晰的错误链

见 [脱敏摘录](../evidence/watchdog-excerpt.txt)。大约在开机第 69 秒，日志出现 DirtyFrag 进入、管理器初始化、SELinux 恢复 Enforcing，紧接着发生：

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

## 前一次是另一条路径

前一次保留记录中有 `0xa8` NULL 指针访问，PC 为 `rt_mutex_setprio+0x2a0/0x4e8`，进程名 `id`，涉及 `rt_mutex_adjust_prio_chain`、`__sched_setscheduler` 和 `sched_setattr`。它与后一次页表锁 / RCU stall 记录分别保留，没有合并成同一个已经证实的原因。

## 实际采取的候选处理

月虹 DirtyFrag VR 页面上：

- 将“激活时去除 VR 探针”由关闭改为开启；
- 保留“从当前内核 BTF 自动获取偏移”；
- 没有手工搬用别的内核偏移，没有关闭 watchdog。

当时的采集报告只记录设置变化，没有执行漏洞。安装包引擎中的运行内核 BTF、偏移解析失败、探针去除失败时在管理器启动前退出等字符串，支持它包含这条候选路径，但不证明该路径已经执行成功。

操作者后来再次运行，后续会话确实拿到 Root，并完成备份和框架工作。这个后续观察比“只改了开关”更强，但仍缺少同条件多次 A/B、每次运行终端和两个回调的精确符号归属；所以本文把它称为**候选规避后观察到 Root 恢复**，没有发布“watchdog 永久修复补丁”。

对应管理器源码线索：[固定提交的 syscall_hook_manager.c](https://github.com/Baka-SU/BakaSU/blob/239e1e8871b8fcd51a6e5b3002e0ba522fdd99fb/kernel/hook/syscall_hook_manager.c)。其中的 sys_enter 注册逻辑不能直接替代本机 sys_exit 两个地址的符号证据。

完整 pstore、DropBox、recovery last_kmsg 和 bugreport 仅在本地保存；公开仓库只给最小错误链。
