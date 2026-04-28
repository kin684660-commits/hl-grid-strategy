# 潮汐网格策略 · Tidal Grid Strategy

skill_id: hl-grid-strategy
basic_skills: [hyperliquid-plugin]

---

## 策略哲学

> 「上善若水，水善利万物而不争」——《道德经》

市场如潮水，涨落皆有规律。网格策略不预测方向，而是在价格的起伏中顺势收割。
每一次波动都是机会；每一笔成交都是市场对耐心的奖赏。

本策略的核心信念：
- **不预测，只应对** — 网格覆盖区间，价格自然来回成交
- **以小搏多** — 10倍杠杆放大名义价值，用频率积累笔数和交易量
- **守住本金** — 严格风控，损耗可控，绝不追仓

---

## 策略参数

| 参数 | 值 | 说明 |
|------|----|------|
| 交易标的 | BTC 永续合约 | 流动性最好，滑点最小 |
| 网格层数 | 10层（5买+5卖）| 覆盖±1.5%价格区间 |
| 网格间距 | 0.3% | 适中，兼顾成交频率与手续费 |
| 单笔 size | 0.0003 BTC | 名义价值约 $28，远超$10最低限制 |
| 杠杆 | **10x 全仓** | 资金利用率最大化，保留$10缓冲 |
| 订单类型 | Limit GTC（挂单）| Maker费率0.015%，比Taker省2/3 |
| 策略ID | **hl-grid-strategy** | 比赛统计必填，每笔必带 |

> 资金分配：30 USDC × 10x = $300 名义。10笔 × $28 = $280 名义，占用保证金 $28，剩余 $2 缓冲。
> 爆仓线：距当前价约 -10%，网格区间 ±1.5%，安全边际充足。

---

## 风控规则（铁律）

**硬止损：**
- 账户净值低于 **$22**，立即取消所有订单并停止策略
- BTC价格偏移超过 **±2.5%**，先取消再重新布局，不强撑

**Gas 保护：**
- Arbitrum ETH 余额低于 **0.0003 ETH** 时，先充值再操作
- 检查方式：`hyperliquid address`

**仓位控制：**
- 每次刷新时检查净持仓，若偏斜超过 **0.0006 BTC**，先部分平仓再布新网格
- 绝不在净持仓未处理的情况下叠加同向订单

---

## 第一步：启动前检查

```bash
# 检查钱包状态和余额
hyperliquid quickstart

# 首次运行必做：注册签名地址
hyperliquid register
```

若 `register` 返回 `setup_required`，执行：
```bash
hyperliquid deposit --amount 30 --confirm
```

---

## 第二步：获取当前价格

```bash
hyperliquid prices --coin BTC
```

记录 `midPrice`，例如：`94000`

---

## 第三步：计算网格价位

设当前价格为 **P**，间距 **G = 0.003（0.3%）**：

| 层级 | 方向 | 公式 | 示例（P=94000）|
|------|------|------|----------------|
| B1 | 买入 | round(P × 0.997) | 93718 |
| B2 | 买入 | round(P × 0.994) | 93436 |
| B3 | 买入 | round(P × 0.991) | 93154 |
| B4 | 买入 | round(P × 0.988) | 92872 |
| B5 | 买入 | round(P × 0.985) | 92590 |
| S1 | 卖出 | round(P × 1.003) | 94282 |
| S2 | 卖出 | round(P × 1.006) | 94564 |
| S3 | 卖出 | round(P × 1.009) | 94846 |
| S4 | 卖出 | round(P × 1.012) | 95128 |
| S5 | 卖出 | round(P × 1.015) | 95410 |

---

## 第四步：提交网格（批量下单）

⚠️ **`--strategy-id hl-grid-strategy` 每次必带，这是比赛统计的唯一凭证**

将计算好的价格填入 `/tmp/btc-grid.json`（以P=94000为例）：

