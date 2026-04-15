---
name: lark-budget-buddy
version: 1.2.0
description: "智能记账伙伴：基于飞书多维表格的个人/团队记账与财务分析工具。支持自然语言快速记账、支付截图AI识别自动记账（微信/支付宝/银行App截图）、消费查询与统计、预算管理与超支提醒、月度财报生成。当用户需要记账、记一笔、发截图记账、查账、看消费、设预算、生成财报、月度总结、消费分析时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# Budget Buddy — 智能记账伙伴

> **前置条件：** 先阅读 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，了解认证、权限和安全规则。

## 已配置的多维表格

当前记账本已初始化完成，直接使用以下配置：

```
BASE_TOKEN=WrzBbBAvNaU83BsVNt0ci7c9ndf
TRANSACTIONS_TABLE=tblkghyZJ3LiZ6Z1
BUDGETS_TABLE=tbl2Ub4wWTJDwi6R
```

多维表格链接：https://my.feishu.cn/base/WrzBbBAvNaU83BsVNt0ci7c9ndf

## 适用场景

- "记一笔 午饭 35" / "咖啡 28 微信" / "打车 45 支付宝"
- "收入 15000 工资" / "收到红包 200"
- [发送支付截图] + "帮我记一下" / "记账"
- "今天花了多少" / "本月消费统计" / "这周在餐饮上花了多少"
- "设置餐饮预算 2000" / "本月预算还剩多少"
- "生成月度财报" / "生成 3 月财报" / "消费分析"
- "删掉刚才那笔" / "修改上一笔金额为 50"

## 前置条件

确保已认证：
```bash
lark-cli auth login --domain base,docs,im
```

所需 scope：
- `bitable:app` — 多维表格读写
- `docs:doc` — 文档创建
- `im:message` — 消息发送（可选，用于推送提醒）

## 数据架构

```
飞书多维表格 (Budget Buddy 记账本)
├── 交易记录表 (tblkghyZJ3LiZ6Z1) — 存储每一笔收支明细
├── 预算设置表 (tbl2Ub4wWTJDwi6R) — 存储每个分类的月度预算
└── 仪表盘    — 可视化消费趋势和分类占比
```

### 交易记录表字段

| 字段 | 类型 | 说明 |
|------|------|------|
| 日期 | datetime | 交易时间，格式 "yyyy-MM-dd HH:mm:ss" |
| 类型 | select | 支出 / 收入 |
| 金额 | number (currency CNY) | 交易金额 |
| 分类 | select | 餐饮/交通/购物/娱乐/居住/医疗/教育/通讯/日用/其他/工资/奖金/兼职/理财/红包/退款 |
| 备注 | text | 交易描述 |
| 支付方式 | select | 微信/支付宝/现金/银行卡/信用卡/其他 |

### 预算设置表字段

| 字段 | 类型 | 说明 |
|------|------|------|
| 分类 | select | 支出分类（餐饮/交通/购物/娱乐/居住/医疗/教育/通讯/日用/其他） |
| 月预算 | number (currency CNY) | 该分类的每月预算上限 |

## 工作流

### 一、快速记账

#### 自然语言解析规则

AI 需将用户的自然语言输入解析为结构化数据：

| 用户输入 | 类型 | 金额 | 分类 | 备注 | 支付方式 |
|---------|------|------|------|------|---------|
| "午饭 35" | 支出 | 35 | 餐饮 | 午饭 | （默认） |
| "打车 45 支付宝" | 支出 | 45 | 交通 | 打车 | 支付宝 |
| "淘宝买衣服 299 信用卡" | 支出 | 299 | 购物 | 淘宝买衣服 | 信用卡 |
| "收入 15000 工资" | 收入 | 15000 | 工资 | 工资 | （默认） |
| "收到红包 200" | 收入 | 200 | 红包 | 收到红包 | （默认） |
| "电费 200" | 支出 | 200 | 居住 | 电费 | （默认） |
| "看电影 80 微信" | 支出 | 80 | 娱乐 | 看电影 | 微信 |

