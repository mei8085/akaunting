# API 与 Web 双出口数据差异分析

## 1. 两条出口的架构定位

### 1.1 出口一：API 出口
- **路由定义**：[routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/routes/api.php)
- **控制器基类**：[ApiController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Abstracts/Http/ApiController.php)
- **控制器目录**：[app/Http/Controllers/Api/](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Controllers/Api/)
- **响应机制**：通过 `Resource::collection()` 或 `new Resource($model)` 返回，使用 Laravel API Resource 做字段转换
- **设计目标**：面向外部系统集成、移动端 App、第三方调用，返回结构化、规范化的 JSON

### 1.2 出口二：Web 出口
- **路由定义**：[routes/admin.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/routes/admin.php)、[routes/common.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/routes/common.php)、[routes/portal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/routes/portal.php)
- **控制器基类**：[Controller.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Abstracts/Http/Controller.php)
- **响应基类**：[Response.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Abstracts/Http/Response.php)
- **控制器目录**：[app/Http/Controllers/](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Controllers/)（不含 Api 子目录）
- **响应机制**：
  - 默认返回 HTML 视图（Blade 模板）
  - 若 `expectsJson()` 为真，则调用 `toJson()`，返回 `{success, error, data, message}` 包装格式，`data` 为原始 Model 数据
- **设计目标**：面向后台管理界面的 AJAX 请求、表单提交、页面渲染

---

## 2. 共享数据的界限（数据分叉点）

### 2.1 共享的上游层
两条出口在以下层面**完全共享**，不产生差异：

| 层次 | 共享内容 | 示例 |
|------|----------|------|
| Model 层 | 数据表结构、属性类型转换、关联关系、访问器 | `Item::with('category', 'taxes')` |
| Job 层 | 创建/更新/删除业务逻辑 | `CreateItem`、`UpdateItem`、`DeleteItem` |
| Request 层 | 输入验证规则 | `App\Http\Requests\Common\Item` |
| 数据查询 | Eloquent 查询构建、eager loading 策略 | `Item::with(...)->collect()` |

### 2.2 明确的分叉点
数据在 **Controller 层的 action 方法返回处** 产生分叉：

```
                  +------------------+
                  |   Model 数据     |
                  +--------+---------+
                           |
              +------------+------------+
              |                         |
    API 出口  |                         |  Web 出口
              |                         |
              v                         v
+------------------------+   +------------------------+
| Resource::toArray()    |   | Response::toJson()     |
| （白名单字段裁剪）     |   | Arr::first($data)      |
| （格式化处理）         |   | （原样输出）           |
| （关联 Resource 嵌套） |   | （附加包装层）         |
+------------------------+   +------------------------+
```

**关键分叉代码对比**（以 Items 模块 index 为例）：

