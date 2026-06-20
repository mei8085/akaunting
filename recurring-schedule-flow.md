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

#### 3.3.1 frequency 与 interval 的 custom 分支存值规则（★修正）

```php
// frequency 规则
$frequency = ($request['recurring_frequency'] != 'custom')
    ? $request['recurring_frequency']
    : $request['recurring_custom_frequency'];

// interval 规则
$interval = (($request['recurring_frequency'] != 'custom') || ($request['recurring_interval'] < 1))
    ? 1
    : (int) $request['recurring_interval'];
```

| recurring_frequency 取值 | frequency 存值来源 | interval 取值 |
|---------------------------|-------------------|---------------|
| `no` | — (不创建 recurring) | — |
| `daily` / `weekly` / `monthly` / `yearly` | 直接取 `recurring_frequency` (原值) | **强制=1**（忽略用户输入） |
| `custom` | 取 `recurring_custom_frequency` (用户自定义频率字符串) | `recurring_interval >= 1` 时用用户值，否则兜底=1 |

> **关键理解修正**：interval 字段**仅在 custom 分支下才允许用户自定义间隔数**。非 custom 的 daily/weekly/monthly/yearly 四种内置频率无论用户填什么 interval，数据库里一律存 `1`。即内置频率本身语义就是"每隔 1 天/周/月/年"。

#### 3.3.2 created_from 与 created_by 的落库规则（★补充）

```php
$source = !empty($request['created_from']) ? $request['created_from'] : source_name();
$owner  = !empty($request['created_by'])   ? $request['created_by']   : user_id();
```

