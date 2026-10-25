# 模式 B · 允许使用麦金卡优惠（实跑记录）

> 运行时间：2026-10-25 14:30（北京时间）
> 门店：`1990583`（麦当劳南京软件谷科创城餐厅，07:00–23:00）
> 运行方式：`calculate-price` **顶层携带 `withOrder`**，参数来自 `query-meals` 实时返回
> 本文件不含 Token、完整 couponId / couponCode、手机号

> ## ⚠️ 本文件的证据范围（必须先读）
>
> 1. **这是「会员模式专项核价记录」，不是完整的六步拼单链路。**
>    全程**未调用 `query-meal-detail`**，因此没有套餐 `rounds` / `diffPrice` 数据，
>    也没有做替换项优化与三套目标的完整求解。本文件只证明「带 `withOrder` 的核价如何生效、
>    金额如何构成、有哪些静默陷阱」。
> 2. **模式名是「允许使用麦金卡优惠」，不是「已有麦金卡」。**
>    MCP **无法验证用户是否已持有麦金卡**（`query-my-account` 只返回积分，无会员卡状态）。
>    本 Skill 能确认的只有一件事：这次核价带上了随单购优惠参数，接口按会员价算了。
> 3. **`withOrder` 核价结果包含接口返回的卡片行费用。**
>    本店该行是 `521234` 麦金卡体验卡（15天）¥9.90，`price` 已含它。
>    已持卡用户若被系统识别为无需再买，**实际可能少付这 9.90**；
>    但**本项目无法验证这一点**，所以金额可能高估，只能如实标注。

---

## 1. `withOrder` 的真实来源（脱敏后的原始结构）

`query-meals` 返回体里，随单购商品带有 `withOrder` 字段（traceId `a8f1d122caa002a61aec8428a6ac2c1a`）：

```json
"9900011056": {
  "name": "全明星双人分享餐八件套",
  "currentPrice": "49.9",
  "originalPrice": "115.5",
  "discountType": "随单购麦金卡优惠",
  "withOrder": { "membershipCode": "106", "specId": "880" },
  "canWithOrder": true
}
```

本店全部 **14 个**随单购商品的 `withOrder` 取值一致（`membershipCode: 106`、`specId: 880`）。

### 关键：字段名要换

`query-meals` 给的是 **`specId`**，而 `calculate-price` 顶层要的是 **`membershipSpecId`**。
这不是文档规定，是实测出来的：

| `withOrder` 写法 | 结果 |
|---|---|
| `{membershipCode, membershipSpecId}` | ✅ 生效，5980 |
| `{membershipCode}` 单独 | ✅ 生效，5980 |
| `{membershipSpecId}` 单独 | ❌ 不生效，11550，**不报错** |
| `{membershipCode, specId}`（原名） | ⚠️ 生效，属未文档化兼容，**不采用** |
| 放进 `items[]` 内 | ❌ 不生效，11550，**不报错** |

**所以规则是**：`membershipCode` 从 `query-meals` 原样取；`specId` 取到后**改名为 `membershipSpecId`** 再传给 `calculate-price`。
**`106` / `880` / `521234` 一律不得写死在代码里**——换门店、换活动就会变。

---

## 2. 对照组：同一份订单，传与不传 `withOrder`

请求（两组的 `items` 完全相同）：

```json
{
  "storeCode": "1990583", "orderType": 1, "beType": 1,
  "items": [
    {"productCode": "9900005462", "quantity": 1},
    {"productCode": "9900011056", "quantity": 1},
    {"productCode": "9900004236", "quantity": 1}
  ]
}
```

### 2.1 不传 `withOrder`（等价模式 A，仅作对照）

```json
{
  "code": 200,
  "traceId": "bbe5776e65d658dae93a4ab6fea98e9c",
  "data": {
    "price": 20750,
    "originalPrice": 20750,
    "discount": 0,
    "productList": [
      {"productCode": "9900005462", "productName": "板烧鸡腿堡三件套",         "quantity": 1, "subtotal": 3450},
      {"productCode": "9900011056", "productName": "全明星双人分享餐八件套",   "quantity": 1, "subtotal": 11550},
      {"productCode": "9900004236", "productName": "人气超值四件套随心选",     "quantity": 1, "subtotal": 5750}
    ]
  }
}
```

**¥207.50**，无卡片行。

### 2.2 顶层传 `withOrder`

请求多加一个顶层字段：

```json
{
  "storeCode": "1990583", "orderType": 1, "beType": 1,
  "withOrder": { "membershipCode": "106", "membershipSpecId": "880" },
  "items": [ ... 同上 ... ]
}
```

返回：

```json
{
  "code": 200,
  "traceId": "306a2e65ac3b0ed523c58f0e3df69097",
  "data": {
    "price": 12230,
    "originalPrice": 21740,
    "discount": 9510,
    "productList": [
      {"productCode": "9900005462", "productName": "板烧鸡腿堡三件套",       "quantity": 1, "subtotal": 3450},
      {"productCode": "9900011056", "productName": "全明星双人分享餐八件套", "quantity": 1, "originalSubtotal": 11550, "subtotal": 4990},
      {"productCode": "9900004236", "productName": "人气超值四件套随心选",   "quantity": 1, "originalSubtotal": 5750,  "subtotal": 2800},
      {"productCode": "521234",   "productName": "麦金卡体验卡（15天）",     "quantity": 1, "subtotal": 990}
    ]
  }
}
```

