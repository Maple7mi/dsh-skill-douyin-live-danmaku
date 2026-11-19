# douyin-live-danmaku

抖音直播弹幕助手 —— 一个 DSH 技能（skill），也是一套可落地的实现方案：
**采集直播间全量弹幕 → Aurora Dark 风格桌面看板 → 按天静态归档站点 → 单文件 exe**。

配套实现：`Maple7mi/DouyinLiveWebFetcher`（上游 `saermart/DouyinLiveWebFetcher`）。

> 这个 README 除了讲功能，也把**已知缺陷、数据准确性边界、安全与隐私风险**写清楚了。
> 用它之前请先看第 7 节，别把它当成「官方数据接口」。

---

## 1. 它解决什么问题

抖音网页版的弹幕只活在浏览器里：刷新就没、不能导出、看不到观众等级、礼物流水也没法复盘。
这个工具把 WebSocket 推送的消息接下来，落成结构化数据，同时给你一个能一直挂在副屏上的看板。

适合：

- 主播 / 房管：实时看弹幕、礼物、付费人数、本场涨粉
- 运营：复盘一场直播的礼物流水、付费结构、峰值
- 数据党：把弹幕导出 CSV，自己算词频、留存、互动曲线

不适合：

- 需要「官方口径」的统计（人数、流水以抖音后台为准）
- 需要长期无人值守归档（依赖网页协议，接口一变就得改）

---

## 2. 功能清单

### 2.1 采集

| 能力 | 说明 | 状态 |
|---|---|---|
| 全量弹幕 | `WebcastChatMessage`，含 `user_id` / 昵称 / 内容 / 粉丝团等级 / 消费等级 | ✅ |
| 进场消息 | `WebcastMemberMessage`，可配阈值（默认只显示 15 级以上进场） | ✅ |
| 礼物消息 | `WebcastGiftMessage`，按 `group_key` 合并连击，记录数量与钻数 | ✅ |
| 点赞 / 在线人数 | `LikeMessage` / `RoomUserSeqMessage` / `RoomStatsMessage` | ✅ |
| 表情消息 | `WebcastEmojiChatMessage` + 文本表情 token 双通道 | ⚠️ 见 7.2 |
| 房间信息 | 在线人数、累计观看、开播状态，首次连接时拉一次并周期性刷新 | ✅ |
| Cookie 登录 | 拉起 Edge 扫码，落 `cookie.txt`；未登录时礼物类消息不推送 | ✅ |

### 2.2 看板（Aurora Dark）

- 左侧 75%：弹幕流，每行 = **等级徽章 + 粉丝团徽章 + 星守护 / 房管标 + 加粗昵称 + 「：」+ 内容 + 表情**
- 右侧 25%：品牌卡、房间卡（连接 / 断开 / 导出 / 登录）、四宫格指标（当前在线 / 本场涨粉 / 付费人数+粉丝团人数 / 礼物总额）、礼物流水（8 条铺满，滚轮回看本场历史）
- 高亮规则：消费 35+ / 粉丝 15+ 金色、40+ / 18+ 红色、入场 15+ 青色、礼物 90+ 洋红
- 无滚动条设计：滚轮 + 按住拖动，行宽随窗口自适应
- 双击任意弹幕复制完整内容（长弹幕被换行截断时很有用）

### 2.3 数据落地

- **CSV**（14 列，追加写，按直播日分文件）
- **JSONL**：礼物原始消息、badge 采样、可选的协议全量转储（有体积上限）
- **静态站点**：按天聚合的页面、礼物榜、CSV 下载、删除当日
- **下播自动归档 + 自动接下一场**：收到 `ControlMessage.status == 3`（直播结束）后，先清空窗口数据，再自动导出当天站点、起本地服务并打开浏览器页面；随后回到「等待开播」（20 秒冷却）继续轮询，下一场开播自动接上。一场只导出一次。

### 2.4 打包

- PyInstaller 单文件 exe（约 55 MB，含 Pillow 与表情包）
- 稳定版 / 测试版隔离：不同目录、不同 mutex、各自的 `logs/`、`site/`、`cookie.txt`