**解析优先级：**
1. 金额：提取数字（必须有）
2. 类型：包含"收入/工资/奖金/兼职/理财/红包/退款"关键词为收入，否则默认支出
3. 分类：根据备注内容智能匹配最合适的分类
4. 支付方式：提取"微信/支付宝/现金/银行卡/信用卡"关键词，未提及则不填
5. 备注：去除金额和支付方式后的描述文本
6. 日期：默认当天，用户可指定（如"昨天午饭 35"）

#### 写入命令

```bash
lark-cli base +record-upsert \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --json '{
    "日期": "2026-04-15 12:30:00",
    "类型": "支出",
    "金额": 35,
    "分类": "餐饮",
    "备注": "午饭",
    "支付方式": "微信"
  }'
```

**写入后：**
- 回复用户确认信息："已记录：支出 ¥35（餐饮 - 午饭）"
- 如果当月该分类已设预算，顺带计算剩余额度并提醒

#### 批量记账

用户可一次输入多笔：

```
午饭 35，地铁 4，咖啡 18
```

按逗号/句号/换行分割后逐条解析，**串行调用** `+record-upsert`（批次间延迟 0.5s），最后统一回复汇总。

---

### 二、图片识别记账

当用户在飞书聊天中发送**支付截图**（微信/支付宝/银行App）并附带"记账"、"帮我记一下"等意图时，执行图片识别记账流程。

> 详细识别规则和异常处理请阅读：[`references/payment-screenshot.md`](references/payment-screenshot.md)

#### 工作流

```
Step 1: 检测图片消息 → 下载图片
    │
Step 2: AI 视觉分析截图 → 提取交易结构化信息
    │
Step 3: 展示解析结果 → 等待用户确认
    │
Step 4: 确认无误 → 写入多维表格
```

#### Step 1: 下载图片

```bash
lark-cli im +messages-resources-download \
  --message-id <message_id> \
  --file-key <image_key> \
  --type image \
  --output ./payment_screenshot.png
```

其中 `message_id`（`om_xxx`）和 `image_key`（`img_v3_xxx`）从消息列表中获取。

#### Step 2: AI 视觉分析

将图片交给 AI 视觉模型分析，提取以下字段：

| 提取字段 | 说明 |
|---------|------|
| 金额 | 交易金额数字 |
| 支付方向 | 支出（默认）/ 收入（特殊标识） |
| 商户/收款方 | 收款商户名称 |
| 交易时间 | 交易发生时间 |
| 支付方式 | 从截图UI判断：微信/支付宝/银行卡 |
| 商品/服务描述 | 具体的商品或服务 |

然后自动映射消费分类（参考 `references/payment-screenshot.md` 中的分类映射规则）。

#### Step 3: 展示确认

```
📸 已识别支付截图：

| 项目 | 内容 |
|------|------|
| 金额 | ¥35.00 |
| 类型 | 支出 |
| 分类 | 餐饮（自动识别：沙县小吃） |
| 商户 | 沙县小吃 |
| 时间 | 2026-04-15 12:30 |
| 支付方式 | 微信 |

确认记录？（回复"确认"或"修改XX为XX"）
```

#### Step 4: 写入

用户确认后写入：

```bash
lark-cli base +record-upsert \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --json '{
    "日期": "2026-04-15 12:30:00",
    "类型": "支出",
    "金额": 35,
    "分类": "餐饮",
    "备注": "沙县小吃 午餐",
    "支付方式": "微信"
  }'
```

#### 异常处理

- **截图模糊** → 提示用户发更清晰的截图，或手动输入
- **无法判断分类** → 列出可选分类，请用户指定
- **非支付截图** → 提示发送支付截图或手动记账

---

### 三、查询与统计

#### 2.1 按时间段查询消费总额

```bash
# 本月支出总额
lark-cli base +data-query \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "tblkghyZJ3LiZ6Z1"}},
    "measures": [{"field_name": "金额", "aggregation": "sum", "alias": "total_amount"}],
    "filters": {
      "type": 1,
      "conjunction": "and",
      "conditions": [
        {"field_name": "类型", "operator": "is", "value": ["支出"]},
        {"field_name": "日期", "operator": "is", "value": ["CurrentMonth"]}
      ]
    },
    "shaper": {"format": "flat"}
  }'
```