```bash
cat > /tmp/btc-grid.json << 'EOF'
[
  {"coin":"BTC","side":"buy","size":"0.0003","type":"limit","price":"93718","tif":"Gtc"},
  {"coin":"BTC","side":"buy","size":"0.0003","type":"limit","price":"93436","tif":"Gtc"},
  {"coin":"BTC","side":"buy","size":"0.0003","type":"limit","price":"93154","tif":"Gtc"},
  {"coin":"BTC","side":"buy","size":"0.0003","type":"limit","price":"92872","tif":"Gtc"},
  {"coin":"BTC","side":"buy","size":"0.0003","type":"limit","price":"92590","tif":"Gtc"},
  {"coin":"BTC","side":"sell","size":"0.0003","type":"limit","price":"94282","tif":"Gtc"},
  {"coin":"BTC","side":"sell","size":"0.0003","type":"limit","price":"94564","tif":"Gtc"},
  {"coin":"BTC","side":"sell","size":"0.0003","type":"limit","price":"94846","tif":"Gtc"},
  {"coin":"BTC","side":"sell","size":"0.0003","type":"limit","price":"95128","tif":"Gtc"},
  {"coin":"BTC","side":"sell","size":"0.0003","type":"limit","price":"95410","tif":"Gtc"}
]
EOF
```

先预览（不签名）：
```bash
hyperliquid order-batch \
  --orders-json /tmp/btc-grid.json \
  --strategy-id hl-grid-strategy
```

确认价格正确后执行：
```bash
hyperliquid order-batch \
  --orders-json /tmp/btc-grid.json \
  --strategy-id hl-grid-strategy \
  --confirm
```

---

## 第五步：监控（每5分钟）

```bash
# 查看未成交挂单
hyperliquid orders --coin BTC

# 查看持仓与盈亏
hyperliquid positions

# 查看账户总资产（风控检查）
hyperliquid quickstart
```

**判断逻辑：**
- 有订单成交 → 执行第六步刷新
- 无成交 + 价格在区间内 → 继续等待
- 价格偏移超过±2.5% → 立即取消重新布局
- 账户净值 < $22 → 停止策略

---

## 第六步：刷新网格

```bash
# 1. 查询所有未成交订单ID
hyperliquid orders --coin BTC
# 记录所有 oid

# 2. 批量取消旧订单
hyperliquid cancel-batch \
  --coin BTC \
  --oids OID1,OID2,OID3,OID4,OID5,OID6,OID7,OID8,OID9,OID10 \
  --confirm

# 3. 检查净持仓，若偏斜 > 0.0006 BTC，部分平仓
hyperliquid positions
# 若需要平仓：
# hyperliquid close --coin BTC --size 0.0003 --strategy-id hl-grid-strategy --confirm

# 4. 重新获取价格，回到第三步布局
hyperliquid prices --coin BTC
```

---

## 安全退出

```bash
# 取消全部挂单
hyperliquid cancel-batch --coin BTC --oids <所有oid> --confirm

# 平掉所有持仓
hyperliquid close --coin BTC --strategy-id hl-grid-strategy --confirm

# 查看可提现余额后提现（注意：每次提现扣$1手续费，尽量一次提完）
hyperliquid positions
hyperliquid withdraw --amount <金额> --confirm
```

---

## 损耗预算

| 项目 | 金额 | 说明 |
|------|------|------|
| Maker手续费 | $0.004/笔 | $28 × 0.015% |
| 10笔一轮 | $0.04/轮 | 极低 |
| 30 USDC预算 | 可跑约750轮 | 约3750笔交易 |
| 提现手续费 | $1（仅结束时）| 一次性 |
| Arbitrum Gas | ~$0.01/次存取 | 几乎忽略 |

> 哲学小结：真正的损耗不是手续费，而是**不在市场里的时间**。
> 让网格持续运行，才是30 USDC的最高效使用方式。

---

## 比赛合规清单

| 要求 | 状态 |
|------|------|
| 使用 hyperliquid-plugin Basic Skill | ✅ |
| 通过 Agentic Wallet (onchainos) 发交易 | ✅ |
| 每笔订单含 `--strategy-id hl-grid-strategy` | ✅ |
| close 操作同样含 `--strategy-id` | ✅ |
| GitHub skill_id 与 plugin.yaml 一致 | ✅ |
| 已填写参赛表单 | ✅ |
