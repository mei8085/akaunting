# Recurring Document 调度与生成单据流程

## 一、整体架构概览

Recurring (定期/循环) 功能支持以下4种类型的单据自动周期性生成：

| 类型 | 模板 type | 生成后 type | 对应模型 |
|------|-----------|-------------|----------|
| 销售发票 | `invoice-recurring` | `invoice` | Document |
| 采购账单 | `bill-recurring` | `bill` | Document |
| 收入 | `income-recurring` | `income` | Transaction |
| 支出 | `expense-recurring` | `expense` | Transaction |

系统使用第三方库 `bkwld/laravel-cloner` (Cloneable Trait) 实现模型克隆，使用 `simshaun/recurr` (Recurr\Rule + ArrayTransformer) 生成 RRULE 重复日程。

---

## 二、调度入口

### 2.1 调度器配置

文件：[Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Kernel.php#L23-L37)

```php
protected function schedule(Schedule $schedule)
{
    $schedule_time = config('app.schedule_time');

    $schedule->command('recurring:check')
        ->dailyAt($schedule_time)
        ->runInBackground();
}
```

- **执行频率**：每天一次，时间由 `app.schedule_time` 配置决定
- **运行模式**：后台执行 (`runInBackground()`)
- **命令签名**：`recurring:check`

### 2.2 命令类

文件：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php)

核心方法 `handle()` 是整个调度流程的入口。

---

## 三、创建 Recurring 模板（前置准备）

在调度执行之前，用户需先创建一条 **Recurring 模板**。模板就是带有 `-recurring` 后缀 type 的 Document 或 Transaction 记录。

### 3.1 Document 模板创建路径

