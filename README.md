# Cloudflare Workers VLESS TCP v3

轻量级 VLESS TCP 转发 Worker，使用 Cloudflare 边缘提供的 HTTPS/WebSocket 和 `cloudflare:sockets`。节点为标准 VLESS + WebSocket + TLS，可导入使用 Xray Core 的 v2rayN。

## 通过 Workers Builds 部署

1. 将本项目中的文件直接放在 GitHub 仓库根目录。根目录应能看到 `package.json`、`wrangler.jsonc` 和 `src/`，不要多套一层项目目录。
2. 在 Cloudflare Dashboard 的 **Workers & Pages** 中创建 Worker 并连接 Git 仓库。选择正确的 GitHub 账号和仓库，生产分支使用 `main`。
3. 项目根目录保持默认值（仓库根目录）。Worker 名称必须是 `cloudflare-vless-worker`，与 `wrangler.jsonc` 中的 `name` 一致。
4. Workers Builds 设置：构建命令为 `npm ci && npm run check`，部署命令为 `npm run deploy`。
5. 创建并部署 Worker 后，在 **Settings → Variables and Secrets** 添加 Secret：名称 `UUID`，值为自行生成的 UUID。不要把 UUID 写入仓库或公开截图。也可在已登录 Cloudflare 的终端运行 `npx wrangler secret put UUID`。
6. 可选：添加普通变量 `WS_PATH` 设置 WebSocket 路径，默认 `/ws`。自定义值只能包含字母、数字、`_`、`-` 和 `/`，不能包含查询字符串。
7. 可选：在 **Settings → Domains & Routes → Add → Custom Domain** 绑定自己的域名。域名需要在此 Cloudflare 账号中启用 DNS，Cloudflare 会为该域名配置 TLS；也可以使用 `workers.dev` 域名。

Workers Builds 会管理构建部署凭据，不需要为本部署方式添加 GitHub Actions 用的 Cloudflare API Token 或 Account ID Secrets。

UUID 可在 macOS/Linux 使用 `uuidgen` 生成，也可在 Windows PowerShell 使用 `[guid]::NewGuid().ToString()` 生成。

## GitHub 授权后仓库没有出现在列表

公开仓库也需要 Cloudflare 的 GitHub App 获得仓库访问权限。检查以下项目：

1. 确认当前登录 Cloudflare 的 GitHub 账号就是拥有该仓库的账号，并且 Cloudflare Dashboard 处于预期的 Cloudflare 账号。
2. 在 GitHub 打开 **Settings → Applications → Installed GitHub Apps → Cloudflare Workers and Pages → Configure**。
3. 查看 **Repository access**。如果选择了 **Only select repositories**，将 `cloudflare-vless-worker` 加入允许列表；也可以按需选择 **All repositories**。保存后回到 Cloudflare 刷新仓库列表。
4. 如果仓库属于组织，确认 Cloudflare Workers and Pages App 已安装到该组织。安装通常需要组织 Owner 或拥有 GitHub Apps 管理权限的成员；组织的第三方 App 审批策略也可能需要管理员批准。
5. 确认该 GitHub 账号没有把此集成授权给另一个 Cloudflare 账号。Cloudflare 建议一个 GitHub 账号只关联一个 Cloudflare 账号。
6. 如果权限已正确但仓库仍未出现，可在 GitHub App 安装设置中移除并重新安装 Cloudflare Workers and Pages，再回到 Cloudflare 重新连接。

这些仓库列表权限由 GitHub App 安装设置控制，Worker 源码或 Wrangler 构建命令不能替代该授权。

## 获取 v2rayN 节点

部署并设置 `UUID` 后，打开 `https://你的域名/你的UUID`。页面会返回一条 `vless://` 分享链接，复制后在 v2rayN 使用“从剪贴板导入”。如果绑定了自定义域名，请通过自定义域名访问配置地址；链接会自动使用当前访问域名作为服务器地址、TLS SNI 和 WebSocket Host。

部署配置生成 `security=tls`、`type=ws`、`encryption=none` 的标准 VLESS 链接。WebSocket 路径使用 `WS_PATH` 的实际值。

## 本地开发

```sh
npm ci
```

在本地创建未跟踪的 `.dev.vars` 文件：

```dotenv
UUID="你自己的UUID"
WS_PATH="/ws"
```

`.dev.vars` 已加入 `.gitignore`。运行检查和本地 Worker：

```sh
npm run check
npm run dev
```

## 行为与限制

- 仅支持 VLESS TCP 命令。UDP 和 MUX 请求会关闭连接。
- 允许公网 TCP 目标；拒绝私网/保留 IP、localhost、本地域名、Worker 自身域名和 TCP 25 端口。Cloudflare Runtime 还会拒绝其禁止的网络目标。部分端口可能受 Cloudflare 平台策略限制。
- 普通 HTTP 路径返回静态状态页；只有配置的 WebSocket 路径接受隧道连接。VLESS UUID 才是认证凭据，WebSocket 路径不替代 UUID 鉴权。
- TLS 在 Cloudflare 边缘终止。Worker 无法提供 XTLS 或 Reality 握手；实际可达性取决于域名和访问者到 Cloudflare 的网络连通情况。
- 配置地址和节点分享链接都包含 UUID，按密码保护，不要公开分享。
