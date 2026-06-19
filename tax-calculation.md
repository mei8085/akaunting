# Akaunting 税额与金额计算分析

## 概述

本文档详细分析 Akaunting 财务系统中税率、折扣、舍入和单据总额之间的计算顺序和逻辑。

## 核心代码文件

| 模块 | 文件路径 | 说明 |
|------|----------|------|
| 后端计算核心 | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | 单据项目和总额计算主流程 |
| 行项目计算 | [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItem.php) | 单行项目金额、折扣、税计算 |
| 前端计算 | [documents.js](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/resources/assets/js/views/common/documents.js) | 前端实时计算逻辑 |
| 金额舍入 | [money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/resources/assets/js/plugins/money.js) | 前端金额舍入实现 |
| 税率模型 | [Tax.php](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Models/Setting/Tax.php) | 税率类型定义 |
| 货币工具 | [Currencies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Traits/Currencies.php) | 货币转换与金额处理 |

---

## 一、税率类型

系统支持 5 种税率类型，计算顺序按类型严格区分：

| 类型 | 说明 | 计算公式 |
|------|------|----------|
| **inclusive**（含税） | 价格已包含税额，需反向计算税额 | `税额 = 折扣后金额 - (折扣后金额 / (1 + 税率/100))` |
| **fixed**（固定税） | 按数量计算固定税额 | `税额 = 税率 × 数量` |
| **normal**（普通税） | 在不含税价格基础上计算 | `税额 = 计税基数 × (税率/100)` |
| **withholding**（预扣税） | 从总额中扣除的税（负值） | `税额 = -(计税基数 × (税率/100))` |
| **compound**（复合税） | 在已含税金额基础上再计税 | `税额 = 当前累计总额 × (税率/100)` |

**税率处理顺序**（`CreateDocumentItem.php#L82-L155`）：
```
inclusive → fixed → normal → withholding → compound
```

---

## 二、折扣类型

### 2.1 折扣层级

系统支持两级折扣：

| 层级 | 类型 | 说明 | 代码位置 |
|------|------|------|----------|
| 行折扣 | 百分比/固定 | 针对单行商品的折扣 | `CreateDocumentItem.php#L39-L45` |
| 全局折扣 | 百分比/固定 | 针对整张单据的折扣 | `CreateDocumentItem.php#L48-L56` |

### 2.2 全局固定折扣分摊

当全局折扣为固定金额时，需按行金额比例分摊到各行：

```php
// CreateDocumentItemsAndTotals.php#L265-L289
public function fixedDiscountCalculate()
{
    foreach ($items as $item) {
        $sub = $item['price'] * $item['quantity'];
        // 先扣除行折扣
        if ($item['discount']) {
            $sub -= 行折扣金额;
        }
        $total += $sub;
    }
    // 按比例分摊
    $item_global_discount = (行金额 / 总金额) × 全局折扣;
}
```

---

## 三、完整计算顺序

### 3.1 单行项目计算流程（后端）

