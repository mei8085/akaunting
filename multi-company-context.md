# Akaunting 多公司模式下公司上下文管理分析

## 一、整体架构概览

Akaunting 采用 **多租户共享数据库** 架构（Multi-Tenant, Shared Database），所有公司数据存储在同一数据库中，通过 `company_id` 字段进行逻辑隔离。公司上下文的生命周期贯穿 HTTP 请求处理、队列任务执行、定时任务调度的每个阶段。

核心组件关系：

```
HTTP 请求路径：
HTTP Request → 路由匹配 ({company_id} 前缀) → IdentifyCompany 中间件 
    → 解析 company_id → 权限校验 → company()->makeCurrent()
    → 加载公司设置/货币/模块 → Eloquent Global Scope 自动过滤
    → 菜单构建 → 业务逻辑 → 响应输出

异步队列路径：
Job 入队 → createPayloadUsing 注入 company_id → 存储到队列驱动
    → Worker 取出任务 → JobProcessing 事件
    → 从 payload 读取 company_id → company()->makeCurrent() → registerModules()
    → Laravel 反序列化 Job 对象（⚠️ 此处关键时序）
    → Job::handle() 执行 → 完成/失败

定时任务路径：
Schedule 触发 → Artisan Command::handle()
    → allCompanies() 遍历 → 循环内 makeCurrent() → 业务逻辑 → forgetCurrent()
```

---

## 二、公司上下文识别逻辑

### 2.1 中间件注册与路由分组

所有需要公司上下文的路由均通过 `{company_id}` URL 前缀定义，并挂载 `company.identify` 中间件（`app/Http/Middleware/IdentifyCompany.php`）。

中间件挂载位置（`app/Http/Kernel.php`）：

| 路由组       | 中间件包含 `company.identify` | URL 前缀模式                  |
|-------------|------------------------------|-----------------------------|
| `api`       | ✅ 是                        | `api/{alias}/...`           |
| `common`    | ✅ 是                        | `{company_id}/...`          |
| `admin`     | ✅ 是                        | `{company_id}/...`          |
| `wizard`    | ✅ 是                        | `{company_id}/wizard/...`   |
| `portal`    | ✅ 是                        | `{company_id}/portal/...`   |
| `preview`   | ✅ 是                        | `{company_id}/preview/...`  |
| `signed`    | ✅ 是                        | `{company_id}/signed/...`   |

路由宏定义（`app/Providers/Route.php`）自动为模块路由注入 `{company_id}` 前缀：
```php
// 示例：Route::macro('module') 自动拼接 {company_id} 前缀
$attributes['prefix'] = '{company_id}/' . $attrs['prefix'];
```

### 2.2 Company ID 多源识别策略

识别逻辑集中在 `app/Traits/Companies.php` Trait 的 `getCompanyId()` 方法，根据请求类型采用不同的优先级策略：

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

在 `app/Http/Middleware/IdentifyCompany.php` 中：

```php
// 非签名请求必须校验用户是否属于该公司
if ($this->request->isNotSigned($company_id) && $this->isNotUserCompany($company_id)) {
    throw new AuthenticationException('Unauthenticated.', $guards);
}
```

用户-公司关联通过 `user_companies` 中间表维护，校验方法为 `app/Traits/Users.php::isUserCompany()`。

---

## 三、公司上下文保存与切换逻辑

### 3.1 上下文容器绑定

公司上下文通过 Laravel 服务容器（Service Container）的单例绑定实现全局访问。核心方法定义在 `app/Models/Common/Company.php`：

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

`app/Http/Middleware/IdentifyCompany.php` 是上下文初始化的入口：

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

切换入口在 `app/Http/Controllers/Common/Companies.php::switch()`：

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

公司生命周期事件定义（`app/Events/Common/`）：

| 事件类 | 触发时机 | 用途 |
|--------|---------|------|
| `CompanyMakingCurrent` | `makeCurrent()` 绑定容器之前 | 预处理：关闭旧连接、清理缓存 |
| `CompanyMadeCurrent` | `makeCurrent()` 加载设置之后 | 模块初始化、菜单重建 |
| `CompanyForgettingCurrent` | `forgetCurrent()` 解绑之前 | 保存状态、清理资源 |
| `CompanyForgotCurrent` | `forgetCurrent()` 解绑之后 | 后清理工作 |
| `CompanySwitched` | 用户主动点击切换时（在 switch 控制器中） | UI 层面的响应（刷新菜单、提示等） |

### 3.5 登出时的清理

`app/Listeners/Auth/Logout.php`：
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

- **Admin 菜单**：`menu.admin` 中间件 → `app/Http/Middleware/AdminMenu.php`
- **Portal 菜单**：`menu.portal` 中间件 → `app/Http/Middleware/PortalMenu.php`

中间件优先级（`app/Http/Kernel.php`）：
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

默认菜单项由 `app/Listeners/Menu/ShowInAdmin.php` 监听器注册，通过 `canAccessMenuItem()` 进行权限判断，权限基于当前用户在**当前公司**的角色。

### 4.3 公司切换对菜单的影响

1. **翻译文本变化**：`trans_choice()`、`trans()` 基于 `setting('default.locale')`（公司级设置），切换后菜单项文字可能变化
2. **权限可见性变化**：用户在不同公司可能有不同角色，`$this->canAccessMenuItem()` 结果可能不同
3. **模块注入变化**：`registerModules()` 会加载当前公司启用的模块，模块的 `AdminCreating` 监听器可能动态增减菜单项
4. **URL 变化**：所有 `route()` 生成的 URL 自动携带新的 `{company_id}`（通过 `url()->defaults()` 设置）

