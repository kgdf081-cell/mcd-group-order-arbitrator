# 调用流程与参数（pipeline）

六个 MCP 工具、正常路径 12–20 次调用。超过 30 次说明在暴力枚举，停下来收窄候选。

---

## 步骤 0 · 前置检查 + 模式确认

读取本目录的 `fairness.md` / `risk-glossary.md` / `coupon-pool.md`。
读不到就**停止**，提示用户重新安装整个 `src/` 目录。

**然后必须让用户选麦金卡模式**（三选一，不得替用户默认）：

```
A. 没有麦金卡，按普通价格计算
B. 允许使用麦金卡优惠（不要求你已持卡；接口无法验证持卡状态）
C. 想比较是否值得购买麦金卡
```

| 模式 | 建池时的随单购商品 | `withOrder` | 额外调用 |
|---|---|---|---|
| A（默认） | 剔除 | 不传 | 0 |
| B | 允许 | 顶层必传 | 0（同一次核价里带上） |
| C | 允许 | 顶层必传 | 每个候选**多跑一次**无卡核价 |

用户没回答 → 按 **A** 执行，并在输出首行写明「模式 A · 无麦金卡」。
用户选 B 或 C 但门店菜单里**没有任何** `withOrder` 商品 → 说明该店当前无随单购活动，退回 A。

---

## 步骤 1 · 定门店（2 调用）

```json
// now-time-info  {}
// → 用返回时间判断营业状态，不要用模型自己的时间

// query-nearby-stores
{"city":"南京市","keyword":"软件谷","searchType":2,"beType":1}
```

输出前 5 家给用户选：名称 / `storeCode` / 距离 / 营业时间。
**后续所有调用都用用户选中的 `storeCode`，不得硬编码。**

> `beType: 1` 返回无 `beCode`，后续不要传。

---

## 步骤 2 · 建池（2 调用）

```json
// query-meals
{"storeCode":"<选中的>","orderType":1,"beType":1}

// query-store-coupons
{"storeCode":"<选中的>","orderType":1,"beType":1}
```

处理顺序（严格按此顺序）：

1. **按模式**处理 `discountType` 含「随单购」的商品：
   - 模式 A → **剔除**
   - 模式 B / C → **保留**，并记下每个商品的 `meals[code].withOrder` 对象（`membershipCode` + `specId`）
   - 见 `coupon-pool.md` §1
2. 剔除学生/儿童专属、周边商品、`currentPrice` ≤ 0
3. 按 `risk-glossary.md` 打辣度档：高 / 疑似 / 未知
4. 券商品单独入池（可能不在菜单里），标注「券 ×1」

> 模式 B/C 下，`withOrder` 对象**只从本次 `query-meals` 返回里取**。
> 取不到（某商品 `canWithOrder: true` 但没有 `withOrder` 字段）→ 该商品按模式 A 处理，并在输出里说明。

---

## 步骤 3 · 取套餐结构（3–5 调用）

```json
// query-meal-detail
{"storeCode":"<选中的>","orderType":1,"beType":1,"code":"9900005466"}
```

**选模板的标准**（不要随机选）：

- 覆盖各价格档位（最贵的给预算宽的人，最便宜的给预算紧的人）
- 覆盖各成员的核心软偏好（牛肉 / 鸡肉 / 鸡翅 / 小食）
- 券套餐必选（它通常是最低价解）

记录每个模板：`rounds[].id`、`rounds[].name`、`choices[].code`、`choices[].diffPrice`、`minQuantity` / `maxQuantity`、`modification`。

> 券商品的详情通常**每轮只有 1 个 choice**，没有替换空间。

---

## 步骤 4 · 生成候选（零调用）

对每个「成员 × 模板」组合，按软偏好选 round 配置：

```
命中偏好 → 选命中项（记下 diffPrice）
未命中 → 选默认项（isDefault:1）
```

估算：`est = 模板基准价 + Σ diffPrice`。**仅用于排序候选，绝不输出。**

预算初筛：

```
minPossible = base + Σ_round min(diffPrice)     ← 用这个，不用 base
若 minPossible > B_m → 该组合不可行
若 base > B_m 但 minPossible ≤ B_m → 保留，核价后再判
```

> 反例：`9900005466` base 37.50，但换小杯玉米杯 −1 后 36.50。用 base 砍会误杀。

**模式 B/C 的预算初筛要多算卡费**：

```
若这一单里包含至少一个「随单购」商品：
    卡费上界 = 该单首次核价时 productList 里卡行的 subtotal（实时读，不写死）
    组合可行 ⟺ minPossible_全单 + 卡费上界 ≤ B_total
```

- 卡费是**整单一次**，不要按人数乘
- 第一次核价前还不知道卡价 → **先不要用它砍候选**，核价后拿真实卡行再判
- **禁止把卡价写成常量**（如 9.90）——卡种、卡价随门店和活动变化

> 实测教训：把两人份的随单购商品塞进 150 预算的单子，卡费 9.90 会让总价从 142.40 变成 152.30，直接超预算（`demo/membership-probe-20261025.md` §4.2）。

按三套目标（`min max S` / `min Σ S` / `min price`）各留 **top 3–5** 组合。

---

## 步骤 5 · 核价（5–8 调用）

```json
// calculate-price · 单套餐带替换项
{
  "storeCode":"<选中的>","orderType":1,"beType":1,
  "items":[{
    "productCode":"9900005466","quantity":1,
    "roundList":[
      {"round":"1","comboItemList":[{"code":"1100","quantity":1}]},
      {"round":"2","comboItemList":[{"code":"4810","quantity":1}]},
      {"round":"3","comboItemList":[{"code":"3755","quantity":1}]}
    ]
  }]
}
```

