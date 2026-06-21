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

复合税的计算基数（base）是**动态累加**的，但有一个**关键的剥离机制**：价内税会被从 base 中剥离，不会被复合税叠加计算。

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
// ⚠️ 重要：$item_amount 此时未变，还是 $item_discounted_amount！
//         价内税只影响 $actual_price_item，不影响 $item_amount
//         这就是为什么复合税 base 中不包含价内税

// ── 第2步：fixed 固定税 ────────────────────────────────────
// Line 98-111
foreach ($fixeds as $tax) {
    // ⚠️ 数量参数说明：
    // 这里的 $this->request['quantity'] 已经经过 FormRequest 中的
    // calculation_to_quantity() 转换（Document.php L97-118）
    // 例如输入 "2x3" 已经变成 6，所以这里 (double) 只是类型转换
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
    // 重点：base = $item_amount = 折扣后金额 + fixed + normal + withholding
    //       但 不包含 价内税！因为 $item_amount 在价内税阶段没有被修改
    $tax_amount = ($item_amount / 100) * $compound->rate;
    $item_tax_total += $tax_amount;
}
```

**⚠️ 复合税 Base 漏掉价内税的原理：**

关键在于 Line 60 的初始化和 Line 95 的赋值：
```php
Line 60:  $actual_price_item = $item_amount = $item_discounted_amount;
          // 三者指向同一个值，但后续修改互不影响

Line 82-96: 价内税计算时
          - 只修改了 $item_tax_total（累加）
          - 只修改了 $actual_price_item（剥离价内税）
          - ❌ 没有修改 $item_amount！！

Line 95:  $actual_price_item = $item_discounted_amount - $item_tax_total;
          // 价内税只从 $actual_price_item 中剥离
```

所以复合税 base 的真实构成是：

```
compound_base = $item_amount
              = $item_discounted_amount  (初始值)
              + Σ fixed_tax_amount       (第2步累加)
              + Σ normal_tax_amount      (第3步累加)
              + Σ withholding_tax_amount (第4步累加，负数)
```

**修正后的复合税 Base 公式：**
```
compound_base = 折扣后金额 + fixed税总额 + normal税总额 + withholding税总额
              = 不含价内税，且是扣除全局折扣后的金额
```

> **关键区别**：
> - normal/withholding 税基于 `$actual_price_item`（折扣后 - 价内税 - 全局折扣）
> - compound 税基于 `$item_amount`（折扣后 - 全局折扣 + fixed/normal/withholding）
> - **两者都不包含价内税，但基数完全不同**

**前端对应逻辑（documents.js:513-524）**
```javascript
// Line 513: 先累加 fixed/normal/withholding 税到 grand_total
// total_tax_amount = Σ fixed + Σ normal + Σ withholding(负数)
item.grand_total += total_tax_amount;

// Line 515-524: 复合税基于累加后的 grand_total 计算
if (compounds.length) {
    compounds.forEach(function(compound) {
        // 这里的 grand_total 中也不包含价内税！
        // 因为价内税在 L471 是做减法：item.total = grand_total - inclusive_tax_total
        // 而这里用的是 item.grand_total（减法之前的值）
        item.tax_ids[compound.tax_index].price = 
            this.numberFormat((item.grand_total / 100) * compound.tax_rate, this.currency.precision);
        
        item.grand_total += item.tax_ids[compound.tax_index].price;
    }, this);
}
```

**前端 fixed 税的数量参数（documents.js:477）：**
```javascript
// 前端用 calculationToQuantity() 计算原始字符串表达式
// 与后端 FormRequest 中提前转换不同，前端是即时计算
item.tax_ids[fixed.tax_index].price = 
    this.numberFormat(fixed.tax_rate * calculationToQuantity(item.quantity), ...);