**代码位置**：[CreateDocumentItem.php#L34-L156](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItem.php#L34-L156)

```
1. 计算行原始金额
   └─ 行金额 = 单价 × 数量
      （数量支持数学表达式解析，如 "2*3" → 6）

2. 应用行折扣
   ├─ 百分比折扣：折扣后金额 = 行金额 × (1 - 折扣率/100)
   └─ 固定折扣：折扣后金额 = 行金额 - 折扣额

3. 应用全局折扣
   ├─ 百分比折扣：折扣后金额 = 折扣后金额 × (1 - 全局折扣率/100)
   └─ 固定折扣：折扣后金额 = 折扣后金额 - 分摊的全局折扣额

4. 计算税额（按税率类型顺序）
   ├─ 4.1 含税税（inclusive）
   │    ├─ 从折扣后金额中反向分离税额
   │    └─ 实际价格 = 折扣后金额 - 含税税额
   │
   ├─ 4.2 固定税（fixed）
   │    └─ 税额 = 税率 × 数量
   │
   ├─ 4.3 普通税（normal）
   │    └─ 税额 = 实际价格(或折扣后金额) × 税率
   │
   ├─ 4.4 预扣税（withholding）
   │    └─ 税额 = -(实际价格 × 税率)  [负值]
   │
   └─ 4.5 复合税（compound）
        └─ 税额 = (当前累计金额) × 税率

5. 行最终金额
   └─ 行总额 = 实际价格 + 各类税额合计
```

### 3.2 单据总额计算流程（后端）

**代码位置**：[CreateDocumentItemsAndTotals.php#L29-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L158)

```
1. 遍历所有行项目，累加得到：
   ├─ sub_total（小计）= Σ(各行原始金额)
   ├─ actual_total（折扣后合计）= Σ(各行折扣后金额)
   ├─ discount_amount_total（行折扣合计）= Σ(各行折扣额)
   └─ taxes（各税种合计）= 按税号分组的税额累计

2. 写入单据合计明细（document_totals 表）
   ├─ [1] sub_total：小计（舍入后）
   ├─ [2] item_discount：行折扣合计（如存在）
   ├─ [3] discount：全局折扣（如存在）
   ├─ [4] tax：各税种分别记录
   ├─ [5] extra：额外费用（如运费）
   └─ [6] total：最终应付总额

3. 最终总额计算
   └─ amount = actual_total + Σ税额 + Σ额外费用 - 全局折扣
```

### 3.3 前端计算流程（JavaScript）

**代码位置**：[documents.js#L310-L393](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/resources/assets/js/views/common/documents.js#L310-L393)

```javascript
onCalculateTotal() {
    1. calculateTotalBeforeDiscountAndTax()
       └─ 计算每行折扣后金额

    2. 逐行处理：
       ├─ item.total = 单价 × 数量
       ├─ 应用行折扣得到 item_discounted_total
       ├─ 应用全局折扣
       ├─ calculateItemTax()  // 计算各类税额
       └─ item.grand_total = 折扣后金额 + 税额

    3. 累加得到：
       ├─ sub_total = Σ(item.total)
       ├─ grand_total = Σ(item.grand_total)
       └─ 各税项合计
}
```

---

## 四、舍入策略

### 4.1 舍入精度

由货币精度（`currency.precision`）决定，通常为 2 位小数。

```php
// CreateDocumentItem.php#L32
$precision = currency($this->document->currency_code)->getPrecision();
```

### 4.2 舍入时机

| 操作 | 舍入时机 | 代码位置 |
|------|----------|----------|
| 单价存储 | 创建行项目时 | `CreateDocumentItem.php#L172` |
| 税额计算 | 每个税项计算后 | `CreateDocumentItem.php#L193` |
| 行总额 | 行项目计算完成 | `CreateDocumentItem.php#L176` |
| 单据合计项 | 写入 document_totals 时 | `CreateDocumentItemsAndTotals.php#L44` |
| 最终总额 | 最终计算完成 | `CreateDocumentItemsAndTotals.php#L144` |

### 4.3 舍入算法

**前端舍入**（`money.js#L118-L123`）：
```javascript
round(amount, mode) {
    let precision = this.currency.getPrecision();
    var amount_sign = amount >= 0 ? 1 : -1;
    
    // 加 0.0001 修正浮点精度问题，使用 Math.round 四舍五入
    return parseFloat(
        (Math.round(
            (amount * Math.pow(10, precision)) + (amount_sign * 0.0001)
        ) / Math.pow(10, precision)).toFixed(precision)
    );
}
```

**后端舍入**（PHP 内置 `round` 函数）：
```php
round($value, $precision);  // 四舍五入到指定小数位
```

---

## 五、含税 vs 不含税金额

### 5.1 含税金额（inclusive 税处理）

当税类型为 `inclusive` 时，输入的单价已包含税额：

```php
// CreateDocumentItem.php#L83-L96
$tax_amount = $item_discounted_amount - ($item_discounted_amount / (1 + $rate / 100));
$actual_price_item = $item_discounted_amount - $item_tax_total;  // 不含税价格
```

### 5.2 获取不含税总额

**代码位置**：[Document.php#L501-L516](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Models/Document/Document.php#L501-L516)

```php
public function getAmountWithoutTaxAttribute()
{
    $amount = $this->amount;
    
    // 从总额中扣除各税种金额（预扣税除外）
    $this->totals->where('code', 'tax')->each(function ($total) use(&$amount) {
        $tax = Tax::name($total->name)->first();
        
        if (!empty($tax) && ($tax->type == 'withholding')) {
            return;  // 预扣税不从不含税金额中扣除
        }
        
        $amount -= $total->amount;
    });
    
    return $amount;
}
```

---

## 六、计算示例

### 场景

- 商品 A：单价 ¥100，数量 2，行折扣 10%
- 商品 B：单价 ¥200，数量 1，无行折扣
- 全局折扣：5%（百分比）
- 税率：普通税 13%

### 计算过程

```
商品 A 原始金额 = 100 × 2 = 200
商品 A 行折扣 = 200 × 10% = 20
商品 A 折扣后 = 200 - 20 = 180

商品 B 原始金额 = 200 × 1 = 200
商品 B 折扣后 = 200

小计 sub_total = 200 + 200 = 400

全局折扣 = (180 + 200) × 5% = 19
商品 A 全局折扣后 = 180 - (180 × 5%) = 171
商品 B 全局折扣后 = 200 - (200 × 5%) = 190

商品 A 税额 = 171 × 13% = 22.23
商品 B 税额 = 190 × 13% = 24.70
税额合计 = 46.93

商品 A 行总额 = 171 + 22.23 = 193.23
商品 B 行总额 = 190 + 24.70 = 214.70

最终总额 = 193.23 + 214.70 = 407.93
或 = (400 - 20 - 19) + 46.93 = 407.93
```

---

## 七、常见混淆点与注意事项

### 7.1 全局折扣计算基数

**注意**：全局折扣计算基数是 `sub_total`（折扣前小计），不是 `sub_total - 行折扣`。

```php
// CreateDocumentItemsAndTotals.php#L74
$discount_total = $sub_total * ($this->request['discount'] / 100);
```

### 7.2 预扣税的特殊性

- 预扣税税额为负值，从总额中扣除
- 计算不含税金额时，预扣税不参与扣除

### 7.3 复合税的计税基数

复合税基于**已包含其他税额**的金额计算，而非原始价格。

### 7.4 舍入误差累积

由于每一步都进行舍入，可能产生微小误差。系统通过在每一步舍入来减少累积误差。

### 7.5 数量表达式解析

数量字段支持数学表达式（如 "2*3"），通过 `eval()` 计算：

```php
// helpers.php#L455-L482
function calculation_to_quantity($quantity)
{
    // 安全校验后执行表达式
    $result = eval('return ' . $quantity . ';');
}
```

---

## 八、关键数据结构

### document_totals 表结构

| code | name | 说明 |
|------|------|------|
| `sub_total` | 小计 | 所有行项目原价合计 |
| `item_discount` | 行折扣 | 所有行项目折扣合计 |
| `discount` | 折扣 | 全局折扣金额 |
| `tax` | 税种名称 | 各税种分别记录 |
| `extra` | 额外费用 | 运费等其他费用 |
| `total` | 总额 | 最终应付金额 |

---

## 九、三部分计算逻辑差异对比（前端预览 vs 后端保存 vs 税额计算）

> **核心问题**：全局折扣的计算基数、单据总额的累加路径、税额计算时的折扣扣除时机在三部分代码中存在不一致。

### 9.1 三部分代码位置与职责

| 部分 | 代码文件 | 核心方法 | 职责 |
|------|----------|----------|------|
| 前端预览 | [documents.js](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/resources/assets/js/views/common/documents.js) | `onCalculateTotal()` + `calculateItemTax()` | 用户界面实时预览计算 |
| 后端保存 | [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | `handle()` + `createItems()` | 计算单据总额、写入 document_totals 表 |
| 行项目/税额计算 | [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItem.php) | `handle()` | 计算单行金额、各类税额、写入 document_items/document_item_taxes 表 |

---

### 9.2 差异总览表

| 对比维度 | 前端预览（documents.js） | 后端保存（CreateDocumentItemsAndTotals） | 税额计算（CreateDocumentItem） |
|----------|--------------------------|-------------------------------------------|--------------------------------|
| **全局百分比折扣基数** | `item_discounted_total`（行折扣后金额） | `sub_total`（未扣行折扣的原始小计） | `item_discounted_amount`（行折扣后金额）|
| **全局固定折扣分摊基数** | `items_amount[index]`（行折扣后） | `for_fixed_discount[]`（先扣行折扣） | 直接接受已分摊的固定金额 |
| **行折扣金额计算基数** | `item.total = price × quantity` | `$document_item->total`（见注1） | `$item_amount = price × quantity` |
| **税计算时是否扣全局折扣** | 是（在 `grand_total = 折扣后金额` 上计税） | 不直接处理，委派给 CreateDocumentItem | **是**（先扣全局折扣再计税，但事后又加回，见注2） |
| **sub_total 含义** | `Σ(price × quantity)` | `Σ($document_item->total)`（含行折扣影响） | 不计算 |
| **最终总额累加路径** | `grand_total = Σ(item.grand_total)`（每行含税后累加） | `actual_total + Σtaxes + Σextra - 全局折扣`（分步汇总） | 不计算 |

> **注1**：`$document_item->total` 的值在 CreateDocumentItem 中经历了「扣行折扣 → 扣全局折扣 → 计税 → 加回全局折扣」的复杂过程，最终等于「行折扣后金额」。
>
> **注2**：CreateDocumentItem L158-L162 存在「计算完税额后加回 global_discount」的逻辑，导致税额是基于「扣了全局折扣的金额」计算，但存储的 `total` 字段又回到「只扣行折扣未扣全局折扣」的状态。

---

### 9.3 差异一：全局百分比折扣的计算基数不一致

#### 前端预览（documents.js L328-L335）
```javascript
// 先调用 calculateTotalBeforeDiscountAndTax() 得到 items_amount
// items_amount[index] = price × quantity - 行折扣 = 行折扣后金额
let item_discounted_total = items_amount[index];

if (this.form.discount_type === 'percentage') {
    // 基数 = 行折扣后金额
    total_discount += (item_discounted_total / 100) * global_discount;
    item_discounted_total -= (item_discounted_total / 100) * global_discount;
}
```

#### 后端保存（CreateDocumentItemsAndTotals.php L71-L74）
```php
if (! empty($this->request['discount'])) {
    if ($this->request['discount_type'] === 'percentage') {
        // 被注释掉的旧代码才是"扣行折扣"：($sub_total - $discount_amount_total) * ...
        // 基数 = sub_total（原始小计，未扣行折扣）
        $discount_total = $sub_total * ($this->request['discount'] / 100);
    }
}
```

#### 税额计算内部（CreateDocumentItem.php L48-L56）
```php
if (! empty($this->request['global_discount'])) {
    if ($this->request['global_discount_type'] === 'percentage') {
        // 基数 = 行折扣后金额（与前端一致）
        $global_discount = $item_discounted_amount * ($this->request['global_discount'] / 100);
    }
    $item_discounted_amount -= $global_discount;
}
```

**矛盾点**：
- 前端预览与 CreateDocumentItem 税额计算一致：全局百分比折扣基于「行折扣后金额」
- 后端 document_totals 表记录的 discount 字段：基于「原始小计 sub_total」
- 这意味着 `document_totals.code='discount'` 的金额与前端展示的全局折扣额**不一致**

---

### 9.4 差异二：全局折扣的「双重扣除」机制

CreateDocumentItem 中存在特殊的「先扣后加」操作（L158-L162）：

```php
// 步骤 A：计算税额前扣除全局折扣 → 税额按"折扣后更小的金额"计算
$item_discounted_amount -= $global_discount;   // L55
$actual_price_item = $item_amount = $item_discounted_amount;  // L60
// ... 计算各类税额（基于扣了全局折扣的 actual_price_item）...

// 步骤 B：税额计算完后，把全局折扣加回（关键！）
if (! empty($global_discount)) {
    $actual_price_item += $global_discount;   // L159
    $item_amount += $global_discount;         // L160
    $item_discounted_amount += $global_discount;  // L161
}

// 步骤 C：存储的 total = 只扣行折扣、未扣全局折扣的金额
$this->request['total'] = round($actual_price_item, $precision);  // L176
```

然后在 CreateDocumentItemsAndTotals 的 createItems() 中进行第二次扣除（L253-L260）：

```php
// 汇总时的 actual_total = Σ($document_item->total) = Σ(未扣全局折扣的金额)
$actual_total += $document_item->total;  // L232

// ...循环结束后整体再扣一次全局折扣：
if (! empty($this->request['discount'])) {
    if ($this->request['discount_type'] === 'percentage') {
        $actual_total -= ($sub_total * ($this->request['discount'] / 100));  // 基数不一致
    } else {
        $actual_total -= $this->request['discount'];
    }
}
```

**计算路径可视化**：
```
单价 × 数量
    │
    ├─ 扣除行折扣 → 行折扣后金额
    │       │
    │       ├─ [CreateDocumentItem 内部]
    │       │    ├─ 扣除全局折扣 ←─ 仅用于计税！
    │       │    │     └─ 以此为基数计算 inclusive/fixed/normal/withholding/compound 税
    │       │    └─ 加回全局折扣 ←─ 税额计算完毕后还原
    │       │
    │       └─ 存储 total = 行折扣后金额（未扣全局折扣）
    │
    └─ [CreateDocumentItemsAndTotals 汇总层]
            ├─ Σtotal（各行未扣全局折扣）
            ├─ Σ税额（基于扣了全局折扣的基数计算得出）
            └─ 整体扣除一次全局折扣（此时基数可能不一致）
```

---

### 9.5 差异三：行折扣计算的基数不一致

#### 前端预览（documents.js L532-L567）
```javascript
calculateTotalBeforeDiscountAndTax() {
    item_total = item.price * calculationToQuantity(item.quantity);
    if (item.discount_type === 'percentage') {
        item.discount_amount = item_total * (item.discount / 100);  // 基数 = price×qty
    }
    amount_before_discount_and_tax[index] = item_total - item.discount_amount;
}
// 行折扣额 = item_total - item_discounted_total（L325）
```

#### 后端保存（CreateDocumentItemsAndTotals.php L220-L228）
```php
# $item_amount 不是 price × quantity，而是 $document_item->total（L218）
$item_amount = $document_item->total;  // = 行折扣后金额（见上文的"加回"机制）

if (! empty($item['discount'])) {
    if ($item['discount_type'] === 'percentage') {
        // 基数 = $document_item->total（此时已是行折扣后金额，与前端不同）
        $discount_amount = ($item_amount * ($item['discount'] / 100));
    }
}
```

**注意**：此处后端计算 `$discount_amount_total`（行折扣合计）的基数与前端不同，但该值仅用于在 `document_totals` 中显示，不影响最终总额的计算。

---

### 9.6 差异四：sub_total 的含义与前端不一致

| 部分 | sub_total 计算方式 | 实际含义 |
|------|--------------------|----------|
| 前端预览 L353 | `Σ(item.price * item.quantity)` | 所有行项目的**原价合计**（未扣任何折扣） |
| 后端保存 L231 | `Σ($document_item->total)` | 所有行项目的**行折扣后金额合计**（已扣行折扣、未扣全局折扣） |

这导致：
- 前端展示的「小计」（sub_total）= 所有商品原价相加
- 后端存储的 `document_totals.code='sub_total'` = 扣完行折扣的金额
- 两者展示在界面上可能对不上

---

### 9.7 含税税（inclusive）处理中的传递差异

#### 前端预览（documents.js L526-L528）
```javascript
if (inclusives.length) {
    // inclusive 情况下，把 total_discount + line_discount_amount 加回 item.total
    // total_discount = 累计到当前行的全局折扣
    item.total += total_discount_amount;
}
```
该逻辑将「全局折扣 + 行折扣」加回到 `item.total`，目的是让 inclusive 的含税基准恢复到折扣前的水平（但这是**累计值**，逐行传递，第一行不受后续行影响）。

#### 后端（CreateDocumentItem.php L83-L96）
```php
// 基于 item_discounted_amount 计算（已扣行折扣、已扣全局折扣）
$tax_amount = $item_discounted_amount - ($item_discounted_amount / (1 + $inclusive->rate / 100));
$actual_price_item = $item_discounted_amount - $item_tax_total;
```
后端直接基于「行折扣+全局折扣后的金额」分离含税税额，不进行折扣回加操作。

---

### 9.8 示例：数值差异验证

假设场景：
- 商品 A：单价 100，数量 2，行折扣 10%（百分比）
- 全局折扣：5%（百分比）
- 税率：普通税 13%

| 项目 | 前端预览计算 | 后端存储（document_totals） | 差异说明 |
|------|-------------|---------------------------|----------|
| **行折扣额** | 200 × 10% = 20 | 基数是 `$document_item->total` | 行折扣合计的统计值不同，但不计入总额 |
| **全局折扣基数** | 180（行折扣后） | 200（sub_total，未扣行折扣） | 基数差 20 |
| **全局折扣额** | 180 × 5% = 9 | 200 × 5% = 10 | **document_totals.discount 比前端多 1** |
| **税基数（每行）** | 180 - 9 = 171 | 180 - 9 = 171（内部扣了全局折扣） | 税额计算基数一致 |
| **商品A税额** | 171 × 13% = 22.23 | 171 × 13% = 22.23 | 税额一致 |
| **最终总额（公式）** | 171 + 22.23 = 193.23 | actual_total(180) - 10 + 22.23 = 192.23 | **总额差 1** |

> ⚠️ 上述差异表明：当同时存在「行折扣」和「全局百分比折扣」时，由于后端 `document_totals.discount` 采用 `sub_total` 为基数（而 CreateDocumentItem 内部扣减和前端以「行折扣后」为基数），最终单据总额在三部分之间可能不一致。

---

### 9.9 一致性保证部分

以下维度在三部分中**保持一致**：

1. **税率处理顺序**：inclusive → fixed → normal → withholding → compound
2. **税额计算公式**：每种税类型的公式在前后端相同
3. **复合税（compound）基数**：均为「已含其他税的当前累计额」
4. **预扣税（withholding）符号**：均为负值从总额扣除
5. **固定税（fixed）计算**：均为 `rate × quantity`
6. **舍入精度**：均以 `currency.precision` 为小数位进行四舍五入
7. **全局固定折扣分摊**：均以「行折扣后金额」为权重按比例分配

---

### 9.10 相关 Issue 与代码注释

在 CreateDocumentItemsAndTotals.php 中可以看到与此差异直接相关的注释：

- L73：注释掉的旧代码 `($sub_total - $discount_amount_total) * ...` 是前端一致的计算方式，当前启用代码改成了 `$sub_total * ...`
- L177-L178：`# This line changed for discount calculator issue` 注释表明 `$item_amount` 从 `price × quantity` 改为 `$document_item->total` 是为了修复折扣问题
- L253：`Disable this lines for global discount issue fixed (https://github.com/akaunting/akaunting/issues/2797)` 表明这是针对 GitHub issue #2797 的修复，但修复引入了新的基数不一致

---

## 十、总结与使用建议

### 含税与不含税计算时的注意事项

1. **以哪个为准**：后端保存的计算结果（写入数据库的）是最终可信值，但需注意 `document_totals` 中的 `sub_total` 和 `discount` 字段含义与前端展示不同。

2. **调试时的坑点**：
   - 不要用前端的 `totals.sub` 与数据库 `document_totals.code='sub_total'` 直接对比（含义不同）
   - 不要用前端的全局折扣额与 `document_totals.code='discount'` 对比（基数不同）
   - 排查税额差异时，重点检查 CreateDocumentItem.php L158-L162 的「加回全局折扣」逻辑

3. **修改建议**：若需要统一全局折扣基数，需同时修改以下三处：
   - 后端 CreateDocumentItemsAndTotals.php L74（discount_total 计算基数）
   - 后端 CreateDocumentItemsAndTotals.php L256（actual_total 扣除时的基数）
   - 前端 documents.js 无需修改（与税额计算内部一致）

4. **审计思路**：验证单据总额正确性时，使用以下公式核对：
   ```
   最终应付 = Σ(price × qty) - Σ行折扣 - Σ全局折扣（正确基数） + Σ税费 + Σ额外费用
   不要直接依赖 document_totals 的各字段简单相加。
   ```

---

## 十一、代码逐行追踪：行折扣、百分比全局折扣与最终汇总扣减的完整关联

> 本章用具体数值逐行追踪三段核心代码的变量变化，彻底讲清三者之间的关联与分歧。

### 11.1 测试场景

| 参数 | 商品 A | 商品 B |
|------|--------|--------|
| 单价 price | 100 | 200 |
| 数量 quantity | 2 | 1 |
| 行折扣 discount | 10（百分比） | 0（无） |
| 行折扣类型 discount_type | percentage | - |
| 全局折扣 | 5（百分比） | - |
| 税率 | 13%（normal 普通税） | 13%（normal 普通税） |

---

### 11.2 第一部分：CreateDocumentItem 逐行追踪（商品 A）

**代码位置**：[CreateDocumentItem.php#L29-L202](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItem.php#L29-L202)

调用参数：`price=100, quantity=2, discount=10, discount_type='percentage', global_discount=5, global_discount_type='percentage'`

| 行号 | 代码 | 变量变化 | 数值结果 |
|------|------|----------|----------|
| L34 | `$item_amount = price × qty` | `$item_amount = 100 × 2` | **200** |
| L36 | `$item_discounted_amount = $item_amount` | `$item_discounted_amount` 初始化 | **200** |
| L39-L45 | 应用**行折扣**（10%） | `$item_discounted_amount -= 200 × 10%` | **180**（行折扣后） |
| L48-L56 | 应用**全局折扣**（5%） | `$global_discount = 180 × 5% = 9`，`$item_discounted_amount -= 9` | **171**（双折扣后，仅用于计税） |
| L60 | 三变量同步赋值 | `$actual_price_item = $item_amount = 171` | **171** |
| L114-L126 | 计算**普通税**（13%） | `$tax_amount = 171 × 13%`，`$item_tax_total = 22.23`，`$item_amount += 22.23` | 税=**22.23**，$item_amount=**193.23** |
| **L158-L162** | **加回全局折扣**（关键！） | `$actual_price_item += 9`，`$item_amount += 9`，`$item_discounted_amount += 9` | `$actual_price_item`=**180**，$item_amount=**202.23** |
| L173 | 存储税额 | `$this->request['tax'] = round(22.23, 2)` | **22.23** |
| **L176** | **存储 total 字段** | `$this->request['total'] = round($actual_price_item, 2)` | **180** |

**结论**：`$document_item->total = 180` = **行折扣后金额（未扣全局折扣）**。全局折扣仅在计税瞬间扣除，税额计算完毕后立即加回。

---

### 11.3 第一部分续：CreateDocumentItem 逐行追踪（商品 B）

调用参数：`price=200, quantity=1, discount=0, global_discount=5, global_discount_type='percentage'`

| 行号 | 代码 | 变量变化 | 数值结果 |
|------|------|----------|----------|
| L34 | `$item_amount = 200 × 1` | `$item_amount` | **200** |
| L36 | `$item_discounted_amount = $item_amount` | 初始化 | **200** |
| L39-L45 | 无行折扣 | 不变 | **200** |
| L48-L56 | 应用全局折扣（5%） | `$global_discount = 200 × 5% = 10`，`$item_discounted_amount -= 10` | **190**（仅用于计税） |
| L60 | 三变量同步 | `$actual_price_item = $item_amount = 190` | **190** |
| L114-L126 | 普通税 13% | `$tax_amount = 190 × 13% = 24.70`，`$item_tax_total = 24.70`，`$item_amount += 24.70` | 税=**24.70**，$item_amount=**214.70** |
| L158-L162 | 加回全局折扣 | `$actual_price_item += 10` | **200** |
| L176 | 存储 total | `$this->request['total'] = round(200, 2)` | **200** |

**结论**：商品 B `$document_item->total = 200` = 原价（无行折扣、无全局折扣）。

---

### 11.4 第二部分：CreateDocumentItemsAndTotals → createItems() 逐行追踪

**代码位置**：[CreateDocumentItemsAndTotals.php#L160-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L160-L263)

#### 循环处理商品 A

| 行号 | 代码 | 变量变化 | 数值结果 |
|------|------|----------|----------|
| L178-L185 | 传递全局折扣（5%） | `$item['global_discount'] = 5`，type='percentage' | 直接透传百分比数值 |
| L214 | 调用 CreateDocumentItem | 返回 `$document_item`，其 `total = 180` | - |
| **L218** | **$item_amount 赋值** | `$item_amount = $document_item->total`（不是 `price × qty`！注释标注了这是修复改动） | **180** |
| L222-L228 | **重算行折扣** | `$discount_amount = $item_amount(180) × 10%` | **18**（注意！前端这里是 200×10%=20，基数已不同） |
| **L231** | **累加 sub_total** | `$sub_total += $item_amount(180)` | `$sub_total` = **180** |
| **L232** | **累加 actual_total** | `$actual_total += $document_item->total(180)` | `$actual_total` = **180** |
| L234 | 累加行折扣合计 | `$discount_amount_total += 18` | **18** |

#### 循环处理商品 B

| 行号 | 代码 | 变量变化 | 数值结果 |
|------|------|----------|----------|
| L218 | $item_amount = $document_item->total | - | **200** |
| L222-L228 | 无行折扣 | $discount_amount = 0 | **0** |
| L231 | $sub_total += 200 | `$sub_total = 180 + 200` | **380** |
| L232 | $actual_total += 200 | `$actual_total = 180 + 200` | **380** |
| L234 | $discount_amount_total += 0 | 行折扣合计 | **18** |

#### 循环结束后（L253-L260）

| 行号 | 代码 | 变量变化 | 数值结果 |
|------|------|----------|----------|
| **L254-L256** | **整体扣减全局折扣** | `$actual_total -= ($sub_total(380) × 5%)` = 380 - 19 | `$actual_total` = **361** |

#### 返回值

```php
return [
    $sub_total             = 380,      // 注意：不是 Σ(price×qty)=400，而是 Σ(行折扣后 total)
    $actual_total          = 361,      // 已扣行折扣+全局折扣的净额
    $discount_amount_total = 18,       // 行折扣合计（基数不同导致与前端 20 不同）
    $taxes                 = [税合计 46.93]
];
```

**关键关联**：此处 `$sub_total` 的含义已被 L218 的修改改变了。原本应该是 `Σ(price × qty) = 400`，但现在实际等于 `Σ($document_item->total) = Σ(行折扣后金额) = 380`。这就是「小计说明不一致」的根源。

---

### 11.5 第二部分续：CreateDocumentItemsAndTotals → handle() 逐行追踪

**代码位置**：[CreateDocumentItemsAndTotals.php#L29-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L158)

| 行号 | 代码 | 变量变化 / 入库 | 数值结果 |
|------|------|-----------------|----------|
| L33 | 接收 createItems 返回值 | `$sub_total=380, $actual_total=361, $discount_amount_total=18, $taxes=46.93` | - |
| L38-L48 | 写入 `document_totals.code='sub_total'` | amount = round(380, 2) | **380**（⚠️ 含义=行折扣后合计，非原价合计） |
| L50 | 初始化单据金额 | `$this->request['amount'] += $actual_total(361)` | amount = **361** |
| L55-L69 | 写入 `document_totals.code='item_discount'` | amount = round(18, 2) | **18**（⚠️ 基数=行折扣后，前端基数=原价） |
| **L71-L74** | **计算全局折扣展示额** | `$discount_total = $sub_total(380) × 5%`（L73 注释掉的旧代码才是 `($sub_total - $discount_amount_total) × 5%`） | **19** |
| L79-L89 | 写入 `document_totals.code='discount'` | amount = round(19, 2) | **19** |
| L95-L113 | 写入 `document_totals.code='tax'`（分税种） | `$this->request['amount'] += 22.23 + 24.70` | amount = 361 + 46.93 = **407.93** |
| L144 | 舍入最终金额 | `$this->request['amount'] = round(407.93, 2)` | **407.93** |
| L147-L157 | 写入 `document_totals.code='total'` | amount = 407.93 | **407.93** |

**三者关联公式**：
```
最终总额 amount
  = actual_total(361)           ← createItems 返回：Σ行折扣后 - 整体全局折扣
  + Σtaxes(46.93)               ← 基于扣了全局折扣的基数计算
  + extra(0)
= 407.93
```

注意：`document_totals.discount = 19` 和 `document_totals.sub_total = 380` 是用于展示的中间值，**不参与**最终 amount 的计算。最终 amount 的计算完全依赖 `actual_total` 和 `taxes`。

---

### 11.6 第三部分：前端 documents.js 逐行追踪

**代码位置**：[documents.js#L310-L393](file:///d:/fz/0601-2/solo-dogfeeding/code/48-akaunting/resources/assets/js/views/common/documents.js#L310-L393)

#### calculateTotalBeforeDiscountAndTax() 预计算（L532-L567）

| 商品 | item_total = price×qty | 行折扣额（基数=原价） | 行折扣后金额 |
|------|----------------------|---------------------|-------------|
| A | 200 | 200 × 10% = 20 | 180 |
| B | 200 | 0 | 200 |
| Σ | - | - | total = **380** |

返回 `items_amount = [0:180, 1:200, total:380]`

#### onCalculateTotal() 主计算（L310-L393）

##### 处理商品 A

| 行号/逻辑 | 计算 | 结果 |
|-----------|------|------|
| L321 | `item.total = item.grand_total = 100 × 2` | **200** |
| L323 | `item_discounted_total = items_amount[0]` | **180**（行折扣后） |
| L325 | `line_discount_amount = 200 - 180` | **20** |
| L328-L335 | 应用全局折扣 5%（基数=行折扣后 180） | `total_discount += 9`，`item_discounted_total = 171` |
| L343-L345 | `item.grand_total = 171`（双折扣后） | **171** |
| L347 | `calculateItemTax(item, ...)`：普通税 13%，基数=171 | 税 = **22.23**，`item.grand_total += 22.23 = 193.23` |
| L349 | `item.total = 100 × 2`（**还原为原价**） | **200** |
| L352 | `line_item_discount_total += 20` | **20** |
| L353 | `sub_total += 200` | **200** |
| L354 | `grand_total += 193.23` | **193.23** |

##### 处理商品 B

| 逻辑 | 计算 | 结果 |
|------|------|------|
| item.total = 200 × 1 | - | **200** |
| item_discounted_total = items_amount[1] | - | **200** |
| line_discount_amount = 200 - 200 | - | **0** |
| 全局折扣 5%（基数=200） | `total_discount += 10`，item_discounted_total = 190 | - |
| item.grand_total = 190 | - | **190** |
| 普通税 13%（基数=190） | 税 = 24.70，grand_total += 24.70 | **214.70** |
| item.total 还原 | - | **200** |
| sub_total += 200 | sub_total = 200 + 200 | **400** |
| grand_total += 214.70 | grand_total = 193.23 + 214.70 | **407.93** |

##### 前端最终结果

| 字段 | 值 |
|------|----|
| totals.sub（小计）| **400**（= Σ原价，与后端 380 不同） |
| totals.item_discount（行折扣合计）| **20**（基数=原价，与后端 18 不同） |
| totals.discount（全局折扣合计）| **19**（180×5% + 200×5%，与后端 380×5%=19 碰巧一致） |
| totals.taxes | **46.93** |
| totals.total（最终总额）| **407.93** |

---

### 11.7 三者数值对比总表

| 字段 | 前端预览 | 后端 document_totals | 后端实际参与总额计算 | 为何不同 |
|------|---------|---------------------|---------------------|----------|
| **sub_total 小计** | 400 = Σ(price×qty) | **380** = Σ(行折扣后 total) | 不参与 | L218 用 `$document_item->total` 替代了 `price×qty` |
| **item_discount 行折扣** | 20 = Σ(原价×行折扣率) | **18** = Σ(total×行折扣率) | 不参与 | 计算基数不同 |
| **discount 全局折扣** | 19 = Σ(行折扣后×5%) | **19** = sub_total×5% | 不参与 | 乘法分配律导致碰巧相等（180+200=380，380×5%=180×5%+200×5%） |
| **taxes 税额** | 46.93 | 46.93 | ✅ 参与 | 计税基数一致（扣了全局折扣的金额） |
| **total 最终总额** | 407.93 | 407.93 | ✅ | normal 税场景下碰巧一致 |

> ⚠️ **注意**：全局折扣的"碰巧一致"只适用于百分比全局折扣 + normal 税的场景。如果涉及 inclusive、withholding、compound 税，或全局折扣为固定金额，则前后端总额也可能出现差异。

---

### 11.8 关联链路全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     用户输入（前端表单）                              │
│  price, quantity, discount(行), global_discount(整单), tax_ids      │
└──────────────────────────────────────┬──────────────────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          ▼                            ▼                            ▼
┌────────────────────┐      ┌────────────────────────┐     ┌──────────────────────┐
│   前端预览计算      │      │  CreateDocumentItem    │     │ CreateDocumentItems- │
│  (documents.js)    │      │   （每行独立调用）      │     │    AndTotals 汇总层   │
├────────────────────┤      ├────────────────────────┤     ├──────────────────────┤
│ sub_total =        │      │ L34: price × qty       │     │ L231: $sub_total      │
│   Σ(price × qty)   │      │   = 原始金额           │     │   = Σ($document_item  │
│                    │      │ L39-45: 扣行折扣       │     │     ->total)          │
│ 行折扣基数=原价    │──┐   │ L48-56: 扣全局折扣     │     │   = Σ(行折扣后金额)   │
│                    │  │   │   ↓ (仅用于计税！)     │     │                      │
│ 全局折扣基数=      │  │   │ L60: actual_price_item │     │ L222-228: 重算行折扣  │
│  行折扣后金额      │  │   │   = 双折扣后金额        │     │   基数 = total(已扣   │
│                    │  │   │ L82-155: 按类型算税    │     │   行折扣)             │
│                    │  │   │   (基数=双折扣后)       │     │                      │
│ 税后 grand_total   │  │   │                        │     │ L74: discount_total   │
│   = 双折扣后 + 税  │  │   │ L158-162: ⭐加回全局折扣│     │   = $sub_total × 折扣率│
│                    │  │   │   (税额不受影响)        │     │   (基数=行折扣后合计) │
│                    │  │   │                        │     │                      │
│ 最终 total =       │  │   │ L176: total =          │     │ L256: $actual_total  │
│  Σ(grand_total)    │  │   │   actual_price_item    │     │   -= $sub_total × 折扣│
│                    │  │   │   = 行折扣后金额        │     │                      │
└────────────────────┘  │   └───────────┬────────────┘     └──────────┬───────────┘
                        │               │                             │
                        │               ▼                             ▼
                        │      $document_item->total       $this->request['amount']
                        │         = 行折扣后金额            = actual_total + Σtaxes
                        │               │                        = 407.93
                        │               │                             │
                        │               └──────────────┬──────────────┘
                        │                              │
                        │                              ▼
                        │                    ┌──────────────────┐
                        │                    │   最终总额对比     │
                        │                    │ 前端: 407.93      │
                        │                    │ 后端: 407.93      │
                        │                    │  (本场景一致)     │
                        │                    └──────────────────┘
                        │
                        └─► 中间展示字段（sub_total、item_discount、discount）
                              前后端含义和数值不一致，但不影响最终总额
```

---

### 11.9 为什么最终总额有时一致？

在「百分比全局折扣 + 仅 normal 税」场景下，乘法分配律保证了总额一致：

```
前端全局折扣合计 = Σ(行折扣后 × 折扣率)
后端全局折扣合计 = Σ(行折扣后) × 折扣率

两者相等（乘法分配律），税额基数也一致（都是扣了全局折扣的金额），
所以最终总额相同。
```

但以下场景可能出现不一致：
1. **inclusive 税**：前端 L526-L528 有 `item.total += total_discount_amount` 的折扣回加逻辑，后端没有对应处理
2. **固定金额全局折扣**：分摊比例在前后端可能因舍入时机不同产生尾差
3. **compound 税**：基于已含税金额的二次计税，对舍入时机极度敏感
4. **withholding 税**：负值税的处理顺序可能影响复合税基数

---

### 11.10 小结：代码理解要点

1. **`$document_item->total` 不是原价**：经过 CreateDocumentItem 处理后，它等于「行折扣后金额」，不是 `price × quantity`。这是 L218 注释标记的修复改动带来的含义变化。

2. **全局折扣的「两步走」设计**：
   - 第一步：CreateDocumentItem 内部扣全局折扣 → 仅用于计算税额
   - 第二步：税额算完后加回 → 存储的 total 不含全局折扣
   - 第三步：汇总层 L256 整体再扣一次全局折扣 → 计入最终总额

3. **document_totals 的展示值与计算值分离**：`sub_total`、`item_discount`、`discount` 存入数据库的是「展示值」，最终 `amount` 不依赖这些字段相加，而是通过 `actual_total + Σtaxes + Σextra` 独立计算。

4. **前端 L349 的还原操作**：`item.total = price × qty` 把行 total 还原为原价，目的是让 `sub_total` 展示为原价合计。但后端 L218 改成了用 `$document_item->total`（行折扣后），这就是「小计说明不一致」的直接原因。
