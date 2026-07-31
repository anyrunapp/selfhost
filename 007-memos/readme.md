# 007-memos

[Memos](https://github.com/usememos/memos) 轻量卡片笔记 / 备忘,随手记灵感、待办、片段。
后端库用 **PostgreSQL**(非默认 sqlite)。端口段 **20070-20079**(由目录名 `007` 算出)。

| 变量     | 端口  | 用途                           |
| -------- | ----- | ------------------------------ |
| `PORT_0` | 20070 | Memos HTTP(仅 127.0.0.1 回环) |
| `PORT_1` | 20071 | PostgreSQL(仅 127.0.0.1 回环) |

## 为什么能用 PostgreSQL

Memos 原生支持 sqlite / mysql / postgres 三种驱动,通过环境变量选择:

```
MEMOS_DRIVER=postgres
MEMOS_DSN=postgres://user:pass@db:5432/memos?sslmode=disable
```

这里直接把默认 sqlite 换成 compose 内的 `db` 服务。笔记数据存 PostgreSQL,
附件 / 缩略图等文件仍落在数据目录 `./data/memos`。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填数据库密码
sed -i "s/CHANGE_ME/$(openssl rand -hex 24)/" .env
vim .env            # 改 MEMOS_INSTANCE_URL 为你的真实反代域名
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f memos
```

容器以 root 启动、entrypoint 自动把 `./data/memos` chown 给 `MEMOS_UID:GID`(默认 10001)
后再降权运行,bind mount **无需**手动 chown。

首次打开 `http://127.0.0.1:20070`,第一个注册的账号即成为管理员(host)。

## 连接 / 网络

- Memos 通过 compose 网络用服务名连库:`db:5432`,不依赖宿主端口。
- 两个端口都只绑 `127.0.0.1`,不对公网暴露;对外访问请在前面加 Caddy/Nginx 反代,
  并让 `MEMOS_INSTANCE_URL` 与反代域名一致。
- 本机维护库:`psql -h 127.0.0.1 -p 20071 -U memos -d memos`。

## 注意事项

- **首次启动才会建库**:`POSTGRES_DB/USER/PASSWORD` 只在 `data/postgres` 为空时生效。
- 数据库密码**只用字母数字**:它会拼进 `MEMOS_DSN` 的 URL,含特殊字符要 URL 编码,直接避开。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

```bash
# 备份(PostgreSQL 里的笔记数据)
docker compose exec -T db pg_dump -U memos memos | gzip > memos-$(date +%F).sql.gz

# 恢复
gunzip -c memos-YYYY-MM-DD.sql.gz | docker compose exec -T db psql -U memos -d memos

# 附件等文件另行打包
tar czf memos-data-$(date +%F).tar.gz data/memos
```

## 升级

```bash
vim .env                       # 改 MEMOS_TAG
docker compose pull && docker compose up -d
```

Memos 启动时自动迁移数据库结构。升级前先做上面的备份,并留意上游 release note。
