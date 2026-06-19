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

## 三、单据总额输出流程

### 3.1 整体流程

入口任务：[CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php)

```
创建单据 → 创建明细行（items） → 计算分项总额（totals） → 写入总额（amount）
```

### 3.2 分项计算顺序

| 序号 | 项目 | code | 计算方式 |
|------|------|------|----------|
| 1 | 小计 | sub_total | 所有行项目金额之和 |
| 2 | 行折扣 | item_discount | 每个 item 的折扣之和 |
| 3 | 整单折扣 | discount | 百分比/固定金额 |
| 4 | 税 | tax | 支持 normal/inclusive/fixed/withholding/compound 类型 |
| 5 | 额外项 | extra | 运费等自定义项（可加可减） |
| 6 | 总额 | total | 最终金额 |

### 3.3 明细行计算

见 [CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItem.php)：

```
单价 × 数量 → 行金额
  → 减去行折扣
    → 减去整单折扣分摊
      → 加上各种税（按不同类型分别计算）
        → 行最终金额
```

### 3.4 存储位置

- 分项明细：`document_totals` 表，模型 [DocumentTotal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Document/DocumentTotal.php)
- 最终总额：`documents.amount` 字段
- 每个 total 都有 `sort_order` 控制显示顺序

### 3.5 模板显示

在 [default.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/components/documents/template/default.blade.php#L313-L373) 中：

```php
@foreach ($document->totals_sorted as $total)
    <x-money :amount="$total->amount" :currency="$document->currency_code" />
@endforeach
```

按 sort_order 遍历 totals，使用 `<x-money>` 组件格式化显示。

---

## 四、金额格式化

### 4.1 PHP 端格式化

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

### 4.2 JS 端格式化

**Vue 组件**：`<akaunting-money>`
- 输入组件：[resources/views/components/form/input/money.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/components/form/input/money.blade.php)
- 核心类：[resources/assets/js/plugins/money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/money.js)
- 货币类：[resources/assets/js/plugins/currency.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/currency.js)

**format() 方法逻辑**：
```
amount → 除以 subunit 得到主单位
  → 按 precision 保留小数
    → 添加千位分隔符
      → 添加货币符号前缀/后缀
        → 处理负号
```

### 4.3 输入解析（Middleware）

请求输入的金额通过 [Money middleware](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Middleware/Money.php) 解析：

1. 检测 `amount`、`sale_price`、`purchase_price`、`opening_balance` 等参数
2. 处理不同格式的数字字符串（逗号/小数点）
3. 调用 `money()` 解析为标准金额值
4. 将解析后的值写回 request

---

## 五、币种汇率与金额格式化的边界

### 5.1 核心边界划分

```
┌─────────────────┐      ┌─────────────────┐
│   汇率（Rate）   │──────│  格式化（Format）│
│   数值转换       │      │  显示呈现       │
└─────────────────┘      └─────────────────┘
         ↓                        ↓
   改变金额的数值            不改变数值
   从一种货币换算到另一种     只是换一种表现形式
```

### 5.2 汇率的职责

**属于业务逻辑层**，处理货币之间的价值换算：

- 输入：源金额 + 源币种 + 目标币种 + 汇率
- 输出：换算后的金额数值
- 影响：改变金额的数值大小
- 使用场景：多币种报表、跨币种支付、账户余额折算

**相关代码**：
- [app/Traits/Currencies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Traits/Currencies.php)
- `convertToDefault()` / `convertFromDefault()` / `convertBetween()`

### 5.3 格式化的职责

**属于显示层**，处理数字到字符串的转换：

- 输入：金额数值 + 币种
- 输出：带符号、分隔符的可读字符串
- 影响：不改变金额数值，只改变表现形式
- 使用场景：页面显示、PDF 打印、报表输出

**相关代码**：
- `money()` 函数
- `<x-money>` 组件
- `akaunting-money` Vue 组件

### 5.4 精度的交叉角色

精度是汇率与格式化之间的交叉点：

| 层面 | 精度作用 | 关联 |
|------|----------|------|
| 汇率计算 | 转换后需要 round 到目标货币精度 | 数值层面 |
| 格式化显示 | 控制显示的小数位数 | 显示层面 |

**注意**：精度首先是币种的固有属性，其次才是格式化参数。格式化不能独立于币种存在，必须依附于具体币种来确定精度。

### 5.5 混淆点澄清

1. **汇率 ≠ 格式**：汇率转换后得到的仍是数值，不是格式化字符串
2. **格式化不做转换**：格式化只改变显示样式，不做汇率换算
3. **两者可组合**：先转换汇率得到数值，再格式化显示字符串

### 5.6 典型调用链

**场景：显示单据总额**

```
document.amount (数值，单据币种)
  → money(amount, currency_code)  [格式化]
    → "$1,234.56" (显示字符串)
```

**场景：报表按默认货币汇总**

```
document.amount (数值，单据币种)
  → getAmountConvertedToDefault()  [汇率转换]
    → default_amount (数值，默认货币)
      → money(default_amount, default_currency)  [格式化]
        → "¥8,765.43" (显示字符串)
```

---

## 六、关键文件索引

| 文件 | 作用 |
|------|------|
| [config/money.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/config/money.php) | 货币基础配置 |
| [public/money.json](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/public/money.json) | JS 端货币配置 |
| [app/Models/Setting/Currency.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Setting/Currency.php) | 币种模型 |
| [app/Traits/Currencies.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Traits/Currencies.php) | 汇率转换 Trait |
| [app/Http/Middleware/Money.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Http/Middleware/Money.php) | 金额输入解析中间件 |
| [app/Jobs/Document/CreateDocumentItemsAndTotals.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) | 单据总额计算 |
| [app/Jobs/Document/CreateDocumentItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Jobs/Document/CreateDocumentItem.php) | 明细行计算 |
| [app/Models/Document/DocumentTotal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/app/Models/Document/DocumentTotal.php) | 分项总额模型 |
| [resources/views/vendor/money/components/money.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/views/vendor/money/components/money.blade.php) | 显示组件 |
| [resources/assets/js/plugins/money.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/money.js) | JS 端金额类 |
| [resources/assets/js/plugins/currency.js](file:///d:/fz/0601-2/solo-dogfeeding/code/46-akaunting/resources/assets/js/plugins/currency.js) | JS 端货币类 |
