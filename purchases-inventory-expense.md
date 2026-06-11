# 采购单据与库存增减、费用入账协作链路分析

> 本文档基于 Akaunting v3.x 源码，深入分析采购账单（Bill）从创建到支付的全链路中，**库存增减记录**与**费用记账（Transaction）**是如何协作触发的。

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

## 三、库存增减机制：从 quantity 字段到动态化

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

### 3.1 当前库存跟踪机制

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

### 3.2 采购明细创建链路

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

### 3.3 写入 DocumentTotals：账单汇总

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

## 四、费用入账（Transaction）：从「收到账单」到「实际付款」

采购 Bill 与费用 Transaction 是**松耦合**的 —— 它们通过**事件驱动**在适当时机关联。

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

### 4.3 实际付款：创建 Expense Transaction（费用入账核心）

付款有两个入口，最终汇聚到同一个 Job：

#### 入口 A：模态框手动添加支付

控制器 [DocumentTransactions.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php#L128-L143)：

```php
public function store(Document $document, Request $request)
{
    $response = $this->ajaxDispatch(
        new CreateBankingDocumentTransaction($document, $request)
    );
    // ...
}
```

#### 入口 B：PaymentReceived 事件驱动

在 [Event.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Providers/Event.php#L73-L76) 中绑定：

```php
PaymentReceived::class => [
    CreateDocumentTransaction::class,      // ← 创建 Transaction
    SendDocumentPaymentNotification::class,
],
```

监听器 [CreateDocumentTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Listeners/Document/CreateDocumentTransaction.php#L20-L50)：

```php
public function handle(Event $event)
{
    $this->dispatch(new CreateBankingDocumentTransaction(
        $event->document, $event->request
    ));
}
```

#### 核心 Job：CreateBankingDocumentTransaction

费用入账的真正逻辑在 [CreateBankingDocumentTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L30-L131)：

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
├─ 判断 transaction type（核心！）
│   └─ bill 类型 → type = 'expense'
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

### 4.4 事务联动：Transaction Observer

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

### 4.5 取消/作废：删除关联的费用 Transaction

监听器 [MarkDocumentCancelled.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Listeners/Document/MarkDocumentCancelled.php#L20-L41) 触发 Job [CancelDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CancelDocument.php#L20-L34)：

```php
\DB::transaction(function () {
    $this->deleteRelationships($this->model, [
        'transactions',   // ← 级联删除所有费用 Transaction
        'recurring'
    ]);
    $this->model->status = 'cancelled';
    $this->model->save();
});
```

**注意**：删除 transactions 时，`Transaction::mute()`（在 DeleteDocument 中）会临时禁用 Observer，以避免重复的状态回写。

反过来，[RestoreDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Listeners/Document/RestoreDocument.php) 仅将状态改回 `draft`（**不会恢复 transactions**，需要重新付款）。

---

## 五、三者完整协作链路图

### 5.1 场景一：创建采购账单 → 确认收到 → 全额支付

```
用户操作: POST /bills
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


用户操作: 模态框添加支付（点击"Add Payment"）
  │
  ▼
[DocumentTransactions::store]
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

### 5.2 场景二：取消采购账单 → 回滚

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
  ├─ deleteRelationships(['transactions'])
  │   └─ DELETE FROM transactions WHERE document_id = ?
  │      → 费用记录被撤销 ❌
  └─ bill.status = 'cancelled'
```

### 5.3 数据联动汇总表

| 操作 | documents (status) | document_items (库存) | transactions (费用) | 触发事件 |
|------|--------------------|----------------------|---------------------|----------|
| 创建 Bill | `draft` | ✅ **INSERT** (入库) | - | DocumentCreated |
| 标记已收到 | `received` | - | - | DocumentReceived |
| 部分付款 | `partial` | - | ✅ **INSERT** (expense) | PaymentReceived → TransactionCreated |
| 再次付款（结清） | `paid` | - | ✅ **INSERT** | 同上 |
| 修改 Bill | 不变 | ❌ **DELETE + REINSERT**（先删后重建） | -（若未对账） | DocumentUpdated |
| 删除 Transaction (Observer) | `received`/`partial` | - | ❌ **DELETE** | (Observer deleted) |
| 取消 Bill | `cancelled` | -（保留，但不再参与库存计算）| ❌ **全 DELETE** | DocumentCancelled |
| 恢复 Bill | `draft` | - | -（不恢复支付） | DocumentRestored |
| 删除 Bill | softDelete | softDelete | softDelete | DocumentDeleted |

---

## 六、关键设计决策解析

### 6.1 库存为何采用动态计算而非字段保存？

**优点**：
1. **避免并发更新冲突**：高并发场景下无需对 `items.quantity` 行锁
2. **完全可追溯**：每一笔库存变动都有对应的 Bill/Invoice 单据来源
3. **无冗余数据**：修改历史单据时只需重建 `document_items`，无需逆向计算库存调整
4. **支持审计**：库存差异可追溯至具体单据

**代价**：
- 查询实时库存需聚合两个关联（bill_items + invoice_items），性能略差
- 需配合索引优化：`document_items(item_id, type)` 复合索引

### 6.2 Bill 与 Transaction 为何松耦合（事件驱动）？

1. **权责清晰**：Bill 代表"应付/应收"（权责发生制），Transaction 代表"实收/实付"（收付实现制）
2. **灵活拆分**：一张 Bill 可分多次付款，每次产生独立 Transaction
3. **可扩展**：模块可监听 PaymentReceived 事件做扩展（如发送通知、集成支付网关）
4. **审计友好**：付款历史完整记录，与账单主数据分离

### 6.3 费用入账的 category_id 继承链

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

## 七、关键源码位置索引

| 层级 | 文件 | 核心职责 |
|------|------|----------|
| 控制器 | [Bills.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Purchases/Bills.php) | HTTP 入口（CRUD + 标记状态） |
| 控制器 | [DocumentTransactions.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php) | 支付模态框入口 |
| 表单验证 | [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Http/Requests/Document/Document.php) | quantity 表达式预处理 |
| 工具函数 | [helpers.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Utilities/helpers.php#L433-L460) | `calculation_to_quantity()` |
| **创建 Bill** | [CreateDocument.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocument.php) | 主流程编排 |
| **明细+汇总** | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | 遍历 items、建库存记录 |
| **行计算** | [CreateDocumentItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Document/CreateDocumentItem.php) | 折扣+5种税计算 |
| **创建支付** | [CreateBankingDocumentTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php) | 费用入账编排（含超额校验）|
| **落地费用** | [CreateTransaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Jobs/Banking/CreateTransaction.php) | 写入 transactions (expense) |
| Observer | [Transaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Observers/Transaction.php) | 删除支付时回滚 Bill 状态 |
| 事件配置 | [Event.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Providers/Event.php) | 事件-监听器绑定表 |
| 模型 | [Document.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Document/Document.php) | Bill/Invoice 统一模型 |
| 模型 | [DocumentItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Document/DocumentItem.php) | 明细行（quantity 承载） |
| 模型 | [Item.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Common/Item.php) | 商品（bill_items/invoice_items 关联） |
| 模型 | [Transaction.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/app/Models/Banking/Transaction.php) | 收支流水（expense 费用） |
| 迁移文件 | [2022_05_10_000000_core_v300.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/database/migrations/2022_05_10_000000_core_v300.php#L37-L43) | 删除 items.quantity 的关键迁移 |
| 测试 | [BillsTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/130-akaunting/tests/Feature/Purchases/BillsTest.php) | 采购+支付的集成测试 |

---

## 八、一句话总结

> **采购 Bill 创建时，`document_items.quantity` 即记录了库存入库；当用户触发支付（通过模态框或事件），`CreateBankingDocumentTransaction` Job 才会写入一条 `type = 'expense'` 的 Transaction 完成费用入账。二者通过事件驱动解耦，Bill 管应付，Transaction 管实付，document_items 管库存，三线并行又彼此关联。**