// 例如 item.quantity = "2x3"，这里先计算 2*3=6，再乘税率
```

#### 4.2.1.1 全局折扣加回与税种计算的边界判断

**⚠️ 极其重要但容易被忽略的逻辑（CreateDocumentItem.php:158-162）：**

在所有税种计算完成后，代码会执行全局折扣「加回」操作：

```php
// Line 47-56: 计算时先扣除全局折扣
if (! empty($this->request['global_discount'])) {
    if ($this->request['global_discount_type'] === 'percentage') {
        $global_discount = $item_discounted_amount * ($this->request['global_discount'] / 100);
    } else {
        $global_discount = $this->request['global_discount'];
    }
    $item_discounted_amount -= $global_discount;  // ← 扣除全局折扣
}

// ... 中间执行所有税种计算（基于扣除全局折扣后的金额）...

// Line 158-162: 税种计算完后，把全局折扣加回来
if (! empty($global_discount)) {
    $actual_price_item += $global_discount;  // ← 加回
    $item_amount += $global_discount;        // ← 加回
    $item_discounted_amount += $global_discount;  // ← 加回
}
```

**边界判断条件分析：**

| 场景 | `$this->request['global_discount']` | `$global_discount` 变量 | Line 48 执行 | Line 158 执行 |
|------|-------------------------------------|------------------------|-------------|--------------|
| 无全局折扣 | 空/0 | 未定义 | 否 | 否（未定义变量被当作 null，`!empty(null)` = false） |
| 有全局折扣，百分比类型 | 非空，类型=percentage | 定义 = 行金额 × 折扣率% | 是 | 是（加回） |
| 有全局折扣，固定类型 | 非空，类型=fixed | 定义 = 分摊到该行的折扣额 | 是 | 是（加回） |

**加回逻辑的业务含义：**
```
税种计算基数 = 行折扣后金额 - 全局折扣   （基于扣除后的金额算税）
最终存储 total = 税种计算后金额 + 全局折扣  （加回来，用于行合计显示）
```

> **结论：全局折扣只影响税种计算的基数，不影响最终行金额（total 字段）。**
> 全局折扣是在 DocumentTotal 层（汇总层）统一扣减的，而不是在 DocumentItem 层。

#### 4.2.1.2 DocumentItemTax 中 withholding 存储绝对值的机制

**⚠️ 存储时使用 abs() 取绝对值（CreateDocumentItem.php:193）：**

```php
// Line 130: withholding 计算时是负数
$tax_amount = -($actual_price_item * ($tax->rate / 100));  // 例: -9.09

// Line 135: $item_taxes[] 中存的是带符号的原始值
$item_taxes[] = [
    ...
    'amount' => $tax_amount,  // 仍然是 -9.09
];

// Line 139: 累加时用的是带符号值（所以总计会被扣减）
$item_tax_total += $tax_amount;

// ...

// Line 191-197: 写入数据库时，用 abs() 取绝对值！
foreach ($item_taxes as $item_tax) {
    $item_tax['document_item_id'] = $document_item->id;
    $item_tax['amount'] = round(abs($item_tax['amount']), $precision);
    //         ^^^^^ 关键！-9.09 变成了 9.09 存入表中
    
    DocumentItemTax::create($item_tax);
}
```

**两层汇总时的不同处理：**

```
DocumentItem 层（Line 173）:
    $this->request['tax'] = round($item_tax_total, $precision);
    // 这里用的是 $item_tax_total，包含了负数的 withholding
    // 所以 tax 是 (价内税 + fixed + normal + withholding(负) + compound)

DocumentTotal 层（CreateDocumentItemsAndTotals.php:240-250）:
    foreach ($document_item->item_taxes as $item_tax) {
        // ⚠️ item_tax['amount'] 此时是存储在内存中的原始值（带符号）
        //     因为 Line 187 赋值的是 $item_taxes 原始数组（还没被 abs）
        //     而 Line 193 只是在 create 时用了 abs，但没有修改内存中的值
        $taxes[$item_tax['tax_id']]['amount'] += (float) $item_tax['amount'];
    }

DocumentTotal 存储时（CreateDocumentItemsAndTotals.php:103）:
    DocumentTotal::create([
        ...
        'amount' => round(abs($tax['amount']), $precision),
        //       ^^^^ 同样用 abs() 存绝对值
    ]);

