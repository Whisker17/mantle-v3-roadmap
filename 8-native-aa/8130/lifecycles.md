上篇有一个讲解上的问题：**从“账户已经配置好”开始讲交易，跳过了 Alice 如何拥有这个账户、第一把钥匙如何登记，以及链为什么认可这把钥匙。**

我们这次沿着 Alice 的实际操作走一遍：

**创建账户 → Alice 亲自买入 → Alice 授权 Agent → Agent 自己买入。**

先记住最重要的一句话：

> **8130 把“资产属于哪个账户”“哪把钥匙有权操作”“谁支付手续费”拆开了。它们可以属于不同主体。**  
> Account 是资产归属的账户；Actor 是被这个账户授权的操作身份；Authenticator 是验证该身份签名的方法。:chatgpt-content-reference{index="0"}

以下是假设一条链已经支持 8130，并有配套钱包和代付服务的教学场景，不是 Mantle 已上线功能说明。8130 截至 **2026 年 9 月 29 日**仍为 Draft；开户细节补充依据本次核对的官方规范与参考合约。:chatgpt-content-reference{index="1"}

---

# 一、Alice 怎样创建一个 8130 AA 账户？

先选择一条具体路径：**Alice 创建一个新账户，使用手机 Passkey 控制它，平台替她支付开户 Gas。**

已有 EOA 原地址使用 8130 是另一条路径，后面单独解释，先不混在一起。

## 第一步：Alice 点击“创建账户”，生成一把控制账户的钥匙

假设产品界面显示：

> 创建交易账户  
> 使用 Passkey

Alice 在手机上确认，Passkey 系统生成签名凭证。钱包获取它的公钥信息，并由此确定一个标识，我们叫它：

```text
Alice 的 Passkey
    ↓ 对应
Actor ID：alice-key
```

`alice-key` 是为了方便阅读取的名字。实际 Passkey Actor ID 是根据公钥计算出的 32 字节标识；Passkey Authenticator 验证签名后，会返回这个标识。:chatgpt-content-reference{index="2"}

**此时只是有了钥匙，还没有完成链上开户。**

这里要特别区分：

| 东西 | 是什么 |
|---|---|
| Alice 的 Passkey | 能够对交易签名的凭证 |
| `alice-key` | 链上识别这把凭证的 Actor ID |
| Alice 的 AA 账户地址 | 接下来创建、用来持有资产的地址 |

**Actor ID 不是她接收 USDC 的账户地址。**

## 第二步：钱包准备账户代码、初始权限，并算出账户地址

这些由钱包软件完成，Alice 不需要手动部署合约或填写字段。

钱包准备三类信息：

```text
账户代码：
    这个账户采用哪一种钱包实现。

初始控制者：
    alice-key
    验签方法 = Passkey Authenticator
    权限 = ADMIN，即账户管理员

区分因子：
    一个 salt，用来参与地址计算。
```

然后计算出未来的账户地址，记作 **`A`**。

可以把这个计算理解成：

```text
账户代码 + 初始 Actor 配置 + salt
                  ↓
             账户地址 A
```

这是简化表达，实际使用 CREATE2 地址派生规则。参考 Keystore 提供 `computeAddress()`，可以在部署之前计算地址；初始 Actor 的身份、认证器和权限都会参与地址承诺。:chatgpt-content-reference{index="3"}

这意味着：

> **不是先找到“地址 A 对应的 EOA 私钥”，再用它控制 A。**  
> **而是创建一个地址为 A 的账户，并规定：Alice 的 Passkey 有权控制 A。**

这是理解新建 8130 账户最关键的一步。

## 第三步：钱包构造一笔“创建账户”的 AA 交易

钱包联系代付方，确定由地址 **`P`** 支付开户费用，然后准备如下交易。

下面省略 nonce、费用和有效期等字段，只保留与开户有关的部分：

```text
交易类型：8130 AA 交易

sender：
    A，即将创建的账户地址

account_changes：
    Create：
        账户代码
        salt
        初始 Actor：
            alice-key
            Passkey Authenticator
            ADMIN

calls：
    本例为空，只开户，不做其他业务

payer：
    P，平台的 Gas 付款账户

sender_auth：
    Alice 用 Passkey 对这笔交易生成的签名及认证数据

payer_auth：
    平台对承担这笔交易 Gas 的付款授权
```

