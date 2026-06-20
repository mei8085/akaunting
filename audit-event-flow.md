# Akaunting 审计跟踪与模型事件配合流程

## 核心设计概览

Akaunting 并未使用 `owen-it/laravel-auditing` 等通用审计包，而是采用一套**三层混合式审计架构**：

| 层级 | 机制 | 适用范围 | 粒度 |
|------|------|----------|------|
| 字段级 | Trait + Job 接口自动注入 | 几乎所有模型 | 谁在何时从哪里创建 |
| 业务状态级 | Event → Listener → Job → History 表 | Document / Module 等关键实体 | 完整状态流转与操作描述 |
| 关联副作用级 | Eloquent Observer (Mutable) | Transaction 等有连锁影响的模型 | 级联更新与补偿记录 |

---

## 第一层：字段级审计 — created_by / created_from / company_id

### 核心 Trait 与接口

#### 1. Owners Trait — 归属判断
位置：[app/Traits/Owners.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Owners.php#L1-L18)

```php
trait Owners
{
    public function isOwnable()
    {
        $ownable = $this->ownable ?: true;
        return ($ownable === true) && in_array('created_by', $this->getFillable());
    }
}
```

- **判断逻辑**：只有当模型显式将 `created_by` 加入 `$fillable` 时，才被认为是「可归属的」
- 可通过在模型中设置 `protected $ownable = false;` 强制关闭

#### 2. Sources Trait — 来源追踪
位置：[app/Traits/Sources.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Sources.php#L1-L72)

```php
public function getSourceName($request = null, $alias = null): string
{
    // 优先级：console > queue > api > ui
    // 格式：{alias}::{source}，如 core::ui, payroll::api
}
```

- 来源取值：`console` / `queue` / `api` / `ui`
- 自动识别当前运行环境（CLI、队列、API 请求、Web 请求）

#### 3. HasOwner & HasSource 接口 — Job 层自动注入
位置：[app/Abstracts/Job.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Job.php#L40-L62)

**注入时机**：在 Job 构造函数的 `bootCreate()` 阶段自动执行：

```php
public function bootCreate(...$arguments): void
{
    if (! $this instanceof ShouldCreate) return;

    // 转换 request 实例...

    if ($this instanceof HasOwner) {
        $this->setOwner();   // 注入 created_by = user_id()
    }

    if ($this instanceof HasSource) {
        $this->setSource();  // 注入 created_from = source_name()
    }
}
```

`setOwner()` 与 `setSource()` 的实现（[Job.php#L111-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Job.php#L111-L135)）：
- 如果 Request 中已经手动设置了对应字段，则**不覆盖**
- 否则调用全局辅助函数 `user_id()` / `getSourceName()` 填充

### 辅助函数
位置：[app/Utilities/helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Utilities/helpers.php)

| 函数 | 作用 |
|------|------|
| `user_id()` | 获取当前认证用户 ID，未登录返回 `null` |
| `company_id()` | 获取当前公司 ID，基于 session |
| `source_name()` | 获取当前来源标识（委托 Sources Trait） |

### 多租户隔离（Company Scope）
位置：[app/Scopes/Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Scopes/Company.php#L21-L50)

通过 `Tenants` Trait（[app/Traits/Tenants.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Tenants.php#L14-L17)）在模型 `bootTenants()` 时自动注册全局 Scope：

```php
protected static function bootTenants()
{
    static::addGlobalScope(new Company);
}
```

Scope 行为：
- 检查 `isNotTenantable()` 以及 `$fillable` 中是否有 `company_id`
- 对查询自动追加 `WHERE company_id = ?` 条件
- **注意**：Scope 只负责查询过滤，`company_id` 的写入需在 Job/Request 层手动或通过中间件设置

---

## 第二层：业务状态级审计 — 专用 History 表

### 典型代表：DocumentHistory
位置：[app/Models/Document/DocumentHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Document/DocumentHistory.php#L1-L57)

**表结构关键字段**：
| 字段 | 含义 |
|------|------|
| `company_id` | 租户 |
| `type` | 单据类型（invoice / bill 等） |
| `document_id` | 关联单据 |
| `status` | 当时的状态码（draft/sent/paid/...） |
| `notify` | 是否发送通知 |
| `description` | 人类可读的操作描述（多语言） |
| `created_from` | 来源 |
| `created_by` | 操作人 |

### 完整触发链路（以「创建发票」为例）

```
HTTP Request
     │
     ▼
[Controller] Invoices::store()
  app/Http/Controllers/Sales/Invoices.php#L100-L125
     │
     │  $this->ajaxDispatch(new CreateDocument($request))
     ▼
[Job] CreateDocument::__construct()
  → bootCreate() 自动注入 created_by / created_from
  （HasOwner + HasSource 接口生效）
     │
     ▼
[Job] CreateDocument::handle()
  app/Jobs/Document/CreateDocument.php#L20-L57
     │
     ├─► event(new DocumentCreating($request))   // 前置事件
     │
     ├─► \DB::transaction(function () {
     │      Document::create($request->all());    // 模型写入
     │      ...
     │      $this->model->update(...);
     │   })
     │
     └─► event(new DocumentCreated($model, $request))  // 后置事件 ★
                              │
                              ▼
              [EventServiceProvider] 事件路由
                app/Providers/Event.php#L54-L58
                DocumentCreated::class => [
                  CreateDocumentCreatedHistory::class,  // ★ 监听器
                  IncreaseNextDocumentNumber::class,
                  SettingFieldCreated::class,
                ]
                              │
                              ▼
              [Listener] CreateDocumentCreatedHistory::handle()
                app/Listeners/Document/CreateDocumentCreatedHistory.php#L19-L24
                              │
                              │  $this->dispatch(new CreateDocumentHistory(...))
                              ▼
              [Job] CreateDocumentHistory::handle()
                app/Jobs/Document/CreateDocumentHistory.php#L31-L47
                              │
                              │  DocumentHistory::create([...])
                              ▼
                       [DB] document_histories 表
```

### 状态变更类事件的监听模式

以「标记为已发送」为例，位置 [app/Listeners/Document/MarkDocumentSent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/MarkDocumentSent.php#L14-L45)：

```php
public function handle(DocumentMarkedSent|DocumentSent $event): void
{
    // 1. 先更新单据自身状态
    if (! in_array($event->document->status, ['partial', 'paid'])) {
        $event->document->status = 'sent';
        $event->document->save();
    }

    // 2. 再写入历史记录
    $this->dispatch(new CreateDocumentHistory(
        $event->document,
        0,
        $this->getDescription($event)  // 生成多语言描述
    ));
}
```

### 已注册的 Document 事件与对应监听器
位置：[app/Providers/Event.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Event.php#L54-L90)

| 事件类 | 监听器 | 效果 |
|--------|--------|------|
| `DocumentCreated` | `CreateDocumentCreatedHistory` | 写入创建历史 |
| `DocumentReceived` | `MarkDocumentReceived` | 更新状态 + 历史 |
| `DocumentCancelled` | `MarkDocumentCancelled` | 更新状态 + 历史 |
| `DocumentRestored` | `RestoreDocument` | 恢复 + 历史 |
| `PaymentReceived` | `CreateDocumentTransaction` + `SendDocumentPaymentNotification` | 创建付款交易 + 通知 |
| `DocumentMarkedSent` / `DocumentSent` | `MarkDocumentSent` | 更新状态 + 历史 |
| `DocumentUpdated` | `SettingFieldUpdated` | 自定义字段更新 |
| `DocumentViewed` | `MarkDocumentViewed` + `SendDocumentViewNotification` | 标记已查看 + 通知 |

### 另一例：ModuleHistory
位置：[app/Listeners/Update/CreateModuleUpdatedHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Update/CreateModuleUpdatedHistory.php#L17-L41)

监听 `UpdateFinished` 事件后，直接通过 `ModuleHistory::create()` 写入：

```php
ModuleHistory::create([
    'company_id'   => $model->company_id,
    'module_id'    => $model->id,
    'version'      => $event->new,
    'description'  => trans('modules.updated_2', ['module' => $module->getAlias()]),
    'created_from' => source_name(),   // 直接调用辅助函数
    'created_by'   => user_id(),       // 直接调用辅助函数
]);
```

---

## 第三层：关联副作用级 — Eloquent Observer

### 注册入口
位置：[app/Providers/Observer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Observer.php#L25-L28)

```php
public function boot()
{
    Transaction::observe('App\Observers\Transaction');
}
```

### Mutable Trait 的作用
位置：[app/Abstracts/Observer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Observer.php#L1-L10)

```php
abstract class Observer
{
    use \Akaunting\MutableObserver\Traits\Mutable;
}
```

`akaunting/laravel-mutable-observer` 包的 `Mutable` Trait 提供的能力（根据类名和使用方式推断）：
- 允许观察者的方法在运行时被修改/装饰（Mutable = 可变）
- 可能用于支持模块动态地向已有观察者追加逻辑
- **注意**：vendor 目录不存在，此包源码未安装

### 典型观察者：Transaction Observer
位置：[app/Observers/Transaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Observers/Transaction.php#L23-L79)

**监听 `deleted` 事件后的完整处理链**：

```
Transaction::delete()
     │
     ▼
[Observer] Transaction::deleted($transaction)
     │
     ├─ 情况A：关联了 Document（invoice/bill）
     │      │
     │      ├─ 重新计算 document->transactions_count
     │      ├─ event(new TransactionsCounted($document))
     │      ├─ 根据剩余付款数更新 document.status（sent→partial）
     │      ├─ $document->save()
     │      │
     │      └─ dispatch(new CreateDocumentHistory(...))  ← 触发第二层审计
     │
     └─ 情况B：是拆分交易（split_id 非空）
            │
            └─ 如果拆分明细已空，dispatch(new UpdateTransaction(...))
               把拆分父交易恢复为普通交易类型
```

**关键观察**：Observer 不直接写审计表，而是 **重新调度第二层的 Job（CreateDocumentHistory）**，保持审计写入路径一致。

---

## 完整数据流总图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         应用层 HTTP / CLI / Queue                   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Controller / Command / 直接调用                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  所有写操作都封装为 Job，不直接操作 Model                     │    │
│  └──────────────────────────────┬──────────────────────────────┘    │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Job 构造函数（Abstracts\Job）                                      │
│    ├─ bootCreate() 阶段：                                           │
│    │    ├─ instanceof HasOwner   → setOwner()  注入 created_by     │
│    │    └─ instanceof HasSource  → setSource() 注入 created_from   │
│    ├─ bootUpdate() / bootDelete() 阶段                              │
│    └─ 注入 company_id 通常在 Request 验证或中间件中完成             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Job::handle() 实际执行                                             │
│    ├─ event(new XxxCreating)        ← 业务前置事件（可选）          │
│    ├─ DB::transaction {                                             │
│    │    Model::create / update / delete                             │
│    │      │                                                         │
│    │      ▼                                                         │
│    │   Eloquent 生命周期事件                                        │
│    │     → creating / created / updating / updated                  │
│    │     → deleting / deleted  ★ 触发 Observer                     │
│    │  }                                                             │
│    └─ event(new XxxCreated)         ← 业务后置事件（写入审计的入口）│
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
┌───────────────────────────┐     ┌──────────────────────────────────┐
│   Eloquent Observer       │     │   Event → Listener 路由           │
│   (Mutable Trait)         │     │   (EventServiceProvider)          │
│                           │     │                                  │
│  监听模型 deleted 等事件  │     │  监听 DocumentCreated 等业务事件  │
│  处理关联数据一致性       │     │                                  │
│  → 可能二次触发业务事件   │     │  → dispatch CreateXxxHistory Job  │
└─────────────┬─────────────┘     └──────────────┬───────────────────┘
              │                                  │
              │    ┌─────────────────────────────┘
              │    │
              ▼    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  审计写入层（最终落盘）                                             │
│                                                                     │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐    │
│  │  字段级审计          │  │  业务状态级审计                   │    │
│  │  （所有模型）        │  │  （Document / Module 等关键实体） │    │
│  │                      │  │                                  │    │
│  │  主表字段：          │  │  专用 history 表：               │    │
│  │    · created_by      │  │    · document_histories          │    │
│  │    · created_from    │  │    · module_histories            │    │
│  │    · company_id      │  │    · (bill_histories 已废弃，    │    │
│  │    · created_at      │  │       合并到 document_histories) │    │
│  │    · updated_at      │  │                                  │    │
│  │    · deleted_at      │  │  包含：状态、描述、通知标志       │    │
│  └──────────────────────┘  └──────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计原则与容易混淆的点

### 1. 三层审计是「互补」不是「重复」
- **字段级**回答「谁创建了这个记录」
- **状态级**回答「这个单据经历了什么操作」（发票从草稿→已发送→已付款，有完整时间线）
- **副作用级**回答「这次删除还影响了谁」

### 2. `created_by` 的注入位置选择
| 方式 | 适用场景 | 代码位置 |
|------|----------|----------|
| Job `bootCreate()` + `HasOwner` 接口 | 规范的 CRUD Job（90% 场景） | [Job.php#L55-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Job.php#L55-L57) |
| 手动调用 `user_id()` | 不在 Job 中的直接写入，如 Listener 里 | [CreateModuleUpdatedHistory.php#L39](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Update/CreateModuleUpdatedHistory.php#L39) |
| 直接在 Request 中传入 | 需要指定非当前用户（如导入数据时指定原始创建者） | [Traits/Import.php#L138](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Import.php#L138) |

### 3. 为什么 Observer 不直接写审计记录？
Observer 监听的是 **Eloquent 底层事件**，此时业务语义可能不完整：
- `Transaction::deleted` 可能是用户手动删除、也可能是账单删除时的级联
- 不同业务场景需要的审计描述不同

所以 Observer **处理数据一致性，调度业务层审计**，最终还是通过 `CreateDocumentHistory` Job 写入。

### 4. 没有通用的「所有模型变更审计」
与典型 `laravel-auditing` 包自动记录所有模型 `created/updated/deleted` 的 `old/new` 值不同，Akaunting 的审计是**手动、选择性**的：
- 不需要的模型不记 History
- 记录什么描述由 Listener 明确指定（多语言、业务友好）
- 字段级审计只记录「创建人」不记录每次修改的 diff

这是一个设计取舍：**牺牲自动化换取业务语义清晰与存储空间节省**。

---

## 补充章节一：UpdateDocument 完整路径分析

### 类声明与接口
位置：[app/Jobs/Document/UpdateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/UpdateDocument.php#L1-L16)

```php
use App\Interfaces\Job\ShouldUpdate;

class UpdateDocument extends Job implements ShouldUpdate
{
    // 仅实现 ShouldUpdate，未实现 HasOwner / HasSource
}
```

**接口决策理由**：更新操作不改变 `created_by` 和 `created_from`（创建人/来源始终是最初创建时的信息），因此不需要这两个接口。

### 完整执行流程

```
HTTP Request (PATCH/PUT)
     │
     ▼
[Controller] Invoices::update() / Bills::update()
     │
     │  $this->ajaxDispatch(new UpdateDocument($document, $request))
     ▼
[Job 构造] UpdateDocument::__construct($document, $request)
  → bootUpdate() 执行：
      1. 识别 arguments[0] 为 Model，赋值给 $this->model
      2. 识别 arguments[1] 并转为 Request 实例，赋值给 $this->request
     │
     │  ★ 因为未实现 HasOwner/HasSource，所以不注入 created_by / created_from
     ▼
[Job] UpdateDocument::handle()
  app/Jobs/Document/UpdateDocument.php#L19-L87
     │
     ├─ authorize()  权限校验：锁定状态下不能改 contact_id
     │
     ├─ event(new DocumentUpdating($this->model, $this->request))   // ★ 前置业务事件
     │
     ├─ 记录 originalContactId（用于后续同步交易的 contact_id）
     │
     ├─ \DB::transaction(function () use ($originalContactId) {
     │     │
     │     ├─ 处理附件上传/移除
     │     ├─ deleteRelationships([items, item_taxes, totals], true)  // 重建行项目
     │     ├─ dispatch(new CreateDocumentItemsAndTotals($model, $request))
     │     │
     │     ├─ ★ 状态自修正逻辑：
     │     │   $this->model->paid_amount = $this->model->paid;
     │     │   event(new PaidAmountCalculated($this->model));
     │     │   if (已付款 == 总金额) → request['status'] = 'paid'
     │     │   if (已付款 > 0 && < 总金额) → request['status'] = 'partial'
     │     │
     │     ├─ $this->model->update($this->request->all())   // 更新主表
     │     │
     │     ├─ contact_id 变化时，同步未勾兑交易的 contact_id
     │     │
     │     └─ $this->model->updateRecurring(...)
     │  })
     │
     └─ event(new DocumentUpdated($this->model, $this->request))    // ★ 后置业务事件
                              │
                              ▼
              [EventServiceProvider] 事件路由
                app/Providers/Event.php#L84-L86
                DocumentUpdated::class => [
                  SettingFieldUpdated::class,   // 唯一监听器
                ]
                              │
                              ▼
              [Listener] SettingFieldUpdated::handle()
                app/Listeners/Document/SettingFieldUpdated.php#L22-L82
                              │
                              │  处理 $request['setting'] 自定义字段：
                              │    · company_logo 上传
                              │    · invoice.footer / bill.notes 等单据级设置
                              │    · setting()->save()
                              │
                              │  ★ 此 Listener 不写 DocumentHistory
                              ▼
                    （没有历史记录被写入）
```

### UpdateDocument 的历史记录策略

**关键结论：UpdateDocument 本身不写 DocumentHistory。**

历史记录只在以下**状态变更事件**中写入（均通过各自 Listener → CreateDocumentHistory）：

| 操作 | 触发事件 | 写历史的监听器 |
|------|----------|----------------|
| 创建 | `DocumentCreated` | `CreateDocumentCreatedHistory` |
| 标记已发送 / 发送邮件 | `DocumentMarkedSent` / `DocumentSent` | `MarkDocumentSent` |
| 标记已收到 | `DocumentReceived` | `MarkDocumentReceived` |
| 标记已取消 | `DocumentCancelled` | `MarkDocumentCancelled` |
| 标记已查看 | `DocumentViewed` | `MarkDocumentViewed` |
| 恢复（取消→草稿） | `DocumentRestored` | `RestoreDocument` (Listener) |
| 收到付款 | `PaymentReceived` | `CreateDocumentTransaction` 中间接触发 |
| 删除一条关联交易 | `deleted` (Observer) | Transaction Observer 内 dispatch |

这样设计的原因：
- 纯数据字段修改（改金额、改日期、改备注）不产生"业务状态"的跃迁，不记历史
- 只有**对外可见的状态变更**（给客户发了、客户看了、钱到了、取消了）才需要审计追踪
- 真正的字段变更追踪依赖 `updated_at` + 数据库层面的 binlog / 快照

---

## 补充章节二：DeleteDocument 完整路径与 Mute 机制

### 类声明与接口
位置：[app/Jobs/Document/DeleteDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DeleteDocument.php#L1-L13)

```php
use App\Interfaces\Job\ShouldDelete;

class DeleteDocument extends Job implements ShouldDelete
{
    // 仅实现 ShouldDelete，未实现 HasOwner / HasSource
    // 额外 use 了 Transaction Observer 类名（用于静态调用 mute/unmute）
}
```

### 完整执行流程 + Mute 机制详解

```
HTTP Request (DELETE)
     │
     ▼
[Controller] Invoices::destroy($invoice)
  app/Http/Controllers/Sales/Invoices.php#L223-L231
     │
     │  $this->ajaxDispatch(new DeleteDocument($invoice))
     ▼
[Job 构造] DeleteDocument::__construct($model)
  → bootDelete() 识别 arguments[0] 为 Model，赋值给 $this->model
     │
     │  ★ 不实现 HasOwner/HasSource，不注入任何字段
     ▼
[Job] DeleteDocument::handle()
  app/Jobs/Document/DeleteDocument.php#L14-L37
     │
     ├─ authorize()  不允许删除已勾兑交易的单据
     │
     ├─ event(new DocumentDeleting($this->model))   // ★ 前置事件（也无监听器）
     │
     ├─ \DB::transaction(function () {
     │     try {
     │         // ★★★ 核心：Mute 整个 Transaction Observer ★★★
     │         Transaction::mute();
     │         // ┌─────────────────────────────────────────────┐
     │         // │ 此后所有 Transaction 模型的                │
     │         // │   creating / created /                    │
     │         // │   updating / updated /                    │
     │         // │   deleting / deleted                      │
     │         // │ 全部被静音，App\Observers\Transaction     │
     │         // │ 的对应方法不会被 Laravel 调度执行         │
     │         // └─────────────────────────────────────────────┘
     │
     │         deleteRelationships($this->model, [
     │             'items',            // 行项目（软删）
     │             'item_taxes',       // 行税（软删）
     │             'histories',        // ★ 单据历史本身也被软删！
     │             'transactions',     // ★ 关联交易（软删）—— 如果不 mute，这里每条交易删除都会触发
     │             'recurring',        //   Transaction Observer 的 deleted()，级联写 DocumentHistory
     │             'totals'            //   但此刻 Document 即将被删，写历史毫无意义
     │         ]);
     │
     │         $this->model->delete();  // 软删单据本身
     │
     │     } finally {
     │         // ★★★ 无论成功与否都要 unmute，避免后续操作受影响 ★★★
     │         Transaction::unmute();
     │     }
     │  })
     │
     └─ event(new DocumentDeleted($this->model))   // ★ 后置事件
                              │
                              ▼
              [EventServiceProvider] 路由查询
                app/Providers/Event.php
                              │
                              │  ★ DocumentDeleted::class 未注册任何监听器！
                              │
                              ▼
                         （空执行，无任何副作用）
```

### 为什么必须 Mute Transaction Observer？

**不 mute 的恶果（反向推演）**：

假设删除一张已部分付款的发票：

```
DeleteDocument::handle()
  deleteRelationships([...'transactions'...])
    → 遍历 transactions，每条 $transaction->delete()
      → Laravel 触发 deleting / deleted Eloquent 事件
        → App\Observers\Transaction::deleted($transaction) 被调用
          │
          ├─ 取关联的 $document（就是那张正在被删的发票！）
          ├─ 重新计算 transactions_count
          ├─ event(new TransactionsCounted($document))
          ├─ 根据 count 重置 $document->status 为 'sent' 或 'partial'
          ├─ $document->save()   ← ★ 此时 document 即将被删，还在写它！
          │
          └─ dispatch(new CreateDocumentHistory($document, ...))
               ↓
             在一个事务里：
               1. 先删了 histories 表（deleteRelationships 做的）
               2. 又想 Insert 一条 DocumentHistory ← 外键或逻辑错乱！
```

**Mute 的本质**：告诉 Observer「这次删除是上层业务明确发起的级联删除，不需要你做补偿逻辑」。

### Mute/Unmute 实现原理（来自 `akaunting/laravel-mutable-observer`）

该包使用**服务容器代理模式**（基于 README 分析）：

1. `Transaction::mute()` 调用后，在 Laravel 容器中注册一个代理对象
2. 当 Eloquent 尝试解析 `App\Observers\Transaction` 并调用 `deleted()` 时
3. 代理对象检查当前事件名是否在「静音列表」中
   - 如果是 `*`（全部静音）或匹配指定事件，则吞掉调用（返回 null）
   - 否则放行到真正的 Observer 实例

**API 完整形式**：
```php
Transaction::mute();                  // 静音所有事件（DeleteDocument 用的就是这个）
Transaction::mute('deleted');         // 只静音 deleted
Transaction::mute(['deleting', 'deleted']);  // 静音多个
Transaction::unmute();                // 恢复所有
```

### DocumentDeleted 无监听器时的历史处理

这是一个容易困惑的点，分步说明：

#### 步骤一：为什么 DocumentDeleted 不注册监听器？

因为在 `deleteRelationships` 中，**`histories` 关联已经被一起软删除了**！

代码证据（[DeleteDocument.php#L24-L26](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DeleteDocument.php#L24-L26)）：
```php
$this->deleteRelationships($this->model, [
    'items', 'item_taxes', 'histories', 'transactions', 'recurring', 'totals'
    //               ^^^^^^^^^—— 历史本身被删除了
]);
```

此时如果在 DocumentDeleted 的监听器里写 `CreateDocumentHistory`，就会遇到：
- 外键约束问题（document 本身被软删了，但 document_id 外键仍然指向它，取决于数据库设置）
- 逻辑荒谬：给一条被整体删除的单据再追加一条"它被删了"的历史，而查询时因为 SoftDeletes Scope 根本看不到 document，也看不到它的 histories

#### 步骤二：那如何追溯"谁删了这张单据"？

Akaunting 的设计答案是：**不追溯**。它只记录创建人 `created_by`，不记录删除人 `deleted_by`。

如果要追溯删除，需要依赖：
1. **Web 服务器 access log** — 对 `/invoices/{id}` 的 DELETE 请求
2. **Laravel 日志** — 如果有异常会写入
3. **数据库 binlog / 物理备份** — 真正的灾难恢复手段
4. **SoftDeletes 本身的 `deleted_at` 字段** — 可以知道什么时候删的，但不知道是谁

#### 步骤三：与 Cancel（取消）操作对比

用户容易混淆"删除"和"取消"，但它们在审计上完全不同：

| 维度 | DeleteDocument（删除） | CancelDocument（取消） |
|------|------------------------|------------------------|
| 触发方式 | DELETE /invoices/123 | 点击「取消」按钮，触发 `event(DocumentCancelled)` |
| 数据结果 | Document + Histories + Transactions 全部软删 | Document.status = 'cancelled'，历史保留 |
| 能否恢复 | 需要 `withTrashed()->restore()`，且 histories 也要单独 restore | 一键 `DocumentRestored` 事件 → 变回 draft |
| 审计记录 | 不写入 DocumentHistory | **`MarkDocumentCancelled` Listener 写入历史**，描述为"XX 已取消" |
| 监听器 | DocumentDeleted 无监听器 | DocumentCancelled → `MarkDocumentCancelled` → CreateDocumentHistory |

CancelDocument 的执行路径（[CancelDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CancelDocument.php#L20-L34) + [MarkDocumentCancelled.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/MarkDocumentCancelled.php#L20-L41)）：

```
Controller::markCancelled()
  → event(new DocumentCancelled($document))
      → MarkDocumentCancelled::handle()
          ├─ dispatch(new CancelDocument($document))
          │     → deleteRelationships([transactions, recurring]) 只删交易不删历史
          │     → $document->status = 'cancelled'; $document->save()
          │
          └─ dispatch(new CreateDocumentHistory($document, 0, '发票 INV-001 已取消'))
                                              ↑
                                  正常写入 document_histories 表
```

---

## 补充章节三：HasOwner / HasSource 接口在 Document 和 Module 体系中的应用对照表

### 接口生效原理回顾

只有当 Job 同时满足以下条件时，`created_by` / `created_from` 才会在构造函数中**自动注入**：

1. Job 实现了 `ShouldCreate` 接口（因为只在 `bootCreate()` 阶段注入）
2. Job 同时实现了 `HasOwner`（对应 created_by）或 `HasSource`（对应 created_from）接口
3. Request 中尚未设置对应字段（`setOwner`/`setSource` 会先检查 `$request->has(...)`）

代码参考：[Abstracts/Job.php#L40-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Job.php#L40-L68)

### Document 相关 Job 全景表

| Job 类名 | 文件 | ShouldCreate | ShouldUpdate | ShouldDelete | HasOwner | HasSource | created_by 注入方式 | 备注 |
|---------|------|:------------:|:------------:|:------------:|:--------:|:---------:|---------------------|------|
| **CreateDocument** | [CreateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CreateDocument.php#L16) | ✅ | ❌ | ❌ | ✅ | ✅ | bootCreate 自动注入 | 核心创建 Job |
| **CreateDocumentHistory** | [CreateDocumentHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CreateDocumentHistory.php#L13) | ✅ | ❌ | ❌ | ✅ | ✅ | bootCreate 自动注入，但 handle() 里又手动赋值 | **双保险**：bootCreate 注入 request，handle() 再 `user_id()` 兜底 |
| **CreateDocumentItem** | [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CreateDocumentItem.php#L15) | ✅ | ❌ | ❌ | ✅ | ✅ | bootCreate 自动注入 | 创建行项目 |
| **CreateDocumentItemsAndTotals** | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L16) | ✅ | ❌ | ❌ | ✅ | ✅ | bootCreate 自动注入 | 批量创建行项目+合计 |
| **UpdateDocument** | [UpdateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/UpdateDocument.php#L15) | ❌ | ✅ | ❌ | ❌ | ❌ | （不注入） | 保留原有 created_by / created_from |
| **DeleteDocument** | [DeleteDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DeleteDocument.php#L12) | ❌ | ❌ | ✅ | ❌ | ❌ | （不注入） | 软删除，不改写字段 |
| **CancelDocument** | [CancelDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CancelDocument.php#L9) | ❌ | ❌ | ❌ | ❌ | ❌ | （不注入） | 只改 status，不写新记录 |
| **RestoreDocument** | [RestoreDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/RestoreDocument.php#L8) | ❌ | ❌ | ❌ | ❌ | ❌ | （不注入） | 只改 status |
| **DuplicateDocument** | [DuplicateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DuplicateDocument.php#L10) | ❌ | ❌ | ❌ | ❌ | ❌ | **间接注入** | 自身不实现接口，但 `duplicate()` 返回新 Model 后 `event(new DocumentCreated)` 触发的 CreateDocumentCreatedHistory → CreateDocumentHistory 链会用当前请求的 user |
| **SendDocument** | [SendDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/SendDocument.php#L10) | ❌ | ❌ | ❌ | ❌ | ❌ | （不注入） | 触发 DocumentSent 事件，由 Listener 走 CreateDocumentHistory 链 |
| **SendDocumentAsCustomMail** | [SendDocumentAsCustomMail.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/SendDocumentAsCustomMail.php#L11) | ❌ | ❌ | ❌ | ❌ | ❌ | （不注入） | 纯邮件发送 |
| **DownloadDocument** | [DownloadDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DownloadDocument.php#L9) | ❌ | ❌ | ❌ | ❌ | ❌ | （不注入） | 纯 PDF 生成 |

### Module 体系对比

Module（应用市场模块）的历史记录走的是**完全不同的路径**——没有 Job，直接在 Listener 中 Model::create()：

| 场景 | 入口 | 历史写入方式 | created_by / created_from 获取方式 | 代码位置 |
|------|------|-------------|-----------------------------------|----------|
| **模块升级后** | `UpdateFinished` 事件 | `ModuleHistory::create()` 直接写入 | 手动调用 `user_id()` 和 `source_name()` 辅助函数 | [CreateModuleUpdatedHistory.php#L33-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Update/CreateModuleUpdatedHistory.php#L33-L40) |
| 模块安装后 | `Installed` 事件 | 未写入 ModuleHistory（只跑 migrate + 绑定权限） | （无） | [FinishInstallation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Module/FinishInstallation.php) |
| 模块卸载后 | `Uninstalled` 事件 | 未写入 ModuleHistory | （无） | [FinishUninstallation.php] |

关键差异代码对比：

**Document 体系（规范路径，自动注入）**：
```php
// Job 声明
class CreateDocumentHistory extends Job implements HasOwner, HasSource, ShouldCreate

// bootCreate 自动把 created_by / created_from merge 进 $this->request

// handle() 里直接用 request 数据
DocumentHistory::create($this->request->all() + [...])
```

**Module 体系（Listener 直接写，手动取值）**：
```php
// Listener handle() 直接写，没有 Job 层
ModuleHistory::create([
    'company_id'   => $model->company_id,
    'module_id'    => $model->id,
    'version'      => $event->new,
    'description'  => trans('modules.updated_2', [...]);
    'created_from' => source_name(),   // 直接调全局函数
    'created_by'   => user_id(),       // 直接调全局函数
]);
```

### 规范场景与例外场景的取舍总结

| **推荐做法（HasOwner/HasSource + ShouldCreate）** | **直接 `user_id()` / `source_name()`** |
|--------------------------------------------------|----------------------------------------|
| ✅ 有明确的创建型 Job 封装 | ❌ 只是 Listener 里顺手写一条历史 |
| ✅ 需要允许调用方手动覆盖 created_by（如导入指定原创建人） | ❌ 不可能有其他调用方，事件参数就是最终信息 |
| ✅ 需要走统一的 bootCreate 生命周期 | ❌ 一次性逻辑，不参与 Job 调度 |
| CreateDocument / CreateTransaction 等主流程 | CreateModuleUpdatedHistory 等边角历史 |

---

## 扩展文件索引（补充）

| 分类 | 文件 | 作用 |
|------|------|------|
| 核心更新 Job | [UpdateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/UpdateDocument.php) | 单据更新 + 状态自修正 |
| 核心删除 Job | [DeleteDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DeleteDocument.php) | 含 mute/unmute + 级联软删 |
| 取消 Job | [CancelDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CancelDocument.php) | 取消：改 status 为 cancelled |
| 恢复 Job | [RestoreDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/RestoreDocument.php) | 恢复：改 status 为 draft |
| 复制 Job | [DuplicateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DuplicateDocument.php) | 克隆后触发 DocumentCreated |
| 发送 Job | [SendDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/SendDocument.php) | 发邮件并触发 DocumentSent |
| 取消监听器 | [MarkDocumentCancelled.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/MarkDocumentCancelled.php) | 取消操作写历史 |
| 恢复监听器 | [RestoreDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/RestoreDocument.php) | 恢复操作写历史 |
| 已收到监听器 | [MarkDocumentReceived.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/MarkDocumentReceived.php) | 收到操作写历史 |
| 更新设置监听器 | [SettingFieldUpdated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/SettingFieldUpdated.php) | DocumentUpdated 唯一监听器，不写历史 |
| 模块升级历史 | [CreateModuleUpdatedHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Update/CreateModuleUpdatedHistory.php) | 直接 user_id()/source_name() |
| 级联关系删除 Trait | [Relationships.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Relationships.php) | deleteRelationships 实现软删循环 |
| 事件定义 | [DocumentDeleted.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Events/Document/DocumentDeleted.php) | 无监听器的后置事件 |
| 事件定义 | [DocumentDeleting.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Events/Document/DocumentDeleting.php) | 无监听器的前置事件 |
| 事件定义 | [DocumentUpdating.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Events/Document/DocumentUpdating.php) | 更新前置事件 |

## 关键文件索引

| 分类 | 文件 | 作用 |
|------|------|------|
| 字段级 Trait | [Owners.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Owners.php) | created_by 判断 |
| 字段级 Trait | [Sources.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Sources.php) | created_from 判断 |
| 字段级 Trait | [Tenants.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Tenants.php) | 注册 Company Scope |
| Job 抽象类 | [Job.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Job.php) | bootCreate 自动注入 |
| 全局 Scope | [Scopes/Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Scopes/Company.php) | 查询级租户隔离 |
| 事件路由 | [Providers/Event.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Event.php) | 事件→监听器映射 |
| 观察者注册 | [Providers/Observer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Observer.php) | Transaction::observe() |
| 观察者基类 | [Abstracts/Observer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Observer.php) | 使用 Mutable Trait |
| 观察者示例 | [Observers/Transaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Observers/Transaction.php) | 删除时的级联处理 |
| 历史记录 Job | [CreateDocumentHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/CreateDocumentHistory.php) | 写入 document_histories |
| 历史模型 | [DocumentHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Document/DocumentHistory.php) | 单据历史模型 |
| 历史模型 | [ModuleHistory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Module/ModuleHistory.php) | 模块历史模型 |
| 辅助函数 | [helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Utilities/helpers.php) | user_id / company_id / source_name |
| 基础模型 | [Abstracts/Model.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Model.php) | use Owners, Sources, Tenants 等 |
