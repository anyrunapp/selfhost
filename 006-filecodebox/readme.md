# 006-filecodebox

[FileCodeBox](https://github.com/vastsa/FileCodeBox) 文件快递柜:匿名用口令分享文件 / 文本,
对方凭取件码提取。端口段 **20060-20069**(由目录名 `006` 算出)。

| 变量     | 端口  | 用途                                |
| -------- | ----- | ----------------------------------- |
| `PORT_0` | 20060 | FileCodeBox HTTP(仅 127.0.0.1 回环) |

## 关于数据库

FileCodeBox(v2.5.4)**内置 SQLite**,连接配置在代码里写死(`core/database.py`),
镜像也没打包 postgres/mysql 驱动,**无法外挂数据库**。所以这里就是单服务 + 一个数据卷,
SQLite 库(`filecodebox.db`)和上传的文件都放在 `./data/filecodebox`。

> 需要多实例/高并发再考虑上游后续版本;当前版本用 SQLite 足够个人和小团队。

## 部署

```bash
# 1. 生成 .env(自动填入端口段)
../bin/ports.sh --write .
vim .env            # 按需改 WORKERS / 反代网段 FORWARDED_ALLOW_IPS
chmod 600 .env

# 2. 起
docker compose up -d
docker compose logs -f filecodebox
```

首启后**在浏览器打开站点完成管理员密码初始化**(本版本不再用默认密码,首次进入引导设置)。
访问:`http://127.0.0.1:20060`,后台入口 `/#/admin`(或 `/admin`)。

## 连接 / 网络

- 端口只绑 `127.0.0.1`,不对公网暴露;对外访问在前面加 Caddy/Nginx 反代 + TLS。
- 走反代时,把 `.env` 的 `FORWARDED_ALLOW_IPS` 设成反代所在网段(如 `172.16.0.0/12`),
  否则应用只信直连 IP,限流 / 日志里的客户端 IP 会是反代地址。

## 注意事项

- 容器以 root 运行,entrypoint 直接用挂载目录,bind mount 无需手动 chown。
- 所有运行期配置(站点名、限流、存储后端 S3/OneDrive/WebDAV 等)在后台页面里改,存进 SQLite。
- `data/` 已被 git 忽略,不要提交。

## 备份 / 恢复

整个数据卷打包即可(含库 + 文件):

```bash
# 备份(建议先停服保证 SQLite 一致性)
docker compose stop filecodebox
tar czf filecodebox-$(date +%F).tar.gz data/filecodebox
docker compose start filecodebox

# 恢复
docker compose down
tar xzf filecodebox-YYYY-MM-DD.tar.gz
docker compose up -d
```

## 升级

```bash
vim .env                       # 改 FILECODEBOX_TAG
docker compose pull && docker compose up -d
```

升级前先做上面的备份;应用会自动执行内置数据库迁移(`apps/base/migrations`)。
