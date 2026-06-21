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

## 七个核心代码机制深度解析

### 机制一：跨账户转账作业 —— 双流水插入 + 关联表 + 跨币种自动换算

**代码位置**：
- [CreateTransfer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateTransfer.php)
- [Transfer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Banking/Transfer.php)

#### 执行时序

```
handle()
  ↓
event(TransferCreating)           // 前置钩子
  ↓
DB::transaction {
  1. 取两个账户的币种和汇率
     getCurrencyCode('from' / 'to')
     getCurrencyRate('from' / 'to')

  2. 创建支出流水（转出方）
     dispatch CreateTransaction
       type = EXPENSE_TRANSFER_TYPE
       account_id = from_account_id
       amount = 原始金额
       currency_code/rate = 转出方账户币种

  3. 跨币种换算（仅币种不一致时）
     if from_currency != to_currency:
         amount = convertBetween(
             amount, from_code, from_rate, to_code, to_rate
         )

  4. 创建收入流水（转入方）
     dispatch CreateTransaction
       type = INCOME_TRANSFER_TYPE
       account_id = to_account_id
       amount = 换算后的金额
       currency_code/rate = 转入方账户币种

  5. 写入关联表 transfers
     Transfer::create([
         expense_transaction_id => $expense->id,
         income_transaction_id  => $income->id,
     ])

  6. 处理附件上传
}
  ↓
event(TransferCreated)            // 后置钩子
```

#### 关键设计要点

1. **两条 Transaction 流水**：一笔转账产生两条独立记录，分别挂在转出和转入账户下，各自带独立的 `currency_code` 和 `currency_rate`
2. **跨币种换算桥**：通过"默认货币"作为中间桥接进行换算
   ```php
   // Traits/Currencies.php:43-54
   public function convertBetween($amount, $from_code, $from_rate, $to_code, $to_rate)
   {
       $default_amount = $amount;
       if ($from_code != default_currency()) {
           $default_amount = $this->convertToDefault($amount, $from_code, $from_rate);  // ÷ from_rate
       }
       return $this->convertFromDefault($default_amount, $to_code, $to_rate);          // × to_rate
   }
   ```
3. **Transfer 关联表**：仅存 `expense_transaction_id` + `income_transaction_id` 两个外键，其余所有字段（日期、金额、描述、支付方式等）都通过 `expense_transaction` 动态读取（见 Transfer 模型的各 `getAttribute` 访问器）
4. **原子性**：两条流水和关联表在同一个 `DB::transaction` 内创建，任何一步失败全部回滚

---

### 机制二：账户余额属性 —— 不做汇率折算，直接原值相加