**Job**：[CreateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Jobs/Document/CreateDocument.php#L20-L57)

```
用户提交创建请求
    ↓
DocumentCreated Event 前置检查
    ↓
DB::transaction {
    Document::create(...)         ← 落库 documents 表
    ↓
    CreateDocumentItemsAndTotals  ← 创建明细行和总计行
    ↓
    $model->update(...)
    ↓
    $model->createRecurring(...)  ← ★ 创建 recurring 记录
}
    ↓
DocumentCreated Event 触发
```

关键代码 (第 51 行)：
```php
$this->model->createRecurring($this->request->all());
```

### 3.2 Transaction 模板创建路径

**Job**：[CreateTransaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Jobs/Banking/CreateTransaction.php#L16-L52)

```
用户提交创建请求
    ↓
TransactionCreating Event
    ↓
判定 type：若设置了 recurring_frequency 则设置为 *-recurring 类型
    ↓
DB::transaction {
    Transaction::create(...)      ← 落库 transactions 表
    ↓
    CreateTransactionTaxes        ← 创建税项
    ↓
    $model->createRecurring(...)  ← ★ 创建 recurring 记录
}
    ↓
TransactionCreated Event 触发
```

### 3.3 createRecurring() 核心逻辑

**Trait**：[Traits/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L13-L43)

方法 `createRecurring($request)` 从请求中提取定期参数，向 `recurring` 表插入一条记录：

| 字段 | 说明 | 来源 |
|------|------|------|
| `company_id` | 公司ID | $this->company_id |
| `frequency` | 频率 (daily/weekly/monthly/yearly/custom) | recurring_frequency / recurring_custom_frequency |
| `interval` | 间隔周期 (数字) | recurring_interval，默认 1 |
| `started_at` | 开始日期 | recurring_started_at，默认今天 |
| `status` | 状态 | recurring_status，默认 active |
| `limit_by` | 限制方式 (count/date) | recurring_limit，默认 count |
| `limit_count` | 限制次数 | recurring_limit_count，默认 0 (无限) |
| `limit_date` | 限制截止日期 | recurring_limit_date |
| `auto_send` | 是否自动发送邮件 | recurring_send_email，默认 0 |

### 3.4 模板识别：isRecurring 作用域

识别一个 Document 或 Transaction 是否为 recurring **模板**的方式是检查 `type` 字段是否以 `-recurring` 结尾。

**Abstract Model**：[Model.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Abstracts/Model.php#L242-L250)

```php
public function scopeIsRecurring(Builder $query): Builder
{
    return $query->where($this->qualifyColumn('type'), 'like', '%-recurring');
}
```

Recurring Model 的 `recurable()` 多态关联中也使用了此 scope：

```php
// App\Models\Common\Recurring
public function recurable()
{
    return $this->morphTo()->isRecurring();  // 只取模板类型
}
```

---

## 四、调度执行：RecurringCheck::handle()

**文件**：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L40-L162)

### 4.1 主流程

```
handle() 入口
    ↓
① 预准备：关闭模型缓存、绑定自身到容器
    ↓
② 获取所有 active 状态的 recurring 记录游标
    ↓
③ 逐条遍历 $recur {
    3.1 公司有效性检查 (不存在/禁用/无活跃用户 则清理)
    ↓
    3.2 company($company_id)->makeCurrent()  ← 切换公司上下文
    ↓
    3.3 获取关联模板 $template = $recur->recurable()->first()
    ↓
    3.4 getRemainingSchedules()  ← 计算剩余待生成的日程
    ↓
    3.5 若无剩余日程 → status = completed，跳过
    ↓
    3.6 endsBefore(明天) 过滤 → 只取今天及之前到期的
    ↓
    3.7 逐条遍历 $schedules {
        recur($template, $schedule_date)  ← ★ 生成单据
    }
}
    ↓
④ 清除当前公司上下文，结束
```

### 4.2 公司有效性检查（跳过/清理策略）

| 检查项 | 条件 | 处理 |
|--------|------|------|
| 公司不存在 | `empty($recur->company)` | 删除 recurring + 模板，跳过 |
| 公司禁用 | `!$recur->company->enabled` 且 company 更新时间 > 3个月前 | 删除 recurring + 模板，跳过 |
| 无活跃用户 | 所有用户 `last_logged_in_at` 均超过 3个月 | 删除 recurring + 模板，跳过 |

### 4.3 计算剩余日程：getRemainingSchedules()

**文件**：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L243-L262)

```
getRemainingSchedules($template, $recur)
    ↓
① 确定日期字段：
    - Document → issued_at
    - Transaction → paid_at
    ↓
② 查询已生成的子单据日期：
    SELECT `date_field` FROM `table`
    WHERE type = 去掉-recurring后缀的真实类型
      AND parent_id = 模板id
    ↓
③ 调用 $recur->getRecurringSchedule() 获取完整日程列表
    ↓
④ 过滤：排除 已生成日期 存在于 ②中的日程
    ↓
返回 RecurrenceCollection (剩余待生成)
```

关键点：**生成过的单据通过 `parent_id` 与模板关联**，用日期去重来保证不重复生成。

### 4.4 日程生成引擎：getRecurringSchedule()

**Trait**：[Traits/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L93-L121)

基于 `simshaun/recurr` 库：

```
getRecurringSchedule()
    ↓
① 构建 ArrayTransformerConfig：
   - enableLastDayOfMonthFix() → 月末日期修正 (1/31→2/28)
   - setVirtualLimit()         → 虚拟上限 (daily=732, weekly=104, monthly=24, yearly=2)
    ↓
② 构建 Recurr\Rule：
   - setStartDate(started_at)
   - setFreq(frequency)        → WEEKLY/MONTHLY/...
   - setInterval(interval)     → 间隔数
   - 若 limit_by=date → setUntil(limit_date)
   - 若 limit_by=count 且 limit_count>0 → setCount(limit_count)
    ↓
③ ArrayTransformer::transform($rule)  → RecurrenceCollection
```

---

## 五、生成单据：recur() 方法

**文件**：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L164-L188)

```php
protected function recur(Document|Transaction $template, Date $schedule_date): void
{
    DB::transaction(function () use ($template, $schedule_date) {
        if (! $model = $this->getModel($template, $schedule_date)) {
            return;
        }

        switch ($template::class) {
            case Document::class:
                event(new DocumentCreated($model, request()));
                event(new DocumentRecurring($model));
                break;
            case Transaction::class:
                event(new TransactionCreated($model));
                event(new TransactionRecurring($model));
                break;
        }
    });
}
```

- 每个 schedule 在独立的数据库事务中执行
- 异常被 `getModel()` 捕获，不影响其他 schedule 继续执行
- 事务成功后触发对应的 Created 事件和 Recurring 事件

### 5.1 getModel() 分发

```
getModel($template, $schedule_date)
    ↓
判断模板类型：
    Transaction → getTransactionModel()
    Document    → getDocumentModel()
    ↓
try { 调用具体方法 } catch { report + return false }
```

### 5.2 生成 Document 子单据：getDocumentModel()

