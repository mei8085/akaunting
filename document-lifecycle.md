# 单据（Document）生命周期与状态流转机制

## 概述

Akaunting 系统中的单据（Document）采用**事件驱动架构**实现状态流转。核心模型为 [Document](app/Models/Document/Document.php)，通过 `Event → Listener → Job` 的调用链驱动状态变化和业务动作。

单据主要分为两类：
- **Invoice（发票）**：销售方向，状态流转为 `草稿 → 已发送 → 已查看 → 部分付款 → 已付款 / 已作废`
- **Bill（账单）**：采购方向，状态流转为 `草稿 → 已接收 → 部分付款 → 已付款 / 已作废`

---

## 一、状态定义

### 1.1 Invoice 状态集

定义于 [Documents.php](app/Traits/Documents.php#L72-L82) `getDocumentStatuses()` 方法：

| 状态 | 说明 |
|------|------|
| `draft` | 草稿 |
| `sent` | 已发送 |
| `viewed` | 已查看 |
| `approved` | 已批准 |
| `partial` | 部分付款 |
| `paid` | 已付款 |
| `overdue` | 已逾期 |
| `unpaid` | 未付款 |
| `cancelled` | 已作废 |

### 1.2 Bill 状态集

| 状态 | 说明 |
|------|------|
| `draft` | 草稿 |
| `received` | 已接收 |
| `partial` | 部分付款 |
| `paid` | 已付款 |
| `overdue` | 已逾期 |
| `unpaid` | 未付款 |
| `cancelled` | 已作废 |

---

## 二、事件监听配置

所有事件-监听器映射定义于 [Event.php](app/Providers/Event.php) 服务提供者：

```php
protected $listen = [
    DocumentCreated::class => [
        CreateDocumentCreatedHistory::class,      // 创建历史记录
        IncreaseNextDocumentNumber::class,        // 递增单据编号
        SettingFieldCreated::class,               // 处理扩展字段
    ],
    DocumentSent::class => [
        MarkDocumentSent::class,                  // 标记为已发送
    ],
    DocumentMarkedSent::class => [
        MarkDocumentSent::class,                  // 同上，复用监听器
    ],
    DocumentReceived::class => [
        MarkDocumentReceived::class,              // 标记为已接收
    ],
    DocumentViewed::class => [
        MarkDocumentViewed::class,                // 标记为已查看
        SendDocumentViewNotification::class,      // 发送查看通知
    ],
    DocumentCancelled::class => [
        MarkDocumentCancelled::class,             // 标记为已作废
    ],
    PaymentReceived::class => [
        CreateDocumentTransaction::class,         // 创建付款交易
        SendDocumentPaymentNotification::class,   // 发送付款通知
    ],
    // ...
];
```

---

## 三、状态流转详情

### 3.1 创建：草稿（Draft）状态

**入口**：[CreateDocument](app/Jobs/Document/CreateDocument.php) Job

```
Controller::store()
    ↓
CreateDocument Job
    ├─ event(new DocumentCreating($request))
    ├─ DB::transaction()
    │   ├─ Document::create($request)        // status 默认值为 draft
    │   ├─ 上传附件
    │   ├─ CreateDocumentItemsAndTotals Job  // 创建明细和合计
    │   └─ 创建循环单据
    └─ event(new DocumentCreated($model, $request))
           ├─ CreateDocumentCreatedHistory Listener  → 创建历史记录
           ├─ IncreaseNextDocumentNumber Listener    → 递增单据编号
           └─ SettingFieldCreated Listener           → 处理扩展字段
```

**核心代码** ([CreateDocument.php#L35-L54](app/Jobs/Document/CreateDocument.php#L35-L54))：
```php
\DB::transaction(function () {
    $this->model = Document::create($this->request->all());
    // 上传附件...
    $this->dispatch(new CreateDocumentItemsAndTotals($this->model, $this->request));
    $this->model->update($this->request->all());
    $this->model->createRecurring($this->request->all());
});
event(new DocumentCreated($this->model, $this->request));
```

---

### 3.2 发送：草稿 → 已发送（Draft → Sent）

**适用类型**：Invoice

**入口**：
- [Invoices::markSent()](app/Http/Controllers/Sales/Invoices.php#L259-L268) - 手动标记已发送
- [SendDocument](app/Jobs/Document/SendDocument.php) Job - 邮件发送

```
Controller::markSent()
    ↓
event(new DocumentMarkedSent($invoice))
    ↓
MarkDocumentSent Listener
    ├─ 检查状态不是 partial/paid
    ├─ $document->status = 'sent'
    ├─ 特殊：金额为 0 时直接标记为 'paid'
    ├─ $document->save()
    └─ CreateDocumentHistory Job → 记录 "已标记为已发送"
```

**邮件发送路径** ([SendDocument.php#L17-L27](app/Jobs/Document/SendDocument.php#L17-L27))：
```php
public function handle(): void
{
    event(new DocumentSending($this->document));
    $notification = config('type.document.' . $this->document->type . '.notification.class');
    $this->document->contact->notify(new $notification($this->document, 'invoice_new_customer', true));
    event(new DocumentSent($this->document));  // 触发状态更新
}
```

---

### 3.3 接收：草稿 → 已接收（Draft → Received）

**适用类型**：Bill

**入口**：[Bills::markReceived()](app/Http/Controllers/Purchases/Bills.php#L229-L238)

```
Controller::markReceived()
    ↓
event(new DocumentReceived($bill))
    ↓
MarkDocumentReceived Listener
    ├─ 检查状态不是 partial/paid
    ├─ $document->status = 'received'
    ├─ 特殊：金额为 0 时直接标记为 'paid'
    ├─ $document->save()
    └─ CreateDocumentHistory Job → 记录 "已标记为已接收"
```

**核心代码** ([MarkDocumentReceived.php#L19-L49](app/Listeners/Document/MarkDocumentReceived.php#L19-L49))：
```php
if (! in_array($event->document->status, ['partial', 'paid'])) {
    $event->document->status = 'received';
    if ($event->document->amount == 0) {
        $event->document->status = 'paid';
    }
    $event->document->save();
}
$this->dispatch(new CreateDocumentHistory(...));
```

---

### 3.4 查看：已发送 → 已查看（Sent → Viewed）

**适用类型**：Invoice

**入口**：客户在门户查看发票时触发 `DocumentViewed` 事件

```
DocumentViewed Event
    ↓
MarkDocumentViewed Listener
    ├─ 仅当 status == 'sent' 时执行
    ├─ $document->status = 'viewed'
    ├─ $document->save()
    └─ CreateDocumentHistory Job → 记录 "已被查看"
```

**核心代码** ([MarkDocumentViewed.php#L19-L49](app/Listeners/Document/MarkDocumentViewed.php#L19-L49))：
```php
if ($document->status != 'sent') {
    return;
}
$document->status = 'viewed';
$document->save();
$this->dispatch(new CreateDocumentHistory(...));
```

---

### 3.5 付款：→ 部分付款 → 已付款（→ Partial → Paid）

**入口**：
- 在线支付完成：[PaymentController::dispatchPaidEvent()](app/Abstracts/Http/PaymentController.php#L170-L180)
- 手动记录付款：`PaymentReceived` 事件

```
PaymentReceived Event
    ├─ CreateDocumentTransaction Listener
    │   └─ CreateBankingDocumentTransaction Job
    │       ├─ event(new DocumentTransactionCreating())
    │       ├─ prepareRequest()
    │       ├─ checkAmount()  ← 状态更新核心逻辑
    │       │   ├─ 计算已付金额和未付金额
    │       │   ├─ 检查超额支付
    │       │   └─ 更新状态：
    │       │       ├─ 支付金额 == 未付金额 → 'paid'
    │       │       └─ 支付金额 < 未付金额 → 'partial'
    │       ├─ CreateTransaction Job → 创建银行交易记录
    │       ├─ $model->save()
    │       ├─ CreateDocumentHistory Job → 记录付款金额
    │       └─ event(new DocumentTransactionCreated())
    │
    └─ SendDocumentPaymentNotification Listener
        ├─ 通知客户付款成功
        └─ 通知公司管理员
```

**状态更新核心逻辑** ([CreateBankingDocumentTransaction.php#L74-L123](app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123))：
```php
protected function checkAmount(): bool
{
    // 计算已付金额和未付金额
    $this->model->paid_amount = $this->model->paid;
    event(new PaidAmountCalculated($this->model));
    $total_amount = round($this->model->amount - $this->model->paid_amount, $precision);

    $compare = bccomp($amount, $total_amount, $precision);

    if ($compare === 1) {
        throw new \Exception(trans('messages.error.over_payment', ...));
    } else {
        $this->model->status = ($compare === 0) ? 'paid' : 'partial';
    }
    return true;
}
```

---

### 3.6 更新时的状态自动调整

**入口**：[UpdateDocument](app/Jobs/Document/UpdateDocument.php) Job

当更新单据时，如果已有付款记录，系统会自动根据已付金额调整状态：

```php
$this->model->paid_amount = $this->model->paid;
event(new PaidAmountCalculated($this->model));

if ($this->model->paid_amount > 0) {
    if ($this->request['amount'] == $this->model->paid_amount) {
        $this->request['status'] = 'paid';      // 已全额支付
    }
    if ($this->request['amount'] > $this->model->paid_amount) {
        $this->request['status'] = 'partial';   // 部分支付
    }
}
```

---

### 3.7 作废：→ 已作废（→ Cancelled）

**入口**：
- [Invoices::markCancelled()](app/Http/Controllers/Sales/Invoices.php#L277-L286)
- [Bills::markCancelled()](app/Http/Controllers/Purchases/Bills.php#L247-L256)

```
Controller::markCancelled()
    ↓
event(new DocumentCancelled($document))
    ↓
MarkDocumentCancelled Listener
    ├─ CancelDocument Job
    │   ├─ authorize()  ← 安全检查
    │   │   └─ 禁止作废已对账的单据
    │   └─ DB::transaction()
    │       ├─ deleteRelationships(transactions, recurring)
    │       ├─ $model->status = 'cancelled'
    │       └─ $model->save()
    └─ CreateDocumentHistory Job → 记录 "已标记为已作废"
```

**安全检查** ([CancelDocument.php#L41-L49](app/Jobs/Document/CancelDocument.php#L41-L49))：
```php
public function authorize(): void
{
    if ($this->model->transactions()->isReconciled()->count()) {
        throw new \Exception(trans('messages.warning.reconciled_doc', ...));
    }
}
```

---

## 四、状态流转总图

```
Invoice 路径:
   新建 → draft ──┐
                  ├─ markSent() → DocumentMarkedSent → sent
                  ├─ 邮件发送 → DocumentSent → sent
                  │                       ↓
                  │               客户查看 → DocumentViewed → viewed
                  │                       ↓
                  └───────────────────────┼─────────────────┐
                                          ↓                 │
                            部分付款 → PaymentReceived → partial
                                          ↓                 │
                            全部付款 → PaymentReceived → paid ◄┘
                                          ↓
                            手动作废 → DocumentCancelled → cancelled


Bill 路径:
   新建 → draft ──┐
                  ├─ markReceived() → DocumentReceived → received
                  │                       ↓
                  └───────────────────────┼─────────────────┐
                                          ↓                 │
                            部分付款 → PaymentReceived → partial
                                          ↓                 │
                            全部付款 → PaymentReceived → paid ◄┘
                                          ↓
                            手动作废 → DocumentCancelled → cancelled
```

---

## 五、关键设计模式

### 5.1 事件驱动架构（Event-Driven Architecture）

- **解耦**：状态变更的触发点（Controller）与业务逻辑（Listener/Job）完全分离
- **可扩展**：新增业务动作只需添加新的 Listener，无需修改核心流程
- **可追溯**：每个状态变化都通过 `DocumentHistory` 留痕

### 5.2 领域事件模式

每个状态变化都有对应的领域事件：
- `DocumentCreating` / `DocumentCreated`
- `DocumentSending` / `DocumentSent`
- `DocumentCancelled`
- `PaymentReceived`

### 5.3 状态守卫（Status Guard）

每个状态变更前都有前置检查：
- 作废前检查是否已对账
- 联系人变更时检查状态是否已锁定（`sent`, `received`, `viewed`, `partial`, `paid`, `overdue`, `unpaid`, `cancelled`）
- 查看状态仅能从 `sent` 转入 `viewed`

---

## 六、核心代码文件索引

| 类型 | 文件路径 |
|------|---------|
| 模型 | [Document.php](app/Models/Document/Document.php) |
| 模型 | [DocumentHistory.php](app/Models/Document/DocumentHistory.php) |
| 特性 | [Documents.php](app/Traits/Documents.php) |
| 服务提供者 | [Event.php](app/Providers/Event.php) |
| 创建 Job | [CreateDocument.php](app/Jobs/Document/CreateDocument.php) |
| 更新 Job | [UpdateDocument.php](app/Jobs/Document/UpdateDocument.php) |
| 发送 Job | [SendDocument.php](app/Jobs/Document/SendDocument.php) |
| 作废 Job | [CancelDocument.php](app/Jobs/Document/CancelDocument.php) |
| 付款 Job | [CreateBankingDocumentTransaction.php](app/Jobs/Banking/CreateBankingDocumentTransaction.php) |
| 监听器 | [MarkDocumentSent.php](app/Listeners/Document/MarkDocumentSent.php) |
| 监听器 | [MarkDocumentReceived.php](app/Listeners/Document/MarkDocumentReceived.php) |
| 监听器 | [MarkDocumentViewed.php](app/Listeners/Document/MarkDocumentViewed.php) |
| 监听器 | [MarkDocumentCancelled.php](app/Listeners/Document/MarkDocumentCancelled.php) |
| 控制器 | [Invoices.php](app/Http/Controllers/Sales/Invoices.php) |
| 控制器 | [Bills.php](app/Http/Controllers/Purchases/Bills.php) |
