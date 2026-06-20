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
