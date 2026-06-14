# 批量动作授权过滤八处深度分析

## 一、admin 中间件 baseline 闸

**代码位置**：[Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php#L76-L88)

### 路由注册与中间件组

在 [Route.php::mapAdminRoutes()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Providers/Route.php#L228-L234) 中，所有 admin 路由统一挂载 `admin` 中间件组：

```php
protected function mapAdminRoutes()
{
    Facade::prefix('{company_id}')
        ->middleware('admin')
        ->namespace($this->namespace)
        ->group(base_path('routes/admin.php'));
}
```

批量操作路由 `bulk-actions/{group}/{type}` 定义在 [admin.php:42](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/routes/admin.php#L42-L42)，自然继承 admin 中间件组的全部 baseline 闸。

### admin 中间件组的 8 道闸门

[Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php#L76-L88) 中 `admin` 组定义：

```php
'admin' => [
    'web',                      // Session、CSRF、语言、防火墙
    'auth',                     // 登录认证
    'auth.disabled',            // 用户是否被禁用
    'company.identify',         // 租户识别（{company_id} 参数）
    'bindings',                 // 路由模型绑定
    'read.only',                // 只读模式检查
    'wizard.redirect',          // 安装向导未完成重定向
    'menu.admin',               // 管理员菜单构建
    'permission:read-admin-panel',  // 核心权限闸：必须能访问后台
    'plan.limits',              // 套餐限制检查
    'module.subscription',      // 模块订阅检查
],
```

### baseline 闸的授权含义

| 中间件 | 作用 | 对批量操作的影响 |
|--------|------|----------------|
| `auth` | 确保用户已登录 | 未登录直接 401 |
| `auth.disabled` | 检查用户是否被禁用 | 被禁用用户被强制登出 |
| `company.identify` | 从 `{company_id}` 路由参数识别当前租户 | 决定后续所有查询的 `company_id` 全局 scope |
| `permission:read-admin-panel` | **核心授权基线** | 无后台访问权限的用户（如仅前台客户）无法进入任何批量操作 |
| `plan.limits` / `module.subscription` | 付费限制 | 超套餐或模块过期时批量操作也被拦截 |

> **关键点**：`read-admin-panel` 是所有批量操作的前置授权基线。即使某用户拥有 `delete-sales-invoices` 权限，只要没有 `read-admin-panel`，在中间件层就被拒绝，根本到达不了 `BulkActionsController`。

---

## 二、abstract enable 直接 save 绕 Job authorize、与 disable 不对称

**代码位置**：[BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L168-L193)

### enable 方法在抽象基类中的实现

```php
public function enable($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->enabled = true;
        $item->save();  // 直接调用 Model::save()，不经过 Job
    }
}

public function disable($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->enabled = false;
        $item->save();  // 同上，直接 save
    }
}
```

### 与子类 disable 走 Job 的不对称性

**子类 Customers/Vendors** 覆盖了 disable 方法，委托基类方法走 Job：

[Customers.php::disable()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php#L81-L84)
```php
public function disable($request)
{
    $this->disableContacts($request);  // 委托基类的 disableContacts，走 Job
}
```

[BulkAction.php::disableContacts()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L223-L234)
```php
public function disableContacts($request)
{
    $contacts = $this->getSelectedRecords($request, 'user');

    foreach ($contacts as $contact) {
        try {
            $this->dispatch(new UpdateContact($contact, request()->merge(['enabled' => 0])));
        } catch (\Exception $e) {
            flash($e->getMessage())->error()->important();
        }
    }
}
```

### 授权绕过问题

| 路径 | 是否走 Job | 是否调用 Job::authorize() | 业务规则校验 |
|------|-----------|--------------------------|------------|
| 抽象基类 `enable()` | ❌ 直接 `$item->save()` | ❌ 无 | ❌ 完全绕过 |
| 抽象基类 `disable()` | ❌ 直接 `$item->save()` | ❌ 无 | ❌ 完全绕过 |
| 子类 Customers `disable()` | ✅ `dispatch(UpdateContact)` | ✅ 调用 | ✅ 检查关联数据 |
| 子类 `destroy()` | ✅ `dispatch(Delete*)` | ✅ 调用 | ✅ 检查关联数据 |

**以客户禁用为例**，[UpdateContact::authorize()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Common/UpdateContact.php#L59-L66) 中有业务规则：

```php
public function authorize(): void
{
    // 禁用时检查是否有关联交易/单据
    if (($this->request->has('enabled') && ! $this->request->get('enabled')) 
        && ($relationships = $this->getRelationships())) {
        throw new \Exception($message);
    }
}
```

**风险场景**：
- 如果一个 Item 子类（如 [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php)）没有覆盖 enable/disable，直接继承基类实现
- 此时批量启用/禁用商品时，**完全跳过** [UpdateItem Job](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Common/UpdateItem.php) 中可能存在的业务校验
- 而如果走单条编辑页面的 update 流程，是会经过 `UpdateItem` Job 的 authorize 校验的

---

## 三、公司更新按归属 skip 不计失败

**代码位置**：[Companies.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Companies.php#L63-L82)

```php
public function update($request)
{
    $companies = $this->getSelectedRecords($request);

    foreach ($companies as $company) {
        try {
            if ($this->isNotUserCompany($company->id)) {
                continue;  // 不属于当前用户的公司，直接 skip
            }

            $request->merge([
                'enabled' => $company->enabled,
            ]);

            $this->dispatch(new UpdateCompany($company, $this->getUpdateRequest($request), company_id()));
        } catch (\Exception $e) {
            flash($e->getMessage())->error()->important();
        }
    }
}
```

### skip 的逻辑分析

1. **`isNotUserCompany()`** 检查当前登录用户是否属于该公司（通过 `user_companies` 关联表）
2. 如果用户不属于该公司，**静默 `continue`**，不产生任何 flash 消息
3. 没有 `flash()` 调用，因此在 [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L70-L74) 的失败统计中，这条 skip 不会被计入 `$not_passed`

```php
flash()->messages->each(function ($message) use (&$not_passed) {
    if (in_array($message->level, ['danger', 'warning'])) {
        $not_passed++;
    }
});
```

### 造成的结果失真

假设选中 5 家公司，其中 2 家不属于当前用户：

- 实际处理：3 家处理（可能成功或失败）+ 2 家静默跳过
- 统计逻辑：`$count = 5`，`$not_passed` 只统计那 3 家中的失败数
- 显示消息：`"成功处理了 {5 - not_passed} 条"`
- **问题**：那 2 家被跳过的公司既不算成功也不算失败，用户完全不知情

### 对比 Job 中的同类检查

在 [UpdateCompany::authorize()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Common/UpdateCompany.php#L118-L122) 中也有同样检查，但行为不同：

```php
if ($this->isNotUserCompany($this->model->id)) {
    $message = trans('companies.error.not_user_company');
    throw new \Exception($message);  // 抛异常 → 被 catch → flash error → 计入 not_passed
}
```

| 检查位置 | 不属于归属公司时的行为 | 是否计入失败 |
|----------|----------------------|-------------|
| BulkAction `update()` foreach 内 | `continue` 静默跳过 | ❌ 不计入 |
| Job `authorize()` | `throw new Exception` | ✅ 计入 |

---

## 四、客户/供应商 destroy 委托基类走 Job

**代码位置**：[Customers.php::destroy()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php#L86-L89)、[Vendors.php::destroy()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Vendors.php#L86-L89)

### 委托模式

```php
// Customers.php
public function destroy($request)
{
    $this->deleteContacts($request);  // 委托给抽象基类方法
}

// Vendors.php  
public function destroy($request)
{
    $this->deleteContacts($request);  // 同上
}
```

### 基类实现

[BulkAction.php::deleteContacts()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L236-L247)

```php
public function deleteContacts($request)
{
    $contacts = $this->getSelectedRecords($request, 'user');

    foreach ($contacts as $contact) {
        try {
            $this->dispatch(new DeleteContact($contact));  // 走 Job
        } catch (\Exception $e) {
            flash($e->getMessage())->error()->important();
        }
    }
}
```

### 与其他 destroy 实现的对比

| 资源类型 | destroy 实现方式 | 是否走 Job |
|---------|----------------|-----------|
| Customers/Vendors | 委托基类 `deleteContacts()` | ✅ `dispatch(DeleteContact)` |
| Invoices | 子类直接实现 `foreach + dispatch(DeleteDocument)` | ✅ |
| Items | 子类直接实现 `foreach + dispatch(DeleteItem)` | ✅ |
| Companies | 子类直接实现 `foreach + dispatch(DeleteCompany)` | ✅ |
| Accounts | 子类直接实现 `foreach + dispatch(DeleteAccount)` | ✅ |
| **Categories/Taxes/Currencies** | **继承抽象基类 `destroy()`** | **❌ 直接 `$item->delete()`** |

### 抽象基类的 default destroy 实现

[BulkAction.php::destroy()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L214-L221)

```php
public function destroy($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->delete();  // 直接 Model::delete()，不走 Job，绕过 Job authorize
    }
}
```

### 风险：分类/币种/税率批量删除绕过业务校验

[DeleteCategory::authorize()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Setting/DeleteCategory.php#L46-L56) 中有检查：

```php
public function authorize(): void
{
    if ($relationships = $this->getRelationships()) {
        // 有关联交易/单据的分类不能删
        throw new \Exception(trans('messages.warning.deleted', [...]));
    }
}
```

但 [Categories.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Categories.php) 没有覆盖 `destroy()`，直接继承基类实现 → **不走 `DeleteCategory` Job → 绕过 authorize 校验**。

而客户/供应商因为走了委托基类的 `deleteContacts()` → `dispatch(DeleteContact)` → **正确经过 Job authorize 校验**。

> **设计不一致**：客户/供应商批量删除是安全的，但分类/币种/税率批量删除绕过了业务规则校验。

---

## 五、选中加载 hook 让分类 scope 缩窗

**代码位置**：[Categories.php::getSelectedRecords()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Categories.php#L105-L116)

### 分类 BulkAction 重写了选中记录加载方法

```php
public function getSelectedRecords($request, $relationships = null)
{
    if (empty($relationships)) {
        $model = $this->model::query();
    } else {
        $relationships = Arr::wrap($relationships);
        $model = $this->model::with($relationships);
    }

    // 关键：调用 getWithoutChildren() 缩小查询范围
    return $model->getWithoutChildren()->find($this->getSelectedInput($request));
}
```

### `getWithoutChildren()` 的作用

[Category Builder](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Builders/Category.php#L43-L46) 中定义：

```php
public function getWithoutChildren($columns = ['*'])
{
    return parent::get($columns);  // 不触发自定义 get() 中的 withChildren 逻辑
}

// 对比默认 get() 会递归加载所有子分类：
public function get($columns = ['*'])
{
    $collection = parent::get($columns);
    return $collection->withChildren('sub_categories', function (...) {
        // 递归加载所有层级子分类...
    });
}
```

### 缩窗的实际效果

| 方法 | SQL 查询 | 返回结果 | 隐含的 scope 缩窗 |
|------|----------|----------|----------------|
| 默认 `get()` | 查分类表 → PHP 层递归加载所有子孙 | 包含选中分类 + 所有后代 | ❌ 可能无意中批量操作到子分类 |
| `getWithoutChildren()` | 仅查分类表，不递归 | 仅选中的那些分类记录 | ✅ 严格限定在用户勾选的 ID |

### 为什么这是一个「授权过滤」点

如果没有 `getWithoutChildren()`：

1. 用户勾选了某个父分类 ID = 5
2. `getSelectedRecords()` 返回分类 5 + 它的所有子分类 5.1, 5.2, 5.3
3. 后续 `foreach` 循环会处理 4 条记录，而用户只勾选了 1 条
4. 用户可能无权操作子分类（虽然系统没有记录级权限，但业务上属于越权操作）

`getWithoutChildren()` 相当于在**数据加载层**做了一次 scope 过滤，确保只处理用户实际勾选的记录。

---

## 六、Company 全局 scope 跨租户 id 静默丢

**代码位置**：[Company.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Scopes/Company.php#L21-L50)

### 全局 Scope 定义

```php
public function apply(Builder $builder, Model $model)
{
    if (method_exists($model, 'isNotTenantable') && $model->isNotTenantable()) {
        return;
    }

    $table = $model->getTable();

    $skip_tables = [
        'jobs', 'firewall_ips', 'firewall_logs', 'migrations', 'notifications', 
        'role_companies', 'role_permissions', 'sessions', 'user_companies', 
        'user_dashboards', 'user_permissions', 'user_roles',
    ];

    if (in_array($table, $skip_tables)) {
        return;
    }

    if ($this->scopeColumnExists($builder, '', 'company_id')) {
        return;
    }

    // 静默追加 company_id 过滤
    $builder->where($table . '.company_id', '=', company_id());
}
```

### 对批量操作的影响

在 [BulkAction.php::getSelectedRecords()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L81-L92) 中：

```php
public function getSelectedRecords($request, $relationships = null)
{
    // ...
    return $model->find($this->getSelectedInput($request));
}
```

假设场景：

1. 用户当前在 `company_id = 1` 的租户中
2. 前端传来 `selected = [101, 102, 203]`，其中 203 是 `company_id = 2` 的记录（可能是从其他公司复制过来的 ID，或者恶意构造）
3. `Model::find([101, 102, 203])` 被全局 scope 追加 `where company_id = 1`
4. **实际返回只有 [101, 102]**，ID 203 被静默过滤掉
5. 没有任何错误或警告，用户以为 3 条都处理了，实际上只处理了 2 条

### 静默丢的特征

| 现象 | 表现 |
|------|------|
| **静默** | 不抛异常、不打日志、不 flash 消息 |
| **自动** | 由 Eloquent 全局 scope 自动触发，业务代码无感知 |
| **统计失真** | `$count = count($request->get('selected'))` 统计的是提交的 ID 数（3），但实际处理的是 scope 过滤后的数量（2） |
| **消息误导** | 最终显示 `"成功处理了 {3 - not_passed} 条"`，实际只处理了 2 条 |

### 代码证据：统计与实际处理分离

[BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L67-L76)

```php
$result = $bulk_actions->{$handle}($request);  // 实际处理可能因 scope 少了 N 条

$count = count($request->get('selected'));     // 统计的是提交的数量
$not_passed = 0;

// ...
$message = trans($bulk_actions->messages['general'], [
    'type' => $handle, 
    'count' => $count - $not_passed  // 计算基数是提交数，不是实际处理数
]);
```

> **隐蔽风险**：如果用户勾选了 10 条，其中 5 条不属于当前公司，`getSelectedRecords()` 只返回 5 条，循环处理 5 条，但 `$count` 还是 10，消息显示「成功处理了 10 条」，实际只成功 5 条。

---

## 七、扩展事件让 module 改 actions

**代码位置**：[BulkActionsAdding.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Events/Common/BulkActionsAdding.php)

### 事件触发点

在视图组件 [Bulkaction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/View/Components/Index/Bulkaction.php#L42-L107) 和 Trait [ViewComponents.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Traits/ViewComponents.php#L512-L512) 中触发：

```php
// View/Components/Index/Bulkaction.php
if (class_exists($this->class)) {
    $bulk_action = app($this->class);
    event(new BulkActionsAdding($bulk_action));  // 触发事件
    // ...
}

// Traits/ViewComponents.php
$b = new \stdClass();
$b->actions = [];
event(new BulkActionsAdding($b));  // 模块扩展时触发
```

### 事件对象结构

```php
class BulkActionsAdding extends Event
{
    public $bulk_action;  // 传入的 BulkAction 实例或 stdClass

    public function __construct($bulk_action)
    {
        $this->bulk_action = $bulk_action;
    }
}
```

### 模块如何修改 actions

模块可以监听此事件，动态修改 `$bulk_action->actions` 数组：

```php
// 模块的 EventServiceProvider 中
protected $listen = [
    BulkActionsAdding::class => [
        AddCustomBulkAction::class,  // 监听器
    ],
];

// 监听器实现
class AddCustomBulkAction
{
    public function handle(BulkActionsAdding $event)
    {
        // 只针对特定资源类型扩展
        if ($event->bulk_action instanceof \App\BulkActions\Sales\Invoices) {
            $event->bulk_action->actions['sync_to_external'] = [
                'icon' => 'sync',
                'name' => 'external.sync',
                'message' => 'external.sync.message',
                'permission' => 'update-sales-invoices',  // 可以加权限要求
            ];
        }
        
        // 也可以移除某个 action
        if (user()->cannot('delete-special-invoices')) {
            unset($event->bulk_action->actions['delete']);
        }
    }
}
```

### 对授权过滤的影响

| 修改方式 | 授权影响 |
|----------|---------|
| **新增 action** | 可以指定 `permission` 字段，由 [Bulkaction.php:94](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/View/Components/Index/Bulkaction.php#L94-L96) 和 [BulkActions.php:51-53](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L51-L53) 两层校验 |
| **修改现有 action 的 permission** | 直接改变该操作的权限门槛 |
| **移除 action** | 等效于动态收回权限，即使配置中定义了也不显示/不可用 |
| **修改权限字段为 null** | 移除权限检查，任何登录用户都可执行 |

> **关键点**：事件监听是在渲染时（视图层）和执行时（控制器层）都可能触发吗？
> 
> - 视图层：触发 → 过滤按钮显示
> - 控制器层：**不触发**。控制器直接 `app($bulkActionClass)` 后检查 `$bulk_actions->actions[$handle]['permission']`，没有事件触发。
> 
> 这意味着：**如果监听器只修改按钮显示（视图层），但控制器层的实际权限检查不会被修改**。除非监听器在两处都生效，否则可能出现「按钮隐藏了但直接 POST 仍能执行」的不一致。

---

## 八、foreach 内 request merge 污染下一条

### 问题代码模式

在多个 BulkAction 子类的 `update()` 方法中存在这种模式：

[Customers.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php#L63-L79)
```php
public function update($request)
{
    $customers = $this->getSelectedRecords($request);

    foreach ($customers as $customer) {
        try {
            $request->merge([
                'enabled' => $customer->enabled,
                'uploaded_logo' => $customer->logo,
            ]); // for update job authorize..

            $this->dispatch(new UpdateContact($customer, $this->getUpdateRequest($request)));
        } catch (\Exception $e) {
            flash($e->getMessage())->error()->important();
        }
    }
}
```

相同模式出现在：
- [Invoices.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php#L74-L100)
- [Transactions.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transactions.php#L91-L109)
- [Items.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php#L68-L79)
- [Accounts.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Accounts.php#L57-L73)
- [Categories.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Categories.php#L62-L77)
- [Users.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Auth/Users.php#L80-L96)
- [Companies.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Companies.php#L63-L82)

### 污染原理

`$request` 是**同一个 Request 对象实例**在 foreach 中被反复 `merge()`：

| 循环次数 | 操作 | $request 状态 |
|---------|------|--------------|
| 第 1 次（客户 A） | `merge(['enabled' => 1, 'uploaded_logo' => 'logo_a.png'])` | `{enabled:1, uploaded_logo:'logo_a.png', name:'批量修改的名称'}` |
| 第 2 次（客户 B） | `merge(['enabled' => 0, 'uploaded_logo' => 'logo_b.png'])` | `{enabled:0, uploaded_logo:'logo_b.png', name:'批量修改的名称'}` ✅ 正常覆盖 |
| 第 3 次（客户 C，无 logo） | `merge(['enabled' => 1])` ← 只 merge 了 enabled | `{enabled:1, uploaded_logo:'logo_b.png', name:'批量修改的名称'}` ❌ **污染！** |

客户 C 没有 logo，`merge()` 只设置了 `enabled`，但**上一次循环的 `uploaded_logo` 仍然保留在 request 中**，导致客户 C 被错误地赋予了客户 B 的 logo。

### 更隐蔽的场景：表单字段不完整时

假设用户在批量编辑弹窗中只修改了 `category_id`，没有修改其他字段：

```php
// 第 1 次：发票 A，原 category_id = 1
$request->merge(['category_id' => $invoice->category_id]);  // merge 1
// request: {category_id: 1, discount: 10}（discount 来自表单）

// 第 2 次：发票 B，原 category_id = 2，但表单修改为 5
$request->merge(['category_id' => $invoice->category_id]);  // merge 2，试图还原为 2
// 但用户实际要改 category_id 为 5，这里 merge 的 2 会被表单的 5 覆盖吗？
// 取决于 merge 顺序和表单数据的位置
```

### `getUpdateRequest()` 的清理不足

[BulkAction.php::getUpdateRequest()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L99-L108)

```php
public function getUpdateRequest($request)
{
    foreach ($request->all() as $key => $value) {
        if (empty($value)) {
            unset($request[$key]);
        }
    }

    return $request;
}
```

这个方法**只清理了空值**，没有清理上一次循环遗留的非空值。

### 安全隐患

1. **数据污染**：如前面 logo 例子，一条记录的数据泄露到另一条记录
2. **权限绕过**：如果某个字段的修改需要特殊权限，但通过污染被绕过
3. **业务逻辑错误**：如 [Invoices.php::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php#L85-L93) 中 merge 了发票的 items、totals 等复杂结构，污染后果更严重

### 正确写法应该是

```php
// 每次循环创建新的 request 实例，或使用 only() 明确指定字段
$cleanRequest = $request->only(['name', 'email']);  // 只取表单提交的字段
$cleanRequest['enabled'] = $customer->enabled;      // 追加当前记录的字段
$cleanRequest['uploaded_logo'] = $customer->logo;

$this->dispatch(new UpdateContact($customer, $cleanRequest));
```

---

## 八处问题总结表

| # | 问题 | 影响范围 | 严重程度 | 本质 |
|---|------|---------|---------|------|
| 1 | admin 中间件 baseline 闸 | 所有批量操作 | ✅ 设计正确 | 多道前置防线，先认证后授权 |
| 2 | abstract enable 直接 save 绕 Job authorize | 分类、币种、税率、商品等未覆盖子类的 enable/disable | ⚠️ 中高 | 业务校验被绕过，设计不一致 |
| 3 | 公司更新按归属 skip 不计失败 | 公司批量更新 | ⚠️ 中 | 静默跳过导致统计失真、用户不知情 |
| 4 | 客户/供应商 destroy 委托基类走 Job | 客户/供应商批量删除 | ✅ 设计正确 | 经过 Job authorize，对比分类等的不一致 |
| 5 | 分类 getSelectedRecords 用 getWithoutChildren 缩窗 | 分类批量操作 | ✅ 设计正确 | 防止递归加载子分类导致范围扩大 |
| 6 | Company 全局 scope 跨租户 id 静默丢 | 所有批量操作 | ⚠️ 高 | 统计与实际处理脱节，数据丢失无感知 |
| 7 | BulkActionsAdding 事件让 module 改 actions | 模块扩展时 | ⚠️ 中 | 视图层与控制器层事件触发不一致可能导致越权 |
| 8 | foreach 内 request merge 污染下一条 | 所有带 update 方法的 BulkAction | 🔥 高 | 数据交叉污染，严重数据一致性问题 |

---

## 相关代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| admin 中间件组定义 | [Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php) | L76-L88 |
| admin 路由注册 | [Route.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Providers/Route.php) | L228-L234 |
| 抽象基类 enable/disable | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L168-L193 |
| 抽象基类 destroy（直接 delete） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L214-L221 |
| 基类 disableContacts（走 Job） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L223-L234 |
| 基类 deleteContacts（走 Job） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L236-L247 |
| 公司批量更新归属 skip | [Companies.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Companies.php) | L63-L82 |
| 失败数统计逻辑 | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L67-L93 |
| 分类 getSelectedRecords 缩窗 | [Categories.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Categories.php) | L105-L116 |
| 分类 Builder getWithoutChildren | [Category.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Builders/Category.php) | L43-L46 |
| Company 全局 Scope | [Company.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Scopes/Company.php) | L21-L50 |
| 选中记录加载方法 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L81-L92 |
| BulkActionsAdding 事件 | [BulkActionsAdding.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Events/Common/BulkActionsAdding.php) | L7-L20 |
| 视图层事件触发 | [Bulkaction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/View/Components/Index/Bulkaction.php) | L51、L67 |
| 控制器层权限检查（无事件） | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L50-L63 |
| Customers update 污染示例 | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php) | L63-L79 |
| Invoices update 污染示例 | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | L74-L100 |
| getUpdateRequest 清理不足 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L99-L108 |
