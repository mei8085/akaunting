# Payment Method 到银行账户入账关系分析

## 核心概念澄清

### 付款记录 = 交易流水
在 Akaunting 中，**付款记录 (Payment)** 和 **银行流水 (Transaction)** 本质上是同一个东西，都存储在 `transactions` 表中。不存在独立的"付款"表，每一笔付款就是一条交易记录。

- 表名：`transactions`
- 模型：`App\Models\Banking\Transaction`

### 关键字段关系

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 交易ID（主键） |
| `type` | string | 交易类型：`income`(收入) / `expense`(支出) 及其衍生类型 |
| `account_id` | int | 关联的银行账户ID（外键 → `accounts.id`） |
| `payment_method` | string | 支付方式代码（如 `offline-payments.cash.1`） |
| `document_id` | int | 关联的单据ID（发票/账单，可为空） |
| `amount` | double | 交易金额 |
| `paid_at` | datetime | 付款日期 |

## 数据模型关系

```
Account (银行账户)
    └── hasMany → Transaction (交易/流水)
                      ├── belongsTo → Document (单据: 发票/账单)
                      ├── belongsTo → Contact (往来单位: 客户/供应商)
                      └── hasMany → TransactionTax (交易税额)

Document (单据: 发票/账单)
    └── hasMany → Transaction (付款记录/交易流水)
```

- 一个银行账户可以有多条交易流水
- 一张发票/账单可以有多笔付款记录（部分付款）
- 每笔付款记录必须关联一个银行账户和一种支付方式

## Payment Method 与 Bank Account 的入账关系

### 关系本质：松耦合、独立选择
**支付方式 (Payment Method)** 和 **银行账户 (Bank Account)** 是两个独立的属性，不存在强绑定关系。同一支付方式的付款可以入到不同的银行账户，由用户在付款时手动选择，或由系统配置默认值。

### 入账账户的确定逻辑

#### 1. 手动付款场景（后台操作）
用户在付款弹窗中**同时选择**账户和支付方式：

- 视图：`resources/views/modals/documents/payment.blade.php`
- 账户选择组件：`<x-form.group.account>`
- 支付方式选择组件：`<x-form.group.payment-method>`

代码参考：
- 账户默认值：`setting('default.account')`
- 支付方式默认值：`setting('default.payment_method')`

#### 2. 在线支付网关场景（前台支付）
每个支付网关模块可以配置自己的入账账户，若未配置则使用系统默认账户：

```php
// app/Abstracts/Http/PaymentController.php:173
$request['account_id'] = setting($this->alias . '.account_id', setting('default.account'));
$request['payment_method'] = $this->alias;
```

即：
- **支付方式** = 支付网关的别名（如 `paypal`、`stripe`）
- **入账账户** = 该支付方式配置的账户，若无则用系统默认账户

### 系统默认配置

默认支付方式：`offline-payments.cash.1`（现金）
默认账户：系统创建的第一个账户（通常是"Cash"账户）

配置来源：
- `config/setting.php` 中的 `fallback.default.payment_method`
- `database/seeds/Accounts.php` 中设置 `default.account`
- `database/seeds/Settings.php` 中初始化离线支付方式

## 付款记录落到流水的完整流程

### 流程一：后台手动付款（以发票收款为例）

```
用户点击"收款"按钮
    ↓
DocumentTransactions@create
    （加载付款模态框，展示账户、支付方式、金额等选项）
    ↓
用户提交付款表单
    ↓
DocumentTransactions@store
    ↓
CreateBankingDocumentTransaction Job
    ├── prepareRequest() 准备请求数据
    │   ├── account_id = request 或 default.account
    │   ├── payment_method = request 或 default.payment_method
    │   ├── document_id = 单据ID
    │   ├── contact_id = 往来单位ID
    │   └── ...
    ├── checkAmount() 校验金额（防止超额支付）
    └── DB::transaction 数据库事务
        ├── dispatch CreateTransaction Job → 创建 Transaction 记录
        ├── 更新 Document 状态 (paid / partial)
        └── 创建 DocumentHistory 记录
    ↓
Transaction 记录写入 transactions 表
    ↓
账户余额自动更新（通过关联关系动态计算）
```

