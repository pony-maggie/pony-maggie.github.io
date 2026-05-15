+++
title = "用 Rust 写一个工业级 POSP 支付系统"
date = 2026-05-15T20:40:00+08:00
draft = false
+++

### 先说结论


如果你在做支付系统，或者想了解怎么用 Rust 写一个生产级别的金融基础设施，那这篇文章会对你很有价值。


---

## 一、这个系统是干什么的

`posp-rust` 是一个用 Rust 编写的 **POSP（Payment Oriented Switch Platform）支付受理/交换系统**。

简单来说：它处于 POS 终端和上游支付网络之间，负责接收 POS 机的交易请求、解析协议报文、做业务路由、转发给银行或 QR 支付通道、记录流水。

核心能力列表：

- **CUP 2005/ISO8583** 报文解析与组包
- POS TCP 服务入口，支持终端发起交易请求
- 签到请求转发、签到响应工作密钥保存
- QR 支付交易：消费 `305`、撤销/关闭 `307`、查询 `309`
- 卡交易识别与初步处理：余额查询、消费、冲正、退货等 legacy 交易族
- 未知交易/兜底交易 `999` 的短应答与失败流水记录
- legacy 结算请求 `003` 的短应答兼容
- 商户参数下载、商户结算、对账、定时任务、分支机构签名密钥等 worker

---

## 二、架构全览

开源地址：https://github.com/pony-maggie/posp-rust

里面有架构图


从架构图来看，整条链路很清晰：

- **posp-server**：TCP listener，负责接收 POS 终端的连接，是整个系统的入口
- **posp-iso8583**：wire-level 协议解析，处理 CUP 2005 帧头、Bitmap、BCD 编码
- **posp-msg**：typed message model，把裸字节反序列化成交换结构体（`SignOnRequest`、`QrPaymentRequest`、`CardTransactionRequest` 等）
- **posp-business**：业务处理器，负责交易路由、风控校验、MAC 验证、事务逻辑
- **PostgreSQL**：存储交易流水、商户资料、密钥、分片路由信息

---

## 三、最底层：CUP 2005 帧的 wire-level 解析

先来看 `posp-iso8583` 这个 crate，它是整个系统的基石。

### 3.1 BCD 编解码

金融协议大量使用 BCD（Binary-Coded Decimal）编码来表示数字字段，比如金额、时间、流水号。BCD 的特点是每 byte 存储两个十进制 digit，高半字节存十位，低半字节存个位。

```rust
pub fn encode_bcd(digits: &str) -> Result<Vec<u8>> {
    let mut nibbles = Vec::with_capacity(digits.len());
    for ch in digits.chars() {
        let digit = ch.to_digit(10).ok_or(Iso8583Error::NonDigitBcdInput(ch))?;
        nibbles.push(digit as u8);
    }

    let mut bytes = Vec::with_capacity(nibbles.len().div_ceil(2));
    for pair in nibbles.chunks(2) {
        let high = pair[0] << 4;
        let low = pair.get(1).copied().unwrap_or(0);
        bytes.push(high | low);
    }
    Ok(bytes)
}
```

"123456" 编码后会变成 `[0x12, 0x34, 0x56]`。奇数位长度的 digit 串会 padding 低半字节为 0。

解码时需要检查每个 nibble 是否在 0-9 范围内——这是因为某些 nibble 可能出现 `0xA`-`0xF`，在协议里属于非法值。

### 3.2 Bitmap 解析

ISO8583 的 Bitmap 是一个 8 或 16 字节的位图，用来标记哪些域存在。第一个 bit（bit 0）如果为 1，表示还有第二个 bitmap（扩展到 16 字节）。

```rust
impl Bitmap {
    pub fn parse(input: &[u8]) -> Result<Self> {
        if input.len() < 8 {
            return Err(Iso8583Error::BitmapTooShort(input.len()));
        }

        let encoded_len = if input[0] & 0x80 != 0 { 16 } else { 8 };
        if input.len() < encoded_len {
            return Err(Iso8583Error::SecondaryBitmapTooShort(input.len()));
        }

        Ok(Self { bytes: input[..encoded_len].to_vec() })
    }

    pub fn fields(&self) -> Vec<u8> {
        let mut fields = Vec::new();
        for (byte_index, byte) in self.bytes.iter().enumerate() {
            for bit_index in 0..8 {
                if byte & (0x80 >> bit_index) != 0 {
                    let field = (byte_index * 8 + bit_index + 1) as u8;
                    if field != 1 {  // bit 0 of byte 0 是扩展标志，不是域号
                        fields.push(field);
                    }
                }
            }
        }
        fields
    }
}
```

