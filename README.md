# bewcloud-lzcapp

[bewCloud](https://github.com/bewcloud/bewcloud) 的懒猫微服打包：**只发喵喵商店，镜像模式**（`delivery.mode: mirror`）。

## 拓扑（三容器）

| 服务 | 镜像 | 说明 |
|---|---|---|
| website | `ghcr.io/bewcloud/bewcloud:v4.19.0` | 主应用（Deno），8000 端口；**版本源** |
| postgresql | `postgres:18.6` | PGDATA=`/var/lib/postgresql/18/docker`，bind 到 `/lzcapp/var/postgresql` |
| radicale | `tomsquest/docker-radicale:3.7.6.0` | CardDav/CalDav，仅容器网络内可达，不对外暴露 |

## 关键决策

- **`.env` 不落地**：上游 `lib/interfaces/database.ts` 等直接读容器环境变量（`@std/dotenv/load` 只是兜底），全部密钥用 `stable_secret` 注入 `environment`——website 与 postgresql 共用 `db_password`，天然一致。
- **`bewcloud.config.ts` 每次启动生成**：`setup_script`（官方语义：root 权限、原 entrypoint 之前执行）写出到 `/app`——`baseUrl` 用 `{{ .S.AppDomain }}`，`allowSignups` 接向导参数（默认 false），CardDav/CalDav 指向 `http://radicale:5232`。
- **数据库迁移自动化**：上游要手动 `make migrate-db`；setup_script 里等 PostgreSQL 可连接（最多 2 分钟）后以 deno 用户执行，幂等，升级自动补跑。
- **radicale 适配**：镜像自带 `TAKE_FILE_OWNERSHIP=true`（root 启动时 `chown -R radicale:radicale /data`，正好处理平台建的 root 属主目录）；配置用包内 `content/radicale-config/config` 以只读 bind 到 `/config`（`http_x_remote_user` 鉴权，信任来自 bewcloud 的 `X-Remote-User` 头）。
- **healthcheck**：website 用 `deno eval` 探测（deno ubuntu 镜像没有 curl/wget）；radicale 用 busybox wget。
- **资源**：website/postgresql 各 512M + shm 128M，radicale 256M（对齐上游 compose 的 256m 并放宽）。
- **向导参数**：`allow_signups`（bool，默认关）——只有首位注册者（管理员）能建号。

## 版本跟踪

`version_source` 指向 website 镜像（`ghcr.io/bewcloud/bewcloud`，`^v\d+\.\d+\.\d+$`）；postgres 与 radicale 的 tag 固定在 manifest，需要时手动升。

## 商店状态

- **喵喵商店**：`cloud.lazycat.app.bewcloud` 由 Action 发布（`stores.private`）。
- **官方商店**：镜像模式不满足官方商店「镜像必须来自 `registry.lazycat.cloud`」的要求，只发喵喵商店。
