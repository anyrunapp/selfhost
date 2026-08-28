# 010-watchtower

[Watchtower](https://github.com/containrrr/watchtower) 自动更新 docker 容器镜像:定时检查
被监控容器的镜像是否有新版本,有就拉取并重建容器。端口段 **20100-20109**(纯守护进程,实际不占端口)。

## 设计:按标签选择(opt-in),不乱动全部容器

本仓库约定「镜像锁版本」。所以这里默认开 `WATCHTOWER_LABEL_ENABLE=true`,**只更新你显式
打了标签的容器**,其余一律不碰。要不要自动更新、给哪些容器,由你决定。

## 部署

```bash
# 1. 生成 .env
../bin/ports.sh --write .

# 2. 按需改计划 / 通知
vim .env            # WATCHTOWER_SCHEDULE 等
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f
```

## 让某个容器吃自动更新(两步)

**① 给它加标签**(在那个应用的 `docker-compose.yaml` 对应 service 下):

```yaml
    labels:
      com.centurylinklabs.watchtower.enable: "true"
```

**② 给它用「滚动 tag」**,否则没意义:Watchtower 是「同一个 tag 有没有新 digest」才更新。
锁死的 `postgres:17.10-alpine` 基本不会变;想吃更新就用 `caddy:2-alpine`、`redis:7-alpine`
这类跟随小版本的 tag。改完 `docker compose up -d` 生效。

> 权衡:自动更新 vs 锁版本是矛盾的。稳定优先就别打标签(手动升级);
> 省心优先就给非关键、向后兼容好的服务打标签 + 滚动 tag。数据库这类建议**手动**升级。

## 常用操作

```bash
# 立刻检查一次并退出(不等定时),看看会更新啥
docker compose run --rm watchtower --run-once

# 只看不更新(排查用)
docker compose run --rm watchtower --run-once --monitor-only

# 看日志
docker compose logs -f
```

## 计划(cron)

`WATCHTOWER_SCHEDULE` 是 6 段 cron(秒 分 时 日 月 周):

- 每天 04:00:`0 0 4 * * *`(默认)
- 每 6 小时:`0 0 */6 * * *`
- 每周日 03:30:`0 30 3 * * 0`

## 通知(可选)

取消 compose 里 `WATCHTOWER_NOTIFICATION_URL` 注释,填
[shoutrrr](https://containrrr.dev/shoutrrr/) URL(telegram / bark / 邮件 / discord 等),
更新完会推送结果。

## 注意事项

- **安全**:挂了 `/var/run/docker.sock`,等同宿主 root 权限。别再对外暴露本容器,
  socket 也别给不可信的东西。
- **纯守护进程**:无对外端口;官方镜像 `FROM scratch`(无 shell),故未配 healthcheck,
  存活看 `docker compose ps` 和日志。
- 指标 API 默认关;要用就在 compose 里打开两个 `WATCHTOWER_HTTP_API_*` 并放开 `ports`
  (只绑 `127.0.0.1:${PORT_0}`)。
- `.env` 已被 git 忽略,不要提交。

## 升级(它自己)

```bash
vim .env                       # 改 WATCHTOWER_TAG
docker compose pull && docker compose up -d
```