DocumentTotal 累加到 amount（CreateDocumentItemsAndTotals.php:109）:
    $this->request['amount'] += $tax['amount'];
    // 这里 $tax['amount'] 还是带符号的原始值（-9.09）
    // 因为 taxes 数组在 L240-250 累加时用的是内存中的原始 item_tax
```

**withholding 在各层数据中的值对比：**

| 层级 | 字段 | 值（例） | 符号 | 位置 |
|------|------|---------|------|------|
| 计算中间值 | `$tax_amount` (withholding) | -9.09 | 负 | CreateDocumentItem.php:130 |
| 内存数组 `$item_taxes` | `amount` | -9.09 | 负 | CreateDocumentItem.php:135 |
| DocumentItem 表 | `tax` (合计) | 100 + 10 + 23.64 - 9.09 + 6.74 = **131.29** | 含负 | CreateDocumentItem.php:173 |
| **document_item_taxes 表** | `amount` | **9.09** | **正（abs）** | CreateDocumentItem.php:193 |
| DocumentTotal 聚合 `taxes[]` | `amount` | -9.09（累加自内存数组） | 负 | CreateDocumentItemsAndTotals.php:245 |
| **document_totals 表 (code=tax)** | `amount` | **9.09** | **正（abs）** | CreateDocumentItemsAndTotals.php:103 |
| DocumentTotal 累加到 `amount` | - | -9.09（影响总计） | 负 | CreateDocumentItemsAndTotals.php:109 |
| documents 表 | `amount` | 受影响（扣减了 9.09） | 含负 | CreateDocumentItemsAndTotals.php:144 |

> **关键结论：**
> 1. **存储层**（document_item_taxes 和 document_totals 表）存的是 **绝对值**
> 2. **计算层**（内存数组、税合计、单据总计）用的是 **带符号值**
> 3. 这导致一个有趣的现象：表中 withholding 的 amount 是正数，但在最终单据金额计算时它确实是被扣减的
> 4. 读取展示时需要根据税种类型（withholding）来判断是否显示负号（前端展示逻辑处理）

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
    // ⚠️ 注意：这里的 $item_tax['amount'] 来自 CreateDocumentItem 中 L187
    //         赋值的内存数组 $item_taxes（带符号，未 abs），不是从数据库读的
    if (array_key_exists($item_tax['tax_id'], $taxes)) {
        // 已有税种，累加（withholding 为负）
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
        // ⚠️ 存储时取绝对值（withholding -9.09 → 9.09）
        'amount' => round(abs($tax['amount']), $precision),
        'sort_order' => $sort_order++,
    ]);
    
    // 但累加到 amount 时用的是带符号的原始值
    $this->request['amount'] += $tax['amount'];
}
```

**多税聚合示例：**
```
行1: 商品A，数量×单价=100，税种：增值税13% + 预扣税5%
   → 增值税: 13, 预扣税: -5
   
行2: 商品B，数量×单价=200，税种：增值税13% + 复合税2%
   → 增值税: 26, 复合税: (200+26)×2% = 4.52

内存聚合（带符号）:
  - 增值税: 13 + 26 = 39
  - 预扣税: -5
  - 复合税: 4.52

document_totals 表存储（绝对值）:
  - tax (增值税): 39
  - tax (预扣税): 5 (abs)
  - tax (复合税): 4.52

amount 累加（带符号）:
  amount = 300 (行合计) + 39 + (-5) + 4.52 = 338.52
```

#### 4.2.3 复合税完整计算示例（含全局折扣 + 加回逻辑）

**场景：** 
- 单价=100，数量=2，行折扣=0
- 全局折扣类型=percentage，折扣率=10%（即该行分摊 20）
- 税种：价内税10% + 固定税5/件 + 普通税13% + 预扣税5% + 复合税3%

**计算过程：**

