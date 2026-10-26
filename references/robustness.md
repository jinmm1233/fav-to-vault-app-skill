# 健壮性机制

## 目录
1. 死锁（sync.lock）处理
2. 双轨进度条设计
3. Windows 计划任务配置（完整脚本）
4. 常见错误码对照
5. AI 诊断日志

## 1. 死锁（sync.lock）处理

同步进程启动时在 `var/sync.lock` 写 `{pid, createdAt}`。进程被硬杀（电脑睡眠/关机/断电）就会留下死锁，下次启动看到锁就以为在跑、进度条卡死。

三层防护：

**第一层：脚本启动自检**
```powershell
# run-sync.ps1 开头
if (Test-Path var\sync.lock) {
  $lock = Get-Content var\sync.lock | ConvertFrom-Json
  $ageMin = ((Get-Date) - $lock.createdAt).TotalMinutes
  if ($ageMin -gt 360) {  # 6 小时
    Remove-Item var\sync.lock
    Log "STALE-LOCK-CLEARED age=$ageMin min"
  }
}
```

**第二层：复位前 pid 探活**
用户点"一键复位"时，主进程读 lock 里的 pid：
```js
try { process.kill(pid, 0); return { ok: false, reason: "running" }; }
catch { /* pid 已死，安全复位 */ }
```
活着就拒绝复位并提示"同步似乎还在跑"，防误杀。

**第三层：界面一键复位按钮**
告警区出现"检测到同步残留锁"时，文案后面跟一个按钮：
- 点击 → 主进程删 lock + 把 sync-progress.json 重置为 `{running:false, done:0, total:0}`
- 成功后 toast 提示 + 自动刷新页面
- 写一行 STUCK-RESET 进 scheduler.log

不要让普通用户翻文件夹手动删文件。

## 2. 双轨进度条设计

两个进度条必须独立，文案要写清楚各自分母：

- **总览大进度条**：按"处理窗口"算。未设置起始日期/条数 = 全部收藏；设了 = 窗口内收藏。这是"首次处理收藏"的整体进度。
- **今日任务进度条**：今天这次定时任务的实时进度（跑完=100%），和大进度条无关。每日范围 = 昨日新增 + 昨日失败重试，随日期滑动。

用户最容易困惑的地方就是这两个为什么不一致——界面文案要把这个解释写出来。

## 3. Windows 计划任务配置（完整脚本）

```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\path\run-sync.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 8:00am
$settings = New-ScheduledTaskSettingsSet `
  -WakeToRun `                    # 睡眠中到点唤醒
  -StartWhenAvailable `            # 错过后开机补跑
  -RestartCount 3 `               # 失败自动重启 3 次
  -RestartInterval (New-TimeSpan -Minutes 15) `
  -DisallowStartIfOnBatteries:$false `
  -StopIfGoingOnBatteries:$false `
  -ExecutionTimeLimit (New-TimeSpan -Hours 2) `
  -MultipleInstances IgnoreNew    # 已在跑就忽略新实例
Register-ScheduledTask -TaskName "Your App Sync" -Action $action -Trigger $trigger -Settings $settings
```

坑：`RestartInterval` 和 `ExecutionTimeLimit` 在某些 PowerShell 版本里传 TimeSpan 对象会报 0x80041318，直接传 ISO8601 字符串 `"PT15M"` / `"PT2H"` 更稳。

## 4. 常见错误码对照

| 错误码 | 含义 | 处理 |
|---|---|---|
| 0 | 成功 | — |
| 1073807364 (0x40010004) | 任务被强制终止（电脑睡眠/关机杀进程） | 配 WakeToRun + RestartCount，死锁自愈兜底 |
| 0x80041318 | 计划任务参数格式错 | 时间相关参数传 ISO8601 字符串 |

## 5. AI 诊断日志

定时任务失败后，不要只给用户一个错误码。run-sync.ps1 在失败时自动跑一段 AI 诊断：读最近日志 tail，让 LLM 总结"这次为什么失败、下一步怎么修"，用大白话写进 scheduler.log 和界面告警区。

例：
```
AI 诊断：进程在 08:10 建锁后被杀，对应时间点电脑进入睡眠。
建议：已开启 WakeToRun，下次睡眠中会自动唤醒；若仍频繁失败，检查电源设置。
```
用户能看懂，不用自己翻日志。