**¥122.30**，`discount` 9510 分。

### 2.3 这笔账怎么算

```
3450 + 4990 + 2800 + 990 = 12230  ✅ 与 price 一致
原价 3450 + 11550 + 5750 + 990 = 21740  ✅ 与 originalPrice 一致
折扣 21740 − 12230 = 9510  ✅ 与 discount 一致
```

三条恒等式全部成立，所以这张单**确实**按会员价结算，且卡费被计入。

> ⚠️ 菜单标的 `49.9` 既不是原价（115.50），也不是要付的钱（59.80 单买 / 122.30 整单）。
> **会员价的代价是当场买下 `521234` 麦金卡体验卡 ¥9.90。**

---

## 3. 4 人方案（模式 B）

| 成员 | 商品 | 实付 | 说明 |
|---|---|---|---|
| A 小林 | `9900005462` 板烧鸡腿堡三件套 ×1 | 34.50 | 不吃辣，A 已确认可接受 |
| B 小周 + C 小陈 | `9900011056` 全明星双人分享餐八件套 ×1 | 49.90 | 双人餐，会员价（原 115.50） |
| D 小王 | `9900004236` 人气超值四件套随心选 ×1 | 28.00 | 会员价（原 57.50） |
| 全员 | `521234` 麦金卡体验卡（15天） ×1 | **9.90** | 整单一次 |
| | **合计** | **122.30** | traceId `306a2e65…69097` |

- 总预算 150：**122.30 ≤ 150** ✅
- `9900011056` 是双人餐，B/C 各取一半 → 人均 24.95
- 卡费整单一次 9.90，摊到 4 人 = **2.475/人**

**输出必须标注的句子**：

> **模式 B · 允许使用麦金卡优惠（金额已包含接口返回的卡片行 ¥9.90）。**
> 接口无法验证你是否已持有麦金卡；若系统识别到你无需再买这张卡，实际可能少付 ¥9.90，
> **本金额可能高估**。本项目无法验证这一点。

三句话缺一不可：① 模式名是「允许使用优惠」而非「已有卡」；② 金额含卡片行；③ 持卡用户可能少付但无法验证。

> 原需求写的是「未包含新购麦金卡成本」，与实测**方向相反**：接口会在订单里加一行卡费。
> 按「未包含」写会让人以为 122.30 之外还要再付，属于**低估**；按「已包含」写并在持卡场景提示可能高估，才是安全的一侧。

---

## 4. 两个静默陷阱（本次都复现了）

### 陷阱 1：传了 `withOrder`，但单里没有随单购商品

```json
{
  "items": [
    {"productCode": "9900005462", "quantity": 1},
    {"productCode": "9900005466", "quantity": 1}
  ],
  "withOrder": { "membershipCode": "106", "membershipSpecId": "880" }
}
```

```json
{"code": 200, "traceId": "f897ffcad8edfb5853c9fc2b7a75fcdb",
 "data": {"price": 8190, "discount": 0,
  "productList": [
    {"productCode": "9900005462", "subtotal": 3450},
    {"productCode": "9900005466", "subtotal": 3750},
    {"productCode": "521234", "productName": "麦金卡体验卡（15天）", "subtotal": 990}
  ]}}
```

**¥81.90** = 34.50 + 37.50 + **9.90**。
两个商品都**没有**会员折扣（`discount: 0`），但卡费照收。
→ 规则：`discount === 0` 却出现卡片行 = **整单作废**，不得输出。

### 陷阱 2：`withOrder` 放进了 `items[]` 内部

```json
{"items": [{"productCode": "9900011056", "quantity": 1,
            "withOrder": {"membershipCode": "106", "membershipSpecId": "880"}}]}
```

```json
{"code": 200, "traceId": "adfab694a52b7c93283974630b11c6ad",
 "data": {"price": 11550, "discount": 0,
  "productList": [{"productCode": "9900011056", "subtotal": 11550}]}}
```

**¥115.50**，折扣 0，**无任何报错**。
位置错了就等于没传。

---

## 5. 校验结论

| 校验项 | 结果 |
|---|---|
| 请求 3 个顶层 `productCode` 全部出现在 `productList` | ✅ 3/3 |
| 每行 `quantity` 与请求一致 | ✅ |
| `Σ subtotal === price` | ✅ 12230 |
| `Σ originalSubtotal === originalPrice` | ✅ 21740 |
| `discount === originalPrice − price` | ✅ 9510 |
| 卡片行 `quantity === 1`（整单一次） | ✅ |
| `discount > 0` 且卡片行存在 | ✅ 说明会员价确实生效 |
| 空单 / 数量回落 | 未出现 |

---

## 6. 限制

1. **不能验证用户是否已持有麦金卡**：`query-my-account` 只返回积分，无会员卡状态（详见 `membership-mode-c-20261025.md`）。
2. **不能验证卡的有效条件**：「15 天」只出现在商品名里，不是结构化字段。
3. 卡费金额、卡商品编码（`521234`）、`membershipCode` / `specId` **全部运行时读取**，不得硬编码。
4. 不调用 `create-order`，不代买卡。