---

## 五、对数据访问的连锁影响

### 5.1 Eloquent Global Scope 自动过滤

核心机制：`app/Traits/Tenants.php` Trait 在模型 boot 时注册全局作用域 `app/Scopes/Company.php`。

```php
// Tenants.php
protected static function bootTenants()
{
    static::addGlobalScope(new Company);  // 自动注册
}
```

所有继承自 `app/Abstracts/Model.php` 的模型默认使用 `Tenants` Trait：
```php
abstract class Model extends Eloquent implements Ownable
{
    use ..., Tenants;
    protected $tenantable = true;  // 默认启用租户隔离
}
```

### 5.2 Scope 过滤逻辑

`app/Scopes/Company.php` 的 `apply()` 方法：

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

**特别注意**：`Company` 模型本身 `$fillable` 中不含 `company_id`，因此**不会被 Company Scope 过滤**。这是设计使然——上下文未建立时也需要能查询 Company 表。

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

公司设置通过 `akaunting/setting` 包管理，使用 `extra_columns` 机制：

```php
// makeCurrent() 中
setting()->setExtraColumns(['company_id' => $this->id]);  // 绑定当前公司
setting()->forgetAll();                                   // 清空缓存
setting()->load(true);                                    // 重新加载
```

所有 `setting('key')` 调用自动限定在 `company_id = 当前公司` 的记录。

### 5.6 缓存前缀隔离

`app/Utilities/helpers.php` 中定义：
```php
function cache_prefix(): string
{
    return company_id() . '_';  // 每个公司独立缓存命名空间
}
```

结合 `laravel-model-caching`（`app/Abstracts/Model.php` 中使用了 `Cachable` Trait），确保各公司查询缓存互不干扰。

---

## 六、对后台任务（Console / Queue）的连锁影响

### 6.1 后台任务的特殊性

后台命令（Artisan Command）和队列任务（Job）**不经过 HTTP 中间件栈**，因此 `IdentifyCompany` 不会自动执行。但队列任务和 Console 命令采用了不同的上下文管理策略：

| 场景 | 上下文管理方式 | 代码位置 |
|------|--------------|---------|
| HTTP 请求 | `IdentifyCompany` 中间件自动处理 | `app/Http/Middleware/IdentifyCompany.php` |
| Artisan 定时任务 | 开发人员手动遍历 + `makeCurrent()` | 各 Command::handle() |
| 异步队列 Job | `Queue ServiceProvider` 自动注入+恢复 | `app/Providers/Queue.php` |
| 同步 Job (`dispatchSync`) | 继承派发时的请求上下文，无需处理 | 隐式 |

### 6.2 异步队列：公司编号的自动传递与恢复

#### 6.2.1 核心机制：Queue ServiceProvider

`app/Providers/Queue.php` 是异步队列上下文管理的**唯一入口**，通过两个钩子实现全自动传递恢复：

**钩子一：入队时注入 company_id（createPayloadUsing）**

```php
// 每当一个 Job 被序列化推入队列时触发
app('queue')->createPayloadUsing(function ($connection, $queue, $payload) {
    $company_id = company_id();  // 读取派发时刻的当前上下文

    if (empty($company_id)) {
        return [];  // 如果派发时无上下文（如纯 CLI 环境），不注入
    }

    return ['company_id' => $company_id];  // 附加到队列 payload 顶层
});
```

**关键特性**：
- `company_id` 存放在 payload 的**顶层**（而非 `data.command` 内部），Laravel 反序列化 Job 对象时不会自动处理
- 仅当 `company_id()` 非空时才注入，因此从 Console（无上下文）派发的任务不会带 company_id
- 对**所有** `ShouldQueue` 接口的实例生效：包括 `JobShouldQueue` 子类、`Notification`、自定义 `ShouldQueue` 事件监听器等

**钩子二：出队时恢复上下文（JobProcessing 事件）**

```php
// 每当 Worker 从队列取出一个 Job 准备执行前触发
app('events')->listen(JobProcessing::class, function ($event) {
    $payload = $event->job->payload();

    // 1. 提取注入的 company_id
    if (! array_key_exists('company_id', $payload)) {
        return;  // 无 company_id 则跳过（如纯 CLI 派发的任务）
    }

    // 2. 加载 Company 模型，异常或不存在则删除任务
    try {
        $company = company($payload['company_id']);
    } catch (\Throwable $e) {
        $event->job->delete();  // 静默删除，避免反复重试
        logger()->warning('Company could not be resolved...', [...]);
        return;
    }

    if (empty($company)) {
        $event->job->delete();
        logger()->warning('Company not found, job deleted.', [...]);
        return;
    }

    // 3. 恢复上下文（与 HTTP 中间件中的 L47 完全相同）
    $company->makeCurrent();

    // 4. 注册该公司启用的模块（仅在真实异步场景下）
    if (should_queue()) {
        $this->registerModules();  // 加载模块的事件监听器、路由、视图等
    }
});
```

#### 6.2.2 异步任务完整生命周期时序