---

## 3. 界面一览

```
┌────────────────────────────── 75% ──────────────────────────────┬──── 25% ────┐
│ 弹幕   聊天/入场/礼物                                   [计数]  │ 品牌卡       │
│ 消费35+/粉丝15+  消费40+/粉丝18+  入场15+  礼物90+              │ 房间卡       │
│ ────────────────────────────────────────────────────────────── │ 四宫格指标   │
│ [24][9][星][管] Ningning：主播唱得真好听                        │ 礼物流水     │
│ [45][17]        Coconut：闪耀星辰点播😀一个【真的爱你】…        │  (8 条)      │
│ [进]            晚星：进入直播间                                │              │
└─────────────────────────────────────────────────────────────────┴──────────────┘
```

---

## 4. 数据格式

### 4.1 CSV（14 列，顺序固定，站点聚合依赖）

```
时间, kind, msg_id, user_id, nickname, fans_club_level, pay_level,
content, gift_name, gift_count, diamonds, group_id, visible, session_date
```

- `kind`：`chat` / `entry` / `like` / `gift`
- `session_date`：**直播日**（跨午夜算同一天，按开播时间切）
- `fans_club_level` / `pay_level`：0 表示没有 / 未取到

### 4.2 文件布局

```
logs/
  gui_<live_id>_<stamp>.csv          弹幕主表
  gifts_raw_<live_id>_<stamp>.jsonl  礼物原始消息
  badge_samples_*.jsonl              徽章采样（调试）
  unknown_msgs.jsonl                 未建模消息（有上限）
  methods.json                       消息类型计数
  emoji_cache/                       表情图片缓存
site/                                导出后的静态站点
cookie.txt                           登录态
```

---

## 5. 快速开始

```powershell
git clone https://github.com/Maple7mi/DouyinLiveWebFetcher.git
cd DouyinLiveWebFetcher
python -m venv .venv
.\.venv\Scripts\pip install -r requirements.txt

# 首次：登录（会拉起 Edge 扫码）
.\.venv\Scripts\python.exe danmaku_gui.py --login

# 采集
.\.venv\Scripts\python.exe danmaku_gui.py --auto-connect --room https://live.douyin.com/<房间号>

# 打包
.\.venv\Scripts\python.exe -m PyInstaller --noconfirm --clean DouyinDanmaku.spec
```

---

## 6. 结构

```
danmaku_gui.py      WebSocket + protobuf 解析 + CSV/JSONL + 事件队列
liveMan.py          房间信息、JS 算 signature、拼 wss 地址
protobuf/douyin.py  betterproto 生成类（勿手改）
aurora_gui.py       Tkinter 界面（Canvas 虚拟列表、徽章/表情渲染）
site_builder.py     按直播日聚合 → 静态站点
site_server.py      本地 http 预览
browser_login.py    Edge 扫码登录
DouyinDanmaku.spec  PyInstaller 配置
```

数据流：`ws → PushFrame → Response → messages_list → 分发解析 → 事件 dict → GUI 队列 + 落盘`

---

## 7. 已知缺陷与风险

**这一节是重点。** 下面按「功能覆盖 / 数据准确性 / 安全隐私 / 稳定性」分类，标了严重度。

### 7.1 功能覆盖

| 问题 | 严重度 | 说明与规避 |
|---|---|---|
| **会漏弹幕** | 高 | 网页协议不保证下发全部消息：低等级账号、被折叠的「引导类」消息（如 `WebcastLowPcuGuideChatMessage`）、风控限流都可能让我们根本收不到。漏掉的消息**不在任何日志里**，事后无法补。只能靠 `methods.json` + 全量转储定位，无法根治 |
| 未建模的消息被静默丢弃 | 中 | 抖音新增消息类型时不会报错，只是没进 CSV。`methods.json` 会记到，但要人工去补模型 |
| 大表情（EmojiChatMessage）不完整 | 中 | `emoji_content` 经常是空壳，得从原始 payload 扫图片 URL；没样本时无法验证 |
| 会员标识无法识别 | 中 | 观众是否「会员」在弹幕 payload 里没有字段，`MemberMessage` 只是进场不是会员。想显示会员只能另找接口或手工白名单 |
| 粉丝团徽章只到 20 级 | 低 | 官方素材 `fansclub_level_v6_N` 只有 1–20，21+ 一律按 20 显示 |
| 部分文本表情没有图 | 低 | 表情表覆盖 500+ 名字，日志里仍有 3 个左右 token 画不出来，会退化成 `[文字]` |

