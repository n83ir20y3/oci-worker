# OCI Worker 配置成功备忘录

本文档用于记录 `oci-worker` 在 `serv00 + Cloudflare + GitHub Actions` 环境下最终跑通时的关键配置，便于下次复用。

## 一、目标结构

建议拆分为两个公网域名：

- 管理后台域名  
  例如：`https://serv00-oci.555606.xyz`

- Worker API 域名  
  例如：`https://serv00-worker-api.555606.xyz`

管理后台域名可以继续受 Cloudflare Access 保护。  
Worker API 域名不能启用 Cloudflare Access 登录保护，否则 GitHub Actions 无法访问。

## 二、主项目服务端成功配置

主项目服务器配置示例：

```yaml
server:
  host: "0.0.0.0"
  port: 10068
  worker_port: 15020
  worker_api_url: "https://serv00-worker-api.555606.xyz"
  mode: "release"
  tls_enabled: false
  worker_token: "你的 worker token"
```

关键点：

- 管理后台主服务监听 `10068`
- 独立 worker API 监听 `15020`
- `worker_api_url` 填公网 worker API 域名
- 当前程序在 `15020` 上跑的是 `HTTP`，不是 `HTTPS`

## 三、serv00 成功配置

在 serv00 面板创建 worker API 站点时：

- `Website type`: `Proxy`
- `Proxy target`: `localhost`
- `Proxy port`: `15020`
- `Use HTTPS`: 不勾选

成功后的实际代理地址应当类似：

```txt
http://127.0.0.1:15020
```

不要配置成：

```txt
https://127.0.0.1:15020
```

这是本次排障里最关键的错误点之一。

## 四、Cloudflare 成功配置

Worker API 域名的 Cloudflare 配置要点：

- DNS 记录开启代理（橙云）
- 不启用 Cloudflare Access
- `SSL/TLS` 模式使用 `Full`

不要使用：

- `Full (strict)`

原因：

- serv00 源站返回的证书未必是该自定义域名的专属证书
- `Full (strict)` 会校验证书域名，容易触发 `526`
- `Full` 可以让 Cloudflare 与源站继续使用 HTTPS，同时不强制校验证书域名匹配

## 五、oci-worker 仓库成功配置

Repository secrets 需要至少包含：

- `OCI_SERVER_URL`
- `OCI_WORKER_TOKEN`
- `GH_PAT`

示例：

```txt
OCI_SERVER_URL=https://serv00-worker-api.555606.xyz
OCI_WORKER_TOKEN=与主项目 server.worker_token 完全一致
GH_PAT=可读取 suwei8/OCI-Manager-Go 私有 release 的 GitHub PAT
```

说明：

- `OCI_SERVER_URL` 必须是 worker API 的公网基础域名
- 不能填写 `https://localhost:15020`
- 不能填写管理后台域名
- `GH_PAT` 的作用是下载主项目私有仓库中的 worker 二进制，不是触发 worker workflow

## 六、验证方法

### 1. 服务端本机验证

在主项目服务器上验证：

```bash
curl http://127.0.0.1:15020/health
```

预期：

```json
{"status":"ok"}
```

### 2. 公网域名验证

验证健康检查：

```bash
curl -I https://serv00-worker-api.555606.xyz/health
```

预期：

- 返回 `200`

验证 worker API：

```bash
curl -i https://serv00-worker-api.555606.xyz/api/v1/worker/task
```

预期：

- 不带 token 时返回 `401`
- 这说明公网 worker API 已经可达

### 3. GitHub Actions 触发验证

从主项目后台启动任务后：

- 不应再出现 GitHub Workflow `404 Not Found`
- `oci-worker` Actions 应成功启动
- worker 能访问 `OCI_SERVER_URL/health`
- worker 能携带 `OCI_WORKER_TOKEN` 请求 `/api/v1/worker/task`

## 七、常见错误与对应原因

### 1. `GitHub API returned 404 Not Found`

常见原因：

- `oci-worker` 仓库工作流未被 GitHub 正常识别
- workflow 文件未生效或未注册

本次处理方式：

- 确保工作流文件存在于默认分支
- 使用 `actions/checkout@v4`

### 2. `302` 跳转到 Cloudflare Access 登录页

原因：

- `OCI_SERVER_URL` 指向了受 Cloudflare Access 保护的域名

处理：

- Worker API 域名不要启用 Cloudflare Access

### 3. `526 Invalid SSL certificate`

原因：

- serv00 代理目标写成了 `https://127.0.0.1:15020`
- 或 Cloudflare 使用了 `Full (strict)`

处理：

- serv00 回源改成 `http://127.0.0.1:15020`
- Cloudflare SSL 模式改成 `Full`

### 4. `401 Invalid worker token`

原因：

- `OCI_WORKER_TOKEN` 与主项目配置中的 `server.worker_token` 不一致

处理：

- 保证两边完全一致

## 八、最终推荐结论

下次配置时直接按下面执行：

1. 主项目启用 `worker_port: 15020`
2. serv00 新建 `Proxy` 站点到 `localhost:15020`
3. 不勾选 `Use HTTPS`
4. Cloudflare 为 worker API 子域名开启代理
5. Worker API 子域名不要启用 Access
6. Cloudflare `SSL/TLS` 使用 `Full`
7. `OCI_SERVER_URL` 填 worker API 公网域名
8. `OCI_WORKER_TOKEN` 与主项目配置保持一致
9. `GH_PAT` 使用可读取主项目私有仓库 release 的 GitHub 账号 token
