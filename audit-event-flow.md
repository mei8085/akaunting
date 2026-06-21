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

`akaunting/laravel-mutable-observer` 包的 `Mutable` Trait 提供的能力（来自 GitHub 仓库分析）：

- 通过服务容器代理模式实现 Observer 的**运行时静音**，而不是方法修改
- 被 `Mutable` trait 装饰的 Observer 类获得 `mute()` / `unmute()` 两个**静态方法**
- 原理：将 Laravel 容器中已绑定的观察者实例替换为一个代理，代理检查当前事件名是否在"静音列表"里，命中则吞掉调用

**调用方与被调用方的关系（极易混淆）**：

| 概念 | 类 | 位置 |
|------|-----|------|
| **被观察的 Eloquent 模型** | `App\Models\Banking\Transaction` | [app/Models/Banking/Transaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Banking/Transaction.php) |
| **观察模型事件的 Observer 类** | `App\Observers\Transaction` | [app/Observers/Transaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Observers/Transaction.php) |
| **注册绑定时** | `Transaction::observe('App\Observers\Transaction')`（这里 Transaction 是 Model 类） | [app/Providers/Observer.php#L27](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Observer.php#L27) |
| **调用 mute 时** | `use App\Observers\Transaction; Transaction::mute()`（这里 Transaction 是 Observer 类！） | [app/Jobs/Document/DeleteDocument.php#L9-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DeleteDocument.php#L9-L22) |

**关键证据**：DeleteDocument 顶部用了 `use App\Observers\Transaction;`（Observer 类），而不是 Banking Model。这是 PHP 命名空间同名类「覆盖导入」的典型用法。

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
               2. 又想 Insert 一条 DocumentHistory <- 先删后插入孤岛数据，且 histories 已软删，这条也查不到！
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

核心原因：**此时整个单据生态都已经被级联软删了，再写一条 DocumentHistory 既技术上没有障碍，但业务上完全没有意义，且查不到。**

逐一拆解：

**1. 外键约束 — 不存在**
迁移定义（[2019_11_16_000000_core_v2.php#L144-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/database/migrations/2019_11_16_000000_core_v2.php#L144-L159)）：
```php
$table->unsignedInteger('document_id');  // 只是整型字段
$table->index('document_id');            // 只有普通索引
// 没有 $table->foreign('document_id')->references('id')->on('documents')
```
整个项目里，`document_histories.document_id` **没有任何外键约束**（grep 全迁移文件，外键只用于 `transaction_splits.split_id`、`roles_permissions` 等少数几处）。所以"软删 document 导致外键约束报错"的说法不成立。

**2. SoftDeletes 对 INSERT 无影响**
`App\Abstracts\Model` 基类 `use SoftDeletes`（[Abstracts/Model.php#L23](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Model.php#L23)），Document 和 DocumentHistory 都继承了它。但 SoftDeletes 的本质只是给 SELECT/UPDATE/DELETE 自动追加 `WHERE deleted_at IS NULL`，对 INSERT 新记录**完全没有限制**。所以就算 document 已被软删，往 `document_histories` 表插一条带同样 `document_id` 的新记录在数据库层面不会报任何错。

**3. 真正的问题：写了白写 + 语义无意义**

- **查不到**：DocumentHistory 的 `document()` 关联定义是 `belongsTo(Document::class)->withoutGlobalScope(Company::class)`（[DocumentHistory.php#L17-L20](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Document/DocumentHistory.php#L17-L20)），没有 `withTrashed()`，所以从 history 反向查 document 会返回 null。反过来从 `$document->histories` 查询时，document 本身被 SoftDeletes 过滤，正常查询根本拿不到 document，更看不到它的 histories。即便手动 `withTrashed()` 把 document 捞出来，`histories` 关联也只会返回 `deleted_at IS NULL` 的记录 —— 而 `deleteRelationships` 已经把之前所有 history 都软删了。

- **语义无意义**：DeleteDocument 执行的是「整个单据的软删除」，不仅删 documents 表一行，还通过 `deleteRelationships`（[Relationships.php#L41-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Relationships.php#L41-L69)）把 items、item_taxes、histories、transactions、recurring、totals **六个关联全部逐条 `->delete()` 软删**。此时再插一条"单据已被删除"的 history，等于是往一堆 soft deleted 的数据孤岛里再加一条孤岛 —— 想完整恢复这张单据需要把 7 张表全部 `restore()`，单独留一条 history 毫无作用，反而让"恢复"逻辑更困惑（要不要把这条新 history 也算进去？）。

- **与 Akaunting 的审计哲学一致**：如前文所述，Akaunting 不做"所有模型变更都记 diff"的通用审计，删除单据属于「物理/软删除」范畴，交给 `deleted_at` + 服务器日志 + binlog 追溯，不在业务层的 DocumentHistory 里体现。

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

## 补充章节四：PaymentReceived 付款收到事件链路 — 异步队列派发时序与付款通知漏发风险

### 事件注册与监听器属性
位置：[app/Providers/Event.php#L74-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Event.php#L74-L77)

```php
PaymentReceived::class => [
    CreateDocumentTransaction::class,       // ① 创建银行交易
    SendDocumentPaymentNotification::class, // ② 发送付款通知（强依赖 ① 的副作用）
],
```

**关键前提（决定时序的基础）**：
- 两个 Listener 类均**未实现 `ShouldQueue` 接口**（grep 全文确认：[CreateDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/CreateDocumentTransaction.php)、[SendDocumentPaymentNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/SendDocumentPaymentNotification.php) 均只 use Traits，无 implements）
- 因此 Laravel 事件调度器**按注册顺序在当前进程内同步调用**两个 Listener 的 `handle()`，不会把 Listener 自身推到队列
- 竞态风险不来自 Listener 调度，而来自**Listener ① 内部的 `$this->dispatch(...)` 调用**

### dispatch() 内部决策 — should_queue() 函数
位置：[app/Utilities/helpers.php#L136-L144](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Utilities/helpers.php#L136-L144) + [app/Traits/Jobs.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Jobs.php)

```php
// helpers.php
function should_queue(): bool
{
    return ! in_array(config('queue.default'), ['sync', 'null']);
}

// Jobs Trait 的 dispatch() 决策：
if (should_queue()) {
    return dispatch_queue($job);   // dispatchQueue → 推到 Redis/DB 队列，立即返回 PendingDispatch
} else {
    return dispatch_sync($job);    // dispatchSync → 当前进程同步执行完再返回
}
```

### 时序一：queue.default = sync（同步模式，开发/单节点默认）—— 一切正常

```
进程 A  ── HTTP POST /invoices/123/payment ──►
  │
  │ event(new PaymentReceived($document, $request))
  ▼
  ├─► [Listener ①] CreateDocumentTransaction::handle()
  │     │
  │     │ $this->dispatch(new CreateBankingDocumentTransaction(...))
  │     │   → should_queue() === false → dispatch_sync
  │     │
  │     ├─► [同步执行] CreateBankingDocumentTransaction::handle()
  │     │     │
  │     │     ├─► dispatch(new CreateTransaction(...)) → dispatch_sync
  │     │     │     └─► INSERT transactions 表完成 ✅
  │     │     │
  │     │     ├─► $document->paid_amount = xxx; $document->save() 完成 ✅
  │     │     │
  │     │     └─► dispatch(new CreateDocumentHistory(...)) → dispatch_sync
  │     │           └─► INSERT document_histories 表完成 ✅
  │     │
  │     └─ 返回 Transaction 对象
  │
  ├─► [Listener ②] SendDocumentPaymentNotification::handle()
  │     │
  │     │ $event->request['type'] === 'income' ✔（进入发送逻辑）
  │     │
  │     │ $transaction = $document->transactions()->latest()->first();
  │     │   ↑ 事务已在同一 DB 连接中提交，SELECT 能拿到刚 INSERT 的行
  │     │   ↑ 返回非 null 的 Transaction 实例 ✔
  │     │
  │     └─► Notification::sendNow($contact, new PaymentReceived($transaction))
  │         通知正常发出 ✅
  │
  └─ HTTP 200 响应返回
```

### 时序二：queue.default = redis/database（异步队列模式）—— 静默漏发

```
进程 A  ── HTTP POST /invoices/123/payment ──►
  │
  │ event(new PaymentReceived($document, $request))
  ▼
  ├─► [Listener ①] CreateDocumentTransaction::handle()
  │     │
  │     │ $this->dispatch(new CreateBankingDocumentTransaction(...))
  │     │   → should_queue() === true → dispatch_queue
  │     │   → 向队列 Redis 推一条 Job 消息
  │     │   → 立即返回 PendingDispatch 对象，handle() 此时**根本没被执行**！
  │     │   → transactions 表**还没有**这条记录 ❌
  │     │
  │     └─ 返回（约 1ms 级别，立即）
  │
  ├─► [Listener ②] SendDocumentPaymentNotification::handle()
  │     │   ← Listener ① 返回后紧接着执行（同一进程，同步顺序）
  │     │
  │     │ $event->request['type'] === 'income' ✔
  │     │
  │     │ $transaction = $document->transactions()->latest()->first();
  │     │   ↑ 此时 Queue Worker 可能还没抢到任务，也可能抢到了但还在 INSERT 阶段
  │     │   ↑ SELECT 返回 null ❌
  │     │
  │     │ if (! $transaction) { return; }   // ← 直接 return
  │     │                                     // ← 无 Log、无 Exception、无 flash
  │     │
  │     └─ 静默结束，什么都没发 ❌
  │
  └─ HTTP 200 响应返回（用户看起来成功了）


         ▓▓▓ 几毫秒到几秒之后 ▓▓▓

  [Queue Worker B] 从 Redis 取出 CreateBankingDocumentTransaction
     ├─► handle() 正常执行
     │     ├─► INSERT transactions ✅
     │     ├─► UPDATE documents.paid_amount ✅
     │     └─► INSERT document_histories ✅
     │
     └─► Transaction 落盘了，但付款通知**永远不会再触发**（没有 after 补偿逻辑）
```

### SendDocumentPaymentNotification 的静默漏发关键代码
位置：[app/Listeners/Document/SendDocumentPaymentNotification.php#L18-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/SendDocumentPaymentNotification.php#L18-L27)

```php
public function handle(Event $event): void
{
    // 第一层 guard — 非收入类型（bill）不通知
    if ($event->request['type'] !== 'income') {
        return;
    }

    $document = $event->document;
    $transaction = $document->transactions()->latest()->first();

    // 第二层 guard — 拿不到交易直接 return
    if (! $transaction) {
        return;   // ← 异步模式下 99% 命中这个 return 分支
    }

    // 只有同步模式下才能走到下面的 Notification::send
    Notification::send(...);
}
```

**漏发风险量化**：假设 Queue Worker 平均延迟 50ms（非常优秀的水平），Listener ② 在 Listener ① 返回后 1ms 内执行 —— SELECT 比 INSERT 提前约 49ms，**100% 拿不到**。只有在极端情况下（INSERT 极快、或者 Worker 恰好在 SELECT 前的几微秒内刚好 COMMIT）才会命中，基本等于必漏。

### 问题根因五层总结

| 层次 | 具体问题 |
|------|---------|
| **Listner 耦合设计** | Listener ② 强依赖 Listener ① 的副作用（新 transaction 已在 DB），但两者之间没有 `after_commit` / Promise / await 语义 |
| **调度不可见** | 同步 Listener 内部通过 `should_queue()` 动态决定 dispatch 模式，调用方完全感知不到时序变化 |
| **事务边界模糊** | Listener ① 内部的 DB 写入不在 PaymentReceived 的事务范围内（异步模式下根本在另一进程） |
| **静默失败** | 拿不到 transaction 就 `return;`，没有 `Log::warning()`、没有 `report()`、没有队列延迟重试 |
| **命名误导** | Listener 命名 `SendDocumentPaymentNotification` 暗示"文档已付款→发通知"，但在异步模式下它的真实语义更接近"如果此刻能找到最新付款交易就发通知" |

### 修复方向（非本任务范围，仅提示）
- **方案 A**：把通知逻辑移到 `CreateBankingDocumentTransaction::handle()` 末尾，在同一 DB 事务内，当 Transaction 和 History 都落盘后手动 dispatch（或用 `DB::afterCommit()` 钩子）
- **方案 B**：让 SendDocumentPaymentNotification 实现 `ShouldQueue` + `->delay(now()->addSeconds(3))`，延迟 3 秒等 Worker 落盘后再查（牺牲实时性）
- **方案 C**：在 `return;` 前至少 `Log::warning('Payment notification skipped: transaction not found', [...])`，便于事后排查

---

## 补充章节五：Banking 模块三个核心 Transaction Job 的 HasOwner / HasSource 实现矩阵

### 前置：Banking Transaction 类家族的调用关系图

```
                       ┌──────────────────────────────────────┐
                       │  外部入口（Listener / Controller 等） │
                       └─────────────┬────────────────────────┘
                                     │
                                     ▼
                   CreateBankingDocumentTransaction  (编排层)
                      implements ShouldCreate  ✅
                      implements HasOwner      ❌
                      implements HasSource     ❌
                                     │
                                     │  内部 $this->dispatch(...)
                                     ▼
                          CreateTransaction  (落盘层)
                             implements ShouldCreate  ✅
                             implements HasOwner      ✅
                             implements HasSource     ✅
                                     │
                                     │  内部 $this->dispatch(...)
                                     ▼
                       CreateTransactionTaxes  (子实体层)
                          implements ShouldCreate  ✅
                          implements HasOwner      ✅
                          implements HasSource     ✅
```

### 核心三 Job 实现矩阵

| 维度 | **CreateTransaction** | **CreateBankingDocumentTransaction** | **CreateTransactionTaxes** |
|------|---------------------|--------------------------------------|--------------------------|
| 代码文件 | [app/Jobs/Banking/CreateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Banking/CreateTransaction.php#L8-L14) | [app/Jobs/Banking/CreateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L11-L17) | [app/Jobs/Banking/CreateTransactionTaxes.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Banking/CreateTransactionTaxes.php#L6-L13) |
| `implements ShouldCreate` | ✅ 是 | ✅ 是 | ✅ 是 |
| `implements HasOwner` | ✅ 是 | ❌ 否 | ✅ 是 |
| `implements HasSource` | ✅ 是 | ❌ 否 | ✅ 是 |
| `implements ShouldUpdate` | ❌ 否 | ❌ 否 | ❌ 否 |
| 继承链 | extends Job | extends Job | extends Job |
| 角色定位 | **落盘层**：最终 INSERT transactions 表 | **编排层**：组装参数、校验金额、更新单据状态、写历史 | **子实体层**：INSERT transaction_taxes 表 |
| bootCreate 阶段行为 | `setOwner()` 注入 `created_by`，`setSource()` 注入 `created_from` 到 `$this->request` | 仅做 arguments 识别和 Request 转换，**不注入归属字段** | 同 CreateTransaction |
| 归属字段注入方式 | **直接注入**：bootCreate() → merge 进 request | **委托注入**：本 Job 不处理，内部 dispatch `CreateTransaction` 时由它的 bootCreate() 二次注入 | **直接注入** |
| 主要调用者 | CreateBankingDocumentTransaction 内部、CreateTransfer 内部、CreateReconciliation 内部、直接调用（手动建收支） | Listener `CreateDocumentTransaction`（PaymentReceived 事件触发） | CreateTransaction 内部 handle 末尾 |
| created_by / created_from 覆盖可能性 | ✅ 调用方可以在传入 request 时预先设置，HasOwner/HasSource 会跳过已存在的键 | ❌ 编排层本身不可覆盖，但可通过设置传给内部 CreateTransaction 的 request 实现 | ✅ 同 CreateTransaction |

### 设计动机解读（编排层不落盘，不归因）

三个 Job 的接口差异遵循一条核心原则：**谁最终执行 Model::create() 落盘，谁负责归因（记录 created_by/created_from）**。

具体到三 Job：
1. `CreateBankingDocumentTransaction` 的职责是**业务编排**——校验付款金额与单据金额匹配、更新 `$document->status`（partial/paid）、写付款的 DocumentHistory。它自己不执行 `Transaction::create()`，所以不需要归因。
2. `CreateTransaction` 的职责是**执行落盘**——真正跑 `Transaction::create($request)`，因此必须实现 HasOwner + HasSource。
3. 编排层把 `$this->request` 原样传给落盘层时，落盘层的 `setOwner()` 里有 `if ($request->has('created_by')) return;` 的跳过逻辑，所以编排层**可以主动预先设置**特定 created_by（如导入场景指定历史付款人），此时落盘层不会覆盖；反之如果编排层什么都不设，落盘层会自动填入当前 user_id()。

### Banking 模块其它 Job 的接口情况（验证规律普适性）

| Job 类名 | 类型 | ShouldCreate | ShouldUpdate | ShouldDelete | HasOwner | HasSource |
|---------|------|:------------:|:------------:|:------------:|:--------:|:---------:|
| CreateTransfer | 创建 | ✅ | ❌ | ❌ | ✅ | ✅ |
| CreateAccount | 创建 | ✅ | ❌ | ❌ | ✅ | ✅ |
| CreateReconciliation | 创建 | ✅ | ❌ | ❌ | ✅ | ✅ |
| UpdateTransaction | 更新 | ❌ | ✅ | ❌ | ❌ | ❌ |
| UpdateTransfer | 更新 | ❌ | ✅ | ❌ | ❌ | ❌ |
| UpdateBankingDocumentTransaction | 更新 | ❌ | ✅ | ❌ | ❌ | ❌ |
| UpdateAccount | 更新 | ❌ | ✅ | ❌ | ❌ | ❌ |
| UpdateReconciliation | 更新 | ❌ | ✅ | ❌ | ❌ | ❌ |
| SplitTransaction | 更新 | ❌ | ✅ | ❌ | ❌ | ❌ |
| DeleteTransaction | 删除 | ❌ | ❌ | ✅ | ❌ | ❌ |
| DeleteTransfer | 删除 | ❌ | ❌ | ✅ | ❌ | ❌ |
| DeleteAccount | 删除 | ❌ | ❌ | ✅ | ❌ | ❌ |
| DeleteReconciliation | 删除 | ❌ | ❌ | ✅ | ❌ | ❌ |

**规律 100% 成立**：创建型 Job = ShouldCreate + HasOwner + HasSource 三件套；更新/删除型 Job = ShouldUpdate/ShouldDelete 单独存在，不处理归属字段（保留创建时的原值）。

---

## 补充章节六：DocumentSent 与 DocumentMarkedSent 的事件分流 — 触发位置与描述分流

### 分流的业务语义背景

"单据已发送"在 ERP 产品中有两种本质不同的业务含义，Akaunting 用两个事件区分：

| 事件类 | 业务语义 | 可信度 | 典型触发场景 |
|--------|---------|--------|------------|
| **DocumentSent** | 系统刚刚通过 SMTP/Mail driver 把邮件**投递到 MTA**（发出了） | 高（系统行为，可验证） | 用户点"发送"按钮 → 系统调用 `notify()` 成功后触发；循环单据自动发送 |
| **DocumentMarkedSent** | 用户表示"我已经通过某种方式发给客户了"（不管用什么方式） | 低（用户声明，不可验证） | 批量操作菜单勾选多张 → 点击"标记为已发送" |

两种语义的最终效果在**状态字段**上相同（都把 `documents.status` 改成 `sent`），但**审计历史**必须区分——这就是事件分流的根本原因。

### 各事件的具体触发位置（按事件类聚合）

#### 事件 A：`DocumentSent` — 邮件真的发出了

**触发点 1**：标准单据发送 Job
位置：[app/Jobs/Document/SendDocument.php#L26](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/SendDocument.php#L26)

```php
// SendDocument::handle()
if ($this->document->contact && $this->document->contact->email) {
    $this->document->contact->notify(
        new DocumentSent($this->document)   // Laravel Notification 发送邮件
    );
}
event(new DocumentSent($this->document));   // 邮件发完才触发事件
// ↑ 注意：即使 contact->email 为空（没有收件人），也会触发事件
```

**触发点 2**：自定义内容邮件发送 Job
位置：[app/Jobs/Document/SendDocumentAsCustomMail.php#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/SendDocumentAsCustomMail.php#L65)

```php
// SendDocumentAsCustomMail::handle()
Notification::route('mail', $request->to_address)
    ->notify(new CustomDocument($this->document, $request));
event(new DocumentSent($this->document));   // 自定义邮件成功发出后触发
```

**触发点 3**：循环单据自动通知（动态事件类名）
位置：[app/Listeners/Document/SendDocumentRecurringNotification.php#L40-L42](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L40-L42)

```php
$event_class = config('type.' . $document->type . '.event.sent', DocumentSent::class);
event(new $event_class($document));
// ↑ 从配置文件动态读取事件类，默认就是 DocumentSent
//   这样模块可以自定义自己的 sent 事件
```

#### 事件 B：`DocumentMarkedSent` — 用户手动标记了

**触发点 1**：销售发票批量操作
位置：[app/BulkActions/Sales/Invoices.php#L108-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/BulkActions/Sales/Invoices.php#L108-L113)

```php
// Invoices::sent() 方法
$invoices = Invoice::whereIn('id', $request->get('selected'))->cursor();
foreach ($invoices as $invoice) {
    if (in_array($invoice->status, ['partial', 'paid'])) continue;
    event(new DocumentMarkedSent($invoice));
}
```

**触发点 2**：采购账单批量操作
位置：[app/BulkActions/Purchases/Bills.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/BulkActions/Purchases/Bills.php) 的 `received()` 方法（结构完全相同，只是事件是 `DocumentReceived` 而非 `DocumentMarkedSent`——因为账单的"标记已收到"语义对应 Received 事件，不是 MarkedSent）

**设计观察**：Sales 端有"标记已发送"（给客户），Purchases 端对称的是"标记已收到"（从供应商收到）。所以 Purchases 没有 `DocumentMarkedSent`，只有 `DocumentReceived`。

### 合流：同一个 Listener 处理两个事件

事件服务提供者注册（[app/Providers/Event.php#L78-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Providers/Event.php#L78-L83)）：
```php
DocumentMarkedSent::class => [ MarkDocumentSent::class ],
DocumentSent::class       => [ MarkDocumentSent::class ],
// 同一个 Listener 类绑定到两个不同事件
```

#### Listener 内部的"事件合流 → 描述分流"机制
位置：[app/Listeners/Document/MarkDocumentSent.php#L10-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/MarkDocumentSent.php#L10-L45)

```php
class MarkDocumentSent
{
    // ★ PHP 8 联合类型：handle 方法的 $event 形参接受两种事件类
    public function handle(DocumentMarkedSent|DocumentSent $event): void
    {
        $document = $event->document;

        // 合流部分（两事件逻辑相同）
        if (! in_array($document->status, ['partial', 'paid'])) {
            $document->status = 'sent';
            $document->save();
        }

        // 描述生成时再次分流
        $this->dispatch(new CreateDocumentHistory(
            $document, 0, $this->getDescription($event)
            //                        ^^^^^^^^^^^^^^^^^^^^—— 在这里按事件类型区分
        ));
    }

    // ★ 真正的二次分流点
    protected function getDescription(DocumentMarkedSent|DocumentSent $event): string
    {
        $type_text = match($event->document->type) {
            Document::INVOICE_TYPE  => trans_choice('general.invoices', 1),
            Document::BILL_TYPE     => trans_choice('general.bills', 1),
            // ... 其他类型
        };

        // 核心：根据事件类不同选不同的多语言 key
        $message_key = ($event instanceof DocumentMarkedSent)
            ? 'documents.messages.marked_sent'   // "X 已标记为已发送"
            : 'documents.messages.email_sent';   // "X 已通过邮件发送"

        return trans($message_key, ['type' => $type_text]);
    }
}
```

#### 最终历史描述差异表（以销售发票为例）

| 事件 | 多语言 key | 英文描述 | 中文（假设语言包为 zh-CN） | 写入 document_histories.description |
|------|-----------|---------|--------------------------|-----------------------------------|
| `DocumentSent` | `documents.messages.email_sent` | `Sales Invoice emailed` | `销售发票已通过邮件发送` | `销售发票已通过邮件发送` |
| `DocumentMarkedSent` | `documents.messages.marked_sent` | `Sales Invoice marked as sent` | `销售发票已标记为已发送` | `销售发票已标记为已发送` |

### 分流设计的数据流完整视图

```
  ┌───────────── 触发点汇总 ─────────────┐
  │                                      │
  │  SendDocument         (邮件真发了)    │────┐
  │  SendDocumentAsCustomMail (自定义)    │────┼──► new DocumentSent
  │  SendDocumentRecurringNotification    │────┘        │
  │                                                     │
  │  BulkActions/Sales/Invoices::sent()  (用户标记)  ───┼──► new DocumentMarkedSent
  │                                                     │
  └─────────────────────────────────────────────────────┘
                              │
                              ▼
               Laravel Dispatcher (按 Event::$listen 路由)
                              │
         ┌────────────────────┴────────────────────┐
         │ 两个事件都路由到同一个 Listener 类        │
         ▼                                         ▼
  MarkDocumentSent::handle(DocumentSent)    MarkDocumentSent::handle(DocumentMarkedSent)
         │                                         │
         ├─────────────────────────────────────────┤
         │ 合流：相同的 status 更新逻辑             │
         │                                         │
         ▼                                         ▼
  CreateDocumentHistory(..., getDescription())  CreateDocumentHistory(..., getDescription())
         │                                         │
         ▼                                         ▼
  getDescription 判断 instanceof               getDescription 判断 instanceof
    → 'documents.messages.email_sent'            → 'documents.messages.marked_sent'
         │                                         │
         ▼                                         ▼
  INSERT document_histories                    INSERT document_histories
  描述 = "销售发票已通过邮件发送"               描述 = "销售发票已标记为已发送"
         │                                         │
         └────────────────────┬────────────────────┘
                              │
                              ▼
                   两张表结构完全一样，只有 description 不同
                   审计追踪时可据此判断：是真发了还是手标了
```

---

## 对 DocumentDeleted 章节分析的最终修正

> **前置说明**：此前版本曾出现过"软删 Document 导致外键约束报错"、"逻辑荒谬"等主观或错误的表述。以下为基于代码的准确分析，所有结论可对照迁移文件、模型定义、源码逐行验证。

### 分析步骤一：检查"外键约束"是否存在 — 结论：不存在

迁移文件 [2019_11_16_000000_core_v2.php#L144-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/database/migrations/2019_11_16_000000_core_v2.php#L144-L159) 中 `document_histories` 表的字段定义：

```php
Schema::create('document_histories', function (Blueprint $table) {
    $table->unsignedInteger('document_id');    // ① 声明字段类型
    // ...
    $table->index('document_id');              // ② 建普通索引（加速查询）
    // 没有：$table->foreign('document_id')->references('id')->on('documents')
    // 没有：->onDelete('cascade') 之类外键约束语句
});
```

全项目 grep 外键语句，外键仅出现在 `transaction_splits.split_id`、`role_user`、`permission_role` 等少数关联表。`document_histories.document_id` **从未声明过 `FOREIGN KEY` 约束**。

因此"往 document_histories 插一条指向已软删 document_id 的行，会触发数据库外键约束异常"的说法在技术层面**完全不成立**。

### 分析步骤二：检查 SoftDeletes 对 INSERT 新记录有无限制 — 结论：无影响

`App\Abstracts\Model` 基类（[Abstracts/Model.php#L23](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Abstracts/Model.php#L23)）使用 `Illuminate\Database\Eloquent\SoftDeletes` Trait，Document 和 DocumentHistory 均继承了它。

根据 Laravel 框架行为，SoftDeletes Trait 的作用域仅在：
- **SELECT**：自动追加 `WHERE deleted_at IS NULL`
- **UPDATE**：自动追加 `WHERE deleted_at IS NULL`（但 `$model->update()` 是按主键更新，走的是 set 方式，不走全局 scope）
- **DELETE**：拦截后改为 `UPDATE ... SET deleted_at = NOW()`

对 **INSERT 新记录**，SoftDeletes 没有任何限制、没有任何钩子。所以即使 `documents.id = 123` 这一行已被软删（`deleted_at` 非空），执行 `DocumentHistory::create(['document_id' => 123, ...])` 在**数据库层面和 Eloquent 层面都不会报错**。

### 分析步骤三：如果在 DocumentDeleted 监听器里补写一条"删除历史"，实际会发生什么？

重新对照 DeleteDocument 的执行顺序（[DeleteDocument.php#L14-L36](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/DeleteDocument.php#L14-L36)）：

```
1. event(new DocumentDeleting)     → 事务前，无监听器
2. DB::transaction 开始
3.   Transaction::mute()
4.   deleteRelationships(...)
       遍历 items/item_taxes/histories/transactions/recurring/totals
       → 每个 $item->delete()（逐条软删，走 Eloquent deleted 事件）
       → histories 关联的 N 行 DocumentHistory 全部被软删（deleted_at 非空）
5.   $this->model->delete()        → documents 该行被软删
6.   Transaction::unmute()
7. DB::transaction 提交             → 以上所有 UPDATE(deleted_at) 持久化
8. event(new DocumentDeleted)      → 事务提交后，此时可补写历史
```

如果在第 8 步注册一个监听器，执行 `DocumentHistory::create(['document_id' => 123, 'description' => '单据已删除'])`：

- 数据库层面：**正常 INSERT 一行**，没有任何错误
- 该行的 `deleted_at` 为 `null`（因为是新 INSERT 的，没有 delete 操作）
- 但 `document_id = 123` 指向的 documents 行已被软删

### 分析步骤四：补写这条历史后，业务查询层面会遇到什么？

分三种查询方向分析：

| 查询场景 | 查询代码 | 实际结果 | 是否符合预期 |
|---------|---------|---------|-------------|
| 正常查看单据详情 | `Document::find(123)` | 返回 `null`（SoftDeletes 过滤掉了被软删的 document） | ✔ 被删的单据不应该看到，符合预期 |
| Document 反向查历史 | `$document->histories` | 根本走不到（document 查不到） | — |
| 管理后台查所有历史（独立列表） | `DocumentHistory::latest()->paginate()` | 能看到"单据已删除"这条新 history（`deleted_at` 是 null，没被过滤） | ✔ 能看到 |
| 从第 3 条 history 点进 Document | `$history->document`（[DocumentHistory.php#L17-L20](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Document/DocumentHistory.php#L17-L20)） | 返回 `null`（SoftDeletes 全局 Scope 过滤了被软删的 document；注意 `withoutGlobalScope('App\Scopes\Document')` 只移除自定义的 Document Scope，**不移除 SoftDeletes Scope**） | ❌ 能看到 history 但点进去 null，体验断层 |
| 管理后台"回收站"（withTrashed） | `Document::withTrashed()->find(123)->histories` | 能拿到 document；但 `histories()` 关联只返回 `deleted_at IS NULL` 的行（SoftDeletes Scope） | ⚠ 结果：只能看到刚 INSERT 的那条"单据已删除"，之前的 N 条历史因为被软删了（步骤 4）全部看不到 |
| 要完整看到单据+所有历史 | `Document::withTrashed()->find(123)->histories()->withTrashed()->get()` | 此时才能看到 旧 N 条（被软删的）+ 1 条新的 | ❌ 没有任何 UI 这么写 |

### 最终结论：Akaunting 选择不补写"删除历史"的真正、代码层面的理由

按重要性排序：

| 序号 | 理由 | 代码依据 |
|------|------|---------|
| 1 | **数据完整性**：DeleteDocument 的 `deleteRelationships` 已把此前所有 DocumentHistory 软删，仅留一条"已删除"会给回收站视图造成信息断层（只剩操作记录，没有之前的上下文） | [Relationships.php#L41-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Relationships.php#L41-L69) 中 `histories` 在删除清单里 |
| 2 | **审计哲学一致性**：Akaunting 的 DocumentHistory 记录的是「单据生命周期中的状态跃迁」（创建/发送/付款/查看/取消/恢复），而 Delete 属于"生命周期终止"——终止之后追加一条终止声明，属于归档范畴，不属于"单据状态变更" | 对照所有 DocumentHistory 写入点，全部是状态变更事件的 Listener |
| 3 | **UI/体验断层**：如上分析，补写的 history 能被"历史总览"看到，但点击进入 document 返回 null，体验不佳（除非同时修改所有 history 查询都加 `withTrashed()` on the relation） | [DocumentHistory.php#L17-L20](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Models/Document/DocumentHistory.php#L17-L20) 关系定义无 `withTrashed()` |
| 4 | **Delete 与 Cancel 的语义分离**：用户想"中止单据 + 保留完整审计" → 应该走 Cancel（DocumentCancelled 事件，有监听器，正常写 history，保留所有关联数据）；Delete 是明确要"从列表中抹去"，抹干净比留一条更清晰 | 对照 [补充章节二] 中 DeleteDocument vs CancelDocument 对比表 |
| 5 | **删除追溯靠基础设施**：如果真要查谁删的，用 `deleted_at` 时间戳 + Web 服务器 access_log + 运维层面 DB binlog，比业务层一条 history 更可靠且不可篡改 | 无需代码依据，行业通用实践 |

这五条理由是 Akaunting 不注册 DocumentDeleted 监听器的完整说明。之前"外键约束会报错"、"逻辑荒谬"等表述均为不准确、情绪化的说法，已移除。

---
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
| 付款通知监听器（竞态相关） | [SendDocumentPaymentNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/SendDocumentPaymentNotification.php) | 查 transactions 为空时静默 return |
| 付款交易创建监听器 | [CreateDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/CreateDocumentTransaction.php) | dispatch CreateBankingDocumentTransaction |
| 银行单据交易 Job | [CreateBankingDocumentTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php) | 编排 Job，HasOwner/HasSource 委托给 CreateTransaction |
| 银行交易基础 Job | [CreateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Banking/CreateTransaction.php) | ShouldCreate + HasOwner + HasSource 三件套 |
| 银行交易税项 Job | [CreateTransactionTaxes.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Banking/CreateTransactionTaxes.php) | 交易子实体，同样三件套 |
| 邮件发送触发事件 | [DocumentSent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Events/Document/DocumentSent.php) | 真发邮件后的事件 |
| 人工标记触发事件 | [DocumentMarkedSent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Events/Document/DocumentMarkedSent.php) | 用户手动标记的事件 |
| 发送单据邮件 Job | [SendDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Jobs/Document/SendDocument.php) | notify() 后触发 DocumentSent |
| 批量标记入口 | [BulkActions/Sales/Invoices.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/BulkActions/Sales/Invoices.php) | sent() 方法触发 DocumentMarkedSent |
| 合并监听器（两事件合流） | [MarkDocumentSent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Listeners/Document/MarkDocumentSent.php) | handle 接受联合类型，按 instanceof 分流文案 |
| 队列调度辅助函数 | [helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Utilities/helpers.php#L136-L144) | should_queue() 根据 queue.default 判断 sync |
| Jobs Trait | [Jobs.php](file:///d:/fz/0601-2/solo-dogfeeding/code/64-akaunting/app/Traits/Jobs.php) | dispatch() 动态选择 dispatchSync / dispatchQueue |

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
