# OCI A1 Hunter · v1.0-RC1 生产试运行验收单

> 状态：**离线 CI 通过；preflight 全绿；live 链路已实证走通（OCI 返回真实容量裁决 `capacity`）；尚未捕获 A1 空闲宿主机容量完成实例创建**。
> 本文是受控试运行操作指南，**不是自动授权**。不得把 `preflight` 通过等同于 `live` 已成功。

## 已验证证据（2026-10-24 UTC）

| 检查 | 证据 | 结果 |
|---|---|---|
| 固定私有核心提交的离线测试 | [Actions #40082456051](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40082456051) | **82 passed**；dry-run 成功；无 OCI API 请求 |
| 新核心 SHA | `235df23b90553929df02e01423d05261a1a10c68` | Runner 已固定该 SHA，支持 live 子网/镜像自动发现 |
| 已验证的 Runner SHA | `8962a03cca5256ec53eb5aa9b17ac46360229bab` | CI 通过；HTTP 429 返回非零退出码 75 |
| `ap-singapore-2-8` 历史只读预检 | [Actions #40070679114](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40070679114) | preflight 成功，1 个兼容 AD，A1 使用量 0/0，预算 2/12 |
| `ap-tokyo-1-2` 历史只读预检 | [Actions #40070838583](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40070838583) | preflight 成功，1 个兼容 AD，A1 使用量 0/0，预算 2/12 |

[2026-10-24 的最新版本变更之前，`ap-singapore-2-8` 已有成功真实预检 #40079808448](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40079808448)。本次新增自动发现 live 代码后，**正式试运行前建议再运行一次最新 `preflight`**。兼容 AD 不代表此时有空余宿主机容量，也不代表免费层权益已复核。

## 操作顺序（先单租户，后逐个推广）

1. [打开 Actions 工作流](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/workflows/hunter.yml) → **Run workflow** → 分支 `main` → mode=`preflight` → tenant=`ap-singapore-2-8` → Run workflow。此操作仅调用 OCI 查询 API，不请求创建。
2. 日志必须出现 `preflight: OCI API OK`、`target status=ready`、`compatible ADs` 大于 0；否则停在此步排障。不要把 `rate-limit`、`quota`、`error` 或作业绿色但未输出 `preflight:` 视为通过。
3. 在 OCI Console 核对该 Tenancy / Region 的 **VM.Standard.A1.Flex** 真实剩余额度、免费套餐或 PAYG 计费风险，以及 IAM **最小必要创建权限**。当前 CI 并不能证明 Create 权限已生效。
4. **无需再填写** `OCI_SUBNET_OCID` 和 `OCI_IMAGE_OCID`：更新后的 `live` 将自动执行只读子网/Ubuntu 镜像发现，并筛选 A1 兼容 AD。已配置的值继续优先使用。共享 SSH 公钥只需 Repository Secret `OCI_SSH_PUBLIC_KEY`（不能是 SSH 私钥）；若存在多个子网，则必须额外指定一个 `OCI_SUBNET_OCID`。自动发现增加 OCI 只读 API 调用。
5. 确认最新 `preflight` 日志中的兼容 AD、已用量和预算。如果启用低请求模式，仍须先确认全租户资源库存并固定 `OCI_AD_NAMES`；否则保持低请求模式关闭，`live` 会保守扫描全租户 Compartment。**自动发现镜像会选最新可用版本**；若必须固定镜像，使用可选 `OCI_IMAGE_OCID`。
6. **仅在所有检查合格且操作者明确批准创建实例后**：只需设置一次 Repository Variable `OCI_HUNTER_LIVE_ENABLED=true`，不需要逐个租户添加 `OCI_TENANT_LIVE_APPROVED`。选择 mode=`live`、tenant=`ap-singapore-2-8` 手动运行一次。**不要选择 all；不会自动定时重试。**
7. 在 Actions 日志查看 `created`（表示 OCI 接受创建请求，但仍可能正在 PROVISIONING），再去 OCI Console 验证实例最终状态、引导、网络和 SSH 访问；`capacity` 表示本轮无容量，可由操作者择时重新手动运行；`exists` 和 `quota` 需复核现有资源或免费额度；`rate-limit` 表示停止、退出 75，人工设置冷却期。
8. 试运行结束，立即将 Repository Variable `OCI_HUNTER_LIVE_ENABLED` 改回非 `true`，避免误操作。需要再次试运行时重新审批。

## 运行限制与保护

- 没有 `schedule`：这是**手动单轮抢机**，不是 24 小时无人值守自动抢机服务。
- `OCI_HUNTER_COOLDOWN_UNTIL` 是当前租户 Environment Variable，填写带时区的 UTC ISO8601 时间，例如 `2026-10-25T00:00:00Z`；超时前 live 被拒绝。现在**不会**在 HTTP 429 后自动保存冷却截止时间；必须人工设置并等待。不要在 429 后密集重新运行。
- GitHub-hosted runner 是临时环境；Public Actions 日志可公开查看。不要输出、提交或上传 OCI PEM、PAT、TG Token、SSH 私钥、完整 OCID 或请求异常原文。
- 一次只允许一个租户 live，租户 Job 有独立 concurrency 锁；不能保证跨 OCI 区域的额度统计或实例创建完成后的联网能力。创建后仍须核对资源状态。
- 任何复测若出现 401/403/429、错误镜像/子网、限额不明、日志泄露或未知额外费用风险，都应暂停 live 并先修复。

## 已确诊故障：`HTTP 400 CannotParseRequest`（2026-10-27）

两个租户（`ap-singapore-2-8`、`ap-tokyo-1-2`）的 live 运行 [#40482032994](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40482032994)、[#40483617489](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40483617489) 均在 LaunchInstance 返回 `400/CannotParseRequest`，而 payload 经真实 OCI SDK 序列化验证结构完全合法、preflight 全绿。

**根因（已实证）**：环境 Secret `OCI_TENANCY_OCID`（以及 `OCI_USER_OCID`、`OCI_FINGERPRINT`）值末尾带有一个换行符 `\n`（典型成因：`echo "..." | gh secret set`）。日志 env 块中这三个变量后各有一行空白即为证据。读路径经 `_auth_scalar` 统一 `strip()`，故全部只读 API 正常；但启动 payload 的 `compartmentId` 直接使用未 `strip` 的原始值，`"ocid1.tenancy.oc1..xxxx\n"` 到达 OCI 后被报为 `CannotParseRequest`。

**修复与验证**：
- 私有核心 `cf189f3`：`build_launch_details` 对全部 env 派生字段执行 `strip()`，wire 校验器对 OCID 增加严格 `fullmatch`，污染的 OCID 今后在本地即报 `LaunchPayloadSchemaError`，不再到达 OCI。同提交还带有 4 个有界重试变体与 2 个零创建探针，用于未来同类不透明 400 的定位。
- 实证证据：[push #40629890896](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40629890896) **105 passed**；[preflight #40629985459](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40629985459) 全绿；[live #40630095103](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40630095103) 在 Secret 仍脏的情况下返回 `capacity`（OCI 正常解析并给出真实宿主机容量裁决）——`CannotParseRequest` 消除，根因实证。
- 运维侧（可选卫生项）：重新设置 `OCI_TENANCY_OCID` 环境 Secret（值结尾不要换行；推荐 `gh secret set --body` 或在网页 UI 中粘贴）。`OCI_USER_OCID`/`OCI_FINGERPRINT` 同样建议清理，虽然当前仅用于已 strip 的签名配置。

## 循环抢机（操作者批准的外部循环，2026-10-29）

仓库内仍**不启用 `schedule:`**；循环由运维侧外部 cron 分发器实现（`~/dev_root/a1-hunt/hunt.sh`，每 6 分钟一个 tick），通过 `workflow_dispatch` 对 `ap-singapore-2-8` 单租户触发 `live`。行为：

- 仅当上一次 dispatch 已 completed 才发下一轮；不叠加（另有 per-tenant 并发锁兜底）
- `capacity` → 继续抢；`created`/`exists`/`quota`/`error` → 写 DONE 文件永久停止；`rate-limit` → 冷却 45 分钟后恢复
- 暂停方式：`OCI_HUNTER_LIVE_ENABLED` 改回非 `true`（live 在任何 OCI 调用前自拒），或删除 cron 行
- `created` 由工作流自身 Telegram 通知；成功后受同名实例与预算检查双重保护，不会重复创建

## 未完成项（不冒充生产能力）

- 原版 `ap-singapore-2-8` 真实 OCI preflight 成功：[Actions #40079808448](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40079808448)。此前首次 live 已通过全局授权但因缺少两个 OCID 停止：[Actions #40081317458](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40081317458)。更新后的自动发现版本已离线验证，但**尚未真实 live 验证实例创建**。
- HTTP 429 跨运行自动冷却持久化。
- 自动间隔调度和节流策略（需单独批准，不随 RC1 开启）。
- 创建后 PROVISIONING → RUNNING 的最终核验、网络连接和 SSH 验收。

请保留原始 [Actions 运行链接](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/workflows/hunter.yml) 作为验收证据，不要在公开 Issue 中粘贴密钥或租户敏感信息。
