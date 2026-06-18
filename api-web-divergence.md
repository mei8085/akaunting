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
- **控制器目录**：`app/Http/Controllers/`（不含 `Api` 子目录）
- **响应机制**（三类 JSON 返回模式，详见第 3 章）：
  - **通用包装**（读操作）：`$this->response()` → `Response::toJson()`，返回 `{success, error, data, message}`
  - **直接返回**（写操作）：`ajaxDispatch()` → `response()->json()`，返回 `{success, error, data, message, redirect}`
  - **特殊接口**：`response()->json()` 手动构造，如 Modal、autocomplete、config 等
- **设计目标**：面向后台管理界面的 AJAX 请求、表单提交、页面渲染

---

## 2. 访问器、$appends、Resource 裁剪的关系

### 2.1 五个核心概念

| 概念 | 定义位置 | 作用 | 序列化时自动包含？ |
|------|----------|------|-------------------|
| **数据库字段** | 数据表 + Model `$fillable`/`$casts` | 存储真实数据 | 是 |
| **字段访问器（Field Accessor）** | Model 中 `getXxxAttribute()`，Xxx 是已有数据库字段名 | 重写字段读取逻辑（如 `null` 时取默认值） | 是（自动应用到该字段） |
| **虚拟访问器（Virtual Accessor）** | Model 中 `getXxxAttribute()`，Xxx 不是数据库字段 | 动态计算新属性，通过 `$model->xxx` 调用 | 否（需手动调用或列入 $appends） |
| **序列化附加（$appends）** | Model `$appends` 数组 | 指定哪些**虚拟访问器**在 `toArray()`/`toJson()` 时自动追加 | 是（仅列出的） |
| **隐藏字段（$hidden）** | Model `$hidden` 数组 | 指定哪些字段在序列化时隐藏 | 否（被隐藏） |
| **Resource 裁剪** | `app/Http/Resources/` 中 `toArray()` | API 出口白名单，只返回声明的字段 | 是（仅声明的字段） |

**重要区分**：
- **字段访问器**（如 `getPrecisionAttribute`，`precision` 是数据库字段）：序列化时自动生效，无需 `$appends`
- **虚拟访问器**（如 `getBalanceAttribute`，`balance` 不是数据库字段）：需要列入 `$appends` 才会出现在序列化结果中

### 2.2 字段存在性五层模型

一个字段的"存在"有五个层级，从代码到实际响应逐层收窄：

```
第 1 层：代码中可调用
    = 所有数据库字段 + 所有访问器（字段访问器 + 虚拟访问器）
    │
第 2 层：Model 序列化输出（toArray / toJson）
    = 数据库字段（经字段访问器转换） - $hidden + $appends 中的虚拟访问器
    │
    └─ Web 出口 JSON 模式的 data 字段内容
    │
第 3 层：Resource 白名单（toArray 中显式声明）
    = 在 Resource::toArray() 中显式写出的字段
    │   （可以引用数据库字段、访问器，也可以新增计算字段）
    │
    └─ API 出口的 data 字段内容
    │
第 4 层：外层包装结构
    │
    ├─ API 出口：{ data, links?, meta? } （Laravel Resource 标准）
    └─ Web 出口：{ success, error, data, message, redirect? } （自定义包装）
```

### 2.3 关键规则

**规则 A：虚拟访问器 ≠ 自动序列化**
- 虚拟访问器（`getXxxAttribute` 且 Xxx 不是数据库字段）只是提供了 `$model->xxx` 的调用方式
- 只有列入 `$appends` 的虚拟访问器，才会在 `toArray()`/`toJson()` 时自动出现
- 不在 `$appends` 中的虚拟访问器，只能在 PHP 代码（如 Blade 视图）中手动调用，JSON 序列化时不出现

**规则 B：字段访问器自动生效**
- 针对已有数据库字段的访问器（如 `getPrecisionAttribute`），序列化时自动转换该字段的值
- 不需要列入 `$appends`，也不需要在 Resource 中特殊处理

**规则 C：Resource 完全独立于 $appends 和 $hidden**
- API Resource 的 `toArray()` 是**白名单**机制，跟 Model 的 `$appends` 和 `$hidden` 没有关系
- Resource 可以引用数据库字段，也可以引用任意访问器（不管访问器在不在 `$appends` 里）
- Resource 可以重命名字段（如 `balance` → `current_balance`）
- Resource 可以添加全新的计算字段（如 `*_formatted`）
- Resource 甚至可以返回 `$hidden` 中的字段（只要显式声明）

