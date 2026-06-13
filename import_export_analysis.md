# 导入与导出的职责分层与解析逻辑复用分析

## 一、整体架构概览

本项目基于 **Maatwebsite/Laravel-Excel** 库构建导入导出功能，采用了清晰的领域分层设计。整体分为以下五层：

| 层级 | 目录/文件 | 职责 |
|------|-----------|------|
| **入口层 (Utilities)** | `app/Utilities/Import.php`, `app/Utilities/Export.php` | 对外统一入口，处理同步/异步队列调度、异常捕获、消息通知 |
| **抽象基类层 (Abstracts)** | `app/Abstracts/Import.php`, `app/Abstracts/Export.php`, `app/Abstracts/ImportMultipleSheets.php` | 定义导入导出的通用模板方法、数据映射、验证规则、事件钩子 |
| **辅助 Trait 层** | `app/Traits/Import.php` | 封装跨实体复用的关联解析逻辑（如 ID 反查、实体自动创建） |
| **具体实现层 (Imports/Exports)** | `app/Imports/*`, `app/Exports/*` | 各业务实体的字段声明、自定义映射、数据查询 |
| **Sheet 子层** | `*/Sheets/*.php` | 多 Sheet Excel 的单 Sheet 处理器，对应一张数据表 |
| **统计分类层 (Classifiers)** | `app/Classifiers/Import.php`, `app/Classifiers/Export.php` | 用于代码统计工具识别导入导出类 |

---

## 二、按代码顺序逐层拆解分析

### 2.1 入口层：Utilities —— 统一门面与流程编排

#### [Import.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Utilities/Import.php)

**核心方法：`fromExcel($class, Request $request, string $translation): array`**

```php
public static function fromExcel($class, Request $request, string $translation): array
{
    try {
        $should_queue = should_queue();
        $file = $request->file('import');

        if ($should_queue) {
            self::importQueue($class, $file, $translation);  // 异步队列
        } else {
            $class->import($file);                            // 同步执行
        }
        // ... 返回成功消息
    } catch (Throwable $e) {
        $message = self::flashFailures($e);                  // 异常捕获与用户友好提示
        $success = false;
    }
    return ['success' => $success, 'error' => !$success, 'data' => null, 'message' => $message];
}
```

**职责划分：**
- **不关心具体数据解析**：接收一个 `AbstractsImport` 或 `ImportMultipleSheets` 实例，将文件交给它处理
- **流程控制**：判断是否启用队列（`should_queue()`），同步直接导入，异步则走 `importQueue`
- **异常处理**：统一捕获 `ValidationException` 等异常，通过 `flashFailures()` 转换为用户友好的行级错误提示
- **通知机制**：队列模式下链式追加 `NotifyUser` Job，导入完成后通知用户

#### [Export.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Utilities/Export.php)

**核心方法：`toExcel($class, $translation, $extension = 'xlsx')`**

```php
public static function toExcel($class, $translation, $extension = 'xlsx')
{
    try {
        $file_name = Str::filename($translation) . '-' . time() . '.' . $extension;

        if (should_queue()) {
            // 异步：存到临时磁盘，完成后通过 CreateMediableForExport Job 生成可下载媒体
            $class->queue($file_name, $disk)->onQueue('exports')->chain([
                new CreateMediableForExport(user(), $file_name, $translation),
            ]);
            flash($message)->success();
            return back();
        } else {
            return $class->download($file_name);  // 同步：直接下载响应
        }
    } catch (Throwable $e) {
        report($e);
        flash($e->getMessage())->error()->important();
        return back();
    }
}
```

**职责划分：**
- 与 Import 入口对称：同步直接下载，异步队列化处理
- **导出特有的产物管理**：异步模式下通过 `CreateMediableForExport` 将生成的文件注册为媒体资源，供用户下载
- 文件名生成、异常兜底统一处理

---

### 2.2 抽象基类层：Abstracts —— 模板方法与通用逻辑

这是整个架构的**核心骨架**，实现了导入导出的通用处理流程，具体类只需要覆写少量方法。

#### [Import.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Abstracts/Import.php)

**类签名（接口契约）：**
```php
abstract class Import implements 
    HasLocalePreference,        // 支持用户本地化
    ShouldQueue,                // 支持队列
    SkipsEmptyRows,             // 跳过空行
    WithChunkReading,           // 分块读取（大文件）
    WithHeadingRow,             // 第一行为表头
    WithLimit,                  // 行数限制
    WithMapping,                // 行数据映射
    WithValidation,             // 数据验证
    ToModel                     // 每行转为 Model
{
    use Importable, ImportHelper, Sources;
```

**核心方法按执行顺序：**

| 方法 | 职责 | 可覆写 |
|------|------|--------|
| `map($row): array` | **行级数据预处理**：注入 `company_id`、`created_by`、`created_from`；布尔/整数字段转换；Excel 日期序列转标准日期；触发 `RowPreparing` 事件 | ✅ 子类先 `parent::map($row)` 再追加逻辑 |
| `withValidator($validator)` | **行级验证**：若设置了 `$request_class`，复用对应的 `FormRequest` 规则，对每一行独立验证；支持 `prepareRules()` 扩展 | ❌ 模板方法 |
| `rules(): array` | 额外验证规则（默认空，优先用 `$request_class`） | ✅ |
| `prepareRules(array $rules): array` | 对 FormRequest 规则做二次加工 | ✅ |
| `model(array $row)` | **每行最终落地**：由 Maatwebsite 调用，子类必须实现返回 Model 实例或 null | ✅ 必须实现 |
| `hasRow($row)` | 基于 `$model` + `$columns` 做内存级去重（避免每行重复查库） | ❌ 模板方法 |
| `chunkSize()`, `limit()` | 从 `config('excel.imports.*')` 读取配置 | ❌ |

