<!-- delegate-outcome: pending -->

# SDK 65 部署进度（card/sdk65-deploy-n305-261004）

- 派发：`dlg-20261004-111840-b18694`
- Issue：#166（本卡不关）
- 固定主干：`ffbddbd10042b8f56f81af711252aa1a257d3a23`（PR #168 MERGED）
- 锁定决策：只换 API 镜像；SDK pin `b0818dc7859d1d8100e42f5c70cb75d34da422f7`；
  server/config/users/env/compose 零更改；凭据不轮换。

## 里程碑

- [x] M0 现场核查（2026-10-04T11:21Z）：worktree 干净，`git log -1` = `ffbddbd1`，`git diff --stat HEAD` 空。
- [x] M0.1 确认未接 D3 自动部署：本仓 `.github/workflows/` 只有 `gate.yml` / `gate-disposition.yml` /
      `gate-shadow.yml`（无 build/push/deploy job）；gate-hub `runner/fleet-manifest.json` 只把本仓列在
      `managed_callers`（gate-v2 调用者），`.github/workflows/gate.yml` 只 `uses:` gate-v2
      （quality/primary/aggregator/notify，无部署）；n305 `crontab -l` 无本项目部署条目；
      生产 compose 用 `image: ${VIDEO_TRANSCRIPT_IMAGE:?...}` 钉 digest，无自动拉取。
- [x] M0.2 升级前生产取证（n305）：
      - 容器 `video-transcript-api`：`Image=ghcr.io/zj1123581321/video-transcript-api@sha256:950a1600da44a341b96886dc47e30585b97c227aebbac6f411307a8353ed0487`，
        `ImageID=sha256:ca46d3a86cebe92684b2cd1ac7d6f1e311ec51890381160c059b08bf46ab8374`，
        `StartedAt=2026-10-03T16:11:07.973881807Z`，`Health=healthy`，`RestartCount=0`。
      - 端口 `8200->8000`，挂载 `config/`、`data/`。
      - `.deploy-image` = 同一 digest `sha256:950a1600…`。
      - 容器内 `GIT_SHA=ff92a175243d`（旧 pin 构建）。
      - 容器内 SDK `client.py` SHA-256 = `ff476ad7cd40401b7ed77c7606cb4fc14dc3d529043c29c4714b199c13423cdf`（**旧 pin，缺陷在生产在线**），
        Python 3.11.17，websockets 15.0.1。
      - CapsWriter 服务端（mac-studio，只读）：PID 22665，`start_server.py`，启动于 Thu Oct 1 18:36:04 2026，
        服务仓 HEAD `6b7a2b82fbc3ebe862250a8804902e5bf37f9211`，`/health`=200。部署前后必须不变。
      - 四文件升级前 SHA-256：
        - `docker-compose.yml` `7b0a89b50de84f71c480c8ef7398d64d479bbaf429805b9401d748294e498188`
        - `config/config.jsonc` `a349b81837e0cb06e2ce3dd8ff670df7e4c107b00e911a8b60670a8a334c25cc`
        - `config/users.json` `237cefb5a2d3bda3ea4e6a227750c348f34475cba1dd0fa804c82458cf28f7ad`
        - `.env` `5885382b22606aafaa4aeb55d529d56739c6555347286183dcae4ddf577b3b1e`
      - 服务器 `docker/pull_and_deploy.sh` SHA-256 = `1d98c3eb9a2f5c6e4e4714b5c54d0915645ff1ae08a834ef7f065eb81d1d5db1`，
        与本 worktree 仓内同名脚本逐字节相同 → **无需同步，无 CRLF 问题**。