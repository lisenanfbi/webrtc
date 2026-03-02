# Pion WebRTC 协议传输学习计划

## 学习目标
深入理解 WebRTC 协议栈的传输层实现，包括 ICE、DTLS、SCTP、SRTP 等核心协议的交互机制。

---

## 第一章：WebRTC 协议栈概述

### 1.1 背景知识
- **WebRTC 架构**：理解 PeerConnection、Track、DataChannel 等核心概念
- **协议分层**：
  ```
  应用层: DataChannel / Media (RTP/RTCP)
  传输层: SCTP (DataChannel) / SRTP (Media)
  安全层: DTLS
  网络层: ICE (STUN/TURN/UDP/TCP)
  ```
- **Pion 项目结构**：`peerconnection.go` 是核心入口，协调各传输层组件

### 1.2 代码阅读
- `peerconnection.go` (36-98行): PeerConnection 结构体定义
- `api.go`: API 构造和配置

### 1.3 实验
- 运行 `examples/data-channels` 和 `examples/play-from-disk` 示例
- 使用 Wireshark 抓包观察 WebRTC 连接建立过程

---

## 第二章：ICE (Interactive Connectivity Establishment)

### 2.1 背景知识
- **RFC 5245/8445**: ICE 协议规范
- **Candidate 类型**: Host、Server Reflexive (STUN)、Relay (TURN)
- **ICE 流程**: Gathering → Checking → Nomination → Connected

### 2.2 核心代码
| 文件 | 关键内容 |
|------|----------|
| `icegatherer.go` | ICE 候选地址收集 |
| `icetransport.go` | ICE 传输层管理 |
| `icecandidate.go` | 候选地址数据结构 |
| `internal/mux/mux.go` | 多路复用器 |

### 2.3 关键实现细节
```go
// ICETransport 结构 (icetransport.go:24-44)
type ICETransport struct {
    role ICERole
    gatherer *ICEGatherer
    conn     *ice.Conn  // 底层 ICE 连接
    mux      *mux.Mux   // 多路复用器
}
```

### 2.4 实验
- **实验 1**: 运行 `examples/trickle-ice`，观察候选地址收集过程
- **实验 2**: 运行 `examples/ice-tcp`，对比 UDP/TCP 传输差异
- **实验 3**: 运行 `examples/ice-single-port`，理解端口复用机制

---

## 第三章：DTLS (Datagram TLS)

### 3.1 背景知识
- **RFC 6347**: DTLS 1.2 规范
- **作用**: 为 UDP 提供加密和身份验证
- **握手过程**: ClientHello → ServerHello → Certificate → Finished
- **指纹验证**: a=fingerprint 在 SDP 中交换证书指纹

### 3.2 核心代码
| 文件 | 关键内容 |
|------|----------|
| `dtlstransport.go` | DTLS 传输层实现 |
| `certificate.go` | 证书生成和管理 |
| `dtlsrole.go` | DTLS 角色定义 (Client/Server/Auto) |

### 3.3 关键实现细节
```go
// DTLSTransport 结构 (dtlstransport.go:37-61)
type DTLSTransport struct {
    iceTransport          *ICETransport
    certificates          []Certificate
    remoteParameters      DTLSParameters
    srtpProtectionProfile srtp.ProtectionProfile
    conn *dtls.Conn      // 底层 DTLS 连接
    srtpReady chan struct{}
}
```

### 3.4 DTLS 与 SRTP 的关联
- DTLS 握手完成后导出 SRTP 密钥
- `startSRTP()` 方法启动 SRTP 会话 (dtlstransport.go:195)

### 3.5 实验
- **实验 1**: 运行 `examples/pion-to-pion`，观察 DTLS 握手过程
- **实验 2**: 修改 `examples/reflect`，添加 DTLS 状态监听
- **实验 3**: 使用 OpenSSL s_client 测试 DTLS 连接

---

## 第四章：SCTP (Stream Control Transmission Protocol)

### 4.1 背景知识
- **RFC 4960**: SCTP 协议规范
- **WebRTC 中的 SCTP**: 运行在 DTLS 之上 (RFC 8261)
- **特性**: 多流、有序/无序传输、拥塞控制、部分可靠性

### 4.2 核心代码
| 文件 | 关键内容 |
|------|----------|
| `sctptransport.go` | SCTP 传输层实现 |
| `datachannel.go` | DataChannel 实现 |
| `sctpcapabilities.go` | SCTP 能力协商 |

### 4.3 关键实现细节
```go
// SCTPTransport 结构 (sctptransport.go:24-58)
type SCTPTransport struct {
    dtlsTransport *DTLSTransport
    state SCTPTransportState
    sctpAssociation *sctp.Association
    dataChannels []*DataChannel
    maxChannels *uint16
}
```

### 4.4 实验
- **实验 1**: 运行 `examples/data-channels`，测试消息传输
- **实验 2**: 运行 `examples/data-channels-flow-control`，观察流量控制
- **实验 3**: 运行 `examples/data-channels-detach`，使用底层 SCTP API

---

## 第五章：SRTP/SRTCP (Secure RTP)

