# selfhost

自托管应用的 docker compose 集合。每个应用一个目录,命名固定为 `NNN-appname`。

## 端口约定

端口不手动分配,由目录名前缀算出来:

```
端口段 = PORT_BASE(20000) + 序号 * PORT_STRIDE(10)

001-vaultwarden -> 20010 ~ 20019
002-gitea       -> 20020 ~ 20029
013-immich      -> 20130 ~ 20139
```

每个应用固定拿到 10 个连续端口 `PORT_0` ~ `PORT_9`,`PORT_0` 约定为对外主端口,其余按需使用。
新增应用只要序号不重复,端口就永远不会撞。

`PORT_BASE` / `PORT_STRIDE` 可用环境变量覆盖。

## bin/ports.sh

```bash
bin/ports.sh 001-vaultwarden          # 打印 KEY=VALUE
bin/ports.sh --export 001-vaultwarden # eval $(bin/ports.sh --export .)
bin/ports.sh --json   001-vaultwarden
bin/ports.sh --write  001-vaultwarden # 生成/刷新 .env 里的端口块(幂等)
bin/ports.sh --check                  # 全仓库扫描:序号重复、端口越界、占用提示
```

`--write` 会在 `.env` 不存在时先从 `.env.example` 复制,然后追加/替换一个带标记的生成块,
标记外的内容(密码等)不会被动到。

## 新增一个应用

```bash
mkdir -p 002-gitea && cd 002-gitea
# 写 docker-compose.yml,端口一律用 ${PORT_0} ${PORT_1} ...
# 写 .env.example
../bin/ports.sh --write .
docker compose up -d
```

## Git 忽略

数据目录和密钥不入库,根 `.gitignore` 已统一处理:

```
*/data/
*/.env
*/secrets.env
```

只提交 `docker-compose.yml`、`.env.example`、`secrets.env.example`、`README.md`。
