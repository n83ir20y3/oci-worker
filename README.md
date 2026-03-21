# OCI Worker（公开仓库）

这个公开仓库用于承载 OCI Manager Worker 的 GitHub Actions 工作流。
之所以单独放在公开仓库中，是为了使用公开仓库的 GitHub Actions 配额来运行 worker。

## 基础说明

工作流文件位于 `.github/workflows/oci-worker.yml`，支持两种触发方式：

- 手动触发 `workflow_dispatch`
- 定时触发 `schedule`（当前为每 5 分钟一次）

工作流执行逻辑如下：

1. 检出当前仓库
2. 使用 `GH_PAT` 从私有仓库 `suwei8/OCI-Manager-Go` 下载 worker 二进制
3. 检查 `OCI_SERVER_URL/health`
4. 运行 `oci-manager worker`

## 必需的 Repository Secrets

进入 `Settings -> Secrets and variables -> Actions`，添加以下 secrets：

- `OCI_SERVER_URL`
  worker API 的公网基础地址，例如 `https://worker-api.example.com`

- `OCI_WORKER_TOKEN`
  worker 回连主项目时使用的认证 token，必须与主项目服务端配置中的 `server.worker_token` 完全一致

- `GH_PAT`
  GitHub Personal Access Token（Classic），用于从私有仓库 `suwei8/OCI-Manager-Go` 下载 release 资产  
  此 token 对应的 GitHub 账号必须拥有该私有仓库的读取权限  
  私有仓库场景下通常需要 `repo` 权限

## 重要注意事项

- `OCI_SERVER_URL` 必须填写 worker API 的公网域名，不能填写 `localhost`
- `OCI_SERVER_URL` 不能填写管理后台域名，应该填写专门给 worker 使用的 API 域名
- 如果 worker API 放在 Cloudflare 后面，不要给该子域名启用 Cloudflare Access 登录保护，否则 GitHub Actions 无法访问
- 如果使用反向代理转发到本机端口，需确认源站协议与后端真实监听协议一致

## 使用方式

- 定时运行：GitHub Actions 会按 cron 自动执行
- 手动运行：在仓库 `Actions` 页面中选择工作流并点击运行
- 指定任务：手动触发时可传入 `task_id`

## 配置备忘

完整的成功配置与排障要点见 [SETUP_MEMO.zh-CN.md](/home/sw/dev_root/oci-worker/SETUP_MEMO.zh-CN.md)。
