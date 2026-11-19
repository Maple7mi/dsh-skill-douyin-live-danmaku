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