```
初始：
  基础金额 = 100 × 2 = 200
  行折扣 = 0
  全局折扣扣除: global_discount = 200 × 10% = 20
  $item_discounted_amount = 200 - 0 = 200 → 扣除全局 = 180

Line 60 初始化（扣除全局折扣后的值）：
  $actual_price_item = $item_amount = $item_discounted_amount = 180

第1步 - 价内税 (inclusive 10%)：
  税额 = 180 - (180 / (1 + 10/100) = 180 - 163.64 = 16.36
  $item_tax_total = 16.36
  $actual_price_item = 180 - 16.36 = 163.64
  ⚠️ $item_amount 保持 180 不变（价内税不影响 compound base）

第2步 - 固定税 (fixed 5/件)：
  // 数量已由 FormRequest 转换: "2" → (double)2
  税额 = 5 × 2 = 10
  $item_amount = 180 + 10 = 190
  $item_tax_total = 16.36 + 10 = 26.36

第3步 - 普通税 (normal 13%)：
  税额 = 163.64 × 13% = 21.27
  $item_amount = 190 + 21.27 = 211.27
  $item_tax_total = 26.36 + 21.27 = 47.63

第4步 - 预扣税 (withholding 5%)：
  税额 = -(163.64 × 5%) = -8.18
  $item_amount = 211.27 - 8.18 = 203.09
  $item_tax_total = 47.63 - 8.18 = 39.45

第5步 - 复合税 (compound 3%)：
  ⚠️  base = $item_amount = 203.09
  ( = 全局折扣后金额 180 + fixed 10 + normal 21.27 + withholding -8.18 )
  ( 不含价内税！因为 $item_amount 在价内税阶段没有被修改 )
  税额 = 203.09 × 3% = 6.09
  $item_tax_total = 39.45 + 6.09 = 45.54

Line 158: 全局折扣加回 (global_discount = 20)：
  $actual_price_item = 163.64 + 20 = 183.64  ← 最终 total
  $item_amount = 203.09 + 20 = 223.09
  $item_discounted_amount = 180 + 20 = 200

最终结果：
  行金额 total = $actual_price_item = 183.64 (价内税后净价 + 加回全局折扣)
  税合计 tax = 45.54 (含预扣税的负号影响)
  DocumentItemTax 表中 withholding 的 amount = abs(-8.18) = 8.18 (正数存储)
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

## 七、编辑保存与单独删除行项目的完整联动路径

### 7.1 发票行项目编辑保存的完整链路

**关键设计：编辑时不存在单独保存某一行的 API。** 行项目的编辑修改是通过**提交整张单据**来完成的。

#### 7.1.1 路由与入口

| 模式 | HTTP 方法 | 路由 | Controller 方法 |
|------|----------|------|----------------|
| 创建 | POST | `invoices.store` | [Invoices::store](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Http/Controllers/Sales/Invoices.php#L100-L125) |
| 编辑 | PATCH | `invoices.update` | [Invoices::update](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Http/Controllers/Sales/Invoices.php#L189-L214) |

**路由配置（admin.php:83）**：
```php
Route::resource('invoices', 'Sales\Invoices', [
    'middleware' => ['date.format', 'money', 'dropzone']
]);
```

#### 7.1.2 前端提交链路

**表单构建** [content.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/views/components/documents/form/content.blade.php#L4-L31)：
```blade
<x-form 
    id="{{ $formId }}"
    :route="$formRoute"      // 创建: invoices.store, 编辑: [invoices.update, $invoice->id]
    method="{{ $formMethod }}" // 创建: POST, 编辑: PATCH
    :model="$document"