```
Web 请求线程（公司A上下文）：
  ┌─ 用户提交操作
  │  1. 当前上下文：company_id = 1（公司A）
  │  2. dispatch(new SendInvoiceNotification($invoice))
  │  3. ↓ createPayloadUsing 钩子触发
  │  4. payload.company_id = 1  ← 注入到顶层
  │  5. payload.data.command = serialize(SendInvoiceNotification)
  │     ($invoice 模型被 SerializesModels 转换为仅含 ID)
  │  6. 整个 payload 写入数据库/Redis 队列
  └─ 响应返回用户

Worker 进程：
  ┌─ queue:work 循环取出任务
  │  1. 从队列读取 payload → 解析出 company_id=1 和序列化的 Job 对象
  │  2. ↓ JobProcessing 事件触发
  │  3. company(1)->makeCurrent()    ← 恢复公司A上下文
  │  4. $this->registerModules()     ← 加载公司A启用的模块
  │  5. ↓ Laravel 反序列化 Job 对象（__unserialize）
  │     → getRestoredPropertyValue()
  │       → restoreModel()
  │         → newQueryForRestoration()  ← ⚠️ 此处调用 withoutGlobalScopes()
  │         → SELECT * FROM documents WHERE id = ?  ← 无 company_id 条件！
  │  6. ↓ Job::handle() 执行
  │     此时 setting() = 公司A的配置，新 Eloquent 查询自动带 company_id=1
  │     但反序列化得到的 $invoice 本身可能来自任意公司（如果 payload 被篡改）
  │  7. ↓ JobProcessed 事件（当前未做上下文清理）
  └─ Worker 等待下一个任务（容器中仍保留 company_id=1）
```

#### 6.2.3 Worker 常驻的注意事项

由于 `php artisan queue:work` 是常驻进程，存在以下影响：

| 问题 | 表现 | 风险 |
|------|------|------|
| **上下文残留** | 前一个 Job 执行后，`Company::getCurrent()` 仍保留在容器中 | 下一个无 company_id 的 Job 会无意中继承错误上下文 |
| **配置残留** | `Overrider::load()` 修改的是 `config()` 全局状态 | 后续任务如果上下文切换失败，会读到上一个公司的时区/邮件/货币配置 |
| **模块残留** | `registerModules()` 加载的监听器不会自动卸载 | 已禁用模块的监听器可能仍在后续任务中被触发 |

**代码中未处理**：`app/Providers/Queue.php` 中没有注册 `JobProcessed` 事件监听器来清理上下文。这在多公司混合队列场景下是潜在的风险点。

### 6.3 异步任务中 company_id 的传递层级

除了 `Queue ServiceProvider` 的自动机制外，代码中还存在三层**手动**传递策略：

#### 层级一：Job 属性显式保存（安装模块类 Job）

典型模式见 `app/Jobs/Install/EnableModule.php`：
```php
class EnableModule extends Job  // 同步 Job 基类，不 implements ShouldQueue
{
    protected $alias;
    protected $company_id;    // ← 显式属性
    protected $locale;

    public function __construct($alias, $company_id, $locale = null)
    {
        $this->alias = $alias;
        $this->company_id = (int) $company_id;
        $this->locale = $locale ?: company($company_id)->locale ?: config('setting.fallback.default.locale');
        // 构造时就利用上下文取 locale
    }

    public function handle()
    {
        // 注意：此 Job 不走自动恢复，handle() 内必须自己确保上下文
        $command = "module:enable {$this->alias} {$this->company_id} {$this->locale}";
        Console::run($command);  // 通过 artisan 参数传递给子命令
    }
}
```

同类模式也出现在：
- `app/Jobs/Install/DisableModule.php`
- `app/Jobs/Install/DownloadModule.php`

这些 Job 虽然保存了 `$company_id`，但属于**同步基类**（继承 `App\Abstracts\Job` 而非 `JobShouldQueue`），通常在 `dispatchSync` 中运行，依赖派发时已经存在的上下文。

#### 层级二：JobShouldQueue 基类 + SerializesModels

`app/Abstracts/JobShouldQueue.php` 是异步 Job 的基类：
```php
abstract class JobShouldQueue implements ShouldQueue
{
    use InteractsWithQueue, Jobs, Queueable, Relationships,
        SerializesModels,   // ← 关键：Eloquent 模型序列化时只存 ID
        Sources, Uploads;
    // ...
}
```

**`SerializesModels` 与上下文恢复的配合（⚠️ 关键安全分析）**：

1. 入队序列化时：所有 Eloquent Model 属性（如 `$this->model`、`$this->invoice`）转换为仅含主键的 `ModelIdentifier`
2. 出队反序列化时：Laravel 调用 `restoreModel()` 用主键重新查询数据库
3. **核心问题**：`restoreModel()` 内部调用 `newQueryForRestoration()`，该方法**明确跳过所有 Global Scope**（包括 Company Scope）
4. **时序保护**：反序列化发生在 `JobProcessing` 事件**之后**（即 `makeCurrent()` 已执行），但 `newQueryForRestoration()` 会忽略这一点，仍然跳过 Scope

**Laravel 源码确认**（`vendor/laravel/framework/src/Illuminate/Queue/SerializesAndRestoresModelIdentifiers.php`）：
```php
protected function getQueryForModelRestoration($model, $ids)
{
    return $model->newQueryForRestoration($ids);
}
```

而 `newQueryForRestoration()` 内部（`Illuminate/Database/Eloquent/Builder.php`）：
```php
public function newQueryForRestoration($ids)
{
    // ⚠️ 注意：使用的是 newQueryWithoutScopes()
    return $this->newQueryWithoutScopes()->whereIn(
        $this->getQualifiedKeyName(), Arr::wrap($ids)
    );
}
```

**Akaunting 没有自定义覆盖**：代码搜索确认，Akaunting 未重写 `getQueryForModelRestoration()`、`restoreModel()`、`getRestoredPropertyValue()` 或 `__unserialize()` 中的任何一个。

