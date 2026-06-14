# 批量动作授权过滤六处实现深度分析

## 一、缺 permission 键的 export/download 绕授权

**代码位置**：[BulkAction.php:25-51](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L25-L51)（抽象基类默认 actions 定义）

### 1.1 抽象基类默认 actions 没有 permission

```php
public $actions = [
    'enable'    => [
        'name'          => 'general.enable',
        'message'       => 'bulk_actions.message.enable',
        'permission'    => 'update-common-items',   // ✅ 有 permission
    ],
    'disable'   => [
        'name'          => 'general.disable',
        'message'       => 'bulk_actions.message.disable',
        'permission'    => 'update-common-items',   // ✅ 有 permission
    ],
    'delete'    => [
        'name'          => 'general.delete',
        'message'       => 'bulk_actions.message.delete',
        'permission'    => 'delete-common-items',   // ✅ 有 permission
    ],
    'export'    => [
        'name'          => 'general.export',
        'message'       => 'bulk_actions.message.export',
        'type'          => 'download'
        // ❌ 没有 permission 键！
    ],
    'download' => [
        'name'          => 'general.download',
        'message'       => 'bulk_actions.message.download',
        'type'          => 'download',
        // ❌ 没有 permission 键！
    ],
];
```

### 1.2 控制器层授权检查逻辑

[BulkActions.php:50-63](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L50-L63)

```php
if (
    isset($bulk_actions->actions[$handle]['permission'])   // 用 isset 检查
    && ! user()->can($bulk_actions->actions[$handle]['permission'])
) {
    // 返回 403
}
// 如果 permission 键不存在 → 条件为 false → 跳过检查 → 直接放行
```

### 1.3 所有子类 export/download 也不补 permission 键

抽查所有定义了 `export`/`download` 的子类：

