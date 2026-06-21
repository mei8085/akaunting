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

### 机制八：MatchBankingDocumentTransaction —— 改 document_id 绑定流水到单据并校验超额

**代码位置**：
- [MatchBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php)

#### 与 CreateBankingDocumentTransaction 的区别

| 维度 | CreateBankingDocumentTransaction | MatchBankingDocumentTransaction |
|------|----------------------------------|----------------------------------|
| 场景 | 付款时新建一条流水并关联单据 | 已有独立流水，匹配并绑定到已存在的单据 |
| Transaction | 新建 dispatch(CreateTransaction) | 更新已有 dispatch(UpdateTransaction) |
| 核心动作 | 写入 `document_id`（新流水） | 改写已有 Transaction 的 `document_id` |

#### 执行时序

```
handle()
  ↓
① checkAmount()          金额校验 + 状态计算
  ├── round(amount, 交易币种精度)
  ├── 币种不一致 → convertBetween 换算为单据币种
  ├── $this->model->paid_amount = $this->model->paid   计算累计已付
  ├── event(PaidAmountCalculated)                       触发事件（可由监听者修改 paid_amount）
  ├── $total_amount = amount - paid_amount              剩余应付
  ├── ↓↓ unset($this->model->reconciled)                清空属性缓存
  ├── ↓↓ unset($this->model->paid_amount)               清空临时计算结果
  └── bccomp(本次付款, 剩余应付)
        → 1 → 抛 over_match 异常
        → 0 → status = 'paid'
        → -1 → status = 'partial'
  ↓
② DB::transaction {
     dispatch(UpdateTransaction, [
         document_id => $this->model->id,  // ← 核心：把已有流水绑定到单据
         type        => 保持原 type
     ])
     $this->model->save()                  // 保存新状态 paid/partial
     createHistory()                       // 写入单据历史
   }
  ↓
return $this->transaction
```

#### 设计意图

用于"银行流水导入后匹配单据"的场景：先从银行对账单导入一条独立的 Transaction（`document_id = null`），然后由用户或系统通过这个 Job 将它与某张 Invoice/Bill 绑定，绑定后该流水就被视为该单据的一笔付款，参与已付金额累计。

---

### 机制九：收入支出类型 —— 从 settings 动态读取 + 支持模块扩展