**规则 D：Web JSON 模式 = Model toArray() + 状态包装**
- Web 出口的 `toJson()` 直接拿 Model 的 `toArray()` 结果
- 包含：数据库字段（经字段访问器转换） + `$appends` 中的虚拟访问器 - `$hidden` 中的字段
- 不包含：不在 `$appends` 中的虚拟访问器

---

## 3. Web 出口三类 JSON 返回模式详解

### 3.1 第一类：通用包装（读操作）

**典型场景**：`index` 列表查询、`show` 单条查询（当请求 `expectsJson()` 时）

**代码路径**：
```
Controller action → $this->response(...) → Response 对象 → toJson()
```

**关键代码**（`app/Abstracts/Http/Response.php` 第 23-31 行）：
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

**数据来源规则**：
- `data` 字段 = `Arr::first($this->data)` = 传递给视图的第一个变量
- 变量通常是 Model 集合或单个 Model，会自动调用 `toArray()`
- 完整序列化：数据库字段（经字段访问器转换） + `$appends` 中的虚拟访问器 - `$hidden`

**响应结构**：
```json
{
  "success": true,
  "error": false,
  "data": [ /* Model 序列化结果 */ ],
  "message": ""
}
```

**典型代码**（`app/Http/Controllers/Common/Items.php` 第 26-31 行）：
```php
public function index()
{
    $items = Item::collect();
    return $this->response('common.items.index', compact('items'));
}
```

---

### 3.2 第二类：直接返回（写操作）

**典型场景**：`store`、`update`、`enable`、`disable`、`destroy`、`import` 等写操作

**代码路径**：
```
Controller action → ajaxDispatch(job) → 手动添加 redirect/message → response()->json($response)
```

**关键代码**（`app/Traits/Jobs.php` 第 55-77 行）：
```php
public function ajaxDispatch($job)
{
    try {
        $data = $this->dispatch($job);  // job 返回值，通常是 Model 对象

        $response = [
            'success' => true,
            'error' => false,
            'data' => $data,  // Model 对象，序列化时自动 toArray()
            'message' => '',
        ];
    } catch (Exception | Throwable $e) {
        $response = [
            'success' => false,
            'error' => true,
            'data' => null,
            'code' => $e->getCode(),
            'message' => $e->getMessage(),
        ];
    }
    return $response;
}
```

**数据来源规则**：
- `data` 字段 = Job 的返回值（通常是创建/更新后的 Model 对象）
- **注意**：Controller 不会删除 `data` 字段，只会追加 `redirect`、`message` 等字段
- 序列化规则与通用包装相同：数据库字段（经字段访问器转换） + `$appends` 中的虚拟访问器 - `$hidden`

**响应结构**（成功时）：
```json
{
  "success": true,
  "error": false,
  "data": { /* Model 序列化结果 */ },
  "message": "操作成功提示",
  "redirect": "/common/items"
}
```

**典型代码**（`app/Http/Controllers/Common/Items.php` 第 61-79 行）：
```php
public function store(Request $request)
{
    $response = $this->ajaxDispatch(new CreateItem($request));

    if ($response['success']) {
        $response['redirect'] = route('items.index');
        $message = trans('messages.success.created', ['type' => trans_choice('general.items', 1)]);
        flash($message)->success();
    } else {
        $response['redirect'] = route('items.create');
        $message = $response['message'];
        flash($message)->error()->important();
    }

    return response()->json($response);  // data 字段完整保留
}
```

---

### 3.3 第三类：特殊接口直接返回

**典型场景**：Modal 弹窗接口、autocomplete 自动补全、config 配置查询等

**代码路径**：
- 直接调用 `response()->json(...)`，手动构造响应
- 不经过 `$this->response()`，也不使用 `ajaxDispatch()`

**子模式 A：Modal 弹窗接口**（返回 HTML 字符串）
- **代码位置**：`app/Http/Controllers/Modals/` 目录下所有控制器
- **数据内容**：`html` 字段为 Blade 渲染后的 HTML 字符串，无数据实体
- **响应结构**：
  ```json
  {
    "success": true,
    "error": false,
    "message": "null",
    "html": "<div>...</div>"
  }
  ```
- **典型代码**（`app/Http/Controllers/Modals/Items.php` 第 30-43 行）：
  ```php
  public function create(IRequest $request)
  {
      $taxes = Tax::enabled()->orderBy('name')->get()->pluck('title', 'id');
      $currency = Currency::where('code', default_currency())->first();
      $html = view('modals.items.create', compact('taxes', 'currency'))->render();
      return response()->json([
          'success' => true,
          'error' => false,
          'message' => 'null',
          'html' => $html,
      ]);
  }
  ```