| 子类 | action | 有 permission 键？ |
|------|--------|-------------------|
| [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php#L53-L64) | export/download | ❌ 都没有 |
| [Bills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Bills.php#L53-L58) | export | ❌ |
| [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php#L48-L54) | export | ❌ |
| [Vendors.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Vendors.php#L48-L54) | export | ❌ |
| [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php#L53-L59) | export | ❌ |
| [Transactions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transactions.php#L39-L45) | export | ❌ |
| [Transfers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transfers.php#L28-L33) | export | ❌ |

### 1.4 对比：单条操作（单条导出）的权限保护

在列表页单行操作中，导出权限是受保护的：
[Document.php:583-614](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Models/Document/Document.php#L583-L614)

```php
$actions[] = [
    'title' => trans('general.export'),
    'icon'  => 'description',
    'url'   => route($prefix . '.export', $this->id),
    'permission' => 'read-' . $group . '-' . $permission_prefix,  // ✅ 有 permission
];
```

但批量操作的 `export` 没有对应权限。

### 1.5 漏洞分析

**绕过路径**：

```
任何已登录的后台用户（只要通过 admin 中间件 baseline）
    → POST bulk-actions/sales/invoices
    → handle=export, selected=[1,2,3,4...]
    → 因为 action 配置没有 'permission' 键
    → isset() 返回 false → 授权检查短路
    → 直接执行 $bulk_actions->export($request)
    → 下载所有选中发票的 Excel/PDF
```

**实际影响**：
- 只要能登录后台（`read-admin-panel` 权限），即使没有任何销售发票相关权限，也能批量导出全部发票数据
- download 同理（批量下载 PDF 附件）

---

## 二、abstract 默认 destroy/enable/disable/duplicate 等多处漏 try/catch 抛断 batch

### 2.1 抽象基类中所有默认实现都没有 try/catch

[BulkAction.php:152-260](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L152-L260)

```php
// L152-159
public function duplicate($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->duplicate();   // ❌ 无 try/catch
    }
}

// L168-175
public function enable($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->enabled = true;
        $item->save();        // ❌ 无 try/catch
    }
}

// L185-193
public function disable($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->enabled = false;
        $item->save();        // ❌ 无 try/catch
    }
}

// L202-205
public function delete($request)
{
    $this->destroy($request);
}

// L214-221
public function destroy($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->delete();      // ❌ 无 try/catch
    }
}

// L223-234  disableContacts 是安全的
public function disableContacts($request)
{
    foreach ($contacts as $contact) {
        try {                                  // ✅ 有 try/catch
            $this->dispatch(new UpdateContact($contact, ...));
        } catch (\Exception $e) {
            flash($e->getMessage())->error()->important();
        }
    }
}
```

### 2.2 Reconciliations 子类也漏了 try/catch

[Reconciliations.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Reconciliations.php)

```php
// L41-58
public function reconcile($request)
{
    $reconciliations = $this->getSelectedRecords($request);

    foreach ($reconciliations as $reconciliation) {
        \DB::transaction(function () use ($reconciliation) {
            $reconciliation->reconciled = 1;
            $reconciliation->save();                // ❌ 无 try/catch，在 DB::transaction 内
            // 批量更新相关交易
        });
    }
}

// L60-77 unreconcile 同理无 try/catch
// L79-95 destroy 同理无 try/catch
```

### 2.3 Bills/Invoices 的 received/cancelled 也漏 try/catch

[Bills.php:96-119](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Bills.php#L96-L119)

```php
public function received($request)
{
    $bills = $this->getSelectedRecords($request);

    foreach ($bills as $bill) {
        if ($bill->status == 'received') {
            continue;
        }

        event(new DocumentReceived($bill));   // ❌ 无 try/catch
    }
}

public function cancelled($request)
{
    $bills = $this->getSelectedRecords($request);

    foreach ($bills as $bill) {
        if (in_array($bill->status, ['cancelled', 'draft'])) {
            continue;
        }

        event(new DocumentCancelled($bill));  // ❌ 无 try/catch
    }
}
```

### 2.4 抛断后果

假设勾选 100 条记录，第 37 条触发数据库异常（如唯一约束冲突、外键约束、保存时 Observer 抛异常）：

| 实现方式 | 第 37 条异常时 | 已处理的 1-36 条 | 未处理的 38-100 条 | 错误反馈 |
|----------|----------------|-----------------|-------------------|---------|
| **有 try/catch**（如 update/destroy 走 Job） | catch → flash error → continue | ✅ 保留 | ✅ 继续处理 | 逐条 flash |
| **无 try/catch**（基类 destroy/enable/disable/duplicate） | 异常冒泡 → 请求 500 | ✅ 保留（已提交） | ❌ **全部中断不处理** | 空白错误页/JSON 500 |

### 2.5 同时也漏了 not_passed 统计

由于没有 `catch { flash(error) }`，即使第 37 条异常了（如果有全局异常处理器把请求变成错误响应），在异常发生前已经成功的 1-36 条和失败的第 37 条都**不会被统计**。控制器层 `flash()->messages->each(...)` 根本执行不到（因为异常已抛出）。

---

## 三、duplicate 完全跳 Job authorize

### 3.1 单条 duplicate 的正确流程

[Invoices.php::duplicate()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Sales/Invoices.php#L134-L143)（单条操作）

```php
public function duplicate(Document $invoice)
{
    $clone = $this->dispatch(new DuplicateDocument($invoice));  // ✅ 走 Job
    // ...
}
```

但 `DuplicateDocument` Job 自身**没有 authorize() 方法**：

[DuplicateDocument.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Document/DuplicateDocument.php)

```php
class DuplicateDocument extends Job
{
    public function handle(): Document
    {
        event(new DocumentDuplicating($this->model));

        \DB::transaction(function () {
            $this->clone = $this->model->duplicate();  // 直接调用 Model duplicate
        });

        event(new DocumentCreated($this->clone, request()));
        return $this->clone;
        // ❌ 没有 $this->authorize() 调用
    }
}
```

[DuplicateContact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Common/DuplicateContact.php) 同样没有 authorize。

### 3.2 批量 duplicate 更严重：连 Job 都不走

抽象基类 [BulkAction.php:152-159](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L152-L159)：

```php
public function duplicate($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->duplicate();   // ❌ 直接调用 Model::duplicate()，连 Job 都不 dispatch
    }
}
```

子类覆盖的情况：

[Invoices.php::duplicate()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php#L128-L152)

```php
public function duplicate($request)
{
    $invoices = $this->getSelectedRecords($request);

    foreach ($invoices as $invoice) {
        $clone = $invoice->duplicate();  // ❌ 同上，直接 Model::duplicate()

        $description = trans('messages.success.added', ['type' => $clone->document_number]);
        $this->dispatch(new CreateDocumentHistory($clone, 0, $description));
        // ❌ 不走 DuplicateDocument Job，自然也没有 Job authorize
        // ❌ 连 try/catch 都没有
    }
}
```

[Bills.php::duplicate()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Bills.php#L122-L133) 完全同构。

### 3.3 Model::duplicate() 的本质

使用 `bkwld/cloner` 包提供的 `Cloneable` trait：

[Document.php:16,24,101](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Models/Document/Document.php#L16-L101)

```php
use Bkwld\Cloner\Cloneable;

class Document extends Model
{
    use Cloneable, ...;

    public $cloneable_relations = ['items', 'recurring', 'totals'];
}
```

`$model->duplicate()` 是 Cloner 包提供的纯数据库复制逻辑，**不经过任何应用层业务校验**。

### 3.4 对比：Update/Delete Job 都有 authorize

| 操作 | 单条走 Job？ | 批量走 Job？ | Job 有 authorize？ |
|------|-------------|-------------|-------------------|
| create | ✅ | N/A | 看具体 Job |
| update | ✅ | ✅（dispatch UpdateXxx） | ✅ |
| delete | ✅ | ✅（dispatch DeleteXxx） | ✅ |
| **duplicate** | ✅（DuplicateDocument） | ❌（直接 `$item->duplicate()`） | ❌（Job 本身也没写） |

### 3.5 业务校验缺失的风险

- 对账单/单据的 duplicate：系统可能有业务规则（如已对账的单据不能复制、超信用额度的客户不能被复制出新单据等）——但 duplicate 路径完全跳过了所有校验
- 没有 try/catch（见问题二），一条失败整批中断
- 连权限校验都只依赖入口的整体 permission 检查，如果 duplicate action 缺 permission 键（见问题一的 pattern），则完全无授权

---

## 四、flash 计 not_passed 跨请求残留与 danger/warning 混算

### 4.1 not_passed 的统计方式

[BulkActions.php:67-74](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L67-L74)

```php
$count = count($request->get('selected'));
$not_passed = 0;

flash()->messages->each(function ($message) use (&$not_passed) {
    if (in_array($message->level, ['danger', 'warning'])) {
        $not_passed++;
    }
});
```

### 4.2 问题一：跨请求残留

`laracasts/flash` 包的 flash 消息存储在 **Session** 中，生命周期是「下一次请求」。Session 中的 flash 消息是跨请求的。

**残留场景**：

1. 请求 A（普通页面操作）：产生了 3 条 warning 级别 flash 消息（比如导入部分失败），存入 Session
2. 请求 B（批量删除 POST）：执行批量删除，其中 1 条业务校验失败，catch 中产生 1 条 danger flash
3. 统计时 `flash()->messages` 取出的是 **Session 中所有未消费的 flash 消息** = 3 条 warning + 1 条 danger = 4 条
4. `$not_passed = 4`，但实际批量操作只失败了 1 条
5. 显示：`"成功处理了 {count - 4} 条"`，数字严重失真

**更极端场景**：
- 如果上一次请求残留了 10 条 warning，这次批量操作 100 条全部成功
- `$not_passed = 10`，消息显示 `"成功处理了 90 条"`（info 级别）
- 用户以为 10 条失败了，实际一条都没失败

### 4.3 问题二：danger 与 warning 混算

`['danger', 'warning']` 两种级别不加区分地累加到 `$not_passed`，但两者语义完全不同：

| level | 语义 | 是否应该计入 not_passed |
|-------|------|----------------------|
| **danger** | 严重错误，**该条记录操作失败** | ✅ 应该计入 |
| **warning** | 警告提示，**操作可能成功了但需注意**，或者是与操作结果无关的系统提醒 | ❌ 不应该计入（或需具体判断） |

**warning 污染实例**：

```php
// 假设某条记录 delete 成功，但 Observer 里触发了一条 warning：
flash('该客户关联的 3 个提醒已一并归档，请核实')->warning();

// 统计时这条 warning 会计入 $not_passed，导致：
// - 实际：100 条全部删除成功
// - 统计：$not_passed = 1 → 显示「成功处理了 99 条」
```

### 4.4 问题三：一条失败产生多条 flash 消息

某些异常场景下，单条记录的失败可能产生多条 danger/warning 消息：

```php
// catch 中可能多次 flash：
catch (\Exception $e) {
    flash($e->getMessage())->error()->important();
    // 如果上层或 Observer 也产生了 flash...
    flash('关联的交易记录未清理，请手动处理')->warning();
}
```

一条失败记录产生 2 条 flash → `$not_passed += 2` → 统计值是实际失败数的 2 倍。

### 4.5 问题四：没有 flash 的失败不算失败

已在问题二、三中分析过：
- 基类 `destroy()`/`enable()`/`disable()` 等无 try/catch → 异常直接冒泡 → 连控制器层的 flash 统计都执行不到
- [Companies::update()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Companies.php#L70-L72) 中 `isNotUserCompany()` 用 `continue` 静默跳过，不产生 flash → 不计入 not_passed

### 4.6 设计缺陷总结

| 缺陷 | 表现 |
|------|------|
| 跨请求残留 | Session 中上一次请求的 flash 被计入本次批量操作统计 |
| 级别混算 | warning（非失败）与 danger（失败）同一计数器 |
| 多条消息 | 单条失败产生 N 条消息 → 统计放大 N 倍 |
| 静默失败 | 无 flash 的失败完全统计不到 |

---

## 五、module 分支 dispatch 模块 BulkActions

### 5.1 模块 BulkAction 的 class 定位逻辑

控制器层 [BulkActions.php:35-48](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L35-L48)：

```php
// Check is module
$module = module($group);
$page = ucfirst($type);

if ($module instanceof \Akaunting\Module\Module) {
    // 模块分支
    $tmp = explode('.', $type);
    $file_name = !empty($tmp[1]) ? Str::studly($tmp[0]) . '\\' . Str::studly($tmp[1]) : Str::studly($tmp[0]);

    $bulk_actions = app('Modules\\' . $module->getStudlyName() . '\BulkActions\\' . $file_name);
    // 例如：Modules\DoubleEntry\BulkActions\Banking\Transactions

    $page = ucfirst($file_name);
} else {
    // App 分支
    $bulk_actions = app('App\BulkActions\\' .  ucfirst($group) . '\\' . ucfirst($type));
}
```

视图层（权限按钮过滤）[ViewComponents.php:505-520](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Traits/ViewComponents.php#L505-L520)：

```php
if ($alias = config('type.' . static::OBJECT_TYPE . '.' . $type . '.alias')) {
    $module = module($alias);

    if (! $module instanceof Module) {
        $b = new \stdClass();
        $b->actions = [];
        event(new BulkActionsAdding($b));
        return $b->actions;
    }

    $bulkActionClass = 'Modules\\' . $module->getStudlyName() . '\BulkActions\\' . $file_name;
} else {
    $bulkActionClass = 'App\BulkActions\\' .  $file_name;
}
```

### 5.2 模块与 App 分支的类命名空间差异

| 路由 group/type | 分支 | 解析到的类 |
|----------------|------|-----------|
| `sales/invoices` | App | `App\BulkActions\Sales\Invoices` |
| `banking/transactions` | App | `App\BulkActions\Banking\Transactions` |
| `double-entry/banking.transactions` | Module | `Modules\DoubleEntry\BulkActions\Banking\Transactions` |

### 5.3 模块 BulkAction 的授权差异

模块自己定义的 BulkAction 类的 `$actions` 数组中 permission 键的约定是否与 App 分支一致？

**潜在不一致风险**：

1. **permission 命名不一致**：App 分支用 `delete-sales-invoices`（`{verb}-{group}-{type}`），模块可能用其他命名规则，导致 `user()->can()` 始终 false 或始终 true
2. **缺 permission 键**：模块开发者可能不知道约定，完全不写 permission 键 → 同问题一的绕授权
3. **权限字符串前缀**：`read-double-entry-ledgers` vs `read-module-double-entry-ledgers`，需要权限系统注册一致

### 5.4 模块 BulkAction 不走 `assignPermissionsToController` 中间件

App 分支的普通控制器（非批量）通过 [Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Traits/Permissions.php) 的 `assignPermissionsToController()` 自动绑定权限中间件：

```php
$this->middleware('permission:delete-' . $controller)->only('destroy');
```

但 `BulkActions` 控制器（无论 App 还是 Module）的 `action()` 方法统一走同一个路由，没有中间件自动映射。模块的 BulkAction 完全依赖 `$bulk_actions->actions[$handle]['permission']` 这一行手动检查。

### 5.5 模块与 App 分支的 `$actions` 继承差异

- App 分支的 BulkAction 都继承 `App\Abstracts\BulkAction`，自动获得基类 `$actions`（含 export/download 无 permission 的问题）
- 模块的 BulkAction 如果不继承 `App\Abstracts\BulkAction`，而是自己写一个 `$actions`，则完全由模块开发者决定权限配置——**安全保障更依赖人工**

---

## 六、15 子类 handle 全量矩阵

### 6.1 全部 15 个 BulkAction 子类清单

| # | 类文件 | 资源 | 所在目录 |
|---|--------|------|---------|
| 1 | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | 销售发票 | Sales |
| 2 | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php) | 客户 | Sales |
| 3 | [Bills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Bills.php) | 采购账单 | Purchases |
| 4 | [Vendors.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Vendors.php) | 供应商 | Purchases |
| 5 | [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php) | 产品/服务 | Common |
| 6 | [Companies.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Companies.php) | 公司 | Common |
| 7 | [Dashboards.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Dashboards.php) | 仪表盘 | Common |
| 8 | [Categories.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Categories.php) | 分类 | Settings |
| 9 | [Taxes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Taxes.php) | 税率 | Settings |
| 10 | [Currencies.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Settings/Currencies.php) | 币种 | Settings |
| 11 | [Accounts.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Accounts.php) | 银行账户 | Banking |
| 12 | [Transactions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transactions.php) | 交易 | Banking |
| 13 | [Transfers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transfers.php) | 转账 | Banking |
| 14 | [Reconciliations.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Reconciliations.php) | 对账 | Banking |
| 15 | [Users.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Auth/Users.php) | 用户 | Auth |

### 6.2 全量实现矩阵

| 子类 | edit | update | enable | disable | destroy | export | download | duplicate | sent/recvd | cancel | reconcile | 其他 |
|------|------|--------|--------|---------|---------|--------|----------|-----------|------------|--------|-----------|------|
| **Sales** | | | | | | | | | | | | |
| Invoices | ✅UI | ✅Job try/catch | — | — | ✅Job try/catch | ✅无perm | ✅无perm | ❌直接Model无catch | ✅event无catch | ✅event无catch | — | — |
| Customers | ✅UI | ✅Job try/catch ✨merge污染 | — | ✅委托Job try/catch | ✅委托Job try/catch | ✅无perm | — | — | — | — | — | — |
| **Purchases** | | | | | | | | | | | | |
| Bills | ✅UI | ✅Job try/catch ✨merge污染 | — | — | ✅Job try/catch | ✅无perm | — | ❌直接Model无catch | ✅event无catch | ✅event无catch | — | — |
| Vendors | ✅UI | ✅Job try/catch ✨merge污染 | — | ✅委托Job try/catch | ✅委托Job try/catch | ✅无perm | — | — | — | — | — | — |
| **Common** | | | | | | | | | | | | |
| Items | ✅UI | ✅Job try/catch ✨merge污染 | ⚠️基类直接save 无catch 绕Job | ⚠️子类Job try/catch | ✅Job try/catch | ✅无perm | — | — | — | — | — | — |
| Companies | ✅UI | ✅Job try/catch ⚠️归属skip | ✅子类Job try/catch | ✅子类Job try/catch | ✅Job try/catch | — | — | — | — | — | — | — |
| Dashboards | — | — | ⚠️基类直接save 无catch | ✅Job try/catch | ✅Job try/catch | — | — | — | — | — | — | — |
| **Settings** | | | | | | | | | | | | |
| Categories | ✅UI | ✅Job try/catch ✨merge污染 | ⚠️基类直接save 无catch | ✅子类Job try/catch | ✅子类Job try/catch | — | — | — | — | — | — | — |
| Taxes | ✅UI | ✅Job try/catch | ⚠️基类直接save 无catch | ✅Job try/catch | ✅Job try/catch | — | — | — | — | — | — | — |
| Currencies | ✅UI | ✅Job try/catch ✨merge污染 | ⚠️基类直接save 无catch | ✅Job try/catch | ✅Job try/catch | — | — | — | — | — | — | — |
| **Banking** | | | | | | | | | | | | |
| Accounts | ✅UI | ✅Job try/catch ✨merge污染 | ⚠️基类直接save 无catch | ✅Job try/catch | ✅Job try/catch | — | — | — | — | — | — | — |
| Transactions | ✅UI | ✅Job try/catch ✨merge污染 | — | — | ✅Job try/catch | ✅无perm | — | — | — | — | — | — |
| Transfers | — | — | — | — | ✅Job try/catch | ✅无perm | — | — | — | — | — | — |
| Reconciliations | — | — | — | — | ❌直接save 无catch 无Job | — | — | — | — | — | ❌直接save无catch | reconcile/unreconcile ❌直接save无catch |
| **Auth** | | | | | | | | | | | | |
| Users | ✅UI | ✅Job try/catch ✨merge污染 | — | ✅Job try/catch | ✅Job try/catch | — | — | — | — | — | — | invite ✅Job try/catch |

### 6.3 矩阵符号说明

| 符号 | 含义 |
|------|------|
| ✅Job try/catch | 走对应 Job 类，foreach 内有 try/catch 包住 dispatch |
| ✅UI | 只打开编辑弹窗（edit 方法返回 view），不直接改数据 |
| ✅委托 Job try/catch | 调用基类 disableContacts/deleteContacts，内部有 try/catch + dispatch Job |
| ✅event 无 catch | 触发 event，无 try/catch，异常直接冒泡 |
| ✅无 perm | action 配置缺 permission 键，授权绕过 |
| ⚠️基类直接 save 无 catch | 继承抽象基类 enable/disable，直接 `$item->save()`，不走 Job，无 try/catch |
| ❌直接 Model 无 catch | 直接 `$item->delete()` 或 `$item->duplicate()`，不走 Job，无 try/catch |
| ✨merge 污染 | foreach 内 `$request->merge(...)` 存在交叉污染风险 |
| ⚠️归属 skip | `isNotUserCompany()` 检查用 continue 静默跳过，不计失败 |

### 6.4 15 子类 × 8 类问题关联矩阵

| 子类 | ①export 无perm | ②try/catch 漏 | ③duplicate 跳Job | ⑤模块差异 | ⑥merge污染 |
|------|---------------|--------------|-----------------|----------|------------|
| Invoices | ✅export/download 都有 | — | ✅直接Model无catch | — | ✅update 有 |
| Customers | ✅export 有 | — | — | — | ✅update 有 |
| Bills | ✅export 有 | ✅received/cancelled无catch | ✅直接Model无catch | — | ✅update 有 |
| Vendors | ✅export 有 | — | — | — | ✅update 有 |
| Items | ✅export 有 | ✅enable 基类无catch | — | — | ✅update 有 |
| Companies | — | — | — | — | ✅update 有 + skip |
| Dashboards | — | ✅enable 基类无catch | — | — | — |
| Categories | — | ✅enable 基类无catch | — | — | ✅update 有 |
| Taxes | — | ✅enable 基类无catch | — | — | — |
| Currencies | — | ✅enable 基类无catch | — | — | ✅update 有 |
| Accounts | — | ✅enable 基类无catch | — | — | ✅update 有 |
| Transactions | ✅export 有 | — | — | — | ✅update 有 |
| Transfers | ✅export 有 | — | — | — | — |
| Reconciliations | — | ✅reconcile/unreconcile/destroy 全漏 | — | — | — |
| Users | — | — | — | — | ✅update 有 |
| **合计** | **8 处** | **8 处** | **2 处** | **模块专属** | **10 处** |

---

## 六处问题总结表

| # | 问题 | 影响范围 | 严重程度 | 本质 |
|---|------|---------|---------|------|
| 1 | **export/download 缺 permission 键绕授权** | 8 个子类的 export/download 批量操作 | 🔥 高 | 基类和所有子类 action 配置都未定义 permission，`isset()` 短路放行，任何登录后台用户都能批量导出全部敏感数据 |
| 2 | **abstract 默认方法漏 try/catch 抛断 batch** | 基类 destroy/enable/disable/duplicate；Reconciliations 全部方法；Bills/Invoices 的 received/cancelled | 🔥 高 | 单条异常直接冒泡中断整批，已处理记录不回滚，后续记录全部不处理 |
| 3 | **duplicate 完全跳 Job authorize** | Invoices/Bills 批量 duplicate | ⚠️ 中高 | 直接 `$model->duplicate()`（bkwld/cloner 纯 DB 复制），完全不走 DuplicateDocument Job，自然也没有任何业务校验 |
| 4 | **flash 计 not_passed 跨请求残留与混算** | 所有批量操作的结果统计 | ⚠️ 高 | Session 中上一次请求的 flash 消息被计入；warning 与 danger 混算；单条失败多条消息放大计数；静默失败不计入 |
| 5 | **module 分支 dispatch 模块 BulkActions** | 第三方模块的 BulkAction 类 | ⚠️ 中 | 命名空间解析、权限命名约定、是否继承 AbstractBulkAction 均依赖模块开发者自律，安全保障弱于 App 分支 |
| 6 | **15 子类 handle 全量矩阵** | 全部 15 个 BulkAction 子类 | 🔍 全景 | 各子类实现高度不一致：8 处缺 try/catch、10 处 request merge 污染、8 处 export 无 permission、2 处 duplicate 跳 Job |

---

## 相关代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| 抽象基类 actions 定义（无 export/download perm） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L25-L51 |
| 控制器授权检查（isset 短路） | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L50-L63 |
| 抽象基类 duplicate（直接 Model，无 try/catch） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L152-L159 |
| 抽象基类 enable（直接 save，无 try/catch） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L168-L175 |
| 抽象基类 disable（直接 save，无 try/catch） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L185-L193 |
| 抽象基类 destroy（直接 delete，无 try/catch） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L214-L221 |
| 基类 disableContacts（有 try/catch + Job） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L223-L234 |
| Invoices 批量 duplicate（直接 Model，无 try/catch） | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | L128-L152 |
| Bills received/cancelled（event 无 catch） | [Bills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Bills.php) | L96-L119 |
| Reconciliations 全部方法无 try/catch | [Reconciliations.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Reconciliations.php) | L41-L95 |
| flash not_passed 统计逻辑 | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L67-L74 |
| 控制器模块分支解析 | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L35-L48 |
| 视图层模块分支解析 | [ViewComponents.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Traits/ViewComponents.php) | L505-L520 |
| DuplicateDocument Job（无 authorize） | [DuplicateDocument.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Jobs/Document/DuplicateDocument.php) | L21-L36 |
| Customers update merge 污染 | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php) | L63-L79 |
| Companies update 归属静默 skip | [Companies.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Companies.php) | L63-L82 |
| Invoices actions（export/download 无 perm） | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | L53-L64 |
| bkwld/cloner Cloneable trait 使用 | [Document.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Models/Document/Document.php) | L16、L24、L101 |
| 权限中间件自动绑定（批量操作不走此路径） | [Permissions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Traits/Permissions.php) | L425-L500 |