**设计要点：解析逻辑复用的基础**
- 通过 `$request_class` 属性指向对应的 `FormRequest`，**直接复用业务层的验证规则**，无需在 Import 类中重复写 `rules()`
- `map()` 是父类先做通用处理，子类追加实体特定的字段解析（如分类 ID 反查）

#### [Export.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Abstracts/Export.php)

**类签名：**
```php
abstract class Export implements 
    FromCollection,             // 从集合导出
    HasLocalePreference,
    ShouldAutoSize,             // 列宽自适应
    ShouldQueue,
    WithHeadings,               // 表头
    WithMapping,                // 行映射
    WithTitle,                  // Sheet 标题
    WithStrictNullComparison,
    WithEvents                  // Sheet 事件（用于下拉验证等）
{
    use Exportable;
```

**核心方法按执行顺序：**

| 方法 | 职责 | 可覆写 |
|------|------|--------|
| `fields(): array` | **声明导出的字段列表**，同时用作表头和映射依据 | ✅ 必须实现 |
| `collection()` | 查询要导出的数据集合 | ✅ 必须实现 |
| `headings(): array` | 调用 `fields()` 返回表头，触发 `HeadingsPreparing` 事件 | ❌ |
| `map($model): array` | **模型转数组行**：遍历 `fields()`，处理 `created_by`（ID→邮箱）、日期字段转 Excel 序列、**CSV 注入防护** | ✅ 子类可先追加属性再调用 `parent::map()` |
| `afterSheet($event)` | 导出后处理：给列加下拉验证（`column_validations`）或基于 FormRequest 规则生成验证提示 | ❌ 模板方法 |
| `prepareRows($rows)` | 导出前对行集合做二次加工，触发 `RowsPreparing` 事件 | ✅ |
| `failed()` | 队列失败时通知用户 | ❌ |

#### [ImportMultipleSheets.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Abstracts/ImportMultipleSheets.php)

多 Sheet 导入的聚合器，本身不处理数据，只负责：
- 声明 `WithMultipleSheets` 接口
- 由子类实现 `sheets(): array` 返回多个单 Sheet Import 实例
- 统一配置分块大小、用户上下文、跳过未知 Sheet

---

### 2.3 辅助 Trait 层：Import Trait —— 跨实体复用的关联解析

这是**解析逻辑复用的核心层**。导入时 Excel 里存的是关联对象的名称/编码/邮箱等人类可读值，需要反查为数据库 ID，甚至在关联对象不存在时自动创建。[Traits/Import.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Traits/Import.php) 封装了所有这些逻辑。

**核心解析方法（统一模式：先查→不存在则创建）：**

| 方法 | 解析策略（按优先级） |
|------|---------------------|
| `getAccountId($row)` | `account_id` → `account_name` → `account_number` → `currency_code`（不存在则自动创建 Account） |
| `getCategoryId($row, $type)` | `category_id` → `category_name`（自动创建 Category） |
| `getContactId($row, $type)` | `contact_id` → `contact_email` → `contact_name`（自动创建 Contact） |
| `getCurrencyCode($row)` | 查 Currency 表，不存在则自动创建（支持默认值兜底） |
| `getItemId($row, $type)` | `item_id` → `item_name`（自动创建 Item） |
| `getTaxId($row)` | `tax_id` → `tax_name` → `tax_rate`（自动创建 Tax） |
| `getDocumentId($row)` | `document_id` → `document_number` → `invoice_number`/`bill_number` → `invoice_bill_number` |
| `getParentId($row)` | `parent_id` → `parent_number`（关联单据/交易/分类） |
| `getCreatedById($row)` | `created_by`（邮箱→用户 ID） |
| `getPaymentMethod($row)` | 查已注册支付方式，不存在则尝试创建离线支付方式 |

**每个方法的标准实现模式**（以 `getCategoryIdFromName` 为例）：

```php
public function getCategoryIdFromName($row, $type)
{
    // 1. 先查：已有则直接返回 ID
    $category_id = Category::type($type)->withSubCategory()->where('name', $row['category_name'])->pluck('id')->first();
    if (!empty($category_id)) {
        return $category_id;
    }

    // 2. 没有则组装数据并使用对应的 FormRequest 验证
    $data = [
        'company_id' => company_id(),
        'name' => $row['category_name'],
        'type' => $type,
        // ... 其他字段
    ];
    Validator::validate($data, (new CategoryRequest)->rules());

    // 3. 通过 Job 派发生成（走业务层的创建流程，触发事件/观察者等）
    $category = $this->dispatch(new CreateCategory($data));
    return $category->id;
}
```

**设计亮点：**
- **验证复用**：自动创建关联对象时，直接复用对应实体的 `FormRequest` 规则，保证数据一致性
- **业务流程复用**：通过 `dispatch(new CreateXxx($data))` 走 Job 而非直接 `Model::create()`，确保观察者、事件、审计日志等都被触发
- **优雅降级**：每个解析方法都支持从多种字段反查，用户 Excel 中只要填了任意一个标识字段就能正确解析

