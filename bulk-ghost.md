# 批量动作 Ghost 入口与 Flash 累积深度分析

## 一、duplicate 在多数子类未注册却继承基类仍可 POST 触发

### 1.1 问题现象

`duplicate` 批量操作在绝大多数子类的 `$actions` 配置数组中**根本不存在**，但用户仍然可以通过构造 POST 请求直接触发。

### 1.2 基类默认方法

[BulkAction.php:152-159](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L152-L159)

```php
public function duplicate($request)
{
    $items = $this->getSelectedRecords($request);

    foreach ($items as $item) {
        $item->duplicate();  // bkwld/cloner 的 Cloneable trait 方法
    }
}
```

这个方法在抽象基类中**默认存在**，所有继承 `BulkAction` 的子类自动获得。

### 1.3 基类 $actions 数组中无 duplicate 配置

[BulkAction.php:25-51](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php#L25-L51)

基类的 `$actions` 只有：`enable`、`disable`、`delete`、`export`、`download` —— **没有 duplicate**。

### 1.4 子类也大多不注册 duplicate

遍历 15 个子类的 `$actions` 数组：

| 子类 | actions 中有 duplicate？ | 类中有 duplicate() 方法？ |
|------|-------------------------|--------------------------|
| Invoices | ❌ 无 | ✅ 有（覆盖基类，直接 `$invoice->duplicate()`） |
| Bills | ❌ 无 | ✅ 有（覆盖基类，直接 `$bill->duplicate()`） |
| Customers | ❌ 无 | ❌ 无（继承基类默认实现） |
| Vendors | ❌ 无 | ❌ 无（继承基类默认实现） |
| Items | ❌ 无 | ❌ 无（继承基类默认实现） |
| Taxes | ❌ 无 | ❌ 无（继承基类默认实现） |
| Categories | ❌ 无 | ❌ 无（继承基类默认实现） |
| Currencies | ❌ 无 | ❌ 无（继承基类默认实现） |
| Accounts | ❌ 无 | ❌ 无（继承基类默认实现） |
| Transactions | ❌ 无 | ❌ 无（继承基类默认实现） |
| Transfers | ❌ 无 | ❌ 无（继承基类默认实现） |
| Companies | ❌ 无 | ❌ 无（继承基类默认实现） |
| Dashboards | ❌ 无 | ❌ 无（继承基类默认实现） |
| Reconciliations | ❌ 无 | ❌ 无（继承基类默认实现） |
| Users | ❌ 无 | ❌ 无（继承基类默认实现） |

**结论：15 个子类中，0 个在 $actions 中注册了 duplicate，但所有 15 个都有 duplicate() 方法（要么继承要么覆盖）。**

### 1.5 触发方式

由于控制器直接调用方法名（详见第三节），只需构造 POST 请求：

```http
POST /{company_id}/bulk-actions/sales/invoices
Content-Type: application/x-www-form-urlencoded

handle=duplicate
selected[]=1
selected[]=2
selected[]=3
```

即可触发批量复制，即使：
- 前端没有显示复制按钮
- `$actions` 配置中没有 duplicate 项
- 权限检查因 `isset()` 短路而完全跳过

### 1.6 附带问题：完全无授权校验

因为 `$actions` 中没有 `duplicate` 键，控制器的权限检查：

[BulkActions.php:50-53](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L50-L53)

```php
if (
    isset($bulk_actions->actions[$handle]['permission'])  // duplicate 键不存在 → false
    && ! user()->can($bulk_actions->actions[$handle]['permission'])
) {
    // 403，不会执行
}
```

`isset()` 返回 `false` → 整个条件短路 → **直接跳过权限检查** → 执行方法。

**只要能登录后台（通过 `read-admin-panel`），就能对任意资源批量复制。**

---

## 二、enable/disable/delete/destroy 等基类方法让 15 子类形成 ghost 入口

### 2.1 什么是 ghost 入口

**Ghost 入口** = 在 `$actions` 配置数组中没有注册、但通过继承基类方法仍然可以通过 POST 直接调用的批量操作入口。

这些入口不会在前端按钮中显示（因为视图层遍历 `$actions` 渲染按钮），但后端控制器不校验 `$handle` 是否在 `$actions` 白名单中，直接调用方法。

### 2.2 基类提供的所有可调用方法

抽象基类 [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) 中所有 public 且接受 `$request` 参数的方法：

| 方法 | 是否在基类 $actions 中 | 基类默认实现 | 权限键 |
|------|----------------------|-------------|--------|
| `duplicate($request)` | ❌ | `$item->duplicate()` 直接 Model 复制 | 无（ghost） |
| `enable($request)` | ✅ | `$item->enabled = true; $item->save()` | `update-common-items` |
| `disable($request)` | ✅ | `$item->enabled = false; $item->save()` | `update-common-items` |
| `delete($request)` | ✅ | 内部调用 `$this->destroy($request)` | `delete-common-items` |
| `destroy($request)` | ❌（delete 才有） | `$item->delete()` 直接删 | 无（ghost） |
| `export($request)` | ✅ | 走 `Export` 类导出 | 无（缺 permission 键） |
| `download($request)` | ✅ | 基类无默认实现？需确认 | 无（缺 permission 键） |
| `disableContacts($request)` | ❌ | 走 `UpdateContact` Job | 无（内部方法） |
| `deleteContacts($request)` | ❌ | 走 `DeleteContact` Job | 无（内部方法） |
| `deleteTransactions($request)` | ❌ | 走 `DeleteTransaction` Job | 无（内部方法） |

注意：`destroy` 是方法名，`delete` 是 action 配置名。action 配置中可以通过 `handle` 字段指定调用的方法名。

### 2.3 子类覆盖与 ghost 入口矩阵

15 个子类 × 7 个基类可调用方法的覆盖情况：

| 子类 | duplicate | enable | disable | delete | destroy | export | download |
|------|-----------|--------|---------|--------|---------|--------|----------|
| **Sales** | | | | | | | |
| Invoices | ✅覆盖 | ❌继承 | ❌继承 | ❌继承 | ✅覆盖 | ✅覆盖 | ✅覆盖 |
| Customers | ❌继承 | ❌继承 | ✅覆盖 | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 |
| **Purchases** | | | | | | | |
| Bills | ✅覆盖 | ❌继承 | ❌继承 | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 |
| Vendors | ❌继承 | ❌继承 | ✅覆盖 | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 |
| **Common** | | | | | | | |
| Items | ❌继承 | ❌继承（直接save） | ❌继承？ | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 |
| Companies | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| Dashboards | ❌继承 | ❌继承 | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| **Settings** | | | | | | | |
| Categories | ❌继承 | ❌继承（直接save） | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| Taxes | ❌继承 | ❌继承（直接save） | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| Currencies | ❌继承 | ❌继承（直接save） | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| **Banking** | | | | | | | |
| Accounts | ❌继承 | ❌继承（直接save） | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| Transactions | ❌继承 | ❌继承 | ❌继承 | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 |
| Transfers | ❌继承 | ❌继承 | ❌继承 | ❌继承 | ✅覆盖 | ✅覆盖 | ❌继承 |
| Reconciliations | ❌继承 | ❌继承 | ❌继承 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |
| **Auth** | | | | | | | |
| Users | ❌继承 | ❌继承 | ✅覆盖 | ❌继承 | ✅覆盖 | ❌继承 | ❌继承 |

✅ = 子类自己覆盖实现  ❌ = 继承基类默认实现

### 2.4 ghost 入口的风险等级

| ghost 入口 | 风险 | 原因 |
|-----------|------|------|
| **duplicate** | 🔥 高 | 15 个子类全有，全部无权限检查，直接 DB 复制，绕所有业务校验 |
| **destroy** | 🔥 高 | 15 个子类全有（基类默认+部分覆盖），基类默认 `$item->delete()` 直接删无 try/catch |
| **enable** | ⚠️ 中 | 基类 `$actions` 中有配置且有 permission，但部分子类直接继承基类实现，绕 Job 业务校验 |
| **disable** | ⚠️ 中 | 同上 |
| **export** | 🔥 高 | 已在 bulk-residual.md 详述：缺 permission 键，绕授权 |
| **download** | 🔥 高 | 同上，部分子类（如 Invoices）覆盖但无 permission |

### 2.5 一个隐蔽的例子：destroy vs delete

`delete` 是 action 配置名，`destroy` 是方法名。

基类 action 配置中 `delete` 有 permission：
```php
'delete' => [
    'permission' => 'delete-common-items',
],
```

基类方法 `delete()` 内部调用 `destroy()`：
```php
public function delete($request)
{
    $this->destroy($request);
}
```

**问题**：用户可以直接 POST `handle=destroy` 绕过 `delete` action 的权限检查。

| POST handle | 是否经权限检查 | 调用的方法 |
|-------------|---------------|-----------|
| `delete` | ✅ 是（有 permission 键） | `delete()` → `destroy()` |
| `destroy` | ❌ 否（actions 中无 destroy 键） | 直接 `destroy()` |

即使子类覆盖了 `destroy()` 方法（比如走 Job + try/catch），只要 `$actions` 中没有 `destroy` 键，POST `handle=destroy` 就会跳过权限检查。

---

## 三、controller 未把 actions 当白名单校验 handle

### 3.1 控制器入口核心代码

[BulkActions.php:21-65](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L21-L65)

```php
public function action($group, $type, Request $request)
{
    $handle = $request->get('handle', '*');

    if ($handle == '*') {
        return response()->json([...]);  // 只拒绝了 '*'
    }

    // ... 解析 $bulk_actions 类 ...

    if (
        isset($bulk_actions->actions[$handle]['permission'])
        && ! user()->can($bulk_actions->actions[$handle]['permission'])
    ) {
        // 403
    }

    $result = $bulk_actions->{$handle}($request);  // 直接调用，无白名单校验
    // ...
}
```

### 3.2 三层逻辑拆解

| 步骤 | 逻辑 | 作用 |
|------|------|------|
| ① handle ≠ '*' | 只过滤了默认值 `*` | 防误操作，不是安全校验 |
| ② 权限检查（条件式） | `isset(actions[handle].permission) && ! can(...)` | **只有当 handle 在 actions 中且有 permission 键时才检查** |
| ③ 直接调用 | `$bulk_actions->{$handle}($request)` | 无白名单，方法存在就执行 |

### 3.3 安全缺陷图谱

根据 `$handle` 的不同取值，会触发不同的行为：

| handle 取值 | 在 actions 中？ | 有 permission 键？ | 权限检查结果 | 方法存在吗？ | 最终结果 |
|------------|---------------|-------------------|------------|------------|---------|
| `delete` | ✅ 是 | ✅ 是 | ✅ 执行检查 | ✅ 是 | 正常：有权则执行，无权则 403 |
| `export` | ✅ 是 | ❌ 否 | ❌ 跳过检查 | ✅ 是 | 危险：无授权直接执行 |
| `duplicate` | ❌ 否 | 不适用 | ❌ 跳过检查 | ✅ 是（继承） | 危险：ghost 入口，无授权 |
| `destroy` | ❌ 否（只有 delete） | 不适用 | ❌ 跳过检查 | ✅ 是 | 危险：ghost 入口，无授权 |
| `enable` | ✅ 是 | ✅ 是（基类默认） | ✅ 执行检查 | ✅ 是 | 正常：有授权检查 |
| `随便写的方法名` | ❌ 否 | 不适用 | ❌ 跳过检查 | ❌ 否 | 500 错误：BadMethodCallException |

### 3.4 白名单缺失的连锁反应

缺少 `handle ∈ array_keys($actions)` 的白名单校验，导致：

1. **所有基类方法都是 ghost 入口**：duplicate、destroy（绕过 delete 的 permission）等
2. **permission 键缺失的 action 直接放行**：export、download 等
3. **模块自定义方法无保障**：模块新增的 public 方法即使不在 actions 配置中也能被调用
4. **权限与方法脱节**：权限绑定的是 action 配置，不是方法名。配置与实现可以不一致
5. **类型混淆风险**：`handle` 参数只是 `required|string`，没有枚举校验

### 3.5 对比：视图层有白名单（隐式）

视图层 [Bulkaction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/View/Components/Index/Bulkaction.php#L42-L107) 通过遍历 `$bulk_action->actions` 渲染按钮，天然是白名单。

但**前后端白名单不一致**：
- 前端：白名单 = `$actions` 的 keys
- 后端：无白名单，所有 public 方法都可调用

---

## 四、flash 计 not_passed 残留实际来自同请求前置 flash 累积

### 4.1 对"跨请求 session 残留"的纠正

之前的分析认为 `flash()` 消息存在 Session 中，上一次请求的消息会被本次请求统计到 —— **这个理解是不准确的**。

实际情况是：**`flash()->messages` 返回的是当前请求生命周期内累积的内存消息集合**，不是从 Session 中读取的上一次请求的消息。

### 4.2 laracasts/flash 的工作机制

根据 `laracasts/flash` 3.x 的设计：

```
当前请求内存:
  FlashNotifier 实例
    └─ $messages 数组 (内存)
          ├─ 调用 flash('msg')->error()  → push 一条
          ├─ 调用 flash()->messages      → 返回当前所有
          └─ 请求结束时 → 存入 Session 的 flash 域 → 下一次请求显示
```

| 特性 | 说明 |
|------|------|
| **内存累积** | 同一个请求内，多次 `flash()` 调用都 push 到同一个 `$messages` 数组 |
| **当前请求可读** | `flash()->messages` 立即可读，能读到本请求中之前 flash 的所有消息 |
| **Session 持久化** | 请求结束时，所有消息被存入 Session，供下一次请求视图显示 |
| **非跨请求读取** | `flash()->messages` 不读 Session 中的历史消息，只读本请求内存中的 |

### 4.3 not_passed 统计的时机与范围

[BulkActions.php:65-74](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php#L65-L74)

```php
// 时间点 A: 执行 handler
$result = $bulk_actions->{$handle}($request);
// └─ foreach 内 catch { flash($e->getMessage())->error() } → push 到 messages

// 时间点 B: 统计 not_passed
flash()->messages->each(function ($message) use (&$not_passed) {
    if (in_array($message->level, ['danger', 'warning'])) {
        $not_passed++;
    }
});
```

**时间点 B 统计到的 messages = 时间点 A 执行过程中 push 的所有消息 + 时间点 A 之前就已经存在的消息**

### 4.4 同请求前置累积的来源

"前置"指的是进入 `BulkActions::action()` 方法之后、执行 `{$handle}($request)` 之前，就已经 flash 过的消息。

以及 handler 内部除了 catch 块之外的其他 flash（warning 级别的业务提示等）。

#### 来源 1：权限检查时的 403 flash（但不会计入）

```php
// L54
flash(trans('errors.message.403'))->error()->important();
return response()->json([...]);  // 直接 return，到不了统计代码
```

这个分支直接 return 了，所以不计入。

#### 来源 2：handler 内部的非失败 flash

handler 方法中除了 `catch { flash(error) }` 之外，也可能 flash 其他消息：

- **warning 级业务提示**：比如"该记录已被其他用户修改，请刷新"
- **info 级提示**：比如"导出已加入队列"（info 不计入 not_passed）
- **success 级消息**：比如"同步成功"（success 不计入 not_passed）

这些 warning 级别的消息不是"操作失败"，但会被 `in_array($message->level, ['danger', 'warning'])` 统计为 `$not_passed`，导致**失败数虚高**。

#### 来源 3：模型事件/Observer 中的 flash

如果模型的 `saving`、`deleting` 等事件中触发了 flash 消息（虽然不规范，但可能存在），会在 foreach 循环中累积。

#### 来源 4：中间件中的 flash（进入控制器前）

admin 中间件组中的某些中间件可能 flash 消息但继续执行（不 redirect）：

| 中间件 | 可能性 | 说明 |
|--------|--------|------|
| `plan.limits` | 可能 | 接近套餐限制时 flash warning 但继续使用 |
| `module.subscription` | 可能 | 模块即将过期时 flash warning |
| `read.only` | 较低 | 只读模式通常会阻止操作，可能 redirect |
| `wizard.redirect` | 低 | 直接 redirect，到不了控制器 |

如果这些中间件 flash 了 `danger` 或 `warning` 级别的消息然后继续执行，这些消息会在 `action()` 方法开始前就存在于 `flash()->messages` 中，被计入 `$not_passed`。

### 4.5 同请求累积 vs 跨请求残留的对比

| 维度 | 同请求前置累积（正确理解） | 跨请求 session 残留（之前的误解） |
|------|--------------------------|--------------------------------|
| 消息来源 | 当前请求中 handler 执行前就 flash 的消息 | 上一次请求存入 Session 的消息 |
| 存储位置 | 内存中的 `$messages` 数组 | Session |
| 生命周期 | 当前请求结束即销毁 | 跨请求，直到被消费 |
| flash()->messages 返回的 | 当前请求内存中的消息 | **不**返回 Session 中的消息 |
| 统计偏差原因 | 中间件/前置代码 flash 的 warning/danger 被误计入 | 上一次请求遗留的消息被误计入 |
| 谁的问题 | 统计逻辑没做"只计本次 handler 内产生的"过滤 | （实际不存在此问题） |

### 4.6 另一个累积问题：handler 执行前后都在 push

看完整的控制器方法：

```php
// 执行前：可能已有前置 flash 消息 N 条（danger/warning）

$result = $bulk_actions->{$handle}($request);  // handler 内 push M 条

flash()->messages->each(...)                   // 统计到 N + M 条
                                                 // 全部计入 $not_passed

$message = trans(...);                          // 构建总结消息
$level = $not_passed > 0 ? 'info' : 'success';

if (...) {
    flash($message)->{$level}();                // 再 push 1 条总结（不计入 not_passed，因为在统计之后）
}
```

关键点：**总结消息在统计之后 push，所以不会影响 `$not_passed` 统计。**

### 4.7 根本问题：没有"本次操作"的消息边界

统计逻辑的核心缺陷是：**没有办法区分哪些 flash 消息是"本次批量操作产生的失败"，哪些是其他原因产生的。**

```
flash()->messages = [
  // 前置累积的（中间件 warning 等）
  '套餐用量已达 80%', level: warning,     ← 不该计入
  '模块 X 将于 3 天后过期', level: warning,  ← 不该计入
  
  // handler 内 catch 的（真正的失败）
  '记录 #101 已对账，不能删除', level: danger,  ← 该计入
  '记录 #102 已对账，不能删除', level: danger,  ← 该计入
  
  // handler 内其他 warning（非失败）
  '注意：3 条记录涉及多个仓库', level: warning,   ← 不该计入（只是提示）
]
```

统统被统计为 `$not_passed = 5`，但实际失败只有 2 条。

---

## 四处问题总结

| # | 问题 | 根因 | 影响 | 严重程度 |
|---|------|------|------|---------|
| 1 | **duplicate 未注册仍可触发** | 基类默认方法 + 控制器无白名单 | 15 个子类全可批量复制，无授权，绕业务校验 | 🔥 高 |
| 2 | **ghost 入口泛滥** | 基类方法全部可调用 + actions 配置不做白名单 | duplicate/destroy/export/download 等多个方法绕过权限配置 | 🔥 高 |
| 3 | **controller 无 actions 白名单** | 直接 `$obj->{$handle}($request)` 无校验 | 所有 public 方法都是 API 入口，权限检查依赖 action 配置的存在性 | 🔥 高 |
| 4 | **flash 同请求前置累积** | `flash()->messages` 是当前请求内存累积，不是跨 session；且没区分"本次操作失败"与"其他消息" | not_passed 统计不准，warning 级消息（中间件、业务提示）也被当失败计数 | ⚠️ 中高 |

---

## 修复建议摘要

| 问题 | 修复方向 |
|------|---------|
| ghost 入口 & 无白名单 | 在控制器 action() 开头加 `in_array($handle, array_keys($bulk_actions->actions))` 白名单校验，不在白名单直接 404/403 |
| duplicate 无权限 | 在基类 `$actions` 中添加 duplicate 配置并指定 permission，或在方法开头做权限检查 |
| destroy 绕过 delete 权限 | 白名单机制天然解决（delete 在白名单，destroy 不在） |
| export/download 缺 permission | 为每个 export/download action 补上 permission 键，如 `read-sales-invoices` |
| flash 统计不准 | 引入独立的失败计数器（如 `$bulk_actions->failCount`），不依赖 flash 消息反推 |

---

## 相关代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| 控制器 action() 入口（无白名单） | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L21-L65 |
| 控制器权限检查（isset 短路） | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L50-L63 |
| 控制器 not_passed 统计 | [BulkActions.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Controllers/Common/BulkActions.php) | L67-L74 |
| 基类 duplicate 方法（ghost） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L152-L159 |
| 基类 enable/disable 方法 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L168-L193 |
| 基类 delete/destroy 方法 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L202-L221 |
| 基类 $actions 配置 | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Abstracts/BulkAction.php) | L25-L51 |
| Invoices actions（无 duplicate） | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | L26-L65 |
| Invoices duplicate 方法（覆盖） | [Invoices.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/BulkActions/Sales/Invoices.php) | L128-L152 |
| FormRequest 验证（仅 required） | [BulkAction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/Http/Requests/Common/BulkAction.php) | L14-L20 |
| 视图层按钮遍历（隐式白名单） | [Bulkaction.php](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/app/View/Components/Index/Bulkaction.php) | L92-L104 |
| laracasts/flash 包版本 | [composer.lock](file:///d:/fz/0601-1/solo-dogfeeding/code/70-akaunting/composer.lock) | L4847-L4901 |
