# OCI-A1-Hunter-Runner (Public)

公开调度仓库，实际 Python 源码保存在私有仓库 [suwei8/OCI-A1-Hunter](https://github.com/suwei8/OCI-A1-Hunter)。

**当前安全状态：仅 `workflow_dispatch` 手动触发；没有启用 cron；默认 `dry-run`，不访问 OCI。新增 `check` 只读查询，需要单独开关；`live` 创建需要另一个独立开关。**

## 架构

Public Actions -> 私有仓库 Fine-grained PAT（只读） -> 固定源码 commit -> Python CLI -> OCI + Telegram。

Public 标准 GitHub-hosted Runner 免费执行，但 Actions 运行日志是公开的。禁止在输出、artifact 或缓存中存储 OCI 私钥、Fine-grained PAT 和 Telegram Token。

## 安全配置（运行 check 前设置 OCI 凭证）

创建仅限 `suwei8/OCI-A1-Hunter` 仓库、`Contents: Read-only` 的 Fine-grained Personal Access Token。将其放入 Public 仓库的 `PRIVATE_REPO_TOKEN` Secret，避免给予其他仓库权限。Token 到期后需要轮换。

在本 Public 仓库 Settings -> Secrets and variables -> Actions 中设置：

| Secret | 用途 |
|---|---|
| `PRIVATE_REPO_TOKEN` | 只读拉取 Private 仓库的 Fine-grained PAT |
| `OCI_PRIVATE_KEY` | OCI API 签名私钥（PEM 原文） |
| `OCI_TENANCY_OCID` | tenancy OCID |
| `OCI_USER_OCID` | 专用 IAM user OCID |
| `OCI_FINGERPRINT` | OCI API 指纹 |
| `TG_BOT_TOKEN` | Telegram Bot Token |
| `TG_CHAT_ID` | Telegram Chat ID |

在 Actions Variables 中设置：
`OCI_REGION`, `OCI_COMPARTMENT_OCID`, `OCI_SUBNET_OCID`, `OCI_IMAGE_OCID`, `OCI_SSH_PUBLIC_KEY`, `OCI_INSTANCE_NAME`（默认 oci-a1-hunter），可选 `OCI_AD_NAMES`、`OCI_GUARD_COMPARTMENT_OCIDS`（要统计的其他 compartment OCID，逗号分隔）、`OCI_A1_BUDGET_OCPUS` 和 `OCI_A1_BUDGET_MEMORY_GB`（默认 2/12，超出默认值须先核对 OCI 实际权益）。

**注意**：OCI API 私钥不在 Private 代码仓库，也不应放进 Public 文件；它只在 `check` 或 `live` 执行步骤中由 GitHub Actions 注入临时 Runner 环境变量，切勿打印、上传 artifact 或缓存。Public 仓库工作流的编辑权限必须严格限制。

## 手动运行模式

进入本仓库 Actions -> OCI A1 Hunter (manual safe mode) -> Run workflow：

| mode | 所需配置 | 行为 |
|---|---|---|
| `dry-run`（默认） | `PRIVATE_REPO_TOKEN` | 安装 Private 源码并运行模拟；绝不访问 OCI |
| `check` | OCI API Secrets + 基础 Variables + `OCI_HUNTER_READ_ONLY_ENABLED=true` | **只读**调用 OCI List API，统计可见 compartment 内现有 A1 消耗，计算 2 OCPU/12GB 预算，发送可选 TG 摘要 |
| `live` | 全部启动配置 + `OCI_HUNTER_LIVE_ENABLED=true` | **可能创建** A1 实例；当前阶段不要启用 |

`check` 的最小变量是 `OCI_REGION`, `OCI_COMPARTMENT_OCID`；可选 `OCI_AD_NAMES`, `OCI_GUARD_COMPARTMENT_OCIDS`, `OCI_INSTANCE_NAME`、预算变量。**无需 `OCI_SUBNET_OCID`、`OCI_IMAGE_OCID`、`OCI_SSH_PUBLIC_KEY`**，也不需要给 OCI 用户创建实例权限。OCI IAM 专用用户应只获准读取必要的 compartment 资源。缺失或读取失败会报错，不会创建、修改、删除实例。

**审计边界**：只统计配置的 compartment（含额外显式列举）的该区域 A1 实例；不自动跨 region、跨所有子 compartment。若存在其他占用，人工核实 OCI Console 服务限额后才能安全启用生产模式。

## 启用生产前的人工清单

1. 仅限 Private 仓库的只读 Fine-grained PAT、受限 OCI IAM 用户、必要网络资源和子网已创建。
2. 确认账号所有 A1 实例的总用量，尤其核查其他 compartment/region；当前核心扫描目标与手工列举的额外 compartment，不跨 region 自动扫描。
3. 将 Private 代码固定到审核后的 SHA（工作流已固定安全修订提交），安装使用 `constraints.txt` 固定已验证的依赖版本；目前尚未使用包哈希锁定。
4. 配置 Secrets/Variables，限制 Public 工作流变更权限。
5. 先完成 `dry-run`，再设置 `OCI_HUNTER_READ_ONLY_ENABLED=true` 选择 `check` 验证实际可见配额。**只有另外明确批准创建实例时**，才设置 `OCI_HUNTER_LIVE_ENABLED=true` 并选择 `live`。
6. 核验成功后，才能单独提交定时 `schedule` 配置。不要在此阶段启用 cron。

## 安全原则

- 无 `pull_request_target`，不对任何 PR 暴露 Secrets。
- GitHub token 只读；checkout 不持久化凭证。
- 工作流锁住固定 Private commit，不跟踪其 main 的浮动变化。
- 本项目没有删除、停止、重建已有实例的代码。
- OCI 请求日志不含完整异常内容或私钥。

## 默认免费额度保护

OCI 当前 Always Free 文档描述 A1 每月 1,500 OCPU 小时与 9,000 GB 小时，约等于 2C12G 整月运行。因此本项目默认总预算 **2 OCPU / 12GB**。该数值与某些旧版 4C24G 教程不同。请核对具体 tenancy 的权益后再设置更高预算。容量扫描不能取代 OCI 官方 service limits，也不保证全租户/跨区域统计完全。
