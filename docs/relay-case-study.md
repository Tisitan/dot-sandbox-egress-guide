# 案例审查：Minecraft Java 的 TCP-over-WSS 中继

**审查基线：** `mc-over-wss-relay` 固定快照 `6b98173e835a8f7d23166669878032e49c821b55`，以及主指南 `dot-sandbox-egress-guide` 固定快照 `5e7bc0806271ce0a3ef0c1e8d2f2b84a1a67c583`。本页是文档与代码的静态交叉核对，不是运行测试或安全认证；未修改脚本，也未重新跑作者的实验。

## 作者实验记录与可复现性

案例 README 将 Minecraft Java 视为裸 TCP 客户端：客户端连本机 `127.0.0.1:25565`，`relay.py` 建立到公网入口的 TLS/WebSocket 会话，服务端反代将 WebSocket 转给 `ws-bridge.py`，后者再连接目标 TCP 服务。这个架构在协议上把 TCP 字节流封装到 WebSocket 二进制消息中。

作者报告三项数值：建链约 **470 ms**、稳态单帧往返约 **95 ms**、一次会话传输约 **3.1 MB** 区块数据。案例 README 未标注测试日期、Minecraft/Java/Python/反代版本、地区与链路、样本数量或测量方法。本文保留数字作为作者当时的报告，不把它们写成普遍性能指标；补充这些上下文并附原始输出后，读者才更容易复现。

主指南对入口方案的状态是：A（隧道商 `auto_https`、自有证书、DNS-01）标为作者端到端实测；B（Cloudflare Tunnel）与 C（境外 VPS 自建）标为原理推断、尚未实测。改稿保留这一差别。仓库 README 没有附足以独立复核 A 的命令记录、抓包或端到端日志，因此“作者称已测”不能提升为本次审查者的实测结论。

## 代码中可直接核对的行为

### TLS、配置和认证边界