>
```

**表单路由自动生成** [ViewComponents.php:704-721](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Traits/ViewComponents.php#L704-L721)：
```php
protected function getFormRoute($type, $formRoute, $model = false)
{
    $prefix = 'store';
    $parameters = [];
    
    if (! empty($model)) {
        $prefix = 'update';
        $parameters = [$model->id];
    }
    
    return (! empty($model)) ? [$route, $model->id] : $route;
}
```

**前端提交触发**：

1. 点击「保存」按钮 → `onSubmit()` [global.js:290-292](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/mixins/global.js#L290-L292)
```javascript
onSubmit() {
    this.form.submit();
}
```

2. 点击「发送」按钮 → `onSubmitViaSendEmail()` [documents.js:884-889](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/views/common/documents.js#L884-L889)
```javascript
onSubmitViaSendEmail() {
    this.form['senddocument'] = true;
    this.send_to = true;
    this.onSubmit();
}
```

#### 7.1.3 后端处理链路（编辑更新）

**Controller 层** [Invoices::update](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Http/Controllers/Sales/Invoices.php#L189-L214)：
```php
public function update(Document $invoice, Request $request)
{
    $response = $this->ajaxDispatch(new UpdateDocument($invoice, $request));
    // ...
}
```

**Job 层** [UpdateDocument.php:19-87](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/UpdateDocument.php#L19-L87)：
```
UpdateDocument::handle()
    │
    ├─ authorize() → 权限校验
    │
    ├─ event(new DocumentUpdating)
    │
    └─ DB::transaction()
        │
        ├─ 1. deleteRelationships($model, ['items', 'item_taxes', 'totals'], true)
        │   └─ 物理删除所有 DocumentItem / DocumentItemTax / DocumentTotal
        │
        ├─ 2. dispatch(new CreateDocumentItemsAndTotals($model, $request))
        │   ├─ createItems() → 逐行计算 + 生成 DocumentItem
        │   └─ 按顺序创建 DocumentTotal 记录
        │
        ├─ 3. 更新 Document 主表（包括 amount）
        │
        └─ event(new DocumentUpdated)
```

#### 7.1.4 创建时的对比链路

**Job 层** [CreateDocument.php:20-57](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocument.php#L20-L57)：
```
CreateDocument::handle()
    │
    └─ DB::transaction()
        │
        ├─ 1. Document::create() → 创建主表
        │
        ├─ 2. dispatch(new CreateDocumentItemsAndTotals($model, $request))
        │   └─ 与编辑时完全相同的计算逻辑
        │
        └─ 3. $model->update() → 用计算后的 amount 更新主表
```

> **注意**：创建和编辑都会调用同一个 `CreateDocumentItemsAndTotals` Job，区别是编辑前多了一步 `deleteRelationships`。

### 7.2 单独删除行项目的联动路径

**系统没有单独删除某一行项目的 API。** 删除行项目分两个阶段：

#### 7.2.1 阶段一：前端临时删除（未持久化）

**触发**：点击行项目右上角的 × 按钮 → `onDeleteItem(index)` [documents.js:690-695](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/resources/assets/js/views/common/documents.js#L690-L695)

```javascript
onDeleteItem(index) {
    // 从 Vue 响应式数组中移除
    this.items.splice(index, 1);           // UI 显示用
    this.form.items.splice(index, 1);      // 表单提交用
    
    // 触发全局重算
    this.onCalculateTotal();
}
```

**即时效果**：
1. UI 上该行消失
2. `this.totals` 重新计算（sub_total / taxes / total 等实时更新）
3. **但数据库尚未改变**，只是前端内存中的变化

#### 7.2.2 阶段二：后端真正删除（提交保存后）

用户点击「保存」后，后端通过**全删全插**策略实现删除：

```
UpdateDocument Job
    │
    ├─ deleteRelationships(model, ['items', 'item_taxes', 'totals'], true)
    │   └─ 删除数据库中该单据的所有 DocumentItem（包括未删除的）
    │
    └─ CreateDocumentItemsAndTotals
        └─ 根据前端提交的 form.items（已不含被删除行）
           └─ 重新创建所有 DocumentItem
