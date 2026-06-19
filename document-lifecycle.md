# 单据（Document）生命周期与状态流转机制

## 概述

Akaunting 系统中的单据（Document）采用**事件驱动架构**实现状态流转。核心模型为 [Document](app/Models/Document/Document.php)，通过 `Event → Listener → Job` 的调用链驱动状态变化和业务动作。

单据主要分为两类：
- **Invoice（发票）**：销售方向，状态流转为 `草稿 → 已发送 → 已查看 → 部分付款 → 已付款 / 已作废`
- **Bill（账单）**：采购方向，状态流转为 `草稿 → 已接收 → 部分付款 → 已付款 / 已作废`

> **关于已批准（approved）/ 已确认（confirmed）状态**：这两种状态在语言翻译和状态守卫中均有定义，属于**为模块扩展预留的状态**（如报价单、采购订单等单据类型需要审批流程）。在 Invoice 和 Bill 的核心业务流程中，**当前版本没有提供直接进入 approved/confirmed 的入口**，而是把发送（sent）/ 接收（received）当作业务上的"确认"动作。它们在状态守卫中被当作"终态"看待，用于防止重复触发通知。

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

### 3.2 发票发送/确认：草稿 → 已发送（Draft → Sent）

**适用类型**：Invoice

**入口（共 3 个）**：