| 出口 | 代码位置 | 数据返回方式 |
|------|----------|--------------|
| API | [Api/Common/Items.php#L20-L25](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Controllers/Api/Common/Items.php#L20-L25) | `Resource::collection($items)` |
| Web | [Common/Items.php#L26-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Controllers/Common/Items.php#L26-L31) | `$this->response('common.items.index', compact('items'))` |

---

## 3. 各自的裁剪规则详解

### 3.1 API 出口裁剪规则（Resource 层）

API 出口通过 `app/Http/Resources/` 下的 Resource 类进行统一转换，核心规则：

**规则 A：白名单字段机制**
- 只有在 `toArray()` 方法中**显式声明**的字段才会出现在响应中
- 未声明的字段（包括数据库字段、Model `$appends`、动态属性）一律被丢弃

**规则 B：统一格式化规则**
- **时间字段**：统一转换为 ISO8601 格式 → `$this->created_at->toIso8601String()`
- **金额字段**：除原始数值外，额外增加 `_formatted` 后缀的格式化字段
- **关联嵌套**：关联对象使用对应 Resource 递归转换（而非原始 Model）

**规则 C：显式关联资源包装**
- 集合型关联使用 `[static::$wrap => Resource::collection(...)]` 进行包装
- 单个关联使用 `new Resource($this->relation)`

### 3.2 Web 出口 JSON 模式裁剪规则

Web 出口的 JSON 响应通过 [Response.php#L23-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Abstracts/Http/Response.php#L23-L31) 的 `toJson()` 方法处理：

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

**规则 A：无裁剪的全量输出**
- `data` 字段直接使用 `Arr::first($this->data)` 取出传递给视图的第一个变量
- 包含 Model 所有可序列化属性：数据库字段 + `$appends` 访问器 + eager loaded 关联
- **不经过**任何 Resource 转换

**规则 B：附加状态包装层**
- 额外包装为 `{success, error, data, message}` 结构
- 对于 `store/update/enable/disable` 等写操作，使用 `ajaxDispatch()` 返回 `{success, redirect, message}`，不含数据实体

---

## 4. 各模块详细字段差异对比

### 4.1 商品（Item）模块

| 对比维度 | API 出口（Resource） | Web 出口（原始 Model） |
|----------|---------------------|----------------------|
| 代码位置 | [Resources/Common/Item.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Resources/Common/Item.php) | [Models/Common/Item.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Models/Common/Item.php) |
| 字段数量 | 18 个显式字段 + 2 个嵌套关联 | 18 个 fillable + 4 个 appends + 所有关联 |

**字段级差异表**：

| 字段名 | API 出口 | Web 出口 | 说明 |
|--------|----------|----------|------|
| `id` | ✅ | ✅ | 共享 |
| `company_id` | ✅ | ✅ | 共享 |
| `type` | ✅ | ✅ | 共享 |
| `name` | ✅ | ✅ | 共享 |
| `description` | ✅ | ✅ | 共享 |
| `sale_price` | ✅ | ✅ | 共享（原始值） |
| `sale_price_formatted` | ✅ | ❌ | **API 额外增加**，money 格式化 |
| `purchase_price` | ✅ | ✅ | 共享（原始值） |
| `purchase_price_formatted` | ✅ | ❌ | **API 额外增加**，money 格式化 |
| `category_id` | ✅ | ✅ | 共享 |
| `picture` | ✅ | ✅ | 共享 |
| `enabled` | ✅ | ✅ | 共享 |
| `created_from` | ✅ | ✅ | 共享 |
| `created_by` | ✅ | ✅ | 共享 |
| `created_at` | ✅（ISO8601） | ✅（Carbon 对象） | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅（Carbon 对象） | 格式不同 |
| `item_id` | ❌ | ✅ | **Web 额外**（Model $appends，等同 id） |
| `tax_ids` | ❌ | ✅ | **Web 额外**（Model $appends，税 ID 数组） |
| `initials` | ❌ | ✅ | **Web 额外**（访问器，名称首字母） |
| `line_actions` | ❌ | ✅ | **Web 额外**（访问器，操作按钮数组） |
| `category` | ✅（Category Resource） | ✅（原始 Category Model） | 嵌套层级：API 裁剪，Web 全量 |
| `taxes` | ✅（ItemTax Resource 集合） | ✅（原始 ItemTax Model 集合） | 嵌套层级：API 裁剪，Web 全量 |

---

### 4.2 联系人（Contact）模块

| 对比维度 | API 出口（Resource） | Web 出口（原始 Model） |
|----------|---------------------|----------------------|
| 代码位置 | [Resources/Common/Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Resources/Common/Contact.php) | [Models/Common/Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Models/Common/Contact.php) |

**字段级差异表**：

| 字段名 | API 出口 | Web 出口 | 说明 |
|--------|----------|----------|------|
| `id` | ✅ | ✅ | 共享 |
| `company_id` | ✅ | ✅ | 共享 |
| `user_id` | ✅ | ✅ | 共享 |
| `type` | ✅ | ✅ | 共享 |
| `name` | ✅ | ✅ | 共享 |
| `email` | ✅ | ✅ | 共享 |
| `tax_number` | ✅ | ✅ | 共享 |
| `phone` | ✅ | ✅ | 共享 |
| `address` | ✅ | ✅ | 共享 |
| `website` | ✅ | ✅ | 共享 |
| `currency_code` | ✅ | ✅ | 共享 |
| `enabled` | ✅ | ✅ | 共享 |
| `reference` | ✅ | ✅ | 共享 |
| `created_from` | ✅ | ✅ | 共享 |
| `created_by` | ✅ | ✅ | 共享 |
| `created_at` | ✅（ISO8601） | ✅（Carbon） | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅（Carbon） | 格式不同 |
| `contact_persons` | ✅（ContactPerson Resource） | ✅（原始 Model） | 嵌套裁剪差异 |
| `city` | ❌ | ✅ | **API 裁剪掉**（fillable 字段但未声明） |
| `zip_code` | ❌ | ✅ | **API 裁剪掉** |
| `state` | ❌ | ✅ | **API 裁剪掉** |
| `country` | ❌ | ✅ | **API 裁剪掉** |
| `category_id` | ❌ | ✅ | **API 裁剪掉** |
| `location` | ❌ | ✅ | **Web 额外**（$appends，格式化地址） |
| `logo` | ❌ | ✅ | **Web 额外**（$appends，logo 媒体对象） |
| `initials` | ❌ | ✅ | **Web 额外**（$appends，名称首字母） |
| `has_email` | ❌ | ✅ | **Web 额外**（$appends，布尔值） |
| `unpaid` | ❌ | ✅（按需加载） | **Web 额外**（访问器，未结金额） |
| `open` | ❌ | ✅（按需加载） | **Web 额外**（访问器） |
| `overdue` | ❌ | ✅（按需加载） | **Web 额外**（访问器，逾期金额） |
| `line_actions` | ❌ | ✅ | **Web 额外**（访问器，操作按钮） |

**重要发现**：Contact 模块 API 出口**裁剪了 5 个 fillable 字段**（`city`、`zip_code`、`state`、`country`、`category_id`），这些字段在数据库中存在且可批量赋值，但 API 不返回。

---

### 4.3 单据（Document）模块（发票/账单）

| 对比维度 | API 出口（Resource） | Web 出口（原始 Model） |
|----------|---------------------|----------------------|
| 代码位置 | [Resources/Document/Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Resources/Document/Document.php) | - |

**字段级差异表**：

| 字段名 | API 出口 | Web 出口 | 说明 |
|--------|----------|----------|------|
| `id` ~ `attachment` | ✅ 共 26 个显式字段 | ✅ | 共享数据库字段 |
| `amount_formatted` | ✅ | ❌ | **API 额外**，格式化金额 |
| `created_at` / `updated_at` | ✅（ISO8601） | ✅（Carbon） | 格式不同 |
| `issued_at` / `due_at` | ✅（ISO8601） | ✅（Carbon） | 格式不同 |
| `category` | ✅（Category Resource） | ✅（原始 Model） | 嵌套裁剪 |
| `currency` | ✅（Currency Resource） | ✅（原始 Model） | 嵌套裁剪 |
| `contact` | ✅（Contact Resource） | ✅（原始 Model） | 嵌套裁剪（含 Contact 的裁剪规则） |
| `histories` | ✅（DocumentHistory Resource 集合） | ✅ | 嵌套裁剪 |
| `items` | ✅（DocumentItem Resource 集合） | ✅ | 嵌套裁剪 |
| `item_taxes` | ✅（DocumentItemTax Resource 集合） | ✅ | 嵌套裁剪 |
| `totals` | ✅（DocumentTotal Resource 集合） | ✅ | 嵌套裁剪 |
| `transactions` | ✅（Transaction Resource 集合） | ✅ | 嵌套裁剪 |
| `paid` | ❌ | ✅（访问器） | **Web 额外**，已支付金额 |
| `amount_due` | ❌ | ✅（访问器） | **Web 额外**，应付金额 |
| `status_label` | ❌ | ✅（访问器） | **Web 额外**，状态文本 |
| `logo` | ❌ | ✅（访问器） | **Web 额外** |
| `route` | ❌ | ✅（访问器） | **Web 额外**，路由信息 |

---

### 4.4 银行账户（Account）模块

| 对比维度 | API 出口（Resource） | Web 出口（原始 Model） |
|----------|---------------------|----------------------|
| 代码位置 | [Resources/Banking/Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Resources/Banking/Account.php) | - |

**字段级差异表**：

| 字段名 | API 出口 | Web 出口 | 说明 |
|--------|----------|----------|------|
| `id` ~ `bank_address` | ✅ 共 18 个显式字段 | ✅ | 共享 |
| `opening_balance` | ✅ | ✅ | 共享原始值 |
| `opening_balance_formatted` | ✅ | ❌ | **API 额外**，格式化 |
| `current_balance` | ✅（重命名自 `balance`） | ✅（字段名为 `balance`） | **字段名不一致** |
| `current_balance_formatted` | ✅ | ❌ | **API 额外**，格式化 |
| `created_at` / `updated_at` | ✅（ISO8601） | ✅（Carbon） | 格式不同 |

**重要发现**：Account 模块存在**字段重命名**差异：
- Model 层访问器名为 `balance` → API Resource 重命名为 `current_balance`

---

### 4.5 交易（Transaction）模块

| 对比维度 | API 出口（Resource） | Web 出口（原始 Model） |
|----------|---------------------|----------------------|
| 代码位置 | [Resources/Banking/Transaction.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Resources/Banking/Transaction.php) | - |

**差异要点**：
- `amount` 字段：API 额外增加 `amount_formatted`
- 时间字段：`paid_at`、`created_at`、`updated_at` 统一 ISO8601
- 5 个关联对象全部使用对应 Resource 裁剪：`account`、`category`、`currency`、`contact`、`taxes`
- 无裁剪掉的 fillable 字段

---

### 4.6 用户（User）模块

| 对比维度 | API 出口（Resource） | Web 出口（原始 Model） |
|----------|---------------------|----------------------|
| 代码位置 | [Resources/Auth/User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/32-akaunting/app/Http/Resources/Auth/User.php) | - |

**安全敏感字段的裁剪**：

| 字段名 | API 出口 | Web 出口 | 说明 |
|--------|----------|----------|------|
| `password` | ❌ | ✅（但通常不会暴露） | API 明确不返回，保护安全 |
| `remember_token` | ❌ | ✅ | API 明确不返回 |
| `email_verified_at` | ❌ | ✅ | API 裁剪掉 |
| `locale` | ✅ | ✅ | 共享 |
| `landing_page` | ✅ | ✅ | 共享 |
| `last_logged_in_at` | ✅（ISO8601） | ✅ | 共享，格式不同 |
| `companies` | ✅（Company Resource 集合） | ✅ | 嵌套裁剪 |
| `roles` | ✅（Role Resource 集合） | ✅ | 嵌套裁剪 |

---

### 4.7 分类（Category）和币种（Currency）模块

**Category**：
- API 返回 16 个字段，无明显裁剪 fillable 字段
- 时间字段 ISO8601 格式化
- 无额外附加字段

**Currency**：
- API 返回 17 个字段，包含全部 fillable 字段
- 时间字段 ISO8601 格式化

---

## 5. 响应包装结构差异

### 5.1 列表查询（index）响应结构

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
    { "...": "原始 Model 所有字段 + appends + 关联" }
  ],
  "message": ""
}
```

### 5.2 单条查询（show）响应结构

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
    "...": "原始 Model 完整数据"
  },
  "message": ""
}
```

### 5.3 写操作（store/update/enable/disable）响应结构

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

## 6. 差异类型总结矩阵

| 差异类型 | 频率 | 典型场景 | 影响范围 |
|----------|------|----------|----------|
| **时间格式不统一** | 极高 | `created_at`/`updated_at`/`paid_at`/`due_at` 等 | 所有模块 |
| **金额格式化字段缺失** | 高 | `*_formatted` 系列字段 | Item、Document、Account、Transaction |
| **字段重命名** | 中 | `balance` → `current_balance` | Account 模块 |
| **安全字段裁剪** | 中 | `password`/`remember_token` | User 模块 |
| **Fillable 字段被裁剪** | 中 | `city`/`zip_code`/`state`/`country`/`category_id` | Contact 模块 |
| **Model $appends 不返回** | 极高 | `item_id`/`tax_ids`/`location`/`logo`/`line_actions` 等 | 所有模块 |
| **关联对象层级裁剪** | 极高 | 所有嵌套关联 | 所有含关联的模块 |
| **分页元数据缺失** | 高 | `links`/`meta` 分页信息 | 列表查询 |
| **状态包装结构不同** | 极高 | 外层 JSON 结构 | 所有响应 |
| **写操作返回策略** | 高 | 是否返回实体数据 | 所有写操作 |

---

## 7. 关键注意事项

### 7.1 前端对接风险
1. **不要混用**：前端若同时对接两条出口，需针对不同出口编写独立的数据适配层
2. **不要依赖 `$appends`**：API 出口不会返回 Model 的 `$appends` 访问器（如 `line_actions`、`location`）
3. **关联数据一致性**：API 返回的关联对象也是经过 Resource 裁剪的，不要期望拿到完整关联数据

### 7.2 新增字段时的同步维护
新增数据库字段后，需**同时更新**两处：
1. Model 的 `$fillable` 属性（已有的话）
2. 对应 Resource 的 `toArray()` 方法（否则 API 不返回新字段）

### 7.3 调试建议
排查字段缺失时的检查顺序：
1. API 出口：先查对应 Resource 类的 `toArray()` 是否声明了该字段
2. Web 出口：先查 Model 的 `$fillable` / `$appends` / 访问器是否定义
