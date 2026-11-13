# 只读验证命令与复核清单

本轮使用 Android SDK platform-tools 37.0.1，并在独立 ADB server 端口 5038 上操作；正文命令用通用写法，读者需按自己的服务端加 `-P`。Windows 能枚举手机而 ADB 列表为空时，仍要核对授权、驱动、服务端和 USB 连接。改过连接方式不等于已证明断流的唯一根因。

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

## 验收表

- [ ] 完整系统、内核、槽位和镜像一致性都核对过。
- [ ] 候选、已导入、已安装、已激活、功能工作分别记录。
- [ ] 崩溃记录保留时间、开机单调时间、来源与归因边界。
- [ ] 备份通过长度、格式完整性和源 / 目标哈希，而不只是传输进度。
- [ ] 隐藏测试开关一致，检测器没有被授 Root，冷启动过程与共存模块状态可复核。
- [ ] 新方案重复失败时能够退回已知配置，且回退后的实际运行状态重新验证。
- [ ] 图中没有联系人、通知正文、账号、序列号、真实启动标识和目录凭据。
- [ ] 未测试的登录、支付、交易、云端完整性认证、恢复和长期稳定性没有写成通过。