**子模式 B：autocomplete 自动补全接口**（手动构造数据）
- **代码位置**：如 `app/Http/Controllers/Common/Items.php` 的 `autocomplete()` 方法
- **数据内容**：手动遍历 Model 并添加计算字段（如 `total`）
- **响应结构**：
  ```json
  {
    "success": true,
    "message": "Get all items.",
    "errors": [],
    "data": [ /* 手动构造的数据 */ ]
  }
  ```
- **典型代码**（`app/Http/Controllers/Common/Items.php` 第 238-319 行）：
  ```php
  public function autocomplete()
  {
      // ... 手动计算 total 并设置到 $item->total
      return response()->json([
          'success' => true,
          'message' => 'Get all items.',
          'errors' => [],
          'data' => $items,  // 已手动修改的 Model 集合
      ]);
  }
  ```

**子模式 C：config 等简单对象接口**
- **代码位置**：如 `app/Http/Controllers/Settings/Currencies.php` 的 `config()` 方法
- **数据内容**：简单对象或数组，无固定结构

---

### 3.4 三类模式对比总结

| 模式 | 使用场景 | data 字段来源 | 外层包装 | 典型方法 |
|------|----------|--------------|----------|----------|
| **通用包装** | 读操作 `index`/`show` | 视图第一个变量 → Model `toArray()` | `{success, error, data, message}` | `$this->response()` → `toJson()` |
| **直接返回** | 写操作 `store`/`update`/`enable` 等 | `ajaxDispatch()` 中 job 返回值 → Model `toArray()` | `{success, error, data, message, redirect}` | `ajaxDispatch()` → `response()->json()` |
| **特殊接口** | Modal、autocomplete、config 等 | 手动构造（HTML 字符串 / 修改后的 Model / 简单对象） | 不固定 | 直接 `response()->json()` |

---

## 4. 共享数据的界限（数据分叉点）

### 4.1 共享的上游层
两条出口在以下层面**完全共享**，不产生差异：

| 层次 | 共享内容 | 示例 |
|------|----------|------|
| Model 层 | 数据表结构、属性类型转换、关联关系、所有访问器（字段访问器 + 虚拟访问器） | `Item::with('category', 'taxes')` |
| Job 层 | 创建/更新/删除业务逻辑 | `CreateItem`、`UpdateItem`、`DeleteItem` |
| Request 层 | 输入验证规则 | `App\Http\Requests\Common\Item` |
| 数据查询 | Eloquent 查询构建、eager loading 策略 | `Item::with(...)->collect()` |

### 4.2 明确的分叉点
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
| （Laravel 标准）        |   |   data, message, ... }|
+------------------------+   +------------------------+
```

**关键分叉代码对比**（以 Items 模块 index 为例）：

| 出口 | 代码位置 | 数据返回方式 |
|------|----------|--------------|
| API | `app/Http/Controllers/Api/Common/Items.php` 第 20-25 行 | `Resource::collection($items)` |
| Web | `app/Http/Controllers/Common/Items.php` 第 26-31 行 | `$this->response('common.items.index', compact('items'))` |

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
| `picture` | ✅ | ✅ | ✅ | 数据库字段（带字段访问器重写，null 时取默认图） |
| `enabled` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_from` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_by` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_at` | ✅（ISO8601） | ✅（Carbon 序列化） | ✅ | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅（Carbon 序列化） | ✅ | 格式不同 |
| `item_id` | ❌ | ✅ | ✅ | `$appends` 中，等同 id |
| `tax_ids` | ❌ | ✅ | ✅ | `$appends` 中，税 ID 数组 |
| `initials` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends，仅视图可用 |
| `line_actions` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends，仅视图可用 |
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
| `unpaid` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `open` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `overdue` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |

**重要发现**：
1. Contact 模块 API 出口**裁剪了 5 个 fillable 字段**（`city`、`zip_code`、`state`、`country`、`category_id`）
2. `unpaid`、`open`、`overdue`、`line_actions` 等虚拟访问器**不在 `$appends` 中**，Web JSON 模式也不会返回，仅 Blade 视图可调用

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
| `amount_due` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `recurring_status_label` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `template_path` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |

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
| `income_balance` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `expense_balance` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |

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

