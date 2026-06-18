# API 与 Web 双出口数据差异分析

## 1. 两条出口的架构定位

### 1.1 出口一：API 出口
- **路由定义**：`routes/api.php`
- **控制器基类**：`app/Abstracts/Http/ApiController.php`
- **控制器目录**：`app/Http/Controllers/Api/`
- **响应机制**：通过 `Resource::collection()` 或 `new Resource($model)` 返回，使用 Laravel API Resource 做字段转换
- **设计目标**：面向外部系统集成、移动端 App、第三方调用，返回结构化、规范化的 JSON

### 1.2 出口二：Web 出口
- **路由定义**：`routes/admin.php`、`routes/common.php`、`routes/portal.php`
- **控制器基类**：`app/Abstracts/Http/Controller.php`
- **响应基类**：`app/Abstracts/Http/Response.php`
- **控制器目录**：`app/Http/Controllers/`（不含 `Api` 子目录）
- **响应机制**：
  - 默认返回 HTML 视图（Blade 模板）
  - 若 `expectsJson()` 为真，则调用 `toJson()`，返回 `{success, error, data, message}` 包装格式，`data` 为 Model 序列化结果
- **设计目标**：面向后台管理界面的 AJAX 请求、表单提交、页面渲染

---

## 2. 访问器、$appends、Resource 裁剪的关系

### 2.1 三个核心概念

| 概念 | 定义位置 | 作用 | 序列化时是否自动包含 |
|------|----------|------|---------------------|
| **数据库字段** | 数据表 + Model `$fillable`/`$casts` | 存储真实数据 | 是 |
| **访问器（Accessor）** | Model 中 `getXxxAttribute()` 方法 | 动态计算属性，通过 `$model->xxx` 调用 | 否（需手动调用） |
| **序列化附加（$appends）** | Model `$appends` 数组 | 指定哪些访问器在 `toArray()`/`toJson()` 时自动追加 | 是（仅 $appends 中列出的） |
| **隐藏字段（$hidden）** | Model `$hidden` 数组 | 指定哪些字段在序列化时隐藏 | 否（被隐藏） |
| **Resource 裁剪** | `app/Http/Resources/` 中 `toArray()` | API 出口白名单，只返回声明的字段 | 是（仅声明的字段） |

### 2.2 字段存在性五层模型

一个字段的"存在"有五个层级，从代码到实际响应逐层收窄：

```
第 1 层：代码中可调用（所有数据库字段 + 所有访问器）
    │
    ├─ 数据库字段（$fillable 中声明，存在于数据表）
    └─ 访问器方法（getXxxAttribute，通过 $model->xxx 调用）
    │
第 2 层：Model 序列化输出（toArray / toJson）
    │   = 数据库字段 - $hidden + $appends 中的访问器
    │
    └─ Web 出口 JSON 模式的 data 字段内容
    │
第 3 层：Resource 白名单（toArray 中显式声明）
    │   = 在 Resource::toArray() 中显式写出的字段
    │
    └─ API 出口的 data 字段内容
    │
第 4 层：外层包装结构
    │
    ├─ API 出口：{ data, links?, meta? } （Laravel Resource 标准）
    └─ Web 出口：{ success, error, data, message } （自定义包装）
```

### 2.3 关键规则

**规则 A：访问器 ≠ 自动序列化**
- 访问器（`getXxxAttribute`）只是提供了 `$model->xxx` 的调用方式
- 只有列入 `$appends` 的访问器，才会在 `toArray()`/`toJson()` 时自动出现
- 不在 `$appends` 中的访问器，只能在 PHP 代码（如 Blade 视图）中手动调用，JSON 序列化时不出现

**规则 B：Resource 完全独立于 $appends**
- API Resource 的 `toArray()` 是**白名单**机制，跟 Model 的 `$appends` 和 `$hidden` 没有关系
- Resource 可以引用数据库字段，也可以引用访问器（不管访问器在不在 `$appends` 里）
- Resource 可以重命名字段（如 `balance` → `current_balance`）
- Resource 可以添加全新的计算字段（如 `*_formatted`）

