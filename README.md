# fwishu-cli

飞书 CLI 创作者大赛参赛项目。

## lark-budget-buddy

AI 智能记账 Skill，基于飞书 CLI + 多维表格构建。

### 核心能力

- **自然语言记账** — 说"午饭 35 微信"就能自动记录
- **支付截图识别** — 发微信/支付宝/银行App截图，AI 自动提取交易信息并记账
- **消费查询统计** — "本月花了多少"、"各分类占比"，一键出结果
- **预算管理** — 设定分类预算，超支自动提醒
- **月度财报** — 自动生成格式化飞书文档，含分类分析、支付方式分布、预算执行

### 快速开始

```bash
# 安装 lark-cli
npm install -g @larksuite/cli

# 认证
lark-cli auth login --domain base,docs,im

# 复制 Skill 到你的 agents 目录
cp -r skills/lark-budget-buddy ~/.agents/skills/
```

### 项目结构

```
skills/lark-budget-buddy/
├── SKILL.md                     # 核心 Skill 定义
├── README.md                    # 详细使用说明
├── LICENSE                      # MIT
└── references/
    ├── setup-guide.md           # 初始化建表指南
    └── payment-screenshot.md    # 支付截图识别工作流
```

### 参赛信息

- 赛事：[飞书 CLI 创作者大赛](https://www.woshipm.com/it/6376316.html)
- 赛道：GitHub 技术赛道
- 仓库：https://github.com/itcastWsy/lark-budget-buddy