### 5.7 分类（Category）模块

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Setting/Category.php` | `app/Models/Setting/Category.php` |

**$appends 列表**：`display_name`、`color_hex_code`

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| 数据库字段（13个） | ✅ | ✅ | ✅ | `id` ~ `created_by` 等 |
| `created_at` / `updated_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `display_name` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `color_hex_code` | ❌ | ✅ | ✅ | `$appends` 中，Web 有；API 未声明 |
| `balance` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `balance_without_subcategories` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |
| `line_actions` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends |

---

### 5.8 币种（Currency）模块（已重新核对）

| 对比维度 | API 出口（Resource） | Web 出口（Model 序列化） |
|----------|---------------------|-------------------------|
| 代码位置 | `app/Http/Resources/Setting/Currency.php` | `app/Models/Setting/Currency.php` |

**$appends 列表**：无

**$fillable 列表**（14 个字段）：
`company_id`、`name`、`code`、`rate`、`enabled`、`precision`、`symbol`、`symbol_first`、`decimal_mark`、`thousands_separator`、`created_from`、`created_by`

**字段访问器**（5 个，针对数据库字段，自动生效）：
- `getPrecisionAttribute`：null 时从货币库取默认精度
- `getSymbolAttribute`：null 时从货币库取默认符号
- `getSymbolFirstAttribute`：null 时从货币库取默认值
- `getDecimalMarkAttribute`：null 时从货币库取默认值
- `getThousandsSeparatorAttribute`：null 时从货币库取默认值

**虚拟访问器**（1 个，不在 $appends）：
- `getLineActionsAttribute`：操作按钮数组，仅视图可用

**字段级差异表**：

| 字段名 | API 出口 | Web JSON | 代码中可调用 | 说明 |
|--------|----------|----------|-------------|------|
| `id` | ✅ | ✅ | ✅ | 数据库字段 |
| `company_id` | ✅ | ✅ | ✅ | 数据库字段 |
| `name` | ✅ | ✅ | ✅ | 数据库字段 |
| `code` | ✅ | ✅ | ✅ | 数据库字段 |
| `rate` | ✅ | ✅ | ✅ | 数据库字段（cast to double） |
| `enabled` | ✅ | ✅ | ✅ | 数据库字段（cast to boolean） |
| `precision` | ✅ | ✅ | ✅ | 数据库字段（含字段访问器，null 时取默认值） |
| `symbol` | ✅ | ✅ | ✅ | 数据库字段（含字段访问器，null 时取默认值） |
| `symbol_first` | ✅ | ✅ | ✅ | 数据库字段（含字段访问器，null 时取默认值） |
| `decimal_mark` | ✅ | ✅ | ✅ | 数据库字段（含字段访问器，null 时取默认值） |
| `thousands_separator` | ✅ | ✅ | ✅ | 数据库字段（含字段访问器，null 时取默认值） |
| `created_from` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_by` | ✅ | ✅ | ✅ | 数据库字段 |
| `created_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `updated_at` | ✅（ISO8601） | ✅ | ✅ | 格式不同 |
| `line_actions` | ❌ | ❌ | ✅ | 虚拟访问器，不在 $appends，仅视图可用 |

**重要修正**：
- Currency 模块**没有** `$appends` 数组
- Currency 模块**没有** `title` 访问器（之前是 Tax 模型的 grep 结果混淆）
- 5 个字段访问器自动生效，Web 和 API 都会得到转换后的值（无差异）
- API Resource 声明了全部 14 个 fillable 字段 + 2 个时间字段，**无字段裁剪**

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

**Web 出口（JSON 模式，通用包装）**：
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

**Web 出口（JSON 模式，通用包装）**：
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

**Web 出口（直接返回模式）**：
```json
{
  "success": true,
  "error": false,
  "data": {
    "id": 1,
    "...": "Model 序列化完整数据（$appends 中的虚拟访问器也包含）"
  },
  "message": "操作成功提示",
  "redirect": "/common/items"
}
```
- **注意**：`data` 字段**保留**，内容是 job 返回的 Model 序列化结果
- 失败时：`{success: false, error: true, data: null, code: xxx, message: "错误信息", redirect: "..."}`

---

## 7. 差异类型总结矩阵

