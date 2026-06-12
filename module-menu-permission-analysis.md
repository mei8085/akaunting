# 模块菜单注入与权限判断深度分析

## 目录

1. [核心问题解答](#核心问题解答)
2. [公司上下文与模块加载流程](#公司上下文与模块加载流程)
3. [模块服务提供者注册与监听器激活](#模块服务提供者注册与监听器激活)
4. [模块别名的全链路影响](#模块别名的全链路影响)
5. [菜单显示完整调用链](#菜单显示完整调用链)
6. [关键代码追踪](#关键代码追踪)

---

## 核心问题解答

### Q: 模块启用后，菜单是怎么"真正"显示到主界面的？

**一句话总结**：模块启用只更新数据库状态，**菜单显示发生在每次 HTTP 请求时**——请求经过公司识别中间件后，动态加载该公司已启用模块的服务提供者和事件监听器，然后菜单中间件构建菜单时触发事件，模块监听器响应并注入菜单项，最后视图渲染菜单 HTML。

### Q: 模块别名（alias）在哪里影响了什么？

模块别名是贯穿始终的关键标识，影响：
- **路由**：URL 前缀、路由命名、控制器命名空间
- **权限**：权限名称自动生成规则
- **菜单**：菜单项的路由关联、图标路径
- **状态**：数据库中模块的启用状态

### Q: 为什么普通的 `Route::get` 写法无法自动获得 `blog.posts.index` 这样的命名路由？

**核心原因**：Laravel 的路由命名机制中，只有 `Route::resource()` 会**自动生成**资源名称的命名路由，而 `Route::get()` / `Route::post()` 等单个路由方法**必须显式调用 `->name()`** 才能获得路由名。路由组的 `as` 属性只是"前缀叠加"，不是"自动命名"。

模块路由宏 `Route::admin()` 只会为组内路由设置 `as => 'blog.'` 前缀，不会自动为 `Route::get()` 生成名称。

**实际命名规则**：
- `Route::get('/', 'Posts@index')->name('posts.index')` → 路由名：`blog.posts.index`（显式命名 + 前缀叠加）
- `Route::resource('posts', 'Posts')` → 自动生成 `blog.posts.index`、`blog.posts.create` 等

### Q: 模块设置页的读写权限是如何从模块声明一路传递到控制器判断的？

**完整链路**：`module.json` 声明 `settings` 数组 → 安装时 `attachModuleSettingPermissions()` 创建 `read-{alias}-settings` / `update-{alias}-settings` 权限 → 分配给 admin/manager 角色 → `App\Http\Livewire\Menu\Settings` 按 alias 注入设置菜单并检查权限 → 请求到达 `App\Http\Controllers\Settings\Modules` 控制器时，从 URL segment 获取 alias，拼接出 `read-{alias}-settings` / `update-{alias}-settings` 权限进行校验。

这条链路的关键是**模块声明、权限创建、菜单注入、控制器校验四处都围绕同一个 alias 进行**，确保权限名称前后一致。

---

## 公司上下文与模块加载流程

### 1. 请求生命周期中的关键中间件顺序

`admin` 路由组的中间件执行顺序（[Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Http/Kernel.php#L76-L88)）：

```
请求到达
    ↓
web 中间件组 (session, csrf, 语言, 防火墙等)
    ↓
auth 中间件 (用户认证)
    ↓
company.identify 中间件  ← 【关键】识别公司 + 加载模块
    ↓
bindings 中间件
    ↓
read.only 中间件
    ↓
wizard.redirect 中间件
    ↓
menu.admin 中间件  ← 【关键】构建菜单
    ↓
permission:read-admin-panel 中间件
    ↓
plan.limits 中间件
    ↓
module.subscription 中间件
    ↓
控制器
```

### 2. 公司识别中间件详解

**核心文件**：[IdentifyCompany.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Http/Middleware/IdentifyCompany.php)

```php
public function handle($request, Closure $next, ...$guards)
{
    $this->request = $request;

    // 步骤1: 从 URL 中解析 company_id
    $company_id = $this->getCompanyId();

    // 步骤2: 验证用户是否有权访问该公司
    if ($this->request->isNotSigned($company_id) && $this->isNotUserCompany($company_id)) {
        throw new AuthenticationException(...);
    }

    // 步骤3: 设置当前公司上下文
    $company = company($company_id);
    $company->makeCurrent();

    // 步骤4: 【核心】加载该公司的已启用模块
    $this->registerModules();  // ← 模块服务提供者 + 事件监听器在此激活

    // 步骤5: 配置文件系统路径、URL 默认参数等
    config(['filesystems.disks.' . config('filesystems.default') . '.url' => ...]);
    app('url')->defaults(['company_id' => $company_id]);

    return $next($this->request);
}
```

**关键点**：
- 模块是**按公司**独立启用的（多租户）
- 每次请求都会重新注册模块（不是启动时一次性加载）
- `registerModules()` 是模块被"激活"的入口点

### 3. 模块注册的内部调用链

```
IdentifyCompany::handle()
    └── $this->registerModules()          // [Traits/Modules.php]
        └── app(ActivatorInterface::class)->register()
            └── ModuleActivator::register()  // [Utilities/ModuleActivator.php]
                ├── $this->load()         // 从数据库读取该公司的模块状态
                │   └── getStatusesByCompany()
                │       └── readDatabase()  // 查 modules 表
                └── app()->register(Bootstrap::class, true)
                    └── 加载所有已启用模块的服务提供者
                        ├── Event 服务提供者 → 发现并注册事件监听器
                        ├── Main 服务提供者 → 其他业务逻辑
                        └── ...
```

**关键代码**：[ModuleActivator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/ModuleActivator.php#L173-L178)

```php
public function register(): void
{
    $this->load();          // 加载模块状态

    app()->register(\Akaunting\Module\Providers\Bootstrap::class, true);
}
```

### 4. 模块状态数据来源

模块的启用/禁用状态存储在 `modules` 表中，按公司区分：

| 字段 | 说明 |
|------|------|
| `company_id` | 公司 ID（多租户隔离） |
| `alias` | 模块别名（如 `blog`, `offline-payments`） |
| `enabled` | 是否启用（布尔值） |
| `created_from` | 创建来源 |

**读取逻辑**：[ModuleActivator::readDatabase()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/ModuleActivator.php#L133-L164)

```php
public function readDatabase(): array
{
    // ...
    $modules = Model::companyId($this->company_id)
        ->pluck('enabled', 'alias')
        ->toArray();

    // 还会检查订阅状态
    foreach ($modules as $alias => $enabled) {
        $subscription = $this->getSubscription($alias);
        // 订阅过期可能导致模块被禁用
    }

    return $modules;
}
```

---

## 模块服务提供者注册与监听器激活

### 1. 模块的服务提供者配置

每个模块在 `module.json` 中声明自己的服务提供者：

**模板文件**：[json.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/json.stub)

```json
{
    "alias": "blog",
    "providers": [
        "Modules\\Blog\\Providers\\Event",
        "Modules\\Blog\\Providers\\Main"
    ]
}
```

当 `Bootstrap` 服务提供者被注册时，它会遍历所有已启用模块，并逐个注册它们的服务提供者。

### 2. 事件监听器的自动发现

模块的 `Event` 服务提供者启用了**事件自动发现**机制：

**模板文件**：[event.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/providers/event.stub)

```php
class Event extends Provider
{
    public function shouldDiscoverEvents()
    {
        return true;  // 启用自动发现
    }

    protected function discoverEventsWithin()
    {
        return [
            __DIR__ . '/../Listeners',  // 扫描 Listeners 目录
        ];
    }
}
```

**工作原理**：
- 当 `Event` 服务提供者被注册时，Laravel 会自动扫描 `Listeners` 目录
- 通过反射分析每个 Listener 类的 `handle()` 方法参数
- 根据参数类型自动推断该监听器监听的事件
- 自动完成事件-监听器的绑定

**以菜单监听器为例**：
```php
// Modules/Blog/Listeners/Menu/ShowInAdmin.php
class ShowInAdmin
{
    public function handle(\App\Events\Menu\AdminCreated $event)
    {
        // 因为 handle 方法参数是 AdminCreated 类型
        // 所以 Laravel 自动将此监听器绑定到 AdminCreated 事件
    }
}
```

### 3. 监听器激活时机

| 阶段 | 时机 | 说明 |
|------|------|------|
| 注册 | `company.identify` 中间件 | 模块的 Event 服务提供者被注册，监听器被发现 |
| 触发 | `menu.admin` 中间件 | 菜单构建时触发 `AdminCreated` 事件 |
| 执行 | 事件触发后 | 所有绑定的监听器按顺序执行，注入菜单项 |

### 4. 与核心监听器的关系

核心模块的菜单监听器在 `EventServiceProvider` 中显式注册：

**注册位置**：[Event.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Event.php#L96-L98)

```php
protected $listen = [
    \App\Events\Menu\AdminCreated::class => [
        \App\Listeners\Menu\ShowInAdmin::class,  // 核心菜单监听器
    ],
    // ...
];
```

**模块监听器 vs 核心监听器**：
- 核心监听器：在应用启动时就注册，始终存在
- 模块监听器：在 `company.identify` 阶段动态注册，仅当模块启用时存在
- 执行顺序：核心监听器先执行（先注册先执行），模块监听器后执行

---

## 模块别名的全链路影响

模块别名（alias）是模块的唯一标识，采用 kebab-case 命名（如 `blog`, `offline-payments`）。它贯穿了路由、控制器、权限、菜单等各个层面。

### 1. 对路由的影响

#### 1.1 路由宏定义

系统提供了 `Route::admin()` / `Route::portal()` 等宏来简化模块路由定义：

**定义位置**：[Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L40-L113)

```php
Facade::macro('module', function ($alias, $routes, $attrs) {
    $attributes = [
        'middleware' => $attrs['middleware'],
    ];

    // 1. 控制器命名空间: Modules\Blog\Http\Controllers
    $attributes['namespace'] = 'Modules\\' . module($alias)->getStudlyName() . '\Http\Controllers';

    // 2. URL 前缀: {company_id}/blog
    $attributes['prefix'] = '{company_id}/' . $alias;

    // 3. 路由名称前缀: blog.
    $attributes['as'] = $alias . '.';

    return Facade::group($attributes, $routes);
});
```

#### 1.2 路由宏的完整代码分析

`Route::admin()` 等宏是在 `Route` 服务提供者的 `register()` 方法中定义的。让我们逐个分析：

**基础宏：Route::module()** — [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L40-L73)

```php
Facade::macro('module', function ($alias, $routes, $attrs) {
    // 1. 中间件：从 $attrs 中获取（必传参数）
    $attributes = [
        'middleware' => $attrs['middleware'],
    ];

    // 2. 控制器命名空间
    //    如果 $attrs['namespace'] = null，表示不设置命名空间
    //    如果显式传入，则使用传入值
    //    否则自动生成：Modules\{StudlyName}\Http\Controllers
    if (isset($attrs['namespace'])) {
        if (!is_null($attrs['namespace'])) {
            $attributes['namespace'] = $attrs['namespace'];
        }
    } else {
        // module($alias)->getStudlyName() 将 blog 转为 Blog
        $attributes['namespace'] = 'Modules\\' . module($alias)->getStudlyName() . '\Http\Controllers';
    }

    // 3. URL 前缀
    //    如果 $attrs['prefix'] = null，表示不加前缀
    //    如果显式传入，则用传入值加上公司ID前缀
    //    否则自动生成：{company_id}/{alias}
    if (isset($attrs['prefix'])) {
        if (!is_null($attrs['prefix'])) {
            $attributes['prefix'] = '{company_id}/' . $attrs['prefix'];
        }
    } else {
        $attributes['prefix'] = '{company_id}/' . $alias;
    }

    // 4. 路由命名前缀（as）
    //    规则与 prefix 类似
    //    默认：{alias}.
    if (isset($attrs['as'])) {
        if (!is_null($attrs['as'])) {
            $attributes['as'] = $attrs['as'];
        }
    } else {
        $attributes['as'] = $alias . '.';
    }

    // 创建路由组
    return Facade::group($attributes, $routes);
});
```

**衍生宏：Route::admin()** — [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L75-L79)

```php
Facade::macro('admin', function ($alias, $routes, $attributes = []) {
    // 只传 middleware = admin，其他属性使用默认值
    //   namespace: Modules\Blog\Http\Controllers
    //   prefix:    {company_id}/blog
    //   as:        blog.
    return Facade::module($alias, $routes, array_merge([
        'middleware' => 'admin',
    ], $attributes));
});
```

**衍生宏：Route::portal()** — [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L89-L95)

```php
Facade::macro('portal', function ($alias, $routes, $attributes = []) {
    // portal 路由显式指定了 prefix 和 as：
    //   prefix: {company_id}/portal/blog
    //   as:     portal.blog.
    return Facade::module($alias, $routes, array_merge([
        'middleware'    => 'portal',
        'prefix'        => 'portal/' . $alias,
        'as'            => 'portal.' . $alias . '.',
    ], $attributes));
});
```

**衍生宏：Route::api()** — [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L105-L113)

```php
Facade::macro('api', function ($alias, $routes, $attributes = []) {
    // API 路由有独立的命名空间和域名配置
    return Facade::module($alias, $routes, array_merge([
        'namespace'     => 'Modules\\' . module($alias)->getStudlyName() . '\Http\Controllers\Api',
        'domain'        => config('api.domain'),
        'middleware'    => config('api.middleware'),
        'prefix'        => config('api.prefix') ? config('api.prefix') . '/' . $alias : $alias,
        'as'            => 'api.' . $alias . '.',
    ], $attributes));
});
```

**衍生宏：Route::signed()** — [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L97-L103)

```php
Facade::macro('signed', function ($alias, $routes, $attributes = []) {
    return Facade::module($alias, $routes, array_merge([
        'middleware'    => 'signed',
        'prefix'        => 'signed/' . $alias,
        'as'            => 'signed.' . $alias . '.',
    ], $attributes));
});
```

#### 1.3 所有路由宏的属性对比

| 宏方法 | middleware | namespace (默认) | prefix (默认) | as (默认) |
|--------|-----------|------------------|---------------|-----------|
| `Route::admin()` | `admin` | `Modules\{Name}\Http\Controllers` | `{company_id}/{alias}` | `{alias}.` |
| `Route::portal()` | `portal` | 同上 | `{company_id}/portal/{alias}` | `portal.{alias}.` |
| `Route::preview()` | `preview` | 同上 | `{company_id}/preview/{alias}` | `preview.{alias}.` |
| `Route::signed()` | `signed` | 同上 | `{company_id}/signed/{alias}` | `signed.{alias}.` |
| `Route::api()` | `config('api.middleware')` | `Modules\{Name}\Http\Controllers\Api` | `{api_prefix}/{alias}` | `api.{alias}.` |

#### 1.4 核心路由 vs 模块路由对比

让我们对比核心系统路由和模块路由的定义方式：

**核心系统路由**（[admin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/routes/admin.php)）：

```php
// routes/admin.php —— 核心路由
// 这些路由在 RouteServiceProvider::mapAdminRoutes() 中加载
Route::prefix('{company_id}')
    ->middleware('admin')
    ->namespace($this->namespace)    // App\Http\Controllers
    ->group(base_path('routes/admin.php'));

// 里面的路由定义：
Route::group(['prefix' => 'common'], function () {
    Route::resource('items', 'Common\Items');
    // URL:       /{company_id}/common/items
    // 路由名:    items.index
    // 命名空间:  App\Http\Controllers\Common\Items
});

Route::group(['prefix' => 'sales'], function () {
    Route::resource('invoices', 'Sales\Invoices');
    // URL:       /{company_id}/sales/invoices
    // 路由名:    invoices.index
    // 命名空间:  App\Http\Controllers\Sales\Invoices
});
```

**模块路由**（[admin.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/routes/admin.stub)）：

```php
// modules/Blog/Routes/admin.php —— 模块路由
Route::admin('blog', function () {
    Route::get('/', 'Main@index');
    Route::resource('posts', 'Posts');
    // URL:       /{company_id}/blog/posts
    // 路由名:    blog.posts.index
    // 命名空间:  Modules\Blog\Http\Controllers\Posts
});
```

#### 1.5 模块路由示例

模块的 admin.php 路由文件：
```php
// modules/Blog/Routes/admin.php
Route::admin('blog', function () {
    Route::get('/', 'Posts@index');    // 路由名: blog.posts.index
    Route::get('/{id}', 'Posts@show'); // 路由名: blog.posts.show
    Route::post('/', 'Posts@store');   // 路由名: blog.posts.store
});
```

**最终生成的路由**：

| 属性 | 值 | 示例 |
|------|-----|------|
| URL | `{company_id}/{alias}/...` | `/1/blog/posts/1` |
| 命名空间 | `Modules\{StudlyName}\Http\Controllers` | `Modules\Blog\Http\Controllers` |
| 路由名 | `{alias}.{controller}.{action}` | `blog.posts.index` |

#### 1.6 路由命名中的模块别名影响

以 alias = `offline-payments`（kebab-case）为例：

```php
Route::admin('offline-payments', function () {
    Route::get('/', 'Settings@edit');
});
```

生成结果：
- **URL**: `/{company_id}/offline-payments/`
- **路由名**: `offline-payments.settings.edit`
- **控制器命名空间**: `Modules\OfflinePayments\Http\Controllers\Settings`
- **StudlyName 转换**: `offline-payments` → `OfflinePayments`（在命名空间中使用）

路由名在菜单项和视图中使用：
```php
// 菜单监听器中
$menu->route('offline-payments.settings.edit', '离线支付', [], 60, ...);

// 视图中
{{ route('offline-payments.settings.edit') }}
```

### 2. 对控制器权限的影响

#### 2.1 自动权限生成原理

系统有两个控制器基类：`Controller`（Web）和 `ApiController`（API），它们都在构造函数中调用 `assignPermissionsToController()` 来自动分配权限：

**Web 控制器基类**：[Controller.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Http/Controller.php#L22-L32)

```php
use App\Traits\Permissions;

abstract class Controller extends BaseController
{
    use Permissions;  // 引入权限 trait

    public function __construct()
    {
        $this->assignPermissionsToController();  // 自动分配权限
    }
}
```

**API 控制器基类**：[ApiController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Http/ApiController.php#L17-L27)

```php
use App\Traits\Permissions;

abstract class ApiController extends BaseController
{
    use Permissions;  // 同样引入权限 trait

    public function __construct()
    {
        $this->assignPermissionsToController();  // 同样自动分配权限
    }
}
```

这意味着**所有继承自这两个基类的控制器都会自动拥有权限检查**，不需要手动配置。

#### 2.2 assignPermissionsToController() 完整代码分析

**完整方法**：[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L425-L500)

```php
public function assignPermissionsToController()
{
    // 【特殊情况】控制台运行时不检查权限（CLI 命令、队列 Job 等）
    if (app()->runningInConsole()) {
        return;
    }

    // 【分支一】API 路由中对 contacts/documents 的特殊处理
    // 这些通用 API 端点需要根据请求参数 type 来判断是哪个模块的资源
    $table = request_is_api() ? request()->segment(2) : '';

    if (in_array($table, ['contacts', 'documents'])) {
        $controller = '';

        // 从查询字符串 search=type:customer 中解析 type
        $type = $this->getSearchStringValue('type');

        if (! empty($type)) {
            // 从配置文件读取类型对应的模块别名、分组、权限前缀
            // 配置路径: type.contact.customer.alias / group / permission.prefix
            $alias = config('type.' . Str::singular($table) . '.' . $type . '.alias');
            $group = config('type.' . Str::singular($table) . '.' . $type . '.group');
            $prefix = config('type.' . Str::singular($table) . '.' . $type . '.permission.prefix');

            // 如果有模块别名，加上模块前缀 (如: offline-payments-)
            if (! empty($alias)) {
                $controller .= $alias . '-';
            }

            // 如果有分组（子目录），加上分组前缀 (如: sales-)
            if (! empty($group)) {
                $controller .= $group . '-';
            }

            // 加上权限前缀 (如: invoices)
            $controller .= $prefix;
        }
        // 示例结果:
        //   类型 customer → alias=null, group=sales, prefix=customers → sales-customers
        //   类型 vendor   → alias=null, group=purchases, prefix=vendors → purchases-vendors
    } else {
        // 【分支二】普通控制器：从路由的控制器命名空间解析权限前缀

        // 步骤1: 获取当前路由对应的控制器完整类名
        // $route->getAction()['uses'] 类似: "App\Http\Controllers\Sales\Invoices@index"
        // 先 explode('@', ...)[0] 得到类名部分，再按 '\' 分割
        $route = app(Route::class);
        $arr = array_reverse(explode('\\', explode('@', $route->getAction()['uses'])[0]));

        $controller = '';

        // 步骤2: 判断是否是模块控制器（命名空间包含 Modules）
        //
        // 命名空间层级分析:
        // 模块控制器:     Modules\Blog\Http\Controllers\Posts
        //                reverse → [Posts, Controllers, Http, Blog, Modules]
        //                                                 arr[3] arr[4]
        //
        // 模块Portal控制器: Modules\Blog\Http\Controllers\Portal\Posts
        //                  reverse → [Posts, Portal, Controllers, Http, Blog, Modules]
        //                                                       arr[4] arr[5]
        //
        // 核心App控制器:    App\Http\Controllers\Sales\Invoices
        //                  reverse → [Invoices, Sales, Controllers, Http, App]
        //
        // 核心API控制器:    App\Http\Controllers\Api\Common\Items
        //                  reverse → [Items, Common, Api, Controllers, Http, App]
        //

        // 模块识别逻辑（两种深度都要检测）
        if (isset($arr[3]) && isset($arr[4])) {
            // 情况1: Modules\Blog\Http\Controllers\Posts
            // arr[4] = Modules, arr[3] = Blog
            if (strtolower($arr[4]) == 'modules') {
                $controller .= Str::kebab($arr[3]) . '-';  // Blog → blog-
            }
            // 情况2: Modules\Blog\Http\Controllers\Portal\Posts
            // arr[5] = Modules, arr[4] = Blog
            elseif (isset($arr[5]) && (strtolower($arr[5]) == 'modules')) {
                $controller .= Str::kebab($arr[4]) . '-';  // Blog → blog-
            }
        }

        // 步骤3: 添加子目录名（排除 api 和 controllers）
        // arr[1] 是控制器前的子目录
        //   - 如果 arr[1] 是 'api' 或 'controllers'，说明没有子目录，跳过
        //   - 否则，arr[1] 是子目录名，需要加上
        if (! in_array(strtolower($arr[1]), ['api', 'controllers'])) {
            $controller .= Str::kebab($arr[1]) . '-';
        }

        // 步骤4: 添加控制器类名
        // arr[0] 始终是控制器类名本身
        $controller .= Str::kebab($arr[0]);

        // 步骤5: 跳过白名单（这些控制器不需要权限检查）
        $skip = ['portal-dashboard'];
        if (in_array($controller, $skip)) {
            return;
        }

        // 注释中的完整示例:
        // App\Http\Controllers\FooBar                  -->> foo-bar
        // App\Http\Controllers\FooBar\Main             -->> foo-bar-main
        // Modules\Blog\Http\Controllers\Posts          -->> blog-posts
        // Modules\Blog\Http\Controllers\Portal\Posts   -->> blog-portal-posts
    }

    // 【最后一步】将 CRUD 权限分配给控制器方法
    // 使用 Laratrust 的 permission 中间件
    $this->middleware('permission:create-' . $controller)->only('create', 'store', 'duplicate', 'import');
    $this->middleware('permission:read-'   . $controller)->only('index', 'show', 'edit', 'export');
    $this->middleware('permission:update-' . $controller)->only('update', 'enable', 'disable');
    $this->middleware('permission:delete-' . $controller)->only('destroy');
}
```

#### 2.3 不同命名空间下控制器的权限前缀解析详解

让我们通过多个实际存在的控制器示例来完整演示解析过程：

##### 示例 1：核心控制器（无子目录）

```
控制器类: App\Http\Controllers\Modules\My
命名空间分割: ['App', 'Http', 'Controllers', 'Modules', 'My']
反转后 $arr:
    $arr[0] = My
    $arr[1] = Modules
    $arr[2] = Controllers
    $arr[3] = Http
    $arr[4] = App

解析步骤:
1. 检测模块: $arr[4] = 'app'，不是 'modules' → 不加模块前缀
2. 添加子目录: $arr[1] = 'modules'，不在排除列表['api','controllers']中
             → 'modules-'
3. 添加控制器名: 'my'
4. 最终权限前缀: modules-my

对应权限:
  create-modules-my, read-modules-my, update-modules-my, delete-modules-my
```

##### 示例 2：核心控制器（有子目录 Sales）

```
控制器类: App\Http\Controllers\Sales\Invoices
命名空间分割: ['App', 'Http', 'Controllers', 'Sales', 'Invoices']
反转后 $arr:
    $arr[0] = Invoices
    $arr[1] = Sales
    $arr[2] = Controllers
    $arr[3] = Http
    $arr[4] = App

解析步骤:
1. 检测模块: $arr[4] = 'app' → 不加模块前缀
2. 添加子目录: $arr[1] = 'sales' → 'sales-'
3. 添加控制器名: 'invoices'
4. 最终权限前缀: sales-invoices

对应权限:
  create-sales-invoices, read-sales-invoices, update-sales-invoices, delete-sales-invoices
```

##### 示例 3：核心 API 控制器

```
控制器类: App\Http\Controllers\Api\Common\Items
命名空间分割: ['App', 'Http', 'Controllers', 'Api', 'Common', 'Items']
反转后 $arr:
    $arr[0] = Items
    $arr[1] = Common
    $arr[2] = Api
    $arr[3] = Controllers
    $arr[4] = Http
    $arr[5] = App

解析步骤:
1. 检测模块: $arr[4] = 'http'，$arr[5] = 'app' → 都不是 'modules' → 不加
2. 添加子目录: $arr[1] = 'common' → 'common-'
3. 添加控制器名: 'items'
4. 最终权限前缀: common-items

对应权限:
  create-common-items, read-common-items, update-common-items, delete-common-items
```

##### 示例 4：模块控制器（无子目录）

```
控制器类: Modules\Blog\Http\Controllers\Posts
命名空间分割: ['Modules', 'Blog', 'Http', 'Controllers', 'Posts']
反转后 $arr:
    $arr[0] = Posts
    $arr[1] = Controllers
    $arr[2] = Http
    $arr[3] = Blog
    $arr[4] = Modules

解析步骤:
1. 检测模块: $arr[4] = 'modules' → Str::kebab($arr[3]) = 'blog-'
2. 添加子目录: $arr[1] = 'controllers'，在排除列表中 → 不加
3. 添加控制器名: 'posts'
4. 最终权限前缀: blog-posts

对应权限:
  create-blog-posts, read-blog-posts, update-blog-posts, delete-blog-posts
```

##### 示例 5：模块控制器（有子目录 Portal）

```
控制器类: Modules\Blog\Http\Controllers\Portal\Posts
命名空间分割: ['Modules', 'Blog', 'Http', 'Controllers', 'Portal', 'Posts']
反转后 $arr:
    $arr[0] = Posts
    $arr[1] = Portal
    $arr[2] = Controllers
    $arr[3] = Http
    $arr[4] = Blog
    $arr[5] = Modules

解析步骤:
1. 检测模块: $arr[5] = 'modules' → Str::kebab($arr[4]) = 'blog-'
2. 添加子目录: $arr[1] = 'portal'，不在排除列表中 → 'portal-'
3. 添加控制器名: 'posts'
4. 最终权限前缀: blog-portal-posts

对应权限:
  create-blog-portal-posts, read-blog-portal-posts,
  update-blog-portal-posts, delete-blog-portal-posts
```

##### 示例 6：带连字符的模块别名（如 offline-payments）

```
控制器类: Modules\OfflinePayments\Http\Controllers\Settings
命名空间分割: ['Modules', 'OfflinePayments', 'Http', 'Controllers', 'Settings']
反转后 $arr:
    $arr[0] = Settings
    $arr[1] = Controllers
    $arr[2] = Http
    $arr[3] = OfflinePayments
    $arr[4] = Modules

解析步骤:
1. 检测模块: $arr[4] = 'modules' → Str::kebab('OfflinePayments') = 'offline-payments-'
2. 添加子目录: $arr[1] = 'controllers' → 排除，不加
3. 添加控制器名: 'settings'
4. 最终权限前缀: offline-payments-settings

对应权限:
  create-offline-payments-settings, read-offline-payments-settings,
  update-offline-payments-settings, delete-offline-payments-settings
```

#### 2.4 控制器 → 权限映射汇总表

| 控制器完整类名 | 权限前缀 | 说明 |
|---------------|---------|------|
| `App\Http\Controllers\Common\Items` | `common-items` | 核心通用控制器 |
| `App\Http\Controllers\Sales\Invoices` | `sales-invoices` | 核心销售控制器 |
| `App\Http\Controllers\Purchases\Bills` | `purchases-bills` | 核心采购控制器 |
| `App\Http\Controllers\Banking\Accounts` | `banking-accounts` | 核心银行控制器 |
| `App\Http\Controllers\Modules\My` | `modules-my` | 核心模块管理控制器 |
| `App\Http\Controllers\Api\Common\Items` | `common-items` | 核心 API 控制器 |
| `App\Http\Controllers\Portal\Dashboard` | (跳过) | 白名单，不做权限检查 |
| `Modules\Blog\Http\Controllers\Posts` | `blog-posts` | 模块后台控制器 |
| `Modules\Blog\Http\Controllers\Portal\Posts` | `blog-portal-posts` | 模块门户控制器 |
| `Modules\OfflinePayments\Http\Controllers\Settings` | `offline-payments-settings` | 含连字符别名的模块 |
| `Modules\OfflinePayments\Http\Controllers\Portal\Payments` | `offline-payments-portal-payments` | 含连字符别名的模块门户 |

#### 2.5 方法级权限分配

**代码位置**：[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L495-L500)

```php
// create 权限 -> create, store, duplicate, import 方法
$this->middleware('permission:create-' . $controller)
    ->only('create', 'store', 'duplicate', 'import');

// read 权限 -> index, show, edit, export 方法
$this->middleware('permission:read-' . $controller)
    ->only('index', 'show', 'edit', 'export');

// update 权限 -> update, enable, disable 方法
$this->middleware('permission:update-' . $controller)
    ->only('update', 'enable', 'disable');

// delete 权限 -> destroy 方法
$this->middleware('permission:delete-' . $controller)
    ->only('destroy');
```

方法与权限的映射关系是固定的，意味着：
- 如果你的控制器有一个 `index()` 方法，用户需要 `read-{prefix}` 权限才能访问
- 如果你的控制器有一个 `store()` 方法，用户需要 `create-{prefix}` 权限才能访问
- 如果你的控制器自定义了方法名（如 `approve()`），则不会被自动分配权限，需要手动处理

**完整示例**：`Modules\Blog\Http\Controllers\Posts` 控制器的权限

| 权限 | 对应方法 | 说明 |
|------|---------|------|
| `create-blog-posts` | create, store, duplicate, import | 创建、保存、复制、导入 |
| `read-blog-posts` | index, show, edit, export | 列表、详情、编辑、导出 |
| `update-blog-posts` | update, enable, disable | 更新、启用、禁用 |
| `delete-blog-posts` | destroy | 删除 |

### 3. 对菜单的影响

#### 3.1 菜单项的权限检查

菜单项在添加前会检查用户是否有权限：

**检查方法**：[canAccessMenuItem()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L502-L513)

```php
public function canAccessMenuItem($title, $permissions)
{
    $permissions = Arr::wrap($permissions);

    $item = new \stdClass();
    $item->title = $title;
    $item->permissions = $permissions;

    // 触发授权事件，允许其他模块修改权限要求
    event(new \App\Events\Menu\ItemAuthorizing($item));

    // 检查用户是否有任一权限
    return user()->canAny($item->permissions);
}
```

**菜单监听器中的使用**：
```php
// ShowInAdmin.php
$title = trim(trans_choice('general.items', 2));
if ($this->canAccessMenuItem($title, 'read-common-items')) {
    $menu->route('items.index', $title, [], 20, ['icon' => 'inventory_2']);
}
```

#### 3.2 模块菜单图标路径

菜单呈现器会根据路由中的模块别名来查找自定义图标：

**代码位置**：[Menu.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/View/Presenters/Menu.php#L246-L279)

```php
protected function getCustomIcon($item)
{
    // 从路由中解析模块别名
    if (! empty($item->properties['route'])) {
        $route = $item->properties['route'][0];
        $module_alias = explode('.', $route)[0];  // blog.posts.index -> blog
    }

    // 如果是模块，图标路径指向模块目录
    if (module($module_alias) != null) {
        $base_path = 'modules/' . Str::studly($module_alias) . '/Resources/assets/img/icons/';
    }

    // ...
}
```

#### 3.3 菜单项排序

每个菜单项通过 `order` 参数控制显示位置：

| 菜单项 | order 值 |
|--------|----------|
| 仪表盘 | 10 |
| 商品 | 20 |
| 销售 | 30 |
| 采购 | 40 |
| 银行 | 50 |
| 报表 | 60 |
| 应用/模块 | 80 |

模块可以选择合适的 order 值插入到菜单中（如 55 表示放在银行和报表之间）。

---

## 路由命名机制深度解析

### 1. 为什么普通 `Route::get` 无法自动获得 `blog.posts` 命名路由

这是 Laravel 路由系统的核心机制问题，让我们从代码层面深入分析：

#### 1.1 Laravel 路由命名的三种来源

| 路由定义方式 | 是否自动命名 | 命名规则 | 示例 |
|-------------|-------------|---------|------|
| `Route::get()` | ❌ 否 | 无，必须显式调用 `->name()` | `Route::get('/', 'Posts@index')->name('posts.index')` |
| `Route::resource()` | ✅ 是 | `{resource}.{action}` | `Route::resource('posts', 'Posts')` → `posts.index`, `posts.create` 等 |
| `Route::apiResource()` | ✅ 是 | 同上（不含 create/edit） | `posts.index`, `posts.store` 等 |

#### 1.2 路由组 `as` 属性的真实作用

路由组的 `as` 属性只是**名称前缀叠加**，不是**自动命名**。它的工作机制是：

```php
// 路由组设置 as = 'blog.'
Route::group(['as' => 'blog.'], function () {
    
    // 情况1: Route::get 没有显式 name → 最终没有路由名
    // as 前缀无法作用于一个没有名字的路由
    Route::get('/', 'Posts@index');
    // 结果：没有路由名，as 前缀被丢弃

    // 情况2: Route::get 显式设置 name → 前缀叠加
    Route::get('/', 'Posts@index')->name('posts.index');
    // 结果：路由名 = 'blog.' + 'posts.index' = 'blog.posts.index'

    // 情况3: Route::resource 自动生成名称 → 前缀叠加
    Route::resource('posts', 'Posts');
    // 结果：每个资源路由都会自动获得名称，然后叠加前缀
    // 'blog.' + 'posts.index' = 'blog.posts.index'
    // 'blog.' + 'posts.create' = 'blog.posts.create'
    // ...
});
```

**关键代码证明**（来自 [admin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/routes/admin.php)）：

```php
// 核心路由中的实际写法对比
Route::group(['prefix' => 'common'], function () {
    
    // Route::get 必须显式调用 ->name()
    Route::get('items/autocomplete', 'Common\Items@autocomplete')
        ->name('items.autocomplete');  // ← 显式命名
    
    Route::get('items/{item}/enable', 'Common\Items@enable')
        ->name('items.enable');  // ← 显式命名
    
    // Route::resource 自动生成名称，不需要 ->name()
    Route::resource('items', 'Common\Items');
    // 自动生成: items.index, items.create, items.store, 
    //          items.show, items.edit, items.update, items.destroy
});
```

#### 1.3 模块路由宏中的 `as` 属性

`Route::admin()` 宏设置了 `as => $alias . '.'`，但它不会自动为 `Route::get()` 生成名称：

**宏定义**：[Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php#L67-L72)

```php
// as 属性只是设置前缀，不会自动命名
if (isset($attrs['as'])) {
    if (!is_null($attrs['as'])) {
        $attributes['as'] = $attrs['as'];  // 只是设置值，不做额外处理
    }
} else {
    $attributes['as'] = $alias . '.';  // 默认: blog.
}
```

#### 1.4 模块路由的正确写法对比

**模块路由模板**：[admin.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/routes/admin.stub)

```php
// modules/Blog/Routes/admin.php
Route::admin('blog', function () {
    
    // ❌ 错误写法：没有路由名，as 前缀无法生效
    Route::get('/', 'Main@index');
    // 访问可以用 URL，但无法用 route() 函数生成 URL
    
    // ✅ 正确写法1：显式命名
    Route::get('/', 'Main@index')->name('main.index');
    // 路由名: blog.main.index
    
    // ✅ 正确写法2：使用 Route::resource 自动命名
    Route::resource('posts', 'Posts');
    // 自动生成: blog.posts.index, blog.posts.create, ...
});
```

#### 1.5 为什么菜单监听器中用 `route()` 不会报错

菜单监听器中使用 `$menu->route('blog.posts.index', ...)` 时，Laravel 会检查路由是否存在：

```php
// ShowInAdmin.php 中
$menu->route('blog.posts.index', $title, [], 55, ['icon' => 'article']);
```

如果路由名不存在（比如忘记给 `Route::get()` 加 `->name()`），会抛出 `RouteNotFoundException`。

### 2. `Route::resource()` 的自动命名规则

`Route::resource()` 会为标准 CRUD 操作自动生成以下命名路由：

| HTTP 方法 | URL | 控制器方法 | 路由名 | 权限 |
|----------|-----|-----------|--------|------|
| GET | `/posts` | index | `posts.index` | read |
| GET | `/posts/create` | create | `posts.create` | create |
| POST | `/posts` | store | `posts.store` | create |
| GET | `/posts/{post}` | show | `posts.show` | read |
| GET | `/posts/{post}/edit` | edit | `posts.edit` | read |
| PUT/PATCH | `/posts/{post}` | update | `posts.update` | update |
| DELETE | `/posts/{post}` | destroy | `posts.destroy` | delete |

**与权限系统的完美对应**：
- 路由名 `posts.index` → 权限名 `read-blog-posts`（控制器解析得到）
- 路由名 `posts.create` → 权限名 `create-blog-posts`
- 这种一致性是 Akaunting 架构的核心设计之一

---

## 模块设置权限传递链路

模块设置页的权限从声明到控制器判断经过了完整的链路，每一步都依赖统一的命名约定。

### 1. 完整链路总览

```
模块声明 (module.json)
    │  "settings": ["setting1", "setting2"]
    ▼
模块安装 (FinishInstallation)
    │  $this->attachDefaultModulePermissions($module)
    ▼
权限创建 (attachModuleSettingPermissions)
    │  检查 settings 数组非空
    │  createModuleSettingPermission($module, 'read')
    │  createModuleSettingPermission($module, 'update')
    │  生成权限名: read-blog-settings, update-blog-settings
    ▼
权限分配 (attachPermissionsToAdminRoles)
    │  分配给 admin, manager 角色
    ▼
控制器解析 (Settings 控制器)
    │  继承 Controller 基类
    │  __construct 调用 assignPermissionsToController()
    │  从命名空间解析出权限前缀: blog-settings
    ▼
中间件分配
    │  $this->middleware('permission:read-blog-settings')
    │  $this->middleware('permission:update-blog-settings')
    ▼
请求到达
    │  Laratrust 中间件检查用户权限
    ▼
访问允许/拒绝
```

### 2. 步骤 1：模块声明

模块在 `module.json` 中声明自己有设置功能：

**模板文件**：[json.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/json.stub#L15)

```json
{
    "alias": "blog",
    "name": "Blog",
    "settings": [
        "Blog\\Settings\\General",    // 设置类1
        "Blog\\Settings\\Notifications"   // 设置类2
    ],
    "providers": [
        "Modules\\Blog\\Providers\\Event",
        "Modules\\Blog\\Providers\\Main"
    ]
}
```

**关键点**：`settings` 数组不能为空数组（`[]`），否则权限不会被创建。

### 3. 步骤 2：安装时创建权限

模块安装完成后，`FinishInstallation` 监听器会被触发：

**监听器模板**：[install.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/listeners/install.stub#L21-L27)

```php
// Modules/Blog/Listeners/FinishInstallation.php
class FinishInstallation
{
    use Permissions;

    public $alias = 'blog';

    public function handle(Event $event)
    {
        if ($event->alias != $this->alias) {
            return;
        }

        // 方式1：只创建主模块权限
        $this->attachPermissionsToAdminRoles([
            $this->alias . '-main' => 'c,r,u,d',
        ]);

        // 方式2：创建默认模块权限（包含设置、报表、小部件）
        $this->attachDefaultModulePermissions($this->alias);
    }
}
```

### 4. 步骤 3：`attachDefaultModulePermissions` 内部流程

**代码位置**：[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L121-L128)

```php
public function attachDefaultModulePermissions($module, $require = null)
{
    $this->attachModuleReportPermissions($module, $require);

    $this->attachModuleWidgetPermissions($module, $require);

    $this->attachModuleSettingPermissions($module, $require);  // ← 创建设置权限
}
```

### 5. 步骤 4：`attachModuleSettingPermissions` 详解

**代码位置**：[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L180-L198)

```php
public function attachModuleSettingPermissions($module, $require = null)
{
    if (is_string($module)) {
        $module = module($module);  // 获取模块实例
    }

    // 【关键检查】如果 settings 数组为空，直接返回，不创建任何权限
    if (empty($module->get('settings'))) {
        return;
    }

    $permissions = [];

    // 创建设置页面需要的两个权限：读和更新
    // 注意：设置页面通常不需要 create 和 delete 权限
    $permissions[] = $this->createModuleSettingPermission($module, 'read');
    $permissions[] = $this->createModuleSettingPermission($module, 'update');

    // 分配给角色
    $require
            ? $this->attachPermissionsToAllRoles($permissions, $require)
            : $this->attachPermissionsToAdminRoles($permissions);
}
```

### 6. 步骤 5：`createModuleSettingPermission` 生成权限名

**代码位置**：[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L232-L247)

```php
public function createModuleSettingPermission($module, $action)
{
    // 委托给通用方法
    return $this->createModuleControllerPermission($module, $action, 'settings');
}

public function createModuleControllerPermission($module, $action, $controller)
{
    if (is_string($module)) {
        $module = module($module);
    }

    // 【命名规则】{action}-{alias}-{controller}
    // action = 'read', alias = 'blog', controller = 'settings'
    // → 权限名: read-blog-settings
    $name = $action . '-' . $module->getAlias() . '-' . $controller;
    
    // 显示名: Read Blog Settings
    $display_name = Str::title($action) . ' ' . $module->getName() . ' ' . Str::title($controller);

    return $this->createPermission($name, $display_name);
}
```

**生成结果**：
- `read-blog-settings`
- `update-blog-settings`

### 7. 步骤 6：权限分配给角色

**代码位置**：[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L39-L42)

```php
public function attachPermissionsToAdminRoles($permissions)
{
    // 默认分配给 admin 和 manager 角色
    $this->applyPermissionsToRoles($this->getDefaultAdminRoles(), 'attach', $permissions);
}

public function getDefaultAdminRoles($custom = null)
{
    // 先尝试获取 admin 和 manager
    $roles = role_model_class()::whereIn('name', $custom ?? ['admin', 'manager'])->get();

    if ($roles->isNotEmpty()) {
        return $roles;
    }

    // 如果找不到，就分配给所有拥有 read-admin-panel 权限的角色
    return $this->getRoles('read-admin-panel');
}
```

**权限分配的完整调用链**：

```
attachPermissionsToAdminRoles()
    └── applyPermissionsToRoles()
        ├── isActionList() 检查权限值是否是动作列表（如 'c,r,u,d'）
        │   ├── 是 → applyPermissionsByAction()
        │   │       ├── 解析动作: 'c' → 'create', 'r' → 'read', ...
        │   │       ├── 生成权限名: create-blog-main, read-blog-main, ...
        │   │       └── attachPermission() 给角色附加权限
        │   └── 否 → attachPermission() 直接附加
        └── createPermission() 使用 firstOrCreate 确保权限存在
```

**代码位置**：[applyPermissionsByAction()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L359-L373)

```php
public function applyPermissionsByAction($apply, $role, $page, $action_list)
{
    $function = $apply . 'Permission';  // attachPermission 或 detachPermission

    $actions_map = collect($this->getActionsMap());
    // ['c' => 'create', 'r' => 'read', 'u' => 'update', 'd' => 'delete']

    $actions = explode(',', $action_list);  // 'c,r,u,d' → ['c', 'r', 'u', 'd']

    foreach ($actions as $short_action) {
        $action = $actions_map->get($short_action);  // 'c' → 'create'

        $name = $action . '-' . $page;  // 'create' + '-' + 'blog-main' → 'create-blog-main'

        $this->$function($role, $name);
    }
}
```

### 8. 步骤 7：控制器解析权限

模块的 Settings 控制器继承基类，自动解析权限：

```php
// Modules/Blog/Http/Controllers/Settings.php
namespace Modules\Blog\Http\Controllers;

use App\Abstracts\Http\Controller;

class Settings extends Controller
{
    // 构造函数自动调用 assignPermissionsToController()
    
    public function edit()
    {
        // 需要 read-blog-settings 权限
    }
    
    public function update()
    {
        // 需要 update-blog-settings 权限
    }
}
```

**权限解析过程**（[Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php#L458-L493)）：

```
控制器类: Modules\Blog\Http\Controllers\Settings
命名空间分割: ['Modules', 'Blog', 'Http', 'Controllers', 'Settings']
反转后 $arr:
    $arr[0] = Settings
    $arr[1] = Controllers
    $arr[2] = Http
    $arr[3] = Blog
    $arr[4] = Modules

解析步骤:
1. 检测模块: $arr[4] = 'modules' → Str::kebab($arr[3]) = 'blog-'
2. 添加子目录: $arr[1] = 'controllers' → 在排除列表中，不加
3. 添加控制器名: 'settings'
4. 最终权限前缀: blog-settings
```

**生成的权限**：
- `read-blog-settings` → 作用于 index, show, edit, export 方法
- `update-blog-settings` → 作用于 update, enable, disable 方法

### 9. 步骤 8：中间件检查

请求到达时，Laratrust 的 `permission` 中间件检查用户是否有相应权限：

```php
// 自动分配的中间件
$this->middleware('permission:read-blog-settings')
    ->only('index', 'show', 'edit', 'export');

$this->middleware('permission:update-blog-settings')
    ->only('update', 'enable', 'disable');
```

### 10. 命名一致性的关键

这条链路能够顺畅工作的核心是**四个位置使用了完全一致的命名规则**：

| 位置 | 命名规则 | 结果 |
|------|---------|------|
| module.json | `"settings"` 数组非空 | 触发权限创建 |
| 权限创建 | `{action}-{alias}-settings` | `read-blog-settings` |
| 控制器解析 | 从命名空间解析出 `{alias}-settings` | `blog-settings` → `read-blog-settings` |
| 菜单权限检查 | `canAccessMenuItem($title, 'read-blog-settings')` | 检查同一权限 |

### 11. 常见问题与注意事项

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| 设置页面权限总是 403 | module.json 中 settings 为空数组 | 在 settings 数组中添加设置类 |
| 权限创建了但控制器检查不通过 | 控制器类名不是 Settings，或者命名空间不正确 | 确保控制器在 `Http\Controllers\Settings.php` |
| 所有用户都能访问设置页 | 权限没有分配给角色，或者角色没有正确获取 | 检查 `getDefaultAdminRoles()` 是否返回正确角色 |
| 卸载模块后权限残留 | 卸载时没有清理权限 | 在 Uninstall 监听器中调用 `detachPermissionsFromAdminRoles()` |

---

## 菜单显示完整调用链

### 1. 模块启用操作的调用链

```
用户点击"启用"按钮
    │
    ▼
Modules\My 控制器
    │
    ▼
dispatch(new EnableModule($alias, $company_id))
    │
    ▼
EnableModule Job
    │
    ▼
Console::run("module:enable $alias $company_id")
    │
    ▼
EnableCommand::handle()
    ├── $this->prepare()          // 解析参数
    ├── $this->changeRuntime()    // 切换公司上下文
    ├── $this->model->enabled = true; $this->model->save()  // 更新数据库
    ├── $this->createHistory('enabled')  // 创建历史记录
    ├── event(new Enabled($alias, $company_id))  // 触发启用事件
    │   └── ClearCache@handle  // 清除模块缓存
    └── $this->revertRuntime()  // 恢复原来的公司
```

**关键代码**：
- [EnableCommand.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/overrides/akaunting/laravel-module/Commands/EnableCommand.php)
- [EnableModule.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Jobs/Install/EnableModule.php)

**重要**：模块启用只是更新数据库，**不会立即影响当前请求的菜单**。菜单变化发生在**下一次请求**时。

### 2. HTTP 请求中菜单构建的完整调用链

```
浏览器发起请求 (如 GET /1/dashboard)
    │
    ▼
┌─────────────────────────────────┐
│ company.identify 中间件         │
│  (IdentifyCompany)              │
├─────────────────────────────────┤
│ 1. 解析 company_id              │
│ 2. 设置当前公司上下文           │
│ 3. registerModules()            │ ← 【模块激活点】
│    └── ModuleActivator          │
│        ├── 读取模块状态(DB)     │
│        └── 注册 Bootstrap       │
│            └── 加载已启用模块   │
│                ├── Event 提供者 │ ← 事件监听器被发现并注册
│                └── Main 提供者  │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│ menu.admin 中间件               │
│  (AdminMenu)                    │
├─────────────────────────────────┤
│ menu()->create('admin', ...)    │
│  ├── $menu->style('tailwind')   │
│  ├── event(AdminCreating)       │ ← 菜单创建前事件
│  └── event(AdminCreated)        │ ← 【菜单注入点】
│      ├── ShowInAdmin (核心)     │    添加仪表盘、销售、采购等
│      ├── ShowInAdmin (模块A)    │    模块A注入自己的菜单
│      ├── ShowInAdmin (模块B)    │    模块B注入自己的菜单
│      └── ...                    │
└─────────────────────────────────┘
    │
    ▼
控制器执行
    │
    ▼
视图渲染
    │
    ▼
resources/views/components/layouts/admin/menu.blade.php
    │
    ▼
{!! menu('admin') !!}  ← 【菜单渲染点】输出 HTML
    │
    ▼
浏览器显示菜单
```

### 3. 菜单事件的两个阶段

菜单构建分为两个事件阶段，模块可以选择在合适的时机注入：

| 事件 | 时机 | 用途 |
|------|------|------|
| `AdminCreating` | 菜单刚创建，**还没有**任何菜单项 | 注入需要在最前面的菜单项 |
| `AdminCreated` | 核心菜单已经添加完毕 | 在现有菜单之后/之间注入 |

大多数模块选择监听 `AdminCreated` 事件，在核心菜单之后添加自己的菜单项。

### 4. 菜单渲染输出

菜单在视图中通过 `menu('admin')` 辅助函数渲染：

**视图位置**：[menu.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/resources/views/components/layouts/admin/menu.blade.php#L168-L170)

```blade
<div class="main-menu transform">
    {!! menu('admin') !!}
</div>
```

`menu('admin')` 会调用 `Menu` 呈现器来生成完整的 HTML 菜单结构，包括：
- 一级菜单项
- 下拉子菜单
- 图标
- 活动状态高亮

---

## 关键代码追踪

### 1. 模块状态管理

| 文件 | 职责 |
|------|------|
| [ModuleActivator.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/ModuleActivator.php) | 模块启用状态管理、模块注册入口 |
| [Module.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Models/Module/Module.php) | 模块数据库模型 |
| [Modules.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Modules.php) | 模块相关辅助方法 trait |

### 2. 公司识别与模块加载

| 文件 | 职责 |
|------|------|
| [IdentifyCompany.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Http/Middleware/IdentifyCompany.php) | 公司识别中间件，调用 registerModules |
| [Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Http/Kernel.php) | 中间件组配置，定义执行顺序 |

### 3. 菜单构建

| 文件 | 职责 |
|------|------|
| [AdminMenu.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Http/Middleware/AdminMenu.php) | 菜单构建中间件，触发菜单事件 |
| [AdminCreated.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Events/Menu/AdminCreated.php) | 菜单创建完成事件 |
| [ShowInAdmin.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Listeners/Menu/ShowInAdmin.php) | 核心菜单监听器 |

### 4. 权限判断

| 文件 | 职责 |
|------|------|
| [Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Traits/Permissions.php) | 权限相关辅助方法 trait |
| [Permission.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Models/Auth/Permission.php) | 权限模型 |
| [Controller.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Http/Controller.php) | 控制器基类，自动分配权限 |

### 5. 路由与模块别名

| 文件 | 职责 |
|------|------|
| [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Route.php) | 路由服务提供者，定义 Route::admin() 等宏 |
| [admin.stub](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Console/Stubs/Modules/routes/admin.stub) | 模块路由模板 |

### 6. 菜单呈现

| 文件 | 职责 |
|------|------|
| [Menu.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/View/Presenters/Menu.php) | 菜单 HTML 呈现器 |
| [menu.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/resources/views/components/layouts/admin/menu.blade.php) | 菜单视图模板 |

---

## 总结

### 关键认知点

1. **模块是按公司启用的**：同一模块在不同公司可以有不同的启用状态
2. **模块是请求时加载的**：不是应用启动时一次性加载，而是每次请求识别公司后动态加载
3. **菜单是每次请求重建的**：中间件每次请求都会重新构建菜单
4. **事件驱动解耦**：模块通过监听事件注入菜单，不需要修改核心代码
5. **权限是自动生成的**：控制器权限根据类名自动生成，遵循统一命名约定

### 模块别名影响的完整链路

```
模块别名 (blog)
    │
    ├──→ 目录名: modules/Blog/
    ├──→ 命名空间: Modules\Blog\
    ├──→ 路由前缀: /{company_id}/blog/
    ├──→ 路由命名: blog.*
    │
    ├──→ 权限名: {action}-blog-{controller}
    │   ├── create-blog-posts
    │   ├── read-blog-posts
    │   ├── update-blog-posts
    │   └── delete-blog-posts
    │
    ├──→ 菜单项
    │   ├── 路由关联: route('blog.posts.index')
    │   ├── 权限检查: canAccessMenuItem('read-blog-posts')
    │   └── 图标路径: modules/Blog/Resources/assets/img/icons/
    │
    └──→ 数据库状态
        └── modules 表中 alias=blog 的记录
```

### 模块开发者需要关心的事

1. 在 `module.json` 中声明服务提供者
2. 创建 `Listeners/Menu/ShowInAdmin.php` 监听菜单事件
3. 使用 `canAccessMenuItem()` 检查权限后再添加菜单
4. 在 `FinishInstallation.php` 中创建模块权限
5. 权限命名遵循 `{action}-{alias}-{resource}` 格式
6. 路由使用 `Route::admin()` / `Route::portal()` 宏
