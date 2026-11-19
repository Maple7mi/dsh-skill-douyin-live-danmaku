---
name: douyin-live-danmaku
description: 采集抖音直播间全量弹幕并做成桌面助手（Aurora Dark 风格的 Tkinter GUI）：解析 WebcastChat/Member/Gift/EmojiChat 消息、写 14 列 CSV、渲染等级徽章与抖音表情、按天生成静态归档站点、PyInstaller 打成单文件 exe。用于「做一个抖音弹幕采集器 / 弹幕助手 / 直播数据看板」，或排查这类工具收不到弹幕、徽章不显示、表情变方框、exe 打包后资源丢失等问题。
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
metadata:
  trigger: 抖音弹幕采集、弹幕助手、直播间数据看板、Webcast 协议解析、Aurora Dark 风格弹幕客户端
  source: 基于 saermart/DouyinLiveWebFetcher 的二次开发实践整理
  revision: "2026-11-19"
---

# 抖音直播弹幕助手

把「抖音直播间弹幕采集 + 桌面看板 + 单文件 exe」这套东西完整做出来，并且知道每一步会踩什么坑。
参考实现：`Maple7mi/DouyinLiveWebFetcher`（上游 `saermart/DouyinLiveWebFetcher`）。

## 什么时候用

- 要从抖音直播间抓全量弹幕（含 user_id、粉丝团等级、消费等级、内容）
- 要做直播数据看板：在线人数、本场涨粉、付费人数、礼物总额、礼物流水
- 要给弹幕加等级徽章、抖音表情、房管/星守护标识
- 要把采集数据按直播日归档成静态站点（礼物榜、CSV 导出、删除某天）
- 要打包成 Windows 单文件 exe 分发给主播/房管

## 整体结构

```
danmaku_gui.py      WebSocket 连接 + protobuf 解析 + CSV/JSONL 落盘 + 事件队列
liveMan.py          拉房间 enter 信息、调 JS 算 signature、拼 wss 地址
protobuf/douyin.py  python-betterproto 生成的消息类（勿手改）
aurora_gui.py       纯 Tkinter 界面（Canvas 虚拟列表、自绘卡片、徽章/表情渲染）
site_builder.py     按直播日聚合 → 静态站点（index/day/gift 榜）
site_server.py      本地 http 起站点预览
browser_login.py    拉起 Edge 扫码登录，落 cookie.txt
DouyinDanmaku.spec  PyInstaller 单文件打包配置
```

数据流：`ws → PushFrame → Response → messages_list（method + payload）→ 分发解析 → 事件 dict → GUI 队列 + CSV/JSONL`。

## 关键步骤

### 1. 连接

- `wss://webcast5-ws-web-lf.douyin.com/webcast/im/push/v2/?...`，signature 由 `liveMan.py` 用 `mini_racer` 跑抖音前端 JS 得到
- 首次连接会收到消息帧，**必须回 ack**（`internal_ext` 从 `Response.internal_ext` 取），否则秒断
- 心跳用 `PushFrame` 每隔 10s 左右发一次，别太频繁
- cookie 从 `cookie.txt` 读（`utf-8-sig`，去 BOM），未登录时礼物类消息不推

### 2. 解析

- 运行时是 **betterproto 2.0.0b6**：未设置的 message 字段是 `None`，repeated 字段是 `[]` —— 所有嵌套访问都要 `getattr(x, "y", None)` 兜底，别写 `msg.user.fans_club.data`
- 只用 `WebcastChatMessage` / `WebcastMemberMessage` / `WebcastGiftMessage` / `WebcastEmojiChatMessage` / `WebcastLikeMessage` / `WebcastRoomStatsMessage` / `WebcastRoomUserSeqMessage`
- 白名单外的方法名先落 `logs/methods.json` 计数，再决定要不要建模（`WebcastLowPcuGuideChatMessage`、`WebcastMysteryManMessage` 这类容易漏）
- 弹幕里 `[看]` 这种是**普通文本**，不是 `WebcastEmojiChatMessage`；真正的表情消息是另一个 method

### 3. 落盘

CSV 固定 14 列，别改顺序，站点聚合依赖它：

```
时间, kind, msg_id, user_id, nickname, fans_club_level, pay_level,
content, gift_name, gift_count, diamonds, group_id, visible, session_date
```

- `kind`：`chat` / `entry` / `like` / `gift`
- 礼物按 `group_key` 合并计数，别一条一条算
- 每次写都 `flush()`；进程被关时也要能读到完整历史

### 4. 界面

- 纯 Tkinter（不引 PyQt，单文件体积友好）；暗色卡片用 `Frame` + `highlightbackground`
- 长列表用 `Canvas + create_window(Frame)` 假虚拟列表；窗口尺寸变化时**重算 `wraplength` 并重建行**，否则换行会错乱
- 一行弹幕的排法：`徽章图片列 → 加粗昵称 → 「：」→ 内容 → 表情图`，所有图标高度**等于正文字号高度**（18pt ≈ 24px），否则行高会被图标撑高、下一行被裁
- 文字里混表情时，用 PIL 把「文字 + 表情图」预渲染成一张 PNG 再塞进 `Label(image=...)`，比逐字符 `Text` 控件省事且对齐可控

### 5. 打包

```powershell
.\.venv\Scripts\python.exe -m PyInstaller --noconfirm --clean --distpath dist_x --workpath build_x DouyinDanmaku.spec
```

- `.spec` 的 `datas` 必须带 `assets/`，否则 exe 里没有徽章和表情
- 资源路径用 `sys._MEIPASS` 兜底：`resource_dir()`
- 单文件 exe 大约 55 MB（含 Pillow + 表情包）
- **测试版与稳定版必须隔离**：不同目录、不同 mutex 名、各自的 `logs/`、`cookie.txt`

## 参考

- `references/assets-and-rendering.md`：徽章素材 URL、裁切/染色、表情表来源与内联渲染
- `references/pitfalls.md`：协议、Windows、编码、打包上的坑
- `references/acceptance.md`：交付前逐条验收清单

## 硬规矩

- 改界面先出预览图，对方点头再动代码
- 同一张图只读一次（看图很贵），要对比就自己拼一张对比图
- 验证日志/文件**读内容**，不看文件大小（Windows 目录项对正在写的句柄是懒更新）
- 稳定版 exe 不许被覆盖；新版本先跑测试目录