**代码位置**：
- [Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Banking/Account.php#L123-L169)

#### 余额计算逻辑

```php
// Account.php:123-135
public function getBalanceAttribute()
{
    $total = $this->opening_balance;                        // 期初余额（账户币种）
    $total += $this->income_transactions->sum('amount');     // 加收入（交易币种原值）
    $total -= $this->expense_transactions->sum('amount');    // 减支出（交易币种原值）
    return $total;
}
```

#### 重要发现：不做汇率折算

**账户余额属性对币种不一致的交易不做任何汇率折算**，直接 `sum('amount')` 相加。这意味着：

- 如果账户下挂了**不同币种**的交易，余额会是"不同币种数值的直接代数和"，数值本身没有实际财务意义
- 设计意图：**假设同一账户下所有交易币种与账户币种一致**。因为在创建交易时，交易的 `currency_code` 默认取账户币种（`account_id` 一经确定，币种即锁定）
- 如果需要跨币种汇总报表，使用 `Transaction::getAmountConvertedToDefault()` 先换算成默认货币再求和（参见 Transactions 控制器 index 方法的处理方式）

#### 对比：Transaction 的账户本位币访问器

Transaction 模型提供了 `getAmountForAccountAttribute()`，专门用于将交易金额换算为所属账户的币种：

```php
// Transaction.php:350-363
public function getAmountForAccountAttribute()
{
    $amount = $this->amount;
    if ($this->account->currency_code != $this->currency_code) {
        $to_code = $this->account->currency_code;
        $to_rate = currency($this->account->currency_code)->getRate();
        $amount = $this->convertBetween($amount, $this->currency_code, $this->currency_rate, $to_code, $to_rate);
    }
    return $amount;
}
```

但 `Account::balance` **并没有使用这个访问器**，而是直接 `sum('amount')`。

---

### 机制三：建单据付款作业的三个事件钩子触发时机

**代码位置**：
- [CreateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php)
- [UpdateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/UpdateBankingDocumentTransaction.php)
- [DocumentTransactionCreating.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Events/Banking/DocumentTransactionCreating.php)
- [DocumentTransactionCreated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Events/Banking/DocumentTransactionCreated.php)
- [DocumentTransactionUpdating.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Events/Banking/DocumentTransactionUpdating.php)
- [DocumentTransactionUpdated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Events/Banking/DocumentTransactionUpdated.php)

#### 创建场景的钩子时序

```
handle()
  ↓
① event(DocumentTransactionCreating)    前置钩子：数据准备之前
    参数：$document, $request
    时机：prepareRequest()、checkAmount() 之前
  ↓
② prepareRequest()                       补全默认值（account_id、payment_method等）
  ↓
③ checkAmount()                          校验金额+更新document状态
  ↓
④ DB::transaction {
     dispatch CreateTransaction          创建流水
     $this->model->save()                保存document状态变更
     createHistory()                     创建历史记录
   }
  ↓
⑤ event(DocumentTransactionCreated)     后置钩子：所有数据库写入完成后
    参数：$document, $transaction
    时机：事务提交之后
```

#### 更新场景的钩子时序

```
handle()
  ↓
① event(DocumentTransactionUpdating)    前置钩子
    参数：$document, $transaction, $request
  ↓
② prepareRequest()
  ↓
③ checkAmount()     （更新时会先扣除当前交易金额，再重新校验）
  ↓
④ DB::transaction {
     dispatch UpdateTransaction          更新流水
     $this->model->save()                保存document状态变更
     createHistory()
   }
  ↓
⑤ event(DocumentTransactionUpdated)     后置钩子
    参数：$document, $transaction, $request
```

#### 钩子用途设计

| 钩子 | 可用数据 | 典型用途 |
|------|----------|----------|
| `DocumentTransactionCreating` | 原始请求 + 文档 | 补充/修改请求参数、权限检查 |
| `DocumentTransactionCreated` | 文档 + 已创建的交易 | 发送通知、触发下游业务、集成外部系统 |
| `DocumentTransactionUpdating` | 原始请求 + 文档 + 旧交易 | 变更前的数据比对、审计日志 |
| `DocumentTransactionUpdated` | 文档 + 已更新的交易 + 请求 | 变更后通知、同步外部系统 |

---

### 机制四：超额校验 —— 双币种换算 + 高精度比较决定已付状态

**代码位置**：
- [CreateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123)
- [UpdateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/UpdateBankingDocumentTransaction.php#L75-L119)

#### 创建场景的校验流程

```php
// CreateBankingDocumentTransaction.php:74-123
protected function checkAmount(): bool
{
    $code = $this->request['currency_code'];       // 交易币种
    $rate = $this->request['currency_rate'];       // 交易汇率

    $transaction_precision = currency($code)->getPrecision();
    $document_precision = currency($this->model->currency_code)->getPrecision();

    // ① 交易金额按交易币种精度四舍五入
    $amount = $this->request['amount'] = round($this->request['amount'], $transaction_precision);

    // ② 如果交易币种 ≠ 单据币种，先换算为单据币种再比较
    if ($this->model->currency_code != $code) {
        $converted_amount = $this->convertBetween(
            $amount, $code, $rate,
            $this->model->currency_code, $this->model->currency_rate
        );
        $amount = round($converted_amount, $document_precision);
    }

    // ③ 计算单据已付金额（触发 PaidAmountCalculated 事件）
    $this->model->paid_amount = $this->model->paid;
    event(new PaidAmountCalculated($this->model));

    // ④ 剩余应付 = 单据总额 - 已付
    $total_amount = round($this->model->amount - $this->model->paid_amount, $document_precision);

    // ⑤ 高精度比较：bccomp(本次付款, 剩余应付, 单据币种精度)
    $compare = bccomp($amount, $total_amount, $document_precision);

    if ($compare === 1) {
        // 付款 > 应付：超额支付，抛出异常
        throw new \Exception(trans('messages.error.over_payment', ...));
    } else {
        // compare === 0 → 全额付清 → status = 'paid'
        // compare === -1 → 部分付款 → status = 'partial'
        $this->model->status = ($compare === 0) ? 'paid' : 'partial';
    }
}
```

#### 更新场景的特殊处理

```php
// UpdateBankingDocumentTransaction.php:92
// 先把当前交易金额从已付中扣除，再重新计算
$this->model->paid_amount = ($this->model->paid - $this->transaction->amount_for_document);
```

即更新时使用的是 `getAmountForDocumentAttribute()`（已换算为单据币种的金额）来做扣除。

#### 技术要点

- **双币种双精度**：先按交易币种精度舍入，再换算成单据币种并按单据币种精度舍入
- **bccomp 高精度比较**：使用 BC Math 任意精度函数比较，避免浮点数误差
- **三种状态分支**：
  - `bccomp === 1` → 超额 → 抛异常中断
  - `bccomp === 0` → 刚好 → `status = 'paid'`
  - `bccomp === -1` → 不足 → `status = 'partial'`

---

### 机制五：创建交易作业 —— 按字符串与周期字段推断四类交易类型

**代码位置**：
- [CreateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateTransaction.php#L20-L29)
- [UpdateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/UpdateTransaction.php#L20-L29)

#### 推断逻辑

```php
if (! array_key_exists($this->request->get('type'), config('type.transaction'))) {
    // 请求传入的 type 不在系统配置中 → 需要自动推断

    $isExpense = str_contains((string) $this->request->get('type', ''), 'expense');
    // ↑ 条件一：type 字符串中是否包含 'expense'

    $isRecurring = ! empty($this->request->get('recurring_frequency'))
                 && $this->request->get('recurring_frequency') !== 'no';
    // ↑ 条件二：recurring_frequency 字段非空且不等于 'no'

    $type = $isExpense
        ? ($isRecurring ? Transaction::EXPENSE_RECURRING_TYPE : Transaction::EXPENSE_TYPE)
        : ($isRecurring ? Transaction::INCOME_RECURRING_TYPE : Transaction::INCOME_TYPE);

    $this->request->merge(['type' => $type]);
}
```

#### 四种推断结果

| `type` 包含 `expense` | `recurring_frequency` 有值且 ≠ 'no' | 推断结果 |
|----------------------|------------------------------------|----------|
| ❌ 否 | ❌ 否 | `income` 普通收入 |
| ❌ 否 | ✅ 是 | `income-recurring` 经常性收入 |
| ✅ 是 | ❌ 否 | `expense` 普通支出 |
| ✅ 是 | ✅ 是 | `expense-recurring` 经常性支出 |

#### 设计说明

- **前置条件**：仅当传入的 `type` 不在 `config('type.transaction')` 中时才触发推断。如果模块扩展了新的交易类型并注册到 config，就跳过推断
- **收入/支出的判断依据**：纯字符串匹配——`type` 中包含子串 `"expense"` 即为支出类，否则一律视为收入类
- **经常性判断依据**：看 `recurring_frequency` 字段（如 `daily` / `weekly` / `monthly` 等），存在且不为 `no` 即视为经常性交易

---

### 机制六：更新删除作业 —— 转账类交易的禁改禁删守卫

**代码位置**：
- [UpdateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/UpdateTransaction.php#L62-L76)
- [DeleteTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/DeleteTransaction.php#L29-L43)

#### 守卫逻辑（两个作业完全一致）

```php
public function authorize(): void
{
    // 守卫一：已对账的交易禁止任何修改/删除
    if ($this->model->reconciled) {
        throw new \Exception(trans('messages.warning.reconciled_tran'));
    }

    // 守卫二：转账类交易禁止通过 Transaction 作业修改/删除
    if ($this->model->isTransferTransaction()) {
        // isTransferTransaction() 判断: type 以 '-transfer' 结尾
        // 即 income-transfer / expense-transfer
        throw new \Exception(trans('messages.error.transfer_transaction'));
    }
}
```

#### 守卫判定来源

```php
// Traits/Transactions.php:48-53
public function isTransferTransaction(): bool
{
    $type = $this->type ?? $this->transaction->type ?? $this->model->type ?? Transaction::INCOME_TYPE;
    return Str::endsWith($type, '-transfer');
}
```

#### 设计意图

转账类交易（`income-transfer` / `expense-transfer`）是成对存在的，两条流水通过 `transfers` 关联表绑定。如果单独通过 `UpdateTransaction` / `DeleteTransaction` 修改其中一条，会导致：

1. 两条流水数据不一致（金额/币种/日期不匹配）
2. `transfers` 表出现悬挂的外键

因此必须通过专门的 Transfer 作业来操作：
- 更新：走 `UpdateTransfer`（同时更新两条流水）
- 删除：走 `DeleteTransfer`（同时删除两条流水 + transfers 记录）

**前端层面的守卫**：Transaction 列表中 `getLineActionsAttribute` 也用 `isNotTransferTransaction()` 控制编辑/删除按钮的显示。

---

### 机制七：已付属性 —— 短路返回 + 防 N+1 关联预加载

**代码位置**：
- [Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Document/Document.php#L373-L407)

#### 完整实现

```php
public function getPaidAttribute()
{
    // ① 第一层短路：单据金额为空 → 直接返回 false
    if (empty($this->amount)) {
        return false;
    }

    // ② 第二层短路：状态已标记为 paid → 直接返回单据总额，跳过循环
    if ($this->status == 'paid') {
        return $this->amount;
    }

    $paid = 0;
    $code = $this->currency_code;
    $rate = $this->currency_rate;
    $precision = currency($code)->getPrecision();

    // ③ 防 N+1：若 transactions 关联未预加载，则主动执行一次 lazy eager load
    if (!$this->relationLoaded('transactions')) {
        $this->load('transactions');   // 只执行一次查询，后续循环复用 Collection
    }

    // ④ 逐笔累加（必要时做币种换算）
    if ($this->transactions->count()) {
        foreach ($this->transactions as $transaction) {
            $amount = $transaction->amount;

            if ($code != $transaction->currency_code) {
                $amount = $this->convertBetween(
                    $amount,
                    $transaction->currency_code, $transaction->currency_rate,
                    $code, $rate
                );
            }

            $paid += $amount;
        }
    }

    return round($paid, $precision);
}
```

#### 三层性能优化

| 优化层 | 条件 | 行为 | 节省的开销 |
|--------|------|------|-----------|
| 第一层短路 | `empty($this->amount)` | `return false` | 避免所有后续逻辑 |
| 第二层短路 | `$this->status == 'paid'` | `return $this->amount` | 避免关联查询 + N笔循环 + 可能的币种换算 |
| 防 N+1 | `!$this->relationLoaded('transactions')` | `$this->load('transactions')` | 将 N+1 查询降为 1 次查询 |

#### 第二层短路的正确性

当单据 `status = 'paid'` 时，根据业务规则已付金额必然等于单据总额，因此直接返回 `$this->amount` 完全正确，无需再遍历交易记录。这是一个典型的"用状态字段冗余来换取计算性能"的设计。

#### 币种换算说明

与单据币种不一致的付款交易，会通过 `convertBetween()` 换算为单据币种后再累加，确保 `$paid` 的币种始终与单据币种一致。

---

## 总结

1. **付款 = 交易流水**：每笔付款记录就是一条 transaction，直接体现在银行账户流水中
2. **支付方式与账户松耦合**：两者是独立字段，不存在强制绑定关系
3. **入账账户的确定**：
   - 手动付款：用户在付款时选择账户
   - 在线支付：支付网关可配置自己的入账账户，未配置则用系统默认账户
4. **余额实时计算**：账户余额通过交易记录动态汇总，每笔付款自动影响余额
5. **单据状态联动**：付款记录影响单据的已付金额和状态（partial/paid）
6. **跨账户转账**：一次操作生成两条流水 + 一条关联记录，自动处理跨币种换算
7. **账户余额不折算**：`Account::balance` 直接原值相加，假设同账户币种一致；跨币种汇总需手动换算
8. **事件钩子体系**：单据付款的创建/更新各有前后两个事件，供模块扩展业务逻辑
9. **高精度金额校验**：使用 `bccomp` + 双精度舍入避免浮点误差，精确判断超额/足额/部分付款
10. **交易类型推断**：通过 `type` 字符串含 `expense` 和 `recurring_frequency` 两个维度推断四类交易
11. **转账守卫机制**：`UpdateTransaction`/`DeleteTransaction` 禁止操作 `-transfer` 类型交易，必须走专门的 Transfer 作业
12. **已付属性优化**：双层短路返回 + lazy eager load 防 N+1，兼顾正确性和性能
