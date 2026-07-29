# 001-vaultwarden

Vaultwarden + PostgreSQL。端口段 **20010-20019**(由目录名 `001` 算出)。

| 变量     | 端口  | 用途                          |
| -------- | ----- | ----------------------------- |
| `PORT_0` | 20010 | Vaultwarden HTTP(反代指这里) |
| `PORT_1` | 20011 | PostgreSQL(仅 127.0.0.1)     |

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .

# 2. 填密码
sed -i "s/CHANGE_ME/$(openssl rand -hex 24)/" .env
vim .env            # DOMAIN、TZ、SMTP

# 3. 生成 ADMIN_TOKEN(argon2 哈希)
cp secrets.env.example secrets.env
docker run --rm -it vaultwarden/server:alpine /vaultwarden hash --preset owasp
vim secrets.env     # 把哈希粘进 ADMIN_TOKEN=
chmod 600 .env secrets.env

# 4. (可选)中文化邮件 + 管理界面,见下方"中文化"章节

# 5. 起
docker compose up -d
docker compose logs -f vaultwarden
curl -I http://127.0.0.1:20010/alive
```

## 反向代理(Caddy 示例)

```
vault.example.com {
    reverse_proxy 127.0.0.1:20010
}
```

Vaultwarden 1.30+ 的 WebSocket 通知走 `/notifications/hub`,和主端口同一个,
不需要再单独开 3012。

## 注意事项

- **`.env` 里的 `$` 会被 compose 插值**,密码里有 `$` 要写成 `$$`。所以 `ADMIN_TOKEN`
  的 argon2 哈希(满是 `$`)放在 `secrets.env`,通过 `env_file:` 加载,不做插值。
- `POSTGRES_PASSWORD` 会拼进 `DATABASE_URL`,**只用字母数字**,省掉 URL 编码的坑。
- 首次注册完记得把 `SIGNUPS_ALLOWED` 保持 `false`,靠 `INVITATIONS_ALLOWED` 邀请人。
- `DOMAIN` 必须和实际访问地址完全一致(含 https),否则 WebAuthn / 邮件链接会出问题。
- **附件配额**:`USER_ATTACHMENT_LIMIT` / `ORG_ATTACHMENT_LIMIT`(KB)限的是每用户 /
  每组织的**累计总量**,不是单个文件。想限**单次上传的文件大小**要在反代做:
  Caddy `request_body { max_size 200MB }`、Nginx `client_max_body_size 200m`。
  全站总量 Vaultwarden 无原生开关,靠 per-user 配额×人数或数据卷容量卡。
- 若已用 admin 面板改过配置(生成了 `data/vaultwarden/config.json`),面板值会**覆盖**
  环境变量,两边别打架,统一在一处管。

## 中文化:邮件 + 管理界面

Vaultwarden 默认邮件模板和 `/admin` 管理界面都是英文,而且**服务端只发一种语言**——
用户的语言偏好纯在客户端,服务端不感知,做不到按用户分语言。要中文就得整体替换模板。

社区已有现成翻译包 [WeiYusc/vaultwarden-lang-zhcn](https://github.com/WeiYusc/vaultwarden-lang-zhcn),
覆盖 `admin/`(管理后台)和 `email/`(邮件)两套 Handlebars 模板。手动拷进数据卷就行,
不需要脚本:

```bash
git clone https://github.com/WeiYusc/vaultwarden-lang-zhcn /tmp/vw-zhcn
# 本项目的 /data 挂在 ./data/vaultwarden,模板放到它下面的 templates/
cp -r /tmp/vw-zhcn/templates ./data/vaultwarden/
docker compose restart vaultwarden
```

Vaultwarden 启动时优先从 `/data/templates` 读同名模板,找不到的自动回退内置英文版,
所以只放 `admin/` 和 `email/` 不会影响 `404.hbs`、`scss/` 等其它页面。
`data/` 已被 git 忽略,这份模板不进仓库,升级/迁移时按上面命令重新拷即可。

**模板和版本强绑定**:翻译包按某个 vaultwarden 版本(如 `1.37.0`)对齐,升级镜像时
先到仓库确认 tag 是否匹配当前 `VAULTWARDEN_TAG`,不匹配可能渲染错乱或漏译。

> 管理后台仍有少量英文(配置项说明、tooltip、部分弹窗)来自程序内置数据而非模板,
> 无法通过 `/data/templates` 覆盖,属正常现象。

### CSS

v1.33.0 起 web-vault 的样式可以覆盖,写在 `./data/vaultwarden/templates/scss/user.vaultwarden.scss.hbs`
(容器内路径 `/data/templates/scss/user.vaultwarden.scss.hbs`),常见用途:页脚提示语、换 Logo、隐藏 2FA 选项。

- **别去建 `vaultwarden.scss.hbs`**,那是内置默认,一旦存在就整个覆盖掉,升级必炸。只碰 `user.` 那个。
- 调试时给 vaultwarden 加环境变量 `RELOAD_TEMPLATES=true`,改完刷新即可,不用重启;调完改回 `false`。
- 验证:`curl -s https://vault.example.com/vw_static/vaultwarden.css | tail`,看自己的规则在不在。
  没生效先确认容器里路径对: `docker compose exec vaultwarden ls /data/templates/scss`。
- 选择器跟着 Bitwarden 前端走,**上游改版会随时失效**,升级后顺手看一眼。

## 备份

```bash
# 数据库
docker compose exec -T db pg_dump -U vaultwarden vaultwarden | gzip > vw-$(date +%F).sql.gz
# attachments / rsa_key 等
tar czf vw-data-$(date +%F).tar.gz data/vaultwarden
```

`data/` 已被 git 忽略,不要提交。

## 升级

```bash
vim .env                       # 改 VAULTWARDEN_TAG
docker compose pull && docker compose up -d
```

跨 PostgreSQL 大版本(如 16 -> 17)不能直接改 `POSTGRES_TAG`,要先 dump 再 restore。