### 3.3 CUP 2005 Header

CUP（中国银联）2005 协议在 ISO8583 基础上套了一层自定义帧头，13 字节：

```rust
pub struct Cup2005Header {
    pub total_message_len: usize,   // 报文总长度
    pub tpdu_id: u8,                // TPDU 路由 ID
    pub destination_id: [u8; 2],    // 目标机构代码
    pub source_id: [u8; 2],         // 源机构代码
    pub app_type: String,           // 应用类型（2 位 BCD）
    pub version: String,            // 版本号（2 位 BCD）
    pub terminal_status: char,     // 终端状态
    pub processing_request: char,   // 处理请求
    pub reserved: String,           // 保留字段
}

pub const LEN: usize = 13;
```

### 3.4 FieldSpec：灵活的域定义

一个好的协议解析框架必须能灵活描述各种域的编码方式和长度类型。`FieldSpec` 就是这个抽象：

```rust
pub struct FieldSpec {
    pub field: u8,
    pub length: FieldLength,
    pub encoding: FieldEncoding,
}

pub enum FieldLength {
    Fixed(usize),
    Llvar { max_len: usize },    // 2 位 BCD 长度前缀
    Lllvar { max_len: usize },   // 4 位 BCD 长度前缀
}

pub enum FieldEncoding {
    Bcd,
    Ascii,
    BinaryHex,
}
```

定义了 6 种构造器：

```rust
FieldSpec::fixed_bcd(3, 6);         // 域 3，固定 6 位 BCD
FieldSpec::llvar_bcd(32, 11);      // 域 32，LLVAR，max 11 位 BCD
FieldSpec::fixed_ascii(41, 8);     // 域 41，固定 8 字节 ASCII
FieldSpec::fixed_binary_hex(64, 8); // 域 64，固定 8 字节 BinaryHex
FieldSpec::lllvar_bcd(60, 999);    // 域 60，LLLVAR，max 999 位 BCD
FieldSpec::lllvar_ascii(62, 999); // 域 62，LLLVAR，max 999 ASCII
```

这种设计的优势在于：**新增交易类型时，只需要提供不同的 `FieldSpec` 数组，而不需要改解析引擎本身。**

---

## 四、Typed Message 层：从裸字节到业务结构体

`posp-msg` 依赖 `posp-iso8583`，把解析出来的裸字段组装成强类型的业务结构体。

### 4.1 消息类型枚举

```rust
pub enum PospMessage {
    SignOnRequest(SignOnRequest),
    SignOnResponse(SignOnResponse),
    QrPaymentRequest(QrPaymentRequest),
    CardTransactionRequest(CardTransactionRequest),
    FallbackRequest(FallbackRequest),
}
```

`TryFrom<&Iso8583Message>` 实现负责做类型转换，转换失败时会返回详细的 `MessageError`（比如 MTI 不匹配、缺少必填域）。

### 4.2 交易代码映射：协议的协议

最有意思的部分是交易代码的映射规则。QR 支付和卡交易是通过 **MTI + 处理码 + POS 条件码** 三元组来决定交易类型的：

```rust
fn qr_transaction_code(
    mti: &str,
    processing_code: &str,
    pos_condition_code: &str,
) -> Result<&'static str, MessageError> {
    match (mti, processing_code, pos_condition_code) {
        ("0200", "990019", "00" | "01") => Ok("305"),  // QR 消费
        ("0200", "990020", "00" | "01") | ("0400", "990019", "00" | "01") => Ok("307"), // QR 撤销
        ("0200", "990021", "00" | "01") => Ok("309"),  // QR 查询
        _ => Err(MessageError::UnsupportedQrTransactionMapping(...)),
    }
}
```