此外，该 Trait 还提供了视图层辅助方法：`getImportView()` 用于构建导入页面所需的路径、示例文件链接、表单参数等。

---

### 2.4 具体实现层：Imports/Exports —— 业务实体定制

#### 单 Sheet vs 多 Sheet 结构

**多 Sheet 入口**（如 [Imports/Common/Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Imports/Common/Items.php)）：
```php
class Items extends ImportMultipleSheets
{
    public function sheets(): array
    {
        return [
            'items'      => new Sheets\Items(),       // 主表 Sheet
            'item_taxes' => new Sheets\ItemTaxes(),   // 关联表 Sheet
        ];
    }
}
```

**多 Sheet 导出**（如 [Exports/Common/Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Exports/Common/Items.php)）：
```php
class Items implements WithMultipleSheets
{
    use Exportable;

    public $ids;

    public function __construct($ids = null) { $this->ids = $ids; }

    public function sheets(): array
    {
        return [
            new Sheets\Items($this->ids),
            new Sheets\ItemTaxes($this->ids),
        ];
    }
}
```

#### 单 Sheet 具体实现示例

**导入 Sheet：[Imports/Common/Sheets/Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Imports/Common/Sheets/Items.php)**
```php
class Items extends Import
{
    public $request_class = Request::class;   // ← 复用 Item FormRequest 验证规则
    public $model = Model::class;             // ← 关联的 Eloquent Model
    public $columns = ['type', 'name', 'sale_price', 'purchase_price']; // ← 去重比对字段

    public function model(array $row)
    {
        if (self::hasRow($row)) { return; }   // 内存去重
        return new Model($row);               // 批量赋值（依赖 $fillable）
    }

    public function map($row): array
    {
        $row = parent::map($row);                              // 父类通用处理
        $row['sale_information'] = isset($row['sale_price']);  // 业务字段加工
        $row['purchase_information'] = isset($row['purchase_price']);
        $row['category_id'] = $this->getCategoryId($row, 'item'); // ← 调用 Trait 解析关联
        return $row;
    }
}
```

**导出 Sheet：[Exports/Common/Sheets/Items.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Exports/Common/Sheets/Items.php)**
```php
class Items extends Export
{
    public $request_class = Request::class;  // ← 用于生成 Excel 列验证提示

    public function collection()
    {
        return Model::with('category')->collectForExport($this->ids);  // 查询数据
    }

    public function map($model): array
    {
        $model->category_name = $model->category->name;  // 关联名称展开（供导入时回读）
        return parent::map($model);                      // 父类按 fields() 映射
    }

    public function fields(): array
    {
        return ['name', 'type', 'description', 'sale_price', 'purchase_price', 'category_name', 'enabled'];
    }
}
```

#### 另一个例子：[Imports/Common/Sheets/ItemTaxes.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Imports/Common/Sheets/ItemTaxes.php)

```php
class ItemTaxes extends Import
{
    public $request_class = Request::class;
    public $model = Model::class;
    public $columns = ['item_id', 'tax_id'];

    public function model(array $row)
    {
        if (self::hasRow($row)) { return; }
        return new Model($row);
    }

    public function map($row): array
    {
        if ($this->isEmpty($row, 'item_name')) { return []; }  // 空行跳过
        $row = parent::map($row);
        $row['item_id'] = $this->getItemId($row);   // ← 名称→ID 解析（复用 Trait）
        $row['tax_id'] = $this->getTaxId($row);     // ← 名称/税率→ID 解析
        return $row;
    }
}
```

**具体实现层的职责总结：**
- **声明式配置**：通过 `$request_class`、`$model`、`$columns`、`fields()` 等属性/方法声明元数据
- **最小化定制**：只覆写 `map()` 追加业务字段加工，只实现 `model()` / `collection()` 落地方法
- **依赖注入解析**：通过 `$this->getXxxId()` 调用 Trait，无需重复写关联反查逻辑

---

### 2.5 Classifiers 层：代码统计分类

