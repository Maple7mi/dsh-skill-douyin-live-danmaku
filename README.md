# douyin-live-danmaku

抖音直播弹幕助手（DSH skill）：采集全量弹幕 → Aurora Dark 风格 Tkinter 看板（等级徽章 / 抖音表情 / 礼物流水）→ 按天静态站点 → PyInstaller 单文件 exe。

## 安装

```powershell
# 本地 DSH 技能目录
git clone https://github.com/Maple7mi/dsh-skill-douyin-live-danmaku.git "$env:USERPROFILE\.dsh\skills\douyin-live-danmaku"
```

装好后在会话里说「做一个抖音弹幕采集器」或直接 `/douyin-live-danmaku` 即可触发。

## 内容

- `SKILL.md` — 主流程：连接 / 解析 / 落盘 / 界面 / 打包
- `references/assets-and-rendering.md` — 官方徽章素材 URL、粉丝团长条裁切与染色、表情表来源与内联渲染
- `references/pitfalls.md` — 协议、Windows 文件、编码、Tk、打包上踩过的坑
- `references/acceptance.md` — 交付前验收清单

## 配套实现

参考项目：`Maple7mi/DouyinLiveWebFetcher`（上游 `saermart/DouyinLiveWebFetcher`，本项目在其基础上加了 Aurora Dark 界面、等级徽章、抖音表情、按天归档站点与单文件打包）。

## 许可

MIT
