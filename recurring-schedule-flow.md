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

文件：[Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Kernel.php#L23-L59)

```php
protected function schedule(Schedule $schedule)
{
    $schedule_time = config('app.schedule_time');

    $schedule->command('recurring:check')
        ->dailyAt($schedule_time)
        ->runInBackground();
}

// 调度时区由 scheduleTimezone() 决定：
protected function scheduleTimezone()
{
    return config('app.timezone');
}
```

- **执行频率**：每天一次，时间由 `app.schedule_time` 配置决定
- **运行模式**：后台执行 (`runInBackground()`)
- **命令签名**：`recurring:check`

#### 2.1.1 scheduleTimezone 与 Recurr 公司时区的双时区错位（★补充）

系统存在两层时区语义不同的"时区"：

| 层 | 时区来源 | 作用域 | 影响范围 |
|----|---------|--------|---------|
| **Laravel Scheduler** | `config('app.timezone')`（服务器默认时区，通常 UTC） | 全局单例 | 决定 `dailyAt()` 在几点触发命令 —— **所有公司共享同一个触发时刻** |
| **Recurr 日程引擎** | `setting('localisation.timezone')`（公司本地化配置） | 按公司切换 | 决定 `setTimezone()` 如何解释 started_at / 做 RRULE 日期加法 —— **每家公司独立时区** |

**错位场景**：

```
服务器 app.timezone = UTC
公司A localisation.timezone = Asia/Shanghai (UTC+8)
公司B localisation.timezone = Europe/London (UTC+0)
app.schedule_time = "00:00"

每天 UTC 00:00（北京时间 08:00，伦敦时间 00:00）命令被唤醒，
同时遍历所有公司的 recurring，
每家公司内部用自己的 localisation.timezone 做 RRULE 计算。

问题：对于 UTC+8 公司，"今天"的定义是北京时间 00:00~23:59，
      但调度触发时北京时间是 08:00，"今天 00:00 的那条 schedule"
      在 RecurringCheck 里的 endsBefore(明天) 仍然包含，能被生成，
      不会错过，但语义上不是在"该公司当天凌晨"生成。
```

> 设计权衡：多租户共享一个 cron 触发点是常见取舍；真正精细的"按公司时区在该公司本地 00:00 触发"需要分公司调度队列，复杂度高得多。这里通过 Recurr 层按公司时区计算，**保证日期不会错生成**，只是触发时刻有偏移。

### 2.2 命令类

文件：[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php)

核心方法 `handle()` 是整个调度流程的入口。

---

## 三、创建 Recurring 模板（前置准备）

在调度执行之前，用户需先创建一条 **Recurring 模板**。模板就是带有 `-recurring` 后缀 type 的 Document 或 Transaction 记录。

### 3.1 Document 模板创建路径

**Job**：[CreateDocument.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Jobs/Document/CreateDocument.php#L20-L68)

```
handle() 入口
    │
    ├─► 【子步骤 1】authorize() ← 套餐配额鉴权（事务外）
    │     └─ getAnyActionLimitOfPlan()
    │        仅对 type=invoice 检查；超限则 throw Exception 直接终止
    │
    ├─► 【子步骤 2】amount 空值兜底（事务外）
    │     └─ empty(amount) → amount = 0
    │
    ├─► 【子步骤 3】discount 字段兼容（事务外）
    │     └─ !empty(discount) → discount_rate = discount
    │        （历史兼容 issue #2797，老字段名 discount → 新字段名 discount_rate）
    │
    ├─► event(DocumentCreating)  ← 前置事件
    │
    ▼
┌─ DB::transaction ────────────────────────────────────────────┐
│                                                              │
│  ┌─ 【子步骤 4】attachment 媒体上传 ───────────────────────┐  │
│  │ Document::create(...)  → 主记录落库                     │  │
│  │   ↓                                                     │  │
│  │ request 有 attachment 文件 → 遍历上传：                  │  │
│  │   getMedia($file, 'invoices')  ← Uploads::getMedia()    │  │
│  │     → 路径: YYYY/MM/DD/{company_id}/{type}s/            │  │
│  │   → $model->attachMedia($media, 'attachment')           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                              │
│      ↓                                                        │
│  dispatch(new CreateDocumentItemsAndTotals)  ← 明细行+总计行  │
│      ↓                                                        │
│  $model->update($request->all())                              │
│      ↓                                                        │
│  $model->createRecurring($request->all())  ← ★ 生成 recurring│
│                                                              │
└──────────────────────────────────────────────────────────────┘
    ↓
event(DocumentCreated)
```

**四个子步骤详解（★补充）：**

| 序号 | 子步骤 | 位置 | 核心逻辑 | 代码行 |
|------|--------|------|----------|--------|
| 1 | **authorize 鉴权** | 事务外，最前置 | `getAnyActionLimitOfPlan()` 取套餐配额；仅对 `type=invoice` 生效；`action_status=false` 则抛出异常直接中止创建 | 第 22/62-68 行 |
| 2 | **amount 空值兜底** | 事务外 | `empty($this->request['amount'])` → 强制设为 `0`，防止空值入库 | 第 24-26 行 |
| 3 | **discount 字段兼容** | 事务外 | 若请求传了 `discount`（老字段），同步赋值给 `discount_rate`（新字段）；注释明确写了是 issue #2797 全局折扣问题修复后保留的兼容 | 第 28-31 行 |
| 4 | **attachment 媒体上传** | 事务内，主记录创建后 | 遍历 `$request->file('attachment')` 数组；`getMedia()` 调 `MediaUploader` 上传到 `YYYY/MM/DD/{company_id}/invoices/` 目录；`attachMedia()` 建立媒体-单据关联 | 第 39-45 行 |

> 注：前 3 个在事务外执行，失败不涉及回滚；第 4 个在事务内，上传成功后若后续步骤失败，事务回滚但**已上传的物理文件不会自动删除**（MediaUploader 写磁盘不在 DB 事务内）。

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

### 3.4 updateRecurring()：frequency=no 删除分支 + update/create 双写入路径（★补充）

**Trait**：[Traits/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L45-L91)

updateRecurring 与 createRecurring 结构不同，它有 **三条分支**：

```
updateRecurring($request)
    │
    ├─► 【分支 1】frequency=no 或 empty → 删除分支
    │     └─ if (empty($request['recurring_frequency']) || ($request['recurring_frequency'] == 'no')
    │           $this->recurring()->delete()
    │           return;
    │
    │ 含义：用户把原来的 recurring 模板改为"不重复"了，相当于解除 recurring 关联
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ frequency != no（走下面的 update 主流程（正常 recurring）            │
│                                                          │
│   ① 计算 frequency / interval / started_at 等字段          │
│   （与 createRecurring 完全相同的逻辑                          │
│                                                          │
│   ② $recurring = $this->recurring()                        │
│      $model_exists = $recurring->count()                    │
│                                                          │
│   ├─► 【分支 2】$model_exists = true → update 分支            │
│   │     $recurring->update($data)                       │
│   │       不写 created_from / created_by                  │
│   │       recurring_status 只在 $request 有值时才写
│   │
│   └─► 【分支 3】$model_exists = false → create 分支           │
│         $recurring->create(array_merge($data, [               │
│             'status'       => ACTIVE_STATUS,                       │
│             'created_from' => source 兜底,                        │
│             'created_by'  => owner 兜底,                          │
│         ])                                                  │
│         含义：模型本身是普通单据（原无 recurring 关联，用户在编辑时                 │
│         时第一次勾选 recurring                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3.4.1 createRecurring vs updateRecurring 对比

| 维度 | createRecurring | updateRecurring |
|------|---------------|----------------|
| **frequency=no/empty | `return;`（静默不操作 | **`$this->recurring()->delete()`（删除已有） |
| **已有关联记录时 | `create（必须不存在才调用） | `update`（已存在则更新） |
| **无关联记录时 | create（正常创建） | create（补创建，默认 status=active） |
| **created_from/by** | 必写入（总是写） | update分支**不写**；create分支**写入 |
| **status 写入** | 从 recurring_status 默认 ACTIVE_STATUS | update分支有 request 有值才写；create分支强制 ACTIVE_STATUS |

> 典型触发路径：用户在编辑账单详情页点击 End 按钮，Controller 调 UpdateDocument/UpdateTransaction Job，Job 内部调用 `$model->updateRecurring($request)`，传 `recurring_status = END_STATUS`，走 update 分支的 status 写入路径（因为 request 有 recurring_status）。

### 3.5 模板识别：isRecurring 作用域

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
② Recurring::with('company')
        ->active()
        ->allCompanies()        ← ★ 禁用 Company 全局 scope，取所有公司
        ->cursor()
    ↓
③ 逐条遍历 $recur {
    3.1 【孤儿清理1】公司不存在 → $recur->delete() + continue
    ↓
    3.2 取模板 $template = $recur->recurable()
                                    ->where('company_id', $recur->company_id)
                                    ->first()
    ↓
    3.3 【孤儿清理2】公司禁用 超3个月 → $recur->delete() + $template->delete() + continue
    ↓
    3.4 【孤儿清理3】无活跃用户 超3个月 → $recur->delete() + $template->delete() + continue
    ↓
    3.5 company($recur->company_id)->makeCurrent()  ← ★ 切换公司上下文
    ↓
    3.6 【孤儿清理4】模板已被删($template=null) → $recur->delete() + continue
    ↓
    3.7 getRemainingSchedules()  ← 计算剩余待生成的日程
    ↓
    3.8 若无剩余日程 → status = completed，跳过
    ↓
    3.9 endsBefore(明天) 过滤 → 只取今天及之前到期的
    ↓
    3.10 逐条遍历 $schedules {
        recur($template, $schedule_date)  ← ★ 生成单据
    }
}
    ↓
④ Company::forgetCurrent() + 清除容器实例，结束
```

#### 4.1.1 allCompanies 禁用 Company 全局 scope 与 makeCurrent 切上下文（★补充）

**allCompanies 的实现**：
[Abstracts/Model.php#L77-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Abstracts/Model.php#L77-L80)
```php
public function scopeAllCompanies($query)
{
    return $query->withoutGlobalScope('App\Scopes\Company');
}
```

**Company 全局 scope 的作用**：
[Scopes/Company.php#L21-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Scopes/Company.php#L21-L35)

所有继承了 `Abstracts\Model` 且使用了 `Tenants` trait 的模型，**每一次查询**都会被自动附加 `WHERE company_id = company_id()` 条件，实现多租户隔离。

| 阶段 | 是否有 Company scope | company_id 值 |
|------|---------------------|---------------|
| **初始状态**（CLI 启动） | 有（自动附加） | `null` → 可能查不到任何数据，或查到当前默认公司 |
| **allCompanies() 之后**（查 recurring 列表） | **被移除** | 不限制 → 查出 **所有公司** 的 active recurring |
| **makeCurrent() 之后**（处理某条 recurring） | 有（自动附加） | 切换为 `$recur->company_id` → 后续所有查询（模板、子单据、setting 配置）只看到该公司 |
| **forgetCurrent() 之后**（遍历结束） | 有（自动附加） | 回到 null / 默认值 |

**为什么要两步走？**
1. `allCompanies()` 是**查询层面**去租户限制 → 保证能遍历到每个公司的 recurring 记录
2. `makeCurrent()` 是**上下文层面**切租户 → 保证处理单条 recurring 时：
   - `setting('localisation.timezone')` 返回该公司的时区
   - 新建的 Document/Transaction 自动写入正确的 company_id
   - Eloquent 查询默认带上该公司的过滤，不会误读写其他公司数据

#### 4.1.2 四条孤儿清理路径（★补充）

handle() 主循环中内置 4 个垃圾回收分支，防止脏数据（recurring 还在但关联对象已被删）长期占用调度资源：

| 编号 | 检查点 | 代码位置 | 判断条件 | 清理动作 | 连带清理模板 |
|------|--------|---------|---------|---------|------------|
| **1** | 公司不存在 | [L62-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L62-L68) | `empty($recur->company)` | `$recur->delete()` | 否（无法取到） |
| **2** | 公司禁用超 3 个月 | [L77-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L77-L89) | `!$company->enabled && updated_at < now-3month` | `$recur->delete()` | 是 `$template->delete()` |
| **3** | 无活跃用户超 3 个月 | [L92-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L92-L112) | 所有用户 `last_logged_in_at < now-3month` | `$recur->delete()` | 是 `$template->delete()` |
| **4** | 关联模板已被删 | [L116-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L116-L122) | `$template === null`（recurable 多态关联找不到对应记录） | `$recur->delete()` | 否（已不存在） |

> 注意：检查点 1-3 在 **makeCurrent() 之前**执行，检查点 4 在 makeCurrent() 之后执行。因为检查点 4 需要先切到正确公司上下文，才能走 Company scope 查到对应的模板（多态关联的 recurable 表也受 Company scope 约束）。

### 4.2 计算剩余日程：getRemainingSchedules()

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

### 4.3 日程生成引擎：getRecurringSchedule()

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

#### 4.3.1 setTimezone 调用的作用（★补充）

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

#### 4.3.2 VirtualLimit 的兜底动机（★补充）

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

### 5.1 getModel() 分发 + 异常出口（★补充）

**源码位置**：[RecurringCheck.php#L190-L203](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L190-L203)

```
getModel(Document|Transaction $template, Date $schedule_date): Document|Transaction
    │
    ▼
判断模板类型：
    Transaction → $this->getTransactionModel($template, $schedule_date)
    Document    → $this->getDocumentModel($template, $schedule_date)
    │
    ▼
try { 调用具体方法 }
    │
    ├── 成功 → 返回 Document 或 Transaction 实例
    │
    └── 失败 → catch(\Throwable $e) { report($e); return false; }
                └─ ★ 源码缺陷：声明只返回 Document|Transaction，实际 return false
                   在 strict_types=1 下会抛 TypeError，非严格模式 PHP 隐式转换
```

**返回类型与异常出口要点：**

| 项 | 说明 |
|----|------|
| **声明返回类型** | `Document\|Transaction`（源码 [L190](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L190)，**不是**三联合） |
| **正常返回** | `Document` 或 `Transaction` Eloquent 模型实例 |
| **异常出口** | `catch (\Throwable $e) { report($e); return false; }` —— **任何 Throwable 都被吞掉**（比 \Exception 范围更大，含 Error/TypeError 等），写日志后返回 `false` |
| **源码类型违例** | 声明返回 `Document\|Transaction`，实际 `return false` —— PHP 非 strict_types 模式下会隐式转换不报错，但严格模式下触发 `TypeError: Return value must be of type Document\|Transaction, bool returned` |
| **调用方的短路处理** | `recur()` 里 `if (! $model = $this->getModel(...)) { return; }` —— 用 false 做松散比较直接跳过该 schedule |
| **设计意图** | 失败隔离：某一个 schedule 日期生成失败（如 Cloneable 克隆异常、save 失败、关联更新异常），不影响其他日期和其他 recurring 继续执行 |

> **隐患**：`false` 与 null/0/空字符串 在 PHP 松散比较下都为 falsy。这里依赖 `$model = ...` 赋值后再 `!` 取反，只要返回 false 就跳过，语义是清晰的；但若有一天 getDocumentModel 意外返回 null 或 0，也会被"静默跳过"。

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

### 6.2 Recurring 状态机（★补充）

**模型常量定义**：[Common/Recurring.php#L13-L15](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Models/Common/Recurring.php#L13-L15)

```php
public const ACTIVE_STATUS   = 'active';
public const END_STATUS      = 'ended';
public const COMPLETE_STATUS = 'completed';
```

#### 6.2.1 三种状态语义

| 状态 | 英文常量 | 含义 |
|------|---------|------|
| **active** | ACTIVE_STATUS | 激活中，正常按计划生成单据 |
| **ended** | END_STATUS | 用户手动终止（提前结束） |
| **completed** | COMPLETE_STATUS | 全部到期/生成完毕（自然终结） |

> 区别：`ended` 是人为干预的"提前结束"，`completed` 是全部 schedule 都生成完后的"自然结束"。两者都是终态，不会再生成新单据。

#### 6.2.2 两条状态流转路径

```
                    ┌──────────────────┐
                    │      active      │  ← 初始状态（createRecurring 时默认）
                    └──────────────────┘
                          /       \
                         /         \
    【路径A】手动 end     /           \    【路径B】自动 complete
   用户点击"End"按钮    /             \   所有日程生成完毕
                       /               \
                      ▼                 ▼
            ┌──────────────┐    ┌───────────────┐
            │    ended     │    │   completed   │
            │ (手动终止)   │    │  (自然结束)    │
            └──────────────┘    └───────────────┘
                       \             /
                        \           /
                         \         /
                          终态，无回退
```

**路径 A：active → ended（手动终止）**

| 维度 | 说明 |
|------|------|
| **触发入口** | 三个 Controller 的 `end()` 方法 |
| **Controller 文件** | [RecurringInvoices.php#L209-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L209-L226)、[RecurringBills.php#L209-L224](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Http/Controllers/Purchases/RecurringBills.php#L209-L224)、[RecurringTransactions.php#L245-L262](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Http/Controllers/Banking/RecurringTransactions.php#L245-L262) |
| **执行路径** | Controller 调用 `ajaxDispatch(new UpdateDocument/UpdateTransaction(...))`，传入 `recurring_status = Recurring::END_STATUS` → Update Job 内部调用 `$model->updateRecurring()` → 走 [Traits/Recurring.php](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Traits/Recurring.php#L45-L91) 的 update 分支 `$this->recurring()->update(...)` |
| **关键参数** | 除了 recurring_status=ended，还会原样传入当前的 frequency、started_at、limit 等字段（避免 update 时被覆盖丢失） |
| **UI 显隐逻辑** | 模板列表的操作按钮里，只有当 `recurring->status != 'ended'` 时才显示 "End" 按钮（[Document.php#L750](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Models/Document/Document.php#L750) / [Transaction.php#L724](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Models/Banking/Transaction.php#L724)） |

**路径 B：active → completed（自动迁移）**

| 维度 | 说明 |
|------|------|
| **触发入口** | `RecurringCheck::handle()` 调度命令内部 |
| **代码位置** | [RecurringCheck.php#L130-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/62-akaunting/app/Console/Commands/RecurringCheck.php#L130-L135) |
| **触发条件** | `getRemainingSchedules()` 返回的 `$schedules->count() == 0`（所有应生成的日程都已落库，无剩余） |
| **执行操作** | `$recur->update(['status' => Recurring::COMPLETE_STATUS])` |
| **后续行为** | `continue` 跳过后续 schedule 循环（本来就为空），处理下一条 recurring |
| **调度匹配范围** | 只有 `scopeActive()` 的记录会被 `Recurring::active()` 查询到；状态变为 completed 后，下次调度不再进入遍历 |

> 注意：`Recurring::active()` 是 scope，只查 status = active 的记录。所以 ended 和 completed 的 recurring 都不会再被调度处理。

#### 6.2.3 状态机与调度查询的关系

调度命令起始处用 `Recurring::active()->allCompanies()->cursor()` 取数，意味着：
- **只有 active 状态的 recurring 才会被调度处理**
- ended 状态：用户手动终止 → 不再调度（用户主动行为）
- completed 状态：全部生成完 → 不再调度（自然完成行为）

### 6.3 documents 表（单据主表）

当生成 Document 类型子单据时：

- `parent_id` = 模板 Document 的 id（建立父子关系，用于去重）
- `type` = 去掉 `-recurring` 的真实类型（invoice/bill）
- `issued_at` = 计划日期
- `due_at` = 计划日期 + 模板到期间隔
- `created_from` = `core::recurring`

同时 `document_items`、`document_totals` 的 `type` 也被同步更新。

### 6.4 transactions 表（交易主表）

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

11. **getModel 类型违例 + 静默异常出口**：源码声明返回 `Document|Transaction`（非三联合），但 catch `\Throwable` 里实际 `return false`，是一个类型违例（strict_types 下 TypeError）。调用方用 `!$model` 松散比较短路跳过，所有异常都被吞掉，只能靠日志发现。

12. **三状态机 + 双流转路径**：active(运行中)、ended(手动终止)、completed(自然完成) 三态。ended 由用户点击 End 按钮经 Update Job 的 updateRecurring 写入；completed 由调度器检测剩余日程为 0 时自动迁移。两态都是终态，不再进入 active scope 的调度遍历。

13. **CreateDocument 四步前置流水线**：事务外依次执行 authorize(套餐配额校验仅 invoice) → amount 空值兜底为 0 → discount 老字段兼容为 discount_rate → DocumentCreating 事件；事务内第一步是 Document::create 紧接着 attachment 媒体上传(磁盘IO不在DB事务内，回滚不删文件)。

14. **scheduleTimezone/Recurr 双时区错位**：Laravel Scheduler 用 `config('app.timezone')`（UTC）全局唯一触发时刻；Recurr 内部用 `setting('localisation.timezone')`（按公司切换）。多公司各时区下调度触发有偏移，但 Recurr 层保证日期计算不偏差。

15. **allCompanies 去 scope + makeCurrent 切上下文**：allCompanies() 移除 Company 全局 scope 查出所有公司 recurring；逐条 makeCurrent() 切上下文，使 setting() 配置、后续查询、新建模型的 company_id 自动属于该公司。防止跨公司数据泄露。

16. **4 条孤儿清理路径**：handle() 主循环内置 4 个 GC 分支——公司不存在、公司禁用超3个月、无活跃用户超3个月、关联模板已被删——分别在 makeCurrent 前后分布执行，防止脏数据长期占用调度。

17. **updateRecurring 三分支架构**：frequency=no 走删除分支（与 createRecurring 的 no return 不同）；frequency 有效时按 recurring 关联是否存在分别走 update（不改 created_from/by）或 create（补创建，强制 ACTIVE_STATUS）分支。End 按钮改 ended 状态正是走 update 分支的 status 写入路径。
