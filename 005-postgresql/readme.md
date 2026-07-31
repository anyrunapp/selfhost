# 005-postgresql

独立 PostgreSQL 17,给多个应用做后端库。端口段 **20050-20059**(由目录名 `005` 算出)。

| 变量     | 端口  | 用途                             |
| -------- | ----- | -------------------------------- |
| `PORT_0` | 20050 | PostgreSQL(仅 127.0.0.1 回环)   |

> 说明:很多应用(如 003-openlist、004-freshrss)已在各自目录内自带 postgres,彼此隔离、
> 好备份好升级,是**推荐做法**。这个独立实例适合你想集中管理、多个轻量应用共用一套库的场景。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填密码 + 改库名/账号
sed -i "s/CHANGE_ME/$(openssl rand -hex 24)/" .env
vim .env            # 改 POSTGRES_DB / POSTGRES_USER 等
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f postgresql
docker compose exec postgresql pg_isready
```

## 连接

- **同机其它 compose 应用**:优先走容器网络。取消 `docker-compose.yaml` 里 `networks` 段的
  注释,应用侧也 join 外部网络 `selfhost-postgresql`,然后用服务名 `postgresql:5432` 连,
  别依赖宿主端口。
- **本机维护 / 备份**:`psql -h 127.0.0.1 -p 20050 -U appname -d appname`。
- 端口只绑 `127.0.0.1`,不对公网暴露;要给外部应用用,前面自己加隧道或内网。

## 多库 / 多账号

`POSTGRES_DB/USER` 只建**一个**库和账号。要给别的应用再开库,手动执行:

```bash
docker compose exec -it postgresql psql -U appname -d postgres
```
```sql
CREATE USER app2 WITH PASSWORD 'xxxx';
CREATE DATABASE app2 OWNER app2;
```

## 注意事项

- **首次启动才会建库建账号**:`POSTGRES_DB/USER/PASSWORD` 只在 `data/postgres` 为空时生效。
  已初始化过再改这些变量不会重建,得用 SQL 改或清空数据目录。
- 密码**只用字母数字**,省掉 `$` 被 compose 插值和 URL 编码的坑。
- 数据库时区 `PG_TIMEZONE` 和容器 `TZ` 对齐;`--data-checksums` 已默认开启。
- `PGDATA` 指到卷内子目录 `pgdata`,升级迁移更干净。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

```bash
# 单库备份
docker compose exec -T postgresql pg_dump -U appname appname | gzip > appname-$(date +%F).sql.gz

# 整实例备份(所有库 + 角色)
docker compose exec -T postgresql pg_dumpall -U appname | gzip > all-$(date +%F).sql.gz

# 恢复(单库)
gunzip -c appname-YYYY-MM-DD.sql.gz | docker compose exec -T postgresql psql -U appname -d appname
```

## 升级

```bash
vim .env                       # 改 POSTGRES_TAG
docker compose pull && docker compose up -d
```

⚠️ 跨大版本(如 17 -> 18)数据目录格式不兼容,**不能**直接换 tag:
先用旧版本 `pg_dumpall` 导出,起新版本空实例再 restore(或用 `pgautoupgrade` 镜像)。
小版本(17.x)可直接换 tag 平滑升级。
