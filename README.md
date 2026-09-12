# 基于 Orbit / Flux 的 3X-UI 多路径分流部署

本项目将 [3X-UI](https://github.com/MHSanaei/3x-ui)（Xray-core 管理面板）的官方面板打包运行在 Flux 的部署平台 [Orbit](https://orbit.runonflux.com) 的容器中。

Flux/Orbit 默认每个容器仅向外暴露**一个带有 TLS 加密的公网端口**（即 "App Port"）。但由于管理面板、Xray 代理节点（Inbound）以及面板的订阅服务器（Subscription）都需要能从外部访问，因此本项目在它们前面运行了一个用 Go 语言编写的轻量级反向代理，在同一个公网端口上根据 **HTTP 请求路径（Path）进行路由分流**（Multiplexing）。

## 工作原理

1. 首次运行时，`main.go` 会直接下载预先编译好的 3X-UI 官方二进制包（`x-ui-linux-amd64.tar.gz`），而不是从源码编译 3X-UI（因为该环境无法编译 Vue 前端，从源码构建需要在 `go build` 之前执行 `npm run build`）。
2. 在**仅限内部访问**的端口上启动真实的 `x-ui` 二进制程序（通过环境变量 `XUI_PORT` 指定）—— 该端口无法从互联网直接访问。
3. 在**公网端口**上启动自带的轻量级 HTTP 服务器。该服务器会检查传入请求的路径（Path），并将请求反向代理转发至对应的内部服务：

   | 请求路径前缀 (Path) | 转发目标（内部端口） | 用途 |
   |---|---|---|
   | `/xvpnws/` | Xray 的 VLESS+WebSocket 入站节点 | 代理流量 |
   | `/sub/` | 面板的订阅服务器 | 订阅链接 |
   | *（其他任意路径）* | 3X-UI 管理面板 | Web 管理后台 |

由于 Go 语言的 `net/http/httputil.ReverseProxy` 原生支持 `Upgrade: websocket` 报头（在完成 101 握手后会自动劫持连接并进行双向字节传输），因此这种方式能够无缝支持基于 WebSocket 的 Xray 传输协议，无需额外的处理代码。

## Orbit/Flux 端前置要求

- **App Port** 的值必须与 `main.go` 中的 `publicPort` 完全一致（当前设置：`2053`）。
- 容器必须具备对 `github.com` 的出站访问权限，以便在首次运行时下载 3X-UI 的 Release 发布包。

## 3X-UI 面板内前置要求

对于你创建的每个节点（Inbound）：
- **Port（端口）** 字段必须与 `main.go` 中定义的内部端口完全一致。
- **Path（路径）**（位于 Stream/传输设置下）必须包含相同的前缀（首尾均带斜杠 `/`）。
- 节点的安全传输（Security）必须设置为 `none`，因为 TLS 证书已由 Flux 的边缘节点处理，流量到达该容器时已经是解密后的状态。

对于订阅服务：
- 将 **设置 → 订阅设置 → 反向代理 URI** 改为 `https://<你的域名>.app.runonflux.io/sub/`，这样面板生成的订阅链接就会指向公网路径，而不是内部端口。

## 添加新路径

如果你想在同一个公网端口上暴露另一个 Xray 节点：

1. 选择一个未被占用的内部端口和一个唯一的路径前缀。
2. 将这两者添加到 `main.go` 中的 `routes` 切片（slice）中。
3. 提交（Commit）并推送（Push）代码，然后在 Orbit 中点击 **Pull & Build** 按钮（注意不要只点 Redeploy，因为 Go 二进制文件需要重新编译）。
4. 在面板中创建一个新节点，端口和路径需与上面设置的完全一致。

## 局限性

- 只有基于 HTTP 的传输协议（如 WebSocket，理论上也包括 XHTTP）才能通过这种方式进行多路复用，因为分流是基于 HTTP 请求路径（Path）判断的。
- 原始 TCP 和 REALITY 节点**不兼容**此架构的公网主端口。因为 TLS 在简单的 HTTP 请求到达此代理之前就已经被 Flux 的边缘节点解密了 —— 而 REALITY 需要客户端原始且未修改的 TLS ClientHello，原始 TCP 则根本没有用于路由的 HTTP 路径。如果你部署时额外开放了第二个原始端口（可在 FluxCloud 的 **Specifications** 标签页下的 `Container Ports` 查看），可以将原始 TCP 或 REALITY 节点直接绑定到该端口，从而完全绕过此代理。

## 安全提示

这是一个用于自托管的个人部署项目。请妥善保管面板的登录凭据、客户端 UUID 以及订阅链接，切勿外泄。
