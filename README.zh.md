# XTUIC

精心设计的 0-RTT 代理协议

XTUIC 是 [advanced TUIC repo](https://github.com/itsusinn/tuic) 的一个分支。

相比原始版本，此分支的新特性包括：

- 树内 [docker 镜像构建](https://github.com/Itsusinn/tuic/pkgs/container/tuic-server)
- 最新依赖项
- 更加宽松的锁
- 通过 [cross-rs](https://github.com/cross-rs/cross) 实现更多 CI 目标
- 支持自签名证书和 `skip_cert_verify`
- ServerCert 自动热重载
- 及更多 [特性](https://github.com/EAimTY/tuic/compare/dev...Itsusinn:tuic:dev)

## 简介

XTUIC 是一个代理协议，旨在通过尽可能多的中继来最小化额外的握手延迟，同时保持协议本身简单易于实现。

XTUIC 最初设计用于在 [QUIC](https://en.wikipedia.org/wiki/QUIC) 协议之上使用，但理论上您可以将其与任何其他协议（例如 TCP）一起使用。

与 QUIC 配合使用时，XTUIC 可以实现：

- 0-RTT TCP 代理
- 具有 NAT 类型 [Full Cone](https://www.rfc-editor.org/rfc/rfc3489#section-5) 的 0-RTT UDP 代理
- 0-RTT 认证
- 两种 UDP 代理模式：
  - `native`：具有原生 UDP 机制的特点
  - `quic`：使用 QUIC 流无损地传输 UDP 数据包
- 完全复用
- QUIC 的所有优点，包括但不限于：
  - 双向用户空间拥塞控制
  - 可选的 0-RTT 连接握手
  - 连接迁移

可以在 [SPEC.md](https://github.com/XTUIC/Rust/blob/main/TUIC_SPEC.en.md) 中找到完整的 XTUIC 协议规范。

## 概述

该存储库提供 4 个板条箱：

- **[tuic](https://github.com/Itsusinn/tuic/tree/dev/tuic)** - 库。 协议本身、协议和模型抽象、同步/异步编组
- **[tuic-quinn](https://github.com/Itsusinn/tuic/tree/dev/tuic-quinn)** - 库。 [quinn](https://github.com/quinn-rs/quinn) 上的一层薄层，用于提供 TUIC 的功能
- **[tuic-server](https://github.com/Itsusinn/tuic/tree/dev/tuic-server)** - 二进制。 最小的 TUIC 服务器实现作为参考
- **[tuic-client](https://github.com/Itsusinn/tuic/tree/dev/tuic-client)** - 二进制。 最小的 TUIC 客户端实现作为参考

## 贡献 XTUIC

[搜索代码库中的 TODO](https://github.com/search?q=repo%3AItsusinn%2Ftuic%20todo&type=code) 或 [协助解决开放问题](https://github.com/Itsusinn/tuic/issues?q=label%3A%22help+wanted%22+is%3Aissue+is%3Aopen)

## 许可

此存储库中的代码遵循 [GNU 通用公共许可证 v3.0](https://github.com/Itsusinn/tuic/blob/dev/LICENSE)

然而，XTUIC 协议的概念是私有的。 您不能在没有任何限制的情况下实施、修改和重新分发协议，即使用于商业用途。
