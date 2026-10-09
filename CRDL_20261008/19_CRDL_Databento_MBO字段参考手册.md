# Databento MBO 数据字段完整参考手册
## 来源：https://databento.com/docs/schemas-and-data-formats/mbo
## 来源：https://databento.com/docs/standards-and-conventions/common-fields-enums-types
## 生成时间：2026-05-22

---

## 一、MBO Schema 字段列表

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `ts_recv` | uint64 | Databento服务器接收时间戳（纳秒，UNIX epoch）|
| `ts_event` | uint64 | 交易所撮合引擎接收时间戳（纳秒，UNIX epoch）|
| `rtype` | uint8 | 记录类型标识，MBO始终为160（0xA0）|
| `publisher_id` | uint16 | Databento分配的发布者ID，标识数据集和交易所 |
| `instrument_id` | uint32 | 数字化证券ID，仅保证同一日内唯一 |
| `action` | char | 事件动作类型，详见下方Action枚举 |
| `side` | char | 事件发起方向，详见下方Side枚举 |
| `price` | int64 | 订单价格，**定点精度格式**，每1单位 = 1e-9（即0.000000001）|
| `size` | uint32 | 订单数量 |
| `channel_id` | uint8 | Databento分配的通道ID，从0开始递增 |
| `order_id` | uint64 | 交易所分配的订单ID |
| `flags` | uint8 | 位字段，标识事件结束、消息特征和数据质量 |
| `ts_in_delta` | int32 | 交易所发送时间戳，表示为ts_recv之前的纳秒数 |
| `sequence` | uint32 | 交易所分配的消息序列号 |

---

## 二、Action 枚举（事件动作类型）

| 值 | 名称 | 英文说明 | 中文说明 | 是否影响订单簿 |
|----|------|---------|---------|--------------|
| `A` | Add | Insert a new order into the book | 新订单挂入订单簿 | ✅ 是 |
| `M` | Modify | Change an order's price and/or size | 修改订单价格或数量 | ✅ 是 |
| `C` | Cancel | Fully or partially cancel an order from the book | 全部或部分撤销订单 | ✅ 是 |
| `R` | Clear | Remove all resting orders for the instrument | 清空该品种所有挂单 | ✅ 是 |
| `T` | Trade | An aggressing order traded | 主动方成交记录 | ❌ 否 |
| `F` | Fill | A resting order was filled | 被动方（挂单方）成交记录 | ❌ 否 |
| `N` | None | No action | 无动作，可能携带flags等信息 | ❌ 否 |

### T 与 F 的关系

- **T（Trade）**：记录的是**主动方（aggressor）**的成交。side字段标注的是aggressor方向。
- **F（Fill）**：记录的是**被动方（resting order）**被吃的成交。side字段标注的是被吃挂单的方向。
- 同一笔成交会同时产生一条T记录和一条F记录，是同一笔交易的两面。
- **计算Delta时应使用T记录**，因为T的side直接标注了谁在主动出击。

---

## 三、Side 枚举（⚠️ 核心字段，按action分类解读）

### 当 action = T（Trade，主动成交）

| side值 | 英文官方定义 | 中文含义 | Delta计算角色 |
|--------|------------|---------|-------------|
| **`A`** | **The trade aggressor was a seller** | **卖方主动成交（主动卖）** | 计入卖量 |
| **`B`** | **The trade aggressor was a buyer** | **买方主动成交（主动买）** | 计入买量 |
| `N` | No side specified | 无方向 | 不计入有向Delta |

### 当 action = F（Fill，被动成交）

| side值 | 英文官方定义 | 中文含义 |
|--------|------------|---------|
| `A` | A resting sell order was filled | 挂着的卖单被吃掉 |
| `B` | A resting buy order was filled | 挂着的买单被吃掉 |
| `N` | No side specified | 无方向 |

### 当 action = A / M / C（挂单 / 改单 / 撤单）

| side值 | 英文官方定义 | 中文含义 |
|--------|------------|---------|
| `A` | A resting sell order updated the book | 卖方挂单变动（Ask侧） |
| `B` | A resting buy order updated the book | 买方挂单变动（Bid侧） |
| `N` | No side specified | 无方向 |

### 当 action = R（Clear book）

side 始终为 `N`。

---

## 四、Side = N 出现的场景

官方文档明确列举以下情况 side 会为 N：

1. 数据源本身不提供成交方向
2. 开盘/收盘集合竞价（opening and closing auctions）的成交
3. 非显示订单（non-displayed orders / dark）的成交
4. 隐含订单（implied orders）的成交
5. 场外交易（off-exchange trades）

---

## 五、净Delta正确计算公式

```
筛选条件：action == 'T'（仅用Trade记录）

主动买量 = sum(size) where side == 'B'
主动卖量 = sum(size) where side == 'A'
净Delta  = 主动买量(B) - 主动卖量(A)

净Delta > 0 → 买方主导
净Delta < 0 → 卖方主导
```

### ⚠️ 常见错误

