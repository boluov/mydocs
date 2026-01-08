# PICA Offramp 第一期开发 PRD（内部）

**文档版本**: v1.0  
**创建日期**: 2026-01-08  
**目标受众**: PICA 内部开发团队  
**项目负责人**: Tech Lead

---

## 0. 文档说明

本文档是面向 PICA 内部开发团队的实施 PRD，基于：
1. 现有 Demo 收银台实现（`/dashboard/offramp/demo`）
2. 产品需求文档（`offramp_prd_demo.md`）中的 API 设计

**第一期目标**：实现最小可用产品（MVP），支持法币机构通过 API 集成 Offramp 功能。

---

## 1. 项目概述

### 1.1 第一期范围

根据产品前言，第一期只实现以下功能：

| 功能模块           | 第一期范围                    | 后续扩展                           |
| ------------------ | ----------------------------- | ---------------------------------- |
| **OTC 对接模式**   | 仅模式 B（DTCpay 模式）       | 后续支持模式 A（PICA 生成地址）    |
| **收银台集成方式** | 仅 Web Iframe                 | 后续支持移动端重定向               |
| **下单金额类型**   | 仅固定 crypto 金额（PAYMENT） | 后续支持固定法币金额（SETTLEMENT） |
| **法币机构数量**   | 单一法币机构（JPay）          | 后续支持多法币机构                 |

### 1.2 技术架构