**规则 C：Web JSON 模式 = Model toArray() + 状态包装**
- Web 出口的 `toJson()` 直接拿 Model 的 `toArray()` 结果
- 包含：所有数据库字段 + `$appends` 中的访问器 - `$hidden` 中的字段
- 不包含：不在 `$appends` 中的访问器（如 `line_actions`、`amount_due`）

### 2.4 典型示例：Account 模块字段层级对照

以 `app/Models/Banking/Account.php` 为例：

| 字段 | 第1层 代码可调用 | 第2层 Web JSON | 第3层 API Resource | 说明 |
|------|----------------|---------------|-------------------|------|
| `id` | ✅ | ✅ | ✅ | 数据库字段 |
| `name` | ✅ | ✅ | ✅ | 数据库字段 |
| `opening_balance` | ✅ | ✅ | ✅ | 数据库字段 |
| `balance`（访问器） | ✅（`$appends` 中） | ✅ | ✅（重命名为 `current_balance`） | 在 $appends 中，Web 有；Resource 引用了它并改名 |
| `title`（访问器） | ✅（`$appends` 中） | ✅ | ❌ | 在 $appends 中，Web 有；Resource 未声明 |
| `initials`（访问器） | ✅（`$appends` 中） | ✅ | ❌ | 在 $appends 中，Web 有；Resource 未声明 |
| `income_balance`（访问器） | ✅（不在 $appends） | ❌ | ❌ | 仅代码中可调用，两处响应都没有 |
| `expense_balance`（访问器） | ✅（不在 $appends） | ❌ | ❌ | 仅代码中可调用，两处响应都没有 |
| `line_actions`（访问器） | ✅（不在 $appends） | ❌ | ❌ | 仅 Blade 视图中可用，JSON 中没有 |
| `opening_balance_formatted` | ❌ | ❌ | ✅ | Resource 中额外计算的字段 |
| `current_balance_formatted` | ❌ | ❌ | ✅ | Resource 中额外计算的字段 |

---

## 3. 共享数据的界限（数据分叉点）

### 3.1 共享的上游层
两条出口在以下层面**完全共享**，不产生差异：

| 层次 | 共享内容 | 示例 |
|------|----------|------|
| Model 层 | 数据表结构、属性类型转换、关联关系、所有访问器 | `Item::with('category', 'taxes')` |
| Job 层 | 创建/更新/删除业务逻辑 | `CreateItem`、`UpdateItem`、`DeleteItem` |
| Request 层 | 输入验证规则 | `App\Http\Requests\Common\Item` |
| 数据查询 | Eloquent 查询构建、eager loading 策略 | `Item::with(...)->collect()` |

### 3.2 明确的分叉点
数据在 **Controller 层的 action 方法返回处** 产生分叉：

```
                  +------------------+
                  |   Model 数据     |
                  | （第1层：全量）   |
                  +--------+---------+
                           |
              +------------+------------+
              |                         |
    API 出口  |                         |  Web 出口
              |                         |
              v                         v
+------------------------+   +------------------------+
| Resource::toArray()    |   | Model->toArray()      |
| （第3层：白名单裁剪）   |   | （第2层：$appends）    |
| （格式化处理）          |   | （无主动裁剪）         |
| （关联 Resource 嵌套）  |   | （关联原始 Model）     |
+------------------------+   +------------------------+
              |                         |
              v                         v
+------------------------+   +------------------------+
| { data, links, meta }  |   | { success, error,     |
| （Laravel 标准）        |   |   data, message }     |
+------------------------+   +------------------------+
```

**关键分叉代码对比**（以 Items 模块 index 为例）：

| 出口 | 代码位置 | 数据返回方式 |
|------|----------|--------------|
| API | `app/Http/Controllers/Api/Common/Items.php` 第 20-25 行 | `Resource::collection($items)` |
| Web | `app/Http/Controllers/Common/Items.php` 第 26-31 行 | `$this->response('common.items.index', compact('items'))` |