**实际安全边界**：
- ✅ 如果 payload 未被篡改，反序列化查询的 ID 是派发时确定的，属于正确的公司，即使跳过 Scope 也没问题
- ✅ 反序列化恢复模型后，`Job::handle()` 中的**所有新查询**仍会正常应用 Company Scope
- ✅ 模型关联查询（`$invoice->items`）也会应用 Scope（因为关联查询走的是正常的 `newQuery()`）
- ❌ 如果攻击者能篡改队列 payload（SQL 注入修改 `jobs` 表、Redis 写入权限），将 `invoice_id` 改为其他公司的有效 ID，反序列化会成功加载跨公司数据

#### 层级三：Eloquent 关联隐式获取（Notification 类）

`app/Notifications/Sale/Invoice.php` 等通知类不保存 company_id，但通过模型关联隐式获取：
```php
class Invoice extends Notification  // Notification 基类已 implements ShouldQueue
{
    public $invoice;  // ← 整个 Document 模型，序列化时只存 ID

    public function getTagsReplacement(): array
    {
        return [
            // ...
            $this->invoice->company->name,    // ← 关联查询时 Scope 已生效
            $this->invoice->company->email,
            // ...
            'company_id' => $this->invoice->company_id,  // ← 模型属性直接取
        ];
    }

    public function toMail($notifiable): MailMessage
    {
        $message = $this->initMailMessage();  // ← 内部调用 company_id()
        // ...
    }
}
```

在 `app/Abstracts/Notification.php` 中：
```php
public function initMailMessage(): MailMessage
{
    app('url')->defaults(['company_id' => company_id()]);  // 确保 URL 生成正确
    $message = (new MailMessage)
        ->from(config('mail.from.address'), config('mail.from.name'))  // 已恢复的邮件配置
        // ...
}
```

### 6.4 模块注册在异步任务中的影响

#### 6.4.1 为什么需要 registerModules()

`registerModules()` 定义在 `app/Traits/Modules.php` 中，实质调用 `ModuleActivator::register()`。它负责：

1. **加载模块的 ServiceProvider**：`register()` 和 `boot()` 方法
2. **注册模块的事件监听器**：如 `DocumentCreated`、`TransactionCreated` 等业务事件
3. **注册模块的路由**：包括 API、Admin、Portal 路由
4. **注册模块的视图命名空间**：`view('module-alias::...')`
5. **注册模块的翻译文件**：`trans('module-alias::...')`
6. **挂载模块的菜单监听器**：`AdminCreating`、`SettingsCreated` 等

#### 6.4.2 在异步任务中的执行条件

```php
// JobProcessing 中：
if (should_queue()) {
    $this->registerModules();
}
```

`should_queue()` 判断 `QUEUE_CONNECTION !== 'sync'`。这意味着：

| 场景 | 是否执行 registerModules | 原因 |
|------|------------------------|------|
| `QUEUE_CONNECTION=database` + Worker | ✅ 是 | `should_queue()` = true |
| `QUEUE_CONNECTION=redis` + Horizon | ✅ 是 | `should_queue()` = true |
| `QUEUE_CONNECTION=sync` + dispatchSync | ❌ 否 | 同步模式下仍在 HTTP 请求生命周期内，模块已由 IdentifyCompany 加载 |
| Artisan 命令内手动 dispatch | 视 `QUEUE_CONNECTION` 而定 | 如用 database/redis 则会执行 |

#### 6.4.3 模块注册失败导致的业务影响

如果异步任务中**跳过**了 `registerModules()`，会导致：

1. **事件监听器缺失**：例如某支付模块监听 `DocumentCreated` 事件来自动创建支付记录 — 在异步生成重复发票（`recurring:check` 中 `DocumentRecurring` 事件）时不会触发
2. **翻译字符串回退**：`trans('module-alias::invoice.subject')` 无法加载，直接返回 key 字符串
3. **视图找不到**：邮件模板中 `view('module-alias::pdf.invoice')` 抛出 `InvalidArgumentException`
4. **自定义 Eloquent 方法缺失**：模块通过 `Builder::macro()` 注册的查询方法无法调用
5. **任务调度丢失**：模块在 `boot()` 中动态注册的 Cron 调度不会生效

### 6.5 通知任务的公司隔离风险分析

通知任务由于其特殊的触发方式和收件人模型，面临独特的隔离风险：

#### 风险1：通知内容泄露

**场景**：发票付款提醒通知 `app/Notifications/Sale/Invoice.php`

```php
// 序列化时：$invoice 只存 ID
// 反序列化时：newQueryForRestoration() 跳过 Company Scope
// 如果 payload 中 invoice_id 被篡改 → 加载其他公司的发票
public function getTagsReplacement(): array
{
    return [
        '{invoice_number}' => $this->invoice->document_number,  // ⚠️ 跨公司数据
        '{invoice_amount}' => money($this->invoice->amount),    // ⚠️ 跨公司数据
        '{customer_name}' => $this->invoice->contact->name,     // ⚠️ 关联查询受 Scope 保护
        '{company_name}' => $this->invoice->company->name,       // ⚠️ 关联查询受 Scope 保护
    ];
}
```

**防护层级分析**：
- `$this->invoice->document_number`：直接从反序列化的模型属性读取，**无 Scope 保护**
- `$this->invoice->contact->name`：通过关联查询，**受 Scope 保护**（因为 `contact()` 关联走正常的 `newQuery()`）
- `$this->invoice->company->name`：通过关联查询，**受 Scope 保护**

#### 风险2：收件人越权

