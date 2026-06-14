# 批量操作 Ghost Handle 终版名单与威胁模型

## 核心纠偏

| 之前结论 | 纠正后事实 | 依据 |
|---------|-----------|------|
| 基类有 export/download 方法 | **基类没有 export/download 方法**，只有 `$actions` 配置中有这两个 key | 基类实际有 `exportExcel()`/`downloadPdf()` 工具方法，但参数签名不同，不能直接当 handle 调用 |
| ghost 入口全是绕授权 | 分两类：**无配置无方法 = 500 报错**、**有方法无配置 = 真正的 ghost** | 控制器直接 `$obj->{$handle}()`，方法不存在则 BadMethodCallException |
| flash 残留是跨请求 session | **是同请求内前置累积**，不是跨请求 | `flash()->messages` 读的是当前请求内存中的消息集合 |
| 外部 CSRF 可利用 | **必须认证 + CSRF token**，属于内部提权，不是外部伪造 | admin 组继承 web 组，web 组有 `csrf` 中间件，`$except` 为空 |

---

## 一、Ghost Handle 终版名单

### 1.1 Ghost Handle 的定义

**Ghost Handle = 在 `$actions` 配置数组中没有对应键（因此前端不显示、也不做权限检查），但 BulkAction 类上存在可被调用的 public 方法**。

判定条件：
1. ✅ 类上存在该 public 方法
2. ✅ 方法接受 `$request` 参数（可被控制器以 `$bulk_actions->{$handle}($request)` 调用）
3. ❌ `$actions` 配置数组中没有以该 handle 为名的键
4. ❌ action 配置中没有 `permission` 键 → 控制器权限检查短路跳过

### 1.2 抽象基类中的全部 public 方法清单

[BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php)

| 方法名 | 方法签名 | 在 $actions 中？ | $actions 中有 permission？ | 是不是 ghost |
|--------|---------|-----------------|--------------------------|-------------|
| `getSelectedRecords` | `($request, $relationships = null)` | 否（工具方法） | — | ❌ 调用返回值，不是操作 |
| `getSelectedInput` | `($request)` | 否（工具方法） | — | ❌ 同上 |
| `getUpdateRequest` | `($request)` | 否（工具方法） | — | ❌ 同上 |
| `response` | `($view, $data = [])` | 否（工具方法） | — | ❌ 参数不对 |
| `importExcel` | `($class, $request, $translation)` | 否（工具方法） | — | ❌ 参数不对 |
| `exportExcel` | `($class, $translation, $extension)` | 否（工具方法） | — | ❌ 参数不对 |
| `downloadPdf` | `($selected, $class, $file_name, $translation)` | 否（工具方法） | — | ❌ 参数不对 |
| **`duplicate`** | **`($request)`** | **否** | — | **✅ GHOST** |
| **`enable`** | **`($request)`** | **是（有配置）** | **是**（`update-common-items`） | ❌ 不是 ghost（有配置有 permission） |
| **`disable`** | **`($request)`** | **是（有配置）** | **是**（`update-common-items`） | ❌ 不是 ghost（有配置有 permission） |
| **`delete`** | **`($request)`** | **是（有配置）** | **是**（`delete-common-items`） | ❌ 不是 ghost（有配置有 permission） |
| **`destroy`** | **`($request)`** | **否**（只有 delete 有） | — | **✅ GHOST** |
| **`disableContacts`** | **`($request)`** | **否** | — | **✅ GHOST** |
| **`deleteContacts`** | **`($request)`** | **否** | — | **✅ GHOST** |
| **`deleteTransactions`** | **`($request)`** | **否** | — | **✅ GHOST** |
| `export` | — | 是（有配置） | 否（缺 permission 键） | — | **基类无此方法** |
| `download` | — | 是（有配置） | 否（缺 permission 键） | — | **基类无此方法** |

### 1.3 基类导出的 5 个 Ghost Handle

