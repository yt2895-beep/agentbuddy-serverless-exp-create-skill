---
name: sia-exp-create
description: "创建和管理 SIA Shadow In Auction 实验的 Serverless 环境（exp_e2e_task）。支持：创建实验（自动分配 TCE 集群）、查询任务状态、查询实验列表、关闭实验（下线集群）、按 biz_env 查询各机房明细及失败原因。固定参数：PSM=ad.analytics.collector, tenant=exp, branch=master, vregions=[SG,GCP,NOR,TTP1,TTP2], operator=yiwen.tan。唯一变量：biz_env（实验环境 ID）。Use when tasks mention SIA 实验, exp_e2e_task, biz_env, serverless exp, 创建实验环境, 查询实验, 关闭实验, or 查询机房状态/失败原因."
---

# SIA 实验环境管理

## 功能
通过 API 管理 SIA Shadow 实验的 Serverless 环境（exp_e2e_task）。
包括：创建、查询详情、列表查询、关闭实验。

## 基础信息
- 服务地址: `https://0mkweoh0.sg-fn.tiktok-row.net`
- 无需额外鉴权（直接 curl，不需要 Cookie 或 Token）

## 固定参数
- tenant: `exp`
- psm: `ad.analytics.collector`
- branch: `master`
- vregions: `["SG","GCP","NOR","TTP1","TTP2"]`（VA 已废弃）

## 变量参数
- `biz_env`：用户提供的实验环境 ID（每次不同）
- `operator`：调用者的 ByteDance 用户名（邮箱前缀，例如 `yiwen.tan`），如果用户没有指定就询问

---

## 1. 创建实验环境

将 `<BIZ_ENV>` 替换为用户提供的实验 id：

```bash
curl -X POST 'https://0mkweoh0.sg-fn.tiktok-row.net/api/v1/serverless/exp/exp_e2e_task/create' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'content-type: application/json' \
  -H 'origin: https://quantum-sg.bytedance.net' \
  -H 'referer: https://quantum-sg.bytedance.net/' \
  -H 'user-agent: Mozilla/5.0' \
  -d '{"tenant":"exp","biz_env":"<BIZ_ENV>","branch_info":{"ad.analytics.collector":{"branch":"master"}},"vregions":["SG","GCP","NOR","TTP1","TTP2"],"operator":"<OPERATOR>"}'
```

成功返回：
```json
{"msg": "", "code": 0, "data": {"task_id": 218, "bits_workflow_url": "https://bits.bytedance.net/devops/.../open_build/..."}}
```

- `task_id`：保存这个值，后续查询/关闭任务使用
- `bits_workflow_url`：BITS 流水线运行链接，可在浏览器打开查看各阶段详情

---

## 2. 查询实验状态

将 `<TASK_ID>` 替换为创建时返回的 task_id：

```bash
curl 'https://0mkweoh0.sg-fn.tiktok-row.net/api/v1/serverless/exp/exp_e2e_task/detail?task_id=<TASK_ID>' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'origin: https://quantum-sg.bytedance.net' \
  -H 'referer: https://quantum-sg.bytedance.net/'
```

返回的 `tasks[0]` 关键字段说明：

| 字段 | 含义 |
|------|------|
| `status` | `running`=流水线运行中 / `completed`=完成 / `fail`=失败 |
| `bits_workflow_url` | BITS 流水线链接（浏览器打开可看具体步骤卡在哪） |
| `actual_biz_env_specs_e2e_deploy_info` | 部署完成后各机房分配的 TCE 集群 Spec ID（completed 后才有） |
| `biz_env` | 实验环境 ID |
| `vregions` | 部署的机房列表 |

**状态流程**：创建 → `running`（BITS 流水线跑）→ `completed`（TCE 集群就绪，Libra 可配流量）或 `fail`

**注意**：BITS 流水线内部分步骤状态只能通过浏览器打开 `bits_workflow_url` 查看，API 只能查到整体状态。

---

## 3. 查询实验列表

```bash
curl 'https://0mkweoh0.sg-fn.tiktok-row.net/api/v1/serverless/exp/exp_e2e_task/list?tenant=exp&page_num=1&page_size=20' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'origin: https://quantum-sg.bytedance.net' \
  -H 'referer: https://quantum-sg.bytedance.net/'
```

---

## 4. 关闭实验环境（下线 TCE 集群）

将 `<TASK_ID>` 替换为要关闭的 task_id：

```bash
curl -X POST 'https://0mkweoh0.sg-fn.tiktok-row.net/api/v1/serverless/exp/exp_e2e_task/close' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'content-type: application/json' \
  -H 'origin: https://quantum-sg.bytedance.net' \
  -H 'referer: https://quantum-sg.bytedance.net/' \
  -H 'user-agent: Mozilla/5.0' \
  -d '{"task_id": <TASK_ID>, "operator": "<OPERATOR>"}'
```

成功返回：
```json
{"msg": "", "code": 0, "data": {"close_bits_workflow_url": "https://bits.bytedance.net/..."}}
```

---

## 5. 按 biz_env 查询各机房明细（含失败原因）

`exp_e2e_task/detail` 的 `actual_biz_env_specs_e2e_deploy_info.status` 只反映"创建动作是否发起完成"，不代表每个机房真的就绪，也不会暴露具体失败原因。要查每个机房的真实状态（含失败 err_msg）需要查询底层 spec 接口：

将 `<BIZ_ENV>` 替换为实验环境 ID：

```bash
curl 'https://quantum-sg.bytedance.net/api/serverless/spec/list/simple?tenant=exp&biz_env=<BIZ_ENV>&page=1&size=20' \
  -H 'accept: application/json'
```

返回 `spec_list` 中每个机房一条记录，关键字段：

| 字段 | 含义 |
|------|------|
| `region` | 机房（SG/GCP/NOR/TTP1/TTP2） |
| `spec_state` | 3=service_creating（创建中） / 5=service_create_failed（创建失败） |
| `state` | 同上的文字版状态 |
| `status` | JSON 字符串，正常时含 `deployment_status`；失败时含 `err_msg`（具体失败原因，如机房资源不足） |

如果想查单个机房更细的信息（qps、repo version、env_vars 等），先从上面拿到 `id`（即 spec_id），再查：

```bash
curl 'https://quantum-sg.bytedance.net/api/serverless/spec/detail?spec_id=<SPEC_ID>' \
  -H 'accept: application/json'
```

**无需鉴权，可直接 curl。**

---

## 使用示例

用户说："帮我创建实验，biz_env=sia_exp_test_terminal_1530"
→ 如果未提供 operator，询问用户的 ByteDance 用户名（邮箱前缀）；执行创建命令，替换 biz_env 和 operator，返回 task_id 和 bits_workflow_url

用户说："查一下 task 218 的状态"
→ 执行查询命令，返回 status、bits_workflow_url、以及部署完成的 spec 信息

用户说："task 218 的 bits 流水线卡在哪了"
→ 查询详情拿到 bits_workflow_url，告知用户在浏览器打开该链接查看每个 Step 的状态

用户说："关闭 task 218 的实验环境"
→ 执行 close 命令，返回 close_bits_workflow_url

用户说："最近创建了哪些实验"
→ 执行 list 命令，返回最近 20 条记录

用户说："task 218 现在到哪一步了" / "查一下每个机房的进度"
→ 先查 exp_e2e_task/detail 拿到 biz_env，再用 biz_env 查 spec/list/simple，逐机房列出 state/status；如有 spec_state=5（失败），从 status 字段里的 err_msg 提取具体失败原因（如资源不足）一并告知用户