通知通过 `$notifiable`（通常是 User 模型）发送。如果 `$notifiable` 也通过 `SerializesModels` 序列化：
- 反序列化时同样跳过 Company Scope
- 可能导致跨公司用户发送通知
- 但 `$notifiable` 通常是 `auth()->user()`，在派发时就确定，payload 篡改需要同时改多个字段

#### 风险3：邮件发件人身份伪造

`initMailMessage()` 使用 `config('mail.from')`，该配置由 `makeCurrent()` + `Overrider::load('settings')` 设置。
- 如果上下文恢复到错误的公司，邮件会以错误的公司名义发出
- 攻击者可能利用此伪造发件人身份进行钓鱼

### 6.6 导入导出任务的公司隔离风险分析

#### 6.6.1 导出任务（Export 抽象类）

`app/Abstracts/Export.php` 实现了 `ShouldQueue` 但**不使用 `SerializesModels`**，而是使用 Maatwebsite Excel 的 `Exportable` trait。

**安全链路分析**：

```php
// 构造时保存 ID 数组（普通属性，完整序列化）
public function __construct($ids = null)
{
    $this->ids = $ids;
    // ...
}

// collection() 中查询数据
public function collection()
{
    // collectForExport 是一个普通 Local Scope
    return Model::with('category')->collectForExport($this->ids);
}
```

`collectForExport` scope 定义在 `app/Abstracts/Model.php`：
```php
public function scopeCollectForExport($query, $ids = [], $sort = 'name', $id_field = 'id')
{
    if (!empty($ids)) {
        $query->whereIn($id_field, (array) $ids);  // 普通 whereIn
    }
    // ... 搜索、排序、分页 ...
    return $query->cursor();  // ⚠️ 注意：此处没有 withoutGlobalScopes()
}
```

**关键结论**：`collectForExport` 是一个普通 Local Scope，**不会移除 Company Global Scope**。因此：
- ✅ 即使 `$this->ids` 被篡改包含其他公司的 ID，Company Scope 也会追加 `WHERE company_id = ?`
- ✅ 最终 SQL 为 `WHERE id IN (1,2,3) AND company_id = 1`，仅返回当前公司的记录
- ✅ 相对安全，但如果 `$ids` 被恶意修改且正好包含当前公司的未授权 ID，仍可能越权导出

**批量导出的特殊风险**：`app/Jobs/Common/CreateZipForDownload.php`
```php
class CreateZipForDownload extends JobShouldQueue  // 使用 SerializesModels！
{
    public $selected;  // ID 数组
    public $class;     // PDF 生成类名

    public function handle()
    {
        // 路径中使用 company_id()
        $folder_path = 'temp/' . company_id() . '/bulk_actions/';

        foreach ($this->selected as $selected) {
            // $this->class 通常是 InvoicePDF 类，内部会查询数据库
            $pdf_path = $this->dispatch(new $this->class($selected, $folder_path));
            // ...
        }
    }
}
```
- `$selected` 是普通数组，序列化时完整保留
- 但 `$this->dispatch()` 的内部类如果使用模型查询，受 Scope 保护
- `company_id()` 用于路径拼接，如果上下文错误，会写入错误的目录

#### 6.6.2 导入任务（Import 抽象类）

`app/Abstracts/Import.php` 同样实现 `ShouldQueue` 但不使用 `SerializesModels`。

**安全链路分析**：

```php
// map() 中显式设置 company_id
public function map($row): array
{
    $row['company_id'] = company_id();  // ⚠️ 依赖上下文正确
    $row['created_by'] = $this->getCreatedById($row);
    // ...
    return $row;
}

// model() 中创建新模型
public function model(array $row)
{
    if (self::hasRow($row)) {  // 重复检查
        return;
    }
    return new Model($row);  // $row 中已包含 company_id
}

// hasRow() 中的查询受 Scope 保护
public function hasRow($row)
{
    // ...
    $this->has_row = $this->model::withoutEvents(function () {
        return $this->model::get($this->columns);  // ✅ 受 Company Scope 保护
    });
    // ...
}
```

**关键风险点**：
1. **上下文依赖强**：`$row['company_id'] = company_id()` 完全依赖上下文正确恢复
2. **导入数据污染**：如果上下文恢复失败（`company_id()` 返回 `null`），所有导入行的 `company_id` 为 `null`，导致数据归属错误
3. **重复检查漏洞**：`hasRow()` 中查询受 Scope 保护，但如果上下文错误，可能错误地判断"不存在重复"而导致跨公司数据重复
4. **分类 ID 解析**：`getCategoryId()` 通过名称查找分类 ID，查找受 Scope 保护，相对安全

### 6.7 定时任务中的典型模式

三个定时任务（`app/Console/Commands/RecurringCheck.php`、`app/Console/Commands/InvoiceReminder.php`、`app/Console/Commands/BillReminder.php`）均遵循"遍历所有公司 → 切换上下文 → 处理 → 清理"模式：

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

### 6.8 各定时任务的上下文管理分析

#### 6.8.1 `recurring:check` — 重复账单检查

`app/Console/Commands/RecurringCheck.php`：

| 行号 | 操作 | 说明 |
|------|------|------|
| L50-L57 | `Recurring::...->allCompanies()->cursor()` | 遍历所有公司的重复计划 |
| L62-L68 | 检查 `$recur->company` 是否存在 | 处理脏数据 |
| L77-L89 | 检查公司是否启用 | 禁用公司超3个月无活跃则删除重复模板 |
| L92-L112 | 检查是否有活跃用户 | 3个月无登录用户则跳过并清理 |
| L114 | `company($recur->company_id)->makeCurrent()` | **切换上下文** |
| L127-L153 | 执行重复账单生成 | 此期间所有查询均带该公司过滤；`DocumentCreated`、`DocumentRecurring` 事件被触发，**需确保模块已加载** |
| L156 | `Company::forgetCurrent()` | 循环后清理 |

