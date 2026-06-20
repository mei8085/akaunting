# Item 数量与单据行金额联动计算链路

## 整体架构概览

系统采用 **前端实时计算 + 后端二次校验计算** 的双层架构：

```
用户输入 (quantity/price/discount/tax)
        ↓
前端 Vue.js 实时计算 (onCalculateTotal)
        ↓
表单提交 → 后端 FormRequest 验证 (calculation_to_quantity)
        ↓
后端 Job 业务逻辑计算 (CreateDocumentItem)
        ↓
数据持久化 (DocumentItem / DocumentTotal)
```

---

## 一、前端联动层（UI 实时计算）

### 1.1 触发联动的输入事件

| 输入字段 | 触发事件 | 文件位置 |
|---------|---------|---------|
| 数量 (quantity) | `@input="onCalculateTotal"` | [line-item.blade.php:110](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/views/components/documents/form/line-item.blade.php#L110-L110) |
| 单价 (price) | `change="onCalculateTotal"` | [line-item.blade.php:140](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/views/components/documents/form/line-item.blade.php#L140-L140) |
| 行折扣 (discount) | `@input="onCalculateTotal"` | [line-item.blade.php:247](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/views/components/documents/form/line-item.blade.php#L247-L247) |
| 税种选择 (tax) | `@change="onCalculateTotal()"` | [line-item.blade.php:329](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/views/components/documents/form/line-item.blade.php#L329-L329) |

### 1.2 前端核心计算流程

**主入口方法：** `onCalculateTotal()` [documents.js:310-393](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/views/common/documents.js#L310-L393)

```
onCalculateTotal()
    ├─ calculateTotalBeforeDiscountAndTax()
    │   └─ 计算每行: item_total = price * calculationToQuantity(quantity)
    │   └─ 计算行折扣: item.discount_amount
    │   └─ 返回折扣前金额数组
    │
    ├─ 遍历所有 items:
    │   ├─ 基础计算: item.total = item.price * calculationToQuantity(item.quantity)
    │   ├─ 应用行折扣
    │   ├─ 应用全局折扣 (按比例分摊)
    │   ├─ calculateItemTax() → 计算各种类型的税
    │   └─ 累加: sub_total / grand_total
    │
    └─ 更新 this.totals (sub/item_discount/discount/taxes/total)
```

### 1.3 前端数学表达式计算

**函数：** `calculationToQuantity(quantity)` [functions.js:31-33](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/plugins/functions.js#L31-L33)

```javascript
const { evaluate } = require('mathjs');

function calculationToQuantity(quantity) {
    return evaluate(quantity.toString());
}
```

**支持的表达式：**
- 四则运算：`2+3`, `10-4`, `5*6`, `20/4`
- 乘法简写：`5x3` (自动转换为 `5*3`)
- 小数：`1.5`, `2,5` (逗号自动转点)
- 复杂表达式：`(2+3)*4`

**示例：**
- 输入 `2x3` → 计算结果 `6`
- 输入 `1.5+2.5` → 计算结果 `4`

---

## 二、表单验证层（FormRequest）

**文件：** [Document.php:97-118](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Http/Requests/Document/Document.php#L97-L118)

在验证阶段就会调用 `calculation_to_quantity()` 进行预处理：

```php
try {
    $items[$key]['quantity'] = calculation_to_quantity($item['quantity']);
} catch (\InvalidArgumentException $e) {
    $quantityRule[] = function ($attribute, $value, $fail) {
        $fail(trans('validation.custom.invalid_quantity', [...]));
    };
}
```

> **关键点：** 验证阶段已经将数学表达式转换为实际数值，后续 Job 拿到的已经是计算后的数量。

---

## 三、后端辅助函数

**文件：** [helpers.php:455-483](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Utilities/helpers.php#L455-L483)

```php
function calculation_to_quantity($quantity)
{
    $quantity = trim((string) $quantity);
    
    // 安全校验：只允许数字和运算符
    if (! preg_match('/^[0-9,+\-x*\/().\s]+$/', $quantity)) {
        throw new \InvalidArgumentException('Invalid mathematical expression.');
    }
    
    $quantity = str_replace(',', '.', $quantity);
    $quantity = str_ireplace('x', '*', $quantity);
    
    try {
        $result = eval('return ' . $quantity . ';');
    } catch (\Throwable $e) {
        throw new \InvalidArgumentException('Error evaluating the expression: ' . $e->getMessage());
    }
    
    return $result;
}
```

> **安全机制：** 先用正则白名单校验，再用 eval 计算，防止代码注入。

---

## 四、后端业务逻辑层（Jobs）

### 4.1 主流程：CreateDocumentItemsAndTotals

**文件：** [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php)

```
handle()
    ├─ createItems() → 循环创建所有行项目
    │   ├─ 处理全局折扣分摊 (fixedDiscountCalculate)
    │   └─ 为每个 item 调用 CreateDocumentItem
    │
    ├─ 创建 DocumentTotal 记录：
    │   ├─ sub_total (小计)
    │   ├─ item_discount (行折扣汇总)
    │   ├─ discount (全局折扣)
    │   ├─ tax (按税种分组的税额)
    │   └─ total (最终总计)
    │
    └─ 更新 $this->request['amount'] 为最终总金额
```

### 4.2 单行计算核心：CreateDocumentItem

**文件：** [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php)

#### 核心计算公式

```php
// Line 34: 基础金额 = 单价 × 数量
$item_amount = (double) $this->request['price'] * (double) calculation_to_quantity($this->request['quantity']);

// Line 36-45: 应用行折扣
if (! empty($this->request['discount'])) {
    if ($this->request['discount_type'] === 'percentage') {
        $item_discounted_amount -= ($item_amount * ($this->request['discount'] / 100));
    } else {
        $item_discounted_amount -= $this->request['discount'];
    }
}

// Line 48-56: 应用全局折扣
if (! empty($this->request['global_discount'])) {
    // 类似行折扣逻辑
}
```

#### 税种计算逻辑（Line 69-156）

| 税种类型 | 计算公式 | 代码位置 |
|---------|---------|---------|
| **inclusive (价内税)** | `金额 - (金额 / (1 + 税率/100))` | [Line 84](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php#L84-L84) |
| **fixed (固定税)** | `税率 × 数量` | [Line 100](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php#L100-L100) |
| **normal (普通税)** | `实际价格 × (税率/100)` | [Line 115](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php#L115-L115) |
| **withholding (预扣税)** | `-(实际价格 × (税率/100))` | [Line 130](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php#L130-L130) |
| **compound (复合税)** | `(当前累计金额 / 100) × 税率` | [Line 145](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php#L145-L145) |

> **税种执行顺序：** inclusive → fixed → normal → withholding → compound
>
> 注意：compound 税是基于前面所有税计算后的累计金额计算的。

#### 4.2.1 复合税（Compound）Base 的累加构成详解

复合税的计算基数（base）是**动态累加**的，每一步计算都会影响最终结果。

**后端计算链路（CreateDocumentItem.php:58-155）**

```php
// Line 60: 初始化三个关键变量
$actual_price_item = $item_amount = $item_discounted_amount;
// 此时：三者都等于 折扣后的金额 (price × quantity - 行折扣 - 全局折扣)

// ── 第1步：inclusive 价内税 ──────────────────────────────────
// Line 82-96
foreach ($inclusives as $inclusive) {
    $tax_amount = $item_discounted_amount - ($item_discounted_amount / (1 + $inclusive->rate / 100));
    $item_tax_total += $tax_amount;
}
$actual_price_item = $item_discounted_amount - $item_tax_total;
// 此时：$actual_price_item = 不含价内税的净价
// 注意：$item_amount 此时未变，还是折扣后的金额

// ── 第2步：fixed 固定税 ────────────────────────────────────
// Line 98-111
foreach ($fixeds as $tax) {
    $tax_amount = $tax->rate * (double) $this->request['quantity'];
    $item_amount += $tax_amount;  // ← 累加到 $item_amount
}

// ── 第3步：normal 普通税 ───────────────────────────────────
// Line 113-126
foreach ($normals as $tax) {
    $tax_amount = $actual_price_item * ($tax->rate / 100);
    $item_amount += $tax_amount;  // ← 累加到 $item_amount
}

// ── 第4步：withholding 预扣税 ──────────────────────────────
// Line 128-141
foreach ($withholdings as $tax) {
    $tax_amount = -($actual_price_item * ($tax->rate / 100)); // 负数
    $item_amount += $tax_amount;  // ← 累加到 $item_amount（扣减）
}

// ── 第5步：compound 复合税 ─────────────────────────────────
// Line 143-155
foreach ($compounds as $compound) {
    // 重点：base 是 $item_amount，包含了前面所有税！
    $tax_amount = ($item_amount / 100) * $compound->rate;
    $item_tax_total += $tax_amount;
}
```

**复合税 Base 构成公式：**
```
compound_base = 折扣后金额 + fixed税总额 + normal税总额 + withholding税总额
```

> **关键区别**：normal/withholding 税基于 `$actual_price_item`（折扣后 - 价内税）计算，
> 而 compound 税基于 `$item_amount`（折扣后 + 所有已算税种）计算。

**前端对应逻辑（documents.js:513-524）**
```javascript
// Line 513: 先累加 fixed/normal/withholding 税到 grand_total
item.grand_total += total_tax_amount;

// Line 515-524: 复合税基于累加后的 grand_total 计算
if (compounds.length) {
    compounds.forEach(function(compound) {
        item.tax_ids[compound.tax_index].price = 
            this.numberFormat((item.grand_total / 100) * compound.tax_rate, this.currency.precision);
        
        item.grand_total += item.tax_ids[compound.tax_index].price;
    }, this);
}
```

#### 4.2.2 多税聚合（多税种、多行项目）

当一张单据包含多行、每行又包含多个税种时，需要按税种分组聚合。

**前端聚合逻辑（documents.js:569-587）**
```javascript
calculateTotalsTax(totals_taxes, id, name, price) {
    let total_tax_index = totals_taxes.findIndex(total_tax => total_tax.id === id);
    
    if (total_tax_index === -1) {
        // 新税种，新增记录
        totals_taxes.push({ id: id, name: name, total: price });
    } else {
        // 已有税种，累加金额
        totals_taxes[total_tax_index].total = 
            parseFloat(totals_taxes[total_tax_index].total) + parseFloat(price);
    }
    return totals_taxes;
}
```

**后端聚合逻辑（CreateDocumentItemsAndTotals.php:240-250）**
```php
foreach ((array) $document_item->item_taxes as $item_tax) {
    if (array_key_exists($item_tax['tax_id'], $taxes)) {
        // 已有税种，累加
        $taxes[$item_tax['tax_id']]['amount'] += 
            round((float) $item_tax['amount'], $this->document->currency->precision);
    } else {
        // 新税种，新增
        $taxes[$item_tax['tax_id']] = [
            'name' => $item_tax['name'],
            'amount' => round((float) $item_tax['amount'], $this->document->currency->precision),
        ];
    }
}
```

**聚合后写入 DocumentTotal（CreateDocumentItemsAndTotals.php:95-113）**
```php
// 为每个税种创建一条 DocumentTotal 记录
foreach ($taxes as $tax) {
    DocumentTotal::create([
        'code' => 'tax',
        'name' => Str::ucfirst($tax['name']),
        'amount' => round(abs($tax['amount']), $precision),
        'sort_order' => $sort_order++,
    ]);
}
```

**多税聚合示例：**
```
行1: 商品A，数量×单价=100，税种：增值税13% + 消费税5%
   → 增值税: 13, 消费税: 5
   
行2: 商品B，数量×单价=200，税种：增值税13% + 复合税2%
   → 增值税: 26, 复合税: (200+26)×2% = 4.52

聚合后 DocumentTotal:
  - tax (增值税): 13 + 26 = 39
  - tax (消费税): 5
  - tax (复合税): 4.52
```

### 4.3 全局折扣分摊逻辑

**函数：** `fixedDiscountCalculate()` [CreateDocumentItemsAndTotals.php:265-289](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L265-L289)

当全局折扣是固定金额（非百分比）时，需要按各行金额比例分摊：

```php
// 每行分摊的折扣比例
$item['global_discount'] = ($for_fixed_discount[$key] / ($for_fixed_discount['total'] / 100)) * ($this->request['discount'] / 100);
```

**示例：**
- 行1金额：100，行2金额：300，总计：400
- 全局固定折扣：40
- 行1分摊：(100/400) × 40 = 10
- 行2分摊：(300/400) × 40 = 30

---

## 五、数据模型层

### 5.1 DocumentItem（单据行）

**文件：** [DocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Models/Document/DocumentItem.php)

| 字段 | 类型 | 说明 |
|-----|------|------|
| `quantity` | double | 计算后的实际数量 |
| `price` | double | 单价 |
| `total` | double | 该行总金额（含折扣不含税？看计算逻辑） |
| `tax` | double | 该行税额总计 |
| `discount_rate` | double | 折扣率 |
| `discount_type` | string | 折扣类型：normal/fixed/percentage |

### 5.2 DocumentTotal（单据汇总）

**文件：** [DocumentTotal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Models/Document/DocumentTotal.php)

按 `sort_order` 存储多级汇总：
- `sub_total` - 小计（所有行 `price × quantity` 之和）
- `item_discount` - 行折扣汇总
- `discount` - 全局折扣
- `tax` - 各税种金额（多条记录）
- `total` - 最终总计

### 5.3 Document（单据主表）

**文件：** [Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Models/Document/Document.php)

| 字段 | 说明 |
|-----|------|
| `amount` | 单据总金额（等于 DocumentTotal 中 code=total 的 amount） |
| `discount_rate` | 全局折扣率 |
| `discount_type` | 全局折扣类型 |

---

## 六、完整调用链路时序图

```
用户操作
    │
    ▼
[UI] 输入 quantity = "2x3"
    │
    ├─ @input 事件触发
    │
    ▼
[前端 Vue] onCalculateTotal()
    ├─ calculationToQuantity("2x3") → mathjs.evaluate → 6
    ├─ item.total = price × 6
    ├─ 计算折扣、税金
    └─ 更新 UI 显示的金额
    │
    ▼
[用户提交] 表单 POST
    │
    ▼
[后端 FormRequest] Document::rules()
    ├─ calculation_to_quantity("2x3") → eval → 6
    ├─ 验证通过，将 quantity 替换为 6
    └─ 传递给 Job
    │
    ▼
[后端 Job] CreateDocumentItemsAndTotals::handle()
    ├─ 遍历 items，为每个调用 CreateDocumentItem
    │
    ▼
[后端 Job] CreateDocumentItem::handle()
    ├─ $item_amount = price × quantity (已是6)
    ├─ 应用行折扣
    ├─ 应用全局折扣分摊
    ├─ 计算各税种金额
    ├─ 写入 DocumentItem
    └─ 写入 DocumentItemTax
    │
    ▼
[CreateDocumentItemsAndTotals 继续]
    ├─ 汇总所有行，创建 DocumentTotal 记录
    ├─ sub_total / item_discount / discount / taxes / total
    └─ 更新 Document.amount = total
    │
    ▼
[完成] 数据持久化
```

---

## 七、关键代码引用速查

| 功能 | 文件 | 行号 |
|-----|------|------|
| 前端计算入口 | [documents.js](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/views/common/documents.js) | L310 |
| 前端表达式计算 | [functions.js](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/plugins/functions.js) | L31 |
| 后端表达式计算 | [helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Utilities/helpers.php) | L455 |
| 表单验证预处理 | [Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Http/Requests/Document/Document.php) | L97 |
| 单行金额计算 | [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php) | L34 |
| 税种计算 | [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItem.php) | L69-L156 |
| 全局折扣分摊 | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | L265 |
| 汇总计算 | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | L29 |
| 单据行模型 | [DocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Models/Document/DocumentItem.php) | - |
| 单据主模型 | [Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Models/Document/Document.php) | - |

---

## 八、计算示例（端到端）

**场景：** 创建一张发票，包含一行商品

| 参数 | 值 |
|-----|----|
| 商品名称 | 测试商品 |
| 数量 | `2x3` (表达式，实际6) |
| 单价 | 100.00 |
| 行折扣 | 10% |
| 税种 | 普通增值税 13% |
| 全局折扣 | 固定金额 50 |

**计算过程：**

1. **前端计算：**
   - `calculationToQuantity("2x3")` → 6
   - 基础金额：100 × 6 = 600
   - 行折扣：600 × 10% = 60 → 折后：540
   - 全局折扣分摊：540 (只有一行，全部分摊) → 折后：490
   - 增值税：490 × 13% = 63.7
   - 行总计：490 + 63.7 = 553.7

2. **后端验证：**
   - `calculation_to_quantity("2x3")` → 6
   - 验证通过

3. **后端 Job 计算：**
   - 与前端逻辑一致，二次校验计算
   - 写入 DocumentItem: quantity=6, price=100, total=490, tax=63.7
   - 写入 DocumentTotal: sub_total=600, item_discount=60, discount=50, tax=63.7, total=553.7
   - 更新 Document.amount = 553.7