### 流程二：在线支付（以网关支付为例）

```
用户在前台选择支付方式并支付
    ↓
支付网关回调 / 支付确认
    ↓
PaymentController::finish()
    ↓
dispatchPaidEvent()
    ├── account_id = setting(alias.account_id, default.account)
    ├── payment_method = 支付方式别名
    ├── amount = 单据金额
    ├── type = 'income'
    └── 触发 PaymentReceived 事件
    ↓
PaymentReceived 事件
    ↓
CreateDocumentTransaction 监听器
    ↓
CreateBankingDocumentTransaction Job
    ↓
CreateTransaction Job → 创建 Transaction 记录
    ↓
Transaction 记录写入 transactions 表
```

### 流程三：直接创建交易（银行流水手动录入）

用户也可以直接在"交易"菜单中创建收入/支出记录，不关联任何单据：

```
Transactions@create → Transactions@store
    ↓
CreateTransaction Job
    ↓
Transaction 记录写入 transactions 表
```

## 核心代码位置

### 模型层
- [Transaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Banking/Transaction.php) - 交易/流水模型
- [Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Banking/Account.php) - 银行账户模型
- [Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Document/Document.php) - 单据模型（发票/账单）

### 作业层（核心业务逻辑）
- [CreateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php) - 创建单据付款交易
- [UpdateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/UpdateBankingDocumentTransaction.php) - 更新单据付款交易
- [CreateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateTransaction.php) - 创建交易记录

### 控制器层
- [DocumentTransactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php) - 单据付款模态框控制器
- [Transactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Http/Controllers/Banking/Transactions.php) - 交易管理控制器

### 事件与监听器
- [PaymentReceived.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Events/Document/PaymentReceived.php) - 付款收到事件
- [CreateDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Listeners/Document/CreateDocumentTransaction.php) - 创建交易监听器
- [Event.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Providers/Event.php) - 事件服务提供者（事件-监听器映射）

### 支付抽象类
- [PaymentController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Abstracts/Http/PaymentController.php) - 支付控制器抽象基类

## 账户余额计算

账户余额不是存储字段，而是通过交易记录**动态计算**的：

```php
// app/Models/Banking/Account.php:123-135
public function getBalanceAttribute()
{
    $total = $this->opening_balance;          // 期初余额
    $total += $this->income_transactions->sum('amount');  // 加收入
    $total -= $this->expense_transactions->sum('amount'); // 减支出
    return $total;
}
```

因此每创建一笔交易（付款记录），账户余额会自动反映变化。

## 单据已付金额计算

单据的已付金额也是通过关联交易动态计算的：

```php
// app/Models/Document/Document.php:373-407
public function getPaidAttribute()
{
    $paid = 0;
    foreach ($this->transactions as $transaction) {
        $amount = $transaction->amount;
        // 处理币种转换...
        $paid += $amount;
    }
    return round($paid, $precision);
}
```

当已付金额 = 单据总金额时，单据状态变为 `paid`（已付款）；否则为 `partial`（部分付款）。

## 交易类型说明

交易类型 (`type` 字段) 包含多种子类型：

| 类型 | 说明 |
|------|------|
| `income` | 收入（普通收款） |
| `expense` | 支出（普通付款） |
| `income-transfer` | 转账收入 |
| `expense-transfer` | 转账支出 |
| `income-split` | 拆分收入 |
| `expense-split` | 拆分支出 |
| `income-recurring` | 经常性收入 |
| `expense-recurring` | 经常性支出 |

可通过配置 (`config/type.php`) 和设置 (`setting('transaction.type.income')`) 扩展更多类型。

## 总结

1. **付款 = 交易流水**：每笔付款记录就是一条 transaction，直接体现在银行账户流水中
2. **支付方式与账户松耦合**：两者是独立字段，不存在强制绑定关系
3. **入账账户的确定**：
   - 手动付款：用户在付款时选择账户
   - 在线支付：支付网关可配置自己的入账账户，未配置则用系统默认账户
4. **余额实时计算**：账户余额通过交易记录动态汇总，每笔付款自动影响余额
5. **单据状态联动**：付款记录影响单据的已付金额和状态（partial/paid）
