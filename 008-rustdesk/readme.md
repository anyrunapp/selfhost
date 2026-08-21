# 008-rustdesk

[RustDesk](https://github.com/rustdesk/rustdesk-server) 自建远程桌面服务端(开源 OSS 版)。
自己掌控 ID/中继服务器,不再依赖官方公共服务器。端口段 **20080-20089**(由目录名 `008` 算出)。

服务端由两个进程组成,这里拆成两个容器共享同一份数据/密钥:

- **hbbs**(ID/Rendezvous):设备注册、心跳、打洞、NAT 探测、web 客户端。
- **hbbr**(Relay 中继):打洞失败时转发音视频流量。

## 端口

RustDesk 客户端按**固定偏移**推算端口(NAT 测试口 = ID 口 − 1),所以宿主端口必须与容器端口
**连续同序**。`PORT_0..4` 恰好连续,映射关系是固定的:

| 变量     | 端口  | 容器端口     | 服务 | 用途                         |
| -------- | ----- | ------------ | ---- | ---------------------------- |
| `PORT_0` | 20080 | 21115/tcp    | hbbs | NAT 类型探测                 |
| `PORT_1` | 20081 | 21116/tcp+udp| hbbs | ID 注册/心跳/打洞(**主端口**) |
| `PORT_2` | 20082 | 21117/tcp    | hbbr | 中继                         |
| `PORT_3` | 20083 | 21118/tcp    | hbbs | web 客户端 websocket         |
| `PORT_4` | 20084 | 21119/tcp    | hbbr | web 客户端 websocket         |

客户端里 **ID Server** 填 `你的地址:20081`(即 `PORT_1`)。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 按需修改 .env
vim .env
#    - 对公网提供服务:BIND_ADDR=0.0.0.0,并在防火墙放行 20080-20084(TCP)+ 20081(UDP)
#    - 宿主端口若与容器端口不一致(经四层代理改了口),用 -r 告知客户端真实中继地址:
#      RUSTDESK_HBBS_OPTS=-r 你的公网IP:21117
chmod 600 .env

# 3. 起
docker compose up -d
docker compose logs -f
```

首次启动会在 `./data/` 生成一对密钥 `id_ed25519` / `id_ed25519.pub` 和 sqlite 库
`db_v2.sqlite3`。两个容器共用 `./data`,务必让它们拿到同一份密钥。

## 客户端配置

在 RustDesk 客户端「设置 → 网络 / ID/中继服务器」填写:

- **ID Server**:`你的地址:20081`
- **Relay Server**:`你的地址:20082`(可留空,由 hbbs 下发)
- **Key**:`./data/id_ed25519.pub` 的内容(公钥),用于校验服务器身份。

拿公钥:

```bash
cat data/id_ed25519.pub
```

强制客户端校验公钥可加 `RUSTDESK_HBBS_OPTS=-k _`(推荐,防中间人)。

## 网络 / 暴露

- 默认所有端口只绑 `127.0.0.1`(遵循仓库“仅本机 + 前置代理”约定),此时只能本机自测。
- RustDesk 走裸 TCP/UDP,**不能**用普通 HTTP 反代(Caddy/Nginx 的 http 块)转发。
  对公网提供服务请二选一:
  1. `BIND_ADDR=0.0.0.0` 直接对外(配合防火墙),简单直接;
  2. 用四层(L4 TCP/UDP)转发,且**保持端口连续同序**,否则打洞/中继会失败。
- 别忘了 UDP:`21116`(`PORT_1`)的 UDP 必须放行,否则打洞不可用。

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