[Classifiers/Import.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Classifiers/Import.php) 和 [Classifiers/Export.php](file:///d:/fz/0601-1/solo-dogfeeding/code/54-akaunting/app/Classifiers/Export.php) 是 `wnx/laravel-stats` 包的扩展，用于将 Import/Export 类单独统计为 "Imports" / "Exports" 应用代码分类，不参与业务逻辑。

---

## 三、导入导出职责对比总结

| 维度 | Import（导入） | Export（导出） |
|------|----------------|----------------|
| **数据方向** | Excel → 数组 → Model → DB | DB → Model 集合 → 数组 → Excel |
| **核心接口** | `ToModel`, `WithMapping`, `WithValidation` | `FromCollection`, `WithMapping`, `WithHeadings` |
| **关键处理** | 字段解析（名称→ID）、验证、去重、自动创建关联 | 字段展开（ID→名称）、日期格式化、CSV 注入防护 |
| **FormRequest 用途** | 复用为行数据验证规则 | 复用为 Excel 列输入提示（数据验证警告） |
| **Trait 依赖** | 重度依赖 `App\Traits\Import` 做关联解析 | 无独立 Trait，全部逻辑在抽象类中 |
| **异步产物** | 队列完成后发送 `ImportCompleted` 通知 | 队列完成后生成 `Media` 供用户下载 |
| **fields() 作用** | 无此字段（由 Excel 表头驱动） | 同时作为表头声明和映射遍历依据 |

---

## 四、同一套解析逻辑的复用机制

### 4.1 复用维度一：FormRequest 验证规则

Import 和 Export **都通过 `$request_class` 属性指向同一个 FormRequest 类**：

```
App\Http\Requests\Common\Item (同一个)
        ↙            ↘
Import\Common\Sheets\Items    Export\Common\Sheets\Items
  (用于行数据 validate)         (用于 Excel 列验证提示)
```

- **Import 侧**：`withValidator()` 中反射实例化 `$request_class`，调用其 `rules()` 对每行数据做验证
- **Export 侧**：`afterSheet()` → `validationWarning()` 中解析同一份 `rules()`，将 `required` / `email` / `integer` / `date_format` 等规则翻译为 Excel 单元格的输入提示

### 4.2 复用维度二：字段对称性（Import ↔ Export）

导出时**故意展开关联名称**（如 `category_name`），导入时再通过 Trait 的 `getCategoryId()` 从名称反查 ID，形成闭环：

```
Export: $model->category_name = $model->category->name;   // ID → 名称
          ↓ (写入 Excel)
Import: $row['category_id'] = $this->getCategoryId($row); // 名称 → ID
```

这种设计使得 **导出的 Excel 文件可以直接再次导入**，实现了导出-导入的往返兼容。

### 4.3 复用维度三：Trait 跨 Import 类共享关联解析

所有 Import Sheet 类通过 `use Importable, ImportHelper, Sources;`（在抽象类中）获得 Trait 的全部解析能力：
- `getAccountId()` / `getCategoryId()` / `getContactId()` / `getCurrencyCode()` / `getItemId()` / `getTaxId()` / `getDocumentId()` / `getParentId()` / `getCreatedById()` / `getPaymentMethod()`
- 这些方法在 Items、Taxes、Invoices、Bills、Transactions 等所有实体的 Import 类中被反复调用，无需重复实现

### 4.4 复用维度四：模板方法模式

抽象类中实现 `map()`、`withValidator()`、`afterSheet()` 等模板方法，具体类只需：
1. 调用 `parent::map($row)` 复用通用处理
2. 追加少量业务特定逻辑

### 4.5 复用维度五：日期字段统一处理

Import 和 Export 抽象类中都硬编码了同一组日期字段：
```php
$date_fields = ['paid_at', 'invoiced_at', 'billed_at', 'due_at', 'issued_at', 'transferred_at'];
```
- **Import**：Excel 日期序列 → `Y-m-d H:i:s`；或按用户本地语言解析字符串
- **Export**：`Y-m-d` → Excel 日期序列

保证了日期字段在导出/导入往返中格式一致。

---

## 五、完整调用链路

### 导入链路
```
控制器调用 App\Utilities\Import::fromExcel(new ItemsImport(), $request, 'items')
  ↓
判断 should_queue()
  ├─ 同步：$class->import($file)
  └─ 异步：$class->queue($file)->chain([NotifyUser])
      ↓
Maatwebsite 逐行处理（WithHeadingRow → WithMapping → WithValidation → ToModel）
      ↓
Abstracts\Import::map()  →  通用字段注入 + RowPreparing 事件
      ↓
具体 Sheet::map()        →  业务字段加工 + Trait::getXxxId() 关联解析
      ↓
Abstracts\Import::withValidator()  →  基于 $request_class 的行级验证
      ↓
具体 Sheet::model()      →  hasRow() 去重 → new Model($row) 入库
```

### 导出链路
```
控制器调用 App\Utilities\Export::toExcel(new ItemsExport($ids), 'items')
  ↓
判断 should_queue()
  ├─ 同步：$class->download($filename)  →  直接响应下载
  └─ 异步：$class->queue(...)->chain([CreateMediableForExport])
      ↓
Maatwebsite 处理（FromCollection → WithHeadings → WithMapping → WithEvents）
      ↓
具体 Sheet::collection()  →  查询数据（含关联预加载）
      ↓
Abstracts\Export::headings()  →  fields() + HeadingsPreparing 事件
      ↓
具体 Sheet::map()         →  关联名称展开（如 category_name）
      ↓
Abstracts\Export::map()   →  按 fields() 遍历 + 日期转换 + CSV 注入防护
      ↓
Abstracts\Export::afterSheet()  →  基于 column_validations 或 $request_class 设置 Excel 列验证
```

---

## 六、设计模式总结

| 设计模式 | 应用位置 |
|----------|----------|
| **模板方法模式** | `Abstracts\Import::map()`、`Abstracts\Export::map()` / `afterSheet()` 定义流程骨架，子类覆写特定步骤 |
| **门面模式** | `Utilities\Import::fromExcel()`、`Utilities\Export::toExcel()` 对外提供简化的统一入口 |
| **Trait 横向复用** | `App\Traits\Import` 提供跨实体的关联解析能力，被所有 Import 类通过 `use ImportHelper` 引入 |
| **策略模式** | `$request_class` 属性可灵活替换为不同 FormRequest，切换验证策略 |
| **事件钩子模式** | `RowPreparing`、`HeadingsPreparing`、`RowsPreparing`、`ImportViewCreating`、`ImportViewCreated` 等事件供模块扩展 |
| **多态** | `Abstracts\Import` / `ImportMultipleSheets` 可被 `Utilities\Import::fromExcel()` 无差别处理 |

---

## 七、接口层在父子 Sheet 导出中的作用

### 7.1 问题背景：为什么需要多 Sheet？

对于复杂实体（如 Bills、Invoices），一条主记录对应多张子表：
- **Bills** → BillItems → BillItemTaxes → BillTotals → BillHistories → BillTransactions
- **Invoices** → InvoiceItems → InvoiceItemTaxes → InvoiceTotals → InvoiceHistories → InvoiceTransactions

如果将这些扁平化为一张 Sheet，会导致大量冗余字段和数据不规整。多 Sheet 设计让每个子表独立一个 Sheet，通过外键字段关联。

### 7.2 接口层角色拆解

#### 导出侧：`WithMultipleSheets` 接口

Maatwebsite 提供的 `WithMultipleSheets` 接口要求实现 `sheets(): array` 方法。项目中的父级导出类（如 [Bills.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Exports/Purchases/Bills/Bills.php)）直接实现此接口：

```php
class Bills implements WithMultipleSheets
{
    use Exportable;

    public $ids;

    public function __construct($ids = null)
    {
        $this->ids = $ids;
    }

    public function sheets(): array
    {
        return [
            new Base($this->ids),           // 主表 Sheet：bills
            new BillItems($this->ids),      // 子表 Sheet：bill_items
            new BillItemTaxes($this->ids),  // 子表 Sheet：bill_item_taxes
            new BillHistories($this->ids),  // 子表 Sheet：bill_histories
            new BillTotals($this->ids),     // 子表 Sheet：bill_totals
            new BillTransactions($this->ids), // 子表 Sheet：bill_transactions
        ];
    }
}
```

**关键设计：`$ids` 的透传机制。** 父级类接收 `$ids` 参数，然后将其传给每一个子 Sheet。`$ids` 来自两个场景：
1. **用户手动勾选导出**：BulkAction 传入选中行的 ID 数组
2. **全量导出（无 ids）**：Utilities/Export 中会自动补全（见下文 7.3）

每个子 Sheet 都继承 `Abstracts\Export`，独立实现 `collection()` 和 `fields()`，拥有自己的查询逻辑和字段映射。

#### 导入侧：`ImportMultipleSheets` + `WithMultipleSheets` + `SkipsUnknownSheets`

[ImportMultipleSheets.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/ImportMultipleSheets.php) 是导入侧的聚合基类：

```php
abstract class ImportMultipleSheets implements ShouldQueue, WithChunkReading, WithMultipleSheets, SkipsUnknownSheets
{
    use Importable;

    public $user;

    public function __construct()
    {
        $this->user = user();
    }

    public function chunkSize(): int
    {
        return config('excel.imports.chunk_size');
    }
    
    public function onUnknownSheet($sheetName)
    {
        // 静默跳过
    }
}
```

**与导出侧的差异：**

| 维度 | 导出侧 (WithMultipleSheets) | 导入侧 (ImportMultipleSheets) |
|------|---------------------------|------------------------------|
| 实现方式 | 直接 `implements WithMultipleSheets` | 继承 `ImportMultipleSheets` 抽象类 |
| Sheet 返回格式 | `new SheetClass($ids)` 对象数组 | `'sheet_name' => new SheetClass()` 关联数组（key 为 Sheet 名） |
| 额外接口 | 无 | `WithChunkReading`（分块读取）、`SkipsUnknownSheets`（跳过未知 Sheet） |
| 是否 ShouldQueue | 不在父级（各 Sheet 自行决定） | 父级声明 `ShouldQueue`，整个导入走队列 |

`SkipsUnknownSheets` 的 `onUnknownSheet()` 方法默认为空实现，意味着当导入的 Excel 中包含多余的 Sheet 时**静默忽略**而非报错——这保证了向后兼容：旧版导出文件可能没有新增的 Sheet。

### 7.3 `WithParentSheet` 接口：突破分页限制的标记接口

[WithParentSheet.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Interfaces/Export/WithParentSheet.php) 是一个**纯标记接口**（Marker Interface），内部没有任何方法：

```php
interface WithParentSheet
{
    //
}
```

它的唯一作用是在 `scopeCollectForExport()` 中作为判断依据，用于**突破分页限制**。具体见下一节。

---

## 八、如何突破分页限制

### 8.1 问题的根源

列表页面通常有分页（如每页 25 条），导出时如果直接复用 `scopeCollect()`，就只能导出当前页的数据。项目通过 `scopeCollectForExport()` 方法解决了这个问题。

### 8.2 `scopeCollect()` vs `scopeCollectForExport()` 对比

**[scopeCollect()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Model.php#L103-L127)** —— 列表页使用，**有分页**：

```php
public function scopeCollect($query, $sort = 'name')
{
    $request = request();
    $request_sort = $request->get('sort');

    $query->usingSearchString()->sortable($sort);

    if ($request->expectsJson() && $request->isNotApi()) {
        return $query->get();    // API 请求：返回全部
    }

    $request->merge(['sort' => $request_sort]);
    $limit = (int) $request->get('limit', setting('default.list_limit', '25'));

    return $query->paginate($limit);  // Web 请求：分页返回
}
```

**[scopeCollectForExport()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Model.php#L152-L173)** —— 导出使用，**突破分页**：

```php
public function scopeCollectForExport($query, $ids = [], $sort = 'name', $id_field = 'id')
{
    $request = request();

    // 1. 如有 ids 则限定范围
    if (!empty($ids)) {
        $query->whereIn($id_field, (array) $ids);
    }

    $search = $request->get('search');
    $query->usingSearchString($search)->sortable($sort);

    // 2. 分页参数计算
    $page = (int) $request->get('page');
    $limit = (int) $request->get('limit', setting('default.list_limit', '25'));
    $offset = $page ? ($page - 1) * $limit : 0;

    // 3. 关键判断：是否跳过分页限制
    if (! $this instanceof WithParentSheet && (empty($ids) || count((array) $ids) > $limit)) {
        $query->offset($offset)->limit($limit);
    }

    // 4. 使用 cursor() 而非 get()，减少内存占用
    return $query->cursor();
}
```

### 8.3 分页突破的三重机制

**机制一：有 ids 时直接 `whereIn`，不分页**

当用户勾选了特定行导出时，`$ids` 不为空且数量 ≤ `$limit`，此时 `whereIn` 已限定了范围，不需要再分页。

**机制二：`WithParentSheet` 标记接口跳过分页**

这是最核心的设计。判断条件：

```php
if (! $this instanceof WithParentSheet && (empty($ids) || count((array) $ids) > $limit)) {
    $query->offset($offset)->limit($limit);
}
```

- **主表 Sheet**（如 Bills）：**不实现** `WithParentSheet`，所以会走分页逻辑（`offset + limit`），只导出当前页
- **子表 Sheet**（如 BillItems, BillItemTaxes）：**实现了** `WithParentSheet`，所以 `$this instanceof WithParentSheet` 为 `true`，条件不成立，**跳过分页**，通过 `$ids` 的 `whereIn('document_id', $ids)` 来限定范围

为什么子表要跳过分页？因为子表的数据是按父表 ID 过滤的，而非独立分页。如果子表也分页，就会丢失与父表当前页不对应的子记录。

**机制三：`cursor()` 替代 `get()`，流式输出**

`scopeCollectForExport()` 最终返回 `$query->cursor()` 而非 `$query->get()`。
- `get()` 一次性加载所有数据到内存
- `cursor()` 返回 `LazyCollection`，逐行从数据库读取，**内存占用恒定**，适合大批量导出

### 8.4 导出全量时的 ids 自动补全

在 [Utilities/Export.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/Export.php#L26-L28) 中：

```php
//Todo: This improvement solves the filter issue on multiple excel sheets. This solution is a temporary solution.
if (empty($class->ids) && method_exists($class, 'sheets') && is_array($sheets = $class->sheets())) {
    $class->ids = ! empty($ids = (new $sheets[array_key_first($sheets)])->collection()->pluck('id')->toArray()) ? $ids : [0];
}
```

当 `$ids` 为空（全量导出）且是多 Sheet 导出时，先实例化第一个 Sheet（主表），调用其 `collection()` 获取所有 ID，然后回填给 `$class->ids`。这样后续子 Sheet 的 `whereIn($id_field, $ids)` 才能正确限定范围。注释中标注为 "temporary solution"，因为要提前执行一次查询。

### 8.5 完整的分页突破决策树

```
用户触发导出
  ↓
是否有选中 ids？
  ├─ 有 → scopeCollectForExport: whereIn(ids), 不分页, cursor()
  └─ 无 → 是否多 Sheet？
       ├─ 否（单 Sheet）→ scopeCollectForExport: offset+limit 分页, cursor()
       └─ 是（多 Sheet）→ 先查主表获取全部 ids
            ├─ 主表 Sheet（非 WithParentSheet）→ offset+limit 分页, cursor()
            └─ 子表 Sheet（WithParentSheet）→ whereIn(parent_ids), 不分页, cursor()
```

### 8.6 导入侧的分块机制

导入侧通过 `WithChunkReading` + `WithLimit` 控制读取量：

- **[chunkSize()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Import.php#L150-L153)**：`config('excel.imports.chunk_size')`，每块处理的行数（如 100 行），减少单次内存峰值
- **[limit()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Import.php#L155-L158)**：`config('excel.imports.row_limit')`，最大导入行数，防止用户上传超大文件

两者协同：`limit` 控制总共读多少行，`chunkSize` 控制每批处理多少行。配合 `ShouldQueue`，每个 chunk 是一个独立的队列 Job，实现真正的大文件并行处理。

---

## 九、导入失败提示的完整处理链路

导入失败提示分为**同步**和**异步**两条路径，处理方式完全不同。

### 9.1 同步导入的失败处理

当 `should_queue()` 返回 `false` 时，[Utilities/Import.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/Import.php#L34-L35) 直接同步执行：

```php
$class->import($file);
```

如果出错，进入 catch 块，调用 [flashFailures()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/Import.php#L82-L99)：

```php
protected static function flashFailures(Throwable $e): string
{
    // 非 ValidationException → 直接返回错误消息字符串
    if (! $e instanceof ValidationException) {
        return $e->getMessage();
    }

    // ValidationException → 逐条解析失败信息并 flash 到 session
    foreach ($e->failures() as $failure) {
        $message = trans('messages.error.import_column', [
            'message'   => collect($failure->errors())->first(),  // 第一条错误消息
            'column'    => $failure->attribute(),                  // 字段名
            'line'      => $failure->row(),                       // 行号
        ]);

        flash($message)->error()->important();
    }

    return '';  // 返回空串，因为消息已通过 flash 写入 session
}
```

**关键细节：**
- `SheetNotFoundException` 不上报（`! $e instanceof SheetNotFoundException` 才 `report($e)`），因为多 Sheet 导入时用户的 Excel 可能缺少某些 Sheet，这属于正常情况
- `ValidationException` 的行级错误被翻译为 `messages.error.import_column` 模板，格式类似 "第 5 行的 name 字段：该字段为必填项"
- 非 `ValidationException` 的错误直接返回 `getMessage()`，由调用方处理

### 9.2 异步导入的失败处理

当 `should_queue()` 为 `true` 时，导入走队列。此时无法直接 flash 消息到 session（请求已结束），需要通过通知机制。

**失败处理入口：[Providers/Queue.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Queue.php#L82-L127)**

```php
app('events')->listen(JobFailed::class, function ($event) {
    // 1. 只关心 ValidationException
    if (!$event->exception instanceof \Maatwebsite\Excel\Validators\ValidationException) {
        return;
    }

    // 2. 从 Job payload 中反序列化出原始 Import 类
    $body = $event->job->getRawBody();
    $payload = json_decode($body);
    $excel_job = unserialize($payload->data->command);

    // 3. 只处理 ReadChunk Job（Maatwebsite 的分块读取 Job）
    if (!$excel_job instanceof \Maatwebsite\Excel\Jobs\ReadChunk) {
        return;
    }

    // 4. 通过反射获取 Import 实例
    $ref = new \ReflectionProperty($excel_job, 'import');
    $ref->setAccessible(true);
    $class = $ref->getValue($excel_job);

    // 5. 只处理本项目的 Import 类
    if (!$class instanceof \App\Abstracts\Import 
        && !$class instanceof \App\Abstracts\ImportMultipleSheets) {
        return;
    }

    // 6. 组装错误并发送通知
    $errors = [];
    foreach ($event->exception->failures() as $failure) {
        $message = trans('messages.error.import_column', [
            'message'   => collect($failure->errors())->first(),
            'column'    => $failure->attribute(),
            'line'      => $failure->row(),
        ]);
        $errors[] = $message;
    }

    if (!empty($errors)) {
        $class->user->notify(new ImportFailed($errors));
    }
});
```

**这段代码的设计要点：**
- 监听的是 Laravel 队列的 `JobFailed` 事件，而非 Maatwebsite 的内部事件，因为队列 Job 失败时已经脱离了 HTTP 请求上下文
- 通过反射（`ReflectionProperty`）从序列化的 Job 中提取 Import 实例，再从中获取 `$class->user` 来发送通知
- 错误消息格式与同步模式完全一致（`messages.error.import_column`），保证用户体验统一

**[ImportFailed 通知](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Notifications/Common/ImportFailed.php)** 同时通过 **邮件** 和 **数据库** 两个渠道发送：
- `toMail()`：逐条列出错误
- `toArray()`：写入 `notifications` 表，前端可展示

### 9.3 异步导入的成功处理

[Utilities/Import.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Utilities/Import.php#L65-L79) 中的 `importQueue()` 方法：

```php
protected static function importQueue($class, $file, string $translation): void
{
    $rows = $class->toArray($file);  // 先预读获取行数

    $total_rows = 0;
    if (! empty($rows[0])) {
        $total_rows = count($rows[0]);
    } else if (! empty($sheets = $class->sheets())) {
        $total_rows = count($rows[array_keys($sheets)[0]]);
    }

    $class->queue($file)->onQueue('imports')->chain([
        new NotifyUser(user(), new ImportCompleted($translation, $total_rows))
    ]);
}
```

**注意：** `toArray($file)` 在入队前执行了一次完整读取，仅用于统计行数。这是因为队列完成后需要知道导入了多少行，而队列 Job 内部无法再获取总数。这是一个性能代价（文件被读了两次），但换来了用户体验上的行数反馈。

### 9.4 导出失败的异步处理

[Abstracts\Export::failed()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Export.php#L126-L133) 是 Laravel 队列 Job 的 `failed` 钩子：

```php
public function failed(\Throwable $exception): void
{
    if (! $this->user) {
        return;
    }

    $this->user->notify(new ExportFailed($exception->getMessage()));
}
```

导出失败的处理比导入简单，只发送 `ExportFailed` 通知（包含异常消息），不像导入那样需要行级错误解析。

### 9.5 同步 vs 异步失败处理对比

| 维度 | 同步模式 | 异步模式 |
|------|----------|----------|
| **触发时机** | `catch (Throwable $e)` 直接捕获 | `JobFailed` 事件监听 |
| **消息传递** | `flash()` 写入 session | `notify()` 发送通知 |
| **错误格式** | 行级（行号+字段+消息） | 行级（行号+字段+消息） |
| **通知渠道** | 页面闪存消息 | 邮件 + 数据库 |
| **非验证错误** | 直接 `getMessage()` | 不处理（只关心 ValidationException） |
| **SheetNotFound** | 静默忽略（不 report） | 不处理 |

---

## 十、事务边界分析

### 10.1 导入侧：无全局事务，逐行独立

**关键发现：Import 抽象类和具体实现中没有任何 `DB::transaction` 包裹。**

[Abstracts\Import::model()](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Abstracts/Import.php) 没有定义默认实现，由子类覆写。以 [Items Sheet](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Imports/Common/Sheets/Items.php) 为例：

```php
public function model(array $row)
{
    if (self::hasRow($row)) { return; }
    return new Model($row);  // 仅创建 Model 实例，由 Maatwebsite 批量插入
}
```

这意味着：
- **没有全局事务**：1000 行导入中第 500 行失败，前 499 行已经入库，不会回滚
- **没有行级事务**：每行的 `new Model($row)` 不在事务中

### 10.2 为什么不需要全局事务？

这是**有意为之的设计选择**，原因有三：

1. **分块队列模式**：`WithChunkReading` + `ShouldQueue` 意味着数据被分成多个 chunk，每个 chunk 是一个独立的队列 Job。不同 chunk 可能在不同 worker 上并行执行，无法共享数据库连接，**物理上不可能**使用全局事务

2. **容错优先**：会计系统导入几百条交易时，宁可部分成功部分失败，也不愿因一条错误行导致全部回滚后重头来过。部分成功的行用户可以看到哪些已导入，只修复失败行即可

3. **Trait 层的自动创建已在 Job 事务中**：当导入时通过 Trait 的 `getCategoryIdFromName()` 等方法自动创建关联对象时，走的是 `dispatch(new CreateCategory($data))`。而 [CreateCategory Job](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Jobs/Setting/CreateCategory.php) 内部有 `\DB::transaction()` 包裹，保证了单个关联对象创建的原子性

### 10.3 事务边界的具体位置

```
导入流程中的事务边界：

┌─────────────────────────────────────────┐
│ 队列 Job: ReadChunk (chunk_size=100)    │  ← 无事务包裹
│                                         │
│  foreach row in chunk:                  │
│    map($row)                            │
│      └─ getCategoryId()                 │
│           └─ dispatch(CreateCategory)   │  ← 内部有 DB::transaction
│    withValidator()                      │
│    model($row)                          │
│      └─ new Model($row)                 │  ← 无事务，Maatwebsite 批量插入
│                                         │
│  [验证失败] → ValidationException       │
│    → 当前 chunk 的所有行不会插入        │
│    → 但之前 chunk 的行已持久化          │
└─────────────────────────────────────────┘
```

**验证失败的"隐式回滚"：** Maatwebsite 的 `WithValidation` 会在整个 chunk 验证完毕后才批量插入。如果 chunk 内任何一行验证失败，整个 chunk 的 `model()` 返回值都不会被插入——但这仅限当前 chunk，之前已处理的 chunk 不会回滚。

### 10.4 导出侧的事务特性

导出侧天然是**只读操作**（`FromCollection` + `cursor()`），不涉及写入，因此不需要事务考虑。

唯一涉及写入的是异步导出完成后的 `CreateMediableForExport` Job，它负责在数据库中创建 Media 记录供用户下载，但这个写入是独立的、幂等的，失败后可以重试。

### 10.5 队列上下文的特殊事务问题

[Providers/Queue.php](file:///d:/fz/0601-1/solo-dogfeeding/code/26-akaunting/app/Providers/Queue.php#L32-L40) 中通过 `createPayloadUsing` 向每个队列 Job 的 payload 注入 `company_id`：

```php
app('queue')->createPayloadUsing(function ($connection, $queue, $payload) {
    $company_id = company_id();
    if (empty($company_id)) {
        return [];
    }
    return ['company_id' => $company_id];
});
```

然后在 `JobProcessing` 事件中恢复公司上下文：

```php
app('events')->listen(JobProcessing::class, function ($event) {
    $payload = $event->job->payload();
    if (! array_key_exists('company_id', $payload)) {
        return;
    }

    try {
        $company = company($payload['company_id']);
    } catch (\Throwable $e) {
        $event->job->delete();  // 公司不存在则删除 Job
        return;
    }

    $company->makeCurrent();  // 恢复多租户上下文
});
```

**这对事务的影响：** 多租户的 `company_id` 上下文必须在 Job 执行前恢复，否则 Trait 中的 `company_id()` 和全局 scope 都会失效。这个机制不直接控制事务边界，但保证了导入 Job 在正确的租户上下文中运行——这是数据隔离的前提，比事务包裹更基础。

### 10.6 事务边界总结

| 层级 | 事务策略 | 原因 |
|------|----------|------|
| **整体导入** | ❌ 无全局事务 | 分块队列模式物理上不支持；容错优先于原子性 |
| **单 chunk** | ❌ 无显式事务 | Maatwebsite 内部管理：验证失败则整 chunk 跳过插入 |
| **Trait 自动创建关联** | ✅ Job 内有 `DB::transaction` | 如 `CreateCategory`、`CreateItem` 等 Job 内部包裹事务，保证单条关联创建的原子性 |
| **主表 `new Model($row)`** | ❌ 无事务 | 依赖 Eloquent 的 `$fillable` 批量赋值，单行单次 INSERT |
| **导出** | N/A | 只读操作，无需事务 |
| **队列上下文** | `company_id` 注入 | 保证多租户隔离，不控制事务但更基础 |