---

## 4. 各自的裁剪规则详解

### 4.1 API 出口裁剪规则（Resource 层）

API 出口通过 `app/Http/Resources/` 下的 Resource 类进行统一转换，核心规则：

**规则 A：白名单字段机制**
- 只有在 `toArray()` 方法中**显式声明**的字段才会出现在响应中
- 未声明的字段（包括数据库字段、Model `$appends`、动态属性）一律被丢弃
- 跟 Model 的 `$hidden` 无关（Resource 甚至可以返回 `$hidden` 中的字段，如果显式声明的话）

**规则 B：统一格式化规则**
- **时间字段**：统一转换为 ISO8601 格式 → `$this->created_at->toIso8601String()`
- **金额字段**：除原始数值外，额外增加 `_formatted` 后缀的格式化字段
- **关联嵌套**：关联对象使用对应 Resource 递归转换（而非原始 Model）

**规则 C：显式关联资源包装**
- 集合型关联使用 `[static::$wrap => Resource::collection(...)]` 进行包装
- 单个关联使用 `new Resource($this->relation)`

**规则 D：可引用访问器并重命名**
- Resource 可以直接引用访问器值（如 `'current_balance' => $this->balance`）
- 可以对字段进行重命名，与 Model 层字段名不一致

### 4.2 Web 出口 JSON 模式裁剪规则

Web 出口的 JSON 响应通过 `app/Abstracts/Http/Response.php` 第 23-31 行的 `toJson()` 方法处理：

```php
public function toJson()
{
    return response()->json([
        'success'   => true,
        'error'     => false,
        'data'      => Arr::first($this->data),  // 取 view 数据的第一个变量
        'message'   => '',
    ]);
}
```

**规则 A：Model 序列化输出，无主动裁剪**
- `data` 字段直接使用 `Arr::first($this->data)` 取出传递给视图的第一个变量
- 内容等于 Model 的 `toArray()` 结果：数据库字段 + `$appends` 中的访问器 - `$hidden` 中的字段
- **不经过**任何 Resource 转换

**规则 B：仅 $appends 中的访问器会出现**
- 不在 `$appends` 中的访问器（如 `line_actions`、`amount_due`）不会出现在 JSON 中
- 但可以在 Blade 视图中通过 `$model->line_actions` 手动调用

**规则 C：附加状态包装层**
- 额外包装为 `{success, error, data, message}` 结构
- 对于 `store/update/enable/disable` 等写操作，使用 `ajaxDispatch()` 返回 `{success, redirect, message}`，不含数据实体

---

## 5. 各模块详细字段差异对比

### 5.1 商品（Item）模块

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Common/Item.php` | `app/Models/Common/Item.php` |
| 字段来源 | Resource `toArray()` 白名单 | 数据库字段 + `$appends` |

**$appends 列表**：`item_id`、`tax_ids`

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| `id` | ✅ | ✅ | ✅ | 数据库字段 |
| `company_id` | ✅ | ✅ | ✅ | 数据库字段 |
| `type` | ✅ | ✅ | ✅ | 数据库字段 |
| `name` | ✅ | ✅ | ✅ | 数据库字段 |
| `description` | ✅ | ✅ | ✅ | 数据库字段 |
| `sale_price` | ✅ | ✅ | ✅ | 数据库字段（原始值） |
| `sale_price_formatted` | ✅ | ❌ | ❌ | **API 额外增加**，money 格式化 |
| `purchase_price` | ✅ | ✅ | ✅ | 数据库字段（原始值） |
| `purchase_price_formatted` | ✅ | ❌ | ❌ | **API 额外增加**，money 格式化 |
| `category_id` | ✅ | ✅ | ✅ | 数据库字段 |
| `picture` | ✅ | ✅ | ✅ | 数据库字段（带访问器重写） |
| `enabled` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_from` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_by` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_at` | ✅（ISO8601） | ✅（Carbon 序列化） | ✅ | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅（Carbon 序列化） | ✅ | 格式不同 |
| `item_id` | ❌ | ✅ | ✅ | `$appends` 中，等同 id |
| `tax_ids` | ❌ | ✅ | ✅ | `$appends` 中，税 ID 数组 |
| `initials` | ❌ | ❌ | ✅ | 访问器，不在 $appends，仅视图可用 |
| `line_actions` | ❌ | ❌ | ✅ | 访问器，不在 $appends，仅视图可用 |
| `category` | ✅（Category Resource） | ✅（原始 Model） | ✅ | 关联，嵌套层级不同 |
| `taxes` | ✅（ItemTax Resource 集合） | ✅（原始 Model 集合） | ✅ | 关联，嵌套层级不同 |