- `relay.py` 在 WSS 路径调用 `ssl.create_default_context()`，并以目标主机名包装 TLS socket；静态代码中没有关闭证书或主机名校验的开关。[relay.py，TLS 与握手实现](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L254-L338)
- `relay.py` 将配置文件装入 `_config` 一次，发生在 `main()` 启动阶段；没有文件变更监视或重新读取逻辑。编辑 `relay.conf` 后需要重启进程，新连接本身不会使已加载的配置文件自动刷新。[relay.py，配置加载与启动](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L117-L172) [relay.py，main](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L681-L759)
- `ws-bridge.py` 监听普通 TCP socket 并处理 HTTP/WebSocket Upgrade；TLS 终止由前置反向代理承担。桥本身没有 HTTP Basic 校验或 TLS 客户端证书验证。README 提到 Basic、mTLS、IP 白名单等，是反代层可选配置，不是代码已经实现的控制。[ws-bridge.py，握手和监听](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/ws-bridge.py#L114-L180) [案例 README，边界声明](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/README.md#L366-L381)
- 客户端中继的 HTTP 代理解析不接受 `user:pass@` 凭据；这与“反代层可另行配置 Basic”是不同认证位置。部署者要确认实际客户端、代理和 TLS 终止点分别能否实施所需认证。

### 并发与数据泵

- 客户端 `relay.py` 的 accept 循环为每条入站连接创建一条线程；每条连接里的 `pump()` 用一个 `select` 循环双向读写两个端点。此处同一连接的 TLS/WebSocket socket 由该泵操作。
- 服务端 `ws-bridge.py` 每条连接运行 handler 线程，并另外启动一条 `ws_to_tcp` 线程；handler 主线程负责 TCP 到 WebSocket 的方向。发送 WebSocket 帧时使用模块级 `_lock_out` 包围 `conn.sendall(...)`。这把锁会跨连接共享、序列化被锁住的帧发送；它不是日志输出锁，也不能据此说所有桥接逻辑都是单线程 `select`。[relay.py，pump 与连接线程](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L561-L679) [ws-bridge.py，帧发送与双向线程](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/ws-bridge.py#L233-L359)

### 升级握手后的残余字节

这是条件性边界，取决于 TCP `recv()` 如何合并到达的数据：

- 客户端 `relay.py` 在读取 HTTP 101 响应头时也可能读入后续字节，并把 `leftover` 传给 `WsEndpoint`。`WsEndpoint` 将这些字节放进原始数据缓冲区，在调用 WebSocket 帧解析器前先直接返回。因此，若 101 响应和服务端首个 WebSocket 帧恰好在同一次 `recv()` 中到达，该帧的原始编码会被当作 Minecraft 字节交给本地 TCP 客户端。[relay.py，握手 leftover](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L284-L338) [relay.py，WsEndpoint 缓冲与读取](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L429-L498)
- 服务端 `ws-bridge.py` 的 `do_handshake()` 会丢弃 Upgrade 请求头结束后的 `_rest`。若请求和其后续 WebSocket 帧在同一次读取中出现，残余帧字节会被丢掉；常规客户端应等服务器响应 101 后才发首帧，所以这是一个边界条件，不是每次连接都会触发。[ws-bridge.py，Upgrade 读取](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/ws-bridge.py#L114-L180)

### Ping、帧格式和关闭语义

- 客户端 `WsEndpoint.read_chunk()` 遇到 ping 会回 pong，然后继续在同一个同步调用中等待下一帧；该调用发生于单线程 `pump()`。如果此时没有后续 WebSocket 数据，泵可能停在读取上，暂时不处理另一个方向已经就绪的 TCP 数据。[relay.py，控制帧与 pump](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L445-L594)
- 代码实现了常用长度编码、客户端发送掩码、基本 binary/text 数据、continuation、ping/pong 和 close 处理；但当前解析路径没有完整验证 RFC 6455 的 RSV 位、保留 opcode、控制帧 FIN 与 125 字节上限、所有分片顺序等规则。服务端读取路径也没有拒绝未掩码的客户端帧。因而 README 应描述为“覆盖本案例所需的帧子集”，不要称其为完整 RFC 6455 实现。规范要求客户端帧掩码、服务端帧不掩码；控制帧不可分片且 payload 不超过 125 字节。[RFC 6455 §§5.2–5.5](https://www.rfc-editor.org/rfc/rfc6455#section-5.2) [relay.py 帧解析](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L340-L386) [ws-bridge.py 帧解析](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/ws-bridge.py#L188-L228)
- `RawEndpoint.half_close()` 对 TCP socket 调用 `shutdown(SHUT_WR)`；`WsEndpoint.half_close()` 则发送 WebSocket Close 帧。WebSocket Close 是控制帧和关闭握手的一部分，TCP FIN/half-close 是一个传输方向关闭，语义不同。不能把两者说成等价的半关闭。[relay.py，endpoint shutdown](https://github.com/Tisitan/mc-over-wss-relay/blob/6b98173e835a8f7d23166669878032e49c821b55/relay.py#L394-L519) [RFC 6455 §5.5.1](https://www.rfc-editor.org/rfc/rfc6455#section-5.5.1) [RFC 9293 §3.6.1](https://www.rfc-editor.org/rfc/rfc9293#section-3.6.1)

## 建议的下一轮验证

以下是复核清单，不代表本次已运行：

1. 用受控的客户端与服务端做握手共读测试，分别验证 101 后同批到达首帧，以及 Upgrade 请求后同批到达帧的处理。
2. 在服务端发送 ping 后暂停应用数据，同时从 TCP 目标端发送数据，确认客户端泵是否出现反向流量停顿。
3. 覆盖掩码、RSV、控制帧长度/FIN、未知 opcode、起始/continuation 顺序、分片消息与 Close 交互，并将“拒绝”与“继续转发”行为记录下来。
4. 分别在 TCP 单向 EOF、WebSocket Close、RST 和空闲超时场景记录两端日志与 socket 关闭顺序。
5. 在反代层单独验证 Basic 或 mTLS 的配置，并确认 `ws-bridge.py` 本身只接受受控反代可达的连接。
6. 补齐作者实测的时间、软件版本、链路、样本量、测量命令和原始日志。A/B/C 各自标明实际执行与否。

这些是后续工程验证的建议；本次交付只修正文档中的结论范围和已知边界，没有重构脚本、调整网络配置或宣称问题已修复。