卡交易有自己的映射表，支持消费（101）、冲正（103）、退货（105）等一大串 legacy 交易族，映射到 `005`、`101`、`103`、`105` ... `125`。

如果一个报文既不匹配 QR 交易也不匹配卡交易，就会落入 **FallbackRequest（兜底请求）**，交易代码为 `999` 或 `003`（结算）。`999` 会记录失败流水，`003` 会返回短应答（不需要转发上游）。

### 4.3 签到响应的密钥块解析

签到响应（MTI=0810）中的 field 62 包含工作密钥块，解析后要存入 `BranchWorkingKeys`：

```rust
pub fn handle(&mut self, response: &SignOnResponse) -> Result<TransactionAction> {
    if response.response_code == "00" && let Some(reserved62) = &response.reserved62 {
        let key_block = BranchWorkingKeyBlock::parse(reserved62)
            .map_err(|err| PospError::Validation { field: "field_62", message: err.to_string() })?;
        self.key_repository.save_branch_working_keys(&BranchWorkingKeys {
            branch_code: response.acquiring_institution_id.clone(),
            zpk: key_block.zpk,
            zak: key_block.zak,
        })?;
    }
    // ...
}
```

---

## 五、业务处理层：路由与处理器链

`posp-business` 是整个系统的业务核心，定义了 `TransactionRouter` 和各种处理器。

### 5.1 路由决策

```rust
pub fn route(&self, transaction: &ParsedTransaction) -> Result<TransactionAction> {
    match transaction {
        ParsedTransaction::SignOnRequest(_) => Ok(TransactionAction::ForwardToCup { transaction_code: "001".to_string() }),
        ParsedTransaction::QrPaymentRequest(request) => Ok(TransactionAction::ForwardToQr { transaction_code: request.transaction_code.clone() }),
        ParsedTransaction::CardTransactionRequest(request) => Ok(TransactionAction::ForwardToCard { transaction_code: request.transaction_code.clone() }),
        ParsedTransaction::FallbackRequest(request) => Ok(TransactionAction::RespondToPos {
            transaction_code: request.transaction_code.clone(),
            response_code: if request.transaction_code == "003" { "00" } else { "06" }.to_string(),
        }),
    }
}
```

### 5.2 QrPaymentProcessor：完整的 QR 交易处理链

```rust
pub fn submit_order(
    &mut self,
    request: &QrPaymentRequest,
    context: &QrOrderContext,
    client: &impl QrOrderClient,
) -> Result<TransactionAction> {
    match request.transaction_code.as_str() {
        "305" => {
            // 1. 前置检查（MAC 验证、原交易状态）
            let precheck = self.handle(request)?;
            if !matches!(precheck, TransactionAction::ForwardToQr { .. }) {
                return Ok(precheck);
            }
            // 2. 构建 QR 上游订单请求
            let order_request = QrOrderRequest::payment(...);
            // 3. 转发到 QR 上游
            let response = client.post_order(QrOrderOperation::Payment, &order_request)?;
            // 4. 成功后更新原交易状态
            if response.is_success() {
                self.transaction_repository.update_qr_payment_status(...)?;
            }
            Ok(self.action_from_order_response(request, &response))
        }
        "307" => self.handle_reversal(request, context, client),
        "309" => self.handle_query(request, context, client),
        _ => self.handle(request),
    }
}
```

### 5.3 CardTransactionProcessor：支持多层的验证链

卡交易处理器支持**可插拔的 MAC 验证器和风控验证器**：

```rust
pub struct CardTransactionProcessor<
    R,
    M = AllowAllCardMerchantRepository,
    V = AllowAllCardMacValidator,
    K = AllowAllCardRiskValidator,
> {
    transaction_repository: R,
    merchant_repository: M,
    mac_validator: V,
    risk_validator: K,
}
```

验证链按顺序执行：商户状态检查 → MAC 校验 → 风控金额限制 → 业务权限校验，每一步失败都会立即返回对应的响应码。

### 5.4 HsmCardMacValidator

MAC 验证通过 HSM 完成：