---

### 5.2 联系人（Contact）模块

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Common/Contact.php` | `app/Models/Common/Contact.php` |

**$appends 列表**：`location`、`logo`、`initials`、`has_email`

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| `id` | ✅ | ✅ | ✅ | 数据库字段 |
| `company_id` | ✅ | ✅ | ✅ | 数据库字段 |
| `user_id` | ✅ | ✅ | ✅ | 数据库字段 |
| `type` | ✅ | ✅ | ✅ | 数据库字段 |
| `name` | ✅ | ✅ | ✅ | 数据库字段 |
| `email` | ✅ | ✅ | ✅ | 数据库字段 |
| `tax_number` | ✅ | ✅ | ✅ | 数据库字段 |
| `phone` | ✅ | ✅ | ✅ | 数据库字段 |
| `address` | ✅ | ✅ | ✅ | 数据库字段 |
| `website` | ✅ | ✅ | ✅ | 数据库字段 |
| `currency_code` | ✅ | ✅ | ✅ | 数据库字段 |
| `enabled` | ✅ | ✅ | ✅ | 数据库字段 |
| `reference` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_from` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_by` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `contact_persons` | ✅（ContactPerson Resource） | ✅（原始 Model） | ✅ | 关联，嵌套裁剪差异 |
| `city` | ❌ | ✅ | ✅ | **API 裁剪掉**（fillable 字段但 Resource 未声明） |
| `zip_code` | ❌ | ✅ | ✅ | **API 裁剪掉** |
| `state` | ❌ | ✅ | ✅ | **API 裁剪掉** |
| `country` | ❌ | ✅ | ✅ | **API 裁剪掉** |
| `category_id` | ❌ | ✅ | ✅ | **API 裁剪掉** |
| `location` | ❌ | ✅ | ✅ | `$appends` 中，格式化地址 |
| `logo` | ❌ | ✅ | ✅ | `$appends` 中，logo 媒体对象 |
| `initials` | ❌ | ✅ | ✅ | `$appends` 中，名称首字母 |
| `has_email` | ❌ | ✅ | ✅ | `$appends` 中，布尔值 |
| `unpaid` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `open` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `overdue` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 访问器，不在 $appends |

**重要发现**：
1. Contact 模块 API 出口**裁剪了 5 个 fillable 字段**（`city`、`zip_code`、`state`、`country`、`category_id`）
2. `unpaid`、`open`、`overdue`、`line_actions` 等访问器**不在 `$appends` 中**，Web JSON 模式也不会返回，仅 Blade 视图可调用

---

### 5.3 单据（Document）模块（发票/账单）

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Document/Document.php` | `app/Models/Document/Document.php` |

