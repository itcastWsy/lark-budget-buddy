# Budget Buddy 首次初始化指南

> 本文档描述如何创建记账所需的飞书多维表格结构。仅需在首次使用时执行一次。

## 前置条件

```bash
lark-cli auth login --domain base
```

## Step 1: 创建 Base

```bash
lark-cli base +base-create --name "Budget Buddy 记账本"
```

记录返回的 `base-token`（如 `MAGObxxxxxx`）。

> 如果使用 bot 身份创建，CLI 会自动尝试为当前 user 添加管理员权限。

## Step 2: 创建"交易记录"表

### 2.1 创建表

```bash
lark-cli base +table-create \
  --base-token <BASE_TOKEN> \
  --name "交易记录"
```

记录返回的 `table-id`。

### 2.2 创建字段

按以下顺序串行创建字段（每次创建间隔 0.5s）：

```bash
# 字段 1: 日期
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <TABLE_ID> \
  --json '{
    "type": "datetime",
    "name": "日期",
    "style": {"format": "yyyy-MM-dd HH:mm"}
  }'

# 字段 2: 类型（单选：支出/收入）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <TABLE_ID> \
  --json '{
    "type": "select",
    "name": "类型",
    "multiple": false,
    "options": [
      {"name": "支出", "hue": "Red", "lightness": "Light"},
      {"name": "收入", "hue": "Green", "lightness": "Light"}
    ]
  }'

# 字段 3: 金额（货币）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <TABLE_ID> \
  --json '{
    "type": "number",
    "name": "金额",
    "style": {"type": "currency", "precision": 2, "currency_code": "CNY"}
  }'

# 字段 4: 分类（单选）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <TABLE_ID> \
  --json '{
    "type": "select",
    "name": "分类",
    "multiple": false,
    "options": [
      {"name": "餐饮", "hue": "Orange", "lightness": "Light"},
      {"name": "交通", "hue": "Blue", "lightness": "Light"},
      {"name": "购物", "hue": "Purple", "lightness": "Light"},
      {"name": "娱乐", "hue": "Carmine", "lightness": "Light"},
      {"name": "居住", "hue": "Turquoise", "lightness": "Light"},
      {"name": "医疗", "hue": "Red", "lightness": "Light"},
      {"name": "教育", "hue": "Wathet", "lightness": "Light"},
      {"name": "通讯", "hue": "Gray", "lightness": "Light"},
      {"name": "日用", "hue": "Lime", "lightness": "Light"},
      {"name": "其他", "hue": "Gray", "lightness": "Lighter"},
      {"name": "工资", "hue": "Green", "lightness": "Standard"},
      {"name": "奖金", "hue": "Green", "lightness": "Light"},
      {"name": "兼职", "hue": "Green", "lightness": "Lighter"},
      {"name": "理财", "hue": "Yellow", "lightness": "Light"},
      {"name": "红包", "hue": "Red", "lightness": "Lighter"},
      {"name": "退款", "hue": "Wathet", "lightness": "Lighter"}
    ]
  }'

# 字段 5: 备注（文本）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <TABLE_ID> \
  --json '{
    "type": "text",
    "name": "备注"
  }'

# 字段 6: 支付方式（单选）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <TABLE_ID> \
  --json '{
    "type": "select",
    "name": "支付方式",
    "multiple": false,
    "options": [
      {"name": "微信", "hue": "Green", "lightness": "Light"},
      {"name": "支付宝", "hue": "Blue", "lightness": "Light"},
      {"name": "现金", "hue": "Yellow", "lightness": "Light"},
      {"name": "银行卡", "hue": "Orange", "lightness": "Light"},
      {"name": "信用卡", "hue": "Red", "lightness": "Light"},
      {"name": "其他", "hue": "Gray", "lightness": "Lighter"}
    ]
  }'
```

### 2.3 删除默认字段（可选）

新建表时系统会自动创建一个默认文本字段。可通过 `+field-list` 查看后删除：

```bash
lark-cli base +field-list --base-token <BASE_TOKEN> --table-id <TABLE_ID>
# 找到系统默认字段的 field_id，然后删除
lark-cli base +field-delete --base-token <BASE_TOKEN> --table-id <TABLE_ID> --field-id <DEFAULT_FIELD_ID> --yes
```

## Step 3: 创建"预算设置"表

### 3.1 创建表

```bash
lark-cli base +table-create \
  --base-token <BASE_TOKEN> \
  --name "预算设置"
```

记录返回的 `table-id`。

### 3.2 创建字段

```bash
# 字段 1: 分类（单选，与交易记录表的支出分类一致）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <BUDGETS_TABLE_ID> \
  --json '{
    "type": "select",
    "name": "分类",
    "multiple": false,
    "options": [
      {"name": "餐饮", "hue": "Orange", "lightness": "Light"},
      {"name": "交通", "hue": "Blue", "lightness": "Light"},
      {"name": "购物", "hue": "Purple", "lightness": "Light"},
      {"name": "娱乐", "hue": "Carmine", "lightness": "Light"},
      {"name": "居住", "hue": "Turquoise", "lightness": "Light"},
      {"name": "医疗", "hue": "Red", "lightness": "Light"},
      {"name": "教育", "hue": "Wathet", "lightness": "Light"},
      {"name": "通讯", "hue": "Gray", "lightness": "Light"},
      {"name": "日用", "hue": "Lime", "lightness": "Light"},
      {"name": "其他", "hue": "Gray", "lightness": "Lighter"}
    ]
  }'

# 字段 2: 月预算（货币）
lark-cli base +field-create \
  --base-token <BASE_TOKEN> \
  --table-id <BUDGETS_TABLE_ID> \
  --json '{
    "type": "number",
    "name": "月预算",
    "style": {"type": "currency", "precision": 2, "currency_code": "CNY"}
  }'
```

同样可选择删除默认字段。

## Step 4: 创建仪表盘（可选）

如果用户需要可视化面板，可创建仪表盘：

```bash
lark-cli base +dashboard-create \
  --base-token <BASE_TOKEN> \
  --name "消费概览"
```

仪表盘 Block 的创建较复杂，建议先读取 [dashboard-block-data-config.md](../../lark-base/references/dashboard-block-data-config.md) 了解 `data_config` 结构后再创建图表块。

## 初始化完成

完成后告知用户保存以下信息：

```
你的记账本已创建完成！请保存以下信息：
- Base Token: <BASE_TOKEN>
- 交易记录表 ID: <TRANSACTIONS_TABLE_ID>
- 预算设置表 ID: <BUDGETS_TABLE_ID>
- 多维表格链接: https://<domain>/base/<BASE_TOKEN>

现在可以开始记账了！试试说"午饭 35"
```