**代码位置**：
- [Transactions.php (Trait)](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Traits/Transactions.php#L84-L128)
- [setting.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/config/setting.php#L188-L195)

#### 默认值（fallback）

```php
// config/setting.php:193-194
'transaction.type.income'  => 'income,income-transfer',
'transaction.type.expense' => 'expense,expense-transfer',
```

收入类型默认包含 `income` 和 `income-transfer`；支出类型默认包含 `expense` 和 `expense-transfer`。

#### 动态读取

```php
// Traits/Transactions.php:94-103
public function getTransactionTypes(string $index, string $return = 'array'): string|array
{
    $types = (string) setting('transaction.type.' . $index);  // 从 settings 表读取
    return $return == 'array'
        ? ($types === '' ? [] : explode(',', $types))
        : $types;
}
```

#### 判断收入/支出

```php
// Traits/Transactions.php:12-29
public function isIncome(): bool
{
    $type = $this->type ?? $this->transaction->type ?? ...;
    return in_array($type, $this->getIncomeTypes());  // ← 从 setting 动态拉取数组比对
}

public function isExpense(): bool
{
    return in_array($type, $this->getExpenseTypes());
}
```

不是简单判断 `type == 'income'` 或 `str_contains($type, 'income')`，而是查 setting 里配置的类型列表。

#### 模块扩展机制

```php
// Traits/Transactions.php:105-128
public function addIncomeType(string $new_type): void
{
    $this->addTransactionType($new_type, 'income');
}

public function addExpenseType(string $new_type): void
{
    $this->addTransactionType($new_type, 'expense');
}

public function addTransactionType(string $new_type, string $index): void
{
    $types = explode(',', setting('transaction.type.' . $index));
    if (in_array($new_type, $types)) return;  // 去重

    $types[] = $new_type;
    setting(['transaction.type.' . $index => implode(',', $types)])->save();
    // ↑ 直接写 settings 表持久化
}
```

第三方模块安装时可以调用 `addIncomeType('my-module-income-type')` 把自定义类型注册进去，之后系统中 `isIncome()` 判断就会识别该类型为收入。

#### 配套：config/type.php 定义类型元信息

虽然收入/支出列表在 setting 里动态维护，但每种交易类型的详细元信息（名称、对应单据类型、路由别名、图标等）仍然在 `config/type.transaction` 中定义。setting 存的是字符串列表，config 存的是类型说明。

---

### 机制十：Transaction 模型默认全局过滤 —— 排除经常性和拆分交易

**代码位置**：
- [Transaction.php (Scope)](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Scopes/Transaction.php)
- [Transaction.php (Model)](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Banking/Transaction.php#L97-L262)
- [Scopes.php (Trait)](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Traits/Scopes.php)

#### 注册全局 Scope

```php
// Models/Banking/Transaction.php:101-103
protected static function booted()
{
    static::addGlobalScope(new Scope);   // ← App\Scopes\Transaction
}
```

所有 `Transaction::query()` 默认都会带上这个 Scope。

#### Scope 逻辑

```php
// Scopes/Transaction.php:21-26
public function apply(Builder $builder, Model $model)
{
    $this->applyNotRecurringScope($builder, $model);  // 排除 *-recurring
    $this->applyNotSplitScope($builder, $model);      // 排除 *-split
}
```

底层调用两个本地 scope：

```php
// Models/Banking/Transaction.php:249-262
public function scopeIsNotRecurring(Builder $query): Builder
{
    return $query->where($this->qualifyColumn('type'), 'not like', '%-recurring');
}

public function scopeIsNotSplit(Builder $query): Builder
{
    return $query->where($this->qualifyColumn('type'), 'not like', '%-split');
}
```

#### 智能跳过机制

```php
// Traits/Scopes.php:11-31
public function applyNotRecurringScope(Builder $builder, Model $model): void
{
    // 如果 where 条件里已经显式指定了 type 列 → 跳过 Scope，不追加过滤
    if ($this->scopeColumnExists($builder, $model->getTable(), 'type')) {
        return;
    }
    $builder->isNotRecurring();
}
```

`scopeColumnExists()` 遍历当前 query 的所有 where 条件，只要发现任何一个涉及 `transactions.type` 列，就认为开发者已在业务层自行控制类型过滤，全局 Scope 不再追加 `not like '%-recurring'` 条件，避免冲突。

#### 需要查看经常性/拆分交易时的做法

```php
// 方法一：移除整个全局 Scope
Transaction::withoutGlobalScope(\App\Scopes\Transaction::class)->...

// 方法二：Transfer 模型已自动移除
// Models/Banking/Transfer.php:66-68
return $this->belongsTo(Transaction::class, 'expense_transaction_id')
                ->withoutGlobalScope('App\Scopes\Transaction')   // ← 自动加
                ->withDefault([...]);
```

Transfer 关联交易时会自动 `withoutGlobalScope`，因为转账流水的 type 是 `income-transfer`/`expense-transfer`，不会被 recurring/split 过滤命中，但为了保险也移除了。

#### Document 模型同样有全局 Scope

`App\Scopes\Document` 只应用 `applyNotRecurringScope`（不应用 split scope），因此单据默认查询也排除 `*-recurring` 类型。

---

### 机制十一：PaidAmountCalculated 事件 —— 多处触发但无核心监听者

**代码位置**：
- [PaidAmountCalculated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Events/Document/PaidAmountCalculated.php)

#### 触发点（共四处）

| 触发文件 | 位置 | 场景 |
|----------|------|------|
| `CreateBankingDocumentTransaction.php` | checkAmount() | 新建单据付款流水前 |
| `UpdateBankingDocumentTransaction.php` | checkAmount() | 更新单据付款流水前 |
| `MatchBankingDocumentTransaction.php` | checkAmount() | 匹配流水到单据时 |
| `UpdateDocument.php` | handle() | 修改单据金额/币种时 |

#### 触发模式（完全一致）

```php
$this->model->paid_amount = $this->model->paid;   // ① 先把已付金额算出，挂到模型属性上
event(new PaidAmountCalculated($this->model));    // ② 触发事件
// ③ 后续用 $this->model->paid_amount 做状态判断
$total_amount = round($this->model->amount - $this->model->paid_amount, $precision);
```

#### 无核心监听者

在 `App\Providers\Event` 事件服务提供者中，`PaidAmountCalculated` **没有注册任何监听者**。核心代码中没有类监听这个事件。

#### 扩展钩子设计

这个事件是纯粹的**扩展点**，留给第三方模块/插件使用。监听者拿到 `$event->model` 后：

1. 可以**读取** `$model->paid_amount` 查看当前已付金额
2. 可以**修改** `$model->paid_amount = xxx`，后续校验逻辑会使用修改后的值
3. 典型用途：实现"折扣""预存抵扣""信用额度"等需要在付款时冲减已付金额的功能

#### 配合 unset 机制使用（见机制十二）

事件修改完 `paid_amount` 后，紧接着就会 `unset($this->model->paid_amount)`，不会把监听者改的值持久化到数据库，只用于本次校验。

---

### 机制十二：unset 已付和对账属性 —— 清理缓存避免脏数据驱动重算

**代码位置**：
- [MatchBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L64-L65)
- [UpdateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Document/UpdateDocument.php#L69-L70)
- [CreateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php)
- [UpdateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Jobs/Banking/UpdateBankingDocumentTransaction.php)

#### 典型代码片段

```php
// MatchBankingDocumentTransaction.php:59-66
$this->model->paid_amount = $this->model->paid;       // ① 手动赋值，挂在模型 attributes 数组上
event(new PaidAmountCalculated($this->model));         // ② 监听者可能进一步修改 paid_amount

$total_amount = round($this->model->amount - $this->model->paid_amount, $precision);  // ③ 用临时值计算

unset($this->model->reconciled);      // ← ④ 清理临时属性
unset($this->model->paid_amount);     // ← ④ 清理临时属性
```

#### Eloquent 属性缓存原理

Laravel Eloquent Model 的 attributes 数组是读写缓存：

- `$model->paid_amount` 先查 `$this->attributes['paid_amount']`，有就直接返回（不走 accessor）
- 如果 attributes 中不存在，再走 `getPaidAmountAttribute()` accessor 动态计算

#### 为什么要 unset

| 属性 | 原因 |
|------|------|
| `paid_amount` | ① checkAmount() 中临时赋值用于计算，存的是"已换算成单据币种的金额"，不是数据库字段。如果不 unset，后续 `$model->save()` 会把它当字段写入导致 SQL 报错，或后续读取 `$model->paid` 时不走 accessor 而是返回这个过期临时值 |
| `reconciled` | 不是 Document 表字段，是 `getReconciledAttribute()` 动态计算（看关联交易是否都已对账）。若有人在流程中给 Document 临时挂了 `reconciled` 属性，会导致后续 accessor 被短路，返回错误值 |

一句话：**计算过程中临时挂载到模型上的值，用完必须 unset，避免污染 attributes 缓存影响后续 accessor 或 save 操作**。

#### 在 UpdateDocument 中的另一个作用

```php
// UpdateDocument.php:69-72
unset($this->model->reconciled);
unset($this->model->paid_amount);
$this->model->update($this->request->all());
```

如果不先 unset，`$this->model->paid_amount` 会出现在 Model 的 `$this->attributes` 里，`update()` 方法会把它当数据库字段一起 `UPDATE documents SET paid_amount = xxx` 执行，直接 SQL 报错。

---

### 机制十三：Transfer 展示字段 —— 从两条流水反向拼装

**代码位置**：
- [Transfer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/63-akaunting/app/Models/Banking/Transfer.php)

#### 表结构极简

`transfers` 表只有 5 个字段：

| 字段 | 说明 |
|------|------|
| `id` | 主键 |
| `company_id` | 公司ID |
| `expense_transaction_id` | FK → 支出流水ID |
| `income_transaction_id` | FK → 收入流水ID |
| `created_from` / `created_by` / `created_at` / `updated_at` | 审计字段 |

**金额、日期、描述、支付方式、账户、币种、汇率等所有业务字段全部不存在 transfers 表中**。

#### 反向拼装原理

通过 Eloquent accessor（`getXxxAttribute`）从两条关联 Transaction 动态读取：

```
transfers 表
  ├── expense_transaction_id → Transaction(type=expense-transfer)
  │     ├── account_id      → from_account_id
  │     ├── currency_code   → from_currency_code
  │     ├── currency_rate   → from_account_rate
  │     ├── paid_at         → transferred_at
  │     ├── description     → description
  │     ├── amount          → amount
  │     ├── payment_method  → payment_method
  │     └── reference       → reference
  │
  └── income_transaction_id  → Transaction(type=income-transfer)
        ├── account_id       → to_account_id
        ├── currency_code    → to_currency_code
        └── currency_rate    → to_account_rate
```

#### accessor 实现模式

```php
// Transfer.php:139-142
public function getFromAccountIdAttribute($value = null)
{
    return $value ?: $this->expense_transaction->account_id;
    //         ↑ DB 字段（可为空）  ↑ 关联流水上的字段（兜底）
}

// Transfer.php:199-202
public function getTransferredAtAttribute($value = null)
{
    return $value ?: $this->expense_transaction->paid_at;
}

// Transfer.php:219-222
public function getAmountAttribute($value = null)
{
    return $value ?: $this->expense_transaction->amount;
}
```

所有展示属性都走同一模式：**优先用 transfers 表自己的字段（如果未来有），否则从 expense_transaction（转出流水）取**。转出方流水被视为"主流水"，金额、日期、描述、支付方式、参考号都以转出方为准。转入方流水只提供 `to_account_id`、`to_currency_code`、`to_account_rate` 三个目标账户相关字段。

#### $appends 声明

```php
// Transfer.php:19-32
protected $appends = [
    'attachment',
    'from_account_id', 'from_currency_code', 'from_account_rate',
    'to_account_id', 'to_currency_code', 'to_account_rate',
    'transferred_at', 'description', 'amount', 'payment_method', 'reference',
];
```

这些 accessor 都被加入 `$appends`，序列化（toArray / JSON）时自动输出，前端可直接用。

#### 关联上移除 Transaction 全局 Scope

```php
// Transfer.php:64-68
public function expense_transaction()
{
    return $this->belongsTo(Transaction::class, 'expense_transaction_id')
                    ->withoutGlobalScope('App\Scopes\Transaction')  // ← 必须移除
                    ->withDefault(['name' => trans('general.na')]);
}
```

因为转账流水的 type 是 `income-transfer` / `expense-transfer`，不会被 `-recurring` 或 `-split` 过滤命中，理论上不需要移除 Scope。但为了防止未来 Scope 扩展（比如加 `-transfer` 过滤），这里提前防御性地移除了。

#### 排序字段也来自两条流水

```php
// Transfer.php:46-55
public $sortable = [
    'expense_transaction.paid_at',      // 转账日期（转出方）
    'expense_transaction.reference',
    'expense_transaction.name',
    'income_transaction.name',
    'expense_transaction.rate',
    'income_transaction.rate',
    'expense_transaction.amount',       // 转账金额（转出方币种）
    'income_transaction.amount',        // 入账金额（转入方币种，跨币种时不同）
];
```

列表排序可以按转出方或转入方的任意字段进行，体现了"一条 Transfer = 两条 Transaction 组合视图"的设计理念。

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
13. **流水绑定单据**：`MatchBankingDocumentTransaction` 改写已有 Transaction 的 `document_id`，将独立流水与单据关联并做超额校验
14. **交易类型动态配置**：收入/支出类型列表存于 settings 表，支持模块通过 `addIncomeType()`/`addExpenseType()` 运行时扩展
15. **全局查询过滤**：`Transaction` 模型默认排除 `*-recurring` 和 `*-split` 类型，但 where 中已指定 type 时自动跳过
16. **PaidAmountCalculated 扩展钩子**：四处触发无核心监听者，监听者可修改 `$model->paid_amount` 影响后续校验
17. **unset 清理机制**：计算过程中临时挂到模型上的 `paid_amount`、`reconciled` 用完必须 unset，避免污染 attributes 缓存
18. **Transfer 反向拼装**：transfers 表仅存两外键，所有业务字段通过 accessor 从两条 Transaction 动态读取，转出方为主流水