**$appends 列表**：`attachment`、`amount_without_tax`、`discount`、`paid`、`received_at`、`status_label`、`sent_at`、`reconciled`、`contact_location`

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| 数据库字段（26个） | ✅ | ✅ | ✅ | `id` ~ `parent_id`、`created_by` 等 |
| `amount_formatted` | ✅ | ❌ | ❌ | **API 额外**，格式化金额 |
| `issued_at` / `due_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `created_at` / `updated_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `category` | ✅（Category Resource） | ✅（原始 Model） | ✅ | 嵌套裁剪 |
| `currency` | ✅（Currency Resource） | ✅（原始 Model） | ✅ | 嵌套裁剪 |
| `contact` | ✅（Contact Resource） | ✅（原始 Model） | ✅ | 嵌套裁剪（含 Contact 的裁剪规则） |
| `histories` | ✅（DocumentHistory Resource） | ✅ | ✅ | 嵌套裁剪 |
| `items` | ✅（DocumentItem Resource） | ✅ | ✅ | 嵌套裁剪 |
| `item_taxes` | ✅（DocumentItemTax Resource） | ✅ | ✅ | 嵌套裁剪 |
| `totals` | ✅（DocumentTotal Resource） | ✅ | ✅ | 嵌套裁剪 |
| `transactions` | ✅（Transaction Resource） | ✅ | ✅ | 嵌套裁剪 |
| `attachment` | ✅ | ✅ | ✅ | `$appends` 中（两者都有） |
| `paid` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `status_label` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `amount_without_tax` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `discount` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `received_at` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `sent_at` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `reconciled` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `contact_location` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `amount_due` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `recurring_status_label` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `template_path` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 访问器，不在 $appends |

---

### 5.4 银行账户（Account）模块

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Banking/Account.php` | `app/Models/Banking/Account.php` |

**$appends 列表**：`balance`、`title`、`initials`

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| 数据库字段（15个） | ✅ | ✅ | ✅ | `id` ~ `bank_address`、`enabled` 等 |
| `opening_balance_formatted` | ✅ | ❌ | ❌ | **API 额外**，格式化 |
| `current_balance` | ✅ | ❌ | ✅（通过 `balance` 访问器） | **字段重命名**：取自 $appends 的 balance，改名 |
| `current_balance_formatted` | ✅ | ❌ | ❌ | **API 额外**，格式化 |
| `balance` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 改名为 current_balance |
| `title` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `initials` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `created_at` / `updated_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `income_balance` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `expense_balance` | ❌ | ❌ | ✅ | 访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 访问器，不在 $appends |

**重要发现**：Account 模块存在**字段重命名**差异：
- Model 层 `$appends` 中的 `balance` → API Resource 重命名为 `current_balance`

---

### 5.5 交易（Transaction）模块

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Banking/Transaction.php` | - |

**差异要点**：
- `amount` 字段：API 额外增加 `amount_formatted`
- 时间字段：`paid_at`、`created_at`、`updated_at` 统一 ISO8601
- 5 个关联对象全部使用对应 Resource 裁剪：`account`、`category`、`currency`、`contact`、`taxes`
- 无裁剪掉的 fillable 字段

---

### 5.6 用户（User）模块

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Auth/User.php` | `app/Models/Auth/User.php` |

**$hidden 列表**：`password`、`remember_token`

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| `id` | ✅ | ✅ | ✅ | 数据库字段 |
| `name` | ✅ | ✅ | ✅ | 数据库字段 |
| `email` | ✅ | ✅ | ✅ | 数据库字段 |
| `locale` | ✅ | ✅ | ✅ | 数据库字段 |
| `landing_page` | ✅ | ✅ | ✅ | 数据库字段 |
| `enabled` | ✅ | ✅ | ✅ | 数据库字段 |
| `last_logged_in_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `created_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `companies` | ✅（Company Resource） | ✅ | ✅ | 关联 |
| `roles` | ✅（Role Resource） | ✅ | ✅ | 关联 |
| `password` | ❌ | ❌ | ✅ | `$hidden` 中，安全字段 |
| `remember_token` | ❌ | ❌ | ✅ | `$hidden` 中，安全字段 |
| `email_verified_at` | ❌ | ✅ | ✅ | 数据库字段，API 未声明 |

