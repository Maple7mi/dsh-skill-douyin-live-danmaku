# 容易踩的坑

## 协议 / 解析

- **betterproto 未设置字段为 `None`**：`msg.user.fans_club.data.badge.icons` 这种链式访问必崩，全程 `getattr` 兜底
- **回 ack 必须**：不回 `internal_ext`，连接几秒内被断，表现为「连上了但一条弹幕没有」
- **未建模消息会静默丢失**：先把 `method` 计数落盘（`logs/methods.json`），再决定建模；抖音偶尔上新的 `Webcast*` 消息
- **`emoji_content` 可能是空**：不要据此判断「没表情」，去扫原始 payload 里的图片 URL
- **同一 msg_id 会重复下发**：`_once(msg_id)` 去重
- **`content` 为空**：有些消息只有表情，别直接丢弃

## Windows / 文件

- **正在被写句柄的文件，目录项 size 是懒更新**：`Get-ChildItem` 显示 0 B，实际内容早已写入。验证只读内容：`Get-Content`、`FileStream` 共享读、`[System.IO.FileInfo]::new(p).Length` 也未必准
- **PowerShell 的 `Get-Content -Raw` + `Set-Content` 会破坏 UTF-8 中文**：中文源码/脚本请用 Python 以 `encoding="utf-8"` 读写，或 `-Encoding UTF8`
- **控制台打印中文乱码**只是控制台 codepage 问题，不代表文件坏了；用 `repr()` 或写文件核对
- 单实例 mutex：测试版要换一个名字，否则和正式版互斥

## 界面

- **行高被图标撑高**：图标高度必须等于正文字号高度，`pady` 统一；否则换行行会被下一行压住
- **`wraplength` 不会自动跟着窗口变**：`<Configure>` 里重算并重建，或至少 `configure(wraplength=...)`
- **`Canvas` 里放 `Frame` 要做两件事**：`itemconfigure(width=canvas.winfo_width())` + `<Configure>` 时更新 `scrollregion`
- **去掉滚动条后别把滚动也去掉**：保留 `<MouseWheel>`，再加 `<ButtonPress-1>`/`<B1-Motion>` 拖动（`bind_all` 记得在 `<ButtonRelease-1>` 里解绑）
- **礼物列表要留历史**：只保留 8 条会让「滚回更早」根本不可能；行高固定为 `卡片高 / 8`，列表本身保留上百条，只有视图贴在底部时才自动滚到最新

## 打包

- `datas` 漏了 `assets/` → exe 里徽章/表情全空，但源码跑正常
- `sys._MEIPASS` 兜底，不要用 `__file__` 拼资源路径
- 打包前先关掉正在跑的 exe（文件占用），否则复制失败
- 单文件 exe 首启动会解压到临时目录，慢 1–3 秒是正常的

## 流程

- 改 UI 前先出预览图（PIL 画的也行），对方确认后再改代码 —— 省掉大量返工
- 看图贵：同一张图只读一次，要对比就自己拼一张对比图
- 稳定版 exe 不覆盖；新版本放独立目录，日志/站点/cookie 全部跟目录走
- 大改之前先确认「当前跑的是哪个 exe」：比对 exe 与源码的修改时间，别对着旧包排查

## 误判「未开播」：页面里的 roomId 占位符

**现象**：直播间明明在播，程序一直停在「等待开播…」，日志里连表头 CSV 都是空的。

**根因**：`liveMan.py` 的 `is_room_live()` 兜底用 `roomId.*:(\d+)` 抓房间号，
但直播页前面先出现占位符 `roomId\":0`，于是拿到 0 → `0 > 0` 为假 → 判定未开播。
主判断 `roomInfo...status...user_count_str` 又要求这两个字段紧挨着，
遇到「status 在 room 对象里、user_count_str 在别处」的页面同样匹配不到。

**正确做法**：

1. 只认 SSR 数据里 `roomStore.roomInfo.room` 这一个对象，用大括号配对截取
   （页面里的 JSON 可能是 `\"` 双转义形态，也要支持）；
2. 该对象里 `status`：**2 / 0 = 直播中，4 = 已结束**；
3. 兜底只认带引号的 `roomId":"<数字>"`，裸 `roomId":0` 是占位符；
4. 都判断不出就返回 `None`，让调用方按「可能开播」去连，而不是卡在等待。

```python
def _extract_room_block(text: str) -> str:
    keys = (
        '\\"roomStore\\":{\\"roomInfo\\":{\\"room\\":{',
        '"roomStore":{"roomInfo":{"room":{',
    )
    for key in keys:
        i = text.find(key)
        if i < 0:
            continue
        start, depth, j = i + len(key) - 1, 0, i + len(key) - 1
        escaped = key.startswith('\\')
        while j < len(text):
            ch = text[j]
            if escaped and ch == '\\':
                j += 2
                continue
            if ch == '{':
                depth += 1
            elif ch == '}':
                depth -= 1
                if depth == 0:
                    return text[start:j + 1]
            j += 1
    return ''
```

`status == 2` 的房间是真正在播；`status == 4` 才是下播。
别用 `room_status` 字段名去猜——网页端根本没有这个字段。
