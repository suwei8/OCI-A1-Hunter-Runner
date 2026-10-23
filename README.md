# OCI-A1-Hunter-Runner (Public / 多租户工作流)

公开的 GitHub-hosted Actions 调度层；实际 Python 核心存放在 Private 仓库 [suwei8/OCI-A1-Hunter](https://github.com/suwei8/OCI-A1-Hunter)。

> **当前安全状态：只有手动触发，没有 `schedule`；默认 `dry-run` 不访问 OCI。** 不配置或授权 `live` 就不会创建实例。Public 工作流日志公开，禁止在日志或 artifact 中存放密钥。

## 多租户结构

- Public Repository：工作流、租户别名清单、唯一共享的 Private 源码只读凭证 `PRIVATE_REPO_TOKEN`。
- Private Repository：固定 SHA 的 Python OCI 核心；不包含真实 OCI 凭证。
- GitHub Environment：**每个 OCI Tenancy 一个独立 Environment**（例如 `ap-singapore-2-8` / `ap-singapore-2-9`），拥有独立的 OCI API Secrets、区域/配额配置与 TG Bot。
- `check` 支持选单个租户或 `all`；独立矩阵 Job 执行（最多同时 2 个，失败互不取消）。
- `live` 强制选单个租户；同时需要全局 + 单租户的人工开关。默认均关闭。
- 账号之间不共享 OCI 私钥；每个任务都在临时 GitHub-hosted Runner 中执行。不能承诺绝对保密：拥有 Public 工作流写权限的人仍有机会修改代码窃取其可访问的 Secrets，因此应限制写权限并设置 Environment 审批保护。部署作业引用对应 Environment；普通 `dry-run` 完全不引用任何 Environment。

### 1. 配置 Public Repository 级别的参数

打开 Settings → Secrets and variables → Actions。

| 范围 | 名称 | 设置 |
|---|---|---|
| Repository Secret | `PRIVATE_REPO_TOKEN` | 仅限 `suwei8/OCI-A1-Hunter` 仓库、Contents Read-only 的 Fine-grained PAT |
| Repository Variable | `OCI_TENANTS` | JSON 字符串，如 `["ap-singapore-2-8","ap-singapore-2-9"]` |
| Repository Variable | `OCI_HUNTER_READ_ONLY_ENABLED` | `true` 后才允许执行 `check` |
| Repository Variable | `OCI_HUNTER_LIVE_ENABLED` | **暂不设置为 true**（仅在明确批准实际创建时） |

`OCI_TENANTS` 只包含与 OCI 账号对应的**匿名别名**，不要放 OCID、私钥、邮箱、密码。别名可以使用 `ap-singapore-2-8` 这样“区域+账号序号”的格式。使用 3–48 位小写英文字母、数字和连字符，必须以字母开头、以字母或数字结尾，不能是保留字 `all`，最多 20 个。原有 `oci-main` / `oci-backup` 也继续支持。它们必须与下面的 GitHub Environment 名称**完全一致**。**别名只是标识，不会自动设置实际 OCI 区域**；仍需在每个 Environment 的 `OCI_REGION` 填入正确的 Region ID。

### 2. 每个租户创建独立的 GitHub Environment

打开 Public 仓库 Settings → Environments → New environment。

例如分别创建两个独立 Environment（不能仅修改清单而不创建同名 Environment）：

- `ap-singapore-2-8`
- `ap-singapore-2-9`

两者分别配置下面的 **Environment Secrets**（不是 Repository Secrets）：

| Environment Secret | 用途 |
|---|---|
| `OCI_TENANCY_OCID` | 当前租户 Tenancy OCID |
| `OCI_USER_OCID` | 当前租户专用 IAM User |
| `OCI_FINGERPRINT` | 当前租户 API Key 指纹 |
| `OCI_PRIVATE_KEY` | 当前租户 OCI API 私钥 PEM |
| `TG_BOT_TOKEN` | 可选，当前租户 TG Bot |
| `TG_CHAT_ID` | 可选，通知接收 Chat ID |

分别配置下面的 **Environment Variables**：

| Environment Variable | 用途 |
|---|---|
| `OCI_TENANT_EXPECTED_OCID` | 必填，必须与该 Environment 的 `OCI_TENANCY_OCID` 相同 |
| `OCI_REGION` | 必填，例如 `ap-singapore-1`（按实际租户设置） |
| `OCI_COMPARTMENT_OCID` | 必填，目标 compartment |
| `OCI_GUARD_COMPARTMENT_OCIDS` | 可选，额外需要统计的 compartment OCID，逗号分隔 |
| `OCI_INSTANCE_NAME` | 可选，默认 `oci-a1-hunter` |
| `OCI_AD_NAMES` | 可选，限制 AD 名称 |
| `OCI_A1_BUDGET_OCPUS` | 可选，每租户默认 **2** |
| `OCI_A1_BUDGET_MEMORY_GB` | 可选，每租户默认 **12** |

未来需要 `live` 时，**在对应 Environment** 另加：
`OCI_SUBNET_OCID`, `OCI_IMAGE_OCID`, `OCI_SSH_PUBLIC_KEY`, `OCI_TENANT_LIVE_APPROVED=true`；
同时还需要上面的 Repository `OCI_HUNTER_LIVE_ENABLED=true`。当前不要开启。

**迁移提醒**：若此前把 `OCI_PRIVATE_KEY`、`OCI_USER_OCID` 等 OCI 凭证配置在 Repository Secrets，请在确认所有 Environment 已独立配置后**移除 Repository 级 OCI Secrets**。GitHub Secrets 存在作用域回退，防止错误回退是多租户隔离的重要一环；代码还会检查 `OCI_TENANT_EXPECTED_OCID` 与 `OCI_TENANCY_OCID` 相等，不一致则在连接 OCI 之前拒绝执行。

### 3. 手动测试

进入 [OCI A1 Hunter Actions](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/workflows/hunter.yml) → Run workflow：

| mode | tenant | 作用 |
|---|---|---|
| `dry-run` | `all`（忽略此输入） | 离线验证安装与 Python CLI，不接触 OCI |
| `check` | `ap-singapore-2-8` | 只读验证主租户 OCI 凭证与现有 A1 用量 |
| `check` | `all` | 按 `OCI_TENANTS` 自动生成独立查询 Job，分别报告 |
| `live` | `ap-singapore-2-8` | **可能创建实例**，受两级开关保护 |
| `live` | `all` | **拒绝执行**；禁止跨全部租户一键创建 |

工作流使用固定的 Private commit、GitHub-hosted Runner、`persist-credentials:false`、只读 `GITHUB_TOKEN` 和每租户独立并发锁。不使用自托管 OCI 主机，不将 OCI 凭证写入仓库/数据库/永久文件。

### 安全范围与限制

1. `check` 仅调用 OCI 查询接口；`live` 是不同的显式审批路径。检查前不给 API IAM 用户创建权限是推荐做法。
2. **同一租户**不同 compartment 在同一区域内的非终止 A1 实例会累加；必须显式列举额外 compartment。不会自动跨 OCI region 汇总，也不会自动遍历子 compartment。
3. 不会停止、删除或调整现有实例；同名已有实例会阻止重复创建。
4. 2 OCPU / 12GB 是每个 Environment 默认预算；不要未经验证就扩大预算。OCI 实际服务限额与免费权益应以租户 Console 为准。
5. 每个 Environment 可以单独配置 TG，消息会注明租户**别名**，不要用真实账号标识作为公开别名。
6. 当前没有自动 cron。真实定时任务必须在完成所有租户的只读校验、IAM 最小权限和安全审查后单独启用。

下一步：先重跑 `dry-run`，再在 Environment 完整配置后执行单租户 `check`，最后检查 `all` 的隔离效果。
