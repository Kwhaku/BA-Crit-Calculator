# 断箭 · 重创率计算器 / Broken Arrow · Critical Hit Calculator

《断箭》的重创（暴击）概率计算器。选弹药、选目标（含改装）、选部位（正 / 侧 / 后 / 顶）和距离，算出这一发的**追加概率**和**保底重创次数**，并给出发射 N 发的期望。

单文件 HTML，双击就能用，不联网、不安装。数据来自本机解包，公式来自反汇编。

> 游戏版本 **v1.2.2** · 公式确认日期 **2026-11-20** · 639 种弹药 / 234 个目标

> **本地副本说明**：`index.html` 在本目录重命名为 `重创率计算器.html`，`LICENSE` 移到项目根目录，
> 文内两处链接已随之调整（仓库里仍是 `index.html` + 同级 `LICENSE`）。

---

## 在线使用

<https://kwhaku.github.io/BA-Crit-Calculator/>

离线版：下载 [`重创率计算器.html`](重创率计算器.html)，双击用浏览器打开（约 200 KB，全部数据内嵌）。

---

## 公式

```
q = ⌊K ÷ 100⌋                                  ← 保底重创次数
p = (K − 100q) × (D ÷ HP_max) + 0.02           ← 追加一次的概率（上限 1）
n = q + 1  若 p ≥ 随机数，否则 n = q            ← 这一发实际挂几次
```

- **K**：弹药的 `CriticMultiplier`（界面里的「权重」）
- **D**：装甲计算后的最终伤害
- **HP_max**：目标最大生命
- **期望**：每发 `q + p` 次，N 发 `N × (q + p)`

**重创不减少生命值**，只让目标的探测 / 瞄准 / 装填 / 机动变差。只对**车辆**生效，步兵和飞机不在此列。

**黄与红**：同一目标先累计 **2 个黄色**重创，第 3 个才可能变红。代码里 `BuffConfig.RedCritsActivationThreshold = 2`（构造默认是 1，发货资产覆盖成 2），配置注释是「number of different crits exceeds this」。游戏内提示只写了「2 个等级：黄色和红色」，**没有说明先后顺序**——这条是从资产和实测确认的。

### 两个容易被忽略的推论

1. **重创率和伤害挂钩**——打侧面/后部（装甲低、伤害高）重创率显著更高
2. **权重百位是保底次数**——`K=202`（ATKMS、匕首）不是「低权重」，而是**每发保底挂 2 次重创**；`K=199`（集束弹）是保底 1 次 + 99% 再来一次

---

## 功能

| 模块 | 说明 |
|---|---|
| 弹药 | 639 种，10 个武器族筛选 + 搜索；自动带出伤害、穿深（近/远）、射程、使用该弹的武器 |
| 目标 | 234 个单位，可搜索、可切换是否包含火炮/防空/后勤；**可选改装装甲包** |
| 部位 | 正面 / 侧面 / 后面 / 顶部。选择器在**目标卡的「改装 / 装甲包」下面**，选中后下方直接列出**同一辆车四个面的装甲 / 伤害 / 暴击率**，点任意一行即可切换 |
| 距离 | 滑杆 0–射程；动能弹穿深随距离线性衰减，化学能不变 |
| 连发 | 1–20 发，算「至少一次」和「期望次数」 |
| 结果 | 最终伤害、伤害占比、穿深、目标装甲、伤害系数、权重、追加概率、保底次数、平均几发一暴、N 发至少一次、N 发期望；大数字旁标注当前部位 |
| 曲线 | 重创率随距离变化 |
| 排行榜 | 当前目标/部位下，全部可伤害弹药的追加率与期望排序（前 30） |

---

## 数据与方法

公式不是拟合出来的，是在 `GameAssembly.dll` 里逐条读出来的：

| 环节 | 位置 | 内容 |
|---|---|---|
| 权重来源 | `Ammunitions.0xA0` | 每发弹药自带 `CriticMultiplier` |
| 打包 | `ShellHitSystem.DealUnitDamage` @ `0x7FD5CB` | 伤害 ÷ 最大生命 → `HPLossRate`，与 K 一起塞进 `CriticalDamage` |
| 结构体 | `CriticalDamage` | `HPLossRate @0x0`、`AmmoCriticMultiplier @0x4` |
| 掷骰 | `CriticalEffectSystem.Update` @ `0x7EEA38–0x7EEAA9` | `⌊K/100⌋` → 余数 × `HPLossRate` + 0.02 → 与随机数比较 → 重创次数 n |
| 基础概率 | `BuffConfig.CriticalBaseProbability` | 构造函数默认 `0.05`，实际生效 **`0.02`**（`GameConfig` 资产覆盖） |
| 红色阈值 | `BuffConfig.RedCritsActivationThreshold` | 实际值 **`2`**：先吃两个黄，之后才可能出红 |

