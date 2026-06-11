# 采购单据与库存增减、费用入账协作链路分析

> 本文档基于 Akaunting v3.x 源码，深入分析采购账单（Bill）从创建到支付的全链路中，**库存增减记录**与**费用记账（Transaction）**是如何协作触发的。
>
> **⚠️ 关键勘误（相比初版）：**
> 1. `PaymentReceived` 事件**不在**采购付款路径上，它是销售发票在线支付专用
> 2. 作废（Cancel）采购单**不影响**库存（document_items 保留），只有删除（Delete）才会清除库存记录
> 3. 费用类型（expense/income）由 `config/type.php` 配置决定，通过付款模态框的隐藏字段传入
> 4. `inventory_stock_action` 是配置预留项，核心代码未直接使用

---

## 一、核心架构概览

Akaunting 是基于 **Laravel** 的开源财务系统，其采购业务围绕四个核心模型展开：

| 模型 | 表名 | 职责 | 代码位置 |
|------|------|------|----------|
| `Document` | `documents` | 采购账单/销售发票的统一抽象（`type = 'bill'` 为采购） | [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Document/Document.php) |
| `DocumentItem` | `document_items` | 账单的明细行（含商品数量 quantity） | [DocumentItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Document/DocumentItem.php) |
| `Item` | `items` | 商品/服务主数据（**v3.0+ 不再保存库存数量**） | [Item.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Common/Item.php) |
| `Transaction` | `transactions` | 银行收/付款流水（`type = 'expense'` 为费用支出） | [Transaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Banking/Transaction.php) |

三者的关系：
```
Document (Bill)  1───N  DocumentItem (quantity: 采购数量)
     │                        │
     │                        └─── 关联 Item (purchase_price / sale_price)
     │
     └───1───N  Transaction (expense: 费用支出记账)
```

---

## 二、采购入口：从提交表单到创建 Bill

### 2.1 HTTP 入口

