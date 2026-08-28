# 009-caddy

[Caddy](https://caddyserver.com/) 边缘反向代理 + **自动 HTTPS**。它就是仓库其它应用 readme 里
反复提到的「前面挂的那层反代 + TLS」:统一对公网收 80/443,按域名转发到各应用只绑在
`127.0.0.1:<PORT_x>` 的端口上,证书自动签发、自动续期。

> **端口约定的合理例外**:Caddy 必须对公网监听 80/443,所以不走 `${PORT_x}`、不绑回环。
> 本应用的端口段 20090-20099 对它保留未用。

## 为什么用 host 网络

`network_mode: host` 让 Caddy 直接共享宿主的 `127.0.0.1`。于是它能反代**所有**遵循本仓库约定
(只绑 `127.0.0.1:${PORT_x}`)的应用,**不用改动那些应用、也不用建共享 docker 网络**。
新增一个反代就是往 `Caddyfile` 里加几行,后端写 `127.0.0.1:<目标应用的 PORT_x>`。

## 部署

```bash
# 1. 生成 .env(端口段仅登记,Caddy 实际用 80/443)
../bin/ports.sh --write .

# 2. 填邮箱
vim .env            # 改 ACME_EMAIL 为你的真实邮箱

# 3. 起(先放行防火墙 80/443,并把域名 A/AAAA 解析到本机)
docker compose up -d
docker compose logs -f
```

前置条件(自动签证书需要):
- 域名的 DNS 解析到本机公网 IP;
- 公网能访问本机 **80 和 443**(80 用于 ACME HTTP 校验和跳转,443 提供服务)。

## 加一个反代(日常就干这个)

编辑 `./Caddyfile`,复制一段改域名和端口:

```caddyfile
vault.example.com {
    import backend 20010      # 001-vaultwarden 的 PORT_0
}

memos.example.com {
    import backend 20070      # 007-memos 的 PORT_0
}
```

`import backend <PORT>` 是本模板内置的片段,等价于 `reverse_proxy 127.0.0.1:<PORT>`。
需要自定义(大文件、websocket、加 header)就直接写 `reverse_proxy`。

改完**热加载,不断连**:

```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

（`Caddyfile` 是只读挂载,改宿主文件后 reload 即可;无需重启容器。）

## 端口对照(填后端时查)

| 应用 | 目录 | 后端端口(PORT_0) |
| ---- | ---- | ----------------- |
| Vaultwarden | 001 | 20010 |
| OpenList | 003 | 20030 |
| FreshRSS | 004 | 20040 |
| FileCodeBox | 006 | 20060 |
| Memos | 007 | 20070 |

> 具体以各应用 `.env` 里的 `PORT_0` 为准。

## 证书 / 数据

- 证书和 ACME 账户存在 `./data/caddy_data`,**务必持久化**:删了会重新签发,可能触发
  Let's Encrypt 速率限制。
- 调试期想避免速率限制,可在 `Caddyfile` 全局块启用 `acme_ca` 的 staging 地址(见文件注释),
  签出的证书不被浏览器信任,验证通了再切回正式。

## 备份 / 恢复

```bash
# 备份(证书 + 配置)
docker compose stop
tar czf caddy-$(date +%F).tar.gz Caddyfile data/
docker compose start

# 恢复
docker compose down
tar xzf caddy-YYYY-MM-DD.tar.gz
docker compose up -d
```

## 升级

```bash
vim .env                       # 改 CADDY_TAG
docker compose pull && docker compose up -d
```

## 注意事项

- host 网络下 `ports:` 会被忽略,Caddy 直接绑宿主 80/443/2019;确保这些端口本机没被别的
  进程占用。
- admin API 在 `127.0.0.1:2019`(healthcheck 用它探活),仅本机可达,勿对外暴露。
- `data/` 与 `.env` 已被 git 忽略,不要提交。