8130 的 Create 项放在 `account_changes` 中，由交易的 `sender_auth` 授权，不需要再有一份独立的 Create 授权。账户实现需要额外初始化时，可以通过后面的 `calls` 完成；本例假设账户实现不需要额外初始化。:chatgpt-content-reference{index="4"}

对 Alice 来说，她做的是：

> **确认使用这把 Passkey 创建这个账户。**

平台做的是：

> **确认愿意为这笔开户交易支付手续费。**

两份授权的意思不同。**平台同意付费，并不因此获得 Alice 账户的控制权。**

## 第四步：节点先验证，再真正创建账户

你可能会问：

> **A 还没创建，Keystore 里也没有 Alice 的权限记录，节点怎么验证她？**

答案是：

**首次创建时，节点使用交易中携带的 `initial_actors`，作为这次开户的初始权限配置。**

节点会核对：创建参数是否确实导出地址 A、A 是否满足新建条件，以及 Alice 的签名是否对应初始 Actor 列表中的授权身份。通过验证之后，才在交易执行时部署账户并登记权限。:chatgpt-content-reference{index="5"}

可以这样理解：

```text
普通后续交易：
    去链上查“这个账户已经授权了谁”。

首次开户交易：
    验证“这个新账户将授权谁”，
    并检查地址、创建参数与签名是否一致。
```

账户创建完成后，链上的状态大致是：

```text
账户 A：
    已有账户代码
    可以接收和持有资产

共享 Keystore：
    对于账户 A：
        alice-key → Passkey Authenticator → ADMIN
```

参考 Keystore 的创建逻辑会初始化账户及 Actor 配置，并部署对应账户代码。:chatgpt-content-reference{index="6"}

### 到这里，Keystore 到底是什么？

**它是链上的权限登记簿，不是替 Alice 保存私钥的地方，也不是资产托管池。**

它记录的是：

> 对于账户 A，哪一个 Actor 有权操作，采用哪个 Authenticator 验签，以及权限和有效期是什么。

Alice 不需要为自己单独部署一个 Keystore；账户使用共享 Keystore 中属于自己的记录。:chatgpt-content-reference{index="7"}

开户本身也不意味着获得了交易资金。下面假设 Alice 随后向 A 转入了 **1,000 USDC**。

---

# 二、Alice 亲自买入：交易怎么从点击按钮走到链上执行？

现在的状态是：

```text
账户 A：持有 1,000 USDC
控制钥匙：Alice 的 Passkey
权限：ADMIN
Gas 付款方：平台 P
```

Alice 想在 Launchpad 花 **100 USDC** 买入 MEME。

## 第一步：Alice 点击“买入”，钱包组织业务操作

假设 Launchpad 需要先获得 USDC 授权，钱包需要安排两件事：

```text
操作 1：允许 Launchpad 使用 A 的 100 USDC。
操作 2：在 Launchpad 买入 MEME，接收地址为 A。
```

在普通 EOA、没有已有 allowance、也没有额外 Permit 或批处理机制的基线下，这通常是两笔独立交易，EOA 自己签名和支付 Gas。普通交易的签名地址与 Gas 扣款账户是同一个主体。:chatgpt-content-reference{index="8"}

在 8130 中，可以把这两件事放进**同一笔交易的同一个 Phase**。

Phase 先理解为：

> **一组必须共同成功，否则共同撤销的调用。**:chatgpt-content-reference{index="9"}

## 第二步：钱包让 Alice 签署这笔交易

钱包组织的核心内容变成：

```text
sender：
    A

account_changes：
    []，账户已经创建，不修改权限

calls：
    Phase 0：
        USDC.approve(Launchpad, 100)
        Launchpad.buy(100, 最低买入数量, 接收地址 A)

payer：
    P

sender_auth：
    Alice 的 Passkey 签名

payer_auth：
    平台的付款授权
```

`buy()` 是教学示例接口，不是 8130 定义的标准函数。

注意，此时三种身份分别是：

```text
使用谁的资产？       A
谁授权本次操作？     Alice 的 Passkey
谁付 Gas？          P
```

钱包还会读取当前 nonce、设置有效期和费用上限。Alice 确认的是完整交易内容，而不是只签“买入”两个字；平台也对这笔具体交易提供付款授权。:chatgpt-content-reference{index="10"}

