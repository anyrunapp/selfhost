# 002-mysql

MySQL 8.0,给 taskhub 等应用做后端库。端口段 **20020-20029**(由目录名 `002` 算出)。

| 变量     | 端口  | 用途                        |
| -------- | ----- | --------------------------- |
| `PORT_0` | 20020 | MySQL(仅 127.0.0.1 回环)   |

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填密码(root + 业务账号)
sed -i "s/CHANGE_ME1/$(openssl rand -hex 24)/" .env   # 替换 root 的 密码
sed -i "s/CHANGE_ME2/$(openssl rand -hex 24)/" .env   # 替换 业务用户 的 密码
vim .env            # 也可分别手填两个不同密码、改库名/账号
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f mysql
docker compose exec mysql mysqladmin -uroot -p ping
```

> 注意:`sed` 那条会把 root 和业务账号设成同一个密码;要区分就手动改 `.env` 里的
> `MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD`。

## 连接

- **同机其它 compose 应用**:优先走容器网络,连 `mysql:3306`(用服务名),别依赖宿主端口。
- **本机维护 / 备份**:`mysql -h 127.0.0.1 -P 20020 -u taskhub -p`。
- 端口只绑 `127.0.0.1`,不对公网暴露;要给外部应用用,前面自己加隧道或内网。

## 注意事项

- **首次启动才会建库建账号**:`MYSQL_DATABASE/USER/PASSWORD` 只在 `data/mysql` 为空时生效。
  已经初始化过再改这些变量不会重建,得改 SQL 或清空数据目录。
- `POSTGRES_PASSWORD` 一类密码**只用字母数字**,省掉 `$` 被 compose 插值和 URL 编码的坑。
- 字符集固定 `utf8mb4` + `utf8mb4_0900_ai_ci`,时区 `+08:00` 和 `TZ` 对齐。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

```bash
# 备份
docker compose exec -T mysql mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" \
  --single-transaction --routines --triggers taskhub | gzip > taskhub-$(date +%F).sql.gz

# 恢复
gunzip -c taskhub-YYYY-MM-DD.sql.gz | docker compose exec -T mysql mysql -uroot -p taskhub
```

## 升级

```bash
vim .env                       # 改 MYSQL_TAG
docker compose pull && docker compose up -d
```

跨大版本(如 8.0 -> 8.4 / 9.x)先看官方升级说明,必要时 dump 再 restore,别直接换 tag。
