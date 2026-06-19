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

## 九、前后端一致性

前端（JavaScript）和后端（PHP）的计算逻辑保持一致：

1. 计算顺序相同：行金额 → 行折扣 → 全局折扣 → 税（按类型顺序）
2. 税率处理顺序相同：inclusive → fixed → normal → withholding → compound
3. 舍入策略相同：按货币精度四舍五入

前端实时计算用于用户界面展示，后端重新计算确保数据准确性。
