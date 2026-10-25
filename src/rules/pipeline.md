# 调用流程与参数（pipeline）

六个 MCP 工具、正常路径 12–20 次调用。超过 30 次说明在暴力枚举，停下来收窄候选。

---

## 步骤 0 · 前置检查

读取本目录的 `fairness.md` / `risk-glossary.md` / `coupon-pool.md`。
读不到就**停止**，提示用户重新安装整个 `src/` 目录。

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

1. 剔除 `discountType` 含「随单购」→ 见 `coupon-pool.md` §1
2. 剔除学生/儿童专属、周边商品、`currentPrice` ≤ 0
3. 按 `risk-glossary.md` 打辣度档：高 / 疑似 / 未知
4. 券商品单独入池（可能不在菜单里），标注「券 ×1」

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

### 每次核价后立即跑四级校验

见 `fairness.md` §8。任一项失败 → 弃该组合，补下一个候选。

**容易漏的两条**：

- 券商品 `quantity` 会被静默降为 1，不能靠 `quantity` 断言「两人各一份」
- 带 `roundList` 时 `productList` 只有父套餐，子项校验只能靠「请求来源 + 金额」两段

---

## 步骤 6 · 输出

按 `fairness.md` §9 的 11 项字段。必备限定语：

> 「本次在 N 个已搜索、已核价的候选内计算。算价接口不回显套餐子项，我们只能验证请求选项来自套餐详情且金额与预期一致，无法独立确认服务端实际采用的完整选项。」

---

## 异常处理对照表

| 现象 | 判断 | 处理 |
|---|---|---|
| `productList` 为空 / `price: 0` | 空单 | 判该组合无效，**禁止**当 0 元 |
| 返回 `quantity` < 请求 `quantity` | 静默回落（券） | 判该组合无效，改为每人 1 张不同券 |
| `code: 401` / `403` | Token 失效 | 提示用户更新 `MCD_MCP_TOKEN` |
| `code: 429` | 限流 | 等待后重试一次，仍失败则减半候选数 |
| 门店营业时间外 | 不可下单 | 提示用户换门店或改时间 |
| `query-meal-detail` 返回空 | 商品不存在/已下架 | 该模板移出候选，不猜测其结构 |

---

## 调用预算自检

正常一单 4 人：门店 2 + 建池 2 + 详情 4 + 核价 6 ≈ **14 次**。

若你的调用数 > 30，说明在逐 SKU 核价——立刻停止，回到步骤 3 收窄模板。
