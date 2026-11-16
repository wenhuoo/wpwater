# Western Port Turbidity

**English** · [中文](#中文)

Live suspended-sediment prediction for any point in Western Port, Victoria. Click the map, get the current value and the next 7 days.

**https://wenhuoo.github.io/wpwater/**

---

## What it does

Click anywhere in the bay. The page remembers the coordinates in the URL (`#-38.3831,145.2707`), so you can send a spot to someone else as a link.

The page shows four things and nothing else: current suspended sediment, turbidity and clarity, how that compares with the monthly typical, and a curve you can switch between 24 hours, 7 days and 14 days. Hover or drag on the curve to read the sediment, tide and wind at any moment.

The interface language follows your browser (`zh*` → Chinese, everything else → English). Force it with `?lang=zh` or `?lang=en`. The default chart window is `?r=7d`; `?r=24h` and `?r=14d` work too.

---

## How it works

A **single-point, single-layer, quasi-steady** resuspension model. It is not a 3D hydrodynamic model and it contains no advection.

### 1. Tides

Water level is synthesised from four constituents (M₂, S₂, K₁, O₁), on the tidal planes given in the 2015 conceptual model report:

| HAT | MHWS | MHWN | AHD | MLWN | MLWS | LAT |
|---|---|---|---|---|---|---|
| 1.72 | 1.15 | 0.70 | 0.00 | −0.63 | −1.08 | −1.69 m |

The harmonic constants are calibrated against Stony Point tide tables — RMS residual 3.6 cm over 27 high/low water points. Tidal range amplification (2.2 m at Stony Point → 3.1 m at Tooradin) and phase lag (roughly 2 hours from the entrance to Tooradin) are interpolated from the report and carried by the `amp` and `lag` parameters of each control point.

Tidal current speed comes from the rate of change of water level:

```
U  = kc · |dh/dt|
τc = ρ · Cd · U²        ρ = 1025, Cd = 0.0025
```

`kc` is "how much current one metre per hour of tidal rise produces". Physically it is the ratio of tidal prism to channel cross-section: small on open shallow banks (0.3–0.6), large in constricted entrances (1.9–2.3).

### 2. Wind waves

Local wind sea uses the SMB growth curves, with fetch taken from **real geometry** — not a lookup table. Eight rays are cast from the point against the coastline polygon until each one hits land:

```
Hs = 0.0016 · (U²/g) · √(gF/U²)
T  = 0.40 · (U/g) · (gF/U²)^(1/3)
```

Offshore swell is added through a per-point penetration coefficient, then the dispersion relation `ω² = gk·tanh(kd)` is solved for wavenumber (Newton iteration) to get the bed orbital velocity:

```
ub = π·Hs / (T·sinh(kd))
τw = ½ · ρ · fw · ub²      fw = 0.02
```

### 3. Resuspension

Two **independent** entrainment channels, each following the Partheniades excess-shear law:

```
mob = (τ/τcr − 1)^0.6      τcr = 0.10 Pa

Ceq = bg + M·qf·( mob(τc) + mob(τw)·e^(−d/4) )
```

The important detail is that the depth attenuation `e^(−d/4)` multiplies **only the wave term**. Bed shear from the tide does not care how deep the water is; only waves are attenuated with depth — shallow waves reach the bed, deep ones do not. So in calm weather the tide lifts a thin layer of fluff, and only when a strong wind opens the second channel do you get a spike.

That structure was not tuned in — it was forced by the field observation of 16 October 2026 (below).

### 4. Time evolution

Concentration relaxes towards the equilibrium value with a first-order lag, 0.75 h on the rise and 2.4 h on the settle:

```
C(t+dt) = Ceq + (C − Ceq)·exp(−dt/Ts)
```

Which is why the peak arrives **one to two hours after maximum ebb**, not at maximum current.

### 5. Bed mud stock

A slow 30-day-memory variable `Q` accumulates rainfall and wind disturbance:

```
Q ← Q + (12·rain + 0.8·max(0, U−4) − Q)·(1 − e^(−1h/30d))
qf = 0.30 + 0.55·Q²
```

It answers "how much mud is lying on the bed to be picked up", and gives the model its seasonal and weather-scale memory.

---

## Data

| Quantity | Source |
|---|---|
| Wind, gusts, rain | Open-Meteo Forecast API (ECMWF), hourly, past 7 days + next 7 days |
| Offshore wave height, period | Open-Meteo Marine API |
| Mud stock | Open-Meteo Archive API (ERA5), 95 days hourly |
| Tide | Stony Point harmonic constants, calibrated to tide tables |
| Coastline, depth, tidal planes | OSM water polygon; Western Port Conceptual Model (2015); Sediment Supply & Seagrass (CSIRO 2016) |

Data is fetched on every page load and recomputed every 15 minutes. **It needs internet access** — a preview panel with a restrictive CSP will block the API calls; open it in a normal browser.

---

## Where the parameters come from

This is the part worth being explicit about.

### 7 calibrated points

Only these seven have parameters with evidence behind them:

| Site | Annual median | Basis |
|---|---|---|
| Western Entrance | 3 mg/L | Open sand, 14 m deep; ADCP peak current 1.25 m/s |
| Eastern Entrance | 4 mg/L | 6 m deep, sand |
| Stony Point | 6 mg/L | EPA Hastings / Barrallier about 5 mg/L year-round; ADCP 0.85–1.00 m/s |
| Tankerton | 10 mg/L | Field observation, 16 Oct 2026 (below) |
| Middle Spit | 16 mg/L | Order-of-magnitude estimate |
| Tyabb Bank | 18 mg/L | Order-of-magnitude estimate |
| Quail Bank | 15 mg/L | Order-of-magnitude estimate |

The suspendable mud mass `M` at each point was found by bisection, driving the model with five years (2020–2024) of hourly ERA5 weather until the annual median landed on the target. A 12-month bias-correction table was then fitted so that monthly medians match the EPA 1990–2009 monthly values.

**Caveat:** Tankerton, Middle Spit, Tyabb Bank and Quail Bank have **no co-located measurements**. Their values carry magnitude, not precision. Their "monthly typical" is set from depth (1.3–3.2 m) and proximity to mud banks.

### 15 additional control points

To make an arbitrary location computable, 15 more control points were added (Hastings, Rhyll, Corinella, Tooradin, Lang Lang, the French Island shoreline, and so on). Their parameters are estimated from chart depth and the report's descriptions — **not measured**.

### Spatial interpolation

Parameters at any (lat, lon) come from inverse-distance weighting over the 22 control points (weight ∝ 1/d^2.8), with `M`, `typ` and `CORR` interpolated in log space. The control points themselves take exact values, so results at a calibrated site are identical to the calibrated result.

Fetch is not interpolated — it is ray-cast against the coastline polygon, so island shelter and the open bay mouth come out right automatically.

---

## Field check: 16 October 2026

The only piece of ground truth. On that day at Tankerton the water was clear from 08:00 to 14:00 — bottom visible at 4–5 m, very clear at 2–3 m. On previous fishing trips 4–5 m was usually *not* visible.

**Tide first.** High water 05:00 (1.35 m AHD) → low water 11:03 (−0.71 m) → next high 16:42 (0.99 m). The observation window straddles the 11:03 low-water slack — the six hours of weakest current and lowest bed shear in the whole day. Wind was 3–6 m/s, dead calm 1.4 m/s at 12:00–13:00, no rain. It was always going to be a clear day.

**What the model got wrong.** The old version summed tidal and wave shear and multiplied by a single `e^(−d/4)`. At 08:00, mid-ebb, τ ≈ 0.22 Pa, so it insisted the bed was being stirred and returned 10–12 mg/L — two to three times too high — with the clearest window two to four hours late.

Tried and rejected:

- Lowering `kc` alone — recalibration absorbs it, no change in shape
- Raising `τcr` (0.15–0.35) — `M` runs into its cap and concentration saturates at `Cmax`; structurally impossible → **τcr ≈ 0.10 Pa is the only self-consistent value**
- Adding a flood/ebb phase weight — widened the window but did not fix 08:00–10:00; reverted

What was changed was **structure, not parameters**: tide and waves entrain independently, and only the wave term is attenuated with depth. Tankerton's `kc` was dropped from 0.75 to 0.34 in the same pass (about 0.25 m/s peak ebb current, consistent with 0.2–0.3 m/s on a shallow bank).

**Result:** 02:00 peak 45 → 08:00 6.5 → 11:00 5.0 → 14:00 3.2 → 17:00 2.7 mg/L (clarity 2.2 → 4.9 m), back to 13 on the evening flood. Trend and magnitude line up; the early morning is still about twice the observed turbidity.

---

## Does offshore swell push turbid water in through the entrance?

No — not in Western Port. Checked against ERA5 and Marine data for 18 October – 4 November 2026:

| | Calm day, 18 Oct | Big swell, 26 Oct |
|---|---|---|
| Offshore Hs at Flinders | 0.6–0.7 m | 3.2–3.6 m |
| **Western Entrance TSS** | 1.3–3.4 mg/L | **1.4–4.7 mg/L** |
| Tankerton TSS | 2.5–15 mg/L | 45–150 mg/L |

Offshore swell went up fivefold and the entrance barely moved, from 2.5 to 4 mg/L — because the entrance is 14 m deep, so `sinh(kd)` leaves almost nothing of the orbital velocity at the bed, and the sandy bed has a high entrainment threshold.

Lag correlation does not support transport either: the correlation between offshore Hs and Tankerton TSS peaks at **lag 0 hours** (r = 0.41) and falls off monotonically. A "stirred up at the entrance, arrives hours later" mechanism would show a peak at some positive lag. It does not.

The real mechanism is **common forcing**: the same ocean storm produces both the offshore swell and the strong local wind inside the bay, and that local wind stirs the shallow bed directly. On 26 October, 100 % of Tankerton's bed shear came from waves (τw 0.33–0.90 Pa vs τc 0.00–0.05 Pa).

**Practical upshot:** watching the swell at Flinders to decide whether to go is the right instinct, but the reasoning changes — it is not "the entrance is turbid", it is "the local bed is being stirred". The exception is a high swell with no wind inside the bay, when Tankerton may stay clear.

---

## What it does not know

- **No 3D hydrodynamics, no advection.** The report notes a clockwise circulation around French Island carrying fine mud from the northern banks towards the eastern shore, so these banks are not simply "mud produced here, resuspended here". This model computes local resuspension only; mud advected in from elsewhere is not represented, and the absolute level is anchored by bias correction instead.
- **No flocculation, no grain-size change, no seagrass wave damping, no bioturbation.**
- **"Monthly typical" is a median, not a mean.** The suspended-sediment distribution is strongly right-skewed (a few windy days pull the mean up), so on a calm dry day the reading will sit well below it — that is a real result, not a bug.
- **Uncertainty grows away from the calibrated points.** The 15 additional control points are estimates, and interpolation between them is a smooth blend with no physical constraint behind it.
- **Wind and rain come from a reanalysis/forecast grid**, so near-shore local wind (sea breeze, terrain acceleration) may be smoothed out.

**Absolute uncertainty is roughly ±50 %.** Good for "is it turbid today, and when will it be worst"; not good enough for engineering decisions.

---

## Run locally

A single file, no build step:

```bash
git clone https://github.com/wenhuoo/wpwater.git
cd wpwater
open index.html        # or just open it in a browser
```

To change parameters, edit the `CTRL` array in `index.html` — 22 rows, one per control point:

```
[lat, lon, depth m, kc, log10(M), background mg/L, swell penetration, Cmax,
 tide lag h, range amplification, 12 monthly typicals, 12 monthly corrections]
```

Reload and it takes effect. No compilation.

---

## References

- *Western Port Conceptual Model* — Draft Report, 2015
- *Western Port Sediment Supply and Seagrass* — CSIRO, 2016
- EPA Victoria, 1998–2009 water quality monitoring
- [Open-Meteo](https://open-meteo.com/) — weather and wave data

---

# 中文

[English](#western-port-turbidity) · **中文**

西港（Western Port, Victoria）任意一点的悬沙浓度实时预测。点地图选位置，给出当前值与未来 7 天的走势。

**https://wenhuoo.github.io/wpwater/**

---

## 它做什么

点地图上任意水域即可。页面把坐标记在地址栏里（`#-38.3831,145.2707`），可以直接把某个钓点发给别人。

页面只给四样东西：当前悬沙浓度、浊度与透明度、本月典型值对比、一条可在 24 小时 / 7 天 / 14 天之间切换的曲线。在曲线上滑动（或按住拖动）就能读到那一刻的悬沙、潮位和风速。

界面语言跟随浏览器（`zh*` → 中文，其余 → 英文）。可以用 `?lang=zh` 或 `?lang=en` 强制指定。时间窗默认 `?r=7d`，也支持 `?r=24h` 和 `?r=14d`。

---

## 它算的是什么

一个**单点、单层、准稳态**的再悬浮模型。不是三维水动力模型，不含平流。

### 1. 潮流

潮位用 4 个分潮（M₂、S₂、K₁、O₁）调和合成，基准面取自 2015 年概念模型报告的潮汐特征面：

| HAT | MHWS | MHWN | AHD | MLWN | MLWS | LAT |
|---|---|---|---|---|---|---|
| 1.72 | 1.15 | 0.70 | 0.00 | −0.63 | −1.08 | −1.69 m |

调和常数用 Stony Point 潮汐表标定，27 个高/低潮点残差 RMS 3.6 cm。潮差沿程放大（Stony Point 2.2 m → Tooradin 3.1 m）与潮时滞后（入口到 Tooradin 约 2 小时）按报告逐站内插，由每个控制点的 `amp` 与 `lag` 两个参数表达。

潮流流速由潮位变化率换算：

```
U  = kc · |dh/dt|
τc = ρ · Cd · U²        ρ=1025, Cd=0.0025
```

`kc` 是"单位潮位变化率能推起多快的水"，物理上等于潮汐棱柱与过水断面的面积比。开阔浅滩小（0.3–0.6），收窄的入口大（1.9–2.3）。

### 2. 风浪

局地风浪用 SMB 成长公式，吹程按风向取八方位实测几何——**不是查表，是从岸线多边形里射线求交算出来的**：

```
Hs = 0.0016 · (U²/g) · √(gF/U²)
T  = 0.40 · (U/g) · (gF/U²)^(1/3)
```

外海涌浪按站点透入系数叠加，再解色散方程 `ω² = gk·tanh(kd)` 求波数（牛顿法），算床面轨道速度：

```
ub = π·Hs / (T·sinh(kd))
τw = ½ · ρ · fw · ub²      fw=0.02
```

### 3. 起悬

两条**独立**的起动通道，用 Partheniades 超量切应力律：

```
mob = (τ/τcr − 1)^0.6      τcr = 0.10 Pa

Ceq = bg + M·qf·( mob(τc) + mob(τw)·e^(−d/4) )
```

关键在最后那个 `e^(−d/4)` **只乘风浪项**。潮流的底床切应力与水深无关，只有风浪会随水深衰减——浅水波能直接搅到床面，深水里搅不到。这样平静期潮流只掀起一层浮泥，大风天两条通道同时打开才出现尖峰。

这一条不是凑出来的，是 2026-10-16 的现场观测逼出来的（见下文）。

### 4. 时间演化

悬沙浓度按一阶松弛逼近平衡值，起悬 0.75 h、沉降 2.4 h：

```
C(t+dt) = Ceq + (C − Ceq)·exp(−dt/Ts)
```

所以峰值出现在**最大落潮流之后**一到两小时，而不是流速最大时。

### 5. 床面细泥存量

一个 30 天记忆的慢变量 `Q`，由前期降水与风扰动累积：

```
Q ← Q + (12·rain + 0.8·max(0,U−4) − Q)·(1 − e^(−1h/30d))
qf = 0.30 + 0.55·Q²
```

它决定"床面有多少泥可以被搅起来"，是季节与天气尺度记忆的来源。

---

## 数据

| 量 | 来源 |
|---|---|
| 风、阵风、降水 | Open-Meteo Forecast API（ECMWF），过去 7 天 + 未来 7 天逐时 |
| 外海波高、周期 | Open-Meteo Marine API |
| 细泥存量 | Open-Meteo Archive API（ERA5），95 天逐时 |
| 潮汐 | Stony Point 调和常数，按潮汐表标定 |
| 岸线、水深、潮汐面 | OSM 水域多边形；Western Port Conceptual Model (2015)；Sediment Supply & Seagrass (CSIRO 2016) |

页面每次打开都会重新取数，每 15 分钟自动重算一次。**需要联网**；内置预览面板的 CSP 会拦掉外部请求，用浏览器直接打开即可。

---

## 参数从哪来

这是最需要说清楚的部分。

### 7 个标定过的点

只有这 7 个点的参数是有依据的：

| 站点 | 年均中位数 | 依据 |
|---|---|---|
| Western Entrance | 3 mg/L | 开阔砂底、水深 14 m；ADCP 实测最大流速 1.25 m/s |
| Eastern Entrance | 4 mg/L | 水深 6 m，砂底 |
| Stony Point | 6 mg/L | EPA 在 Hastings / Barrallier 全年约 5 mg/L；ADCP 0.85–1.00 m/s |
| Tankerton | 10 mg/L | 2026-10-16 现场目视（见下） |
| Middle Spit | 16 mg/L | 量级设定 |
| Tyabb Bank | 18 mg/L | 量级设定 |
| Quail Bank | 15 mg/L | 量级设定 |

每个点的可悬浮泥量 `M` 是用 2020–2024 五年 ERA5 逐时气象驱动、二分求出来的，使年均中位数落在目标值上；再按 12 个月算一张偏差订正表，让月度中位数也对上 EPA 1990–2009 的月度值。

**注意**：Tankerton / Middle Spit / Tyabb / Quail 这四个浅滩点**没有同点实测**，只有量级意义。它们的"本月典型值"是按水深 1.3–3.2 m + 紧邻泥滩的量级设定的。

### 另外 15 个控制点

为了让任意位置都能算，另外加了 15 个控制点（Hastings、Rhyll、Corinella、Tooradin、Lang Lang、French Island 沿岸等），参数按海图水深与报告描述估计，**不是实测**。

### 空间插值

任意点 (lat, lon) 的参数由 22 个控制点做反距离加权（权重 ∝ 1/d^2.8）得到，`M`、`typ`、`CORR` 在对数空间插值。控制点本身取精确值，所以在标定点上算出来的结果和标定完全一致。

吹程不用插值——从岸线多边形射线求交直接算，所以岛屿遮挡、湾口开口都自动正确。

---

## 2026-10-16 的现场核对

唯一一次地面真值。当天在 Tankerton，08:00–14:00 水一直清，4–5 m 能看到水底、2–3 m 很清楚；以往钓鱼时 4–5 m 通常看不到底。

**先查潮汐**：高潮 05:00（1.35 m AHD）→ 低潮 11:03（−0.71 m）→ 次高潮 16:42（0.99 m）。视窗 08:00–14:00 **正好跨过 11:03 的低潮平潮**，是一天里潮流最弱、床面切应力最低的六小时。当天风 3–6 m/s、12–13 时静风 1.4 m/s、无雨。这天本来就该是清的。

**模型原来错在哪**：旧版把潮流和风浪切应力加起来再乘一个整体的 `e^(−d/4)`，08:00 落潮中段 τ≈0.22 Pa 一直被判定为"正在起悬"，给 10–12 mg/L，比观感高 2–3 倍，最清时段也偏晚 2–4 小时。

试过并否掉的方案：

- 单纯调低 `kc`：重新标定会把量补回来，治不了
- 抬高 `τcr`（0.15–0.35）：`M` 冲到上限、浓度饱和在 `Cmax`，结构上不成立 → `τcr≈0.10 Pa` 是唯一自洽值
- 加涨落潮相位权重：拉宽了窗口但治不了 08–10 点，已回退

最后改的是**物理结构**而不是参数：潮流、风浪各自独立起动，只有风浪项乘水深衰减。配套把 Tankerton 的 `kc` 从 0.75 降到 0.34（对应落潮峰值流速约 0.25 m/s，与浅滩 0.2–0.3 m/s 的量级一致）。

**结果**：02:00 峰值 45 → 08:00 6.5 → 11:00 5.0 → 14:00 3.2 → 17:00 2.7 mg/L（透明度 2.2 → 4.9 m），20:00 涨潮回到 13。趋势与量级对上了，清晨仍偏高约 2 倍。

---

## 外海涌浪会不会通过入口把浑水带进来？

不会，至少在 Western Port 不会。用 2026-10-18 至 11-04 的 ERA5 + Marine 数据核过：

| | 平静天 10/18 | 大涌浪天 10/26 |
|---|---|---|
| Flinders 外海 Hs | 0.6–0.7 m | 3.2–3.6 m |
| **西口 TSS** | 1.3–3.4 mg/L | **1.4–4.7 mg/L** |
| Tankerton TSS | 2.5–15 mg/L | 45–150 mg/L |

外海涌浪涨 5 倍，西口只从 2.5 涨到 4 mg/L——因为西口水深 14 m，涌浪的底轨道速度被 `sinh(kd)` 衰减得所剩无几，加上粗砂底起动阈值高。

滞后相关分析也不支持输运假说：西口 Hs 与 Tankerton TSS 的相关系数在**滞后 0 小时最高**（r=0.41），之后单调下降。如果是"入口搅浑 → 几小时后输运过来"，应该在某个正滞后出现峰值。

真实机制是**共同响应**：大洋风暴同时制造外海涌浪和湾内大风，湾内浅水区的风浪直接掀本地底泥。10/26 那天 Tankerton 的切应力 100% 来自风浪（τw 0.33–0.90 Pa vs τc 0.00–0.05 Pa）。

**实用结论**：看 Flinders 的涌浪判断该不该去，方向对，但推理路径要改——不是"入口水浑"，是"本地被风浪掀了"。例外是涌浪高但湾内无风的天气，那种情况下 Tankerton 可能依然清。

---

## 它不知道什么

- **没有三维水动力场，不含平流。** 报告指出绕 French Island 的顺时针环流会把北面浅滩的细泥往东岸搬，所以这些浅滩的泥沙并不是"本地产生、本地悬浮"那么简单。本模型只算局地再悬浮，横向输运进来的"别处的泥"算不进来，绝对水平靠偏差订正锚定。
- **没有絮凝、粒径变化、海草实时消浪、生物扰动。**
- **"本月典型值"是中位数不是月均值。** 悬沙分布强烈右偏（少数大风天拉高整体），所以平静少雨的日子读数是会明显低于它——那是真实结论，不是算错。
- **离标定点越远越不确定。** 15 个补充控制点的参数是估计值，两点之间的插值只是平滑过渡，没有物理约束。
- **风与降水来自再分析/预报网格**，近岸局地风（海风、地形加速）可能被平滑。

**绝对值不确定度约 ±50%。** 适合判断"今天浑不浑、什么时候最浑"，不适合工程决策。

---

## 本地运行

单文件，没有构建步骤：

```bash
git clone https://github.com/wenhuoo/wpwater.git
cd wpwater
open index.html        # 或直接用浏览器打开
```

改参数就改 `index.html` 里的 `CTRL` 数组（22 行，每行一个控制点）：

```
[lat, lon, 水深m, kc, log10(M), 背景mg/L, 涌浪透入, Cmax, 潮时滞后h, 潮差放大, 12个月典型值, 12个月订正]
```

改完刷新即可，不需要重新编译。

---

## 参考

- *Western Port Conceptual Model* — Draft Report, 2015
- *Western Port Sediment Supply and Seagrass* — CSIRO, 2016
- EPA Victoria, 1998–2009 水质监测
- [Open-Meteo](https://open-meteo.com/) — 气象与波浪数据
