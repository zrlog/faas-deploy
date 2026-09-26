# AWS Lambda + Cloudflare D1 高级部署

ZrLog 的推荐部署路径是 Native/Zip + SQLite 和 Docker Compose + MySQL。本仓库对应高级路径
AWS Lambda + Cloudflare D1。

本仓库提供两段 Cloudflare Worker 代码：

- [`cloudflare/d1-database-worker/index.js`](cloudflare/d1-database-worker/index.js)：把 Cloudflare D1 暴露为 ZrLog 可用的 WebApi 数据库。
- [`cloudflare/pages-worker/index.js`](cloudflare/pages-worker/index.js)：可选，把公开请求转发到 AWS Lambda 后端。

下面是 AWS Lambda + Cloudflare D1 的最短部署流程。Cloudflare 和 AWS 的控制台名称可能调整，本文只列出代码实际需要的绑定和变量。

## 1. 部署 D1 数据库 Worker

1. 在 Cloudflare 创建一个 D1 数据库和一个 Worker。
2. 将 [`d1-database-worker/index.js`](cloudflare/d1-database-worker/index.js) 作为 Worker 代码发布。
3. 给 Worker 配置以下绑定和变量，然后重新部署：

| 类型 | 名称 | 值 |
| --- | --- | --- |
| D1 数据库绑定 | `DB` | 第 1 步创建的 D1 数据库 |
| 普通变量 | `DB_NAME` | 自定义数据库名，例如 `zrlog` |
| Secret | `DB_USER` | 自定义数据库用户名 |
| Secret | `DB_PASSWORD` | 高强度随机密码 |

发布后访问 `https://<D1 Worker 域名>/_region`。能看到 Worker 所在区域，说明路由已经生效；实际 D1 绑定和凭据会在安装页的“测试连接”中一起验证。

安装页需要填写的值如下：

| 安装页字段 | 填写值 |
| --- | --- |
| 数据库类型 | `WebApi` |
| 数据库地址 | D1 Worker 域名，不带 `https://` 和路径 |
| 端口 | `443` |
| 数据库名 | `DB_NAME` 的值 |
| 用户名 | `DB_USER` 的值 |
| 密码 | `DB_PASSWORD` 的值 |

## 2. 部署 ZrLog FaaS 包

1. 从 [ZrLog 下载中心](https://www.zrlog.com/download#advanced-deployment)下载与函数架构一致的 Linux AMD64 或 ARM64 FaaS 包。
2. 在 AWS Lambda 创建使用自定义运行时的函数，架构必须与下载的包一致，然后上传整个 FaaS ZIP。包内的 `bootstrap` 是函数入口，不要只上传 `zrlog` 文件。
3. 为函数创建可访问的 HTTPS 入口，例如 Lambda Function URL 或 API Gateway 路由。安装期间应先限制入口访问，或按下一节启用安装令牌。
4. 可选：发布 [`pages-worker/index.js`](cloudflare/pages-worker/index.js) 作为公开代理，并设置 `BACKEND_SERVER_URL` 为 Lambda 后端地址。该值末尾不要保留 `/`，否则与请求路径拼接后会出现双斜杠。

不使用代理时，直接以 Lambda 的公开地址作为下文的站点地址。

## 3. 首次安装保护

默认不配置 `ZRLOG_INSTALL_TOKEN`。此时直接打开 `https://<站点地址>/install`，页面不会要求令牌，也不需要查看日志、查找文件或复制系统生成的值。

未安装的站点如果直接暴露在公网，可能被他人抢先初始化。公网、共享网络或无人值守部署应在函数平台的环境变量或 Secret 中设置：

```text
ZRLOG_INSTALL_TOKEN=<由部署者保存的高强度随机值>
```

这个值由部署者设置，不是 ZrLog 生成后放在日志里的值。重新发布函数后，安装页会出现“安装口令”输入框，输入部署时设置的同一个值即可。

需要特别保证：

- 安装期间所有函数版本、别名、冷启动和并发实例使用同一个稳定值。
- 不要在每次冷启动时生成新值，不要写入 URL、代码仓库、普通日志或 `/tmp`。
- `/tmp` 只属于单个临时实例，不能用于向安装人分发或持久化令牌。

## 4. 完成安装并持久化配置

1. 打开 `https://<站点地址>/install`；启用了安装令牌时，输入部署时保存的同一个值。
2. 选择 `WebApi`，按第 1 节的表格填写 D1 Worker 信息，测试连接后完成安装。
3. 安装完成页会给出 `DB_PROPERTIES`。把页面显示的完整值原样添加到函数平台的同名环境变量或 Secret 中，并重新发布函数。
4. 确认所有函数版本和别名都使用同一个 `DB_PROPERTIES`，再访问站点验证。缺少该变量时，新冷启动实例会再次进入安装状态。
5. 确认站点在新实例中正常启动后，可以移除 `ZRLOG_INSTALL_TOKEN` 并再次发布；数据库凭据和 `DB_PROPERTIES` 仍需保留。

FaaS 本地文件不会在所有实例之间可靠共享，因此不要依赖本地 `db.properties`、`install.lock` 或 `/tmp` 完成持久化。D1 数据、`DB_PROPERTIES` 和平台 Secret 才是跨冷启动、跨实例的一致配置来源。

## 排错日志

正常安装不需要查看日志，日志也不会提供安装令牌。页面打不开、连接失败或冷启动异常时，再查看对应平台的函数/Worker 日志：

- AWS Lambda：使用平台日志，或在已经配置 AWS CLI 和权限的终端执行：

  ```bash
  aws logs tail "/aws/lambda/<function-name>" --follow
  ```

- Cloudflare Worker：使用平台提供的实时日志或部署日志，分别检查 D1 Worker 和可选的公开代理 Worker。

常见现象：

- D1 Worker 返回 `401`：`DB_USER` 或 `DB_PASSWORD` 与安装页填写值不一致。
- D1 Worker 返回 `404`：请求路径中的数据库名与 `DB_NAME` 不一致。
- 安装口令被拒绝：当前路由指向的函数版本没有读取到 `ZRLOG_INSTALL_TOKEN`，或页面输入值与部署值不同。
- 冷启动后再次出现安装页：`DB_PROPERTIES` 没有配置到当前版本、别名或全部实例。
- 代理访问异常：检查 `BACKEND_SERVER_URL`、公开路由和函数启动错误，不要从日志中查找令牌。

日志和截图中应隐藏数据库密码、`DB_PROPERTIES` 与 `ZRLOG_INSTALL_TOKEN` 的真实值。
