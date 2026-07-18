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

# 4. (可选)中文邮件模板
./pull-email-templates.sh

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

## 自定义:邮件模板(中文)+ CSS

两者都走同一个 `TEMPLATES_FOLDER`(compose 里已设为 `/data/templates`,由 `./templates` 挂载进去)。
`templates/` 是**配置,要进 git**;只有 `data/` 被忽略。

```
templates/
├── email/                       # 邮件模板(中文翻译)
│   ├── welcome.hbs              # 纯文本版
│   ├── welcome.html.hbs         # HTML 版
│   └── ...
└── scss/
    └── user.vaultwarden.scss.hbs   # 自定义 CSS
```

### 邮件模板

Vaultwarden 只发英文邮件——用户的语言偏好是纯客户端的,服务端不知道,所以做不到按用户分语言。
要中文就得整体替换模板,全站一种语言。

```bash
./pull-email-templates.sh        # 按 .env 的 VAULTWARDEN_TAG 拉对应版本的 63 个模板
# 翻译 templates/email/*.hbs
docker compose restart vaultwarden
```

`.hbs` 格式:**第一行是邮件主题**,`<!---------------->` 是分隔符,下面是正文。
`xxx.hbs` 是纯文本版,`xxx.html.hbs` 是 HTML 版,两个都要改。

```hbs
欢迎使用
<!---------------->
感谢你在 {{url}} 注册账号,现在可以用新账号登录了。

如果这不是你本人操作,忽略本邮件即可。
{{> email/email_footer_text }}
```

`{{变量}}` 和 `{{> email/xxx}}` 一律原样保留,改坏了链接就失效。

不想自己翻,wiki 上有几个现成的社区中文包(`vaultwarden-lang-zhcn` 等),直接拷进
`templates/email/` 就行——但都是社区维护、按需自测。

**模板和版本强绑定**,升级镜像必须同步核对:

```bash
vim .env                                    # 改 VAULTWARDEN_TAG
./pull-email-templates.sh --upstream        # 新版原文拉到 .upstream/(不覆盖翻译)
diff -ru .upstream/email templates/email    # 看上游改了什么,合进翻译
```

### CSS

v1.33.0 起 web-vault 的样式可以覆盖,写在 `templates/scss/user.vaultwarden.scss.hbs`
(仓库里已放了带注释的示例:页脚提示语、换 Logo、隐藏 2FA 选项)。

- **别去建 `vaultwarden.scss.hbs`**,那是内置默认,一旦存在就整个覆盖掉,升级必炸。只碰 `user.` 那个。
- 调试时把 `.env` 的 `RELOAD_TEMPLATES=true`,改完刷新即可,不用重启;调完改回 `false`。
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