伤害公式：

- **动能**：`穿深 ≥ 装甲` 全伤；否则 `2 − 装甲/穿深`，下限 `0.1`；`穿深 ≤ 装甲/2` 时归零
- **化学能**：`穿深² ÷ (穿深² + 装甲²)`

### 交叉验证

- **运行态**：Frida 钩子在真实对局采集 139 发样本，覆盖 `K = 1 / 5 / 98`，预测与实测偏差均 < 0.4 次
- **社区解包**：第三方项目 BA-Units 的 630 条弹药权重与本表 **630/630 一致**
- **独立推导**：俄语站 [strategycon.ru](https://strategycon.ru/articles/broken-arrow-stats-guide) 独立解包得出同一公式（含 `⌊K/100⌋` 与 `n = q+1`）

> **版本提醒**：BA-Units 那次解包是 **v1.2.0.3（2026-10-04）**，本仓库是 **v1.2.2**。
> 两者共同的 630 条弹药里，`CriticMultiplier` 全一致，但 `Damage` 有 **16 条不同**
> （如 20×102mm 1.5→1.75、30×165mm 2.25→2.7），84mm HE 的穿深更是 45→200。
> **权重可以跨版本引用，伤害/穿深不行**——引用社区数据务必带上版本和日期。

---

## 已知限制

- 不计 APS 拦截、命中率、散布、模块损坏、乘员技能 —— 只算「单发命中 + 装甲结算 + 重创掷骰」这条链
- 改装的装甲包来自解包表，只有表里提供配置的车辆才有选项
- 步兵用单一护甲值；掩体、烟幕未建模

---

## 声明

本工具由 **bilibili:Paserly** 使用 DSH 生成，仅供参考 · 仅用于游戏组卡和学习交流。
Made with DSH by bilibili:Paserly — for reference only · For game deck-building and study purposes only.

- © 2026 bilibili:Paserly，按仓库内 [LICENSE](../LICENSE) 发布（可自由使用、修改、非商业分享，须保留署名）
- 非官方工具，与 Steel Balalaika / Slitherine 无关
- 仅用于游戏机制研究与交流，请勿商用
- 单位名称、图标等素材版权归游戏开发商所有，**不在本许可授权范围内**

配套工具：[断箭组卡器](https://github.com/Kwhaku/BA-Deck-Builder)

---

## 配套图片（图/ 目录）

| 文件 | 用途 |
|---|---|
| 01_机制_2160.png / _1080.png | 公式推导 + 两个算例（筒子 / ATACMS）+ 部件倍率表 |
| 02_速查_2160.png / _1080.png | 各类弹药对 19 血 T-14 正面（Armory 改件）的对照表 |
| 02b_速查_T90M侧面_2160.png / _1080.png | 同类对照表：18 血 T-90M 侧面（爆反改件，抗穿 180 / 抗破 600） |
| 03_省流卡_1920x1080.png | 16:9 片头结论卡 |
| 03_超宽结论_3440x1440.png | 21:9 纯文字结论卡 |
| 04_封面_16x9.png / _4x3.png | 视频封面（含标题字） |
| 04_封面纯图_16x9.png / _4x3.png | 视频封面（纯截图裁剪，无文字） |

## 部件损坏的实际效果

页面底部内置了这张表（数值读自 `BuffConfig` 资产，偏移 0x29F938E4 / 0x2AA24D34 两处一致）：

| 类型 | 字段 | 偏移 | 黄 minor | 红 major |
|---|---|---|---|---|
| 装填 Loading | MinorLoadingMultiplier / MajorLoadingMultiplier | 0x60 / 0x64 | 1.5 | 2.0 |
| 瞄准 AimTime | MinorAimTimeMultiplier / MajorAimTimeMultiplier | 0x40 / 0x44 | 1.5 | 2.0 |
| 观瞄 Optics | MinorOpticsMultiplier / MajorOpticsMultiplier | 0x48 / 0x4C | 0.66 | 0.33 |
| 机动 Mobility | MinorMobilityMultiplier / MinorMobilityMultiplierRotation | 0x50 / 0x54 | 速度 0.75 / 转向 0.5 | **无字段 → 直接瘫痪** |
| 散布 Dispersion | MinorDispersionMultiplier / MajorDispersionMultiplier | 0x58 / 0x5C | 1.15 | 1.52 |

**机动没有 Major 字段** —— 红机动不是乘倍率，而是直接瘫痪、不能动。
