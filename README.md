# OCI-A1-Hunter-Runner (Public / 多租户工作流)

**生产试运行入口（v1.0-RC1）：** [查看当前验收证据及单租户上线操作单](docs/GO_LIVE.md)。已通过最新 82 项离线回归测试；历史只读 preflight 成功；**尚未授权或验证 live 创建**，需要操作者明确启用。

公开的 GitHub-hosted Actions 调度层；实际 Python 核心存放在 Private 仓库 [suwei8/OCI-A1-Hunter](https://github.com/suwei8/OCI-A1-Hunter)。

> **当前安全状态：只有手动触发，没有 `schedule`；默认 `dry-run` 不访问 OCI。** 不配置或授权 `live` 就不会创建实例。Public 工作流日志公开，禁止在日志或 artifact 中存放密钥。

## 多租户结构

- Public Repository：工作流、租户别名清单、唯一共享的 Private 源码只读凭证 `PRIVATE_REPO_TOKEN`。
- Private Repository：固定 SHA 的 Python OCI 核心；不包含真实 OCI 凭证。
- GitHub Environment：**每个 OCI Tenancy 一个独立 Environment**（例如 `ap-singapore-2-8` / `ap-singapore-2-9`），拥有独立的 OCI API Secrets、可选配额配置；Telegram Bot 由 Repository Secrets 统一管理。
- `check`（快速检查）、`audit`（递归租户 Compartment 审计）和 `preflight`（正式启动配置只读预检）均可选择单个租户或 `all`；独立矩阵 Job 执行（最多同时 2 个，失败互不取消）。
- `live` 必须由用户手动选择单个租户，只需要 **一个 Repository Variable `OCI_HUNTER_LIVE_ENABLED=true`**；不再需要每租户重复添加 `OCI_TENANT_LIVE_APPROVED`。默认关闭。
- 账号之间不共享 OCI 私钥；每个任务都在临时 GitHub-hosted Runner 中执行。不能承诺绝对保密：拥有 Public 工作流写权限的人仍有机会修改代码窃取其可访问的 Secrets，因此应限制写权限并设置 Environment 审批保护。部署作业引用对应 Environment；普通 `dry-run` 完全不引用任何 Environment。

### 1. 配置 Public Repository 级别的参数

打开 Settings → Secrets and variables → Actions。

| 范围 | 名称 | 设置 |
|---|---|---|
| Repository Secret | `PRIVATE_REPO_TOKEN` | 仅限 `suwei8/OCI-A1-Hunter` 仓库、Contents Read-only 的 Fine-grained PAT |
| Repository Secret | `TG_BOT_TOKEN` | 共享的 Telegram Bot Token（可选） |
| Repository Secret | `TG_CHAT_ID` | 共享的 Telegram Chat ID（可选） |
| Repository Secret | `OCI_SSH_PUBLIC_KEY` | **所有租户共用的一份 SSH 公钥**，仅在此处添加一次 |
| Repository Variable | `OCI_TENANTS` | JSON 字符串，如 `["ap-singapore-2-8","ap-singapore-2-9"]` |
| Repository Variable | `OCI_HUNTER_READ_ONLY_ENABLED` | `true` 后才允许执行 `check`、`audit` 或 `preflight` |
| Repository Variable | `OCI_HUNTER_LIVE_ENABLED` | **暂不设置为 true**（仅在明确批准实际创建时） |

`OCI_TENANTS` 只包含与 OCI 账号对应的**匿名别名**，不要放 OCID、私钥、邮箱、密码。别名可以使用 `ap-singapore-2-8` 这样“区域+账号序号”的格式。使用 3–48 位小写英文字母、数字和连字符，必须以字母开头、以字母或数字结尾，不能是保留字 `all`，最多 20 个。原有 `oci-main` / `oci-backup` 也继续支持。它们必须与下面的 GitHub Environment 名称**完全一致**。**别名只是标识，不会自动设置实际 OCI 区域**；仍需在每个 Environment Secret `REGION` 填入正确的 Region ID。

### 2. 每个租户创建独立的 GitHub Environment

打开 Public 仓库 Settings → Environments → New environment。

例如分别创建两个独立 Environment（不能仅修改清单而不创建同名 Environment）：

- `ap-singapore-2-8`
- `ap-singapore-2-9`

两者分别配置下面的 **5 个 Environment Secrets**（不是 Repository Secrets，也不用另外在 Environment Variables 中重复添加）：

| Environment Secret | 用途 |
|---|---|
| `TENANCY` | 当前租户 Tenancy OCID（对应 OCI 官方 `tenancy`） |
| `USER` | 当前租户 API User OCID（对应 OCI 官方 `user`） |
| `FINGERPRINT` | 当前租户 API Key 指纹（对应 OCI 官方 `fingerprint`） |
| `OCI_PRIVATE_KEY` | 当前租户 OCI API 私钥 PEM | 
| `REGION` | 必填，例如 `ap-singapore-2` 或 `ap-tokyo-1`（按实际租户设置） |

如有特殊需求，再配置下面的 **可选 Environment Variables**（默认不需要填写）：

| Environment Variable | 用途 |
|---|---|
| `OCI_COMPARTMENT_OCID` | 可选，仅当实例位于子 Compartment 时设置；默认使用 `TENANCY`（根 Compartment） |
| `OCI_GUARD_COMPARTMENT_OCIDS` | 可选，额外需要统计的 compartment OCID，逗号分隔 |
| `OCI_INSTANCE_NAME` | 可选，默认 `oci-a1-hunter` |
| `OCI_AD_NAMES` | 可选，限制 AD 名称 |
| `OCI_A1_BUDGET_OCPUS` | 可选，每租户默认 **2** |
| `OCI_A1_BUDGET_MEMORY_GB` | 可选，每租户默认 **12** |

`preflight` 和经授权的 `live` **都可以不配置任何子网或镜像 OCID**。把所有租户共用的 `OCI_SSH_PUBLIC_KEY` 放在唯一的 **Repository Secret**（只添加一次）；程序会在通过库存/预算检查后用只读 API 找到唯一 AVAILABLE 的现有子网，并选择兼容的 Ubuntu 24.04 A1 镜像。需要固定子网或镜像时，再使用以下**可选** Environment Secrets：

| Environment Secret | 用途 |
|---|---|
| `OCI_SUBNET_OCID` | 可选。明确指定目标子网；缺省时只读发现，多个候选时必须手动指定 |
| `OCI_IMAGE_OCID` | 可选。固定 Image OCID；缺省时 `preflight` / `live` 自动选择最新 AVAILABLE Ubuntu 24.04 A1 镜像 |

上述租户专属资源 OCID 可用 Secrets 存放，是因为公开 Actions Job 日志可能显示普通 Variables 的值；这不会影响已成功验证的 5 个身份 Secrets。`check`、`audit`、`dry-run` 均不需要新增配置。如果没有共享 Repository Secret `OCI_SSH_PUBLIC_KEY`，且私有固定配置中也没有公钥，`preflight` / `live` 将拒绝执行。**正式 `live` 不再强制要求 `OCI_SUBNET_OCID` 或 `OCI_IMAGE_OCID`**；选择不唯一、查询失败或配置不兼容时会安全停止。依旧只需一个全局授权开关；尚无自动调度。

执行 `live` 前，只需要在 **Repository Variables** 中一次性设置 `OCI_HUNTER_LIVE_ENABLED=true`，再手动选择唯一目标租户；**不需要任何 Environment 审批变量**。如启用了 GitHub Environments 的 Required reviewers，仍须完成 GitHub 原生审批；结束后把全局开关改回非 `true`。

**迁移提醒**：Public 工作流现已统一读取每个 Environment 的 `USER`、`FINGERPRINT`、`TENANCY`、`OCI_PRIVATE_KEY` 及 `REGION`。此前使用的 `OCI_USER_OCID`、`OCI_FINGERPRINT`、`OCI_TENANCY_OCID`、`OCI_REGION` 是旧配置名，应迁移后删除，避免重复和混淆。尤其不要在 Repository Secrets 设置共享的 OCI 租户凭证。

### 3. 手动测试

进入 [OCI A1 Hunter Actions](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/workflows/hunter.yml) → Run workflow：

| mode | tenant | 作用 |
|---|---|---|
| `dry-run` | `all`（忽略此输入） | 离线运行 Private Python 单元测试并验证 CLI，不接触 OCI |
| `check` | `ap-singapore-2-8` | 只读验证主租户 OCI 凭证与现有 A1 用量 |
| `check` | `all` | 按 `OCI_TENANTS` 自动生成独立快速查询 Job，分别报告 |
| `audit` | `ap-singapore-2-8` | **只读遍历整个 Tenancy 的子 Compartment**，统计当前 Region 的所有可查询 A1 实例 |
| `audit` | `all` | 对已登记的每个租户单独进行可查询 Compartment 审计，不创建实例 |
| `preflight` | `all` | 用每个租户独立的 Environment Secrets，只读验证子网、镜像、SSH 公钥、A1 形状兼容性以及当前区域 A1 用量；缺配置则安全失败 |
| `preflight` | `ap-singapore-2-8` | 只诊断单个租户；不调用创建实例 API |
| `live` | `ap-singapore-2-8` | **可能创建实例**，受全局开关及单租户手动选择保护 |
| `live` | `all` | **拒绝执行**；禁止跨全部租户一键创建 |

工作流使用固定的 Private commit、GitHub-hosted Runner、`persist-credentials:false`、只读 `GITHUB_TOKEN` 和每租户独立并发锁。不使用自托管 OCI 主机，不将 OCI 凭证写入仓库/数据库/永久文件。

`check` / `audit` / `preflight` 如失败，新版核心会给出安全的阶段性原因（`OCI client initialization` / `OCI availability-domain query` / `OCI instances query`），并分类为 DNS、TCP 连接、TLS、连接超时、读取超时或 OCI HTTP 状态。该模式单次请求连接超时 5 秒、读取超时 20 秒，不打印原始请求异常、Endpoint 或 OCI 身份信息。`REGION` 仍为每个 Environment 原有的 Secret，**不需要修改或新增任何一个 OCI Secret**。建议先运行 `dry-run` 让离线测试通过，再运行 `audit` + `all` 核对可查询的租户树（仅使用已有 5 个 Secrets，无需增加 Compartment OCID）。确认共享 Repository Secret `OCI_SSH_PUBLIC_KEY` 后，运行 `preflight`（仍为只读）；不必再为每个租户准备子网或镜像 Secrets。

### 安全范围与限制

1. `check` 与 `audit` 只调用 OCI 查询接口；`live` 是不同的显式审批路径。检查前不给 API IAM 用户创建权限是推荐做法。
2. 默认使用 Tenancy OCID 作为根 Compartment OCID。`check` 只扫描根目录或显式配置的 Compartment；`audit` 使用 OCI `ListCompartments`（`compartment_id_in_subtree=True`、`access_level=ANY`）递归发现包括孙级在内的子 Compartment，逐一统计同一区域 A1；权限不足会失败，不会误报 0。**未来 `live` 也必须先完成相同的全树读取才允许创建。** 不自动跨 OCI Region 汇总；OCI 的免费权益与实际服务配额仍需到 Console 核验。
3. 不会停止、删除或调整现有实例；同名已有实例会阻止重复创建。
4. 2 OCPU / 12GB 是每个 Environment 默认预算；不要未经验证就扩大预算。OCI 实际服务限额与免费权益应以租户 Console 为准。
5. TG Bot/Chat ID 在 Repository Secrets 中共享配置一份，消息会注明租户**别名**，不要用真实账号标识作为公开别名。
6. 当前没有自动 cron。真实定时任务必须在完成所有租户的只读校验、IAM 最小权限和安全审查后单独启用。

下一步：请在最新版本上重新运行单租户 `preflight`，确认 OCI 读取权限、配额和兼容 AD，再在全局授权开关已启用、免费额度和潜在费用风险已确认的前提下手动运行同一租户 `live`。自动子网/镜像发现不需要添加新的租户 Secrets；仍无自动定时创建。

### OCI 官方配置字段映射

| OCI 官方配置 | GitHub Actions 配置 | 存放位置 |
|---|---|---|
| `user` | `USER` | Environment Secret |
| `fingerprint` | `FINGERPRINT` | Environment Secret |
| `tenancy` | `TENANCY` | Environment Secret |
| `region` | `REGION` | Environment Secret |
| `key_file` | `OCI_PRIVATE_KEY`（PEM 私钥全文，不是路径） | Environment Secret |

配置 OCI API 只需上述 **5 个 Environment Secrets**。GitHub 会将 Secrets/Variables 名称显示为大写，但引用不区分大小写。工作流将这些短名称映射为 Private Python 核心所需的内部环境变量。默认使用 `TENANCY` 作为根 Compartment ID；不要求额外 Compartment OCID。OCI SDK 使用内存中的 `key_content`，不把私钥写入文件。

### 2026-10-24 配置简化

私有核心提供可选 `TENANT_LAUNCH_DEFAULTS` 覆盖，但**不需要**为 `ap-singapore-2-8`、`ap-tokyo-1-2` 逐个填写子网或 SSH 公钥；统一公钥使用 Repository Secret。缺少子网/镜像 OCID 时，`preflight` 和经授权的 `live` 使用只读 API 寻找现有子网及最新 Ubuntu 24.04/A1.Flex 镜像，再检查 GetSubnet/GetImage/ListShapes；模糊或无兼容资源即安全失败。

### VCN / Subnet 自动复用（新）

`preflight` 和已授权的 `live` 未提供 `OCI_SUBNET_OCID` 时，会调用只读 OCI `ListVcns` / `ListSubnets` 扫描当前区域的审计范围。仅有一个 AVAILABLE VCN 下的 AVAILABLE 子网时自动复用；发现多个候选需明确选择；没有可读的候选就停止，不会创建网络。 `OCI_IMAGE_OCID` 也可留空，由预检或已授权的 `live` 自动发现 Ubuntu 24.04 A1 镜像。现阶段只需在 Repository Secrets 中设置一次 `OCI_SSH_PUBLIC_KEY` 即可尝试自动发现，不过网络多候选时仍需指定子网。正式 `live` 只强制要求公钥，子网/镜像支持安全的只读自动发现，并受一个 Repository 全局开关保护。无自动建网或定时任务。

**共享公钥唯一入口：** Public Runner → Settings → Secrets and variables → Actions → **Repository secrets** → `OCI_SSH_PUBLIC_KEY`。所有 Environment 的任务均引用同一个 `secrets.OCI_SSH_PUBLIC_KEY`。不要给每个租户重复添加；若之前某个 Environment 曾设置同名 Secret，建议先核对并移除那个旧值，因为 GitHub 同名 Environment Secret 会覆盖 Repository Secret。OCI API 认证的 TENANCY/USER/FINGERPRINT/OCI_PRIVATE_KEY/REGION 仍必须各租户隔离。