**注意**：`RecurringCheck` 是 Artisan 命令，不经过 Queue 的 `JobProcessing` 钩子，因此 `registerModules()` **未自动调用**。如果重复账单生成依赖模块的事件监听器（如电子发票模块需要在 `DocumentCreated` 时生成 XML），需要在该命令中显式调用。

#### 6.8.2 `reminder:invoice` / `reminder:bill` — 催款提醒

`app/Console/Commands/InvoiceReminder.php`：

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

在 `remind()` 方法内部通过 `event(new DocumentReminded($invoice, Notification::class))` 触发通知发送，`Notification` 类如果走异步队列会再次经过 `createPayloadUsing` + `JobProcessing` 的完整上下文传递链。

### 6.9 上下文切换在后台任务中的连锁效应汇总

| 影响维度 | 具体表现 |
|---------|---------|
| **Eloquent 查询** | 无需手动加 `where('company_id', x)`，Global Scope 自动追加（反序列化恢复除外） |
| **设置读取** | `setting('schedule.send_invoice_reminder')` 读取当前公司配置 |
| **时区** | `Overrider::load('settings')` 会覆盖 `config('app.timezone')`，`Date::now()` 随之变化 |
| **邮件配置** | `mail.from`、SMTP 服务器等均为该公司配置，发送邮件时自动使用 |
| **货币** | `Money::setLocale()`、`default_currency()` 等基于该公司设置 |
| **事件/监听器** | `registerModules()` 加载的模块监听器会参与处理（队列 JobProcessing 中自动执行，Artisan 命令需手动） |
| **文件路径** | 文件上传 URL 配置 `filesystems.disks.*.url` 自动拼接 `/{company_id}/uploads` |
| **缓存命名空间** | `cache_prefix()` = `{company_id}_`，避免跨公司缓存污染 |
| **搜索字符串配置** | `categoryTypes` 已被 `loadCategoryTypes()` 重写为公司自定义分类类型 |

### 6.10 Schedule 调度器注册

`app/Console/Kernel.php`：

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

## 七、队列模型反序列化与公司隔离的深度分析

### 7.1 核心问题：`newQueryForRestoration()` 跳过 Global Scope

#### 7.1.1 Laravel 原生行为确认

Laravel 的 `SerializesModels` trait 在反序列化恢复模型时，**明确设计为跳过所有 Global Scope**。这是 Laravel 框架的有意设计，目的是：
- 确保即使模型被软删除（`SoftDeletingScope`）也能被恢复
- 确保队列任务能够获取到派发时引用的精确数据，不受当前上下文过滤规则影响

**调用链路**：
```
__unserialize()
  → getRestoredPropertyValue($value)
    → restoreModel($value) 或 restoreCollection($value)
      → getQueryForModelRestoration($model, $ids)
        → $model->newQueryForRestoration($ids)
          → $this->newQueryWithoutScopes()->whereIn(...)  // ⚠️ 移除所有 Scope
```

#### 7.1.2 对 Akaunting 公司隔离的影响

Company Scope 作为一个 Global Scope，在反序列化恢复时会被跳过，导致：

| 查询场景 | Company Scope 是否生效 | SQL 示例 |
|---------|----------------------|---------|
| 普通查询 `Document::find(123)` | ✅ 是 | `SELECT * FROM documents WHERE id = 123 AND company_id = 1` |
| 关联查询 `$invoice->items` | ✅ 是 | `SELECT * FROM document_items WHERE document_id = 123 AND company_id = 1` |
| 反序列化恢复 | ❌ 否 | `SELECT * FROM documents WHERE id = 123`（无 company_id） |

#### 7.1.3 Akaunting 是否有额外限制？

**答案：没有**。代码搜索确认：

- ❌ 没有重写 `getQueryForModelRestoration()`
- ❌ 没有重写 `restoreModel()`
- ❌ 没有重写 `getRestoredPropertyValue()`
- ❌ 没有重写 `__unserialize()`
- ❌ 没有为队列反序列化添加 `company_id` 校验逻辑

完全依赖 Laravel 原生行为。

### 7.2 实际攻击面与防护分析

#### 7.2.1 攻击前提条件

要利用此漏洞跨公司访问数据，攻击者需要：

1. **队列存储写入权限**：
   - Database 驱动：需要 SQL 注入漏洞修改 `jobs` 表的 `payload` 字段
   - Redis 驱动：需要 Redis 服务器写入权限
   - 同步驱动（`sync`）：不存在队列存储，无此风险

2. **知道目标数据的主键 ID**：
   - 由于只能通过主键查询，攻击者需要知道或猜测其他公司数据的 ID
   - 自增 ID 容易猜测（如 `id = 1` 几乎必然存在）

3. **绕过 payload 签名（如果启用）**：
   - Laravel 队列默认不对 payload 进行签名
   - 如需签名需自定义 `createPayloadUsing` 逻辑

#### 7.2.2 现有防护层级

虽然反序列化查询本身跳过 Scope，但系统存在多层防护降低实际风险：