```

**结果**：未出现在 `form.items` 中的行项目，在新的 DocumentItem 表中就不存在了，相当于被删除。

### 7.3 DocumentTotal 刷新机制详解

`DocumentTotal` 记录不是增量更新，而是**每次保存时全部删除并重新创建**。

#### 7.3.1 创建顺序与 sort_order

**文件：** [CreateDocumentItemsAndTotals.php:29-157](file:///d:/fz/0601-2/solo-dogfeeding/code/61-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L157)

DocumentTotal 按固定顺序创建，`sort_order` 从 1 开始递增：

| sort_order | code | 说明 | 计算来源 |
|-----------|------|------|---------|
| 1 | `sub_total` | 小计 | Σ(price × quantity) 所有行 |
| 2 | `item_discount` | 行折扣汇总 | Σ 各行行折扣金额（>0 才创建） |
| 3 | `discount` | 全局折扣 | 全局折扣（有 discount 才创建） |
| 4...N-2 | `tax` | 按税种分组的税额 | 每个税种一条，按税种聚合 |
| N-1 | `extra` | 额外费用（运费等） | `$request['totals']` 中的自定义项 |
| N | `total` | 最终总计 | 以上各项加减后的结果 |

#### 7.3.2 金额累加逻辑（$this->request['amount']）

```php
// 初始化
$this->request['amount'] = 0;

// 1. 加上所有行的折后金额（不含税）
$this->request['amount'] += $actual_total;

// 2. 加上所有税种金额（withholding 为负）
foreach ($taxes as $tax) {
    $this->request['amount'] += $tax['amount'];
}

// 3. 加上额外费用（按 operator 判断加减）
if (empty($total['operator']) || $total['operator'] == 'addition') {
    $this->request['amount'] += $total['amount'];
} else {
    $this->request['amount'] -= $total['amount'];
}

// 4. 最终结果写入 DocumentTotal (code=total)
$this->request['amount'] = round($this->request['amount'], $precision);
```

#### 7.3.3 DocumentTotal 与 Document.amount 的关系

```
CreateDocumentItemsAndTotals 执行中：
    ├─ 创建所有 DocumentTotal 记录
    ├─ $this->request['amount'] = 最终 total 值
    │
UpdateDocument / CreateDocument 继续执行：
    └─ $this->model->update($this->request->all())
        └─ Document.amount 被更新为 $this->request['amount']
```

> **结论**：`Document.amount` 等于 `DocumentTotal` 表中 `code='total'` 的那条记录的 `amount` 字段。

### 7.4 编辑保存完整时序图

```
[前端] 用户修改行项目（数量/价格/税/折扣，或增删行）
    │
    ├─ 触发 onCalculateTotal()
    │   └─ this.totals / form.items 实时更新
    │
    ▼
[前端] 用户点击「保存」按钮
    │
    ├─ onSubmit() → form.submit()
    │
    ▼
[前端] AJAX 提交整张表单 (PATCH /invoices/{id})
    └─ form.items 数组（已增删改后的完整数据）
    │
    ▼
[后端] FormRequest 验证
    └─ calculation_to_quantity() 转换数量表达式
    │
    ▼
[后端] Invoices::update()
    └─ dispatch(new UpdateDocument($invoice, $request))
    │
    ▼
[后端] UpdateDocument Job (事务内)
    │
    ├─ 1. deleteRelationships(['items', 'item_taxes', 'totals'], true)
    │   ├─ DELETE FROM document_items WHERE document_id = ?
    │   ├─ DELETE FROM document_item_taxes WHERE document_id = ?
    │   └─ DELETE FROM document_totals WHERE document_id = ?
    │
    ├─ 2. dispatch(new CreateDocumentItemsAndTotals)
    │   │
    │   ├─ createItems()
    │   │   └─ 遍历 form.items，为每行调用 CreateDocumentItem
    │   │       ├─ 计算金额/折扣/税
    │   │       ├─ INSERT INTO document_items
    │   │       └─ INSERT INTO document_item_taxes
    │   │
    │   └─ 按 sort_order 创建 DocumentTotal
    │       ├─ INSERT INTO document_totals (code='sub_total')
    │       ├─ INSERT INTO document_totals (code='item_discount')
    │       ├─ INSERT INTO document_totals (code='discount')
    │       ├─ INSERT INTO document_totals (code='tax', ...) × N
    │       └─ INSERT INTO document_totals (code='total')
    │
    └─ 3. UPDATE documents SET amount = ?, ... WHERE id = ?
    │
    ▼
[完成] 事务提交，返回 JSON 响应
    └─ 前端跳转到 invoices.show 页面
```

---

## 八、关键代码引用速查

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

## 九、计算示例（端到端）

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