**注意**：User 模块用的是 `$hidden`（黑名单模式）而非 `$appends`（白名单模式）。`$hidden` 只影响 Web 出口的 JSON 序列化，不影响 API Resource（因为 Resource 是白名单模式，根本不会返回未声明的字段）。

---

### 5.7 分类（Category）和币种（Currency）模块

**Category**：
- **$appends**：`display_name`、`color_hex_code`
- API 返回 16 个字段（数据库字段全部有），但未包含 `display_name` 和 `color_hex_code`
- 时间字段 ISO8601 格式化
- `balance`、`balance_without_subcategories`、`line_actions` 等访问器不在 `$appends` 中，Web JSON 也不返回

**Currency**：
- API 返回 17 个字段，包含全部 fillable 字段
- 时间字段 ISO8601 格式化
- `title` 访问器在 `$appends` 中，Web JSON 有；API 未声明

---

## 6. 响应包装结构差异

### 6.1 列表查询（index）响应结构

**API 出口**（Laravel Resource 标准）：
```json
{
  "data": [
    { "...": "Resource 转换后的字段" }
  ],
  "links": {
    "first": "...",
    "last": "...",
    "prev": null,
    "next": "..."
  },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 5,
    "path": "...",
    "per_page": 25,
    "to": 25,
    "total": 109
  }
}
```

**Web 出口（JSON 模式）**：
```json
{
  "success": true,
  "error": false,
  "data": [
    { "...": "Model 序列化结果（数据库字段 + $appends）" }
  ],
  "message": ""
}
```

### 6.2 单条查询（show）响应结构

**API 出口**：
```json
{
  "data": {
    "id": 1,
    "...": "Resource 转换后的单个对象"
  }
}
```

**Web 出口（JSON 模式）**：
```json
{
  "success": true,
  "error": false,
  "data": {
    "id": 1,
    "...": "Model 序列化完整数据"
  },
  "message": ""
}
```

### 6.3 写操作（store/update/enable/disable）响应结构

**API 出口**：
- `store`：201 状态码 + `Location` header + Resource 数据
- `update`：200 + Resource 数据
- `enable/disable`：200 + Resource 数据
- `destroy`：204 No Content（无响应体）

**Web 出口**：
```json
{
  "success": true,
  "redirect": "/common/items",
  "message": "操作成功提示"
}
```
- **不返回**数据实体，只返回状态和跳转 URL
- 失败时：`{success: false, redirect: ..., message: "错误信息"}`

---

## 7. 差异类型总结矩阵

| 差异类型 | 频率 | 典型场景 | 影响范围 |
|----------|------|----------|----------|
| **时间格式不统一** | 极高 | `created_at`/`updated_at`/`paid_at`/`due_at` 等 | 所有模块 |
| **金额格式化字段仅 API 有** | 高 | `*_formatted` 系列字段 | Item、Document、Account、Transaction |
| **字段重命名** | 中 | `balance` → `current_balance` | Account 模块 |
| **$appends 中字段 API 缺失** | 高 | `title`、`status_label`、`paid`、`location` 等 | 几乎所有模块 |
| **访问器仅代码中可用** | 极高 | `line_actions`、`amount_due`、`unpaid` 等 | 所有模块 |
| **fillable 字段被 API 裁剪** | 中 | `city`/`zip_code`/`state`/`country`/`category_id` | Contact 模块 |
| **安全字段 $hidden** | 低 | `password`/`remember_token` | User 模块 |
| **关联对象层级裁剪** | 极高 | 所有嵌套关联 | 所有含关联的模块 |
| **分页元数据格式不同** | 高 | API 有 `links`/`meta`，Web 没有 | 列表查询 |
| **外层状态包装不同** | 极高 | 外层 JSON 结构 | 所有响应 |
| **写操作返回策略** | 高 | API 返回实体，Web 只返回 redirect | 所有写操作 |

---

## 8. 关键注意事项

