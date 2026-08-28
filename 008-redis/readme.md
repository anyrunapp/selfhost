# 008-redis

独立 Redis 7.4,给多个应用做缓存 / 队列 / 会话后端。端口段 **20080-20089**(由目录名 `008` 算出)。

| 变量     | 端口  | 用途                        |
| -------- | ----- | --------------------------- |
| `PORT_0` | 20080 | Redis(仅 127.0.0.1 回环)   |

> 说明:开了密码认证 + AOF 持久化 + 内存上限。纯缓存场景用 `allkeys-lru` 淘汰;
> 当队列或存重要数据时把 `REDIS_MAXMEMORY_POLICY` 改成 `noeviction`,写满报错而不丢数据。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填密码 + 按需改内存上限
sed -i "s/CHANGE_ME/$(openssl rand -hex 24)/" .env
vim .env            # 改 REDIS_MAXMEMORY / REDIS_MAXMEMORY_POLICY
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f redis
docker compose exec redis redis-cli -a "$(grep '^REDIS_PASSWORD=' .env | cut -d= -f2)" ping
```

## 连接

- **同机其它 compose 应用**:优先走容器网络。取消 `docker-compose.yaml` 里 `networks` 段的
  注释,应用侧也 join 外部网络 `selfhost-redis`,然后用 `redis://:密码@redis:6379/0` 连,
  别依赖宿主端口。
- **本机维护 / 调试**:`redis-cli -h 127.0.0.1 -p 20080 -a 你的密码`。
- 端口只绑 `127.0.0.1`,不对公网暴露;要给外部应用用,前面自己加隧道或内网。

## 多库(DB 编号)

Redis 默认 16 个逻辑库(`0`~`15`),不同应用用不同编号隔离即可,连接串末尾 `/N` 指定:
`redis://:密码@redis:6379/1`。需要更强隔离就再起一个本目录的副本(换序号前缀)。

## 注意事项

- **密码在命令行传入**:`--requirepass` 从 `.env` 注入;`.env` 记得 `chmod 600`,别提交。
- 密码**只用字母数字**,省掉 `$` 被 compose 插值和连接串 URL 编码的坑。
- **持久化**:开了 AOF(`appendfsync everysec`),数据落在 `./data/redis`。
  想要更快、可接受丢几秒数据可改 `no`;想彻底当纯缓存可关掉 `--appendonly`。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

```bash
# 备份:触发一次 RDB 快照,再打包 data 目录
docker compose exec redis redis-cli -a 你的密码 SAVE
docker compose stop
tar czf redis-$(date +%F).tar.gz data/
docker compose start

# 恢复
docker compose down
tar xzf redis-YYYY-MM-DD.tar.gz
docker compose up -d
```

## 升级

```bash
vim .env                       # 改 REDIS_TAG
docker compose pull && docker compose up -d
```

Redis 小版本(7.4.x)可直接换 tag 平滑升级;跨大版本先看上游 release 说明,
AOF/RDB 文件格式一般向后兼容,升级前先备份 `data/`。
