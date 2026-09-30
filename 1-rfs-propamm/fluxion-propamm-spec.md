# Fluxion RFQ & Prop AMM 系统架构与机制规范

> **版本**：v1.0  
> **归属模块**：现货交易层（Spot Trading Layer）  
> **关联决策**：D4（Bybit W2 做市库存对接）、7.4 现货交易层设计

---

## 1. 系统总体架构

Fluxion RFQ & Prop AMM 采用典型的「链下高频撮合定价 + 链上原子清算交割」双层架构：

```
┌────────────────────────────────────────────────────────────────────────┐
│                        前端与路由集成层                                 │
│    Tape Terminal   │  Bybit Alpha 白标端  │  聚合器 (OKX / Kyber / LI.FI)  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 1. 提交 RFQ 询价单 (Taker)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Fluxion Off-chain RFQ 撮合集群                        │
│                                                                        │
│   ┌─────────────────────┐    ┌─────────────────────┐                   │
│   │ Bybit W2 做市引擎   │    │ 外部合作 MM 节点     │                   │
│   │ (CEX 现货/永续联动) │    │ (Hashflow/Wintermute)│                  │
│   └──────────┬──────────┘    └──────────┬──────────┘                   │
│              │                          │                              │
│              └────────────┬─────────────┘                              │
│                           │ 2. 返回带时效签名报价 (EIP-712 Maker Quote)  │
│                           ▼                                            │
│              ┌───────────────────────────┐                             │
│              │ Best Quote Arbiter (比价) │                             │
│              └────────────┬──────────────┘                             │
└───────────────────────────┼────────────────────────────────────────────┘
                            │ 3. 组装完整交易并签名
                            ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      Mantle L2 Sequencer 排序层                        │
│   - Cancel 指令优先通道 (Cancel Priority FIFO)                          │
│   - 无公开 Mempool 保护 (Anti-Sandwich by Design)                      │
│   - 200ms 预确认流广播                                                 │
└───────────────────────────┬────────────────────────────────────────────┘
                            │ 4. 交易打包执行
                            ▼
┌────────────────────────────────────────────────────────────────────────┐
│                FluxionSettlement.sol (链上原子结算合约)                 │
│                                                                        │
│   ┌───────────────────────────┐    ┌───────────────────────────────┐   │
│   │ 签名验证 & 期限/Nonce 检查 │ ──►│ 资产双向安全转移 (TransferFrom)│   │
│   └───────────────────────────┘    └───────────────┬───────────────┘   │
│                                                    │                   │
│                                                    ▼                   │
│                                    ┌───────────────────────────────┐   │
│                                    │ 协议手续费归集 (Protocol Fee) │   │
│                                    └───────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. EIP-712 结构化订单签名规范

为了杜绝滑点与抢跑，做市商报价采用不可篡改的 EIP-712 强类型签名：

```solidity
struct RFQQuote {
    address maker;          // 做市商地址 (Bybit MM / 合作机构)
    address taker;          // 用户地址 (指定地址或零地址允许任意吃单)
    address makerAsset;     // 卖出资产合约 (如 NVDAx, MNT, USDT)
    address takerAsset;     // 买入资产合约 (如 USDT, WETH)
    uint256 makerAmount;    // 卖出数量
    uint256 takerAmount;    // 买入数量
    uint256 nonce;          // 做市商唯一 Nonce (用于单笔/批量失效)
    uint256 expiry;         // Unix 时间戳，报价绝对过期时间 (通常 5-15s)
    bytes32 marketSessionId;// 市场日历时段 ID (开市/休市/盘前)
}
```

- **类型哈希（TypeHash）**：
  ```solidity
  bytes32 constant RFQ_QUOTE_TYPEHASH = keccak256(
      "RFQQuote(address maker,address taker,address makerAsset,address takerAsset,uint256 makerAmount,uint256 takerAmount,uint256 nonce,uint256 expiry,bytes32 marketSessionId)"
  );
  ```

---

## 3. Bybit W2 做市库存接入机制

### 3.1 做市库存池（Bybit Prop Liquidity Vault）
- Bybit 自营盘在 Mantle 链上部署专属白名单热钱包阵列，质押或存入底层资产储备（BTC、ETH、MNT、USDT、USDC 以及 155 种 xStocks）。
- 做市商引擎通过高速 WebSocket 接收 Mantle RFQ 询价事件，并在 50ms 内根据 Bybit CEX 实时 Orderbook 深度与价差生成 EIP-712 签名。

### 3.2 资金效率与再平衡（Rebalancing）
- **净头寸对冲**：链上累计成交的单向敞口达到预设阈值（例如 $100K）时，做市程序在 Bybit CEX 毫秒级对冲相反头寸。
- **UTA 保证金打通（与 W4 联动）**：链上库存地址直接作为 Bybit 统一交易账户（UTA）的子账户抵押物，极大减少资本占用。

---

## 4. 开闭市双模状态机（Market Session State Machine）

代币化股票具有不同于通用加密货币的外生美股作息时间。Fluxion 必须在协议层根据链上市场日历（On-chain Market Calendar Oracle）切换交易模式：

| 市场状态 | 触发条件 | 做市报价行为 | 安全限制 |
|---|---|---|---|
| **Regular Session (正常开市)** | 美股正常开盘（美东 09:30–16:00） | 紧密 Spread（1–3 bps），高频刷新，单笔支持 $500K+ | 无额外限制，极速成交 |
| **Extended Hours (盘前盘后)** | 美股盘前（04:00–09:30）与盘后（16:00–20:00） | Spread 适度放宽（5–15 bps），匹配美股电子盘流动性 | 单笔交易限额 $50K |
| **Closed / Weekend (周末及休市)** | 周末、美股节假日或全天休市 | 做市商停止纯现货裸报价，切入 AMM 缓冲模式或仅提供保守双边流动性 | 启用预言机偏离度熔断 Hook，偏离 >5% 禁止大额成交 |

---

## 5. 链级基础设施协同保障

### 5.1 无公开 Mempool 承诺
- 普通 EVM 链做市商经常因公共交易池泄露报价被 Sandwich 攻击。
- Mantle 的 Sequencer 直接对接 Fluxion 节点。交易在提交到打包期间不对全网广播，完全消除前端套利者的观察窗口。

### 5.2 做市商快速撤单通道（Cancel Priority）
- 当标的资产（如 NVDA 突发财报剧烈异动）瞬间跳空时，做市商需在旧报价被吃单前撤销。
- Mantle Sequencer 设立 **Cancel 专用优先调度逻辑**：
  1. 标记为 `CancelRFQNonce` 的交易拥有区块内优先执行权；
  2. 支持批量 Nonce 废弃：做市商只需更新其合约记录的 `minValidNonce = currentNonce + 100`，之前的所有签名瞬间失效，耗费极低 gas。

---

## 6. 聚合器集成方案

Fluxion RFQ 协议已封装兼容主流 DEX 聚合器的通用适配器（Adapter）：
- **OKX DEX 接入**：实现 OKX 官方 RFQ Provider 标准，暴露 `/rfq/quote` 与 `/rfq/order` 接口。
- **KyberSwap 接入**：支持 KyberSwap RFQ API 协议，作为高优先级流动性源参与动态路径分割。
- **LI.FI 跨链接入**：支持从其他链跨链 Swap 至 Mantle 并在目的地由 Fluxion 一步完成兑换。
