# sia-exp-create

AgentBuddy Skill：SIA Shadow In Auction 实验环境管理。

支持通过自然语言创建、查询、关闭 SIA 实验的 Serverless 环境（exp_e2e_task）。

## 安装

```bash
git clone https://code.byted.org/yiwen.tan/sia-exp-create.git ~/.agents/skills/sia-exp-create
```

> 安装后重启 Agent（Codex / Claude Code / TRAE）即生效，无需其他配置。

## 更新

```bash
cd ~/.agents/skills/sia-exp-create && git pull
```

## 使用示例

```
帮我创建 SIA 实验，biz_env=sia_exp_test_1001
查一下 task 218 的状态
关闭 task 218 的实验环境
最近创建了哪些实验
```

## 说明

- 服务接口：`https://0mkweoh0.sg-fn.tiktok-row.net`（无需鉴权）
- 固定参数：tenant=exp, PSM=ad.analytics.collector, branch=master, vregions=[SG,GCP,NOR,TTP1,TTP2]
- 变量参数：`biz_env`（实验环境 ID）、`operator`（你的 ByteDance 用户名，即邮箱前缀）