### 5.1 背景知识
- **RFC 3711**: SRTP 规范
- **作用**: 加密 RTP/RTCP 媒体数据
- **密钥管理**: 通过 DTLS 握手导出
- **保护配置文件**: SRTP_AES128_CM_HMAC_SHA1_80、SRTP_AEAD_AES_256_GCM

### 5.2 核心代码
| 文件 | 关键内容 |
|------|----------|
| `srtp_writer_future.go` | SRTP 写入流 |
| `dtlstransport.go` | SRTP 会话管理 (195-250行) |
| `rtpsender.go` | RTP 发送器 |
| `rtpreceiver.go` | RTP 接收器 |

### 5.3 关键实现细节
```go
// trackStreams 结构 (rtpreceiver.go:28-45)
type trackStreams struct {
    track *TrackRemote
    rtpReadStream  *srtp.ReadStreamSRTP
    rtcpReadStream *srtp.ReadStreamSRTCP
    repairReadStream *srtp.ReadStreamSRTP  // RTX 重传流
}
```

### 5.4 实验
- **实验 1**: 运行 `examples/rtp-forwarder`，转发 RTP 包
- **实验 2**: 运行 `examples/rtcp-processing`，处理 RTCP 反馈
- **实验 3**: 运行 `examples/save-to-disk`，保存解密的媒体流

---

## 第六章：多路复用 (Mux)

### 6.1 背景知识
- **RFC 7983**: DTLS/SRTP 多路复用
- **原理**: 通过第一个字节区分协议类型
  - [0-3]: STUN
  - [20-63]: DTLS
  - [128-191]: RTP/RTCP

### 6.2 核心代码
| 文件 | 关键内容 |
|------|----------|
| `internal/mux/mux.go` | 多路复用器实现 |
| `internal/mux/endpoint.go` | 虚拟连接端点 |
| `internal/mux/muxfunc.go` | 协议匹配函数 |

### 6.3 关键实现细节
```go
// Mux 结构 (internal/mux/mux.go:35-46)
type Mux struct {
    nextConn   net.Conn      // 底层 ICE 连接
    endpoints  map[*Endpoint]MatchFunc  // 端点映射
    pendingPackets [][]byte  // 待处理包队列
}

// 协议匹配函数 (internal/mux/muxfunc.go:38-48)
func MatchDTLS(b []byte) bool {
    return MatchRange(20, 63, b)  // DTLS 包范围
}
func MatchSRTPOrSRTCP(b []byte) bool {
    return MatchRange(128, 191, b)  // RTP/RTCP 包范围
}
```

### 6.4 实验
- **实验 1**: 阅读 `internal/mux/mux_test.go`，理解测试用例
- **实验 2**: 修改匹配函数，添加自定义日志观察包分发

---

## 第七章：SDP 与信令

### 7.1 背景知识
- **RFC 4566**: SDP 会话描述协议
- **JSEP**: JavaScript Session Establishment Protocol
- **SDP 内容**: ICE 候选、DTLS 指纹、媒体编解码、SSRC 等

### 7.2 核心代码
| 文件 | 关键内容 |
|------|----------|
| `sdp.go` | SDP 解析和生成 |
| `sessiondescription.go` | 会话描述结构 |
| `signalingstate.go` | 信令状态机 |

### 7.3 关键实现细节
```go
// trackDetails 结构 (sdp.go:26-35)
type trackDetails struct {
    mid      string
    kind     RTPCodecType
    streamID string
    id       string
    ssrcs    []SSRC
    rtxSsrc  *SSRC
    fecSsrc  *SSRC
    rids     []string
}
```

### 7.4 实验
- **实验 1**: 运行 `examples/ortc`，使用 ORTC API 手动协商
- **实验 2**: 分析 `examples/play-from-disk` 生成的 SDP
- **实验 3**: 实现自定义信令服务器

---

## 第八章：高级主题

### 8.1 重协商 (Renegotiation)
- **代码**: `peerconnection_renegotiation_test.go`
- **实验**: 运行 `examples/play-from-disk-renegotiation`

### 8.2 Simulcast
- **代码**: `rtptransceiver.go`, `rtpreceiver.go`
- **实验**: 运行 `examples/simulcast`

### 8.3 带宽估计与拥塞控制
- **代码**: `interceptor.go`, `settingengine.go`
- **实验**: 运行 `examples/bandwidth-estimation-from-disk`

### 8.4 虚拟网络测试
- **代码**: `vnet_test.go`
- **实验**: 运行 `examples/vnet`

---

## 学习路径建议

```
第 1-2 周: 第一章 + 第二章 (ICE)
第 3-4 周: 第三章 (DTLS)
第 5-6 周: 第四章 (SCTP)
第 7-8 周: 第五章 (SRTP)
第 9-10 周: 第六章 (Mux) + 第七章 (SDP)
第 11-12 周: 第八章 (高级主题)
```

## 推荐资源

1. **书籍**: [WebRTC for the Curious](https://webrtcforthecurious.com)
2. **RFC 文档**: RFC 5245 (ICE), RFC 6347 (DTLS), RFC 4960 (SCTP), RFC 3711 (SRTP)
3. **Pion 文档**: [pkg.go.dev](https://pkg.go.dev/github.com/pion/webrtc/v4)
4. **示例代码**: `examples/` 目录下的完整示例