| 差异类型 | 频率 | 典型场景 | 影响范围 |
|----------|------|----------|----------|
| **时间格式不统一** | 极高 | `created_at`/`updated_at`/`paid_at`/`due_at` 等 | 所有模块 |
| **金额格式化字段仅 API 有** | 高 | `*_formatted` 系列字段 | Item、Document、Account、Transaction |
| **字段重命名** | 中 | `balance` → `current_balance` | Account 模块 |
| **$appends 中字段 API 缺失** | 高 | `title`、`status_label`、`paid`、`location`、`display_name` 等 | Item、Contact、Document、Account、Category |
| **虚拟访问器仅代码中可用** | 极高 | `line_actions`、`amount_due`、`unpaid`、`income_balance` 等 | 所有模块 |
| **fillable 字段被 API 裁剪** | 中 | `city`/`zip_code`/`state`/`country`/`category_id` | Contact 模块 |
| **安全字段 $hidden** | 低 | `password`/`remember_token` | User 模块 |
| **关联对象层级裁剪** | 极高 | 所有嵌套关联 | 所有含关联的模块 |
| **分页元数据格式不同** | 高 | API 有 `links`/`meta`，Web 没有 | 列表查询 |
| **外层状态包装不同** | 极高 | 外层 JSON 结构 | 所有响应 |
| **写操作返回结构不同** | 高 | API 返回 `{data}`，Web 返回 `{data, redirect}` | 所有写操作 |
| **字段访问器自动转换** | 中 | `precision`/`symbol` 等 null 时取默认值 | Currency 模块（无差异，两处都自动转换） |

---

## 8. 关键注意事项

### 8.1 前端对接风险
1. **不要混用**：前端若同时对接两条出口，需针对不同出口编写独立的数据适配层
2. **不要假设 `$appends` 在 API 中存在**：API 出口不会自动返回 Model 的 `$appends` 虚拟访问器，除非 Resource 中显式声明
3. **关联数据一致性**：API 返回的关联对象也是经过 Resource 裁剪的，不要期望拿到完整关联数据
4. **不要依赖 `line_actions` 等视图专用字段**：这些是虚拟访问器且不在 `$appends` 中，JSON 响应里没有，仅 Blade 模板可用
5. **Web 写操作有 data 字段**：Web 出口写操作的 JSON 响应中**包含**完整的 Model 数据（含 `$appends`），不要误以为只有 `redirect`

### 8.2 新增字段时的同步维护
新增数据库字段后，需根据情况检查以下位置：

| 场景 | 需要更新的位置 |
|------|--------------|
| 新增数据库字段 | 1. Model `$fillable` 2. Model `$casts`（如需要）3. 对应 Resource `toArray()` |
| 新增字段访问器（针对已有数据库字段） | 1. `getXxxAttribute()` 方法（自动生效，无需其他修改） |
| 新增虚拟访问器（新属性） | 1. `getXxxAttribute()` 方法 2. `$appends` 数组（如需 Web JSON 包含）3. Resource `toArray()`（如需 API 返回） |
| 新增关联 | 1. Model 中关联方法 2. Controller 中 eager loading 3. Resource 中嵌套 Resource |

### 8.3 调试建议

**排查字段缺失时的检查顺序**：

1. **API 出口字段缺失**：
   - 先查对应 Resource 类的 `toArray()` 是否声明了该字段
   - 如声明了但值不对，检查 `$this->xxx` 引用的是数据库字段还是访问器
   - 关联字段缺失，检查 Controller 中是否 eager load 了关联

2. **Web 出口 JSON 字段缺失**：
   - 先查字段是不是数据库字段（在 `$fillable` 里吗？表里有吗？）
   - 是虚拟访问器的话，查是否在 `$appends` 数组中
   - 查是否在 `$hidden` 中被隐藏了

3. **Blade 视图中有但 JSON 中没有**：
   - 大概率是不在 `$appends` 中的虚拟访问器
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
| 币种 Currency | `app/Models/Setting/Currency.php` | `app/Http/Resources/Setting/Currency.php` | `app/Http/Controllers/Api/Settings/Currencies.php` | `app/Http/Controllers/Settings/Currencies.php` |

### 8.5 关键文件速查表

| 功能 | 文件位置 |
|------|----------|
| API 控制器基类 | `app/Abstracts/Http/ApiController.php` |
| Web 控制器基类 | `app/Abstracts/Http/Controller.php` |
| Web 响应包装类 | `app/Abstracts/Http/Response.php` |
| Job dispatch & ajaxDispatch | `app/Traits/Jobs.php` |
| API 路由 | `routes/api.php` |
| 管理后台路由 | `routes/admin.php` |
| 公共路由 | `routes/common.php` |
| Portal 路由 | `routes/portal.php` |