## 第三步：节点验证“这把钥匙能不能使用 A”

此时已经不是首次开户，所以节点可以使用链上 Keystore 的现有记录。

用问答来表达就是：

**Authenticator 回答：**

> 这个签名确实来自 `alice-key`。

**Keystore 中的配置回答：**

> `alice-key` 是账户 A 登记的 Actor，使用的认证器匹配，而且拥有 ADMIN 权限。

**协议据此判断：**

> 这个 Actor 有权代表 A 发起这些调用。

这就是 **“验证签名”与“检查权限”分开**的含义：Authenticator 负责确定签名身份，账户的权限配置负责决定这个身份能做什么。:chatgpt-content-reference{index="11"} :chatgpt-content-reference{index="12"} :chatgpt-content-reference{index="13"}

节点另外验证平台的付款授权、余额，以及交易的 nonce 和有效期。交易有效并被纳入后，进入执行。

## 第四步：协议以 A 的身份调用业务合约

**这里不是 Keystore 替 Alice 买币，也不是 Authenticator 替 Alice 买币。**

它们的认证和权限工作完成之后，**8130 协议直接以账户 A 的身份派发调用**：

```text
协议以 A 的身份调用 USDC.approve(...)
    → USDC 看到 msg.sender = A

协议以 A 的身份调用 Launchpad.buy(...)
    → Launchpad 看到 msg.sender = A
```

这条直接调用路径是 8130 的原生执行语义。它不要求先把整笔交易包装成 Sponsor 发起的一笔普通交易，再让应用猜测真正用户是谁。:chatgpt-content-reference{index="14"}

因此，Launchpad 使用的是 **A 的 USDC**，买入结果也按本例参数交给 **A**，而不是交给 Alice 的 Actor ID 或平台 P。

**Alice 作为 ADMIN 亲自操作时，不需要引入 Policy Manager。**

## 第五步：结算结果

假设买入成功：

```text
A 的 USDC：1,000 → 900
A 的 MEME：增加
P 的原生币：扣除实际 Gas
本次使用的 nonce：被消耗
```

如果买入因为滑点失败：

```text
同一 Phase 内的 approve 和 buy 一起回滚；
A 不会完成这次买入；
本次 nonce 仍然被消耗；
已经发生的 Gas 仍然需要支付。
```

同一 Phase 内的共同回滚，不等于“这笔交易从未发生、也没有手续费”。:chatgpt-content-reference{index="15"}

到这里，最基础的 8130 交易已经完整走通：

> **Alice 签名 → 链验证她对 A 的权限 → 平台付 Gas → 协议以 A 的身份执行交易。**

---

# 三、Alice 不想每次确认了：怎样让 Agent 替她交易？

这一步才对应你产品文档里的需求：**Agent 能持续交易，但不能因此获得账户的无限权限；授权还需要有时效、能撤销。**:chatgpt-content-reference{index="16"} :chatgpt-content-reference{index="17"}

假设 Alice 希望：

> 接下来 24 小时，让 Agent 在指定 Launchpad 交易。  
> 单笔最多投入 100 USDC，授权期间累计最多投入 300 USDC。  
> 买入资产必须回到我的账户 A，不能随意转走资金。

## 第一步：Agent 使用自己的钥匙，不拿 Alice 的 Passkey

Agent 生成或使用自己的签名密钥，记作：

```text
agent-key
```

Alice 的 Passkey 仍然由 Alice 保管。

现在需要做的，不是把主钥匙交给 Agent，而是：

> **把 `agent-key` 登记为账户 A 的另一个 Actor，并给它受限权限。**

一个账户可以有多个 Actor，每个 Actor 可以配置自己的认证器、权限和有效期。:chatgpt-content-reference{index="18"} :chatgpt-content-reference{index="19"}

## 第二步：Alice 签署一份权限变更

钱包为 Alice 准备一份管理员授权，将以下配置写入 Keystore：

```text
账户：A

新增 Actor：
    agent-key

Authenticator：
    用来验证 Agent 签名的认证器，例如 k1

Scope：
    POLICY | NONCE

Expiry：
    24 小时后

Policy Manager：
    执行交易规则检查的入口

Policy Commitment：
    Alice 批准的规则参数的哈希承诺
```