日期过滤值可替换：
- 今天：`["Today"]`
- 本周：`["CurrentWeek"]`
- 上月：`["LastMonth"]`

#### 2.2 按分类统计

```bash
# 本月各分类支出
lark-cli base +data-query \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "tblkghyZJ3LiZ6Z1"}},
    "dimensions": [{"field_name": "分类", "alias": "dim_category"}],
    "measures": [
      {"field_name": "金额", "aggregation": "sum", "alias": "total_amount"},
      {"field_name": "金额", "aggregation": "count", "alias": "tx_count"}
    ],
    "filters": {
      "type": 1,
      "conjunction": "and",
      "conditions": [
        {"field_name": "类型", "operator": "is", "value": ["支出"]},
        {"field_name": "日期", "operator": "is", "value": ["CurrentMonth"]}
      ]
    },
    "sort": [{"field_name": "total_amount", "order": "desc"}],
    "shaper": {"format": "flat"}
  }'
```

#### 2.3 按支付方式统计

```bash
lark-cli base +data-query \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "tblkghyZJ3LiZ6Z1"}},
    "dimensions": [{"field_name": "支付方式", "alias": "dim_payment"}],
    "measures": [{"field_name": "金额", "aggregation": "sum", "alias": "total_amount"}],
    "filters": {
      "type": 1,
      "conjunction": "and",
      "conditions": [
        {"field_name": "类型", "operator": "is", "value": ["支出"]},
        {"field_name": "日期", "operator": "is", "value": ["CurrentMonth"]}
      ]
    },
    "sort": [{"field_name": "total_amount", "order": "desc"}],
    "shaper": {"format": "flat"}
  }'
```

#### 2.4 收入统计

将 filters 中 `"类型"` 条件改为 `{"field_name": "类型", "operator": "is", "value": ["收入"]}` 即可。

#### 2.5 查看最近 N 笔记录

```bash
lark-cli base +record-list \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --limit 10
```

> 注意：`+record-list` 用于取明细，不做聚合。需要统计用 `+data-query`。

#### 回复格式

```
📊 本月消费统计（2026年4月）

总支出：¥356.00（共 4 笔）
总收入：¥15,000.00（共 1 笔）
净收支：+¥14,644.00

分类 TOP：
1. 购物  ¥299.00（84.0%）- 1 笔
2. 餐饮  ¥53.00（14.9%）- 2 笔
3. 交通  ¥4.00（1.1%）- 1 笔
```

---

### 四、预算管理

#### 4.1 设置预算

```bash
lark-cli base +record-upsert \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tbl2Ub4wWTJDwi6R \
  --json '{
    "分类": "餐饮",
    "月预算": 2000
  }'
```

> 写入前先用 `+record-list` 检查该分类是否已有预算记录。如有，用 `--record-id` 更新；如无，新建。

#### 4.2 查看预算使用情况

执行步骤：
1. `+record-list` 读取预算设置表 → 获取各分类预算
2. `+data-query` 查询本月各分类实际支出
3. AI 对比计算剩余额度和使用率

回复格式：

```
💰 本月预算使用情况（2026年4月）

| 分类 | 预算 | 已用 | 剩余 | 进度 |
|------|------|------|------|------|
| 餐饮 | ¥2,000 | ¥53 | ¥1,947 | 2.7% |

预算执行正常。
```

#### 4.3 超支预警规则

每次记账后，如果该分类有预算设置：
- 使用率 >= 100%：回复"⚠️ [分类]已超支 ¥XX"
- 使用率 >= 80%：回复"⚠️ [分类]预算已用 XX%，剩余 ¥XX"
- 使用率 < 80%：不额外提醒

---

### 五、月度财报生成

用户说"生成月度财报"或"生成 X 月财报"时执行。

#### 5.1 数据采集

依次执行以下查询：

1. 本月支出总额 + 笔数
2. 本月收入总额 + 笔数
3. 本月各分类支出统计
4. 本月各支付方式统计
5. 本月各分类收入统计
6. 读取预算设置表