```rust
impl<G> CardMacValidator for HsmCardMacValidator<G> where G: CardMacGenerator {
    fn validate_card_request_mac(&self, request: &CardTransactionRequest) -> Result<bool> {
        let Some(received_mac) = request.message_authentication_code.as_deref() else {
            return Ok(false);
        };
        let message_authentication_block = posp_msg::card_message_authentication_block(request)?;
        let generated_mac = self.generator.generate_card_mac(
            &self.terminal_authentication_key,
            &message_authentication_block,
        )?;
        Ok(generated_mac.eq_ignore_ascii_case(received_mac))
    }
}
```

---

## 六、数据库层：Repository 模式

`posp-db` 定义了所有持久化接口，并且提供了内存版本（用于测试）和 PostgreSQL 生产版本（通过 `postgres-driver` feature）。

核心 Repository 接口：

```rust
pub trait TransactionRepository {
    fn save_transaction(&mut self, txn: &StoredTransaction) -> Result<()>;
    fn find_original_qr_payment(&self, lookup: &OriginalQrPaymentLookup) -> Result<Option<StoredTransaction>>;
    fn update_qr_payment_status(&mut self, lookup: &OriginalQrPaymentLookup, status: &str) -> Result<bool>;
    fn update_qr_payment_reversal_status(&mut self, lookup: &OriginalQrPaymentLookup, processing_status: &str, reversal_status: &str) -> Result<bool>;
    fn find_original_card_transaction(&self, lookup: &OriginalCardTransactionLookup) -> Result<Option<StoredTransaction>>;
    fn update_card_transaction_status(&mut self, lookup: &OriginalCardTransactionLookup, processing_status: &str, response_code: &str) -> Result<bool>;
}
```

Repository 设计的好处：业务层完全不知道数据是存在内存还是 PostgreSQL，测试时可以用 `InMemoryTransactionRepository`，上线时切换到 `PostgresTransactionRepository`。

---

## 七、Worker 生态：不只是主服务

除了核心的 `posp-server`，项目还提供了多个独立运行的 Worker：

| App | 职责 |
|-----|------|
| `posp-param-service` | 商户参数下载 |
| `posp-settlement-worker` | 商户结算 |
| `posp-reconciliation-worker` | 对账业务 |
| `posp-sign-branch-key` | 分支机构签名密钥生成 |
| `posp-timer-service` | 定时调度任务 |
| `posp-signin` | 主动签到工具 |

这些 worker 都复用同一套 domain crates（`posp-param`、`posp-settlement`、`posp-reconciliation`），保证了业务逻辑的一致性。

---

## 八、性能基线

README 里给了一份本机冒烟压测数据（release 构建，PostgreSQL 本地，QR 上游为本地 mock）：

| 场景 | 并发 | 吞吐 | 平均延迟 | p95 |
|------|-----:|------:|------:|------:|
| `003` 结算短应答 | 50 | ~1000 rps | ~49 ms | ~61 ms |
| `999` 未知交易写库 | 30 | ~1050 rps | ~28 ms | ~32 ms |
| `305` QR 消费 DB+HTTP | 30 | ~310 rps | ~94 ms | ~117 ms |

QR 消费因为包含了数据库写入和 HTTP 上游调用，延迟明显更高。当前 `posp-server` 是**同步处理模型**，高并发场景主要体现为排队延迟。

---

## 九、总结

`posp-rust` 是一个**设计思路非常干净**的支付系统：

1. **协议层与业务层严格分离**：`posp-iso8583` 只负责 wire-level 编解码，`posp-msg` 做 typed 反序列化，业务层完全不知道协议细节
2. **扩展性通过 FieldSpec 和交易代码映射表实现**，新增交易类型不需要改核心引擎
3. **Repository 模式让测试和生产数据源可以替换**，单元测试不需要真实的 PostgreSQL
4. **可组合的验证链**（MAC 验证器、风控验证器）让不同商户可以配置不同的安全级别

Rust 的 ownership 和类型系统在这里发挥了很好的作用——金融协议解析对正确性要求极高，Rust 的 zero-cost abstraction 和编译期检查能让你在编译阶段就杜绝大量运行时错误。

---

> 欢迎关注收藏我，获取更多硬核技术干货 🚀