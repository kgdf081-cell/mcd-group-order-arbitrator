# MCP 集成说明

> 本文回答三件事：**实际用了哪些 MCP 工具、以什么顺序调用、带来了什么业务价值**。
> 所有数字都来自实测，traceId 可核对。

---

## 1. 接入信息

| 项 | 值 |
|---|---|
| MCP Server | 麦当劳 MCP（`mcd-mcp`） |
| 协议 | Streamable HTTP，配置中写 `"type": "streamablehttp"` |
| Endpoint | `https://mcp.mcd.cn` |
| 鉴权 | `Authorization: Bearer ${MCD_MCP_TOKEN}`（Token 由 [open.mcd.cn/mcp](https://open.mcd.cn/mcp) 领取） |
| 客户端 | WorkBuddy |
| 本地依赖 | **零**。纯 Skill，不需要 Python / Node 环境 |
| 是否调用 `create-order` | **否**。只做查询与核价，不自动下单 |

> ⚠️ **`${MCD_MCP_TOKEN}` 是纯占位符，不是已验证的变量展开语法。**
> 本项目没有在 `~/.workbuddy/mcp.json` 里实测过 `${...}` 自动展开，配置时请手动替换成真实 Token 字符串。
> 仓库内任何位置都不应出现真实 Token。

---

## 2. 实际使用的 6 个工具

| 工具 | 用途 | 关键入参 | 关键出参 |
|---|---|---|---|
| `now-time-info` | 取服务器时间，判断门店是否营业 | 无 | `datetime` / `formatted` |
| `query-nearby-stores` | 动态查门店，禁止硬编码 `storeCode` | `city` / `keyword` / `searchType:2` / `beType:1` | `storeCode` / `distance` / `businessStartTime` / `businessEndTime` / `businessStatus` |
| `query-meals` | 建菜单池 | `storeCode` / `orderType:1` / `beType:1` | `categories[].meals[].code` / `meals{}.name` / `currentPrice` / `discountType` / `canWithOrder` / `tags` |
| `query-store-coupons` | 建券池 | 同上 | `couponId` / `couponCode` / `tradeDateTime` / `products[].productCode` |
| `query-meal-detail` | 取套餐选配结构 | 同上 + `code` | `rounds[].id` / `choices[].code` / `diffPrice` / `minQuantity` / `modification` |
| `calculate-price` | **唯一的价格来源** | 同上 + `items[]`（含 `couponId` / `couponCode` / `roundList`） | `price`（分）/ `discount` / `productList[]` / `traceId` |

未使用的工具：`create-order`（不下单）、`list-nutrition-foods`（MCP 无辣度/过敏原字段，热量不在 MVP 范围）、`query-meals-by-*` 等其余工具。

> 补充探测过、**验证后决定不用**的工具：`query-my-account`（只返回积分，无会员卡状态，因此无法判断用户是否持有麦金卡）。
> 结果见 `demo/membership-probe-20261025.md` §3.1。


---

## 3. 调用流程（六步，正常一单 14 次左右）

```
① 定门店        2 次   now-time-info → query-nearby-stores → 用户选店
② 建池          2 次   query-meals + query-store-coupons
                        按麦金卡模式处理随单购（A 剔除 / B、C 保留）
                        剔除学生专享 / 儿童餐 / 周边；打辣度标签；券商品单独入池
③ 取套餐结构    3–5 次 query-meal-detail（3–5 个模板）
④ 生成候选      0 次   按成员偏好选 round 配置，用 diffPrice 估算排序
⑤ 核价          5–8 次 calculate-price（整单一次，带 roundList / 券 / withOrder）
                        模式 C 需双次核价（无卡 + 带卡）
                        每套跑四项校验，失败即弃
⑥ 输出          0 次   三套方案 + maxS/ΣS + 辣风险三栏 + traceId + 限定语
```

**调用预算：12–20 次。超过 30 次说明在暴力枚举。**

### 参数要点（踩过的坑）

```jsonc
// 套餐替换项：round 必须是数字字符串，内部键名是 comboItemList / code
{
  "storeCode": "1990583", "orderType": 1, "beType": 1,
  "items": [{
    "productCode": "9900005466", "quantity": 1,
    "roundList": [
      { "round": "1", "comboItemList": [{ "code": "1100", "quantity": 1 }] },
      { "round": "2", "comboItemList": [{ "code": "4810", "quantity": 1 }] },
      { "round": "3", "comboItemList": [{ "code": "3050", "quantity": 1 }] }
    ]
  }]
}

// 券商品：item 内必须有 4 个字段
{
  "items": [{
    "productCode": "9900004002",
    "couponId":   "<query-store-coupons 返回>",
    "couponCode": "<query-store-coupons 返回>",
    "quantity": 1
  }]
}

// 随单购（麦金卡模式 B/C）：withOrder 在【顶层】，与 items 同级
{
  "storeCode": "<动态获取>", "orderType": 1, "beType": 1,
  "withOrder": {
    "membershipCode":   "<query-meals → meals[code].withOrder.membershipCode>",
    "membershipSpecId": "<query-meals → meals[code].withOrder.specId>"
  },
  "items": [{ "productCode": "<随单购商品>", "quantity": 1 }]
}
```

- `beType: 1` 的门店**不传 `beCode`**
- 价格单位是**分**，展示时 ÷100
- 同一商品在两个接口名称不同（菜单「薯条」→ 算价「中薯条」），**输出以 `calculate-price` 为准**
- **`withOrder` 必须在顶层**。放进 `items[]` 会被静默忽略（返回普通价、`discount: 0`、不报错）
- **`specId` 要改名为 `membershipSpecId`**；两个值都必须从 `query-meals` 实时读，仓库里不写死
- 带 `withOrder` 时 `productList` 会**多一行卡片商品**（本店 `521234`，小计 9.90），`price` 已含它
- 卡费**整单一次**，不随随单购商品数量增加

---

## 3.1 麦金卡三模式对 MCP 调用的影响

| 模式 | 建池 | `withOrder` | 核价次数 | 关键断言 |
|---|---|---|---|---|
| A 无麦金卡 | 剔除随单购 | 不传 | 5–8 | 与旧版一致 |
| B 允许使用麦金卡优惠 | 保留随单购 | **顶层必传** | 5–8（不增加） | `discount > 0` 且 `productList` 有卡行 |
| C 比较是否值得买 | 保留随单购 | 双次核价 | **翻倍，约 10–16** | 卡行小计 = 卡费，用于算净节省 |

**MCP 里查不到的东西**（这两条决定了模式 B/C 的边界）：

1. **是否持有麦金卡** —— `query-my-account` 只返回积分字段（`availablePoint` 等），无会员卡状态，也无「我已持卡」参数
2. **正式麦金卡的价格与权益** —— 没有对应工具；只能从订单 `productList` 里读到接口本次加入的那张卡

所以模式 C 只算「本单净节省」，不回答「办正式会员值不值」。

实测（与 `demo/membership-mode-c-20261025.md` 同一批调用，金额与 traceId 逐一对齐）：

| 场景 | 无卡 `P_no` | 带卡餐费 `P_meal` | 卡费 `C_card` | 净节省 `Net` | 无卡 traceId | 带卡 traceId |
|---|---|---|---|---|---|---|
| `9900011056` 全明星双人分享餐 | 115.50 | 49.90 | 9.90 | **+55.70** | `db41d2a7…97ba0` | `93cdbbb2…4dd03` |
| `9900004236` 人气超值四件套 | 57.50 | 28.00 | 9.90 | **+19.60** | `abfec8d9…8c23f` | `6db40666…8fd0d` |

> 两笔 `P_no` 都是**单品**无卡核价（`productList` 仅 1 行、`discount: 0`、无卡片行），
> 不用任何多商品整单的 traceId 代表单品价格。

同一批调用里的整单对照（`demo/membership-mode-b-20261025.md` §2）：

| 同一份 `items` | 实付 | `discount` | traceId |
|---|---|---|---|
| 不传 `withOrder` | **207.50** | 0 | `bbe5776e…e9c` |
| 顶层传 `withOrder` | **122.30** | 95.10 | `306a2e65…097` |

差 85.20 = 商品折扣 95.10 − 新增卡片行 9.90；三条恒等式（`Σsubtotal=12230`、`ΣoriginalSubtotal=21740`、`discount=9510`）全部成立。

---

## 4. 实测证据

### 4.1 全链路单次运行（`demo/run-20261024.md`）

7 次调用，52 秒内完成，6 个工具全覆盖：

| # | 工具 | traceId |
|---|---|---|
| 1 | `now-time-info` | `518bab6ee6907fc584511da723ed324b` |
| 2 | `query-nearby-stores` | `18634d6b2d0a4557002bbd1adefa5fa1` |
| 3 | `query-meals` | `e9479ed031fbbd583acf22bf6d24b4fb` |
| 4 | `query-meal-detail` | `1c11325736d0cee0a5175215086a457a` |
| 5 | `calculate-price` 方案① | `bdee196cc29672fe2acf5c58a5d126b2` |
| 6 | `calculate-price` 方案② | `33a869646b89c0b814759ff1ddd9ec59` |
| 7 | `query-store-coupons` | `193b67a152d9a09cd500245edf30df72` |

### 4.2 接口行为探测（`demo/mcp-probe-20261024.md`）

**6 个工具、19 次调用。HTTP 全部 200，业务 17 次符合 / 1 次静默异常 / 1 次空单无效。**

单次运行的券查询时间戳晚于核价，属本次运行瑕疵，已在 `demo/run-20261024.md` §1 明确标注；该券在此前 8 分钟另有一次查询（`1c57ea0bffcc144925054089f4db8a17`），返回值一致，且两套方案均未使用该券，故不影响结论。

---

## 5. 五个接口陷阱（业务价值的核心）

| # | 陷阱 | 实测数据 | 如果不处理 |
|---|---|---|---|
| 1 | **随单购虚假低价** | `9900011056` 菜单 49.90 → 无卡实付 **115.50**，差 65.60（43%）。10-24 `a6ea18c8…4346`，10-25 复核同价 `db41d2a7…97ba0` | 按菜单价排序，核价后总价接近翻倍，超预算 |
| 2 | **一券一份** | 券商品 `quantity:2` → 静默返回 1 份，traceId `28fc879b…b924`；对照组普通商品 ×2 正常 | 把一张券分给两个人，算出的价格根本无法下单 |
| 3 | **空券空单** | 券商品不带券 → `price:0`、`productList:[]`，traceId `2e49e6b5…b99e` | 当成「0 元免费商品」，得出荒谬的最优解 |
| 4 | **子项不回显** | 带 `roundList` 时 `productList` 只有父套餐 | 误以为「校验通过」，实际无法确认替换是否生效 |
| 5 | **同价替换** | 可乐 / 无糖可乐 / 雪碧 / 柠檬红茶 `diffPrice` 全 `+0`（`9900004835` round 3） | 单靠金额等式无法发现替换被忽略 |

陷阱 2、3 的 HTTP 都是 **200**，`success: true`。**只能靠逐项校验发现，不能靠状态码。**

---

## 6. 业务价值

### 6.1 对用户：把「大概差不多」变成「可复核」

- **省钱的幅度可量化**：实测 4 人局（2026-10-25），公平方案 134.00 元 vs 最低方案 111.00 元，差 23 元；差额的来源能逐条说明（小周放弃牛肉、小王放弃鸡翅）。
- **忌口保护有边界**：不是「AI 说没事」，而是「AI 明确说这条我证明不了，需要你确认」。
- **价格可信**：每一个展示金额都带 `traceId`，可以拿着去对账。

### 6.2 对平台：把不完美的接口用对

麦当劳 MCP 的数据形态有明显缺口（无辣度、无过敏原、菜单价与实付价脱节、券与菜单不同池）。本项目的价值不在「调通了 API」，而在：

- **把缺口显式建模**，而不是用模型常识填平（辣度四档）；
- **把静默异常挡在方案之外**（四级校验、券唯一性、空单防御）；
- **把有限搜索说成有限搜索**（不用「最优」「无解」这类词）。

### 6.3 一个反直觉的结论

实测发现：**券不一定便宜**。券商品 22.90 元，而同店普通套餐 `9900011126` 只要 20.50 元。

本次比较样本中该券折扣率为 50.8%，但实付仍贵 2.40 元。任何「有券就用券」或「按折扣率排序」的策略都会选错。

**正确做法：券只是候选池里的一个商品，按核价后实付统一比价，不预设优先级。**

---

## 7. 未做的事（明确边界）

| 项 | 原因 |
|---|---|
| 自动下单 `create-order` | 涉及真实扣款，MVP 不做，演示不进入 |
| 营养/热量推荐 | MCP 有 `list-nutrition-foods`，但 MVP 不做；「控热量」角色已改为「预算最紧」 |
| 过敏原判定 | MCP 无配料数据，不做任何推断与保证 |
| 全局最优 | 有限搜索，不承诺穷举 |
| 支付/订单管理 | 超出拼单仲裁范围 |

---

## 8. 复现方式

```
1. 按 README 安装 Skill（整目录）与 MCP
2. 任选一店：query-nearby-stores {city, keyword, searchType:2, beType:1}
3. query-meals / query-store-coupons {storeCode, orderType:1, beType:1}
4. query-meal-detail {..., code}
5. calculate-price {..., items:[...]}
6. 对照 demo/mcp-probe-20261024.md 的字段行为，验证你的实现是否踩到同一批坑
```

券有有效期，请使用**自己**门店的 `query-store-coupons` 返回值，不要照抄本文档的商品编码。