> `+data-query` 的日期过滤使用 `CurrentMonth`（本月）或 `LastMonth`（上月）关键字。如果用户指定了具体月份（如"3月"），则需要用 `ExactDate` + 毫秒时间戳指定月初和月末。

#### 5.2 生成飞书文档

```bash
lark-cli docs +create \
  --title "月度财报 - YYYY年MM月" \
  --markdown '
# 月度财报 - YYYY年MM月

## 总览

| 指标 | 金额 |
|------|------|
| 总支出 | ¥XX.XX |
| 总收入 | ¥XX.XX |
| 净收支 | +¥XX.XX |
| 交易笔数 | XX 笔 |

## 支出分析

### 分类明细

| 排名 | 分类 | 金额 | 占比 | 笔数 |
|------|------|------|------|------|
| 1 | ... | ¥... | ...% | ... |

### 支付方式分布

| 支付方式 | 金额 | 占比 |
|---------|------|------|
| ... | ¥... | ...% |

## 预算执行情况

| 分类 | 预算 | 实际 | 执行率 | 状态 |
|------|------|------|--------|------|
| ... | ¥... | ¥... | ...% | ... |

## 收入分析

| 分类 | 金额 |
|------|------|
| ... | ¥... |

## AI 小结

（根据数据生成简要分析和建议）
'
```

#### 5.3 文档生成后

- 返回文档链接给用户
- 如果用户要求，可通过 `lark-cli im +messages-send` 将财报链接发到指定群聊

---

### 六、删除 / 修改记录

#### 删除最近一笔

```bash
# Step 1: 获取最新一条记录
lark-cli base +record-list \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --limit 1

# Step 2: 用返回的 record_id 删除
lark-cli base +record-delete \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --record-id <record_id> \
  --yes
```

#### 修改记录

```bash
lark-cli base +record-upsert \
  --base-token WrzBbBAvNaU83BsVNt0ci7c9ndf \
  --table-id tblkghyZJ3LiZ6Z1 \
  --record-id <record_id> \
  --json '{"金额": 50}'
```

---

## 安全规则

1. **写入前确认**：记账时将解析结果展示给用户确认后再写入（金额 > 1000 时强制确认）
2. **删除需确认**：删除操作前告知用户将删除的记录内容
3. **预算修改**：修改预算前展示当前值和新值
4. **不处理敏感数据**：不记录银行卡号、密码等敏感信息

## 意图 → 动作索引

| 用户意图 | 动作 | 核心命令 |
|---------|------|---------|
| 记一笔/记账 | 解析 + 写入 | `base +record-upsert` |
| 发截图记账 | 下载图片 → AI视觉分析 → 写入 | `im +messages-resources-download` → `base +record-upsert` |
| 批量记账 | 分割 + 逐条写入 | `base +record-upsert`（串行） |
| 查消费/看账单 | 聚合查询 | `base +data-query` |
| 看明细/最近 N 笔 | 取记录 | `base +record-list` |
| 设预算 | 写入预算表 | `base +record-upsert` |
| 看预算 | 读预算 + 查实际 | `base +record-list` + `base +data-query` |
| 生成财报 | 多维查询 + 创建文档 | `base +data-query` + `docs +create` |
| 删除/修改 | 定位 + 操作 | `base +record-delete` / `base +record-upsert` |
| 推送提醒 | 发消息 | `im +messages-send` |

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 读写多维表格 | `bitable:app` |
| 聚合查询 | `bitable:app`（需 FA 权限） |
| 创建文档 | `docs:doc` |
| 发送消息 | `im:message` |

## 参考

- [payment-screenshot.md](references/payment-screenshot.md) — 支付截图识别记账工作流（必读）
- [lark-shared](../lark-shared/SKILL.md) — 认证、权限（必读）
- [lark-base](../lark-base/SKILL.md) — 多维表格全部命令
- [lark-doc](../lark-doc/SKILL.md) — 文档创建
- [lark-im](../lark-im/SKILL.md) — 消息发送、图片下载
