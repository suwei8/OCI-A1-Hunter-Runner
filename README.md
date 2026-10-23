# OCI-A1-Hunter-Runner (Public)

公开调度仓库，实际 Python 源码保存在私有仓库 [suwei8/OCI-A1-Hunter](https://github.com/suwei8/OCI-A1-Hunter)。

**当前安全状态：仅 `workflow_dispatch` 手动触发；没有启用 cron；默认 dry-run，不会连接 OCI。**

## 架构

Public Actions -> 私有仓库只读 SSH Deploy Key -> 固定源码 commit -> Python CLI -> OCI + Telegram。

Public 标准 GitHub-hosted Runner 免费执行，但 Actions 运行日志是公开的。禁止在输出、artifact 或缓存中存储 OCI 私钥、SSH Deploy Key 和 Telegram Token。

## 安全配置（暂时不必设置）

在 Private 仓库 Settings -> Deploy keys 添加专用公钥（**Allow write access 保持关闭**）。

在本 Public 仓库 Settings -> Secrets and variables -> Actions 中设置：

| Secret | 用途 |
|---|---|
| `PRIVATE_REPO_SSH_KEY` | 对 Private 仓库只读 Deploy Key 私钥 |
| `OCI_PRIVATE_KEY` | OCI API 签名私钥（PEM 原文） |
| `OCI_TENANCY_OCID` | tenancy OCID |
| `OCI_USER_OCID` | 专用 IAM user OCID |
| `OCI_FINGERPRINT` | OCI API 指纹 |
| `TG_BOT_TOKEN` | Telegram Bot Token |
| `TG_CHAT_ID` | Telegram Chat ID |

在 Actions Variables 中设置：
`OCI_REGION`, `OCI_COMPARTMENT_OCID`, `OCI_SUBNET_OCID`, `OCI_IMAGE_OCID`, `OCI_SSH_PUBLIC_KEY`, `OCI_INSTANCE_NAME`（默认 oci-a1-hunter），可选 `OCI_AD_NAMES`。

**注意**：OCI API 私钥不在 Private 代码仓库，也不应放进 Public 文件；它仅在 live 模式下提供给 GitHub-hosted Runner 内存。

## 启用生产前的人工清单

1. 只读 Deploy Key、受限 OCI IAM 用户、必要网络资源和子网已创建。
2. 确认账号中所有 A1 实例的总用量，尤其检查其他 compartment/region；目前核心只统计目标 compartment。
3. 将 Private 代码固定到审核后的 SHA（工作流已固定首个提交）。
4. 配置 Secrets/Variables，限制 Public 工作流变更权限。
5. 在变量中明确设置 `OCI_HUNTER_LIVE_ENABLED=true`，再手动选择 live 运行一次。
6. 核验成功后，才能单独提交定时 `schedule` 配置。不要在此阶段启用 cron。

## 安全原则

- 无 `pull_request_target`，不对任何 PR 暴露 Secrets。
- GitHub token 只读；checkout 不持久化凭证。
- 工作流锁住固定 Private commit，不跟踪其 main 的浮动变化。
- 本项目没有删除、停止、重建已有实例的代码。
- OCI 请求日志不含完整异常内容或私钥。