**文件**：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L205-L224)

```
1. $template->cloneable_relations = ['items', 'totals']
   (覆盖默认的 ['items', 'recurring', 'totals']，不克隆 recurring 关联)
    ↓
2. $model = $template->duplicate()
   → 调用 Bkwld\Cloner\Cloneable::duplicate()
   → 复制主表记录 + cloneable_relations 指定的关联记录
   → 新记录 id 已生成，但尚未 save()（等后续字段赋值后再 save）
    ↓
3. 计算 due_at 与 issued_at 的差值天数 $diff_days
    ↓
4. 赋值关键字段：
   $model->type         = Str::replace('-recurring', '', $template->type)  // invoice-recurring → invoice
   $model->parent_id    = $template->id         // 建立父子关系
   $model->issued_at    = $schedule_date        // 计划日期作为开单日期
   $model->due_at       = $schedule_date + $diff_days  // 按模板比例顺延到期日
   $model->created_from = 'core::recurring'      // 标记来源
    ↓
5. $model->save()         ← ★ 主记录落库 (documents 表)
    ↓
6. updateRelationTypes() → 将关联的 items、totals 记录的 type 字段也改为新类型 (invoice)
    ↓
返回 $model
```

### 5.3 生成 Transaction 子单据：getTransactionModel()

**文件**：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L226-L241)

```
1. $template->cloneable_relations = ['taxes']
   (覆盖默认的 ['recurring', 'taxes']，不克隆 recurring)
    ↓
2. $model = $template->duplicate()
   → 复制主表记录 + taxes 关联
    ↓
3. 赋值关键字段：
   $model->type         = 去掉 -recurring 后缀
   $model->parent_id    = $template->id
   $model->paid_at      = $schedule_date        // 计划日期作为付款日期
   $model->created_from = 'core::recurring'
    ↓
4. $model->save()         ← ★ 主记录落库 (transactions 表)
    ↓
5. updateRelationTypes() → 更新 taxes 关联记录的 type 字段
    ↓
返回 $model
```

### 5.4 updateRelationTypes() 详解

**文件**：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L274-L283)

由于 Cloneable 克隆子关联时，子记录的 `type` 字段仍然是模板类型（如 `invoice-recurring`），需要批量修正为真实类型：

```php
public function updateRelationTypes($model, $relations)
{
    foreach ($relations as $relation) {
        if (! method_exists($model, $relation)) {
            continue;
        }
        $model->$relation()->update(['type' => $model->type]);
    }
}
```

- Document 的关系：`items` (document_items)、`totals` (document_totals)
- Transaction 的关系：`taxes` (transaction_taxes)

---

## 六、落库表一览

### 6.1 recurring 表（调度计划表）

模型：[Common/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Models/Common/Recurring.php)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | PK | 主键 |
| company_id | FK | 所属公司 |
| recurable_id | FK | 多态关联ID (documents.id 或 transactions.id) |
| recurable_type | string | 多态关联类型 (Document/Transaction 类名) |
| frequency | string | 频率：daily/weekly/monthly/yearly/custom |
| interval | int | 间隔周期 |
| started_at | datetime | 开始日期 |
| status | string | active/ended/completed |
| limit_by | string | count(按次数) / date(按日期) |
| limit_count | int | 限制次数，0=无限 |
| limit_date | datetime | 限制截止日期 |
| auto_send | bool | 是否自动发送邮件 |
| created_from | string | 创建来源 |
| created_by | FK | 创建人 |

### 6.2 documents 表（单据主表）

当生成 Document 类型子单据时：

- `parent_id` = 模板 Document 的 id（建立父子关系，用于去重）
- `type` = 去掉 `-recurring` 的真实类型（invoice/bill）
- `issued_at` = 计划日期
- `due_at` = 计划日期 + 模板到期间隔
- `created_from` = `core::recurring`

同时 `document_items`、`document_totals` 的 `type` 也被同步更新。

### 6.3 transactions 表（交易主表）

当生成 Transaction 类型子单据时：

- `parent_id` = 模板 Transaction 的 id
- `type` = 去掉 `-recurring` 的真实类型（income/expense）
- `paid_at` = 计划日期
- `created_from` = `core::recurring`

同时 `transaction_taxes` 的 `type` 也被同步更新。

---

## 七、事件与通知链路

### 7.1 Event 注册

**文件**：[Event.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Providers/Event.php)

