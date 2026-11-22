# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

flux-panel（哆啦A梦转发面板）: a web panel that manages port/tunnel forwarding on remote nodes running a customized [gost](https://github.com/go-gost/gost). Supports per-user/per-tunnel forward quotas, TCP+UDP, two forward modes (端口转发 / 隧道转发), per-user-tunnel speed limits, and one-way/two-way traffic billing. Upstream (`bqlpfy/flux-panel`) has paused development; this is a fork being maintained independently. Code comments and UI strings are in Chinese.

## Components

| Dir | Stack | Role |
|---|---|---|
| `springboot-backend/` | Java 21, Spring Boot 2.7, MyBatis-Plus, MySQL 5.7 | Panel API on port 6365 |
| `vite-frontend/` | React + TypeScript + Vite + HeroUI + Tailwind | Panel UI, served by nginx in Docker |
| `go-gost/` | Go 1.23 | Node agent: gost with a vendored, modified `go-gost/x` (`replace github.com/go-gost/x => ./x`) |
| `android-app/`, `ios-app/` | Gradle / Xcode | Mobile shells (WebView) around the frontend |

Root files: `gost.sql` (initial schema + seed data, mounted as MySQL init script), `docker-compose-v4.yml` / `-v6.yml` (IPv4/IPv6 network variants), `panel_install.sh` (panel install/update/uninstall), `install.sh` (node install).

## Commands

Backend (`springboot-backend/`):
- Build: `mvn clean package -DskipTests` (what CI and the Dockerfile run)
- Run needs env vars: `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `JWT_SECRET`, `LOG_DIR`
- Only test is the default `AdminApplicationTests` context-load test (needs a DB); there is no real test suite.

Frontend (`vite-frontend/`):
- `npm install`, `npm run dev` (port 3000), `npm run build` (`tsc && vite build`), `npm run lint` (eslint **with --fix**)
- `VITE_API_BASE` sets the backend origin in dev; otherwise requests go to relative `/api/v1/`.

Node (`go-gost/`):
- `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o gost` (CI also builds arm64 and UPX-compresses)
- Reads `config.json` from the working directory (panel `addr`, `secret`, and http/tls/socks flags).

## Architecture

**Panel ↔ node control channel.** Nodes connect *outbound* to the panel via WebSocket at `/system-info` (`WebSocketConfig`, auth in `WebSocketInterceptor` using the node secret; payloads are AES-encrypted, see `AESCrypto`). The panel sends commands with `WebSocketServer.send_msg(nodeId, data, type)`, which tags each message with a `requestId` and blocks on a `CompletableFuture` until the node replies. On the node side, `go-gost/x/socket/websocket_reporter.go` `routeCommand` dispatches: `AddService/UpdateService/DeleteService/PauseService/ResumeService`, `AddChains/UpdateChains/DeleteChains`, `AddLimiters/UpdateLimiters/DeleteLimiters`, `TcpPing`, `SetProtocol`. A new node operation needs both a `GostUtil` method in the backend and a case in `routeCommand`.

**Node → panel HTTP.** Nodes POST traffic to `/flow/upload` and their full running gost config to `/flow/config` (`go-gost/x/service/traffic_reporter.go`, `FlowController`). `/flow/config` triggers `CheckGostConfigAsync`, which reconciles the node's actual config against the DB (removes orphaned services/chains/limiters, re-adds missing limiters). `/flow/**` is excluded from JWT auth and authenticated by the node secret query param.

**Gost config generation.** `GostUtil` builds gost JSON. Every forward becomes **two** gost services, `<name>_tcp` and `<name>_udp`, created/updated together. `UpdateService` sends the full service config and replaces the node's copy, so any field not passed (e.g. `limiter`) is dropped. Callers must always supply the complete desired state. Service name = `forwardId_userId_userTunnelId` (`ForwardServiceImpl.buildServiceName`; `userTunnelId` is 0 when none, e.g. admin-owned forwards). Tunnel forwards (`tunnel.type` tunnel mode) also create a chain on the in-node and a remote service on the out-node.

**Domain model.** `Node`, `Tunnel` (in-node/out-node, type, listen addrs, traffic billing), `Forward` (belongs to a user and tunnel), `UserTunnel` (a user's permission on a tunnel: quota, expiry, flow, `speedId`), `SpeedLimit` (maps to a gost limiter, limiter name = SpeedLimit id). Speed limits apply through `UserTunnel.speedId` → the forward's services' `limiter` field. Most forward logic lives in `ForwardServiceImpl` (permission/quota checks, port allocation, gost calls, rollback on gost failure).

**Backend conventions.** Controllers live under `/api/v1/*` and return `R` (`R.ok` / `R.err`; `code == 0` means success). `JwtInterceptor` guards `/api/**` (exclusions in `WebMvcConfig`). Admin-only endpoints use `@RequireRole` (`RoleAspect`). In services, admin is `roleId == ADMIN_ROLE_ID`. Admins bypass quota checks on their own forwards but are checked against the owning user's `UserTunnel` when editing a user's forward. Scheduled jobs: `ResetFlowAsync` (daily traffic reset), `StatisticsFlowAsync` (hourly stats).

**Frontend.** Pages in `src/pages/` map 1:1 to backend resources (forward, tunnel, node, limit, user, …). All API calls go through `src/api/` (axios, base `/api/v1/`). In production nginx (`vite-frontend/nginx.conf`) proxies `/api/v1/`, `/flow/*`, and `/system-info` to `backend:6365`, so the frontend's port is also the nodes' entry point. In the mobile WebView, the panel address is user-configured (`utils/panel.ts`).

## Schema changes

`gost.sql` only runs on a fresh MySQL volume. Existing installs are migrated by the SQL embedded in `panel_install.sh` `update_panel()`. Schema changes must update both.

## Release / CI

`.github/workflows/docker-build.yml` runs on pushes to `main`/`beta`. The version is the `VERSION` env at the top of that file. Images (`vite-frontend`, `springboot-backend`) are built only if a git tag for that version doesn't exist yet. The gost binary is rebuilt whenever `go-gost/` changed since the tag. Bump `VERSION` to publish. The create-release job tags the commit and uploads gost binaries, install scripts, compose files and `gost.sql` to the GitHub release. Images are pushed to GHCR (`IMAGE_PREFIX: ghcr.io/cotory`, using `GITHUB_TOKEN`). Install scripts and the panel's node-install command (`NodeServiceImpl`) download from this fork's releases. When bumping `VERSION`, also update the same hardcoded version string in the compose files, `install.sh`, `panel_install.sh` and `NodeServiceImpl`, and `vite-frontend/src/config/site.ts`.

Default admin account after install: `admin_user` / `admin_user`.