```
# 错误写法（A/B含义搞反）
buys  = df[df['side']=='A']['size'].sum()   # 错！A是主动卖不是买
sells = df[df['side']=='B']['size'].sum()   # 错！B是主动买不是卖

# 正确写法
buys  = df[df['side']=='B']['size'].sum()   # B = Bid aggressor = 主动买
sells = df[df['side']=='A']['size'].sum()   # A = Ask aggressor = 主动卖
delta = buys - sells
```

---

## 六、Price 字段转换

原始price字段为定点精度整数，每1单位 = 1e-9。

```
# 如果price已经是浮点数（如9.10），说明已经过databento SDK转换，无需再除
# 如果price是大整数（如9100000000），则需要：
price_dollars = price_raw / 1_000_000_000
```

**特殊值**：`UNDEF_PRICE` = 9223372036854775807（INT64_MAX），表示null/未定义价格。

---

## 七、Flags 位字段

| Flag名称 | 位值 | 十进制 | 含义 |
|----------|------|--------|------|
| `F_LAST` | 1<<7 | 128 | 标记同一事件中同一instrument_id的最后一条记录 |
| `F_TOB` | 1<<6 | 64 | 盘口（Top-of-book）消息，非单个订单 |
| `F_SNAPSHOT` | 1<<5 | 32 | 来自回放/快照服务器的消息 |
| `F_MBP` | 1<<4 | 16 | 聚合价格级别消息，非单个订单 |
| `F_BAD_TS_RECV` | 1<<3 | 8 | ts_recv不准确（时钟问题或包重排） |
| `F_MAYBE_BAD_BOOK` | 1<<2 | 4 | 检测到不可恢复的通道间隙 |
| `F_PUBLISHER_SPECIFIC` | 1<<1 | 2 | 语义取决于publisher_id，需查对应数据集文档 |
| （保留位） | 1<<0 | 1 | 内部使用，可忽略 |

---

## 八、时间戳字段

| 字段 | 含义 | 来源 |
|------|------|------|
| `ts_event` | 交易所撮合引擎接收时间（FIX Tag 60） | 交易所提供 |
| `ts_recv` | Databento服务器接收时间（硬件时间戳，GPS同步） | Databento提供 |
| `ts_in_delta` | ts_recv 与交易所发送时间的差值（纳秒） | 计算值 |
| `ts_out` | Databento网关发送时间（仅live数据） | Databento提供 |

所有时间戳均为 **纳秒级UNIX时间戳（UTC）**。

`UNDEF_TIMESTAMP` = 18446744073709551615（UINT64_MAX），表示null/未定义。

**排序/索引时间戳**：优先使用 `ts_recv`（如果存在），否则使用 `ts_event`。

---

## 九、本项目涉及的数据集（Datasets）

| 数据集ID | 交易所 | 数据类型 |
|----------|--------|---------|
| XNAS.ITCH | Nasdaq TotalView-ITCH | 完整MBO（L3） |
| ARCX.PILLAR | NYSE Arca Integrated | 完整MBO（L3） |
| BATS.PITCH | Cboe BZX Depth | 完整MBO（L3） |
| XNYS.PILLAR | NYSE Integrated | 完整MBO（L3） |

---

## 十、MBO数据分析最佳实践

### 10.1 订单簿分析（挂单/撤单）

```python
# 筛选挂单事件
adds = mbo[mbo['action']=='A']      # 新挂单
cancels = mbo[mbo['action']=='C']   # 撤单
modifies = mbo[mbo['action']=='M']  # 改单

# Ask侧（卖方挂单）
ask_orders = adds[adds['side']=='A']
# Bid侧（买方挂单）
bid_orders = adds[adds['side']=='B']
```

### 10.2 成交分析（Delta计算）

```python
# 仅用Trade记录
trades = mbo[mbo['action']=='T']

# 主动买 = side B（Bid aggressor = 买方扫Ask）
active_buys  = trades[trades['side']=='B']['size'].sum()
# 主动卖 = side A（Ask aggressor = 卖方砸Bid）
active_sells = trades[trades['side']=='A']['size'].sum()
# 中性 = side N（集合竞价/暗池等）
neutral      = trades[trades['side']=='N']['size'].sum()

net_delta = active_buys - active_sells
```

### 10.3 订单生命周期追踪

```python
# 每个order_id的生命周期：Add → (Modify)* → Cancel/Trade/Fill
# 通过order_id关联Add时间和Cancel/Trade时间，计算存活时长
adds = mbo[mbo['action']=='A'][['ts_event','order_id','side','price','size']]
ends = mbo[mbo['action'].isin(['C','T','F'])][['ts_event','order_id','action']]
lifecycle = adds.merge(ends, on='order_id', suffixes=('_add','_end'))
lifecycle['lifetime_ns'] = lifecycle['ts_event_end'] - lifecycle['ts_event_add']
```

---

## 附录：快速记忆口诀

```
A = Ask = 卖方 = 主动卖（T记录中）= Ask侧挂单（A/M/C记录中）
B = Bid = 买方 = 主动买（T记录中）= Bid侧挂单（A/M/C记录中）
N = None = 无方向 = 集合竞价/暗池/场外

Delta = B(买) - A(卖)
正Delta = 买方主导
负Delta = 卖方主导
```

---

*本文档基于 Databento 官方文档编写，仅供内部参考。如有更新请以官方文档为准。*