### 7.2 数据准确性

| 问题 | 严重度 | 说明与规避 |
|---|---|---|
| **礼物列表有过滤** | 高 | 只收「整组累计 > 90 钻」的礼物，低于阈值的礼物**不会出现在右侧流水里**（CSV 里仍然有）。看流水时别把面板当全量 |
| 礼物计数可能重复 | 中 | 断线重连后同一 `group_key` 可能再次推送，合并逻辑按 key 累加，极端情况下会多算 |
| 在线人数是采样值 | 中 | 来自 `RoomUserSeqMessage`，不是精确实时值，且抖音本身也会抖 |
| 累计观看 ≠ 本场观看 | 低 | 字段语义就是「累计」，跨场累加 |
| 时间戳是本地时间 | 低 | 没有存时区；跨时区机器上看历史会错位 |
| 「直播日」切分依赖进程一直在跑 | 中 | 跨午夜靠进程内状态维护，中途重启会把一场直播拆成两段 |

### 7.3 安全与隐私

| 风险 | 严重度 | 说明与规避 |
|---|---|---|
| **`cookie.txt` 是明文登录态** | 高 | 等价于你的抖音账号权限。任何能读这个目录的程序都能拿走。别把程序目录发给别人、别放网盘同步、别在公用电脑上登录；泄露后立刻在抖音里退出所有设备 |
| 导出站点含观众信息 | 中 | `site/` 与 CSV 里有昵称、user_id、消费等级。分享前务必脱敏，公开部署等于公开观众名单 |
| 调试转储含完整 payload | 中 | `unknown_msgs.jsonl` / 全量转储里是原始消息，包含头像 URL、ID 等，别随手附带在 issue 里 |
| exe 无代码签名 | 中 | 单文件 PyInstaller 常被 SmartScreen / 杀软误报；分发前告知用户来源，或自行签名 |
| 无自动更新 | 低 | 协议变更只能手动更新，旧版本会「连上了但没数据」 |

### 7.4 稳定性

| 问题 | 严重度 | 说明与规避 |
|---|---|---|
| **长时间运行内存增长** | 中 | 弹幕行只在窗口里滚动，不回收；挂一整天会越来越吃内存。建议关键时刻重启，或自行加行数上限 |
| 日志体积 | 中 | 全量转储有上限（20 MB），但 CSV 与礼物 JSONL 无上限，长场次会占盘 |
| 异常退出可能半行 CSV | 低 | 追加写 + flush，极端情况最后一行被截断；站点聚合时会跳过坏行 |
| 重连策略偏简单 | 低 | 网络抖动后重连，未做指数退避；被风控时表现为反复断连 |
| 单机单实例 | 低 | 靠 mutex 保证；多房间要开多个目录/多份程序 |

### 7.5 合规

- 只读公开直播间，不发送任何消息、不刷礼物、不操作账号
- 但**抓取行为本身可能违反平台用户协议**，仅建议自用、学习、自家直播间复盘
- 不要把采集到的观众数据用于商业分发或二次售卖

---

## 8. 路线图

- [ ] EmojiChatMessage 的 emoji URL 抓取（原始 payload 正则扫描）
- [ ] 弹幕行数上限 + 内存回收
- [ ] 会员 / 星守护识别（需要找到正确的数据源）
- [ ] 重连退避 + 心跳自适应
- [ ] 站点导出自动化（不再依赖手动点「导出」）

---

## 9. 许可

MIT。上游项目版权归 `saermart` 所有。
