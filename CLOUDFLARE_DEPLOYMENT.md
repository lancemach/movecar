# Cloudflare Workers 自动部署记录

## 当前部署方式

本项目是 Cloudflare Worker，不是 Vite 静态站点。Worker 入口为 `movecar.js`，配置文件为 `wrangler.jsonc`，运行时使用 KV Namespace `MOVE_CAR_STATUS`。

Cloudflare Workers Builds 的生产分支配置如下：

| 配置项 | 值 |
| --- | --- |
| 生产分支 | `main` |
| 根目录 | `.`（仓库根目录） |
| 构建命令 | 留空 |
| 部署命令 | `npx wrangler deploy` |
| 非生产分支部署命令 | `npx wrangler versions upload` |

## 遇到的问题

首次自动部署失败，日志为：

```text
Missing entry-point to Worker script or to assets directory
```

当时 Cloudflare Workers Builds 的根目录填写为 `/`。部署命令虽然执行了 `npx wrangler deploy`，但 Wrangler 没有在仓库根目录读取到 `wrangler.jsonc`，因此无法发现 `movecar.js` 入口。

## 修复方式

将 Cloudflare Dashboard → Worker → Settings → Builds 中的根目录从：

```text
/
```

修改为：

```text
.
```

或者留空，使其使用连接仓库的根目录。

仓库中的 `wrangler.jsonc` 必须包含 Worker 入口和 KV 绑定：

```jsonc
{
  "name": "movecar",
  "main": "movecar.js",
  "compatibility_date": "2026-08-19",
  "kv_namespaces": [
    {
      "binding": "MOVE_CAR_STATUS",
      "id": "d4f71878b2a14d228a0ebebc71a57be2"
    }
  ]
}
```

## 成功部署的日志特征

修正根目录后，自动部署成功并出现以下信息：

```text
Total Upload: 38.72 KiB / gzip: 8.61 KiB
env.MOVE_CAR_STATUS ... KV Namespace
Uploaded movecar
Deployed movecar triggers
Success: Deploy command completed
✨ Success! Build completed.
```

成功部署还会输出 Worker 地址和 `Current Version ID`。其中 `MOVE_CAR_STATUS` 绑定显示为预期的 KV Namespace ID，说明 KV 配置已生效。

## 提交代码验证自动部署

先确认当前分支是 `main`，再提交并推送文档：

```powershell
git branch --show-current
git status
git add CLOUDFLARE_DEPLOYMENT.md
git commit -m "docs: record Cloudflare deployment troubleshooting"
git push origin main
```

然后在 Cloudflare Dashboard 中进入：

`Workers & Pages → movecar → Deployments → Build history`

检查本次提交是否出现新的构建记录。生产分支 `main` 应执行：

```text
npx wrangler deploy
```

构建成功时应显示：

```text
Success: Deploy command completed
✨ Success! Build completed.
```

## 注意事项

- `BARK_URL` 不要写入仓库，应在 Worker 的 Variables and Secrets 中作为 Secret 配置。
- `PHONE_NUMBER` 为可选变量，也应在 Cloudflare 的运行时变量中配置。
- `npx wrangler versions upload` 仅用于非生产分支预览，不会把版本直接提升为正式部署。
- `workers_dev` 和 `preview_urls` 的 Wrangler 警告不代表部署失败；如需消除警告，可在 `wrangler.jsonc` 中显式配置。

## GitHub 自动触发排查记录

构建配置正确、Worker 手动部署成功后，推送到 `main` 仍没有立即触发构建。排查发现，GitHub 中 Cloudflare Workers and Pages App 的仓库授权范围选错了：

```text
lancemach/CF-Workers-docker.io
```

而当前项目实际仓库是：

```text
lancemach/movecar
```

因此 Cloudflare 没有获得 `movecar` 仓库的 push 事件权限。修复方式是在 GitHub 的 Cloudflare Workers and Pages App 权限页面中：

1. 保持 `Only select repositories`。
2. 移除 `lancemach/CF-Workers-docker.io`。
3. 添加并选择 `lancemach/movecar`。
4. 点击 `Save`，确认 Cloudflare Worker 的 Git 存储库仍为 `lancemach/movecar`。

修复后使用空提交验证触发器：

```powershell
git commit --allow-empty -m "test: trigger Cloudflare Workers build"
git push origin main
```

Cloudflare 部署历史随后显示：

```text
正在进行  test: trigger Cloudflare Workers build  main
```

这证明 GitHub push 事件已经送达 Cloudflare，自动构建触发器恢复正常。构建完成后还应确认日志包含 `Success: Deploy command completed` 和 `✨ Success! Build completed.`。