- `round` 传**数字字符串**（`"1"`），不是数字，也不是 round 名称
- 内部字段名是 `comboItemList`，元素键是 `code` + `quantity`
- 整单多人时把多个 `items` 放在同一次调用里（一次核价算全单）

### 模式 B/C 的核价：`withOrder` 在**顶层**

```json
{
  "storeCode":"<选中的>","orderType":1,"beType":1,
  "withOrder":{
    "membershipCode":"<query-meals → meals[code].withOrder.membershipCode>",
    "membershipSpecId":"<query-meals → meals[code].withOrder.specId>"
  },
  "items":[
    {"productCode":"<本次 query-meals 返回的随单购商品 A>","quantity":1},
    {"productCode":"<本次 query-meals 返回的普通商品 B>","quantity":2}
  ]
}
```

> ⚠️ **示例里不写具体商品编码。** 原因：菜单会变，写死的编码会被下一个使用者直接照抄，
> 而它可能已经下架（实测 `code 60006`）。所有 `productCode` **只能来自本次 `query-meals` 返回**。

- `withOrder` 与 `items` **同级**；放进 `items[]` 会被静默忽略（见 `coupon-pool.md` §4.1）
- `specId` → `membershipSpecId` 是唯一允许的改名；**不得写死任何会员参数**
- 卡费会作为**额外一行**出现在 `productList`，`price` 已含它
- 模式 C 还要对同一 `items` 再跑一次**不传** `withOrder` 的核价，用来算净节省

### 每次核价后立即跑四级校验

见 `fairness.md` §8。任一项失败 → 弃该组合，补下一个候选。

**容易漏的三条**：

- 券商品 `quantity` 会被静默降为 1，不能靠 `quantity` 断言「两人各一份」
- 带 `roundList` 时 `productList` 只有父套餐，子项校验只能靠「请求来源 + 金额」两段
- **模式 B/C 下 `productList` 会多一行「请求里没有的商品」（卡片行）**。逐项校验时它**不是异常**，要单独识别：`productCode` 不在 `items` 中且 `subtotal > 0` → 记为卡费。除此之外的多余行仍然是异常。

**模式 B/C 的额外断言**（任一失败 → 整单作废）：见 `coupon-pool.md` §4.5。
其中最关键一条：`discount === 0` 却出现卡行 → 说明 `withOrder` 没生效，用户白付卡费，**必须作废而不是照常输出**。

---

## 步骤 6 · 输出

按 `fairness.md` §9 的 11 项字段。必备限定语：

> 「本次在 N 个已搜索、已核价的候选内计算。算价接口不回显套餐子项，我们只能验证请求选项来自套餐详情且金额与预期一致，无法独立确认服务端实际采用的完整选项。」

**模式标签必须出现在输出第一行**：

- `模式 A · 无麦金卡（普通价格）`
- `模式 B · 允许使用麦金卡优惠 — 金额已包含接口返回的卡片行 ¥9.90；若你已持卡、系统识别后无需再买，实际可能少付，但接口无法验证，本金额可能高估`
- `模式 C · 是否值得买卡 — 以下为本次订单的净节省，不代表办正式会员划算`

模式 B/C 还要额外给出一行：`卡费 ¥<productList 卡行 subtotal> × 1（整单一次）`。

---

## 异常处理对照表

| 现象 | 判断 | 处理 |
|---|---|---|
| `productList` 为空 / `price: 0` | 空单 | 判该组合无效，**禁止**当 0 元 |
| 返回 `quantity` < 请求 `quantity` | 静默回落（券） | 判该组合无效，改为每人 1 张不同券 |
| `code: 401` / `403` | Token 失效 | 提示用户更新 `MCD_MCP_TOKEN` |
| `code: 429` | 限流 | 等待后重试一次，仍失败则减半候选数 |
| `code: 60006` + `success: false` | 商品已售罄或不可售（菜单快照过期） | **移出候选并补下一个**，不得当成「0 元」或「免费」。实测：某套餐编码 24 小时内从在售变为 60006（`ea8b5d9a…4eb9`；具体编码见 `demo/membership-mode-a-20261025.md` §3，本规则文件不复述） |
| 门店营业时间外 | 不可下单 | 提示用户换门店或改时间 |
| `query-meal-detail` 返回空 | 商品不存在/已下架 | 该模板移出候选，不猜测其结构 |
| 带 `withOrder` 但 `discount: 0` | 参数未生效（层级错 / 字段名错 / 单里没有随单购商品） | **整单作废**，回头查 `withOrder` 是否在顶层、`membershipCode` 是否来自 `query-meals` |
| `productList` 多出一行未请求的商品 | 随单购卡片行（卡费） | 记为卡费，**不算异常**；但若同时 `discount: 0` 则整单作废 |
| `canWithOrder: true` 但没有 `withOrder` 字段 | 无法确认卡参数 | 该商品按模式 A 处理，输出里说明「此商品缺卡参数，未按会员价计算」 |
| 卡行 `quantity ≠ 1` | 与实测不符（卡应整单一次） | 按返回的真实数量计费，并在输出里标注异常 |

---

## 调用预算自检

正常一单 4 人：门店 2 + 建池 2 + 详情 4 + 核价 6 ≈ **14 次**。

模式对预算的影响：

| 模式 | 变化 |
|---|---|
| A | 不变，约 14 次 |
| B | 不变，`withOrder` 搭在同一次核价里 |
| C | 核价次数**翻倍**（每套方案要跑一次无卡 + 一次带卡），约 **20 次** |

若你的调用数 > 30，说明在逐 SKU 核价——立刻停止，回到步骤 3 收窄模板。