| # | Ghost Handle | 对应基类方法 | 实际行为 | 权限检查 | 严重度 |
|---|-------------|-------------|---------|---------|--------|
| 1 | `duplicate` | [BulkAction.php:152-159](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L152-L159) | 直接 `$item->duplicate()`（bkwld/cloner 复制），不走 Job，无 try/catch，无业务校验 | ❌ 完全跳过（$actions 中无 duplicate 键） | 🔥 高 |
| 2 | `destroy` | [BulkAction.php:214-221](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L214-L221) | 直接 `$item->delete()`，不走 Job，无 try/catch，绕所有业务校验 | ❌ 完全跳过（$actions 中只有 delete 键，没有 destroy 键） | 🔥 高 |
| 3 | `disableContacts` | [BulkAction.php:223-234](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L223-L234) | 走 `UpdateContact` Job + try/catch，有业务校验 | ❌ 完全跳过（$actions 中无此键） | ⚠️ 中 |
| 4 | `deleteContacts` | [BulkAction.php:236-247](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L236-L247) | 走 `DeleteContact` Job + try/catch，有业务校验 | ❌ 完全跳过（$actions 中无此键） | ⚠️ 中 |
| 5 | `deleteTransactions` | [BulkAction.php:249-260](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L249-L260) | 走 `DeleteTransaction` Job + try/catch | ❌ 完全跳过（$actions 中无此键） | ⚠️ 中 |

### 1.4 关于 export/download 的纠正

**之前的错误**：认为基类有 `export` 和 `download` 方法，所以所有子类都是 ghost。

**正确事实**：

- 基类 `$actions` 配置中有 `export` 和 `download` 两个键（含配置项）
- 但基类**没有** `public function export($request)` 和 `public function download($request)` 方法
- 如果子类也没有覆盖这两个方法，POST `handle=export` 会触发 `BadMethodCallException` → 500 错误
- 只有**子类自己实现了 export/download 方法**的那些，才能被调用

**有 export 方法的子类（8 个）**：