这里有三个新概念。

### Scope：协议能直接理解的粗粒度权限

在本例中：

**`POLICY`** 表示：

> 这个 Agent 不能随意直接调用任何地址，只能调用绑定的 Manager。

**`NONCE`** 表示：

> 允许这个 Agent 使用有序 nonce 通道。

它们并没有直接表达“单笔 100 USDC”。金额等业务限制要由 Manager 实现。受限 Agent 也不能为了方便再加上 `OPERATOR`，否则会变成可以绕过 Manager 门禁的宽权限操作主体。:chatgpt-content-reference{index="20"}

### Policy Manager：执行具体规则的代码

Manager 负责检查：

```text
是不是允许的 Launchpad？
是不是允许的资产？
单笔有没有超过 100 USDC？
累计投入有没有超过 300 USDC？
买到的资产是否交给 A？
```

**协议只认识“必须走这个入口”，Manager 才理解业务规则。**:chatgpt-content-reference{index="21"}

### Commitment：防止 Agent 偷换规则

可以把它理解成 Alice 批准的规则的“指纹”。

Agent 执行时提交规则参数，Manager 检查它们是否对应 Keystore 中的 commitment，不能让 Agent 自己把“最多 100”改成“最多 10,000”。

但累计花了多少，还需要 Manager 或产品合约记录和检查；**一个哈希值不会自动帮你记账。**

## 第三步：选定一个具体 Manager 实现，避免“检查完了，谁动钱”这个空白

Manager 可以是独立合约，也可以由账户自身代码承担。

**为了把本例讲完整，我们选择：让账户 A 的代码自身兼任 Manager。**也就是：

```text
manager = A
```

这里有一个明确前提：**Alice 创建时选用的是具备策略检查功能的账户实现，不是直接使用一个无条件信任 self-call 的普通 `executeBatch`。**

这种选择下，职责是：

```text
A 作为 Account：
    持有资产，是对外的账户身份。

A 的策略执行代码作为 Manager：
    判断 Agent 这次具体能不能动这些资产。
```

两种职责可以在同一个地址，但不是同一个概念。官方允许 `manager = account`，同时明确警告：只因调用来自账户自身就放行，会使受限策略失去意义。默认账户代码确实包含信任自身调用的路径，因此不能不加修改地当作这种策略账户使用。:chatgpt-content-reference{index="22"}

## 第四步：Agent 自己签署后续买入，不再找 Alice

授权生效后，Agent 决定买入 100 USDC 的 MEME。

这次交易大致是：

```text
sender：
    仍然是 A

sender_auth：
    改成 agent-key 的签名

calls：
    Phase 0：
        A.executePolicyBuy(
            规则参数,
            买入 100 USDC,
            最低买入数量,
            接收地址 A
        )

payer：
    仍然是平台 P

payer_auth：
    平台本次的付款授权
```

`executePolicyBuy()` 是本例假设的账户函数，不是 EIP 标准接口。

对比前一次交易：

| 项目 | Alice 亲自买入 | Agent 自动买入 |
|---|---|---|
| 资产账户 | A | A |
| 本次签名 | Alice 的 Passkey | Agent 自己的密钥 |
| 权限依据 | Alice 是 ADMIN | Agent 获得受限授权 |
| 调用路径 | 可以直接调用 Token、Launchpad | 必须进入绑定的 Manager |
| Alice 是否逐笔确认 | 是 | 不需要，前提是授权仍有效 |

## 第五步：链和 Manager 分两层检查

完整执行顺序是：

```text
Agent 签名并提交
    ↓
Authenticator 识别出 agent-key
    ↓
链查询 A 的 Keystore 配置：
    agent-key 是否仍有效？
    是否拥有 POLICY 发起权限？
    nonce 等条件是否满足？
    ↓
链检查所有顶层调用：
    是否都指向已绑定的 Manager，也就是 A？
    ↓
A 的策略执行函数开始运行
    ↓
读取本次经过认证的 Actor 身份
    ↓
验证 commitment，检查资产、金额、累计额度和接收人
    ↓
通过后，由 A 的代码调用 Token 和 Launchpad
    ↓
买入完成，资产仍归 A
```

