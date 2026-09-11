# 快捷指令：凌晨自动续签 + 早上切回 Loon

> 面向 standalone SideStore 0.7.0 + standalone LiveContainer 环境。
> 所有动作名称均已在源码 / 官方文档中核实（见文末「来源」）。

---

## 0. 三条前置设置（不做的话自动化大概率跑不起来）

| # | 设置 | 路径 |
|---|---|---|
| 1 | **允许锁定时运行** | 快捷指令 → 每个快捷指令右上角 `···` → 底部「信息」→ **隐私** → 打开**「允许锁定时运行」** |
| 2 | **关闭运行前询问** | 建自动化时最后一步把「**运行前询问**」关掉（选「立即运行」） |
| 3 | **别开低电量模式** | 低电量模式会掐掉后台任务；同时给 SideStore 打开**后台 App 刷新** |

⚠️ **锁屏是硬门槛**：定时自动化里凡是「打开 App」类的动作，在锁屏状态下都会失败
（这是 iOS 的限制，不是设置问题）。下面的配方**刻意避开了「打开 App」**，
全程用后台动作（App Intent / URL Scheme），所以锁屏也能跑。

---

## 1. 自动化 A：每天 05:00 自动续签

**创建**：快捷指令 App → 底部「**自动化**」→ 右上 `+` → **特定时间** → 05:00 →
（重复：每天）→ **新建空白自动化**

**动作按这个顺序添加**（每加一个就在搜索框里搜名字）：

| 顺序 | 动作 | 参数设置 |
|---|---|---|
| 1 | **LocalDevVPN** | Operation = **Enable**（开启） |
| 2 | **等待** | **10** 秒（等隧道建起来） |
| 3 | **Refresh All Apps**（SideStore 提供） | 无参数 |
| 4 | **等待** | **75** 秒（刷新实际要 40–60 秒，别提前断开） |
| 5 | **LocalDevVPN** | Operation = **Disable**（关闭，把 VPN 槽位让出来） |
| 6 | **显示通知** | 文本填：`续签已执行 05:00` |

**第 6 步别省**——自动化跑没跑、跑没成功，第二天早上看通知中心就知道，
否则你根本无从判断（快捷指令的自动化没有运行历史）。

> 找不到「LocalDevVPN」这个动作？说明你装的 LocalDevVPN 版本太老，
> 去 App Store 更新（该动作在 v1.1.2 加入，且 `openAppWhenRun = false`，后台静默执行）。

---

## 2. 自动化 B：每天 05:20 再刷一次（可选但建议加）

同 A，时间设 **05:20**，动作完全一样（通知文本改成 `续签重试 05:20`）。

理由：刷新是**幂等且不消耗 App ID 额度**的（复用已有 ID），多跑一次零成本，
但能把「某一次偶然失败」的风险盖掉。

---

## 3. 自动化 C：每天 08:00 切回 Loon

| 顺序 | 动作 | 参数 |
|---|---|---|
| 1 | **LocalDevVPN** | Operation = **Disable**（保险起见先关一次） |
| 2 | **等待** | 3 秒 |
| 3 | **打开 URL** | `loon://on` |
| 4 | **显示通知** | `已切回 Loon 08:00` |

**关于第 3 步**：

- `loon://on` / `loon://off` 是 Loon **官方文档**给出的 URL Scheme，用来开关 VPN
- **首次**通过快捷指令触发时，iOS 会弹一次「Loon 想要添加 VPN 配置」→ 点**允许**，之后就不再问了
- ⚠️ 如果早上 8 点你手机还是锁着的，这一步**可能失败**（打开 URL 有概率需要解锁）。
  失败的表现是：8 点那条通知没出现。
  **兜底**：把自动化 C 简化成只做第 1、2、4 步（只关 LocalDevVPN），
  你拿起手机后自己点一下 Loon 的启动按钮即可——就是一秒钟的事。

---

## 4. 三个让它更稳的细节

1. **联网**：刷新需要出网连 Apple。Wi-Fi 或蜂窝都行（LocalDevVPN 在蜂窝下也能工作），
   但**别在飞行模式**下跑。
2. **LocalDevVPN 的 IP 保持默认**：App 里如果改过 Tunnel/Device IP，
   快捷指令那个动作也有对应参数（Tunnel IP / Device IP），留空 = 用 App 内的设置，别乱填。
3. **双保险**：自动化的同时，保持 LocalDevVPN 的「**启动时自动连接**」打开 +
   SideStore 自带后台刷新。三层兜底，实际不会过期。

---

## 5. 怎么验收它真的在跑

- **最直接的证据**：第二天看通知中心有没有那两条通知（没有 = 自动化根本没触发）
- **结果证据**：SideStore → My Apps → LiveContainer 的剩余天数保持在 6–7 天
- **终极证据**：连续 10 天不去管它，LiveContainer 还能正常打开

---

## 6. 已知风险（诚实告知）

| 风险 | 概率 | 应对 |
|---|---|---|
| `Refresh All Apps` 在锁屏 + 后台下失败 | 中 | SideStore 的 `RefreshAllAppsIntent` 有「27 秒跑不完就请求切前台」的设计，锁屏时切前台会受阻 → 靠自动化 B（05:20 重试）+ 自带后台刷新兜底 |
| 08:00 那步 URL Scheme 需要解锁 | 中 | 简化成只关 LocalDevVPN，手动点开 Loon |
| 忘了关低电量模式 | 高 | 充电时跑最稳；低电量模式会掐后台 |

---

## 来源（可自行核对）

- **LocalDevVPN 的快捷指令动作**：`jkcoxson/LocalDevVPN` → `LocalDevVPN/VPNShortcuts.swift`
  → `struct ControlLocalDevVPNIntent`，`static var title = "LocalDevVPN"`，
  参数 `operation: enable / disable / toggle`，且 **`static var openAppWhenRun = false`**（后台执行）
- **Loon 的 URL Scheme**：官方文档 <https://nsloon.app/docs/Scheme/>
  → `loon://on` 开启 VPN、`loon://off` 关闭 VPN
- **本地 VPN 原理**：`SideStore/StosVPN` 的 `PacketTunnelProvider.swift`
  （把发往 `10.7.0.1` 的包源/目的 IP 互换后发回设备）