| 防护层级 | 保护对象 | 实现方式 | 强度 |
|---------|---------|---------|------|
| 1. 队列存储访问控制 | 所有队列任务 | Database/Redis 权限隔离 | 强 |
| 2. payload company_id 绑定 | 上下文恢复 | `JobProcessing` 中根据 payload.company_id 调用 `makeCurrent()` | 中 |
| 3. 模型属性 company_id 校验 | 业务逻辑 | `$model->company_id == company_id()` 手动校验 | 弱（代码中不普遍） |
| 4. 后续查询 Scope 保护 | 新查询和关联查询 | 正常查询仍然应用 Company Scope | 强 |
| 5. 收件人限制 | 通知任务 | 通知发送给 `$notifiable`（派发时确定） | 中 |

#### 7.2.3 高风险场景

**场景 A：发票通知内容泄露**
```php
// 攻击者篡改 jobs 表 payload：
// - company_id: 1（正常）
// - data.command 中序列化的 invoice.id: 999（其他公司的发票ID）

// 反序列化结果：$invoice 是公司 2 的 ID=999 的发票
// 但上下文是公司 1

public function toMail($notifiable)
{
    return (new MailMessage)
        ->line('Invoice #' . $this->invoice->document_number)  // ⚠️ 泄露公司 2 的发票号
        ->line('Amount: ' . money($this->invoice->amount))       // ⚠️ 泄露公司 2 的金额
        ->action('View', route('invoices.show', $this->invoice)); // ✅ 链接带 company_id=1，但 ID=999 不存在于公司 1，点击 404
}
```

**场景 B：批量下载路径污染**
```php
// 正常：company_id = 1
// 篡改后：company_id = 2

public function handle()
{
    // 路径使用被篡改的 company_id
    $folder_path = 'temp/' . company_id() . '/bulk_actions/';
    // 文件被写入公司 2 的目录，可能覆盖合法数据
}
```

**场景 C：Worker 上下文残留**
```
1. 任务 A（公司 1）执行完成，未清理上下文
2. 任务 B（无 company_id，如 CLI 派发的清理任务）开始执行
3. 任务 B 中 `company_id()` 返回 1（来自残留）
4. 任务 B 可能错误地清理或修改公司 1 的数据
```

### 7.3 不同任务类型的风险矩阵

| 任务类型 | 使用 SerializesModels | 反序列化跳过 Scope | 风险等级 | 关键风险点 |
|---------|---------------------|-------------------|----------|-----------|
| **通知类（Notification）** | ✅ 是（基类使用） | ✅ 是 | ⚠️ 高 | 邮件内容泄露、发件人伪造 |
| **异步 Job（JobShouldQueue）** | ✅ 是（基类使用） | ✅ 是 | ⚠️ 高 | 业务逻辑处理跨公司数据 |
| **导出任务（Export）** | ❌ 否（Exportable） | ❌ 否 | ✅ 低 | collectForExport 受 Scope 保护 |
| **导入任务（Import）** | ❌ 否 | ❌ 否 | ⚠️ 中 | 依赖 company_id() 正确设置 |
| **批量下载（CreateZipForDownload）** | ✅ 是（基类使用） | ✅ 是 | ⚠️ 中 | 路径拼接使用 company_id() |
| **定时任务（Command）** | ❌ 否 | ❌ 否 | ✅ 低 | 手动遍历管理上下文 |
| **同步 Job（dispatchSync）** | 取决于基类 | 不涉及 | ✅ 低 | 继承请求上下文 |

### 7.4 通知任务的具体风险分析

#### 7.4.1 发票/账单通知（`app/Notifications/Sale/Invoice.php`）

**风险链**：
1. 发票模型 `$invoice` 序列化时仅存 ID
2. 反序列化时跳过 Company Scope，可加载任意公司的发票
3. `getTagsReplacement()` 直接读取模型属性（`document_number`, `amount`），无校验
4. 邮件内容可能包含跨公司敏感数据

**已有的间接保护**：
- `$this->invoice->company->name` 通过关联查询，受 Scope 保护。如果 `$invoice` 属于其他公司，`$invoice->company_id = 2`，关联查询 `WHERE company_id = 1 AND id = 2` 会返回 null
- 实际测试：跨公司发票的 `company` 关联会返回 `null`，模板中可能报错或显示空白

#### 7.4.2 导出完成通知（`app/Notifications/Common/ExportCompleted.php`）

```php
class ExportCompleted extends Notification implements ShouldQueue
{
    use Queueable;  // ⚠️ 注意：只使用了 Queueable，未使用 SerializesModels！

    protected $translation;
    protected $file_name;
    protected $download_url;  // 下载 URL 完整传入

    public function toMail($notifiable): MailMessage
    {
        return (new MailMessage)
            ->from(config('mail.from.address'), config('mail.from.name'))  // 依赖上下文
            ->action(trans('general.download'), $this->download_url);      // URL 完整传入，无校验
    }
}
```

**风险**：
- 不使用 `SerializesModels`，所以没有反序列化问题
- 但 `$download_url` 在构造时传入并完整序列化，如果派发时上下文错误，URL 会包含错误的 `company_id`
- `config('mail.from')` 依赖 `makeCurrent()` 正确恢复

### 7.5 导入导出任务的具体风险分析

#### 7.5.1 导出任务的安全边界

`app/Exports/Common/Sheets/Items.php` 示例：
```php
class Items extends Export
{
    public function collection()
    {
        // with() 关联查询 + collectForExport Local Scope
        return Model::with('category')->collectForExport($this->ids);
    }
}
```

