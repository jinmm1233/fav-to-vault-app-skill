---
name: fav-to-vault-app
description: 把"收藏夹自动整理成本地知识库"这类桌面应用从零做出来的完整产品蓝图与机制清单。当用户想做一个把抖音/B站/小红书/YouTube 收藏的视频或图文，自动转录、AI 提炼、分类归档成 Obsidian/Markdown 知识库的桌面软件（Electron 或同类壳），或要给这类应用加定时同步、失败重试、死锁自愈等机制时使用。涵盖七阶段流水线、跳过无用内容规则、Windows 计划任务加固、UI 交互设计与打包分发经验。
---

# 收藏夹 → 本地知识库 桌面应用蓝图

本 Skill 沉淀了一个已上线的"抖音收藏 → Obsidian 知识库"Electron 应用的全部产品决策与机制。目标是让另一个 AI agent 拿到后，能直接照着重做出一个同类软件，而不是从零试错。

## 产品定位

一句话：用户在短视频平台收藏的内容，本来就躺在他自己账号里；软件只是替他把这些收藏批量搬到本地、转成文字、用 AI 提炼成结构化笔记，按分类归档进一个纯 Markdown 的知识库（Obsidian vault）。

三条不可破的边界：
1. 只处理用户自己账号的收藏，用用户自己的登录态，不爬别人的数据。
2. 所有 AI 提炼在用户自己的电脑或用户自己的 API Key 下跑，不碰用户账号密码。
3. 知识库是纯 .md 文件，用户随时可以用任何编辑器打开，不被软件锁死。

## 核心流水线（七阶段）

每条收藏视频按固定管线走：采集列表 → 下载音视频 → Whisper 本地转录 → LLM 提炼知识点 → 分类归档写 .md → 跨视频共识观点合并 → 汇总成知识卡片。

详细字段、命令、跳过规则见 [references/pipeline.md](references/pipeline.md)。

## 关键机制（这是别人抄不走的部分）

### 1. 自动跳过无用收藏（核心体验）
收藏夹里混着广告、短剧、纯音乐、无对白视频，不能硬跑浪费 API 额度。流水线每一步都有"该跳过"的判定，命中即标记 skipped 不再重试：
- 源视频本身缺音视频流（"media must contain complete audio and video streams"）
- Whisper 转录出来没有可用文本段（纯音乐/画面视频）
- 转录时长覆盖不了媒体时间线（视频被静音/损坏）
- LLM 返回的知识字段校验不通过（claim 为空等）
- 内容类型被识别为广告 / 短剧（shortdrama）
这些规则要在每阶段独立打标，不要在入口一刀切——入口判断不了"这条视频其实没说话"。

### 2. 处理窗口三模式（首次配置一次定终身）
新用户第一次打开必须让他选，否则会把几百条历史收藏全跑一遍烧掉 API：
- **按时间**：只处理视频发布时间在起始日期之后的收藏（平台不给精确收藏时刻，只能用发布时间近似）
- **按数量**：从当前收藏列表末尾数 N 条，保存时锁定快照，之后新收藏不计入这 N 条
- **处理全部**：历史存量全跑，文案必须警告"可能耗费较长时间"
这三个模式决定总览大进度条的分母，和"每日新增"是两套独立进度条，不要混。

### 3. 死锁自愈与一键复位
同步进程被硬杀（电脑睡眠/关机）会留下 var/sync.lock，下次启动看到锁就以为在跑、进度条卡死。三层防护：
- 脚本启动时检测锁年龄 >6 小时视为死锁，自动清
- 锁文件里写 pid，复位前 `process.kill(pid, 0)` 探活，活着就拒绝复位（防误杀正在跑的同步）
- 界面告警区放"一键复位"按钮，普通用户不用翻文件夹删文件
详细见 [references/robustness.md](references/robustness.md)。

### 4. 失败重试窗口
拉取/处理失败不要立刻放弃，也不要无限重试刷爆 API：
- 单条失败 30 秒后重试一次
- 失败条目进入 pending 队列，3 天内的下次定时任务自动补拉
- 超过 3 天不再补，进止损看板让用户手动决定
- 定时任务每天范围 = 昨日新增 + 昨日失败重试，随日期滑动

### 5. Windows 计划任务加固
用户电脑 8 点多半在睡觉，计划任务必须这么配：
- `WakeToRun=True`：睡眠中到点自动唤醒
- `StartWhenAvailable=True`：8 点没开机，开机后自动补跑
- `RestartCount=3` + `RestartInterval=PT15M`：被硬杀后系统级自动重启最多 3 次
- `DisallowStartIfOnBatteries=False` + `StopIfGoingOnBatteries=False`：电池供电也跑
- `ExecutionTimeLimit=PT2H`：防任务挂死占资源
- 注意 PowerShell 里后两个参数必须传 ISO8601 字符串，传 TimeSpan 对象会报 0x80041318

### 6. 首次打开引导
新用户第一次启动弹一次居中卡片，只说两件事：去设置选好处理窗口（时间/数量/全部）、填自己的 LLM API Key（不填只能转录出不了笔记）。点"去设置"直接跳页，点"我知道了"关掉，记 `firstRunWelcomeShown` 永久不再弹。

## 技术选型建议

- 壳：Electron（Windows 桌面，打包成 NSIS 安装包 + 便携版）
- 采集：Python 脚本调平台官方"我的收藏"接口（用户登录态 cookie），输出 JSON 清单
- 转录：faster-whisper-small 本地跑，不上传音频
- 提炼：OpenAI 兼容接口（DeepSeek 等），用户自己填 Key
- 知识库：纯 Markdown 文件落 Obsidian vault，软件只是读写 .md，不建私有数据库
- 定时：Windows 计划任务跑 PowerShell 脚本，不做应用内常驻守护进程

## 分阶段交付路线

1. MVP：手动一键处理 1 条 → 跑通采集/下载/转录/提炼/写 md 全链路
2. 批量：一次 N 条 + 进度条 + 失败重试
3. 定时：计划任务每日自动跑 + 错过补跑
4. 知识库 UI：分类树浏览、全文搜索、笔记编辑
5. 增值：共识观点、知识卡片、语义检索、每周总结报告
6. 分发：electron-builder 打包，内置 Python/Whisper runtime 做到开箱即用

## 参考文件索引

- [references/pipeline.md](references/pipeline.md)：七阶段流水线字段、跳过规则、分类体系、共识观点合并算法
- [references/robustness.md](references/robustness.md)：死锁处理、进度条双轨、计划任务配置、错误码含义
- [references/pitfalls.md](references/pitfalls.md)：Windows 路径/编码/打包/杀软误报等真实踩坑