其中“读取本次经过认证的 Actor”使用的是 **Transaction Context**：可以把它理解为协议提供的本次交易信息窗口。Manager 从中知道：**这次虽然使用的是 A，但实际操作钥匙是 `agent-key`，不是 Alice 的管理员钥匙。**原稿里的 Manager 流程也是通过这个上下文识别账户与 Actor。:chatgpt-content-reference{index="23"}

因此不能只检查：

```text
msg.sender 是不是 A？
```

而应当知道：

```text
这次代表 A 操作的，到底是哪一个 Actor？
这个 Actor 被批准的具体规则是什么？
```

### Agent 越权时，会在哪一层失败？

| Agent 的尝试 | 拦截位置 |
|---|---|
| 绕过 Manager，直接调用 USDC 转账 | 协议的 `POLICY` 目标门禁 |
| 进入正确 Manager，但要求买入 1,000 USDC | Manager 的单笔额度检查 |
| 本次没超过 100，但累计超过 300 USDC | Manager 的累计预算检查 |
| 授权已到期或已被撤销，还继续签交易 | Actor 有效性检查 |

注意前两类业务／门禁失败可能仍是已入块、消耗 nonce 和 Gas 的失败交易，不应理解成“只要违规，就完全没有成本”。:chatgpt-content-reference{index="24"}

**这就是受限 Agent 模式的核心：Alice 批准一段权限，Agent 在这段权限内签具体交易，而不是 Agent 获得 Alice 的主密钥。**

---

# 四、Alice 已经有 EOA，还必须创建一个新地址 A 吗？

**不必须。**

前面为了说明真正的“新建账户”，使用了 Passkey＋Create 路径。

已有 EOA 则可以继续使用原地址和原 secp256k1 密钥发送 8130 AA 交易。对于没有代码、也没有显式 Create／Delegation 项的账户，协议会自动设置到默认账户实现的代码委托。**这是原地址启用能力，不是强制迁移到一个新地址。**:chatgpt-content-reference{index="25"}

因此，可以这样区分：

| Alice 的起点 | 路径 |
|---|---|
| 想新建一个由 Passkey 控制的账户 | Create：确定新地址、安装代码、登记初始 Actor |
| 已有一个 EOA，希望继续用原地址 | 用现有钥匙发 8130 交易，按规则启用默认账户行为 |
| 已有 EOA，还想使用受限 Agent 策略 | 在基础能力上增加 Actor 授权，并选择安全的策略执行实现 |

**“能发 8130 交易”与“已经具备完整的 Agent 风控账户”仍然是两件事。**

---

# 五、现在回头看，各组件分别做了什么？

| 组件 | 在 Alice 的例子里承担什么职责 |
|---|---|
| **Account：A** | 资产归属与对外账户身份；Alice 和 Agent 都是在使用 A |
| **Actor** | `alice-key`、`agent-key` 等登记在册的操作身份，不是各自独立的资金账户 |
| **Authenticator** | 验证签名，识别到底是哪一个 Actor 签的 |
| **Keystore** | 保存“A 授权了哪些 Actor，以及各自的认证器、权限、有效期和策略绑定” |
| **Scope** | 规定协议层面的权限类别，例如管理员、受限调用、付款、有序 nonce |
| **Policy Manager** | 检查单笔金额、累计预算、允许市场等具体业务规则 |
| **Transaction Context** | 让执行代码获取本次真正经过认证的 Actor 等交易信息 |
| **Nonce Manager** | 管理账户的操作序号和通道，防止同一有序交易被重复执行 |
| **Payer：P** | 承担 Gas；付费不意味着获得 A 的资产操作权 |
| **Phase** | 规定哪些业务调用必须共同成功或回滚 |

这些是原稿账户、权限、上下文和执行分组机制在同一个案例中的对应关系。:chatgpt-content-reference{index="26"} :chatgpt-content-reference{index="27"} :chatgpt-content-reference{index="28"} :chatgpt-content-reference{index="29"} :chatgpt-content-reference{index="30"}

最终，把整个故事压缩成三句话：

> **开户：创建账户 A，并登记 Alice 的钥匙有权控制它。**
>
> **亲自交易：Alice 的钥匙签名，链检查权限，然后以 A 的身份执行。**
>
> **Agent 交易：Alice 先登记一把受限钥匙；之后由这把钥匙签名，链限制调用入口，Manager 检查具体交易规则，资产仍然属于 A。**