**安全分析**：
- `with('category')` 预加载关联，关联查询受 Scope 保护
- `collectForExport` 是 Local Scope，内部调用 `whereIn()`，**不调用 `withoutGlobalScopes()`**
- Company Scope 正常追加 `WHERE company_id = ?`
- **结论**：即使 `$this->ids` 被篡改，跨公司 ID 会被 Company Scope 过滤掉

**例外情况**：
- 如果模型本身设置了 `$tenantable = false`（不常见）
- 如果导出类显式调用 `allCompanies()`（代码中未发现此模式）

#### 7.5.2 导入任务的安全边界

`app/Imports/Common/Sheets/Items.php` 示例：
```php
class Items extends Import
{
    public function map($row): array
    {
        $row = parent::map($row);  // 调用父类 map，设置 company_id
        // ...
        return $row;
    }
}
```

父类 `map()` 方法（`app/Abstracts/Import.php`）：
```php
public function map($row): array
{
    $row['company_id'] = company_id();  // ⚠️ 单点故障
    // ...
    return $row;
}
```

**风险分析**：
- 单点依赖 `company_id()` 返回正确值
- 如果上下文恢复失败：
  - `$row['company_id'] = null`
  - 新记录的 `company_id` 为 `null`，不属于任何公司
  - 查询时 `WHERE company_id = ?` 条件不匹配，数据"消失"
- `hasRow()` 重复检查受 Scope 保护，但在上下文为 null 时，`WHERE company_id = null` 永远不匹配，可能导致重复导入

---

## 八、Overrider：公司级配置覆盖

`app/Utilities/Overrider.php` 在 `makeCurrent()` 中被调用三次，将公司设置同步到 Laravel Config：

### 8.1 `loadSettings()` — 基础配置覆盖

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

### 8.2 `loadCurrencies()` — 货币表加载

从 `currencies` 表（已通过 Global Scope 过滤当前公司）读取所有货币，注入到 `money.currencies.*` config，最后调用 `Currency::setCurrencies()` 刷新内存。

### 8.3 `loadCategoryTypes()` — 分类类型注入

读取公司自定义的收入/支出/物品/其他分类类型，更新到 `search-string` config 中的搜索字段路由参数，确保搜索下拉选项为该公司的分类配置。

---

## 九、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| **队列上下文自动管理（核心）** | `app/Providers/Queue.php` |
| **队列模型反序列化风险分析** | `vendor/laravel/framework/src/Illuminate/Queue/SerializesAndRestoresModelIdentifiers.php` |
| 公司识别中间件 | `app/Http/Middleware/IdentifyCompany.php` |
| Company ID 解析 Trait | `app/Traits/Companies.php` |
| Company 模型（上下文管理） | `app/Models/Common/Company.php` |
| 全局辅助函数（company / company_id） | `app/Utilities/helpers.php` |
| HTTP 中间件注册 | `app/Http/Kernel.php` |
| 路由分组与前缀宏 | `app/Providers/Route.php` |
| Eloquent 全局 Scope | `app/Scopes/Company.php` |
| 租户 Trait（boot 时注册 Scope） | `app/Traits/Tenants.php` |
| Model 基类（默认启用 Tenants + collectForExport） | `app/Abstracts/Model.php` |
| 异步 Job 基类（SerializesModels） | `app/Abstracts/JobShouldQueue.php` |
| 同步 Job 基类 | `app/Abstracts/Job.php` |
| Job 派发 Trait（dispatch / dispatchSync） | `app/Traits/Jobs.php` |
| Notification 基类（ShouldQueue） | `app/Abstracts/Notification.php` |
| Export 抽象类（ShouldQueue + Exportable） | `app/Abstracts/Export.php` |
| Import 抽象类（ShouldQueue + Importable） | `app/Abstracts/Import.php` |
| 队列请求集合（替代不可序列化的 Request） | `app/Utilities/QueueCollection.php` |
| 配置覆盖器（设置/货币/分类） | `app/Utilities/Overrider.php` |
| 公司切换控制器 | `app/Http/Controllers/Common/Companies.php` |
| 用户-公司关系校验 Trait | `app/Traits/Users.php` |
| 模块注册 Trait | `app/Traits/Modules.php` |
| Admin 菜单构建中间件 | `app/Http/Middleware/AdminMenu.php` |
| Portal 菜单构建中间件 | `app/Http/Middleware/PortalMenu.php` |
| Admin 默认菜单项监听器 | `app/Listeners/Menu/ShowInAdmin.php` |
| Artisan 调度注册 | `app/Console/Kernel.php` |
| 队列连接配置 | `config/queue.php` |
| 重复账单 Cron 任务（遍历模式） | `app/Console/Commands/RecurringCheck.php` |
| 发票催款 Cron 任务（遍历模式） | `app/Console/Commands/InvoiceReminder.php` |
| 账单催款 Cron 任务（遍历模式） | `app/Console/Commands/BillReminder.php` |
| 安装模块 Job（显式 company_id 属性） | `app/Jobs/Install/EnableModule.php` |
| 发票 Notification（关联隐式获取） | `app/Notifications/Sale/Invoice.php` |
| 导出完成 Notification（无 SerializesModels） | `app/Notifications/Common/ExportCompleted.php` |
| 批量下载 Job（company_id() 直接使用） | `app/Jobs/Common/CreateZipForDownload.php` |
| 导出 Sheet 示例（受 Scope 保护） | `app/Exports/Common/Sheets/Items.php` |
| 导入 Sheet 示例（显式设置 company_id） | `app/Imports/Common/Sheets/Items.php` |
| 登出清理 session | `app/Listeners/Auth/Logout.php` |
| 事件-监听器映射表 | `app/Providers/Event.php` |
