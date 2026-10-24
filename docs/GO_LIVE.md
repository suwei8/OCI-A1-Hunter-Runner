# OCI A1 Hunter · v1.0-RC1 生产试运行验收单

> 状态：**离线 CI 通过；两个租户存在成功的历史只读 preflight；真实 live 创建尚未验收**。
> 本文是受控试运行操作指南，**不是自动授权**。不得把 `preflight` 通过等同于 `live` 已成功。

## 已验证证据（2026-10-24 UTC）

| 检查 | 证据 | 结果 |
|---|---|---|
| 固定私有核心提交的离线测试 | [Actions #40079296287](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40079296287) | 73 passed；dry-run 成功；无 OCI API 请求 |
| 新核心 SHA | `a90da9ed77e1e5c957c30944da3f601f5d8ca301` | Runner 已固定该 SHA |
| 新 Runner SHA | `d1a21cd4fdae761cf8ae90382fc37e2e21fbea67` | CI 通过；429 返回非零退出码 75 |
| `ap-singapore-2-8` 历史只读预检 | [Actions #40070679114](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40070679114) | preflight 成功，1 个兼容 AD，A1 使用量 0/0，预算 2/12 |
| `ap-tokyo-1-2` 历史只读预检 | [Actions #40070838583](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40070838583) | preflight 成功，1 个兼容 AD，A1 使用量 0/0，预算 2/12 |

历史 preflight 早于最新固定 SHA，**正式试运行前应使用最新版本重新运行一次**。兼容 AD 不代表此时有空余宿主机容量，也不代表免费层权益已复核。

## 操作顺序（先单租户，后逐个推广）

1. [打开 Actions 工作流](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/workflows/hunter.yml) → **Run workflow** → 分支 `main` → mode=`preflight` → tenant=`ap-singapore-2-8` → Run workflow。此操作仅调用 OCI 查询 API，不请求创建。
2. 日志必须出现 `preflight: OCI API OK`、`target status=ready`、`compatible ADs` 大于 0；否则停在此步排障。不要把 `rate-limit`、`quota`、`error` 或作业绿色但未输出 `preflight:` 视为通过。
3. 在 OCI Console 核对该 Tenancy / Region 的 **VM.Standard.A1.Flex** 真实剩余额度、免费套餐或 PAYG 计费风险，以及 IAM **最小必要创建权限**。当前 CI 并不能证明 Create 权限已生效。
4. 为选定租户的 GitHub Environment Secrets 明确填入 `OCI_SUBNET_OCID` 和 `OCI_IMAGE_OCID`；仅 preflight 的自动发现**不会保存**这两个 OCID，也不会自动传给将来的 live。共享 SSH 公钥使用 Repository Secret `OCI_SSH_PUBLIC_KEY`；不要填写 SSH 私钥。当前两项 Environment Secrets 是否已填写无法从公开日志确认。
5. 对照 preflight 实际识别的区域、子网、镜像和 AD；如果启用低请求模式，需要先完成全租户 inventory 核对，并显式设置有效的 `OCI_AD_NAMES`。否则可保持低请求开关关闭，live 会执行保守的目录扫描。
6. **仅在所有检查合格且操作者明确批准创建实例后**：只需设置一次 Repository Variable `OCI_HUNTER_LIVE_ENABLED=true`，不需要逐个租户添加 `OCI_TENANT_LIVE_APPROVED`。选择 mode=`live`、tenant=`ap-singapore-2-8` 手动运行一次。**不要选择 all；不会自动定时重试。**
7. 在 Actions 日志查看 `created`（表示 OCI 接受创建请求，但仍可能正在 PROVISIONING），再去 OCI Console 验证实例最终状态、引导、网络和 SSH 访问；`capacity` 表示本轮无容量，可由操作者择时重新手动运行；`exists` 和 `quota` 需复核现有资源或免费额度；`rate-limit` 表示停止、退出 75，人工设置冷却期。
8. 试运行结束，立即将 Repository Variable `OCI_HUNTER_LIVE_ENABLED` 改回非 `true`，避免误操作。需要再次试运行时重新审批。

## 运行限制与保护

- 没有 `schedule`：这是**手动单轮抢机**，不是 24 小时无人值守自动抢机服务。
- `OCI_HUNTER_COOLDOWN_UNTIL` 是当前租户 Environment Variable，填写带时区的 UTC ISO8601 时间，例如 `2026-10-25T00:00:00Z`；超时前 live 被拒绝。现在**不会**在 HTTP 429 后自动保存冷却截止时间；必须人工设置并等待。不要在 429 后密集重新运行。
- GitHub-hosted runner 是临时环境；Public Actions 日志可公开查看。不要输出、提交或上传 OCI PEM、PAT、TG Token、SSH 私钥、完整 OCID 或请求异常原文。
- 一次只允许一个租户 live，租户 Job 有独立 concurrency 锁；不能保证跨 OCI 区域的额度统计或实例创建完成后的联网能力。创建后仍须核对资源状态。
- 任何复测若出现 401/403/429、错误镜像/子网、限额不明、日志泄露或未知额外费用风险，都应暂停 live 并先修复。

## 未完成项（不冒充生产能力）

- 已使用旧 Runner 版本完成过 `ap-singapore-2-8` 的真实 OCI preflight 复测：[Actions #40079808448](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/runs/40079808448)；更新后的简化审批 Runner 仍需单租户 live 受控试运行。
- HTTP 429 跨运行自动冷却持久化。
- 自动间隔调度和节流策略（需单独批准，不随 RC1 开启）。
- 创建后 PROVISIONING → RUNNING 的最终核验、网络连接和 SSH 验收。

请保留原始 [Actions 运行链接](https://github.com/suwei8/OCI-A1-Hunter-Runner/actions/workflows/hunter.yml) 作为验收证据，不要在公开 Issue 中粘贴密钥或租户敏感信息。
