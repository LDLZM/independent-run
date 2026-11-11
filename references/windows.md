# Windows 实施说明

## 任务计划程序

优先使用 `ScheduledTasks` PowerShell 模块，由 Task Scheduler 服务创建和运行任务。先检查模块、调度服务状态和当前身份；遇到权限不足时明确报告，不静默切换到管理员或 SYSTEM。

对仅要求关闭聊天应用后继续的任务，优先当前用户 `Interactive` 登录类型、`Limited` 权限。不配置开机或登录触发器；注册无定时触发器的按需任务后调用 `Start-ScheduledTask` 即可。此方式要求用户保持登录，不承诺注销后继续。

确实要求注销后运行时，再选择受支持的非交互身份。密码登录不要通过聊天收集密码；S4U 对网络资源和加密文件访问有限制，不能盲目替代。用户要求开机或登录自动启动时，才配置对应触发器和身份。

### 动作与包装脚本

- 使用 `New-ScheduledTaskAction` 的 `Execute`、`Argument`、`WorkingDirectory` 明确指定路径。参数是 Windows 命令行文本，需要按目标程序和 shell 的规则正确引用；JSON 序列化不能替代命令行转义。
- 若需要重定向日志或设置环境，生成持久 PowerShell 包装脚本，动作执行 PowerShell 的绝对路径，使用 `-NoProfile -NonInteractive -WindowStyle Hidden -File`。不要无故放宽执行策略。
- 脚本以调用运算符和参数数组调用确切可执行文件并等待结束；分别重定向标准输出和错误到每次运行的日志。保存 `$LASTEXITCODE` 并用 `exit` 传回；脚本异常要记录并以非零码退出。避免再启动一个脱离包装脚本的后台进程，否则任务状态和停止操作无法可靠代表业务进程。
- 不要使用 `Start-Job` 作为跨宿主持久化机制。`Start-Process -WindowStyle Hidden` 只隐藏窗口，仍可能受宿主 Windows Job Object 清理影响。

### 设置

- 长期服务或明确需要无时限的任务，使用 `New-ScheduledTaskSettingsSet -ExecutionTimeLimit ([TimeSpan]::Zero)` 避免默认运行时限；有限任务根据预期时长设置合理时限。
- 通常采用 `-MultipleInstances IgnoreNew` 防止重复运行，除非项目明确要求并发。
- 检查电池供电相关默认设置。需要在电池上继续时，按要求设置 `-AllowStartIfOnBatteries` 和 `-DontStopIfGoingOnBatteries`；不要无关地变更电源计划或休眠策略。
- 不自动添加失败重试、每日调度或开机自启。已有同名配置必须检查归属，不用 `-Force` 覆盖未知任务。

## 状态与生命周期验证

注册后检查动作、身份和设置，再按需启动。等待合理的启动时间，从新的 shell 调用检查：

```powershell
# 替换为刚创建任务的精确名称和路径。
$taskName = '实际任务名称'
$taskPath = '\'
Get-ScheduledTask -TaskName $taskName -TaskPath $taskPath
Get-ScheduledTaskInfo -TaskName $taskName -TaskPath $taskPath
```

`Running` 只代表任务仍在执行；结合日志与服务健康检查。`LastTaskResult` 需要结合上次运行时间及状态解读，正在运行时不能将其当成最终退出码。一次性任务迅速变为 `Ready` 可能是正常完成，核对日志、退出码和产物。

进程身份诊断可使用 `Get-CimInstance Win32_Process` 的 `ProcessId`、`ParentProcessId`、`CreationDate`、`ExecutablePath` 和 `CommandLine`；输出时避免暴露参数中的秘密。任务管理器应用分组不能代替进程父链检查，父链也不能单独证明不存在 Job Object 约束。

## 停止与移除

优先使用应用提供的关闭接口。必要时用精确任务名和路径执行 `Stop-ScheduledTask`，然后检查端口、健康状态和进程，确认业务子进程也已退出；不要假设停止任务自动清理全部后代。

```powershell
Stop-ScheduledTask -TaskName $taskName -TaskPath $taskPath
# 用户要求撤销此独立运行配置时，停止并核对后再移除。
Unregister-ScheduledTask -TaskName $taskName -TaskPath $taskPath -Confirm:$false
```

如果任务有自动触发器，停止一次不等于禁用；按用户要求禁用或移除对应任务。保留计算结果和日志，除非用户要求删除。移除前仍需核对任务确由本次工作创建或属于用户指定对象。
