# 003-openlist

[OpenList](https://github.com/OpenListTeam/OpenList)(AList 的社区分叉)文件列表 / 网盘挂载程序。
后端库用 **PostgreSQL**(非默认 sqlite)。端口段 **20030-20039**(由目录名 `003` 算出)。

| 变量     | 端口  | 用途                              |
| -------- | ----- | --------------------------------- |
| `PORT_0` | 20030 | OpenList HTTP(仅 127.0.0.1 回环) |
| `PORT_1` | 20031 | PostgreSQL(仅 127.0.0.1 回环)    |

## 为什么能用 PostgreSQL

官方镜像 entrypoint 是 `openlist server --no-prefix`,`--no-prefix` 表示配置项环境变量
**不带** `OPENLIST_` 前缀。启动时环境变量会覆盖 `data/config.json`,于是这里直接注入:

```
DB_TYPE=postgres  DB_HOST=db  DB_PORT=5432
DB_NAME/DB_USER/DB_PASS  DB_SSL_MODE=disable
```

即可把默认的 sqlite 换成 compose 内的 `db` 服务。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填数据库密码
sed -i "s/CHANGE_ME/$(openssl rand -hex 24)/" .env
vim .env            # 也可手改域名 SITE_URL / 库名 / 账号
chmod 600 .env

# 3. 建数据目录并改属主(关键,见下)
mkdir -p data/openlist data/postgres
sudo chown -R 1001:1001 data/openlist     # 对齐 .env 里的 OPENLIST_UID/GID

# 4. 起
docker compose up -d
docker compose logs -f openlist
```

## 数据目录权限(v4.1.0 起容器非 root 运行)

容器内以 `openlist` 用户(默认 `UID:GID=1001:1001`)运行,对 `/opt/openlist/data`
需要读写 + 执行权限。宿主的 `./data/openlist` 必须属于同一 UID,否则启动即报
`Current user does not have write and/or execute permissions`。

- 改属主:`sudo chown -R 1001:1001 data/openlist`
- 想换别的 UID,同时改 `.env` 的 `OPENLIST_UID/OPENLIST_GID` 和 chown 目标。

## 首次登录 / 管理员密码

首启会生成随机 admin 密码(打印在日志里)。也可手动设置:

```bash
# 随机生成并打印
docker compose exec openlist ./openlist admin random
# 或指定密码
docker compose exec openlist ./openlist admin set 'YOUR_PASSWORD'
```

登录地址:`http://127.0.0.1:20030`(前面自建反代 + TLS,对应 `SITE_URL`)。

## 连接 / 网络

- OpenList 通过 compose 网络用服务名连库:`db:5432`,不依赖宿主端口。
- 两个端口都只绑 `127.0.0.1`,不对公网暴露;对外访问请在前面加 Caddy/Nginx 反代。
- 本机维护库:`psql -h 127.0.0.1 -p 20031 -U openlist -d openlist`。

## 注意事项

- **首次启动才会建库**:`POSTGRES_DB/USER/PASSWORD` 只在 `data/postgres` 为空时生效。
- 数据库密码**只用字母数字**,省掉 `$` 被 compose 插值和 URL 编码的坑。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

```bash
# 备份(PostgreSQL)
docker compose exec -T db pg_dump -U openlist openlist | gzip > openlist-$(date +%F).sql.gz

# 恢复
gunzip -c openlist-YYYY-MM-DD.sql.gz | docker compose exec -T db psql -U openlist -d openlist

# 另建议一并备份 OpenList 数据目录(挂载配置、加密密钥等)
tar czf openlist-data-$(date +%F).tar.gz data/openlist
```

## 升级

```bash
vim .env                       # 改 OPENLIST_TAG
docker compose pull && docker compose up -d
```

跨大版本先看官方 release note,升级前先做上面的备份。