采购账单的创建入口位于控制器 [Bills.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Purchases/Bills.php#L82-L101)：

```php
public function store(Request $request)
{
    $response = $this->ajaxDispatch(new CreateDocument($request));
    // ...
}
```

**关键点**：
- 控制器非常薄，所有业务逻辑委托给 `CreateDocument` Job
- 采用 `ajaxDispatch` 异步调度（同步执行但返回 AJAX 友好格式）
- 请求通过 [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Requests/Document/Document.php) 表单验证

### 2.2 请求验证阶段的数量预处理

在 [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Requests/Document/Document.php#L92-L119) 表单验证中，有一个关键操作：

```php
foreach ($items as $key => $item) {
    $items[$key]['quantity'] = calculation_to_quantity($item['quantity']);
}
```

**`calculation_to_quantity()` 函数**（位于 [helpers.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Utilities/helpers.php#L433-L460)）：
- 支持数学表达式输入，如 `"2*3"` → `6`，`"(1+2)*5"` → `15`
- 自动替换 `x` → `*`，`,` → `.`
- 使用 `eval()` 安全计算（白名单字符校验）

### 2.3 CreateDocument Job：创建主流程

核心 Job 定义在 [CreateDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocument.php#L20-L57)：

```
┌───────────────────────────────────────────────────────┐
│ DB::transaction 事务包裹                              │
├───────────────────────────────────────────────────────┤
│ 1. Document::create()            ← 写入 documents 表 │
│ 2. 上传附件 (attachMedia)                              │
│ 3. CreateDocumentItemsAndTotals  ← 创建明细 + 汇总    │
│ 4. Document->update()            ← 更新最终 amount    │
│ 5. createRecurring()            ← 定期账单配置        │
└───────────────────────────────────────────────────────┘
    ↓
event: DocumentCreated  →  触发监听器链
```

**事件派发**：
- 创建前：`DocumentCreating`
- 创建后：`DocumentCreated` → 绑定三个监听器（见 [Event.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Providers/Event.php#L53-L57)）
  1. `CreateDocumentCreatedHistory` - 创建历史记录
  2. `IncreaseNextDocumentNumber` - 递增单据编号
  3. `SettingFieldCreated` - 自定义字段处理

---

## 三、库存增减机制：动态计算而非字段保存

### ⚠️ 重要架构变更：v3.0 移除了库存字段

迁移文件 [2022_05_10_000000_core_v300.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/database/migrations/2022_05_10_000000_core_v300.php#L37-L43) 明确展示了这一变更：

```php
Schema::table('items', function(Blueprint $table) {
    $table->dropColumn('quantity');   // ← 删除了 items 表的库存数量字段
});
```

**设计动机**：
- **v1/v2 设计**：`items.quantity` 保存实时库存，每次采购/销售需同步更新该字段
- **v3+ 设计**：库存完全由 `document_items` 动态计算，消除冗余数据与并发更新问题

### 3.1 inventory_stock_action 配置：预留扩展点

在配置文件 [type.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/config/type.php) 中，每种单据类型定义了 `inventory_stock_action`：

| 单据类型 | inventory_stock_action | 含义 |
|----------|------------------------|------|
| `Document::BILL_TYPE` | `'increase'` | 采购增加库存 |
| `Document::INVOICE_TYPE` | `'decrease'` | 销售减少库存 |

**⚠️ 重要说明**：该配置项在核心代码中**未被直接使用**（全代码库搜索无引用），它是为库存管理模块预留的扩展点。实际库存计算由 `document_items` 表的 `type` 字段（bill/invoice）隐式区分。

### 3.2 当前库存跟踪机制

在 [Item.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Common/Item.php#L74-L87) 中，模型提供了两个关键关联：

```php
public function bill_items()
{
    return $this->document_items()->where('type', Document::BILL_TYPE);
}

public function invoice_items()
{
    return $this->document_items()->where('type', Document::INVOICE_TYPE);
}
```

**库存动态计算公式**：
```
当前库存 = Σ(bill_items.quantity) - Σ(invoice_items.quantity)
```

即：
- **采购入库**：创建 `type = 'bill'` 的 DocumentItem 时，`quantity` 字段记录了该商品的采购数量（正向增加）
- **销售出库**：创建 `type = 'invoice'` 的 DocumentItem 时，`quantity` 为销售数量（负向减少）

### 3.3 采购明细创建链路

`CreateDocumentItemsAndTotals` → 逐行调用 `CreateDocumentItem`

[CreateDocumentItemsAndTotals.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L160-L263) 的 `createItems()` 方法核心逻辑：

```
遍历 request['items']:
    ├─ 若 item_id 为空 → 先 dispatch(CreateItem) 新建商品
    ├─ dispatch(CreateDocumentItem) → 写入 document_items 表
    │                                ├─ 保存 quantity = 采购数量
    │                                ├─ 保存 price = 单价
    │                                ├─ 保存 total = 行小计（含折扣后）
    │                                └─ 创建 DocumentItemTax 税金记录
    ├─ 累加 sub_total / actual_total / discount_amount_total
    └─ 聚合 taxes 数组（按 tax_id 分组汇总）
```

[CreateDocumentItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocumentItem.php#L29-L202) 负责数量与金额的精密计算：

```
行计算流程:
1. item_amount = price × quantity（行原始金额）
2. 应用行级折扣（item_discount）→ item_discounted_amount
3. 应用全局折扣（global_discount）
4. 处理5种税类型:
   ├─ inclusives (价内税)   : 从金额中剥离税
   ├─ fixeds (从量税)       : 税额 = 税率 × quantity
   ├─ normals (常规增值税)  : 税额 = (实际价 × 税率%)
   ├─ withholdings (代扣税) : 税额为负（抵扣应付）
   └─ compounds (复合税)    : 基于「含税总额」叠加
5. 写入 document_items: quantity / price / total / tax
6. 写入 document_item_taxes 表
```

### 3.4 写入 DocumentTotals：账单汇总

明细处理完毕后，`CreateDocumentItemsAndTotals` 按 `sort_order` 写入 `document_totals` 表：

| code | 含义 | 说明 |
|------|------|------|
| `sub_total` | 小计 | Σ(price × quantity) |
| `item_discount` | 行折扣 | 各行折扣汇总 |
| `discount` | 全局折扣 | 账单级折扣（百分比/固定额） |
| `tax` | 税金 | 按税种多行（name=税种名） |
| `extra` | 额外费用 | 运费/手续费等（配置项） |
| `total` | 合计 | 最终应付金额 |

完成后通过 `$this->model->update()` 将最终 `amount` 回写到 `documents` 表。

---

## 四、费用入账（Transaction）：付款模态框直接触发

### ⚠️ 关键勘误：PaymentReceived 事件不在采购付款路径上

**重要澄清**：
- `PaymentReceived` 事件**仅用于销售发票（Invoice）的在线支付场景**
- 采购 Bill 的付款**不经过 PaymentReceived 事件**
- 采购付款有自己独立的路径：`DocumentTransactions` 模态框控制器 → `CreateBankingDocumentTransaction` Job

| 路径 | 适用场景 | 触发方式 | type |
|------|----------|----------|------|
| PaymentController → PaymentReceived 事件 | 销售发票在线支付（前端门户/支付网关回调） | 事件驱动 | `income` |
| DocumentTransactions 模态框 → CreateBankingDocumentTransaction | 采购账单付款（后台手动添加） | 直接调用 Job | `expense` |

`PaymentReceived` 事件的唯一定义触发点在 [PaymentController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Abstracts/Http/PaymentController.php#L170-L179)：

```php
public function dispatchPaidEvent($invoice, $request)
{
    $request['type'] = 'income';  // 明确是 income（收入）
    event(new PaymentReceived($invoice, $request));
}
```

参数名 `$invoice` 也明确表明这是给销售发票用的。

### 4.1 状态机：Bill 的生命周期

采购 Bill 的状态流转定义在 [Documents.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Traits/Documents.php#L69-L107) 的 `getDocumentStatuses('bill')`：

```
draft → received → partial → paid
  │         │                    ↑
  │         └─── (若 amount=0) ──┘
  └──→ cancelled (可 restore 回 draft)
```

| 状态 | 含义 | 费用入账是否已发生 |
|------|------|-------------------|
| `draft` | 草稿 | ❌ 未发生 |
| `received` | 已收到账单/确认收到货物 | ❌ 仅记录应付款，未实付 |
| `partial` | 部分付款 | ⚠️ 已部分记账（expense transaction 存在） |
| `paid` | 全额付清 | ✅ 已全额记账 |
| `cancelled` | 作废 | ❌ 关联的 transactions 被删除 |

### 4.2 标记「已收到」：仅更新状态，不产生 Transaction

控制器 [Bills.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Purchases/Bills.php#L229-L238) 的 `markReceived()`：

```php
public function markReceived(Document $bill)
{
    event(new DocumentReceived($bill));  // 触发事件
    // ...
}
```

监听器 [MarkDocumentReceived.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Listeners/Document/MarkDocumentReceived.php#L19-L49) 处理逻辑：

```php
if (! in_array($event->document->status, ['partial', 'paid'])) {
    $event->document->status = 'received';
    if ($event->document->amount == 0) {
        $event->document->status = 'paid';   // 零金额直接视为已付
    }
    $event->document->save();
}
```

**关键点**：这一步只改 `status`，不写 `transactions` 表——它代表"货物/账单已到，应付账款确认"，而非"银行已付款"。

### 4.3 实际付款：费用入账的真实入口

采购付款有两个入口，但最终都汇聚到 `CreateBankingDocumentTransaction` Job：

#### 入口 A：模态框手动添加支付（主要路径）

这是采购付款的**主要路径**。在 Bill 详情页点击"Add Payment"按钮，弹出模态框，提交后由 [DocumentTransactions.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php#L128-L143) 处理：

```php
public function store(Document $document, Request $request)
{
    $response = $this->ajaxDispatch(
        new CreateBankingDocumentTransaction($document, $request)
    );
    // ...
}
```

**⚠️ 没有事件派发，直接调用 Job。**

#### 入口 B：从供应商页面跳转创建费用

在 [Vendors.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Purchases/Vendors.php#L267) 中，有一个创建费用的快捷入口：

```php
return redirect()->route('transactions.create', ['type' => 'expense'])
    ->withInput($data);
```

这会跳转到通用的费用创建页面，不关联具体 Bill。

### 4.4 费用类型如何确定：来自配置 + 隐藏字段

付款模态框的 type 不是在后端动态判断的，而是通过视图的**隐藏字段**直接提交的。

视图 [payment.blade.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/resources/views/modals/documents/payment.blade.php#L138)：

```html
<x-form.input.hidden 
    name="type" 
    :value="config('type.document.' . $document->type . '.transaction_type')" 
/>
```

配置 [type.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/config/type.php#L255-L258)：

```php
Document::BILL_TYPE => [
    'category_type'             => Category::EXPENSE_TYPE,
    'transaction_type'          => Transaction::EXPENSE_TYPE,  // 'expense'
    // ...
    'inventory_stock_action'    => 'increase',
],
```

**完整链路**：
```
Bill 详情页 → "Add Payment" 按钮
    ↓
DocumentTransactions::create() → 渲染模态框
    ↓
隐藏字段 type = config('type.document.bill.transaction_type') = 'expense'
    ↓
用户提交表单
    ↓
DocumentTransactions::store()
    ↓
CreateBankingDocumentTransaction Job
    ↓
CreateTransaction Job → 写入 transactions(type='expense')
```

### 4.5 CreateTransaction 中的类型兜底逻辑

虽然采购付款的 type 在表单中就已经确定，但 [CreateTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateTransaction.php#L20-L29) 中还有一层兜底判断：

```php
if (! array_key_exists($this->request->get('type'), config('type.transaction'))) {
    $isExpense = str_contains((string) $this->request->get('type', ''), 'expense');
    $isRecurring = ! empty($this->request->get('recurring_frequency')) 
        && $this->request->get('recurring_frequency') !== 'no';

    $type = $isExpense
        ? ($isRecurring ? Transaction::EXPENSE_RECURRING_TYPE : Transaction::EXPENSE_TYPE)
        : ($isRecurring ? Transaction::INCOME_RECURRING_TYPE : Transaction::INCOME_TYPE);

    $this->request->merge(['type' => $type]);
}
```

这段逻辑的作用：如果传入的 type 不在 `config('type.transaction')` 的键中（即不是已注册的事务类型），则通过字符串包含 `'expense'` 来判断是收入还是支出。对于正常的采购付款路径，type 已经是 `'expense'`，会直接通过第一层检查，不会走到兜底逻辑。

### 4.6 核心 Job：CreateBankingDocumentTransaction

费用入账的真正业务逻辑在 [CreateBankingDocumentTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L30-L131)：

```
handle() 流程:
┌─ event: DocumentTransactionCreating
├─ prepareRequest()  // 填充默认值
│   ├─ 若无 amount → 自动计算 = 账单总额 - 已付
│   ├─ company_id / document_id / contact_id ← 从 bill 继承
│   ├─ category_id ← bill 的 category_id
│   ├─ account_id ← setting('default.account')
│   ├─ payment_method ← setting('default.payment_method')
│   ├─ paid_at ← 当前时间（若未指定）
│   └─ currency_code / currency_rate ← 账单币种
├─ checkAmount()  // 超额支付校验 & 更新 Bill 状态
│   ├─ 计算：total_amount = bill.amount - bill.paid
│   ├─ 比较：本次支付金额 vs 剩余应付
│   │   ├─ 超出 → throw Exception（messages.error.over_payment）
│   │   ├─ 相等 → bill.status = 'paid'
│   │   └─ 小于 → bill.status = 'partial'
│   └─ 更新 bill->save()
├─ DB::transaction
│   ├─ dispatch(CreateTransaction) → 写入 transactions 表
│   ├─ bill->save()  // 保存状态
│   └─ dispatch(CreateDocumentHistory) → 历史记录
└─ event: DocumentTransactionCreated
```

#### 最终落地：CreateTransaction

[CreateTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateTransaction.php#L16-L52) 执行最底层写入：

```
handle():
┌─ event: TransactionCreating
├─ 类型校验（config type 存在则直接用，否则 str_contains 兜底）
├─ DB::transaction
│   ├─ Transaction::create()
│   │   ├─ type = 'expense'       ← 费用入账关键标识
│   │   ├─ document_id = bill.id  ← 关联账单
│   │   ├─ account_id             ← 资金流出账户
│   │   ├─ amount                 ← 支付金额
│   │   ├─ paid_at                ← 支付日期
│   │   ├─ category_id            ← 费用分类（影响损益表）
│   │   └─ ...
│   ├─ 上传附件
│   ├─ dispatch(CreateTransactionTaxes) → 税分摊
│   └─ createRecurring() → 定期支付配置
└─ event: TransactionCreated → 递增流水号
```

**费用入账标志**：`transactions.type = 'expense'` + `document_id = bill.id`

### 4.7 事务联动：Transaction Observer

[Transaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Observers/Transaction.php#L23-L59) 观察 Transaction 的删除事件，自动回滚 Bill 状态：

```php
public function deleted(Model $transaction)
{
    if (! empty($transaction->document_id)) {
        $type = ($transaction->type == 'income') 
            ? Document::INVOICE_TYPE 
            : Document::BILL_TYPE;   // 采购场景
        $this->updateDocument($transaction, $type);
    }
}

protected function updateDocument($transaction, $type)
{
    // ...
    $document->status = 'received';              // 回退为「已收到」
    if ($document->transactions_count > 0) {
        $document->status = 'partial';           // 还有其他支付则为部分付款
    }
    $document->save();
    // dispatch(CreateDocumentHistory)
}
```

### 4.8 取消/作废：删除关联的费用 Transaction

监听器 [MarkDocumentCancelled.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Listeners/Document/MarkDocumentCancelled.php#L20-L41) 触发 Job [CancelDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CancelDocument.php#L20-L34)：

```php
\DB::transaction(function () {
    $this->deleteRelationships($this->model, [
        'transactions', 'recurring'  // 只删 transactions 和 recurring
    ]);
    $this->model->status = 'cancelled';
    $this->model->save();
});
```

**⚠️ 注意**：
- 作废（Cancel）只删除 `transactions` 和 `recurring`，**不删除 `items`（document_items）**
- 即：作废采购单会清除费用入账记录，但**不会减少库存记录**
- `Transaction::mute()`（在 DeleteDocument 中使用）会临时禁用 Observer，以避免重复的状态回写

反过来，监听器 [RestoreDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Listeners/Document/RestoreDocument.php) 仅将状态改回 `draft`（**不会恢复 transactions**，需要重新付款）。

---

## 五、作废 vs 删除：对库存和费用的不同影响

这是最容易混淆的点，专门用一节说明：

### 5.1 CancelDocument（作废）

代码位置：[CancelDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CancelDocument.php#L24-L31)

```php
$this->deleteRelationships($this->model, [
    'transactions', 'recurring'
]);
```

| 影响对象 | 是否删除 | 说明 |
|----------|----------|------|
| `documents` | 否（改状态） | status = 'cancelled' |
| `document_items` | 否 | 库存记录保留 |
| `document_totals` | 否 | 汇总记录保留 |
| `transactions` | ✅ 是 | 费用记账被清除 |
| `recurring` | ✅ 是 | 定期账单配置删除 |
| `histories` | 否 | 历史记录保留 |

**对库存的影响**：❌ 无影响（document_items 保留，库存不变）
**对费用的影响**：✅ 清除（transactions 删除，费用回退）

### 5.2 DeleteDocument（删除）

代码位置：[DeleteDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/DeleteDocument.php#L20-L32)

```php
$this->deleteRelationships($this->model, [
    'items', 'item_taxes', 'histories', 'transactions', 'recurring', 'totals'
]);
$this->model->delete();  // 软删除
```

| 影响对象 | 是否删除 | 说明 |
|----------|----------|------|
| `documents` | ✅ 是（软删） | deleted_at 标记 |
| `document_items` | ✅ 是 | 库存记录删除 |
| `document_totals` | ✅ 是 | 汇总记录删除 |
| `document_item_taxes` | ✅ 是 | 行税记录删除 |
| `transactions` | ✅ 是 | 费用记账被清除 |
| `recurring` | ✅ 是 | 定期账单配置删除 |
| `histories` | ✅ 是 | 历史记录删除 |

**对库存的影响**：✅ 库存回退（document_items 删除）
**对费用的影响**：✅ 清除（transactions 删除，费用回退）

### 5.3 对比总结

| 操作 | 库存变化 | 费用变化 | Bill 状态 | 数据可恢复 |
|------|----------|----------|-----------|-----------|
| 创建 Bill | ➕ 增加 | 不变 | draft | - |
| 标记已收到 | 不变 | 不变 | received | 可回退到 draft |
| 添加支付 | 不变 | ➕ 增加费用 | partial/paid | 可删除支付回退 |
| 作废 (Cancel) | 不变 | ➖ 清除费用 | cancelled | 可恢复到 draft |
| 删除 (Delete) | ➖ 减少 | ➖ 清除费用 | 软删除 | 可恢复（软删除） |

---

## 六、三者完整协作链路图

### 6.1 场景一：创建采购账单 → 确认收到 → 全额支付

```
用户操作: POST /purchases/bills
  │
  ▼
[Bills::store]
  │
  ▼
[CreateDocument Job] ─── DB Transaction ──────────────────┐
  │  1. Document::create(['type'=>'bill', 'status'=>'draft'])
  │  2. [CreateDocumentItemsAndTotals]
  │     │  遍历 items:
  │     │    ├─ [CreateItem] → items 表（若新商品）
  │     │    └─ [CreateDocumentItem]
  │     │         ├─ quantity 计算（支持表达式）
  │     │         └─ 写入 document_items (库存入库记录 ✅)
  │     └─ 写入 document_totals (sub_total/tax/total)
  │  3. Document->update()  // 回写最终 amount
  └──────────────────────────────────────────────────────────┘
  │
  ▼ (event) DocumentCreated → 历史记录/编号递增


用户操作: 点击「标记已收到」
  │
  ▼
[Bills::markReceived]
  │
  ▼ (event) DocumentReceived
  │
  ▼
[MarkDocumentReceived Listener]
  └─ bill.status = 'received'  // 不影响库存与费用
     └─ (若 amount=0 → 'paid')


用户操作: 点击「Add Payment」→ 模态框提交
  │
  ▼
[DocumentTransactions::store]  ← 采购付款主入口
  │
  ▼
[CreateBankingDocumentTransaction Job]
  ├─ prepareRequest() // 继承 bill 的分类/联系人等
  ├─ checkAmount()
  │   └─ bill.status = 'paid' (本次 = 剩余应付)
  ├─ DB Transaction
  │  ├─ [CreateTransaction Job]
  │  │   └─ Transaction::create([
  │  │        'type' => 'expense',        ← 费用入账 ✅
  │  │        'document_id' => bill.id,
  │  │        'amount' => xxx,
  │  │        'category_id' => bill.category_id
  │  │      ])
  │  ├─ bill.save()  // status = 'paid'
  │  └─ CreateDocumentHistory
  └─ (event) DocumentTransactionCreated
```

### 6.2 场景二：作废采购账单 → 回滚费用（保留库存）

```
用户操作: 点击「Cancel」
  │
  ▼
[Bills::markCancelled]
  │
  ▼ (event) DocumentCancelled
  │
  ▼
[MarkDocumentCancelled Listener]
  │
  ▼
[CancelDocument Job] ─── DB Transaction ──────┐
  ├─ deleteRelationships(['transactions', 'recurring'])
  │   └─ DELETE FROM transactions WHERE document_id = ?
  │      → 费用记录被撤销 ❌
  │   (⚠️ 不删除 document_items，库存保留)
  └─ bill.status = 'cancelled'
```

### 6.3 场景三：删除采购账单 → 库存和费用全部清除

```
用户操作: 点击「Delete」
  │
  ▼
[Bills::destroy]
  │
  ▼
[DeleteDocument Job] ─── DB Transaction ──────┐
  ├─ Transaction::mute()  // 禁用 Observer，避免重复回写
  ├─ deleteRelationships([
  │     'items',         ← document_items 删除 → 库存回退 ❌
  │     'item_taxes',
  │     'histories',
  │     'transactions',  ← 费用删除 ❌
  │     'recurring',
  │     'totals'
  │ ])
  ├─ $this->model->delete()  // 软删除
  └─ Transaction::unmute()
```

### 6.4 数据联动汇总表（修正版）

| 操作 | documents (status) | document_items (库存) | transactions (费用) | 触发事件 |
|------|--------------------|----------------------|---------------------|----------|
| 创建 Bill | `draft` | ✅ **INSERT** (入库) | - | DocumentCreated |
| 标记已收到 | `received` | - | - | DocumentReceived |
| 部分付款 | `partial` | - | ✅ **INSERT** (expense) | DocumentTransactionCreating / Created |
| 再次付款（结清） | `paid` | - | ✅ **INSERT** | 同上 |
| 修改 Bill | 不变 | ❌ **DELETE + REINSERT**（先删后重建） | -（若未对账） | DocumentUpdated |
| 删除单条支付 (Observer) | `received`/`partial` | - | ❌ **DELETE** | (Observer deleted) |
| 取消 Bill (Cancel) | `cancelled` | -（保留） | ❌ **全 DELETE** | DocumentCancelled |
| 恢复 Bill | `draft` | -（保留） | -（不恢复支付） | DocumentRestored |
| 删除 Bill (Delete) | softDelete | ❌ **全 DELETE** | ❌ **全 DELETE** | DocumentDeleted |

---

## 七、关键设计决策解析

### 7.1 库存为何采用动态计算而非字段保存？

**优点**：
1. **避免并发更新冲突**：高并发场景下无需对 `items.quantity` 行锁
2. **完全可追溯**：每一笔库存变动都有对应的 Bill/Invoice 单据来源
3. **无冗余数据**：修改历史单据时只需重建 `document_items`，无需逆向计算库存调整
4. **支持审计**：库存差异可追溯至具体单据

**代价**：
- 查询实时库存需聚合两个关联（bill_items + invoice_items），性能略差
- 需配合索引优化：`document_items(item_id, type)` 复合索引

### 7.2 Bill 与 Transaction 为何松耦合（非事件驱动）？

**⚠️ 再次澄清**：采购付款**不使用** PaymentReceived 事件，而是直接调用 Job。销售发票的在线支付才使用事件。

采购付款采用直接调用而非事件驱动的原因：
1. **权责清晰**：Bill 代表"应付"（权责发生制），Transaction 代表"实付"（收付实现制）
2. **灵活拆分**：一张 Bill 可分多次付款，每次产生独立 Transaction
3. **后台操作直接可靠**：后台手动添加支付是确定性操作，不需要事件解耦
4. **事件用于在线支付**：PaymentReceived 事件是为在线支付网关回调设计的，需要异步处理

### 7.3 作废不删库存的设计考量

作废（Cancel）只清除费用不清除库存的设计原因：
1. **保留审计痕迹**：即使账单作废，商品的入库记录仍然存在，可追溯
2. **业务语义**："作废"通常指"这笔采购取消了付款"，但货物可能已经入库
3. **与删除区分**：如果需要彻底清除，使用 Delete 操作
4. **可恢复性**：Cancel 后可 Restore，库存数据无需重新录入

### 7.4 费用入账的 category_id 继承链

```
Bill.category_id
    │
    ├─► (CreateBankingDocumentTransaction.prepareRequest)
    │      $request['category_id'] = $this->model->category_id
    │
    ▼
Transaction.category_id
    │
    └─► 损益报表依据：
         Transaction 与 Category 关联 → 费用分类统计
```

这就是为什么**创建 Bill 时选择的"类别"最终会体现在费用报表中**——它通过 category_id 从 Bill 传递到了 Transaction。

---

## 八、容易混淆的概念对照表

| 概念 | 所在表 | 含义 | 何时变化 |
|------|--------|------|----------|
| 应付账款 | `documents.amount` | 应该付给供应商多少钱 | 创建/修改 Bill 时 |
| 已付金额 | `transactions.amount` 汇总 | 已经实际支付了多少钱 | 添加/删除支付时 |
| 库存入库 | `document_items.quantity` (type=bill) | 采购了多少数量商品 | 创建/修改/删除 Bill 时 |
| 费用记账 | `transactions` (type=expense) | 银行账户实际支出 | 添加/删除支付时 |
| Bill 状态 | `documents.status` | 当前处于什么阶段 | 状态流转时 |

**一句话区分**：
- **document_items** 管"货"（库存多少）
- **transactions** 管"钱"（花了多少钱）
- **document.status** 管"单"（流程到哪一步）

---

## 九、关键源码位置索引

| 层级 | 文件 | 核心职责 |
|------|------|----------|
| **配置** | [type.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/config/type.php) | 类型映射（bill→expense→increase） |
| 控制器 | [Bills.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Purchases/Bills.php) | HTTP 入口（CRUD + 标记状态） |
| 控制器 | [DocumentTransactions.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php) | 付款模态框入口（采购付款主路径） |
| 表单验证 | [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Requests/Document/Document.php) | quantity 表达式预处理 |
| 工具函数 | [helpers.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Utilities/helpers.php#L433-L460) | `calculation_to_quantity()` |
| **创建 Bill** | [CreateDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocument.php) | 主流程编排 |
| **明细+汇总** | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | 遍历 items、建库存记录 |
| **行计算** | [CreateDocumentItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocumentItem.php) | 折扣+5种税计算 |
| **作废 Bill** | [CancelDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CancelDocument.php) | 作废（删 transactions，保留 items） |
| **删除 Bill** | [DeleteDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/DeleteDocument.php) | 删除（删 items + transactions） |
| **创建支付** | [CreateBankingDocumentTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php) | 费用入账编排（含超额校验）|
| **落地费用** | [CreateTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateTransaction.php) | 写入 transactions (expense) |
| Observer | [Transaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Observers/Transaction.php) | 删除支付时回滚 Bill 状态 |
| 事件配置 | [Event.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Providers/Event.php) | 事件-监听器绑定表 |
| 在线支付 | [PaymentController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Abstracts/Http/PaymentController.php#L170-L179) | PaymentReceived 事件触发（仅 Invoice） |
| 视图 | [payment.blade.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/resources/views/modals/documents/payment.blade.php#L138) | type 隐藏字段 |
| 模型 | [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Document/Document.php) | Bill/Invoice 统一模型 |
| 模型 | [DocumentItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Document/DocumentItem.php) | 明细行（quantity 承载） |
| 模型 | [Item.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Common/Item.php) | 商品（bill_items/invoice_items 关联） |
| 模型 | [Transaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Banking/Transaction.php) | 收支流水（expense 费用） |
| 迁移文件 | [2022_05_10_000000_core_v300.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/database/migrations/2022_05_10_000000_core_v300.php#L37-L43) | 删除 items.quantity 的关键迁移 |
| 测试 | [BillsTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/tests/Feature/Purchases/BillsTest.php) | 采购+支付的集成测试 |

---

## 十、一句话总结

> **采购 Bill 创建时，`document_items.quantity` 即记录了库存入库；当用户通过付款模态框手动添加支付时（走 DocumentTransactions 控制器 → CreateBankingDocumentTransaction Job，不经过 PaymentReceived 事件），才会写入一条 `type = 'expense'` 的 Transaction 完成费用入账。作废（Cancel）Bill 只清费用不清库存，删除（Delete）才会两者都清。库存类型、费用类型都由 `config/type.php` 配置决定，视图通过隐藏字段传入后端。**