### 8.1 前端对接风险
1. **不要混用**：前端若同时对接两条出口，需针对不同出口编写独立的数据适配层
2. **不要假设 `$appends` 在 API 中存在**：API 出口不会自动返回 Model 的 `$appends` 访问器，除非 Resource 中显式声明
3. **关联数据一致性**：API 返回的关联对象也是经过 Resource 裁剪的，不要期望拿到完整关联数据
4. **不要依赖 `line_actions` 等视图专用字段**：这些字段是访问器且不在 `$appends` 中，JSON 响应里没有，仅 Blade 模板可用

### 8.2 新增字段时的同步维护
新增数据库字段后，需根据情况检查以下位置：

| 场景 | 需要更新的位置 |
|------|--------------|
| 新增数据库字段 | 1. Model `$fillable` 2. Model `$casts`（如需要）3. 对应 Resource `toArray()` |
| 新增访问器 | 1. `getXxxAttribute()` 方法 2. `$appends` 数组（如需要 JSON 序列化时包含）3. Resource `toArray()`（如需要 API 返回） |
| 新增关联 | 1. Model 中关联方法 2. Controller 中 eager loading 3. Resource 中嵌套 Resource |

### 8.3 调试建议

**排查字段缺失时的检查顺序**：

1. **API 出口字段缺失**：
   - 先查对应 Resource 类的 `toArray()` 是否声明了该字段
   - 如声明了但值不对，检查 `$this->xxx` 引用的是数据库字段还是访问器
   - 关联字段缺失，检查 Controller 中是否 eager load 了关联

2. **Web 出口 JSON 字段缺失**：
   - 先查字段是不是数据库字段（在 `$fillable` 里吗？表里有吗？）
   - 是访问器的话，查是否在 `$appends` 数组中
   - 查是否在 `$hidden` 中被隐藏了

3. **Blade 视图中有但 JSON 中没有**：
   - 大概率是不在 `$appends` 中的访问器
   - 视图中是通过 `$model->xxx` 调用的，JSON 序列化时不会自动包含

### 8.4 代码位置速查表

| 模块 | Model 位置 | Resource 位置 | API Controller | Web Controller |
|------|-----------|--------------|----------------|----------------|
| 商品 Item | `app/Models/Common/Item.php` | `app/Http/Resources/Common/Item.php` | `app/Http/Controllers/Api/Common/Items.php` | `app/Http/Controllers/Common/Items.php` |
| 联系人 Contact | `app/Models/Common/Contact.php` | `app/Http/Resources/Common/Contact.php` | `app/Http/Controllers/Api/Common/Contacts.php` | `app/Http/Controllers/Common/Contacts.php` |
| 单据 Document | `app/Models/Document/Document.php` | `app/Http/Resources/Document/Document.php` | `app/Http/Controllers/Api/Document/Documents.php` | `app/Http/Controllers/Sales/Invoices.php` 等 |
| 账户 Account | `app/Models/Banking/Account.php` | `app/Http/Resources/Banking/Account.php` | `app/Http/Controllers/Api/Banking/Accounts.php` | `app/Http/Controllers/Banking/Accounts.php` |
| 交易 Transaction | `app/Models/Banking/Transaction.php` | `app/Http/Resources/Banking/Transaction.php` | `app/Http/Controllers/Api/Banking/Transactions.php` | `app/Http/Controllers/Banking/Transactions.php` |
| 用户 User | `app/Models/Auth/User.php` | `app/Http/Resources/Auth/User.php` | `app/Http/Controllers/Api/Auth/Users.php` | `app/Http/Controllers/Auth/Users.php` |
| 分类 Category | `app/Models/Setting/Category.php` | `app/Http/Resources/Setting/Category.php` | `app/Http/Controllers/Api/Settings/Categories.php` | `app/Http/Controllers/Settings/Categories.php` |
| 币种 Currency | `app/Models/Setting/Currency.php` | `app/Http/Resources/Setting/Currency.php` | `app/Http/Controllers/Api/Settings/Currencies.php` | - |
