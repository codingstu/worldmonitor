# Fork 同步与部署最终方案（codingstu/worldmonitor）

## 1. 当前状态

- 你的仓库已在本地克隆到当前工作区。
- 已新增自动同步工作流：`.github/workflows/sync-upstream.yml`。
- 本地 Git 已配置上游远端：`upstream -> https://github.com/koala73/worldmonitor.git`。

---

## 2. 目标

1. 让 fork（`codingstu/worldmonitor`）持续跟进上游（`koala73/worldmonitor`）更新。
2. 让 Web 与桌面构建具备可重复、可回滚、可观测的部署流程。
3. 将敏感配置（API keys / Relay secret / Redis / Convex）全部托管在平台 Secret，不入库。

---

## 3. 上游自动同步方案（已落地）

### 3.1 同步机制

已实现工作流：`sync-upstream.yml`

- 触发方式：
  - 手动触发（`workflow_dispatch`）
  - 定时触发（每 6 小时）
- 行为：
  1. checkout `main`
  2. 拉取 `upstream/main`
  3. 自动 merge 到临时分支
  4. 若有变更则自动创建 PR 到 `main`
  5. 若冲突则退出并提示手工处理

### 3.2 冲突处理策略

当自动 merge 冲突时，按以下优先级处理：

1. 保留你自己的部署/域名/密钥相关改动
2. 对功能代码优先合并上游实现
3. 冲突 PR 必须通过 typecheck 与关键测试后再合并

建议建立标签：`upstream`、`sync`、`needs-manual-merge`。

---

## 4. 部署架构（推荐生产方案）

### 4.1 Web 与 API（Vercel）

- 前端 + `api/` 路由统一部署到 Vercel（项目已有 `vercel.json`）
- 分环境：
  - Production：主域名
  - Preview：PR 预览
  - Development：本地/测试

### 4.2 Relay（自有 VPS）

- 部署入口：`node scripts/ais-relay.cjs`
- 用于 AIS/OpenSky/Telegram/OREF 等长连接或中继场景
- Vercel 侧通过 `WS_RELAY_URL` 调用你的 VPS
- 建议在 VPS 上使用 `systemd` 守护进程 + Nginx 反向代理 + HTTPS 证书

### 4.3 缓存与注册

- Upstash Redis：跨用户缓存与限流（`UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN`）
- Convex：注册数据（`CONVEX_URL`）

---

## 5. 环境变量分层（最小可用）

### 5.1 Vercel 必填（建议）

- `WS_RELAY_URL`
- `RELAY_SHARED_SECRET`
- `UPSTASH_REDIS_REST_URL`
- `UPSTASH_REDIS_REST_TOKEN`
- `CONVEX_URL`
- `WORLDMONITOR_VALID_KEYS`

可选按业务开启：`GROQ_API_KEY`、`OPENROUTER_API_KEY`、`FINNHUB_API_KEY` 等。

### 5.2 VPS Relay 必填

- `AISSTREAM_API_KEY`
- `RELAY_SHARED_SECRET`（必须与 Vercel 一致）
- `RELAY_AUTH_HEADER`（默认 `x-relay-key`）
- `PORT`（如 `3004`）
- （可选）`UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN`

### 5.3 前端构建变量

- `VITE_VARIANT`（full/tech/finance/happy）
- `VITE_WS_API_URL`（桌面或跨域 API 场景）
- `VITE_WS_RELAY_URL`（本地开发回退场景）

---

## 6. CI/CD 与发布流程

### 6.1 代码合并流

1. 上游同步 PR（机器人）
2. 运行检查：typecheck、lint、核心测试
3. 人工审核后合并到 `main`
4. Vercel 自动生产部署

### 6.2 桌面发布流

- 使用现有工作流：`.github/workflows/build-desktop.yml`
- 支持多平台矩阵构建与发布
- macOS 签名凭证可选（缺失时自动 unsigned）

### 6.3 回滚策略

- Web：Vercel 回滚到上一个 deployment
- 桌面：回滚到上一个 release tag
- 同步异常：回滚合并 PR 或 revert merge commit

---

## 7. 安全与治理

1. 所有 Secrets 仅存平台环境变量，不写入仓库。
2. `RELAY_SHARED_SECRET` 定期轮换，Vercel 与 VPS 同步更新。
3. 对自动同步 PR 启用保护分支与必须检查项。
4. 对高风险目录（`api/`、`server/`、`src-tauri/`）保留人工审批。

---

## 8. 你现在可以直接执行的落地顺序

1. 在 GitHub 仓库启用 Actions，并确认 `sync-upstream.yml` 可运行。
2. 在 Vercel 配置生产项目与环境变量。
3. 在 VPS 部署 `scripts/ais-relay.cjs` 并配置密钥。
4. 配置 Nginx + HTTPS，并将域名（如 `relay.yourdomain.com`）反代到 `127.0.0.1:3004`。
5. 在 Vercel 设置 `WS_RELAY_URL=https://relay.yourdomain.com`，并与 VPS 保持同一 `RELAY_SHARED_SECRET`。
6. 手动触发一次 Sync Upstream 工作流，验证自动 PR。
7. 合并 PR 后观察 Vercel Production 是否成功。
8. 视需要触发桌面构建工作流，产出 release 资产。

---

## 9. 验收标准

- 定时任务可自动生成 upstream 同步 PR。
- `main` 合并后 Vercel 自动部署成功。
- Relay 可被 `/api/ais-snapshot`、`/api/opensky` 路径正常访问。
- 关键页面可加载（地图、新闻、关键面板）。
- 出现问题可在 15 分钟内回滚到上一稳定版本。