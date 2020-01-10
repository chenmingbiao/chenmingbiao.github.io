---
layout:     post
title:      "SDP 会话描述协议"
subtitle:   "Session Description Protoocol"
date:       2020-01-10
header-img: "img/post-bg-infinity.jpg"
author:     "CMB"
tags:
    - WebRTC
    - SDP
    - 协议

---

#### SDP 简介

SDP（Session Description Protoocol）是一种通用的会话描述协议，主要用来描述多媒体会话，例如会话声明、会话邀请、会话初始化等。

WebRTC 主要在连接建立的时候使用到 SDP 协议，用于连接双方通过信令服务来交互会话信息的一种协议，包括解码器、网络传输协议等。

#### 协议格式

SDP 协议格式非常简单，就是多行的 `key-value` 组成

```
<type>=<value>
```

其中：

* `<type>` ：属性（大小写敏感），例如 `v` 代表版本；
* `<value>` ：内容，它是结构化文本，对应的格式和属性关联，采用 UTF8 编码；
* `=`  ：符号，两边不能存在空格；

* `=*` ：表示可选。

例子：

```
v=0
o=tom 1578641469 1578641475 IN IP4 www.host.com
s=session name
c=IN IP4 www.host.com
t=0 0
m=audio 49170 RTP/AVP 0
a=rtpmap:0 PCMU/8000
m=video 51372 RTP/AVP 31
a=rtpmap:31 H261/90000
m=video 53000 RTP/AVP 32
a=rtpmap:32 MPV/90000
```

#### 属性

##### 协议版本号：`v`

必填项，格式如下：

```shell
v=0
```

> 不存在子版本号

##### 会话发起者：`o`

必填项，格式如下：

```shell
o=<username> <sess-id> <sess-version> <nettype> <addrtype> <unicast-address>
```

其中：

* `username` ：发起者的用户名，如果没有用户名则用 `-`；
* `sess-id` ： 会话id，规范是使用时间戳，可以自由定义；
* `sess-version` ： 会话版本，会话数据变化时发生变化，规范是使用时间戳，可以自由定义，一般是自增；
* `nettype` ：网络类型，例如 `IN` 表示 `Internet`；
* `addrtype` ：地址类型，例如 `IP4`；
* `unicast-address` ：域名/IP地址。

##### 会话名：`s`

必填项，格式如下：

```shell
s=sessionName
```

> 一般不为空，空的话用空格表示

##### 连接数据：`c`

格式如下：

```shell
c=<nettype> <addrtype> <connection-address>
```

其中：

* `nettype` ：网络类型，例如 `IN` 表示 `Internet`；
* `addrtype` ：地址类型，例如 `IP4`；
* `connection-address` ：如果是广播，则为广播地址，如果是单播，则为单播地址。

> 每个 SDP 至少需要包含一个会话级别的 `c` 字段，或者在每个媒体描述后面各包含一个 `c` 字段，媒体描述后的 `c`  会覆盖会话级别的 `c`

##### 媒体描述：`m`

格式如下：

```shell
m=<media> <port> <proto> <fmt> ...
```

其中：

* `media` ：媒体类型，例如video、audio、text等；
* `port` ：传输媒体流端口号，具体看 `m`  声明和 `proto` 字段；
* `proto` ：传输协议，具体取决于 `c` 中定义的地址类型，例如 `c` 是 `IP4`，这里就是传输协议运行在 `IP4` ，比如：
  * `UDP` ;
  * `RTP/AVP` ：针对视频、音频的 `RTP` 协议，在 `UDP` 之上；
  * `RTP/SAVP`：针对视频、音频的 `SRTP` 协议，在 `UDP` 之上。
* `fmt` ：媒体格式描述，可存在多个。根据 `proto` 的不同，`fmt` 的含义也不同，例如 `proto` 为 `RTP/SAVP` 时，`fmt` 表示 `RTP payload` 的类型，如果有多个，表示在这次会话中，多种 `payload` 类型可能会用到，且第一个为默认的 `payload` 类型。
  * 对于 RTP/SAVP，payload type 分两种类型：
    * 静态类型：[参考](https://en.wikipedia.org/wiki/RTP_payload_formats#RTP/AVP_audio_and_video_payload_types)
    * 动态类型：在 `a=fmtp` 里定义

例子：

1. `audo`，111 为动态类型，表示 `opus/48000/2`：

```shell
m=audio 9 UDP/TLS/RTP/SAVP 111 103 104 9 0 8 126
a=rtpmap:111 opus/48000/2
```

2. `video`：122 为动态类型，表示 `H264/90000` ： 

```shell
m=video 9 UDP/TLS/RTP/SAVP 122 102 100 101 124 120 123 119
a=rtpmap:122 H264/90000
```

##### 附加属性：`a`

格式如下：

```
a=<attribute>
a=<attribute>:<value>
```

作用：用于扩展 SDP

有两种作用范围：会话级别、媒体级别

1. 媒体级别：媒体描述 `m` 后面可以跟任意数量的 `a` 字段，对媒体描述进行扩展；
2. 会话级别：在第一个媒体字段钱，添加就是会话级别

例子：

媒体级别：

```shell
a=rtpmap:0 PCMU/8000
```

会话级别：

```shell
a=recvonly
```

##### 时间：`t`

格式如下：

```shell
t=<start-time> <stop-time>
```

作用：记录会话开始和结束的时间

* `start-time` ：0 表示永久会话；

* `stop-time` ：0 表示会话没有边界，但是会话要在 `start-time` 之后才算事 `active` 状态。

##### 其他：

不一一列出来，具体看[Session Description Protocol](https://en.wikipedia.org/wiki/Session_Description_Protocol)

#### 相关链接：

* [Annotated Example SDP for WebRTC](https://datatracker.ietf.org/doc/draft-ietf-rtcweb-sdp/)

* [SDP: Session Description Protocol](https://tools.ietf.org/html/rfc4566)