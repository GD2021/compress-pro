# Compress Pro 部署与资产运维文档

- **项目名称**：compress-pro（浏览器端隐私优先多媒体/文档/字体压缩与格式转换工具箱）
- **官方仓库**：https://github.com/Scorpio3310/compress-pro
- **GitHub 维护仓库**：https://github.com/GD2021/compress-pro
- **自定义主域名**：https://compress.lucky.cloudns.biz
- **部署架构**：采用 Cloudflare Workers + Static Assets 混合边缘托管架构（SvelteKit `@sveltejs/adapter-cloudflare` + `wrangler [assets]` + `worker/index.js` Accept 头内容协商包装器）。
- **特点**：全客户端浏览器 WebAssembly + WebCodecs 执行，Zero Backend，文件永不上传云端；边缘端仅负责静态资产托管与 Content Negotiation。

---

## 规范命名资产清单

| 资源类型        | Cloudflare 资源名称 / 域名   | 归属 Zone / 绑定                                               | 说明                                                                                |
| --------------- | ---------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Worker 服务** | `compress-pro`               | Account: `Waka` (`fc6f79e77dca61be461e2dedb82244f5`)           | Cloudflare Workers 服务，包含 Static Assets 绑定 (`env.ASSETS`) 与 Markdown 协商层  |
| **自定义域名**  | `compress.lucky.cloudns.biz` | Zone: `lucky.cloudns.biz` (`4fb4ad18a6c1f968afac4c6ed816d010`) | 自动配置 Cloudflare AAAA (`100::`) 代理记录与边缘 TLS 证书                          |
| **安全响应头**  | `_headers` 与 Worker Headers | `ASSETS`                                                       | 配置 COOP (`same-origin`), COEP (`require-corp`), X-Frame-Options, CSP, Link 等策略 |

---

## 自动化同步与持续部署

- **工作流文件**：`.github/workflows/sync-and-deploy.yml`
- **定时调度**：每天北京时间凌晨 **04:07**（UTC 20:07）自动执行，避开整点调度高峰
- **手动调度**：支持在 GitHub Actions 页面通过 `workflow_dispatch` 手动触发（支持 `force_deploy` 参数）
- **自动化流程**：
  1. 检查上游官方仓库 `Scorpio3310/compress-pro:main` 的最新提交；
  2. 若无上游更新且未指定强制部署，安全退出，不触发冗余构建；
  3. 若检测到上游更新，记录更新日志，备份自定义部署配置，执行自动合并；
  4. 如遇合并冲突，立即中止并安全回退，保护当前生产版本；
  5. 安装依赖并执行全套质量门禁（`pnpm lint`, `pnpm check`, `pnpm test`, `pnpm build`, `pnpm check:bundle`）；
  6. 门禁验证通过后更新仓库代码，并通过 Wrangler 部署至 Cloudflare Workers；
  7. 部署后执行自动化 HTTP 200 及 Markdown 协商健康检查。
- **GitHub Secrets 配置**：
  - `CLOUDFLARE_ACCOUNT_ID`：`fc6f79e77dca61be461e2dedb82244f5`（已配置）
  - `CLOUDFLARE_API_TOKEN`：需要包含 Workers (Edit) 权限的 API Token（本地目前已通过已授权的 Wrangler OAuth 凭证完成首次生产部署并全量验证通过；如需 GitHub Actions 每天在云端全自动触发无人工干预部署，需在 GitHub 仓库添加该 Token）。

---

## 本地日常维护与验证

进入本地项目目录：

```bash
cd /Users/jack/Desktop/运维/Cloudflare_项目/compress-pro
```

- **本地即时部署到生产环境**：
  ```bash
  pnpm build && pnpm exec wrangler deploy
  ```
- **本地启动调试服务**：
  ```bash
  pnpm dev
  ```
- **本地运行测试套件**：
  ```bash
  pnpm test && pnpm check && pnpm check:bundle
  ```