| 事件 | 监听者 |
|------|--------|
| DocumentCreated | CreateDocumentCreatedHistory, IncreaseNextDocumentNumber, SettingFieldCreated |
| **DocumentRecurring** | **SendDocumentRecurringNotification** |
| TransactionCreated | IncreaseNextTransactionNumber |
| TransactionRecurring | (无默认监听) |

### 7.2 DocumentRecurring 通知发送

**监听器**：[SendDocumentRecurringNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L19-L57)

```
handle(DocumentRecurring $event)
    ↓
① 获取配置：config('type.document.{type}.notification')
    ↓
② 若 auto_send == false → 跳过（检查父模板的 recurring->auto_send）
    ↓
③ 通知客户：
   $document->contact->notify(new Notification($document, "type_recur_customer", attach_pdf=true))
    ↓
④ 触发 DocumentSent 事件 → MarkDocumentSent (将 status 设为 sent)
    ↓
⑤ 通知公司内有 read-notifications 权限的所有用户：
   $user->notify(new Notification($document, "type_recur_admin"))
```

关键点：生成的 recurring 子单据会自动被标记为 **已发送 (sent)** 状态。

---

## 八、完整流程时序图（以 Invoice 为例）

```
[Cron 每日触发]
    │
    ▼
artisan recurring:check
    │
    ▼
RecurringCheck::handle()
    │
    ├─► Recurring::active()->allCompanies()->cursor()   [取所有 active 的 recurring 记录]
    │
    ├─► 循环每条 $recur {
    │       │
    │       ├─► 公司有效性检查
    │       │
    │       ├─► company($id)->makeCurrent()
    │       │
    │       ├─► $template = $recur->recurable (Document, type="invoice-recurring")
    │       │
    │       ├─► getRemainingSchedules():
    │       │     │
    │       │     ├─► 查 documents 表，parent_id = 模板id 的所有 issued_at
    │       │     │
    │       │     └─► getRecurringSchedule() → 用 Recurr 库生成完整日程 → 过滤掉已生成
    │       │
    │       ├─► endsBefore(明天) 过滤
    │       │
    │       └─► 循环每个 $schedule {
    │               │
    │               ▼
    │           recur($template, $schedule_date)
    │               │
    │               ├─► DB::transaction {
    │               │     │
    │               │     ├─► getDocumentModel():
    │               │     │     │
    │               │     │     ├─► $template->duplicate()  [Cloneable 克隆 documents + items + totals]
    │               │     │     │
    │               │     │     ├─► 修改 type=invoice, parent_id=模板id, issued_at=$schedule_date
    │               │     │     │
    │               │     │     ├─► $model->save()        ★ 落库 documents 表
    │               │     │     │
    │               │     │     └─► updateRelationTypes()  ★ 更新 items/totals 的 type=invoice
    │               │     │
    │               │     ├─► event(DocumentCreated)
    │               │     │     ├─► CreateDocumentCreatedHistory
    │               │     │     ├─► IncreaseNextDocumentNumber
    │               │     │     └─► SettingFieldCreated
    │               │     │
    │               │     └─► event(DocumentRecurring)
    │               │           └─► SendDocumentRecurringNotification
    │               │                 ├─► 通知客户 (带PDF附件)
    │               │                 ├─► event(DocumentSent) → status=sent
    │               │                 └─► 通知公司管理员
    │               │  }
    │            }
    │        }
    │
    ▼
Company::forgetCurrent()
    │
    ▼
[完成]
```

---

## 九、关键设计要点

1. **类型双轨制**：模板用 `*-recurring` type，生成的子单据用普通 type。通过 scopeIsRecurring 进行区分。

2. **父子关联**：子单据的 `parent_id` 指向模板 ID，这是防止重复生成和追溯的唯一依据。

3. **日期去重**：每次调度时先查已生成子单据的日期数组，再从完整日程中过滤，不依赖次数计数。

4. **多公司上下文**：遍历前通过 `company()->makeCurrent()` 切换公司，确保多租户环境下正确使用配置（如时区、文档编号规则）。

5. **原子性**：每个 schedule 日期独立事务，单个失败不影响后续日期继续生成。

6. **克隆 + 修正**：使用 Cloneable 快速复制主记录及关联，再批量修正 type 字段。

7. **状态流转**：Recurring 记录初始为 `active`，全部日程生成完毕后自动更新为 `completed`。
