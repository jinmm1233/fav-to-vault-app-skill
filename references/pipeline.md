# 七阶段流水线细节

## 目录
1. 阶段划分与状态机
2. 跳过规则（每阶段独立打标）
3. 分类体系与归档路径
4. 共识观点合并算法
5. 知识卡片汇总

## 1. 阶段划分与状态机

每条收藏在 SQLite 里有一个 status 字段，流水线按顺序推进，任何阶段失败都落回对应 stage：

```
discovered → downloaded → transcribed → extracted → written → (consensus merged)
```

- `discovered`：刚从收藏接口拉下来，还没处理
- `downloaded`：视频音频已下载到本地临时目录
- `transcribed`：Whisper 已产出文字稿
- `extracted`：LLM 已产出结构化知识点 JSON
- `written`：已写成 .md 落到 Obsidian vault，临时视频文件已删

失败分三类状态，不要混：
- `failed`：真失败（网络错、API 报错、源视频损坏），进重试队列
- `skipped`：判定为无用内容，永久跳过不再重试
- `media_frozen`：下载了但转不出来，标冻结

## 2. 跳过规则（每阶段独立打标）

在每个阶段结束时检查，命中即写 `status: "skipped"` + `stage: <当前阶段>` + `error: <原因>`，不再进入下一阶段：

| 阶段 | 跳过条件 | error 文案 |
|---|---|---|
| discovered | 内容类型识别为广告 | `contentType: "ad"` |
| discovered | 内容类型识别为短剧 | `contentType: "shortdrama"` |
| discovered | 下载后检测到缺音/视频流 | `media must contain complete audio and video streams` |
| media_frozen | 转录产出为 0 个有效文本段 | `transcription produced no usable text segments` |
| media_frozen | 转录时长覆盖不了媒体时间线 | `transcription does not cover enough of the media timeline` |
| transcribed | LLM 返回的知识 JSON 校验失败 | `knowledge field verificationNeeds[0].claim is empty` |

关键：不要在 discovered 阶段一刀切判断"这条视频有没有用"——只有真正下载转录了才知道它是不是纯音乐/广告。入口粗筛（广告/短剧 contentType）能挡掉一部分，剩下的靠每阶段打标。

## 3. 分类体系与归档路径

三级分类：大类 → 组 → 子类，路径即文件夹：
```
vault/Knowledge/<大类>/<组>/<子类>/<视频标题>.md
```

- 收藏夹映射为最大父类（用户在抖音里建的收藏夹名直接当顶层分类）
- LLM 根据笔记内容二次细分到组/子类
- 每个子类目录下放一个 `00-共识观点.md`，存跨视频反复印证的观点

笔记文件名 = 视频标题（清洗掉非法字符），不要用 aweme_id 当文件名——用户在 Obsidian 里要看得懂。

## 4. 共识观点合并算法

新笔记写盘后，拿它的观点列表和同子类下历史笔记的观点做匹配：
1. 每条新观点和历史所有观点算语义相似度（LLM 判断"这两条说的是不是一回事"）
2. 相似度高的配对记录 reason："均通过 X 方式达成 Y"
3. 被命中的历史观点更新引用次数，新观点追加到对应位置
4. 同子类累计 N 条以上命中后，合并写入 `00-共识观点.md`
5. 没匹配上的新观点留在笔记里，不强行归并

对比规模控制：同子类历史笔记全量比对（peersCompared 记录实际比对数），不要全库扫——又慢又费 token。

## 5. 知识卡片汇总

按大类聚合：每周/手动触发一次，把该大类下所有笔记的观点拉出来，让 LLM 重组成：
- 总览（这个大类在讲什么）
- 印证观点（跨多视频反复出现的结论）
- 思维导图（层级结构）
- 印证网络（观点 ↔ 视频 的二部图，连线标"印证/补充"）
- 分类速查（表格）
- 单视频知识（原始笔记索引）

印证网络视图用三列布局：观点左两列、视频右一列，贝塞尔曲线连线错开，节点标题超长加省略号+悬停显示全文，不要让字和线叠在一起。
