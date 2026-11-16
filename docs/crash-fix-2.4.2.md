# 2.4.2：macOS 27.2 连接状态日志崩溃修复

## 原因

Snug 2.4.0 在 macOS 27.2（26B5091g，Apple 芯片）上的 10 份崩溃报告均为
`EXC_BAD_ACCESS / SIGSEGV`，程序计数器为 0，调用方为
`ConnectionTracker.log(_:_:now:) + 336`。运行时间约 49–218 秒。

已安装二进制的 arm64 指令显示，返回地址 `0x1000cec08` 前一条指令调用
`Foundation.Date.<=`，对应源码 `now <= quietUntil`。这个入口是弱链接符号
`$s10Foundation4DateV2leoiySbAC_ACtFZ`：二进制在系统版本至少 27.2 时调用它，
但受影响系统上的 `dlsym` 查找返回空地址。设置过静默期后，连入、换控制端或断开
需要再次写日志时就可能触发；不是固定时长的退出计时器。

修复改为 `quietUntil.timeIntervalSince(now) >= 0`，保留截止时刻仍处于静默期的
语义，避免依赖该日期比较入口。连接判断、防抖时长及屏幕行为不变。

## 验证

- `ConnectionTrackerTests` 覆盖静默期内、截止时刻、过期后的控制端变化和断开。
- GitHub Actions 执行 `scripts/snug check`、`swift test -c release`，并构建
  arm64 / x86_64 通用 App。
- 对最终二进制检查导入表，拒绝引入受影响的 `Date <=` / `Date >=` 符号。
- 本机 Command Line Tools 缺少 `Testing` 模块，完整测试由 Actions 执行。
  本机较新 SDK 重新编译的原代码未重现崩溃；根因依据来自实际崩溃报告、
  原安装包指令与系统符号查询，不能把重新编译后的通过误称为原版本通过。

## 安装与后续开发

本 fork 的修复包发布在 https://github.com/Caun1112/snug/releases 。
Actions 产物 `Snug-2.4.2.zip` 为 ad-hoc 签名，未做 Apple 公证。
替换 App 后若辅助功能权限失效，请在「系统设置 → 隐私与安全性 → 辅助功能」
重新添加 Snug。保留原 App 备份即可回退。

源码中的应用内更新源仍指向上游；此 fork 的修复版请从自己的 Releases 手动更新。
上游的 `scripts/snug ship` 含原作者的发布目标和公证配置，不能直接用于本 fork 发版。
