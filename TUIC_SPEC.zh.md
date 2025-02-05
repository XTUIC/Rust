# TUIC协议

## 版本

`0x05`

## 概述

TUIC协议依赖于可多路复用的TLS加密流。所有的中继任务通过`Header`在`Command`中进行协商。

该协议不关心底层传输层。然而，它主要设计为与[QUIC](https://en.wikipedia.org/wiki/QUIC)一起使用。有关详细的机制，请参见[协议流程](#协议流程)。

所有字段默认采用大端字节序，除非另有说明。

## 命令

```plain
+-----+------+----------+
| VER | TYPE |   OPT    |
+-----+------+----------+
|  1  |  1   | Variable |
+-----+------+----------+
```

其中：

- `VER` - TUIC协议版本
- `TYPE` - 命令类型
- `OPT` - 命令类型特定数据

### 命令类型

共有五种命令类型：

- `0x00` - `Authenticate` - 用于验证多路复用流
- `0x01` - `Connect` - 用于建立TCP中继
- `0x02` - `Packet` - 用于中继（分片的）UDP数据包
- `0x03` - `Dissociate` - 用于终止UDP中继会话
- `0x04` - `Heartbeat` - 用于保持QUIC连接存活

命令`Connect`和`Packet`包含负载（流/数据包片段）

### 命令类型特定数据

#### `Authenticate`

```plain
+------+-------+
| UUID | TOKEN |
+------+-------+
|  16  |  32   |
+------+-------+
```

其中：

- `UUID` - 客户端UUID
- `TOKEN` - 客户端令牌。客户端的原始密码通过当前TLS会话使用[TLS密钥材料导出器](https://www.rfc-editor.org/rfc/rfc5705)哈希成256位长度的令牌。在导出时，`label`应为客户端UUID，`context`应为原始密码。

#### `Connect`

```plain
+----------+
|   ADDR   |
+----------+
| Variable |
+----------+
```

其中：

- `ADDR` - 目标地址。请参见[地址](#地址)

#### `Packet`

```plain
+----------+--------+------------+---------+------+----------+
| ASSOC_ID | PKT_ID | FRAG_TOTAL | FRAG_ID | SIZE |   ADDR   |
+----------+--------+------------+---------+------+----------+
|    2     |   2    |     1      |    1    |  2   | Variable |
+----------+--------+------------+---------+------+----------+
```

其中：

- `ASSOC_ID` - UDP中继会话ID。请参见[UDP中继](#udp-中继)
- `PKT_ID` - UDP数据包ID。请参见[UDP中继](#udp-中继)
- `FRAG_TOTAL` - UDP数据包的总分片数
- `FRAG_ID` - UDP数据包的分片ID
- `SIZE` - （分片的）UDP数据包长度
- `ADDR` - 目标（来自客户端）或源（来自服务器）地址。请参见[地址](#地址)

#### `Dissociate`

```plain
+----------+
| ASSOC_ID |
+----------+
|    2     |
+----------+
```

其中：

- `ASSOC_ID` - UDP中继会话ID。请参见[UDP中继](#udp-中继)

#### `Heartbeat`

```plain
+-+
| |
+-+
| |
+-+
```

### `Address`

`Address`是一个变长字段，用于编码网络地址

```plain
+------+----------+----------+
| TYPE |   ADDR   |   PORT   |
+------+----------+----------+
|  1   | Variable |    2     |
+------+----------+----------+
```

其中：

- `TYPE` - 地址类型
- `ADDR` - 地址
- `PORT` - 端口

地址类型可以是以下之一：

- `0xff`：无
- `0x00`：完全限定域名（第一个字节表示域名的长度）
- `0x01`：IPv4地址
- `0x02`：IPv6地址

地址类型`None`用于在`Packet`命令中，且该命令不是UDP数据包的第一个分片。

域名/IP地址之后使用2字节编码端口号。

## 协议流程

本节详细描述了以QUIC为底层传输的协议流程。

TUIC协议不关心底层传输的管理方式。它甚至可以集成到其他现有服务中，如HTTP/3。

以下是TUIC协议在QUIC连接中的典型流程：

### 认证

客户端打开一个`unidirectional_stream`并发送`Authenticate`命令。此过程可以与其他命令（中继任务）并行执行。

服务器接收`Authenticate`命令并验证令牌。如果令牌有效，则连接被认证，准备进行其他中继任务。

如果服务器在接收到`Authenticate`命令之前收到了其他命令，则应只接受命令头部分并暂停。在连接认证后，服务器应恢复所有暂停的任务。

### TCP中继

命令`Connect`用于初始化TCP中继。

客户端打开一个`bidirectional_stream`并发送`Connect`命令。命令头传输完成后，客户端可以开始使用该流进行TCP中继，而无需等待服务器的响应（实际上服务器不会响应）。

服务器接收`Connect`命令并打开到目标地址的TCP流。流建立后，服务器可以开始在TCP流和`bidirectional_stream`之间中继数据。

### UDP中继

TUIC通过在客户端和服务器之间同步UDP会话ID（关联ID），实现0-RTT完全锥形UDP转发。

客户端和服务器应该为每个QUIC连接创建一个UDP会话表，将每个关联ID映射到一个UDP套接字。

关联ID是客户端生成的一个16位无符号整数。如果客户端希望使用服务器的同一个套接字发送UDP数据包，则`Packet`命令中附加的关联ID应该相同。

当服务器接收到`Packet`命令时，应检查附带的关联ID是否已与UDP套接字关联。如果没有，服务器应为该关联ID分配一个UDP套接字。服务器将使用此UDP套接字发送客户端请求的UDP数据包，同时接收来自任何目标的UDP数据包，将它们加上`Packet`命令头后发送回客户端。

一个UDP数据包可以分割成多个`Packet`命令。字段`PKT_ID`、`FRAG_TOTAL`和`FRAG_ID`用于标识和重新组装分片的UDP数据包。

作为客户端，`Packet`可以通过以下方式发送：

- QUIC `unidirectional_stream`（UDP中继模式quic）
- QUIC `datagram`（UDP中继模式原生）

当服务器收到来自UDP中继会话（关联ID）的第一个`Packet`时，应使用相同的方式返回`Packet`命令。

通过客户端通过QUIC `unidirectional_stream`发送`Dissociate`命令，可以解除UDP会话。服务器将移除UDP会话并释放关联的UDP套接字。

### 心跳

当有任何正在进行的中继任务时，客户端应定期通过QUIC `datagram`发送`Heartbeat`命令，以保持QUIC连接的存活。

## 错误处理

请注意，任何命令都没有响应。如果服务器收到无效命令，或在处理过程中遇到任何错误（例如目标地址无法访问，认证失败），没有*标准*的处理方式。行为是由实现定义的。服务器可能会关闭QUIC连接，或者仅仅忽略该命令。

例如，如果服务器收到一个目标地址无法访问的`Connect`命令，它可能会关闭`bidirectional_stream`以指示错误。