| 入口类型 | 代码位置 | 触发场景 |
|---------|---------|---------|
| 手动标记（单条） | [Invoices::markSent()](app/Http/Controllers/Sales/Invoices.php#L259-L268) | 管理员在详情页点击"标记已发送" |
| 批量操作 | [Invoices::sent()](app/BulkActions/Sales/Invoices.php#L102-L113) | 在列表页批量勾选后选择"标记已发送" |
| 邮件发送 Job | [SendDocument](app/Jobs/Document/SendDocument.php) | 创建时勾选"发送邮件"或手动点击"发送邮件" |

> **业务含义**：在 Invoice 核心流程中，`sent` 状态即是"已确认"状态，表示发票已正式发出给客户，相当于业务上的"确认"动作。`approved` 状态为模块扩展预留，核心流程未使用。

#### 代码走向（手动标记单条）

```
Invoices::markSent($invoice)
    ↓
event(new DocumentMarkedSent($invoice))
    ↓
┌─ Event.php 监听配置 ─────────────────────────────────────┐
│  DocumentMarkedSent → MarkDocumentSent                    │
└───────────────────────────────────────────────────────────┘
    ↓
MarkDocumentSent Listener
    ├─ 状态守卫：if in_array(status, ['partial', 'paid']) 则不改 status
    ├─ $document->status = 'sent'
    ├─ 特殊：金额为 0 → status = 'paid'（零金额直接走完）
    ├─ $document->save()
    └─ dispatch(new CreateDocumentHistory(...))
           └─ 记录描述："Invoice 已标记为已发送"
```

#### 代码走向（发送邮件）

[SendDocument.php#L17-L27](app/Jobs/Document/SendDocument.php#L17-L27)：
```
event(new DocumentSending($document))     // 发送前事件
    ↓
$contact->notify(new Notification(...))   // 实际发送邮件给客户（带 PDF 附件）
    ↓
event(new DocumentSent($document))       // 发送后事件 → 触发 MarkDocumentSent 监听器
                                          // （与手动标记共用同一个监听器）
```

#### 影响面汇总

| 维度 | 影响 |
|------|------|
| **付款** | 进入 `sent` 状态后，才可在门户页面看到付款按钮，客户可在线支付 |
| **状态流转** | `draft` → `sent`；后续可被 `viewed`、`partial`、`paid`、`cancelled` 覆盖 |
| **历史记录** | 创建一条 `document_histories`，status='sent' |
| **通知** | 手动标记：无额外通知；邮件发送：客户收到 `invoice_new_customer` 邮件（含 PDF） |
| **编号** | 不触发编号递增（编号在创建时 DocumentCreated 已递增） |

---

### 3.3 账单接收：草稿 → 已接收（Draft → Received）

**适用类型**：Bill

**入口（共 3 个）**：

| 入口类型 | 代码位置 | 触发场景 |
|---------|---------|---------|
| 手动标记（单条） | [Bills::markReceived()](app/Http/Controllers/Purchases/Bills.php#L229-L238) | 管理员在详情页点击"标记已接收" |
| 批量操作 | [Bills::received()](app/BulkActions/Purchases/Bills.php#L96-L106) | 在列表页批量勾选后选择"标记已接收" |
| 循环账单自动生成 | [SendDocumentRecurringNotification](app/Listeners/Document/SendDocumentRecurringNotification.php#L40-L42) | 定时任务生成循环账单时，按 `auto_send` 配置自动触发 |

> **配置说明**：Bill 的 `auto_send` 配置为 `DocumentReceived::class`（见 [type.php#L268](config/type.php#L268)），因此循环账单生成后会自动进入 received 状态。

#### 完整代码走向

```
入口（任选其一）
    ↓
event(new DocumentReceived($bill))
    ↓
┌─ Event.php 监听配置 ─────────────────────────────────────┐
│  DocumentReceived → MarkDocumentReceived                  │
└───────────────────────────────────────────────────────────┘
    ↓
MarkDocumentReceived Listener ([L19-L49](app/Listeners/Document/MarkDocumentReceived.php#L19-L49))
    ├─ 状态守卫：if in_array(status, ['partial', 'paid']) 则跳过
    ├─ $document->status = 'received'
    ├─ 特殊：金额为 0 → status = 'paid'
    ├─ $document->save()
    └─ dispatch(new CreateDocumentHistory(...))
           └─ 记录描述："Bill 已标记为已接收"
```

#### 影响面汇总

| 维度 | 影响 |
|------|------|
| **付款** | 进入 `received` 状态后，表示确认该账单，可开始记录付款（创建 expense 交易） |
| **状态流转** | `draft` → `received`；后续可被 `partial`、`paid`、`cancelled` 覆盖 |
| **历史记录** | 创建一条 `document_histories`，status='received' |
| **通知** | Bill 的 `notify_contact=false`（见 [type.php#L265](config/type.php#L265)），不会给供应商发邮件 |
| **循环单据** | 循环 Bill 生成后自动触发本事件，自动变为 received |

---

### 3.4 发票查看：已发送 → 已查看（Sent → Viewed）

**适用类型**：Invoice

**入口（共 3 个）**：

| 入口类型 | 代码位置 | 触发场景 |
|---------|---------|---------|
| 门户查看（登录） | [Portal\Invoices::show()](app/Http/Controllers/Portal/Invoices.php#L63) | 客户通过账号登录门户进入发票详情 |
| 签名链接查看（未登录） | [Portal\Invoices::signed()](app/Http/Controllers/Portal/Invoices.php#L187-L189) | 客户点击邮件中的签名链接（访客或本人查看都触发） |
| 测试工厂初始化 | [DocumentFactory](database/factories/Document.php#L343-L346) | 仅在测试环境中，seed 数据时触发 |

> **签名链接守卫条件**：signed() 方法有判断（见 [L187-L189](app/Http/Controllers/Portal/Invoices.php#L187-L189)）：仅当 `user()为空（访客）` 或 `user()->id == $invoice->contact->user_id（确实是该客户本人）` 才会记录查看事件，防止公司管理员自己打开链接误触发。

#### 完整代码走向

```
入口（任选其一）
    ↓
event(new DocumentViewed($invoice))
    ↓
┌─ Event.php 监听配置 ──────────────────────────────────────────────┐
│  DocumentViewed → [                                               │
│    1. MarkDocumentViewed          ← 改状态 + 留历史                │
│    2. SendDocumentViewNotification ← 给公司管理员发通知             │
│  ]                                                                 │
└────────────────────────────────────────────────────────────────────┘
    │
    ├─→ MarkDocumentViewed Listener ([L19-L49](app/Listeners/Document/MarkDocumentViewed.php#L19-L49))
    │     ├─ 状态守卫：$document->status != 'sent' → 直接 return
    │     │           （已看过或更后状态，不重复记）
    │     ├─ $document->status = 'viewed'
    │     ├─ $document->save()
    │     └─ dispatch(new CreateDocumentHistory(...))
    │            └─ 记录描述："Invoice 已被查看"
    │
    └─→ SendDocumentViewNotification Listener ([L18-L50](app/Listeners/Document/SendDocumentViewNotification.php#L18-L50))
          ├─ 终态守卫：if status in ['viewed','approved','received',
          │                          'refused','partial','paid',
          │                          'cancelled','voided',
          │                          'completed','refunded'] → return
          │   （防止重复发送通知）
          ├─ 从配置读取 notification class 和 notify_user 开关
          ├─ 若 notify_user=false → return
          └─ foreach($document->company->users as $user)：
               有权限的用户 → notify("invoice_view_admin" 模板通知)
```

#### 影响面汇总

| 维度 | 影响 |
|------|------|
| **付款** | 不影响付款功能（viewed 与 sent 一样可付款，都属于"待付款"状态） |
| **状态流转** | 仅能从 `sent` 转入 `viewed`；不能从 draft 或其他状态转入 |
| **历史记录** | 创建一条 `document_histories`，status='viewed' |
| **通知** | 给公司内有权限的管理员发送"客户已查看"通知；不给客户发；终态守卫防止重复 |
| **approved 的作用** | 虽然未在核心流程中使用，但在终态守卫列表中出现。若模块扩展将 Invoice 设为 approved，系统会把它视为"已确认终态"，不再触发查看通知 |

---

### 3.4.1 已批准（Approved）/ 已确认（Confirmed）状态说明

**存在位置**：
- 语言翻译：[documents.php#L32](resources/lang/en-US/documents.php#L32) 和 [L60](resources/lang/en-US/documents.php#L60)
- Invoice 状态列表：[Documents.php#L72-L82](app/Traits/Documents.php#L72-L82) 中列有 `approved`
- 终态守卫：[SendDocumentViewNotification.php#L22-L25](app/Listeners/Document/SendDocumentViewNotification.php#L22-L25) 把 approved 列入"已确认终态"

**当前版本行为（Invoice/Bill 核心流程）**：
- ❌ 没有 `DocumentApproved` / `DocumentConfirmed` 事件
- ❌ 没有 `MarkDocumentApproved` / `MarkDocumentConfirmed` 监听器
- ❌ 没有 `markApproved()` / `markConfirmed()` 控制器方法
- ❌ 控制器的行操作中未提供"批准/确认"按钮

**业务上"确认"动作的替代**：
在 Invoice/Bill 的核心流程中，把以下状态视为确认：
- Invoice：`sent`（已发送）即表示业务上已确认发出
- Bill：`received`（已接收）即表示业务上已确认接收

**模块扩展方向**：
如果开发"报价单（Quote/Estimate）"、"采购订单（Purchase Order）"等需要独立审批的单据类型，可以按以下方式接入：
1. 在自定义模块 Service Provider 中补充 Event 监听：
   ```php
   DocumentApproved::class => [YourModule\Listeners\MarkDocumentApproved::class]
   ```
2. 在控制器中加入 `markApproved()`，调用 `event(new DocumentApproved($doc))`
3. 系统会自动把 `approved` 作为终态，不再触发查看通知（终态守卫中已包含该状态，无需修改核心代码）

---

### 3.5 付款完整链路：在线支付 → 确认 → 部分付款/已付款

付款流程分为 **3 大阶段**：**①确认地址生成** → **②请求流转到支付模块** → **③支付成功触发事件并更新状态**。

---

#### 3.5.1 阶段一：确认地址（signed URL）生成

signed URL 使用 Laravel 的 `URL::signedRoute()` 生成，附带 `signature` 查询参数防篡改，不需要客户登录即可访问。

**生成位置（共 5 处）**：

| 场景 | 代码位置 | 生成内容 |
|------|---------|---------|
| 邮件通知发送给客户 | [Invoice.php#L155](app/Notifications/Sale/Invoice.php#L155) | `signed.invoices.show`（发票详情查看地址） |
| 付款完成通知客户 | [PaymentReceived.php#L138](app/Notifications/Portal/PaymentReceived.php#L138) | `signed.invoices.show` |
| 查看链接签名页初始化 payment_actions | [Portal\Invoices@signed#L175-L180](app/Http/Controllers/Portal/Invoices.php#L175-L180) | 循环 `Modules::getPaymentMethods()`，为每个支付方法生成 `signed.{alias}.invoices.show` 作为支付入口 |
| 预览页初始化 payment_actions | [Portal\Invoices@preview#L144-L150](app/Http/Controllers/Portal/Invoices.php#L144-L150) | 同上 |
| PaymentController 动态生成 URL | [PaymentController.php#L155-L160](app/Abstracts/Http/PaymentController.php#L155-L160) | 根据请求来源（portal 或 signed）生成 `confirm`、`return`、`cancel`、`finish`、`notify` 5 种回调 URL |

**signed 路由注册**（见 [signed.php#L12-L21](routes/signed.php#L12-L21)）：
```php
// 核心路由
Route::get( 'invoices/{invoice}',          'Portal\Invoices@signed' )->name('signed.invoices.show');
Route::post('invoices/{invoice}/payment',  'Portal\Invoices@payment')->name('signed.invoices.payment');
Route::post('invoices/{invoice}/confirm',  'Portal\Invoices@confirm')->name('signed.invoices.confirm');
Route::get( 'invoices/{invoice}/finish',   'Portal\Invoices@finish' )->name('signed.invoices.finish');

// 支付模块通过 {alias} 动态占位注册，例如 offline-payments 模块：
// signed.offline-payments.invoices.show    → 支付表单展示
// signed.offline-payments.invoices.confirm → 确认支付回调
```

**生成确认地址的核心逻辑** ([PaymentController.php#L135-L160](app/Abstracts/Http/PaymentController.php#L135-L160))：
```php
public function getConfirmUrl($invoice)
{
    return $this->getModuleUrl($invoice, 'confirm');
}

public function getModuleUrl($invoice, $suffix)
{
    return request()->isPortal($invoice->company_id)
        ? route('portal.' . $this->alias . '.invoices.' . $suffix, $invoice->id)
        : URL::signedRoute('signed.' . $this->alias . '.invoices.' . $suffix, [$invoice->id]);
}
```

> **关键**：`$this->alias` 是支付模块的标识（如 `offline-payments`、`stripe`），由各支付模块的 `PaymentController` 子类继承时赋值。

---

#### 3.5.2 阶段二：请求流转到支付模块

完整的请求流转路径如下：

```
客户收到邮件 → 点击 {invoice_guest_link}
    ↓
signed.invoices.show ([Portal\Invoices@signed](app/Http/Controllers/Portal/Invoices.php#L165))
    │
    ├─ 1. Modules::getPaymentMethods()     ← 触发 PaymentMethodShowing 事件收集所有支付模块
    │                                       见 [Modules.php#L32-L65](app/Utilities/Modules.php#L32-L65)
    │
    ├─ 2. 为每个支付方法生成签名入口 URL
    │     $payment_actions[$codes[0]] = URL::signedRoute('signed.{alias}.invoices.show', ...)
    │     见 [Invoices.php#L175-L180](app/Http/Controllers/Portal/Invoices.php#L175-L180)
    │
    ├─ 3. 渲染 [signed.blade.php](resources/views/portal/invoices/signed.blade.php)
    │     └─ 传给前端 JS：var payment_action_path = payment_actions
    │     └─ 前端 Alpine.js：onChangePaymentMethodSigned('{alias}')
    │            └─ AJAX GET payment_action_path['{alias}']
    │
    └─ 4. 触发 DocumentViewed（仅访客或真实客户）
           见 [Invoices.php#L187-L189](app/Http/Controllers/Portal/Invoices.php#L187-L189)
           ↓
AJAX 请求到达支付模块的 show() 方法
    ↓
{Module}\Http\Controllers\Portal\{Alias}PaymentController@show
    ↓   （继承自 [PaymentController](app/Abstracts/Http/PaymentController.php)）
    │
    ├─ $confirm_url = $this->getConfirmUrl($invoice)  ← 生成 signed confirm URL
    ├─ 渲染支付方法表单视图（hosted：信用卡输入框 / redirect：跳转按钮）
    │   见 [PaymentController.php#L45-L66](app/Abstracts/Http/PaymentController.php#L45-L66)
    └─ 返回 JSON：{ html: "<form action=$confirm_url method=POST>" }
           ↓
客户填写表单或点击按钮 → 提交到 signed confirm URL
```

**支付方法发现机制**：`Modules::getPaymentMethods()` 触发 `PaymentMethodShowing` 事件，各支付模块通过订阅该事件把自己的 `name`、`code`、`customer`、`order` 等信息挂到 `$modules->payment_methods[]` 上，实现支付插件式扩展。

---

#### 3.5.3 阶段三：支付成功 → 触发 PaymentReceived 事件 → 更新状态

**最终触发点**：各支付模块的 `confirm()` 方法或第三方平台异步 `notify()` 回调中，调用 `$this->finish($invoice, $request)`。

**完整代码链路**：

```
{Alias}PaymentController::confirm() / notify()
    ↓
调用父类方法：
    $this->finish($invoice, $request)
    见 [PaymentController.php#L95-L119](app/Abstracts/Http/PaymentController.php#L95-L119)
    │
    ├─ Step 1: $this->dispatchPaidEvent($invoice, $request)
    │   见 [L170-L180](app/Abstracts/Http/PaymentController.php#L170-L180)
    │   │
    │   │   构造 $request：
    │   │   ├─ company_id      = $invoice->company_id
    │   │   ├─ account_id      = setting('{alias}.account_id')
    │   │   ├─ amount          = $invoice->amount       ← 注意：默认传全额
    │   │   ├─ payment_method  = $this->alias
    │   │   ├─ reference       = $this->getReference()  ← 从 session 取第三方交易号
    │   │   └─ type            = 'income'
    │   │
    │   └─ event(new PaymentReceived($invoice, $request))
    │          │
    │          │  PaymentReceived 构造函数补充字段：
    │          │  见 [PaymentReceived.php#L21-L30](app/Events/Document/PaymentReceived.php#L21-L30)
    │          └─ if (empty($request['number'])) {
    │                $request['number'] = $this->getNextTransactionNumber();
    │             }
    │
    │   ┌─ Event.php 监听配置 ─────────────────────────────────────────┐
    │   │  PaymentReceived::class => [                                  │
    │   │    1. CreateDocumentTransaction          ← 改状态 + 记交易    │
    │   │    2. SendDocumentPaymentNotification    ← 发通知             │
    │   │  ]                                                             │
    │   └───────────────────────────────────────────────────────────────┘
    │          │
    │          ├─→ [1] CreateDocumentTransaction Listener
    │          │     见 [CreateDocumentTransaction.php#L20-L50](app/Listeners/Document/CreateDocumentTransaction.php#L20-L50)
    │          │     │
    │          │     └─ dispatch(new CreateBankingDocumentTransaction($doc, $request))
    │          │            见 [CreateBankingDocumentTransaction.php](app/Jobs/Banking/CreateBankingDocumentTransaction.php)
    │          │            │
    │          │            ├─ Step 1: prepareRequest() 补齐 request 字段
    │          │            │
    │          │            ├─ Step 2: checkAmount()  ← **状态计算核心**
    │          │            │   见 [L74-L123](app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123)
    │          │            │   │
    │          │            │   ├─ $model->paid_amount = $model->paid;  // 累计已付
    │          │            │   ├─ event(new PaidAmountCalculated($model))
    │          │            │   ├─ $total = $amount - $paid_amount;     // 未付金额
    │          │            │   └─ bccomp($本次支付, $未付, 精度)
    │          │            │       ├─ 1 (超额)  → throw Exception 阻止
    │          │            │       ├─ 0 (全额)  → $model->status = 'paid'
    │          │            │       └─ -1(部分)  → $model->status = 'partial'
    │          │            │
    │          │            ├─ Step 3: dispatch(new CreateTransaction($request))
    │          │            │   └─ 写入 banking_transactions 表，关联 invoice_id
    │          │            │
    │          │            ├─ Step 4: $model->save()
    │          │            │   └─ documents.status 字段持久化 partial/paid
    │          │            │
    │          │            ├─ Step 5: dispatch(new CreateDocumentHistory(...))
    │          │            │   └─ 写入 document_histories：
    │          │            │      status='partial'/ 'paid'
    │          │            │      notify=0（付款不走通知）
    │          │            │      description=记录付款金额
    │          │            │
    │          │            └─ Step 6: event(new DocumentTransactionCreated(...))
    │          │
    │          └─→ [2] SendDocumentPaymentNotification Listener
    │               见 [SendDocumentPaymentNotification.php#L16-L42](app/Listeners/Document/SendDocumentPaymentNotification.php#L16-L42)
    │               │
    │               ├─ 守卫：$request['type'] !== 'income' → return
    │               │        （Bill 付款不给客户发通知）
    │               ├─ Notify 客户：$document->contact->notify(
    │               │     new PaymentReceived($doc, $txn, 'invoice_payment_customer'))
    │               │
    │               └─ Notify 公司管理员：foreach($company->users)
    │                      有权限的用户 → notify(
    │                         new PaymentReceived($doc, $txn, 'invoice_payment_admin'))
    │
    ├─ Step 2: $this->forgetReference($invoice)  ← 清 session
    │
    └─ Step 3: 生成 finish_url 并 redirect
              见 [L128-L133](app/Abstracts/Http/PaymentController.php#L128-L133)
                   渲染 [finish.blade.php](resources/views/portal/invoices/finish.blade.php)
                   显示付款成功提示
```

#### 3.5.4 影响面汇总（付款）

| 维度 | 影响 |
|------|------|
| **状态流转** | `sent/viewed` → `partial` → `paid`（按本次支付金额与未付金额比例决定）；防止超额支付 |
| **付款记录** | 写入 `banking_transactions`（银行交易），关联 `document_id`，记录 `payment_method` 模块代码和 `reference` 第三方交易号 |
| **历史记录** | 写入 `document_histories`，`status` 字段为当前付款状态，`description` 含付款金额 |
| **通知** | 客户：收到 `invoice_payment_customer` 邮件；管理员：收到 `invoice_payment_admin` 通知；Bill 类型付款无通知（type!==income 守卫） |
| **Session** | 清除 `{alias}_{invoice_id}_reference` 第三方交易号 |
| **异常处理** | `CreateDocumentTransaction` 中 try/catch：超额支付/状态冲突时 flash 错误并 redirect 回 show 页面，不抛异常给客户 |

---

#### 3.5.5 手动记录付款入口（后台操作）

**入口**：管理员在 Invoice/Bill 详情页点击"添加付款"按钮。

路径：`Sales\Transactions@store` → 直接调用 `event(new PaymentReceived($document, $request))`。

与在线支付相同：复用同一套 Listener/Job 链路（`CreateDocumentTransaction` → `CreateBankingDocumentTransaction`），状态计算逻辑完全一致。

---

#### 3.5 付款：→ 部分付款 → 已付款（→ Partial → Paid）

> 详细完整链路见上一节 **3.5.x 付款完整链路**。以下为简化图：

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
                  │   （门户show/signed，双守卫：仅sent可转，且是真实客户）
                  │                       ↓
                  └───────────────────────┼─────────────────┐
                                          ↓                 │
                            部分付款 → PaymentReceived → partial
                                          ↓                 │
                            全部付款 → PaymentReceived → paid ◄┘
                                          ↓
                            手动作废 → DocumentCancelled → cancelled
                        （需通过对账检查；先删关联交易再改状态）


Bill 路径:
   新建 → draft ──┐
                  ├─ markReceived() → DocumentReceived → received
                  ├─ 批量received() → DocumentReceived → received
                  ├─ 循环生成 → auto_send(DocumentReceived) → received
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
- `DocumentSending` / `DocumentSent` / `DocumentMarkedSent`
- `DocumentReceived`
- `DocumentViewed`
- `DocumentCancelled`
- `PaymentReceived` / `PaidAmountCalculated`
- `DocumentTransactionCreating` / `DocumentTransactionCreated`

### 5.3 状态守卫（Status Guard）

每个状态变更前都有前置检查：
- 作废前检查是否已对账
- 联系人变更时检查状态是否已锁定（`sent`, `received`, `viewed`, `partial`, `paid`, `overdue`, `unpaid`, `cancelled`）
- 查看状态仅能从 `sent` 转入 `viewed`
- 查看通知有终态守卫（11 种状态均不再重复发通知）
- 签名链接查看有身份守卫（非真实客户浏览不计入）

---

## 六、核心代码文件索引

| 类型 | 文件路径 |
|------|---------|
| 模型 | [Document.php](app/Models/Document/Document.php) |
| 模型 | [DocumentHistory.php](app/Models/Document/DocumentHistory.php) |
| 特性 | [Documents.php](app/Traits/Documents.php) |
| 配置 | [type.php](config/type.php)（auto_send、notification、status_workflow） |
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
| 监听器 | [SendDocumentViewNotification.php](app/Listeners/Document/SendDocumentViewNotification.php) |
| 监听器 | [SendDocumentRecurringNotification.php](app/Listeners/Document/SendDocumentRecurringNotification.php) |
| 控制器 | [Invoices.php](app/Http/Controllers/Sales/Invoices.php) |
| 控制器 | [Bills.php](app/Http/Controllers/Purchases/Bills.php) |
| 控制器 | [Portal\Invoices.php](app/Http/Controllers/Portal/Invoices.php)（查看入口） |
| 批量操作 | [BulkActions\Sales\Invoices.php](app/BulkActions/Sales/Invoices.php) |
| 批量操作 | [BulkActions\Purchases\Bills.php](app/BulkActions/Purchases/Bills.php) |
