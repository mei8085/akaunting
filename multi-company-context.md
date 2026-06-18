# Akaunting 多公司模式下公司上下文管理分析

## 一、整体架构概览

Akaunting 采用 **多租户共享数据库** 架构（Multi-Tenant, Shared Database），所有公司数据存储在同一数据库中，通过 `company_id` 字段进行逻辑隔离。公司上下文的生命周期贯穿 HTTP 请求处理的每个阶段：从路由解析、中间件识别、数据查询到视图渲染。

核心组件关系：

```
HTTP Request → 路由匹配 ({company_id} 前缀) → IdentifyCompany 中间件 
    → 解析 company_id → 权限校验 → company()->makeCurrent()
    → 加载公司设置/货币/模块 → Eloquent Global Scope 自动过滤
    → 菜单构建 → 业务逻辑 → 响应输出
```

---

## 二、公司上下文识别逻辑

### 2.1 中间件注册与路由分组

所有需要公司上下文的路由均通过 `{company_id}` URL 前缀定义，并挂载 `company.identify` 中间件（[IdentifyCompany.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/IdentifyCompany.php)）。

中间件挂载位置（[Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Kernel.php#L32-L145)）：

| 路由组       | 中间件包含 `company.identify` | URL 前缀模式                  |
|-------------|------------------------------|-----------------------------|
| `api`       | ✅ 是                        | `api/{alias}/...`           |
| `common`    | ✅ 是                        | `{company_id}/...`          |
| `admin`     | ✅ 是                        | `{company_id}/...`          |
| `wizard`    | ✅ 是                        | `{company_id}/wizard/...`   |
| `portal`    | ✅ 是                        | `{company_id}/portal/...`   |
| `preview`   | ✅ 是                        | `{company_id}/preview/...`  |
| `signed`    | ✅ 是                        | `{company_id}/signed/...`   |

路由宏定义（[Route.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Providers/Route.php#L40-L73)）自动为模块路由注入 `{company_id}` 前缀：
```php
// 示例：Route::macro('module') 自动拼接 {company_id} 前缀
$attributes['prefix'] = '{company_id}/' . $attrs['prefix'];
```

### 2.2 Company ID 多源识别策略

识别逻辑集中在 [Companies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Companies.php) Trait 的 `getCompanyId()` 方法，根据请求类型采用不同的优先级策略：

#### 2.2.1 Web 请求识别 (`getCompanyIdFromWeb`)

优先级从高到低：
1. **路由参数**：`$request->route('company_id')` 或 URL 第一段 `$request->segment(1)`
2. **查询参数**：`?company_id=2`
3. **HTTP 头**：`X-Company: 2`

#### 2.2.2 API 请求识别 (`getCompanyIdFromApi`)

优先级从高到低：
1. **OAuth Token**：当 `config('oauth.enabled')=true` 且请求携带 Bearer Token 时，从 Token 关联的 `company_id` 提取（最高优先级，确保 OAuth 客户端仅能访问授权公司）
2. **查询参数**：`?company_id=2`
3. **HTTP 头**：`X-Company: 2`
4. **用户首公司**：取当前用户启用的第一个公司作为兜底

#### 2.2.3 MCP 请求识别 (`getCompanyIdFromMcp`)

与 API 类似，但强制使用 `passport` guard 认证，优先级：
1. OAuth Token
2. 查询参数
3. HTTP 头
4. Passport 用户首公司

### 2.3 权限校验

在 [IdentifyCompany.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/IdentifyCompany.php#L35-L38) 中：

```php
// 非签名请求必须校验用户是否属于该公司
if ($this->request->isNotSigned($company_id) && $this->isNotUserCompany($company_id)) {
    throw new AuthenticationException('Unauthenticated.', $guards);
}
```

用户-公司关联通过 `user_companies` 中间表维护，校验方法为 [Users::isUserCompany()](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Users.php#L27-L40)。

---

## 三、公司上下文保存与切换逻辑

### 3.1 上下文容器绑定

公司上下文通过 Laravel 服务容器（Service Container）的单例绑定实现全局访问。核心方法定义在 [Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Models/Common/Company.php#L601-L672)：

#### 3.1.1 `makeCurrent($force = false)` — 设置当前公司

```php
public function makeCurrent($force = false)
{
    if (!$force && $this->isCurrent()) return $this;

    static::forgetCurrent();                      // 1. 清除旧上下文
    event(new CompanyMakingCurrent($this));      // 2. 触发前置事件
    app()->instance(static::class, $this);       // 3. 绑定到容器

    // 4. 加载公司设置
    setting()->setExtraColumns(['company_id' => $this->id]);
    setting()->forgetAll();
    setting()->load(true);

    // 5. 覆盖系统配置（时区、邮件、货币、分类）
    Overrider::load('settings');
    Overrider::load('currencies');
    Overrider::load('categoryTypes');

    event(new CompanyMadeCurrent($this));        // 6. 触发后置事件
    return $this;
}
```

#### 3.1.2 `getCurrent()` / `company()` — 获取当前公司

```php
// helpers.php 中定义的全局辅助函数
function company(int|null $id = null): Company|null
{
    if (is_null($id)) return Company::getCurrent();  // 从容器取
    if (is_numeric($id)) return Company::find($id);  // 从数据库取
}

function company_id(): int|null
{
    return company()?->id;
}
```

#### 3.1.3 `forgetCurrent()` — 清除上下文

```php
public static function forgetCurrent()
{
    $current = static::getCurrent();
    if (is_null($current)) return null;

    event(new CompanyForgettingCurrent($current));
    app()->forgetInstance(static::class);  // 从容器解绑
    setting()->forgetAll();                // 清除设置缓存
    event(new CompanyForgotCurrent($current));
    return $current;
}
```

### 3.2 IdentifyCompany 中间件完整流程

[IdentifyCompany.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/IdentifyCompany.php#L25-L62) 是上下文初始化的入口：

| 步骤 | 操作 | 代码位置 |
|------|------|----------|
| 1 | 调用 `getCompanyId()` 解析公司 ID | L29 |
| 2 | 空值检查，否则 500 | L31-L33 |
| 3 | 用户-公司权限校验（签名请求除外） | L36-L38 |
| 4 | `company($company_id)` 从数据库加载 Company 模型 | L41 |
| 5 | `$company->makeCurrent()` 绑定上下文并加载设置 | L47 |
| 6 | `registerModules()` 加载公司专属模块（事件监听器、Provider） | L50 |
| 7 | 修正文件上传 URL 路径（`/{company_id}/uploads`） | L53 |
| 8 | 非 API 请求下设置路由默认参数并清除 `company_id` 路由参数 | L56-L59 |

### 3.3 主动切换：用户点击切换公司

切换入口在 [Companies::switch()](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Controllers/Common/Companies.php#L226-L244)：

```php
public function switch(Company $company)
{
    if ($this->isUserCompany($company->id)) {
        $old_company_id = company_id();      // 保存旧ID
        $company->makeCurrent();             // 切换上下文

        // 切换后重置仪表板ID
        session(['dashboard_id' => user()->dashboards()->enabled()->pluck('id')->first()]);

        // 触发切换事件（用于模块感知切换）
        event(new \App\Events\Common\CompanySwitched($company, $old_company_id));

        if (!setting('wizard.completed', false)) {
            return redirect()->route('wizard.edit', ['company_id' => $company->id]);
        }
    }
    return redirect()->route('dashboard', ['company_id' => $company->id]);
}
```

### 3.4 事件系统

公司生命周期事件定义（[Events/Common/](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Events/Common)）：

| 事件类 | 触发时机 | 用途 |
|--------|---------|------|
| `CompanyMakingCurrent` | `makeCurrent()` 绑定容器之前 | 预处理：关闭旧连接、清理缓存 |
| `CompanyMadeCurrent` | `makeCurrent()` 加载设置之后 | 模块初始化、菜单重建 |
| `CompanyForgettingCurrent` | `forgetCurrent()` 解绑之前 | 保存状态、清理资源 |
| `CompanyForgotCurrent` | `forgetCurrent()` 解绑之后 | 后清理工作 |
| `CompanySwitched` | 用户主动点击切换时（在 switch 控制器中） | UI 层面的响应（刷新菜单、提示等） |

### 3.5 登出时的清理

[Logout.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Listeners/Auth/Logout.php#L15-L18)：
```php
public function handle(Event $event)
{
    session()->forget('company_id');
}
```

---

## 四、对菜单的连锁影响

### 4.1 菜单构建时机

菜单通过中间件构建，位于 `company.identify` **之后**：

- **Admin 菜单**：`menu.admin` 中间件 → [AdminMenu.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/AdminMenu.php)
- **Portal 菜单**：`menu.portal` 中间件 → [PortalMenu.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/PortalMenu.php)

中间件优先级（[Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Kernel.php#L78-L90)）：
```
admin 组顺序：
  web → auth → auth.disabled → company.identify → ... → menu.admin → permission
```

**关键保证**：菜单构建时公司上下文已就绪，所有菜单中的权限检查、翻译、路由生成均基于当前公司。

### 4.2 菜单构建流程

以 Admin 菜单为例：

```php
// AdminMenu.php
menu()->create('admin', function ($menu) {
    $menu->style('tailwind');
    event(new AdminCreating($menu));  // 前置事件（模块可注入菜单项）
    event(new AdminCreated($menu));   // 后置事件
});
```

默认菜单项由 [ShowInAdmin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Listeners/Menu/ShowInAdmin.php) 监听器注册，通过 `canAccessMenuItem()` 进行权限判断，权限基于当前用户在**当前公司**的角色。

### 4.3 公司切换对菜单的影响

1. **翻译文本变化**：`trans_choice()`、`trans()` 基于 `setting('default.locale')`（公司级设置），切换后菜单项文字可能变化
2. **权限可见性变化**：用户在不同公司可能有不同角色，`$this->canAccessMenuItem()` 结果可能不同
3. **模块注入变化**：`registerModules()` 会加载当前公司启用的模块，模块的 `AdminCreating` 监听器可能动态增减菜单项
4. **URL 变化**：所有 `route()` 生成的 URL 自动携带新的 `{company_id}`（通过 `url()->defaults()` 设置）

---

## 五、对数据访问的连锁影响

### 5.1 Eloquent Global Scope 自动过滤

核心机制：[Tenants.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Tenants.php) Trait 在模型 boot 时注册全局作用域 [Company Scope](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Scopes/Company.php)。

```php
// Tenants.php
protected static function bootTenants()
{
    static::addGlobalScope(new Company);  // 自动注册
}
```

所有继承自 [Abstracts/Model.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Abstracts/Model.php#L21-L24) 的模型默认使用 `Tenants` Trait：
```php
abstract class Model extends Eloquent implements Ownable
{
    use ..., Tenants;
    protected $tenantable = true;  // 默认启用租户隔离
}
```

### 5.2 Scope 过滤逻辑

[Company.php (Scope)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Scopes/Company.php#L21-L50) 的 `apply()` 方法：

```php
public function apply(Builder $builder, Model $model)
{
    // 1. 模型层面排除：isNotTenantable() 返回 true 的模型
    if (method_exists($model, 'isNotTenantable') && $model->isNotTenantable()) return;

    // 2. 表层面排除：以下系统级表不加 company_id 过滤
    $skip_tables = [
        'jobs', 'firewall_ips', 'firewall_logs', 'migrations', 'notifications',
        'role_companies', 'role_permissions', 'sessions', 'user_companies',
        'user_dashboards', 'user_permissions', 'user_roles',
    ];
    if (in_array($table, $skip_tables)) return;

    // 3. 避免重复：若查询中已包含 company_id 条件则跳过
    if ($this->scopeColumnExists($builder, '', 'company_id')) return;

    // 4. 追加条件：WHERE {table}.company_id = {company_id()}
    $builder->where($table . '.company_id', '=', company_id());
}
```

### 5.3 模型排除机制

模型可通过两种方式跳过公司过滤：

1. **设置 `$tenantable = false`**：在模型中声明，`isTenantable()` 将返回 false
2. **`company_id` 不在 `$fillable` 中**：`isTenantable()` 会检查 `in_array('company_id', $this->getFillable())`

### 5.4 临时绕过过滤

使用 `allCompanies()` scope 可临时绕过：
```php
// Abstracts/Model.php
public function scopeAllCompanies($query)
{
    return $query->withoutGlobalScope('App\Scopes\Company');
}

// 用法示例（后台任务中遍历所有公司的重复记录）
$recurring = Recurring::with('company')
    ->active()
    ->allCompanies()   // 移除公司过滤
    ->cursor();
```

### 5.5 设置（Settings）的特殊处理

公司设置通过 [akaunting/setting](https://github.com/akaunting/setting) 包管理，使用 `extra_columns` 机制：

```php
// makeCurrent() 中
setting()->setExtraColumns(['company_id' => $this->id]);  // 绑定当前公司
setting()->forgetAll();                                   // 清空缓存
setting()->load(true);                                    // 重新加载
```

所有 `setting('key')` 调用自动限定在 `company_id = 当前公司` 的记录。

### 5.6 缓存前缀隔离

[helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Utilities/helpers.php#L158-L166) 中定义：
```php
function cache_prefix(): string
{
    return company_id() . '_';  // 每个公司独立缓存命名空间
}
```

结合 `laravel-model-caching`（[Model.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Abstracts/Model.php#L23) 中使用了 `Cachable` Trait），确保各公司查询缓存互不干扰。

---

## 六、对后台任务（Console / Queue）的连锁影响

### 6.1 后台任务的特殊性

后台命令（Artisan Command）和队列任务（Job）**不经过 HTTP 中间件栈**，因此 `IdentifyCompany` 不会自动执行。开发人员必须手动管理公司上下文。

### 6.2 定时任务中的典型模式

三个定时任务（[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/RecurringCheck.php)、[InvoiceReminder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/InvoiceReminder.php)、[BillReminder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/BillReminder.php)）均遵循"遍历所有公司 → 切换上下文 → 处理 → 清理"模式：

**标准流程模板**：

```php
public function handle()
{
    // 1. 禁用模型缓存（跨公司遍历会导致缓存错乱）
    config(['laravel-model-caching.enabled' => false]);

    // 2. 查询所有目标公司（必须用 allCompanies() 跳过过滤）
    $companies = Company::whereHas(...)->enabled()->cursor();

    foreach ($companies as $company) {
        // 3a. 切换上下文到该公司
        $company->makeCurrent();

        // 3b. 此时 setting()、Eloquent 查询、货币配置均为该公司
        // ... 执行业务逻辑 ...
    }

    // 4. 循环结束后清理上下文
    Company::forgetCurrent();
}
```

### 6.3 各定时任务的上下文管理分析

#### 6.3.1 `recurring:check` — 重复账单检查

[RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/RecurringCheck.php#L40-L162)：

| 行号 | 操作 | 说明 |
|------|------|------|
| L50-L57 | `Recurring::...->allCompanies()->cursor()` | 遍历所有公司的重复计划 |
| L62-L68 | 检查 `$recur->company` 是否存在 | 处理脏数据 |
| L77-L89 | 检查公司是否启用 | 禁用公司超3个月无活跃则删除重复模板 |
| L92-L112 | 检查是否有活跃用户 | 3个月无登录用户则跳过并清理 |
| L114 | `company($recur->company_id)->makeCurrent()` | **切换上下文** |
| L127-L153 | 执行重复账单生成 | 此期间所有查询均带该公司过滤 |
| L156 | `Company::forgetCurrent()` | 循环后清理 |

#### 6.3.2 `reminder:invoice` / `reminder:bill` — 催款提醒

[InvoiceReminder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/InvoiceReminder.php#L34-L77)：

```php
foreach ($companies as $company) {
    $company->makeCurrent();            // 切换上下文

    // setting() 在此之后读取该公司的计划配置
    if (!setting('schedule.send_invoice_reminder')) continue;

    $days = explode(',', setting('schedule.invoice_days'));
    foreach ($days as $day) {
        $this->remind($day);  // Document::invoice() 自动被 Scope 过滤
    }
}
Company::forgetCurrent();
```

### 6.4 上下文切换在 Cron Job 中的连锁效应

| 影响维度 | 具体表现 |
|---------|---------|
| **Eloquent 查询** | 无需手动加 `where('company_id', x)`，Global Scope 自动追加 |
| **设置读取** | `setting('schedule.send_invoice_reminder')` 读取当前公司配置 |
| **时区** | `Overrider::load('settings')` 会覆盖 `config('app.timezone')`，`Date::now()` 随之变化 |
| **邮件配置** | `mail.from`、SMTP 服务器等均为该公司配置，发送邮件时自动使用 |
| **货币** | `Money::setLocale()`、`default_currency()` 等基于该公司设置 |
| **事件/监听器** | `registerModules()` 加载的模块监听器会参与处理（如 DocumentCreated 监听器） |
| **文件路径** | 文件上传 URL 配置 `filesystems.disks.*.url` 自动拼接 `/{company_id}/uploads` |

### 6.5 队列 Job 中的特殊注意

队列 Job 同样不经过 HTTP 中间件，但如果是从 Web 请求中派发的，需要注意：

1. **同步派发 (`dispatchSync`)**：继承当前请求的公司上下文，无需额外处理
2. **异步派发 (`dispatchQueue`)**：Worker 进程中上下文为空，必须：
   - 在 Job 类中保存 `company_id` 到属性
   - 在 `handle()` 方法开头调用 `company($this->company_id)->makeCurrent()`
   - 处理完成后调用 `Company::forgetCurrent()`（特别是 Worker 进程常驻场景）

[Abstracts/Job.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Abstracts/Job.php) 基类中未内置上下文管理，各 Job 需自行实现。

### 6.6 Schedule 调度器注册

[Console/Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Kernel.php#L23-L37)：

```php
protected function schedule(Schedule $schedule)
{
    if (!config('app.installed')) return;  // 未安装时不调度

    $schedule_time = config('app.schedule_time');  // 注意：此为全局配置，非公司级

    $schedule->command('reminder:invoice')->dailyAt($schedule_time);
    $schedule->command('reminder:bill')->dailyAt($schedule_time);
    $schedule->command('recurring:check')->dailyAt($schedule_time)->runInBackground();
    $schedule->command('storage-temp:clear')->dailyAt('17:00');
    $schedule->command('model:prune')->dailyAt('17:00');
}
```

**注意**：调度时间是全局统一的，各公司无法设置各自的调度时间；实际执行时由命令内部遍历各公司决定是否跳过。

---

## 七、Overrider：公司级配置覆盖

[Overrider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Utilities/Overrider.php) 在 `makeCurrent()` 中被调用三次，将公司设置同步到 Laravel Config：

### 7.1 `loadSettings()` — 基础配置覆盖

| 配置项 | 来源 (setting key) | 目标 (config key) |
|--------|-------------------|------------------|
| 时区 | `localisation.timezone` | `app.timezone` + `date_default_timezone_set()` |
| 邮件驱动 | `email.protocol` | `mail.default` |
| 发件人名称 | `company.name` | `mail.from.name` |
| 发件人地址 | `company.email` | `mail.from.address` |
| SMTP 配置 | `email.smtp_*` | `mail.mailers.smtp.*` |
| Sendmail 路径 | `email.sendmail_path` | `mail.mailers.sendmail.path` |
| 语言 | `default.locale` | `app()->setLocale()` |
| 默认货币 | `default.currency` | `money.defaults.currency` |
| Money 包 Locale | app locale | `Money::setLocale()` |

### 7.2 `loadCurrencies()` — 货币表加载

从 `currencies` 表（已通过 Global Scope 过滤当前公司）读取所有货币，注入到 `money.currencies.*` config，最后调用 `Currency::setCurrencies()` 刷新内存。

### 7.3 `loadCategoryTypes()` — 分类类型注入

读取公司自定义的收入/支出/物品/其他分类类型，更新到 `search-string` config 中的搜索字段路由参数，确保搜索下拉选项为该公司的分类配置。

---

## 八、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 公司识别中间件 | [IdentifyCompany.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/IdentifyCompany.php) |
| Company ID 解析 Trait | [Companies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Companies.php) |
| Company 模型（上下文管理） | [Company.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Models/Common/Company.php) |
| 全局辅助函数 | [helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Utilities/helpers.php) |
| 中间件注册 | [Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Kernel.php) |
| 路由分组与前缀 | [Route.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Providers/Route.php) |
| Eloquent 全局 Scope | [Company.php (Scope)](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Scopes/Company.php) |
| 租户 Trait | [Tenants.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Tenants.php) |
| Model 基类 | [Model.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Abstracts/Model.php) |
| 配置覆盖器 | [Overrider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Utilities/Overrider.php) |
| 公司切换控制器 | [Companies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Controllers/Common/Companies.php) |
| 用户-公司关系校验 | [Users.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Users.php) |
| Admin 菜单构建 | [AdminMenu.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Http/Middleware/AdminMenu.php) |
| Admin 默认菜单项 | [ShowInAdmin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Listeners/Menu/ShowInAdmin.php) |
| 定时任务调度 | [Console/Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Kernel.php) |
| 重复账单任务 | [RecurringCheck.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/RecurringCheck.php) |
| 发票催款任务 | [InvoiceReminder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/InvoiceReminder.php) |
| 账单催款任务 | [BillReminder.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Console/Commands/BillReminder.php) |
| 登出清理 | [Logout.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Listeners/Auth/Logout.php) |
| 事件-监听器映射 | [Event.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Providers/Event.php) |
| Job 基类 | [Job.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Abstracts/Job.php) |
| Job Trait（派发） | [Jobs.php](file:///d:/fz/0601-2/solo-dogfeeding/code/33-akaunting/app/Traits/Jobs.php) |
