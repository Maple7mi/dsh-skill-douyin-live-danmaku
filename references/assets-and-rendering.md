# 徽章素材与表情渲染

## 等级徽章（官方素材，可直接下载）

Base：`https://p3-webcast.douyinpic.com/img/webcast/`

| 类型 | 文件 | 原始尺寸 | 覆盖范围 |
|---|---|---|---|
| 荣誉等级（消费） | `new_user_grade_level_v1_<n>.png~tplv-obj.image` | 96×48 | 1–75 全部存在 |
| 粉丝团等级 | `fansclub_level_v6_<n>.png~tplv-obj.image` | 150×48 | **只有 1–20**，21+ 全 404 |

要点：

- 荣誉徽章直接用官方 PNG 等比缩放即可（2:1），30 级起逐级变化，60+ 带金边
- 粉丝团素材是一条**横向金色药丸长条**，左端才是「爱心 + 等级数字」，右侧空白是留给粉丝团名的
- 想只显示左端圆牌：`crop((0, 0, 48, 48))`，再与 `rounded_rectangle(radius=13)` 遮罩相乘，把右侧平切口圆掉
- 原始牌面 alpha 只有 140–190（半透明），直接贴深色背景会发暗发棕 —— 先 `alpha_composite` 到牌面主色上压平再缩放
- 想区分「1–10 金 / 11–20 橙」：对 `>10` 的图做 HSV 色相偏移，只改满足 `S>55 and 18<h<70 and V>90` 的像素，保留奶油色爱心与白色数字
- 房管 / 星守护没有官方独立图，自绘 24×24 圆形标（红「管」/ 蓝「星」）即可

## 抖音表情

两层来源，都要：

1. **文本表情**（弹幕里就是 `[看]`、`[黄脸祈祷]` 这种字符串）
   - 静态站 https://www.emojicenter.com/zh/douyin-emojis/ ，页面卡片形如
     `data-name="[黄脸祈祷] ..."` + `<img src="/douyin/images/huanglianqidao.png">`
   - 可抓到 436 张卡片 / 435 个唯一名字
2. **GitHub 素材** `YiJio/emoji-chinese`
   - `_map/dy.json`：`{"name":["[看]"],"url":"def/kan"}`
   - 真实图片在 `dy/def/<slug>.PNG`（142 个，扩展名**大写**）
   - `raw.githubusercontent.com` 有时 404，改用 `Invoke-WebRequest` 或 API `contents/`

合并后得到 `名字 → 本地文件` 映射（`map.tsv`，一行 `[名字]\t<slug>`），实际可用 500+ 名字。

## 内联渲染

- 纯文字行：`Label(text=..., wraplength=...)` 就够
- 含表情的行：用 PIL 自己排版
  - 逐字符量宽度，中文/表情都按像素换行
  - 文字用 `draw.text(anchor="ls")` 贴基线；表情 `top = baseline + 2 - pic.height`
  - 整行渲染成一张 RGBA，再 `ImageTk.PhotoImage` 塞 `Label`
  - 字体缺字形（`➕` U+2795 之类）会显示成方框：做字符替换表（`➕→＋`、`✔→√`），或按字符选 `seguisym.ttf`
- **真正的表情消息（EmojiChatMessage）**：`emoji_content` 常常是空的，但原始 protobuf 字节里能扫到
  `https://lf5-webcastcdn-tos.douyinstatic.com/obj/webcast/<id>.png`
  —— 用字节级正则扫 `https://[^\x00-\x20"']+\.(png|webp|gif)` 兜底，拿到 URL 后下载到本地缓存（`sha1(url)[:20].png`），再按 22–24px 显示
- 历史 CSV 里的 `[表情]` 没有 URL，无法还原，只能保留占位

## 尺寸约定（实践下来最好看的一组）

| 元素 | 尺寸 |
|---|---|
| 正文 | 18pt（≈24px） |
| 荣誉徽章 | 48×24 |
| 粉丝团徽章 | 24×24 |
| 房管 / 星守护 | 24×24 |
| 图标间距 | 2px |
| 图标 ↔ 昵称 | 6px |
| 行内上下 pady | 统一 8 |