| 子类 | 有 export 方法？ | 有 download 方法？ | actions 中有 permission？ |
|------|----------------|------------------|-------------------------|
| [Invoices](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | ✅ 有 | ✅ 有 | ❌ 都没有 |
| [Bills](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Bills.php) | ✅ 有 | ❌ 无 | ❌ export 没有 |
| [Customers](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php) | ✅ 有 | ❌ 无 | ❌ export 没有 |
| [Vendors](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Purchases/Vendors.php) | ✅ 有 | ❌ 无 | ❌ export 没有 |
| [Items](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php) | ✅ 有 | ❌ 无 | ❌ export 没有 |
| [Transactions](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transactions.php) | ✅ 有 | ❌ 无 | ❌ export 没有 |
| [Transfers](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Banking/Transfers.php) | ✅ 有 | ❌ 无 | ❌ export 没有 |
| Companies | ❌ 无（继承基类=无方法） | ❌ 无 | — |
| Dashboards | ❌ 无 | ❌ 无 | — |
| Categories | ❌ 无 | ❌ 无 | — |
| Taxes | ❌ 无 | ❌ 无 | — |
| Currencies | ❌ 无 | ❌ 无 | — |
| Accounts | ❌ 无 | ❌ 无 | — |
| Reconciliations | ❌ 无 | ❌ 无 | — |
| Users | ❌ 无 | ❌ 无 | — |

> 分类：`export`/`download` 不属于 "ghost handle"（因为 $actions 中有配置键），属于 "有配置但缺 permission 键的弱权限入口"。

---

## 二、15 子类 × Ghost Handle 可达性矩阵

### 2.1 5 个基类 Ghost Handle 对子类的可达性

| Ghost Handle | 对哪些子类有效 | 说明 |
|-------------|---------------|------|
| `duplicate` | **全部 15 个** | 基类有默认实现，所有子类继承 |
| `destroy` | **全部 15 个** | 基类有默认实现，部分子类覆盖（但方法仍存在） |
| `disableContacts` | **有 Contact 类型的子类**（Customers/Vendors） | 调用 `$this->getSelectedRecords($request, 'user')`，非 Contact 模型可能报错或行为异常 |
| `deleteContacts` | 同上 | 同上 |
| `deleteTransactions` | **有 Transaction 类型的子类**（Transactions 等） | 同上 |

### 2.2 Items 类的几个不确定点补全

针对 [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php) 的核对：

| 方法 | 来源 | 实现方式 | 权限检查 |
|------|------|---------|---------|
| `disable` | 继承基类 | 直接 `$item->enabled = false; $item->save()`，**不走 UpdateItem Job**，无 try/catch | ✅ 有（$actions 中有配置+permission） |
| `enable` | 继承基类 | 直接 `$item->enabled = true; $item->save()`，不走 Job | ✅ 有 |
| `destroy` | 子类覆盖 | `dispatch(new DeleteItem($item))` + try/catch，走 Job | ❌ 没有（$actions 中叫 delete，不叫 destroy） |
| `delete` | 继承基类 | 内部调用 `$this->destroy($request)` | ✅ 有 |
| `export` | 子类覆盖 | `exportExcel(...)` | ❌ 缺 permission 键 |
| `download` | 不存在（基类也没有） | 调用会 500 报错 | — |
| `duplicate` | 继承基类 | 直接 `$item->duplicate()` | ❌ 没有（ghost） |

### 2.3 15 子类全量 Ghost Handle 矩阵

| 子类 | duplicate | destroy | disableContacts | deleteContacts | deleteTransactions | 说明 |
|------|-----------|---------|-----------------|-----------------|--------------------|------|
| **Sales** | | | | | | |
| Invoices | ✅ GHOST<br>（子类覆盖，更危险） | ✅ GHOST<br>（子类覆盖，走Job但绕权限） | ❌ 模型不匹配<br>（Document 不是 Contact） | ❌ | ❌ | duplicate 直接 `$invoice->duplicate()` + CreateDocumentHistory |
| Customers | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ✅ GHOST<br>（走 UpdateContact Job） | ✅ GHOST<br>（走 DeleteContact Job） | ❌ | 子类 disable() 调用 `disableContacts()` 但那是普通方法调用，不是 ghost |
| **Purchases** | | | | | | |
| Bills | ✅ GHOST<br>（子类覆盖） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | 同 Invoices |
| Vendors | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ✅ GHOST | ✅ GHOST | ❌ | 同 Customers |
| **Common** | | | | | | |
| Items | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | disable 继承基类（直接save，绕Job） |
| Companies | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | enable/disable 子类覆盖，走 Job |
| Dashboards | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | enable 继承基类（直接save） |
| **Settings** | | | | | | |
| Categories | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | disable 子类覆盖走 Job，enable 继承基类直接save |
| Taxes | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | disable/destroy 子类覆盖走 Job |
| Currencies | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | 同 Taxes |
| **Banking** | | | | | | |
| Accounts | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | enable 继承基类直接save，disable/destroy子类覆盖走Job |
| Transactions | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ✅ GHOST<br>（模型匹配） | deleteTransactions 走 DeleteTransaction Job |
| Transfers | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | |
| Reconciliations | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖，但直接save无Job） | ❌ | ❌ | ❌ | destroy 子类覆盖，直接 DB::transaction 无 try/catch |
| **Auth** | | | | | | |
| Users | ✅ GHOST<br>（继承基类） | ✅ GHOST<br>（子类覆盖） | ❌ | ❌ | ❌ | |

---

## 三、disableContacts 绕子类 disable 权限的实例分析

### 3.1 普通路径（有授权）

```
POST handle=disable
  → 控制器检查：isset(actions['disable']['permission']) = true
  → user()->can('update-sales-customers') ?
  → 有权 → 调用 Customers::disable()
  → 内部调用 $this->disableContacts($request)
  → 走 UpdateContact Job + authorize()
```

权限受控：✅ 有 `update-sales-customers` 权限才能执行。

### 3.2 Ghost 路径（绕授权）

```
POST handle=disableContacts
  → 控制器检查：isset(actions['disableContacts']['permission']) = false
  → （actions 数组中根本没有 disableContacts 这个键）
  → 条件短路 → 跳过权限检查
  → 直接调用 BulkAction::disableContacts($request)
  → 走 UpdateContact Job + authorize()
```

权限受控：❌ **完全跳过**。只要登录后台（有 `read-admin-panel`）就能禁用任意客户。

### 3.3 两种路径的差异

| 对比项 | handle=disable（正常路径） | handle=disableContacts（ghost 路径） |
|--------|---------------------------|------------------------------------|
| 权限检查 | ✅ 检查 `update-sales-customers` | ❌ 完全跳过 |
| 走 Job 吗 | ✅ 走 UpdateContact Job | ✅ 走 UpdateContact Job |
| 业务校验（authorize） | ✅ 有 | ✅ 有 |
| try/catch | ✅ 有 | ✅ 有 |
| 前端可见 | ✅ 有按钮 | ❌ 无按钮 |

> 关键点：`disableContacts` 的实现本身是安全的（走 Job + try/catch + 业务校验），**问题只在于它是 public 的且不在 $actions 白名单中，导致权限检查被跳过**。

### 3.4 deleteContacts / deleteTransactions 同理

| Ghost 方法 | 可被哪些模型的 BulkAction 安全调用 | 绕的是哪个 action 的权限 |
|-----------|--------------------------------|-----------------------|
| `disableContacts` | Customers、Vendors（Contact 模型） | disable 操作的权限 |
| `deleteContacts` | Customers、Vendors（Contact 模型） | delete 操作的权限 |
| `deleteTransactions` | Transactions（Transaction 模型） | delete 操作的权限 |

这些方法的实现**质量反而比基类默认的 enable/disable/destroy 更高**（走 Job + 有 try/catch + 有业务校验），但因为是 public 且没在 `$actions` 中注册，变成了绕权限的后门。

---

## 四、威胁模型：认证 + CSRF 保护下的提权

### 4.1 前置条件核查

#### 认证层

admin 中间件组 [Kernel.php:76-88](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php#L76-L88)：

```php
'admin' => [
    'web',
    'auth',                     // ✅ 必须登录
    'auth.disabled',            // ✅ 用户未被禁用
    'company.identify',
    'bindings',
    'read.only',
    'wizard.redirect',
    'menu.admin',
    'permission:read-admin-panel',  // ✅ 必须有后台访问权限
    'plan.limits',
    'module.subscription',
],
```

**结论**：必须是已登录且拥有 `read-admin-panel` 权限的后台用户。

#### CSRF 层

web 中间件组 [Kernel.php:32-43](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php#L32-L43)：

```php
'web' => [
    'cookies.encrypt',
    'cookies.response',
    'session.start',
    'session.errors',
    'csrf',                 // ✅ CSRF 中间件
    'install.redirect',
    'header.x',
    'language',
    'firewall.all',
],
```

`VerifyCsrfToken` [VerifyCsrfToken.php:21-23](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Middleware/VerifyCsrfToken.php#L21-L23) 的 `$except` 为空，没有排除任何路由。

**结论**：必须携带有效的 CSRF token（`X-CSRF-TOKEN` header 或 `_token` 表单字段）。

### 4.2 攻击路径定性

| 攻击类型 | 是否适用 | 理由 |
|---------|---------|------|
| **外部 CSRF 攻击** | ❌ 不适用 | 有 CSRF token 保护，第三方网站无法构造有效请求 |
| **XSS + CSRF 绕过** | ⚠️ 理论可行 | 如果存在 XSS 漏洞，可以读取 CSRF token 后构造请求 |
| **水平提权** | ✅ 适用 | 低权限用户（仅有 `read-admin-panel`）可以执行本不该有的操作 |
| **功能滥用** | ✅ 适用 | 合法用户利用未公开的接口做超过预期的操作 |

### 4.3 典型攻击场景

**场景：低权限运营批量复制发票**

- 攻击者角色：运营人员，仅有 `read-sales-invoices`（查看）权限，无 `create-sales-invoices`（创建）权限
- 攻击方式：
  1. 登录后台，浏览发票列表，获取 CSRF token（从页面 meta 标签或 cookie 中）
  2. 构造 AJAX 请求：
     ```
     POST /{company_id}/bulk-actions/sales/invoices
     X-CSRF-TOKEN: <token>
     Content-Type: application/x-www-form-urlencoded
     
     handle=duplicate
     selected[]=101
     selected[]=102
     selected[]=103
     ```
  3. 因为 `duplicate` 是 ghost handle（$actions 中无配置 → 权限检查短路），请求直接执行
  4. 100 张发票被复制，攻击者获得了 100 张"新"发票草稿

**场景：只读用户批量删除客户**

- 攻击者角色：审计人员，仅有查看权限
- 攻击方式：POST `handle=deleteContacts` 直接删除客户
- 虽然 DeleteContact Job 有 authorize 业务校验（有关联数据的客户不能删），但**用户权限层面的检查完全跳过了**

### 4.4 风险等级评估

| Ghost Handle | 攻击前置 | 影响 | 风险等级 |
|-------------|---------|------|---------|
| `duplicate` | read-admin-panel + CSRF token | 批量复制任意资源，可能绕过创建权限 | 🔥 **高** |
| `destroy` | read-admin-panel + CSRF token | 批量删除任意资源，绕 delete 权限 | 🔥 **高** |
| `export`（有方法的子类） | read-admin-panel + CSRF token | 批量导出敏感数据 | 🔥 **高**（8个子类） |
| `download`（Invoices 独有） | read-admin-panel + CSRF token | 批量下载 PDF | ⚠️ 中高 |
| `disableContacts` | read-admin-panel + CSRF token + 目标是 Contact 模型 | 批量禁用客户/供应商 | ⚠️ 中 |
| `deleteContacts` | read-admin-panel + CSRF token + 目标是 Contact 模型 | 批量删除客户/供应商（有业务校验兜底） | ⚠️ 中 |
| `deleteTransactions` | read-admin-panel + CSRF token + 目标是 Transaction 模型 | 批量删除交易（有业务校验兜底） | ⚠️ 中 |
| `enable`（继承基类的） | read-admin-panel + CSRF token | 批量启用，绕 Job 业务校验（但有 permission 检查） | ❌ 低（有 permission） |

---

## 五、修复建议

### 5.1 最高优先级：加白名单

在控制器入口加 handle 白名单校验：

```php
// BulkActions.php action() 开头
$handle = $request->get('handle', '*');

if (! isset($bulk_actions->actions[$handle])) {
    abort(404);  // 不在 $actions 白名单中的 handle 直接 404
}
```

这一条修复可以消除所有 ghost handle（duplicate / destroy / disableContacts / deleteContacts / deleteTransactions）。

### 5.2 次高优先级：export/download 补 permission

为所有子类的 export/download action 补上 permission 键，使用 `read-` 前缀的现有权限：

```php
'export' => [
    'icon' => 'file_download',
    'name' => 'general.export',
    'message' => 'bulk_actions.message.export',
    'type' => 'download',
    'permission' => 'read-sales-invoices',  // 补上
],
```

### 5.3 建议性：disableContacts 等改为 protected

`disableContacts` / `deleteContacts` / `deleteTransactions` 是**内部复用方法**，不应该是 public 的。改为 `protected` 可以防止外部直接调用。

### 5.4 建议性：基类 enable/disable 走 Job

基类的 `enable()` / `disable()` 直接 `$item->save()` 绕 Job 业务校验的问题，与 ghost 入口是两个独立问题。即使修复了白名单（handle 必须在 actions 中），enable/disable 仍然绕 Job 校验。

建议基类默认实现也走对应 Job，或标记为 @deprecated 要求子类必须覆盖。

---

## 六、关键代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| 控制器入口（无白名单，直接调用方法） | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L21-L65 |
| 控制器权限检查（isset 短路） | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L50-L63 |
| 基类 $actions 配置 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L25-L51 |
| 基类 duplicate 方法（ghost #1） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L152-L159 |
| 基类 destroy 方法（ghost #2） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L214-L221 |
| 基类 disableContacts 方法（ghost #3） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L223-L234 |
| 基类 deleteContacts 方法（ghost #4） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L236-L247 |
| 基类 deleteTransactions 方法（ghost #5） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L249-L260 |
| 基类 enable/disable 方法 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L168-L193 |
| admin 中间件组 | [Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php) | L76-L88 |
| web 中间件组（含 csrf） | [Kernel.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Kernel.php) | L32-L43 |
| CSRF 中间件（$except 为空） | [VerifyCsrfToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Middleware/VerifyCsrfToken.php) | L21-L23 |
| Items BulkAction（disable 继承基类） | [Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Common/Items.php) | L11-L100 |
| Invoices duplicate（子类覆盖） | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | L128-L152 |
| Customers disable（委托基类方法） | [Customers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Customers.php) | L81-L84 |
