# 111-rustdesk

[RustDesk](https://github.com/rustdesk/rustdesk-server) 自建远程桌面服务端(开源 OSS 版)。
自己掌控 ID/中继服务器,不再依赖官方公共服务器。

> **端口故意选了前缀 `111`**:`base = 20000 + 111*10 = 21110`,算出的端口段
> **21110-21119** 正好覆盖 RustDesk 原生端口 21115-21119。于是 `PORT_5..9` 与容器端口
> 一一对齐(宿主口 = 容器口),客户端直接填默认 21116 即可;同时把整段占住,
> 防止以后别的应用占用到这些端口。

服务端由两个进程组成,这里拆成两个容器共享同一份数据/密钥:

- **hbbs**(ID/Rendezvous):设备注册、心跳、打洞、NAT 探测、web 客户端。
- **hbbr**(Relay 中继):打洞失败时转发音视频流量。

## 端口

前缀 `111` 让端口段与 RustDesk 原生端口对齐,宿主口 = 容器口:

| 变量     | 端口  | 容器端口     | 服务 | 用途                         |
| -------- | ----- | ------------ | ---- | ---------------------------- |
| `PORT_5` | 21115 | 21115/tcp    | hbbs | NAT 类型探测                 |
| `PORT_6` | 21116 | 21116/tcp+udp| hbbs | ID 注册/心跳/打洞(**主端口**) |
| `PORT_7` | 21117 | 21117/tcp    | hbbr | 中继                         |
| `PORT_8` | 21118 | 21118/tcp    | hbbs | web 客户端 websocket         |
| `PORT_9` | 21119 | 21119/tcp    | hbbr | web 客户端 websocket         |

`PORT_0..4`(21110-21114)预留不用,仅为占住整段。客户端 **ID Server** 直接填
`你的地址`(默认端口即 21116)或 `你的地址:21116`。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 按需修改 .env
vim .env
#    - 对公网提供服务:BIND_ADDR=0.0.0.0,并在防火墙放行 21115-21119(TCP)+ 21116(UDP)
#    - 宿主口=容器口,默认无需 -r;如需强制公钥校验可加 RUSTDESK_HBBS_OPTS=-k _
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f
```

首次启动会在 `./data/` 生成一对密钥 `id_ed25519` / `id_ed25519.pub` 和 sqlite 库
`db_v2.sqlite3`。两个容器共用 `./data`,务必让它们拿到同一份密钥。

## 客户端配置

在 RustDesk 客户端「设置 → 网络 / ID/中继服务器」填写:

- **ID Server**:`你的地址`(默认 21116)或 `你的地址:21116`
- **Relay Server**:`你的地址:21117`(可留空,由 hbbs 下发)
- **Key**:`./data/id_ed25519.pub` 的内容(公钥),用于校验服务器身份。

拿公钥:

```bash
cat data/id_ed25519.pub
```

强制客户端校验公钥可加 `RUSTDESK_HBBS_OPTS=-k _`(推荐,防中间人)。

## 网络 / 暴露

- 默认所有端口只绑 `127.0.0.1`(遵循仓库“仅本机 + 前置代理”约定),此时只能本机自测。
- RustDesk 走裸 TCP/UDP,**不能**用普通 HTTP 反代(Caddy/Nginx 的 http 块)转发。
  对公网提供服务推荐 `BIND_ADDR=0.0.0.0` 直接对外(配合防火墙);若用四层(L4)
  转发,请保持端口不变(21115-21119),否则打洞/中继会失败。
- 别忘了 UDP:`21116`(`PORT_6`)的 UDP 必须放行,否则打洞不可用。

## 备份 / 恢复

要备份的就是 `./data`(密钥 + sqlite 库)。**`id_ed25519` 丢了客户端就要重配 Key**。

```bash
# 备份
docker compose stop
tar czf rustdesk-$(date +%F).tar.gz data/
docker compose start

# 恢复
docker compose down
tar xzf rustdesk-YYYY-MM-DD.tar.gz
docker compose up -d
```

## 升级

```bash
vim .env            # 上调 RUSTDESK_TAG(启动日志会提示 new version,如 1.1.16)
docker compose pull
docker compose up -d
```

数据/密钥在 `./data`,升级不受影响。跨大版本升级前先看上游 release 说明。

## 注意事项

- 官方镜像是 `FROM scratch` 的静态二进制,容器内**没有 shell/wget**,无法写容器内
  `healthcheck`,故本应用两个服务都未配置(被迫的例外)。存活状态看 `docker compose ps`
  和日志。
- `data/` 与 `.env` 已被 git 忽略,不要提交(尤其 `id_ed25519` 私钥)。
