# 004-freshrss

[FreshRSS](https://github.com/FreshRSS/FreshRSS) 自托管 RSS 聚合阅读器。
后端库用 **PostgreSQL**(非默认 SQLite)。端口段 **20040-20049**(由目录名 `004` 算出)。

| 变量     | 端口  | 用途                              |
| -------- | ----- | --------------------------------- |
| `PORT_0` | 20040 | FreshRSS HTTP(仅 127.0.0.1 回环) |
| `PORT_1` | 20041 | PostgreSQL(仅 127.0.0.1 回环)    |

## 为什么能用 PostgreSQL

官方镜像支持首启自动安装:环境变量 `FRESHRSS_INSTALL` 会调用 `cli/do-install.php`。
这里传入 `--db-type pgsql --db-host db --db-base/--db-user/--db-password`,把默认的
SQLite 换成 compose 内的 `db` 服务;`FRESHRSS_USER` 则首启自动建管理员。

> ⚠️ `FRESHRSS_INSTALL` / `FRESHRSS_USER` **只在第一次启动生效**。改了这些值不会重装,
> 需要先 `docker compose down -v` 删掉 `data/` 再来。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填密码(管理员 2 个 + 数据库 1 个,建议各不相同)
sed -i "s/CHANGE_ME1/$(openssl rand -hex 24)/" .env   # 管理员登录密码
sed -i "s/CHANGE_ME2/$(openssl rand -hex 24)/" .env   # 管理员 API 密码(移动端)
sed -i "s/CHANGE_ME3/$(openssl rand -hex 24)/" .env   # 数据库密码
vim .env            # 改 BASE_URL / 管理员用户名邮箱 / 语言等
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f freshrss
```

容器以 root 启动、entrypoint 自动修数据目录权限,`data/` 用 bind mount 无需手动 chown。
装完后用 `ADMIN_USER` / `ADMIN_PASSWORD` 登录 `http://127.0.0.1:20040`。

## 连接 / 网络

- FreshRSS 通过 compose 网络用服务名连库:`db:5432`,不依赖宿主端口。
- 两个端口都只绑 `127.0.0.1`,不对公网暴露;对外访问请在前面加 Caddy/Nginx 反代,
  并让 `BASE_URL` 与反代域名一致。
- 本机维护库:`psql -h 127.0.0.1 -p 20041 -U freshrss -d freshrss`。

## 注意事项

- **首次启动才会建库 + 装 FreshRSS + 建管理员**:`POSTGRES_*` 与 `FRESHRSS_INSTALL/USER`
  只在 `data/` 为空时生效。之后改变量不会重来。
- 密码**只用字母数字**,省掉 `$` 被 compose 插值和命令行参数解析的坑。
- `CRON_MIN` 控制订阅自动刷新的分钟位;留空则关闭内置 cron。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

```bash
# 备份(PostgreSQL)
docker compose exec -T db pg_dump -U freshrss freshrss | gzip > freshrss-$(date +%F).sql.gz

# 恢复
gunzip -c freshrss-YYYY-MM-DD.sql.gz | docker compose exec -T db psql -U freshrss -d freshrss

# 另建议备份 FreshRSS 数据目录(config.php、扩展等)
tar czf freshrss-data-$(date +%F).tar.gz data/freshrss data/extensions
```

## 升级

```bash
vim .env                       # 改 FRESHRSS_TAG
docker compose pull && docker compose up -d
```

FreshRSS 应用升级会自动迁移数据库结构。跨大版本 PostgreSQL(如 16 -> 17)不能直接换
`POSTGRES_TAG`,需先 dump 再在新版本 restore。
