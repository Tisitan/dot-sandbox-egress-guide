# 协议与产品来源

本页列出指南中可由公开标准或维护方文档核对的事实。核查基准日期为 2026-10-11；在线产品文档可能继续更新。标准定义支持一般性描述，不证明某一实际网络路径使用了特定产品、协议版本或 TLS 终止位置。

## 协议

- [RFC 6455：The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455) 定义 WebSocket 帧、握手、masking、控制帧及关闭过程。客户端到服务器帧必须 masked；控制帧不可分片且 payload 最多 125 字节。WSS 使用 TLS。传统 HTTP/1.1 握手使用 Upgrade。
- [RFC 8441：WebSocket over HTTP/2](https://www.rfc-editor.org/rfc/rfc8441) 与 [RFC 9220：WebSocket over HTTP/3](https://www.rfc-editor.org/rfc/rfc9220) 分别定义通过扩展 CONNECT 在 HTTP/2 和 HTTP/3 上建立 WebSocket 的方式。参见 [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113)（HTTP/2 over TCP）、[RFC 9114](https://www.rfc-editor.org/rfc/rfc9114)（HTTP/3 over QUIC）和 [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000)（QUIC 使用 UDP 数据报）。这些规范说明可选的协议层次，不确定某次连接实际采用的拓扑。
- [RFC 9110 §15.6.3](https://www.rfc-editor.org/rfc/rfc9110#section-15.6.3) 将 502 定义为充当网关或代理的服务器从其访问的上游服务器收到无效响应。状态码本身不指出代理链中的具体实例或底层原因；“不能仅凭 502 唯一定位”是据标准定义作出的诊断推断。
- [RFC 7617：Basic Authentication](https://www.rfc-editor.org/rfc/rfc7617) 规定 HTTP Basic 凭据编码；Base64 不是加密。[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) 定义源站 401 与代理 407 挑战。[RFC 8446](https://www.rfc-editor.org/rfc/rfc8446) 定义 TLS 1.3 客户端证书认证。认证身份由接收相应 HTTP 头或终止相应 TLS 会话的一端验证。
- [RFC 9293 §3.6.1](https://www.rfc-editor.org/rfc/rfc9293#section-3.6.1) 允许 TCP 两个方向分别关闭；[RFC 6455 §5.5.1](https://www.rfc-editor.org/rfc/rfc6455#section-5.5.1) 的 WebSocket Close 是应用协议控制帧与关闭握手。两者不等价。

## 产品文档

- [Let’s Encrypt：6-day and IP Address Certificates are Generally Available](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability)，发布于 2026-01-15、更新于 2026-03-11：IP 地址证书已一般提供，但必须采用 `shortlived` profile；短期证书有效期为 160 小时。IP 证书的验证方式限 HTTP-01 或 TLS-ALPN-01，不支持 DNS-01。该 CA 能力不代表任意 ACME 客户端或部署流程已支持。
- [mitmproxy 当前 stable 协议文档](https://docs.mitmproxy.org/stable/concepts/protocols/)、[事件 API](https://docs.mitmproxy.org/stable/api/events.html) 与[证书文档](https://docs.mitmproxy.org/stable/concepts/certificates/)说明其 WebSocket 解析与 TLS 检查能力及限制。TLS 内容检查要求客户端信任 mitmproxy CA；具体行为取决于版本、模式和协商扩展。
- [Cloudflare WebSockets 文档](https://developers.cloudflare.com/network/websockets/)（页面标注 2026-08-14 更新）说明网络更新重启及空闲超时都可能终止 WebSocket。RFC 6455 §7.2.1 规定底层传输意外丢失时连接失败；重连或会话恢复是应用设计责任，不是该产品文档承诺的自动恢复行为。

## 本稿所审查的固定公开版本

- [dot-sandbox-egress-guide](https://github.com/Tisitan/dot-sandbox-egress-guide/tree/5e7bc0806271ce0a3ef0c1e8d2f2b84a1a67c583)，固定提交 `5e7bc0806271ce0a3ef0c1e8d2f2b84a1a67c583`。
- [mc-over-wss-relay](https://github.com/Tisitan/mc-over-wss-relay/tree/6b98173e835a8f7d23166669878032e49c821b55)，固定提交 `6b98173e835a8f7d23166669878032e49c821b55`。

以上链接锁定本稿的文档与代码审查基线，不代表后续仓库状态或本地部署配置。
