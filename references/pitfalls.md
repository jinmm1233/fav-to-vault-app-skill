# 真实踩坑记录

## 目录
1. Windows 路径与编码
2. Electron 打包
3. 杀软与 SmartScreen
4. 跨用户分发
5. 抖音平台相关

## 1. Windows 路径与编码

- Obsidian vault 路径含中文时，Node 的某些库（SQLite 驱动、子进程参数）会按 ANSI 码页解码导致乱码。日志里看到 `D:\鐭ヨ瘑搴揬\...` 就是这个问题——统一用 UTF-8 读写，传路径给子进程时走 `--args` 数组不走字符串拼接。
- PowerShell `ConvertFrom-Json` 遇到中文路径/中文内容会报"无法识别的转义序列"——读配置验证时直接 Read 原文，别用 ConvertFrom-Json。
- `Get-Content` 默认按 ANSI 读无 BOM 文件，写配置文件用 `[IO.File]::WriteAllText` 保留原换行，别用 `Out-File`（它会把 CRLF 改掉）。

## 2. Electron 打包

- renderer 文件打进 app.asar，改了源码必须重新 electron-builder，直接重启 win-unpacked 的 exe 看不到改动。
- 打包要内置 Python runtime、Node runtime、Whisper 模型、蒸馏引擎目录，做到用户开箱即用——否则用户电脑没装 Python，点"开始处理收藏"直接报错。
- 用 npmmirror 镜像加速：`ELECTRON_MIRROR` 和 `ELECTRON_BUILDER_BINARIES_MIRROR` 都设成 `https://npmmirror.com/mirrors/electron/`，国内打包快十倍。
- 每次打包前先 `Get-Process` 杀掉正在跑的 exe，否则文件占用打包失败。

## 3. 杀软与 SmartScreen

- 发用户 `.exe` 安装包，不要发 `win-unpacked` 散文件文件夹——散文件几百个小 exe/dll，杀毒软件极易误报。
- 没买代码签名证书，用户第一次跑会弹 SmartScreen"未知发布者"蓝色提示，让用户点"更多信息 → 仍要运行"。发给用户时提前说一声，别让他们以为是病毒。
- 加安装向导、桌面快捷方式、开始菜单入口，专业感直接提升一档。

## 4. 跨用户分发

- 用户配置存在 `%APPDATA%\.douyin-knowledge-app\settings.json`，不在软件目录里——软件升级/重装不会丢配置。
- API Key 让用户自己填自己的，不要内置你的 LLM Key——一内置就被白嫖光。
- 别人下载后纯浏览（知识库、图谱、语义检索）不需要 Python/Node；但"开始处理收藏"必须本地有运行时，所以要打包进去，或者在设置页让高级用户自己填路径。

## 5. 抖音平台相关

- 抖音不提供精确收藏时刻，只有视频发布时间。处理窗口用"视频发布时间之后"近似"收藏之后"，这个口径要在设置页跟用户讲清楚。
- 登录态 cookie 会过期，过期后采集拉到空列表——设置页要显眼放"打开抖音扫码登录"按钮，未登录时点"开始处理收藏"弹窗引导。
- 收藏夹接口能拉到用户自建收藏夹名，直接映射成顶层分类，比让用户手动建分类树好用。
- 失败重试只给一次机会（30s 后重试一次 + 3 天窗口补拉），不要做无限重试——抖音接口限流，死循环刷 API 会封号。
