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

### 3.5 付款完整链路：地址生成 → 前端请求 → 模块确认 → 状态更新

付款流程分为 **4 大阶段**：
**①两类确认地址的注册与生成** → **②前端 Vue + axios 请求支付模块** → **③支付确认（同步 + 异步双路径）** → **④触发 PaymentReceived 事件并更新状态**。

---

#### 3.5.1 阶段一：两类确认地址的注册与生成

系统中有 **两套完全不同层级** 的 "confirm" 路由，切勿混淆：

| 维度 | 核心 confirm 路由 | 支付模块 confirm 路由 |
|------|------------------|---------------------|
| **路由名** | `signed.invoices.confirm` | `signed.{alias}.invoices.confirm` |
| **注册位置** | [routes/signed.php#L16](routes/signed.php#L16) | 各支付模块通过 `Route::signed()` 宏注册 |
| **控制器** | `Portal\Invoices@confirm`（⚠️ 方法不存在，预留） | 各模块自己的 `{Alias}PaymentController@confirm` |
| **用途** | 为模块扩展预留的通用入口 | **实际支付确认真正使用的地址** |
| **是否命中** | 核心流程中不会走到 | 客户提交支付表单时真正命中 |

> **核心路由的 confirm/payment 方法缺失说明**：在 `routes/signed.php` 中注册了 `signed.invoices.payment` 和 `signed.invoices.confirm`，但 [Portal\Invoices.php](app/Http/Controllers/Portal/Invoices.php) 控制器中并没有 `payment()` 和 `confirm()` 方法。这两个路由是**为未来通用支付流程预留的占位路由**，实际支付确认均走各支付模块自己的路由。

##### 支付模块路由如何注册（Route Macro 机制）

在 [Route.php#L97-L103](app/Providers/Route.php#L97-L103) 中定义了 `Route::signed()` 宏：
```php
Facade::macro('signed', function ($alias, $routes, $attributes = []) {
    return Facade::module($alias, $routes, array_merge([
        'middleware'    => 'signed',
        'prefix'        => 'signed/' . $alias,      // URL: /{company_id}/signed/{alias}/...
        'as'            => 'signed.' . $alias . '.', // 路由名: signed.{alias}....
    ], $attributes));
});
```

各支付模块在自己的 `Routes/signed.php` 中调用：
```php
Route::signed('offline-payments', function () {
    Route::get('invoices/{invoice}', 'Invoices@show')->name('invoices.show');
    Route::post('invoices/{invoice}/confirm', 'Invoices@confirm')->name('invoices.confirm');
    Route::post('invoices/{invoice}/return', 'Invoices@return')->name('invoices.return');
    Route::post('invoices/{invoice}/cancel', 'Invoices@cancel')->name('invoices.cancel');
    Route::post('invoices/{invoice}/notify', 'Invoices@notify')->name('invoices.notify');
});
```

最终生成：
- URL：`/{company_id}/signed/offline-payments/invoices/123/confirm`
- 路由名：`signed.offline-payments.invoices.confirm`

##### signed URL 生成位置汇总

| 场景 | 代码位置 | 生成的 URL / 路由名 |
|------|---------|-------------------|
| 邮件中的发票查看链接 | [Invoice.php#L155](app/Notifications/Sale/Invoice.php#L155) | `signed.invoices.show` |
| 付款完成通知中的链接 | [PaymentReceived.php#L138](app/Notifications/Portal/PaymentReceived.php#L138) | `signed.invoices.show` |
| 签名页初始化支付方法入口 | [Portal\Invoices@signed#L178-L179](app/Http/Controllers/Portal/Invoices.php#L178-L179) | `signed.{alias}.invoices.show`（每个支付方法一个） |
| 预览页初始化支付方法入口 | [Portal\Invoices@preview#L148](app/Http/Controllers/Portal/Invoices.php#L148) | 同上 |
| 支付模块动态生成回调 URL | [PaymentController.php#L155-L160](app/Abstracts/Http/PaymentController.php#L155-L160) | `signed.{alias}.invoices.{suffix}`（confirm/return/cancel/finish 四种走 signed） |
| 支付模块异步通知地址 | [PaymentController.php#L150-L153](app/Abstracts/Http/PaymentController.php#L150-L153) | `portal.{alias}.invoices.notify`（**不走 signed，详见下文**） |

##### confirm/return/cancel/finish 走 signed URL，notify 为什么单独生成？

`getNotifyUrl()` 的实现（[PaymentController.php#L150-L153](app/Abstracts/Http/PaymentController.php#L150-L153)）：
```php
public function getNotifyUrl($invoice)
{
    return route('portal.' . $this->alias . '.invoices.notify', $invoice->id);
}
```

**notify 不走 signed URL 的 4 个原因**：

| 原因 | 说明 |
|------|------|
| **调用方不同** | notify 是**第三方支付平台服务器**主动回调（webhook/IPN），不是用户浏览器访问；signed URL 是给人用的 |
| **认证方式不同** | 第三方支付平台用**自己的签名机制**（如 HMAC、RSA 等）验证回调合法性，不识别 Laravel 的 `signature` 参数 |
| **请求方法不同** | signed URL 通过 GET 请求验证 signature；notify 是 **POST 请求**，支付平台把交易数据放在 POST body 里 |
| **会话无关** | notify 是服务器到服务器的调用，不需要 session、不需要用户态；signed URL 中间件可能带有 session 依赖 |

---

#### 3.5.2 阶段二：前端 Vue + axios 如何请求支付模块

前端代码位于 [apps.js](resources/assets/js/views/portal/apps.js)，是一个 Vue 实例 + axios 的组合。

有两个相似方法，分别对应 **portal 登录环境** 和 **signed 访客环境**：

| 方法 | 适用场景 | 路径拼接方式 |
|------|---------|-------------|
| `onChangePaymentMethod` | portal 登录客户 | `url + '/portal/' + alias + '/invoices/' + doc_id` |
| `onChangePaymentMethodSigned` | signed 访客 | 从 `payment_action_path[alias]` 读取 PHP 预先生成的 signed URL |

##### signed 环境完整请求流程

```
PHP 端 (Portal\Invoices@signed)
    │
    ├─ 循环 Modules::getPaymentMethods()
    ├─ 生成 $payment_actions[alias] = URL::signedRoute('signed.{alias}.invoices.show', ...)
    └─ 渲染 Blade 模板，注入 JS 变量：
          var payment_action_path = {!! json_encode($payment_actions) !!};
          见 [signed.blade.php#L155-L156](resources/views/portal/invoices/signed.blade.php#L155-L156)
    ↓
前端 Vue 实例 (apps.js)
    │
    ├─ 用户点击支付方法 Tab → 触发 @click="onChangePaymentMethodSigned('offline-payments')"
    │   见 [signed.blade.php#L43](resources/views/portal/invoices/signed.blade.php#L43)
    │
    ├─ onChangePaymentMethodSigned(payment_method)
    │   见 [apps.js#L211-L286](resources/assets/js/views/portal/apps.js#L211-L286)
    │   │
    │   ├─ 1. payment_method.split('.') → 取出 alias
    │   ├─ 2. 从 payment_action_path[alias] 取完整 signed URL
    │   ├─ 3. 显示 loading 状态（spinner 动画）
    │   ├─ 4. axios.get(payment_action, { params: { payment_method } })
    │   │      ↓ HTTP GET 到支付模块 show 路由
    │   │
    │   └─ 5. .then(response)：
    │        ├─ 若 response.data.redirect → location = redirect (跳转到第三方支付页)
    │        ├─ 若 response.data.html → 动态创建 Vue 组件：
    │        │     template = '<div>' + response.data.html + '</div>'
    │        │     （将支付模块返回的表单 HTML 注入到 Vue 组件中）
    │        └─ 组件中表单提交 → 走支付模块自己的 confirm 路由
    │
    └─ onRedirectConfirm() → axios.post(redirectForm.action, redirectForm.data())
          见 [apps.js#L193-L209](resources/assets/js/views/portal/apps.js#L193-L209)
          （hosted 模式下的二次确认提交）
```

**支付模块 show() 返回内容**：各支付模块的 PaymentController 继承 [PaymentController::show()](app/Abstracts/Http/PaymentController.php#L45-L66)，返回 JSON：
```json
{
  "code": "offline-payments",
  "name": "Offline Payments",
  "description": "...",
  "redirect": false,
  "html": "<form action='{signed_confirm_url}' method='POST'>...表单字段...</form>"
}
```

---

#### 3.5.3 阶段三：支付确认（同步 + 异步双路径）

支付确认有**两条独立路径**，最终都调用 `$this->finish()` 触发付款事件：

| 路径 | 触发方式 | 路由 | 适用场景 |
|------|---------|------|---------|
| **同步 confirm** | 客户浏览器提交表单/跳转返回 | `signed.{alias}.invoices.confirm` | redirect 型支付（跳转到第三方后返回）、hosted 型支付（页面内表单提交） |
| **异步 notify** | 第三方服务器主动 POST 回调 | `portal.{alias}.invoices.notify` | webhook / IPN 异步通知，确保订单状态可靠更新 |

##### 同步 confirm 路径
```
客户填写支付表单 / 第三方支付页跳转返回
    ↓
POST signed.{alias}.invoices.confirm
    ↓
{Alias}PaymentController::confirm()
    ├─ 验证支付结果（从 request 中取参数）
    ├─ 可选：$this->setReference($invoice, $reference) 存第三方交易号到 session
    └─ return $this->finish($invoice, $request)
          ↓ 进入付款事件链路（见阶段四）
```

##### 异步 notify 路径
```
第三方支付平台服务器
    ↓ POST (带第三方签名)
portal.{alias}.invoices.notify  （注意：不走 signed URL 中间件）
    ↓
{Alias}PaymentController::notify()
    ├─ 验证第三方签名（HMAC / RSA 等，各支付模块自己实现）
    ├─ 解析支付结果数据
    └─ return $this->finish($invoice, $request)
          ↓ 进入付款事件链路（见阶段四）
```

> **为什么需要两条路径？** 同步路径给客户即时反馈（跳转到成功页），但用户可能中途关闭浏览器；异步 notify 由第三方服务器可靠送达，确保状态最终一致，这是支付系统的标准设计模式。

---

#### 3.5.4 阶段四：触发 PaymentReceived 事件 → 更新状态

`finish()` 方法位于 [PaymentController.php#L95-L119](app/Abstracts/Http/PaymentController.php#L95-L119)，是支付模块与单据系统的唯一交汇点。

**完整链路**：

```
$this->finish($invoice, $request)
    │
    ├─ Step 1: dispatchPaidEvent()
    │   见 [L170-L180](app/Abstracts/Http/PaymentController.php#L170-L180)
    │   │
    │   │  构造 $request（补齐字段）：
    │   │  ├─ company_id      = $invoice->company_id
    │   │  ├─ account_id      = setting('{alias}.account_id', default.account)
    │   │  ├─ amount          = $invoice->amount       ← 默认传发票全额
    │   │  ├─ payment_method  = $this->alias
    │   │  ├─ reference       = $this->getReference()  ← 从 session 取
    │   │  └─ type            = 'income'
    │   │
    │   └─ event(new PaymentReceived($invoice, $request))
    │          │
    │          │  [PaymentReceived 构造函数](app/Events/Document/PaymentReceived.php#L21-L30)
    │          └─ 若 empty($request['number'])
    │               $request['number'] = $this->getNextTransactionNumber();
    │
    │   ┌─ Event.php 监听配置 ─────────────────────────────────────────┐
    │   │  PaymentReceived::class => [                                  │
    │   │    1. CreateDocumentTransaction          ← 改状态 + 记交易    │
    │   │    2. SendDocumentPaymentNotification    ← 发通知             │
    │   │  ]                                                             │
    │   └───────────────────────────────────────────────────────────────┘
    │          │
    │          ├─→ [Listener 1] CreateDocumentTransaction
    │          │     见 [CreateDocumentTransaction.php#L20-L50](app/Listeners/Document/CreateDocumentTransaction.php#L20-L50)
    │          │     │
    │          │     └─ dispatch(new CreateBankingDocumentTransaction($doc, $request))
    │          │            见 [CreateBankingDocumentTransaction.php](app/Jobs/Banking/CreateBankingDocumentTransaction.php)
    │          │            │
    │          │            ├─ 1. prepareRequest() 补齐请求字段
    │          │            │
    │          │            ├─ 2. checkAmount()  ← 状态计算核心
    │          │            │   见 [L74-L123](app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123)
    │          │            │   │
    │          │            │   ├─ $model->paid_amount = $model->paid
    │          │            │   ├─ event(new PaidAmountCalculated($model))
    │          │            │   ├─ $未付 = $总额 - $已付
    │          │            │   └─ bccomp(本次支付, 未付, 精度):
    │          │            │        1 → 超额 → Exception
    │          │            │        0 → 全额 → status = 'paid'
    │          │            │       -1 → 部分 → status = 'partial'
    │          │            │
    │          │            ├─ 3. CreateTransaction Job → 写 banking_transactions
    │          │            │
    │          │            ├─ 4. $model->save() → documents.status 持久化
    │          │            │
    │          │            ├─ 5. CreateDocumentHistory Job → 写历史
    │          │            │
    │          │            └─ 6. event(new DocumentTransactionCreated())
    │          │
    │          └─→ [Listener 2] SendDocumentPaymentNotification
    │               见 [SendDocumentPaymentNotification.php#L16-L42](app/Listeners/Document/SendDocumentPaymentNotification.php#L16-L42)
    │               │
    │               ├─ 守卫：type !== 'income' → return（Bill 不给客户发通知）
    │               ├─ 客户：notify('invoice_payment_customer' 模板)
    │               └─ 公司管理员：有权限用户 → notify('invoice_payment_admin')
    │
    ├─ Step 2: forgetReference($invoice)  ← 清 session 中的第三方交易号
    │
    └─ Step 3: 生成 finish_url 并 redirect
              渲染 [finish.blade.php](resources/views/portal/invoices/finish.blade.php)
              显示付款成功提示
```

##### 状态更新核心逻辑 ([CreateBankingDocumentTransaction.php#L98-L116](app/Jobs/Banking/CreateBankingDocumentTransaction.php#L98-L116))

```php
$paid_amount = money((float) $this->model->paid, $this->model->currency_code, true)->getValue();

// 触发事件允许其他模块干预已付金额计算
event(new PaidAmountCalculated($this->model));

$total_amount = money((float) $this->model->amount - (float) $this->model->paid_amount, ...)
    ->getValue();

$compare = bccomp($this->model->amount, $this->model->paid_amount, $precision);

if ($compare === 1) {
    throw new \Exception(trans('messages.error.over_payment', ['type' => trans_choice('general.invoices', 1)]));
} else {
    $this->model->status = ($compare === 0) ? 'paid' : 'partial';
}
```

> **注意**：`bccomp(A, B)` 返回 `A > B → 1`、`A == B → 0`、`A < B → -1`。这里比较的是**发票总额 vs 累计已付**，即支付后累计已付达到总额就是 paid，否则 partial。

---

#### 3.5.5 影响面汇总

| 维度 | 影响 |
|------|------|
| **状态流转** | `sent / viewed` → `partial` → `paid`，按累计已付与总额比较决定；超额支付直接抛异常阻止 |
| **付款记录** | 写入 `banking_transactions`，关联 `document_id`，记录 `payment_method` 模块代码和 `reference` 第三方交易号 |
| **历史记录** | 写入 `document_histories`，`status` 为 `partial` / `paid`，`description` 含付款金额 |
| **通知** | 客户收到 `invoice_payment_customer` 邮件；管理员收到 `invoice_payment_admin` 通知；Bill 类型付款无客户通知（`type !== 'income'` 守卫） |
| **Session** | 清除 `{alias}_{invoice_id}_reference` 第三方交易号 |
| **异常处理** | `CreateDocumentTransaction` 中 try/catch：超额支付/状态冲突时 flash 错误并 redirect 回 show 页面，不抛异常给客户 |

---

#### 3.5.6 手动记录付款入口（后台操作）

**入口**：管理员在 Invoice/Bill 详情页点击"添加付款"按钮。

路径：`Sales\Transactions@store` → 直接调用 `event(new PaymentReceived($document, $request))`。

与在线支付**完全复用**同一套 Listener/Job 链路（`CreateDocumentTransaction` → `CreateBankingDocumentTransaction`），状态计算逻辑完全一致。

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