| 字段 | 优先级1 (请求传入) | 优先级2 (兜底) | 说明 |
|------|--------------------|----------------|------|
| `created_from` | `$request['created_from']` | `source_name()` | [helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Utilities/helpers.php#L150-L155) `source_name()` 内部使用 `Traits\Sources::getSourceName()`，根据请求上下文返回 `web` / `api` / `cli` 等来源标识 |
| `created_by` | `$request['created_by']` | `user_id()` | [helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Utilities/helpers.php#L30-L33) `user_id()` = `user()?->id`，即当前登录用户ID（CLI/无用户上下文时为 null） |

> 注意：[updateRecurring()](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L45-L91) 中，`update` 分支**不修改** created_from/created_by；只有 `create` 分支（原 recurring 不存在需新建时）才会写入。

#### 3.3.3 完整字段说明表

| 字段 | 说明 | 来源 |
|------|------|------|
| `company_id` | 公司ID | $this->company_id |
| `frequency` | 频率字符串 | 见 3.3.1 custom 分支规则 |
| `interval` | 间隔周期 | 见 3.3.1 custom 分支规则（非custom恒为1） |
| `started_at` | 开始日期 | recurring_started_at，默认今天 |
| `status` | 状态 | recurring_status，默认 active |
| `limit_by` | 限制方式 (count/date) | recurring_limit，默认 count |
| `limit_count` | 限制次数 | recurring_limit_count，默认 0 (无限) |
| `limit_date` | 限制截止日期 | recurring_limit_date |
| `auto_send` | 是否自动发送邮件 | recurring_send_email，默认 0 |
| `created_from` | 创建来源 | 见 3.3.2 request → source_name() 兜底 |
| `created_by` | 创建人 | 见 3.3.2 request → user_id() 兜底 |

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

#### 4.3.1 日期去重 + 缺行隐式重试机制（★补充）

源码第 126 行的注释已经点明设计意图：
```php
// Get the remaining schedules, including the previously failed ones
$schedules = $this->getRemainingSchedules($template, $recur);
```

**去重的实现本质**：不是用"已生成次数计数器"或"指针位"，而是**每次调度时实时查询子表中 parent_id = 模板id 的所有日期，再从完整日程集合中做集合差集**。

**隐式重试的推导**：

```
场景：某月的 3 日、4 日两个 schedule 到期，
      3日生成成功（documents表有一条 parent_id=X, issued_at=3日 的行），
      4日生成时 getDocumentModel 抛出异常被 catch，事务回滚，
      即 documents 表中没有 issued_at=4日 的子记录。

下一次调度执行时：
    已生成日期数组 = [ "3日" ]
    完整日程仍包含 [ "3日", "4日", ... ]
    filter 后 剩余日程 = [ "4日", ... ]
    → 4日被自动"补单"，无需人工干预
```

> **设计优点**：无需持久化"失败队列"或"重试状态"。利用"日期集合差集"的天然幂等性，任何中断（异常、进程被杀、超时）都会在下次调度时自动补完所有"缺行"日期。代价：每次调度都要查子表日期，O(n) 去重。

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
   - setTimezone(timezone)     → ★ 公司时区
   - setFreq(frequency)        → WEEKLY/MONTHLY/...
   - setInterval(interval)     → 间隔数
   - 若 limit_by=date → setUntil(limit_date)
   - 若 limit_by=count 且 limit_count>0 → setCount(limit_count)
    ↓
③ ArrayTransformer::transform($rule)  → RecurrenceCollection
```

#### 4.4.1 setTimezone 调用的作用（★补充）

**Trait**：[Traits/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L105-L151)

```php
// 构建 Rule 时显式设置时区
$rule = (new Rule())
    ->setStartDate($this->getRecurringRuleStartDate())
    ->setTimezone($this->getRecurringRuleTimeZone())  // ← 关键调用
    ->setFreq(...)
    ->setInterval(...);

// 而 StartDate/UntilDate 构造时也带着时区创建
public function getRecurringRuleDate($date)
{
    return new \DateTime($date, new \DateTimeZone($this->getRecurringRuleTimeZone()));
}

// 时区来源：公司级本地化配置（非 app.timezone）
public function getRecurringRuleTimeZone()
{
    return setting('localisation.timezone');
}
```

**为什么两处都设置时区？**
- `new \DateTime($date, $tz)` 保证输入的 started_at / limit_date 字符串**按公司时区解释**（例：输入"2026-06-21"在 Asia/Shanghai 时区下解析为东八区零点）
- `$rule->setTimezone($tz)` 告诉 Recurr 库在**内部做日期加法运算时**使用该时区（关键：处理 DST 夏令时跳变、跨月边界计算）

**不设置的风险**：如果服务器默认时区（UTC）与公司时区（如 Asia/Shanghai）相差 8 小时，可能出现"1月31日 加1月"的边界运算被错位，导致生成的日期偏移 1 天。

#### 4.4.2 VirtualLimit 的兜底动机（★补充）

**Trait**：[Traits/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L168-L187)

| frequency | VirtualLimit | 对应时间跨度 |
|-----------|-------------|-------------|
| `yearly` | `2` | ≈ 2 年 |
| `monthly` | `24` | ≈ 2 年 |
| `weekly` | `104` | ≈ 2 年（104/52=2） |
| `daily` / 其他 | `732` | ≈ 2 年（732/366≈2） |

**兜底动机**：
- 当 `limit_count = 0`（代表"无限次"）或 `limit_date` 设置得极远（如 2099年），Recurr 库若不加限制会尝试一次性展开数千乃至数万条 Recurrence 对象，导致**内存溢出 / PHP 执行超时**。
- VirtualLimit 提供了一个**硬上界软约束**：无论业务上设置多远，每次最多向前展开"约2年"的日程集合。
- 配合每日调度：2年跨度足够覆盖所有"历史欠账补单"场景（实际只需补到今天），同时后续日期随着每日调度逐批展开即可，不是一次性展开到无穷。

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
| frequency | string | 频率：daily/weekly/monthly/yearly + custom分支自定义字符串 |
| interval | int | 间隔周期（非custom恒为1，custom分支才会>1） |
| started_at | datetime | 开始日期 |
| status | string | active/ended/completed |
| limit_by | string | count(按次数) / date(按日期) |
| limit_count | int | 限制次数，0=无限 |
| limit_date | datetime | 限制截止日期 |
| auto_send | bool | 是否自动发送邮件（控制通知 gate2） |
| created_from | string | 创建来源（request[created_from] 或 source_name() 兜底） |
| created_by | FK | 创建人（request[created_by] 或 user_id() 兜底） |

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

### 7.2 DocumentRecurring 通知发送：三层 Gate + 配置驱动（★大幅修正补充）

**监听器**：[SendDocumentRecurringNotification.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L19-L57)

```
handle(DocumentRecurring $event)
    │
    ▼
┌─ Gate 1 ─ 配置层 ─────────────────────────────────────────┐
│ $config = config('type.document.{type}.notification')     │
│ if (empty($config) || empty($config['class'])) return;    │
│ 含义：该单据类型(如invoice/bill)未配置通知类则直接跳过     │
└───────────────────────────────────────────────────────────┘
    │
    ▼
┌─ Gate 2 ─ 业务开关层 ─────────────────────────────────────────┐
│ if ($document->parent?->recurring?->auto_send == false) return;│
│ 含义：用户在 recurring 模板上关闭了 auto_send 开关则跳过       │
│ 注意：用 nullsafe ?-> 操作符，parent/recurring 缺失时不报错    │
└───────────────────────────────────────────────────────────────┘
    │
    ├──┐
    │  │
    │  ▼
    │ ┌─ Gate 3a ─ 客户可达性(内部4层) ───────────────────────┐
    │ │ $this->canNotifyTheContactOfDocument($document)      │
    │ │   内部判断:                                            │
    │ │   ① config.notify_contact == true ?                   │
    │ │   ② contact 存在 && contact->enabled == 1 ?           │
    │ │   ③ !empty(contact_email) ?                           │
    │ │   ④ EmailValidator(RFC + DNS MX 记录校验) 通过 ?       │
    │ │ 以上全部满足才发送客户通知                              │
    │ └───────────────────────────────────────────────────────┘
    │      │
    │      ▼ 全部通过
    │   通知客户（带PDF附件）：
    │   $contact->notify(new InvoiceNotification, "invoice_recur_customer", true)
    │
    ▼
┌─ ★ sent 事件触发点（配置驱动）────────────────────────────────┐
│ $sent = config('type.document.{type}.auto_send',              │
│               DocumentSent::class);                            │
│ event(new $sent($document));                                   │
│ 含义：用哪个事件类来标记"已发送/已收到"状态完全由配置决定       │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
┌─ Gate 3b ─ 用户通知总开关 ───────────────────────────────────┐
│ if (!$config['notify_user']) return;                          │
│ 含义：配置里关闭了 notify_user 则在此处 return                 │
│ ★注意：sent 事件已经触发了，单据状态已改为 sent/received      │
└───────────────────────────────────────────────────────────────┘
    │
    ▼
遍历公司所有用户：
    │
    ▼
┌─ Gate 3c ─ 用户权限层(循环内continue) ─────────────────────┐
│ if ($user->cannot('read-notifications')) continue;          │
└─────────────────────────────────────────────────────────────┘
    │
    ▼ 通过
通知用户（无PDF）：
$user->notify(new InvoiceNotification, "invoice_recur_admin")
```

#### 7.2.1 三层 Gate 汇总

| Gate 编号 | 层级 | 检查内容 | 失败动作 | 代码位置 |
|-----------|------|---------|---------|---------|
| Gate 1 | 配置层 | 该 type 的 notification 配置是否存在 class | `return` | Line 24-26 |
| Gate 2 | 业务层 | 模板 recurring.auto_send 是否开启 | `return` | Line 28-30 |
| Gate 3a | 客户可达性 | notify_contact开关+contact启用+有邮箱+邮箱合法（RFC+DNS） | 不通知客户，**继续往下走** | 方法内 return false |
| Gate 3b | 用户通知开关 | config.notify_user == true | `return` | Line 45-47 |
| Gate 3c | 用户权限 | 用户有 `read-notifications` 权限 | `continue` 单用户 | Line 51-53 |

> 关键点：Gate 3a 失败只是不通知客户，但 sent 事件和用户通知逻辑仍会继续执行（除非 Gate 3b 总开关关闭）。

#### 7.2.2 配置驱动的 sent 事件类（★补充）

**配置文件**：[type.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/config/type.php)

监听器代码：
```php
$sent = config('type.document.' . $document->type . '.auto_send', DocumentSent::class);
event(new $sent($document));
```

| 子单据 type | 配置里的 auto_send 值 | 对应事件类 | 触发的监听者动作 |
|------------|----------------------|-----------|----------------|
| `invoice` | `App\Events\Document\DocumentSent` | `DocumentSent` | `MarkDocumentSent` → status = `sent` |
| `bill` | `App\Events\Document\DocumentReceived` | `DocumentReceived` | `MarkDocumentReceived` → status = `received` |

> **设计含义**：发票(invoice)是"我方发给客户"→ 状态流转为 `sent`；账单(bill)是"我方从供应商收到"→ 状态流转为 `received`。两种单据的业务语义相反，因此触发不同的 sent/received 事件，由配置决定而非硬编码。

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
    │       │     │   (日期差集去重 + 缺行隐式重试)
    │       │     │
    │       │     └─► getRecurringSchedule()
    │       │           ├─ setTimezone(公司本地化时区)
    │       │           ├─ setVirtualLimit(~2年软上限)
    │       │           └─ Recurr 展开 → 过滤已生成日期
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
    │               │                 ├─► Gate1(配置存在?)
    │               │                 ├─► Gate2(auto_send开?)
    │               │                 ├─► Gate3a(客户可达?4条件)→通知客户
    │               │                 ├─► ★event(config(DocumentSent/DocumentReceived))
    │               │                 ├─► Gate3b(notify_user开关?)
    │               │                 └─► Gate3c(用户权限?)→通知管理员
    │               │  }  // 事务失败则该日期缺行，下次自动补
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

3. **日期差集去重 + 隐式重试**：每次调度实时查询子表日期做集合差集，天然实现"失败自动补单"，无需单独持久化重试队列。（代码注释明确写了 including the previously failed ones）

4. **custom 频率双字段**：内置频率(daily~yearly)的 interval 强制为 1，只有 custom 分支才允许自定义 interval + recurring_custom_frequency 字符串组合。

5. **created_from/created_by 双级兜底**：先从 request 取，缺失时分别用 source_name()（识别 web/api/cli）和 user_id()（当前登录用户ID）兜底，updateRecurring 时不覆写。

6. **多公司上下文 + 公司时区**：遍历前通过 `company()->makeCurrent()` 切换公司；Recurr Rule 的 `setTimezone()` + DateTime 构造都使用 `setting('localisation.timezone')`，与 app.timezone 解耦。

7. **VirtualLimit ≈ 2年软上限**：对无限/远期的 recurring 设置约2年的虚拟展开上限，防止 Recurr 一次性展开无穷多日期导致内存溢出。

8. **三层通知 Gate + 配置驱动状态事件**：配置存在→auto_send→(客户可达性 / sent事件 / 用户通知开关 / 用户权限) 逐级短路。sent 事件类由 type 配置决定（invoice→DocumentSent，bill→DocumentReceived），实现业务语义差异。

9. **原子性 + 失败隔离**：每个 schedule 日期独立事务，getModel() 内 try/catch 捕获异常返回 false，单个日期失败不影响后续日期及其他 recurring。

10. **克隆 + 修正两步走**：使用 Cloneable 快速复制主记录及关联(去掉recurring以免模板关联链被复制)，再通过 updateRelationTypes 批量更新子关联的 type 字段与主记录对齐。
