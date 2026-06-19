# 币种汇率与金额格式化流程梳理

## 一、汇率来源

### 1.1 配置层（基础定义）

系统中有两级货币定义：

**PHP 端配置**：
- 文件：[config/money.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/config/money.php)
- 内容：所有支持的法定货币定义，包括 name、code、precision、subunit、symbol、decimal_mark、thousands_separator 等属性
- 由 `akaunting/laravel-money` 包提供 `currency()` 辅助函数读取

**JS 端配置**：
- 文件：[public/money.json](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/public/money.json)
- 内容：包含法定货币 + 加密货币定义
- JS 端 `Currency` 类从该文件加载

### 1.2 数据库层（业务数据）

**币种表（currencies）**：
- 文件：[database/migrations/2017_09_14_000000_core_v1.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/database/migrations/2017_09_14_000000_core_v1.php#L155-L170)
- 模型：[app/Models/Setting/Currency.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Setting/Currency.php)
- 关键字段：
  - `code`：货币代码（如 USD、CNY）
  - `rate`：汇率（double, 15,8），相对于默认货币的比值
  - `precision`、`symbol`、`decimal_mark`、`thousands_separator`：可覆盖默认配置

**快照汇率**：
- `documents.currency_rate`：单据创建时的汇率快照
- `transactions.currency_rate`：交易创建时的汇率快照
- 设计意图：单据/交易的汇率不随币种表更新而变化，保持历史一致性

### 1.3 默认货币

- 辅助函数：[default_currency()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Utilities/helpers.php#L247-L255)
- 来源：`setting('default.currency')`，从系统设置中读取
- 默认货币的汇率恒为 1.0

### 1.4 汇率使用方式

通过 [Currencies trait](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Traits/Currencies.php) 进行货币转换：

```php
// 核心转换方法
convert($method, $amount, $from, $to, $rate, $format = false)

// 常用封装
convertToDefault($amount, $from, $rate, $format = false)   // 除以汇率，转为默认货币
convertFromDefault($amount, $to, $rate, $format = false)  // 乘以汇率，从默认货币转出
convertBetween($amount, $from_code, $from_rate, $to_code, $to_rate)  // 两种货币间互转
```

**转换原理**：以默认货币为中间桥梁
1. 源货币 → 默认货币：`amount / from_rate`
2. 默认货币 → 目标货币：`amount * to_rate`

---

## 二、精度处理

### 2.1 精度的三重含义

精度（precision）在系统中同时扮演三个角色：

| 角色 | 说明 | 示例 |
|------|------|------|
| 存储精度 | 数据库中金额字段的小数位数 | USD: 2位, JPY: 0位 |
| 计算精度 | 业务计算时 round 的小数位数 | 各阶段 totals 计算 |
| 显示精度 | 格式化显示时的小数位数 | 金额格式化输出 |

### 2.2 PHP 端精度处理

**存储单位**：金额以**主单位**存储（如美元的"元"，而非"分"）

**精度获取**（Currency 模型的访问器）：
- [app/Models/Setting/Currency.php#L132-L139](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Setting/Currency.php#L132-L139)
- 优先级：数据库字段值 → config/money.php 中的默认值

**计算中的精度控制**：
在 [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) 中：

```php
$precision = currency($this->document->currency_code)->getPrecision();

// 每个计算阶段都做 round
round($sub_total, $precision);
round($discount_total, $precision);
round($tax['amount'], $precision);
round($this->request['amount'], $precision);
```

### 2.3 JS 端精度处理

**存储单位**：金额以**最小单位（subunit）**存储（如美分）

- 文件：[resources/assets/js/plugins/money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/money.js)
- `getValue()`：`amount / subunit` 得到主单位
- `format()`：使用 precision 控制小数位数
- `round()`：按货币精度做四舍五入

### 2.4 常见货币精度

| 货币 | 精度 | 最小单位 | 说明 |
|------|------|----------|------|
| USD / CNY / EUR | 2 | 100 | 大多数货币 |
| JPY / KRW | 0 | 1 | 日元、韩元无小数 |
| BHD / IQD / KWD | 3 | 1000 | 部分中东货币 |

---

## 三、页面录单的汇率切换

### 3.1 入口位置

汇率切换位于单据录单页面底部的总额区域：
- 视图：[resources/views/components/documents/form/totals.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/components/documents/form/totals.blade.php#L180-L196)
- 组件：总额行（tr-total）中的 `<x-form.group.select name="currency_code">` 下拉框

### 3.2 切换事件流程

触发方法：[onChangeCurrency()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L794-L828)

```
用户选择新币种 → onChangeCurrency(currency_code)
  ↓
首次触发时：异步拉取 /settings/currencies 接口，获取公司所有币种列表
  ↓
遍历 currencies，匹配 code
  ↓
1. this.currency = 匹配到的币种对象（含 precision、symbol 等）
2. this.form.currency_code = 新币种代码
3. this.form.currency_rate = 新币种的 rate（来自 currencies 表）
4. 更新 currency_symbol（用于换算显示）
  ↓
调用 currencyConversion() → 调整输入框宽度
```

### 3.3 汇率可手动编辑

在币种切换行下方，当**选择的币种 ≠ 公司默认币种**时，显示汇率换算组件：

- 组件：[AkauntingCurrencyConversion.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/components/AkauntingCurrencyConversion.vue)
- 显示内容：
  - 左侧：换算后的默认币种金额（只读）：`(totals.total / form.currency_rate).toFixed(2)`
  - 中间：说明文本
  - 右侧：`currency_rate` 输入框，用户可手动编辑
- 事件：输入框 `@input` 触发 `onChange` → emit 到父组件 → `form.currency_rate = $event`

### 3.4 切换币种后总额的行为

**关键**：切换币种后，`totals.sub`、`totals.discount`、`totals.taxes`、`totals.total` 的**数值本身不会被立即重新换算**。它们仍然按原来币种的数值存储，只是：

1. 金额输入框的格式（符号、千分位、小数位）会按新币种的属性变化（由 `<akaunting-money>` 的 `dynamicCurrency` 机制响应）
2. 汇率组件按新的 `currency_rate` 展示换算后的默认币种金额
3. **真正按新币种做数值换算的工作在后端完成**：当用户点击保存后，后端 Job 会按 `form.currency_code` 和 `form.currency_rate` 重新处理所有金额

### 3.5 编辑状态下的限制

- 编辑单据时（`edit.status = true`），前两次币种切换**不生效**（`edit.currency` 计数器从 1 递增到 3 后才允许切换）
- 这是为了防止编辑时因 created 钩子中的初始化流程误触发币种变更

---

## 四、付款换算（跨币种付款）

### 4.1 场景说明

当用户用与**单据币种不同**的账户/币种付款时，需要处理付款金额与单据金额的换算。

涉及文件：
- 视图：[resources/views/modals/documents/payment.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/modals/documents/payment.blade.php)
- 控制器：[app/Http/Controllers/Modals/DocumentTransactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php)
- 逻辑：[resources/assets/js/views/common/documents.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js)

### 4.2 模态框中的基准字段

付款模态框在渲染时，会注入以下字段作为换算基准（关键币种修正：均以**单据币种**为基准，而非公司默认币种）：

| 字段 | 含义 | 币种 | 来源 |
|------|------|------|------|
| `document_currency_code` | 单据币种代码 | — | `$document->currency_code` |
| `document_currency_rate` | 单据创建时的汇率快照 | — | `$document->currency_rate` |
| `document_default_amount` | **剩余应付金额**（总额 − 已付款） | **单据币种** | 控制器中 `round($d_total − $paid, $currency->precision)`，`$currency` 是**单据币种对象**（[DocumentTransactions.php#L60-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php#L60-L68)） |
| `company_currency_code` | 公司默认币种代码 | — | `default_currency()` |
| `paid_amount` | 累计已付款 | 单据币种 | `$document->paid`（由 `getPaidAttribute()` 换算到单据币种） |
| `amount` | 付款输入框初始值 | 单据币种 | 同 `document_default_amount` |
| `default_amount`（回显字段） | 校验用的应付金额回显 | **单据币种** | `<x-form.input.money>` 的 `:currency="$currency"` 绑定的是单据币种对象（[payment.blade.php#L100-L110](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/modals/documents/payment.blade.php#L100-L110)） |

> **命名提醒**：字段名中的 `default` 不是"默认货币"，而是指"单据本身的（基准）金额"。

### 4.3 付款账户切换后的校验（先换算回单据币种再比较）

触发方法：[onChangeCurrencyPaymentAccount(currency_code)](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L120-L153)

当用户选择付款账户（即切换到新的付款币种 `code` = `currency_code`）时，**校验始终在单据币种层面进行**：

```
  新付款币种 == 单据币种？
    ├─ 是 → 无需换算，直接通过
    └─ 否 → Step 1：换算到单据币种做校验
              ┌────────────────────────────────────────────┐
              │  convertBetween(                           │
              │    form.amount,                            │ ← 用户输入的金额
              │    新币种code, 新rate,                      │ ← 视为：源币种 = 新付款账户
              │    单据币种code, 单据rate)                  │ ← 目标币种 = 单据币种
              │  → 得到【单据币种】的 equivalent_amount     │
              └────────────────────────────────────────────┘
                                 ↓
           Step 2：单据币种 vs 单据币种 做比较
              equivalent_amount  >  document_default_amount？
                                 ↓
           Step 3：若溢出 → 反向换算回新付款币种写入
              ┌────────────────────────────────────────────┐
              │  convertBetween(                           │
              │    document_default_amount,                │ ← 剩余应付（单据币种）
              │    单据币种code, 单据rate,                   │ ← 源币种 = 单据币种
              │    新币种code, 新rate)                       │ ← 目标币种 = 新付款账户
              │  → 得到【新付款币种】的上限值 error_amount    │
              └────────────────────────────────────────────┘
                                 ↓
           Step 4：form.amount = error_amount（截断到付款币种上限）
                    form.default_amount = document_default_amount（单据币种回显不变）
```

### 4.4 付款金额变化时的双向校验

触发方法：[onChangeAmount(amount)](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L155-L189)

用户在付款输入框输入金额后（`amount` 是付款币种数值），流程与切换账户一致：

1. **先换算回单据币种**：`convertBetween(amount, 付款币种, 付款汇率, 单据币种, 单据汇率)`
2. **单据币种内比较**：换算结果 vs `document_default_amount`
3. **写回单据币种回显**：`form.default_amount = 换算后的单据币种金额`（用户可见的左侧回显）
4. **若溢出**：反向换算回付款币种 → `error_amount` 用于截断（但当前方法中仅更新 `default_amount`，实际截断由提交前的 `checkAmount()` 处理）

### 4.5 汇率手动变更 & "全额付款"

两种方式调整汇率：

**方式一：直接编辑汇率输入框** → [onChangeRatePayment()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L191-L197)
- 更新 `form.currency_rate`
- 重新触发 `onChangeAmount` 刷新换算回显

**方式二：勾选「Pay in full」（全额付款）** → [onChangePayInFull()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L199-L213)
- 反推汇率：`rate = (document_currency_rate / document_default_amount) * form.amount`
- 更新 `form.currency_rate`
- 重新触发 `onChangeAmount` 刷新换算回显
- 此时 `currency_rate` 输入框被设为 disabled（不允许手动改）

### 4.6 后端：单据已付款金额的累计换算

属性访问器：[getPaidAttribute()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Document/Document.php#L373-L407)

```php
$paid = 0;
foreach ($this->transactions as $transaction) {
    $amount = $transaction->amount;  // 付款币种金额

    if ($单据币种 != $transaction->currency_code) {
        // convertBetween：付款币种 → 单据币种
        $amount = $this->convertBetween(
            $amount,
            $transaction->currency_code,
            $transaction->currency_rate,
            $单据币种,
            $单据->currency_rate
        );
    }
    $paid += $amount;
}
return round($paid, $单据币种精度);
```

**要点**：每笔交易都用自己创建时快照的 `currency_rate` 做换算，累计后按单据币种精度 round。

---

## 五、单据总额输出流程（税额 & 行金额拆分详解）

### 5.1 整体流程

入口任务：[CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php)

```
创建单据 → 遍历 items：
              每个 item 调用 CreateDocumentItem（完成单条 item 的折扣+税额计算）
          ↓
汇总各 item 的 sub_total、actual_total、折扣合计、按税种聚合的 taxes 数组
          ↓
按 sort_order 依次写入 document_totals：
  sub_total → item_discount → discount → tax[n] → extra[n] → total
          ↓
最终总额写入 documents.amount
```

### 5.2 单条明细行计算（CreateDocumentItem）

见 [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItem.php#L29-L202)

#### 5.2.1 变量定义与三个"金额"的区别

处理一条 item 时会同时维护三个不同的金额变量，**这是之前文档描述不准确的关键**：

| 变量 | 含义 | 用途 | 最终写入字段 |
|------|------|------|------------|
| `$item_amount` | 单价×数量，不考虑行折扣（含税/不含税随税类型动态变化） | 用于累计 sub_total 的基础 | — |
| `$item_discounted_amount` | 扣掉**行折扣**和**整单折扣**后的金额 | 作为计算各种税的基数（初始） | — |
| `$actual_price_item` | 实际行金额（**不含税**） | 最终的 `document_items.total` | `document_items.total` |

#### 5.2.2 计算步骤（按顺序）

**Step 1：行折扣 + 整单折扣**
```php
$item_amount = price × quantity;                              // 原始行金额
$item_discounted_amount = $item_amount;

// 行折扣（percentage / fixed）
if (item.discount) {
    $item_discounted_amount -= 行折扣金额;
}
// 整单折扣（分摊到此行）
if (item.global_discount) {
    $item_discounted_amount -= 整单折扣分摊额;
}
```

**Step 2：按税类型依次处理**（注意：不同税类型写入不同变量）

```php
$tax_amount = 0;
$item_tax_total = 0;
$actual_price_item = $item_amount = $item_discounted_amount;  // 三者初始值相等

// ───────── ① inclusive 税（价内税） ─────────
if (inclusive 税) {
    $tax_amount = $item_discounted_amount
                - ($item_discounted_amount / (1 + rate/100));  // 反向推税额
    $item_tax_total += $tax_amount;

    // ⚠️ 关键点：价内税需要从行金额中剥离出来
    $actual_price_item = $item_discounted_amount - $item_tax_total;
}

// ───────── ② fixed 税（固定税，按数量计） ─────────
if (fixed 税) {
    $tax_amount = tax.rate × quantity;                         // 固定税/件 × 数量
    $item_tax_total += $tax_amount;
    $item_amount += $tax_amount;                               // ⚠️ 只加 item_amount
}

// ───────── ③ normal 税（常规价外税） ─────────
if (normal 税) {
    $tax_amount = $actual_price_item × (rate/100);             // 基于不含税金额算
    $item_tax_total += $tax_amount;
    $item_amount += $tax_amount;                               // ⚠️ 只加 item_amount
}

// ───────── ④ withholding 税（代扣税，负值） ─────────
if (withholding 税) {
    $tax_amount = -( $actual_price_item × (rate/100) );        // 负税额
    $item_tax_total += $tax_amount;
    $item_amount += $tax_amount;                               // ⚠️ 相当于扣减
}

// ───────── ⑤ compound 税（复合税，基于含税总价再计征） ─────────
if (compound 税) {
    // ⚠️ 此时 item_amount 已包含 fixed/normal/withholding 的税额
    $tax_amount = ($item_amount / 100) × compound_rate;
    $item_tax_total += $tax_amount;
    // 注意：compound 税不单独加 item_amount，
    // 后续会和其他税一起累加到 grand_total
}
```

**Step 3：最终写入**
```php
$request['total'] = round($actual_price_item, $precision);    // 行金额（不含税）
$request['tax']   = round($item_tax_total, $precision);       // 行税额合计

// 行 taxes 明细写入 document_item_taxes（withholding 用 abs 存绝对值）
DocumentItemTax::create([
    'amount' => round(abs($item_tax['amount']), $precision),
    ...
]);
```

**结论修正**：
- `document_items.total` 存的是**不含税**的行金额（价税分离后）
- `document_items.tax` 存的是**税额合计**
- 价内税（inclusive）会**剥离行金额**；价外税（normal、fixed、compound）是在行金额**之外累加**；withholding 是负值**扣减**

### 5.3 前端（documents.js）的对应计算

对应方法：[onCalculateTotal()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L310-L393) 与 [calculateItemTax()](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js#L395-L530)

前端同样维护 `item.total` 和 `item.grand_total`：
- `item.total = price × quantity`（每次计算开始重置为原始值）
- `item.grand_total` = 扣完行折扣、整单折扣后，**加上税后**的最终行金额

### 5.4 分项总额汇总与写入

回到 [CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L158) 的 `handle()`：

```php
// ① createItems 遍历 items，返回四个聚合值
list($sub_total, $actual_total, $discount_amount_total, $taxes)
    = $this->createItems();

// ② 依次写入 document_totals（按 sort_order）
```

| code | 写入金额 | 来源与说明 |
|------|----------|------------|
| `sub_total` | `round($sub_total)` | 所有 item 的 `$document_item->total` 之和，即**不含税行金额累计** |
| `item_discount` | `round($discount_amount_total)` | 各行「行折扣金额」之和，> 0 时才写入 |
| `discount` | `round($discount_total)` | 整单折扣（%：sub_total × d%；fixed：整单折扣额） |
| `tax` | `round(abs($tax['amount']))` | 每个税种一条记录，按 `tax_id` 聚合；withholding 也取 abs 存 |
| `extra` | `round(abs($total['amount']))` | 额外项（运费等）；operator=addition/subtraction 控制加减到 amount |
| `total` | `$this->request['amount']` | 最终总额：actual_total − 整单折扣 + Σtax.amount + Σextra（±） |

### 5.5 模板显示

在 [default.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/components/documents/template/default.blade.php#L313-L373) 中：

```php
@foreach ($document->totals_sorted as $total)
    <x-money :amount="$total->amount" :currency="$document->currency_code" />
@endforeach
```

按 `sort_order` 遍历 `document_totals` 记录，使用 `<x-money>` 组件格式化显示。
注意：`document_totals.code='tax'` 的记录有多个（每个税种一条），显示时会依次列出。

---

## 六、金额格式化

### 6.1 PHP 端格式化

**辅助函数**：`money($amount, $currency, $convert = true)`
- 由 `akaunting/laravel-money` 包提供
- 返回格式化后的字符串（包含符号、千位分隔符等）

**Blade 组件**：
- 视图：[resources/views/vendor/money/components/money.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/vendor/money/components/money.blade.php)
- 用法：`<x-money :amount="$total->amount" :currency="$document->currency_code" />`

**格式化要素**：
- 货币符号（symbol）
- 符号位置（symbol_first）
- 小数点符号（decimal_mark）
- 千位分隔符（thousands_separator）
- 小数位数（precision）

### 6.2 JS 端格式化

**Vue 组件**：`<akaunting-money>`
- 组件：[resources/assets/js/components/AkauntingMoney.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/components/AkauntingMoney.vue)
- 视图：[resources/views/components/form/input/money.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/components/form/input/money.blade.php)
- 核心类：[resources/assets/js/plugins/money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/money.js)

**动态响应币种切换**：
```
watch.dynamicCurrency → this.money = {
    decimal:   currency.decimal_mark,
    thousands: currency.thousands_separator,
    prefix:    symbol_first ? symbol : '',
    suffix:    !symbol_first ? symbol : '',
    precision: parseInt(currency.precision),
}
```
币种切换后，无需重算数值，输入框的显示格式会自动按新币种刷新。

### 6.3 输入解析（Middleware）

请求输入的金额通过 [Money middleware](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Middleware/Money.php) 解析：

1. 检测 `amount`、`sale_price`、`purchase_price`、`opening_balance` 等参数，以及 `items[n].price`
2. 处理不同格式的数字字符串（逗号/小数点混用）
3. 调用 `money($value, $currency_code, false)` 解析为标准金额值（第三个参数 = false 表示不做格式化，只取数值）
4. 将解析后的数值写回 request

---

## 七、币种汇率与金额格式化的边界

### 7.1 核心边界划分

```
┌─────────────────┐      ┌─────────────────┐
│   汇率（Rate）   │──────│  格式化（Format）│
│   数值转换       │      │  显示呈现       │
└─────────────────┘      └─────────────────┘
         ↓                        ↓
   改变金额的数值            不改变数值
   从一种货币换算到另一种     只是换一种表现形式
```

### 7.2 汇率的职责

**属于业务逻辑层**，处理货币之间的价值换算：

- 输入：源金额 + 源币种 + 目标币种 + 汇率
- 输出：换算后的金额数值
- 影响：改变金额的数值大小
- 使用场景：多币种报表、跨币种支付、账户余额折算

**相关代码**：
- [app/Traits/Currencies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Traits/Currencies.php)
- `convertToDefault()` / `convertFromDefault()` / `convertBetween()`
- `getPaidAttribute()`、`getAmountDueAttribute()` 中跨币种累计

### 7.3 格式化的职责

**属于显示层**，处理数字到字符串的转换：

- 输入：金额数值 + 币种
- 输出：带符号、分隔符的可读字符串
- 影响：不改变金额数值，只改变表现形式
- 使用场景：页面显示、PDF 打印、报表输出

**相关代码**：
- `money()` 函数
- `<x-money>` 组件
- `akaunting-money` Vue 组件
- `<akaunting-currency-conversion>` 组件（仅显示换算，不改变表单数值）

### 7.4 精度的交叉角色

精度是汇率与格式化之间的交叉点：

| 层面 | 精度作用 | 关联 |
|------|----------|------|
| 汇率计算 | 转换后需要 round 到目标货币精度 | 数值层面 |
| 格式化显示 | 控制显示的小数位数 | 显示层面 |

**注意**：精度首先是币种的固有属性，其次才是格式化参数。格式化不能独立于币种存在，必须依附于具体币种来确定精度。

### 7.5 混淆点澄清

1. **汇率 ≠ 格式**：汇率转换后得到的仍是数值，不是格式化字符串
2. **格式化不做转换**：格式化只改变显示样式，不做汇率换算
3. **两者可组合**：先转换汇率得到数值，再格式化显示字符串
4. **录单切换币种 ≠ 立即换算总额**：前端只是换格式+回显换算值，真正数值换算在保存时由后端 Job 处理
5. **行金额 ≠ 含税总额**：`document_items.total` 存的是不含税价；税额单独存在 `document_item_taxes` 和 `document_items.tax`

### 7.6 典型调用链

**场景 1：显示单据总额**

```
document.amount (数值，单据币种)
  → money(amount, currency_code)  [格式化]
    → "$1,234.56" (显示字符串)
```

**场景 2：报表按默认货币汇总**

```
document.amount (数值，单据币种)
  → convertBetween(amount, doc_currency, doc_rate,
                   default_currency, 1.0)  [汇率转换]
    → default_amount (数值，默认货币)
      → money(default_amount, default_currency)  [格式化]
        → "¥8,765.43" (显示字符串)
```

**场景 3：跨币种付款 → 累计已付款**

```
transaction.amount (付款币种金额)
  → convertBetween(amount, pay_currency, pay_rate,
                   doc_currency, doc_rate)  [汇率转换]
    → 累加多条 transaction 的换算值
      → round(合计, 单据币种精度)
        → document.paid (单据币种数值)
```

---

## 八、关键文件索引

| 文件 | 作用 |
|------|------|
| [config/money.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/config/money.php) | 货币基础配置 |
| [public/money.json](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/public/money.json) | JS 端货币配置 |
| [app/Models/Setting/Currency.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Setting/Currency.php) | 币种模型 |
| [app/Traits/Currencies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Traits/Currencies.php) | 汇率转换 Trait |
| [app/Http/Middleware/Money.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Middleware/Money.php) | 金额输入解析中间件 |
| [app/Jobs/Document/CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | 单据总额计算 Job |
| [app/Jobs/Document/CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItem.php) | 明细行折扣与税额计算 Job |
| [app/Models/Document/Document.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Document/Document.php) | 单据模型（getPaidAttribute 跨币种累计） |
| [app/Models/Document/DocumentTotal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Document/DocumentTotal.php) | 分项总额模型 |
| [app/Http/Controllers/Modals/DocumentTransactions.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Controllers/Modals/DocumentTransactions.php) | 付款模态框控制器 |
| [resources/views/components/documents/form/totals.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/components/documents/form/totals.blade.php) | 录单页面总额区域（币种下拉+汇率输入） |
| [resources/views/modals/documents/payment.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/modals/documents/payment.blade.php) | 付款模态框视图 |
| [resources/views/vendor/money/components/money.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/vendor/money/components/money.blade.php) | 显示组件 |
| [resources/assets/js/views/common/documents.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/views/common/documents.js) | 录单/付款前端逻辑（币种切换、付款换算、税额计算） |
| [resources/assets/js/components/AkauntingMoney.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/components/AkauntingMoney.vue) | JS 端金额输入组件 |
| [resources/assets/js/components/AkauntingCurrencyConversion.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/components/AkauntingCurrencyConversion.vue) | JS 端汇率换算显示组件 |
| [resources/assets/js/plugins/money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/money.js) | JS 端金额类 |
| [resources/assets/js/plugins/currency.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/currency.js) | JS 端货币类 |