```
┌─────────────────────────────────────────────────────────────┐
│                    法币机构网站（JPay）                        │
│  - 调用 PICA API 创建订单                                      │
│  - 嵌入 Iframe 收银台                                         │
│  - 接收 Webhook 通知                                          │
└────────────────────┬────────────────────────────────────────┘
                     │ API 调用
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                      PICA Backend                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ API Gateway  │  │ Offramp Core │  │ Webhook      │      │
│  │ - 鉴权       │  │ - 订单管理    │  │ - 异步通知    │      │
│  │ - 限流       │  │ - 状态机     │  │ - 重试机制    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                          │                                   │
│                          ▼                                   │
│                  ┌──────────────┐                            │
│                  │ OTC Adapter  │                            │
│                  │ - DTCpay集成 │                            │
│                  └──────────────┘                            │
└────────────────────┬────────────────────────────────────────┘
                     │ OTC API 调用
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    DTCpay OTC 服务                           │
│  - 下单接口                                                   │
│  - 查询接口                                                   │
│  - Webhook 回调                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 后端开发任务

### 2.1 API 接口实现

#### 2.1.1 创建出金会话（Init Session）

**接口**: `POST /v1/session/init`

**功能**: 法币机构调用此接口创建 Offramp 订单并获取收银台 URL

**请求参数**:
```typescript
interface InitSessionRequest {
  externalId: string;      // 法币机构侧订单号，必填
  customerId: string;      // 商户 ID，必填
  otcId: string;          // OTC 机构代码（第一期固定为 DTCpay），必填
  amount: string;         // 第一期：固定为 crypto 金额（如 "1000"），必填
  amountType: 'PAYMENT';  // 第一期：固定为 PAYMENT
  callbackUrl: string;    // Webhook 回调地址，必填
  returnUrl?: string;     // 支付完成后跳转地址，选填
}
```

**响应**:
```typescript
interface InitSessionResponse {
  code: number;
  data: {
    systemOrderNo: string;     // PICA 系统订单号
    merchantOrderNo: string;   // 法币机构订单号
    paymentUrl: string;        // 收银台 URL
    status: 'pendingPayment';  // 初始状态
  }
}
```

**实现要点**:
1. 生成唯一的 `systemOrderNo`（格式：`GIM-YYYYMMDD-XXXX`）
2. 生成带 token 的 `paymentUrl`（格式：`/checkout?token={encrypted_token}`）
3. Token 需加密包含：`systemOrderNo`, `otcId`, `amount`, `amountType`, `customerId`
4. 订单初始状态为 `CREATED`
5. 验证 `otcId` 是否为有效的 OTC 机构

#### 2.1.2 获取订单详情

**接口**: `GET /v1/orders/{systemOrderNo}`

**响应**:
```typescript
interface OrderDetailsResponse {
  code: number;
  data: {
    orderSummary: {
      merchantOrderNo: string;
      systemOrderNo: string;
      customerId: string;
      status: OrderStatus;
      requestAmount: number;
      requestAmountType: 'PAYMENT';
      cryptoAmount: number;    // 应付数币金额
      depositAsset: string;    // USDT/USDC
      createdAt: string;
    };
    // 收币信息（如已生成地址）
    fiatInstCollection?: {
      asset: string;
      network: string;
      address: string;
      actualReceived?: number;
      completedAt?: string;
    };
    // OTC 信息（如已下单）
    otcChannelDeposit?: {
      otcId: string;
      otcName: string;
      orderNo: string;         // OTC 订单号
      status: 'pending' | 'completed' | 'failed';
    };
    // 兑换信息（如已兑换）
    otcExchange?: {
      otcRate: number;
      fiatAmount: number;
      fiatCurrency: 'USD';
      exchangeTime?: string;
    };
  }
}
```

订单状态机已完成实施）下单接口

**接口**: `POST /checkout/generate-address`

**功能**: 收银台调用，用户选择参数后生成充币地址

**请求参数**:
```typescript
interface GenerateAddressRequest {
  token: string;          // Session token
  currency: string;       // USDT/USDC
  network: string;        // TRC20/ERC20/Polygon等
  amount: string;         // 第一期固定为 crypto 金额
}
```

**响应**:
```typescript
interface GenerateAddressResponse {
  code: number;
  data: {
    address: string;        // 充币地址（来自 DTCpay）
    network: string;
    currency: string;
    amount: number;
    expiryTime: string;     // 30分钟后过期
    otcOrderNo: string;     // OTC 订单号
  }
}
```

**实现要点**:
1. 解密并验证 token
2. 调用 DTCpay 下单接口，传递：
   - 币种（currency）
   - 网络（network）
   - 金额（amount）
   - 回调地址（PICA 的 webhook URL）
3. DTCpay 返回充币地址和订单号
4. 更新 PICA 订单状态为 `INBOUND_PENDING`
5. 设置订单过期时间（30分钟）
6. **同时调用法币机构的全球收款申请接口**（新增需求）

#### 2.1.4 取消订单

**接口**: `POST /v1/orders/{systemOrderNo}/cancel`

**请求参数**:
```typescript
interface CancelOrderRequest {
  reason: string;
}
```

**响应**: 返回更新后的订单状态

---

### 2.2 OTC 集成（DTCpay Adapter）

#### 2.2.1 下单接口封装

封装 DTCpay 的下单接口，参考产品文档中的"模式 B"流程。

**调用时机**: 用户在收银台点击"获取充值地址"

**调用参数** (based on DTCpay API):
```typescript
interface DTCPayCreateOrderRequest {
  merchantOrderNo: string;   // PICA 系统订单号
  amount: string;
  currency: string;         // USDT/USDC
  network: string;
  callbackUrl: string;      // PICA 的 webhook endpoint
}
```

**返回**: 充币地址 + DTCpay 订单号

#### 2.2.2 Webhook 接收

实现接收 DTCpay 的 Webhook 通知：

**事件类型**:
1. `deposit.received` - 收到用户打币
2. `exchange.completed` - 兑换完成（自动转USD）
3. `order.failed` - 订单失败

**处理逻辑**:
1. 验证签名
2. 根据事件类型更新 PICA 订单状态
3. 触发对法币机构的 Webhook 通知
4. 实现幂等性处理

---

### 2.3 Webhook 通知（发送给法币机构）

#### 2.3.1 通知事件

**1. order.cryptoReceived**
```typescript
{
  event: 'order.cryptoReceived',
  data: {
    systemOrderNo: string;
    merchantOrderNo: string;
    status: 'confirming';
    actualReceived: number;
    asset: string;
  }
}
```

**2. order.usdReady**（重点）
```typescript
{
  event: 'order.usdReady',
  data: {
    systemOrderNo: string;
    merchantOrderNo: string;
    status: 'usdReady';
    otcId: string;
    exchangeRate: number;
    usdAmount: number;
    currency: 'USD';
    readyTime: string;
  }
}
```

> **重要**: 根据版本对比表格，PICA 需在此时通知 JPay 换汇成功，JPay 工作人员据此去 OTC 提现。

**3. order.exception**
```typescript
{
  event: 'order.exception',
  data: {
    exceptionType: 'amountMismatch' | 'timeout' | 'otcFailed';
    expectedAmount: number;
    actualReceived?: number;
    status: 'exception';
  }
}
```

#### 2.3.2 重试机制

1. 最多重试 5 次
2. 重试间隔：1min, 2min, 5min, 10min, 30min
3. 全部失败后记录日志并发送告警

---

### 2.4 数据库设计

#### 2.4.1 offramp_orders 表

```sql
CREATE TABLE offramp_orders (
  id BIGSERIAL PRIMARY KEY,
  system_order_no VARCHAR(50) UNIQUE NOT NULL,
  merchant_order_no VARCHAR(100) NOT NULL,
  customer_id VARCHAR(100) NOT NULL,
  otc_id VARCHAR(50) NOT NULL,
  
  -- 金额信息
  request_amount DECIMAL(18,6) NOT NULL,
  request_amount_type VARCHAR(20) DEFAULT 'PAYMENT',
  crypto_amount DECIMAL(18,6),
  fiat_amount DECIMAL(18,6),
  
  -- 状态
  status VARCHAR(50) NOT NULL,
  
  -- 收币信息
  deposit_asset VARCHAR(10),
  deposit_network VARCHAR(20),
  deposit_address VARCHAR(100),
  actual_received DECIMAL(18,6),
  
  -- OTC 信息
  otc_order_no VARCHAR(100),
  otc_status VARCHAR(50),
  exchange_rate DECIMAL(10,6),
  
  -- Webhook
  callback_url VARCHAR(500) NOT NULL,
  return_url VARCHAR(500),
  
  -- 时间
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  expired_at TIMESTAMP,
  
  -- 重试次数
  webhook_retry_count INT DEFAULT 0,
  
  INDEX idx_system_order_no (system_order_no),
  INDEX idx_merchant_order_no (merchant_order_no),
  INDEX idx_customer_id (customer_id),
  INDEX idx_status (status),
  INDEX idx_created_at (created_at)
);
```

#### 2.4.2 webhook_logs 表

```sql
CREATE TABLE webhook_logs (
  id BIGSERIAL PRIMARY KEY,
  order_id BIGINT NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  target_url VARCHAR(500) NOT NULL,
  request_payload JSON NOT NULL,
  response_status INT,
  response_body TEXT,
  retry_count INT DEFAULT 0,
  success BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  INDEX idx_order_id (order_id),
  INDEX idx_event_type (event_type),
  INDEX idx_created_at (created_at)
);
```

---

## 3. 前端开发任务

### 3.1 收银台页面（`/checkout`）

基于现有 Demo 实现（`/dashboard/offramp/demo/page.tsx`），创建生产收银台。

#### 3.1.1 Token 验证

**路由**: `/checkout?token={encrypted_token}`

**流程**:
1. 从 URL 获取 token
2. 调用后端接口验证 token 并获取订单信息
3. 如果 token 失效，显示错误页面

#### 3.1.2 配置步骤（Step 0）

**UI 元素** (参考 Demo 的 `renderIframeConfig`):
- 币种选择（USDT/USDC）
- 网络选择（TRC20/ERC20/Polygon/BSC/Arbitrum/Optimism/Base/Avalanche）
- 金额显示（第一期：金额已预设，灰色不可编辑）
- 实时汇率显示
- 预计到账金额
- "获取充值地址"按钮

**数据源**:
- 从 token 解析得到预设金额
- 从后端 API 获取可用币种和网络
- 从后端 API 获取实时汇率

**交互**:
- 用户选择币种 → 更新可用网络列表
- 用户选择网络 → 更新汇率
- 点击"获取充值地址" → 调用 `/checkout/generate-address` → 进入支付步骤

#### 3.1.3 支付步骤（Step 1）

**UI 元素** (参考 Demo 的 `renderIframePayment`):
- 返回按钮
- 倒计时（30分钟）
- 警告提示：必须使用{network}网络
- 应付金额（大字号显示）
- QR 码（包含地址和金额）
- 充币地址（可复制）
- 订单号
- 支付状态指示器

**支付状态**:
- `waiting` - 等待用户打币（黄色Loading）
- `confirming` - 链上确认中（蓝色Loading）
- `received` - 已收到完整金额（绿色Success）
- `partial` - 部分支付（橙色Warning，显示已收/还需金额）
- `overpaid` - 多付（橙色Warning，显示超付金额，60秒后自动跳转）
- `expired` - 订单超时（红色Error）

**数据更新**:
- WebSocket 或轮询（每5秒）查询订单状态
- 收到状态更新后，更新 UI

#### 3.1.4 成功页面（Step 2）

**UI 元素** (参考 Demo 的 `renderIframeSuccess`):
- 成功图标（绿色勾）
- 成功文案
- 订单详情卡片：
  - 订单号
  - OTC 服务商
  - 已收金额
  - 实时汇率
  - 预计到账金额
- "完成"按钮（跳转到 returnUrl 或关闭 iframe）

---

### 3.2 Iframe 集成支持

#### 3.2.1 PostMessage 通信

收银台需要通过 postMessage 与父窗口通信：

**1. 高度更新**
```typescript
window.parent.postMessage({
  type: 'PICA_HEIGHT_UPDATE',
  height: document.body.scroll使用Height
}, '*');
```

**2. 关闭收银台**
```typescript
window.parent.postMessage({
  type: 'PICA_CHECKOUT_CLOSE',
  reason: 'user_cancel' | 'payment_complete'
}, '*');
```

**3. 支付成功**
```typescript
window.parent.postMessage({
  type: 'PICA_PAYMENT_SUCCESS',
  orderNo: string
}, '*');
```

#### 3.2.2 响应式设计

- Desktop: 420px宽 × 560px高
- Mobile: 100%宽 × 100%高（全屏）

---

## 4. 测试计划

### 4.1 单元测试

- [ ] API 接口参数验证
- [ ] Token 加密/解密
- [ ] 订单状态机转换
- [ ] Webhook 签名验证
- [ ] 金额计算逻辑

### 4.2 集成测试

- [ ] DTCpay 下单流程
- [ ] DTCpay Webhook 接收
- [ ] 法币机构 Webhook 发送
- [ ] 完整支付流程（正常收币）
- [ ] 异常流程（部分支付/超时/多付）

### 4.3 E2E 测试

基于 Demo 模拟器，测试完整流程：
1. 法币机构调用 Init Session
2. 打开收银台 Iframe
3. 用户选择参数并生成地址
4. 模拟用户打币（完整/部分/超额）
5. 验证状态更新和 Webhook 通知

---

## 5. 实施计划

### Phase 1: 后端核心（Week 1-2）

- [ ] **Week 1.1**: API Gateway + Init Session
- [ ] **Week 1.2**: Generate Address + DTCpay 集成
- [ ] **Week 1.3**: Order Status API
- [ ] **Week 1.4**: Webhook发送模块
- [ ] **Week 2.1**: DTCpay Webhook 接收
- [ ] **Week 2.2**: 订单状态机实现
- [ ] **Week 2.3**: 数据库设计与迁移
- [ ] **Week 2.4**: 单元测试

### Phase 2: 前端收银台（Week 3）

- [ ] **Week 3.1**: Token 验证 + 路由
- [ ] **Week 3.2**: 配置步骤UI
- [ ] **Week 3.3**: 支付步骤UI + 状态更新
- [ ] **Week 3.4**: 成功页面 + PostMessage

### Phase 3: 集成测试（Week 4）

- [ ] **Week 4.1**: 内部集成测试
- [ ] **Week 4.2**: 与 DTCpay 联调
- [ ] **Week 4.3**: 与 JPay 联调
- [ ] **Week 4.4**: Bug 修复

### Phase 4: 部署上线（Week 5）

- [ ] **Week 5.1**: Staging 环境部署
- [ ] **Week 5.2**: 性能测试 + 安全review
- [ ] **Week 5.3**: Production 部署
- [ ] **Week 5.4**: 监控 + 文档

---

## 6. 技术细节

### 6.1 安全性

1. **API 鉴权**: Bearer Token (JWT)
2. **Token 加密**: AES-256
3. **Webhook 签名**: HMAC-SHA256
4. **HTTPS**: 强制 TLS 1.2+
5. **Rate Limiting**: 100 req/min per API key

### 6.2 性能要求

- API 响应时间: P95 < 500ms
- Webhook 发送延迟: < 5s
- 收银台加载时间: < 2s
- 订单状态更新延迟: < 10s

### 6.3 监控与告警

**关键指标**:
1. API 成功率 > 99.9%
2. Webhook 成功率 > 99%
3. 订单完成率
4. 平均处理时长

**告警规则**:
- API 错误率 > 1% 持续 5min
- Webhook 重试失败 > 10 次
- DTCpay 调用失败 > 5 次/10min
- 订单长时间卡在某状态（> 1小时）

---

## 7. 已知限制与后续优化

### 7.1 第一期限制

1. **仅支持 DTCpay**: 后续需支持多 OTC 机构
2. **仅支持固定 crypto 金额**: 后续需支持固定法币金额（SETTLEMENT）
3. **仅支持 Iframe**: 后续需支持移动端重定向
4. **单一法币机构**: 后续需支持多法币机构管理

### 7.2 技术债务

1. **缺少订单归档机制**: 建议后续将完成订单归档到历史表
2. **Webhook 重试策略简单**: 后续可引入消息队列
3. **缺少订单搜索**: 后续需加入 ElasticSearch
4. **缺少实时汇率缓存**: 建议引入 Redis

---

## 8. 附录

### 8.1 Demo vs Production 差异

| 功能         | Demo 实现               | Production 需求               |
| ------------ | ----------------------- | ----------------------------- |
| OTC 选择     | 下拉框可选              | 第一期固定 DTCpay             |
| 金额类型     | 可选 PAYMENT/SETTLEMENT | 第一期固定 PAYMENT            |
| 设备类型     | 可选 Mobile/Desktop     | 第一期仅 Desktop Iframe       |
| 预设金额     | 可选填                  | 第一期必填，收银台不可修改    |
| 模拟控制     | 有模拟按钮              | Production 无，真实OTC回调    |
| 全球收款申请 | 无                      | Production 需在生成地址时调用 |

### 8.2 关键决策点

1. **Q**: 订单过期后用户打币怎么处理？  
   **A**: 参考 DTCpay 策略，线下退款

2. **Q**: 部分支付是否接受？  
   **A**: 给法币机构后台"接受抹平"功能，允许接受实收金额继续流程

3. **Q**: 多付如何处理？  
   **A**: 按下单金额结算，超付部分线下退款

4. **Q**: 汇率波动导致到账金额不符？  
   **A**: 第一期按实际汇率结算，显示预计到账金额仅供参考

---

## 9. 参考文档

1. **产品 PRD**: `docs/offramp_prd_demo.md`
2. **Demo 实现**: `src/app/dashboard/offramp/demo/page.tsx`
3. **DTCpay API 文档**: (需补充链接)
4. **JPay 全球收款 API**: (需补充链接)
