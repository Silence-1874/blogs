### 实验大意
实现`TCPConnection`，将前面实现的`TCPReceiver`和`TCPSender`整合起来。

#### 接收报文基本规则
- 如果设置了`RST`标识，将输入流和输出流都设置成错误状态，并永久关闭`TCPConnection`。

- 将报文传送给`TCPReceiver`，使其得到其所关心的**seqno,SYN,payload,FIN**。

- 如果设置了`ACK`标识，使`TCPSender`得到其所关心的**ackno**和**window_size**。

- 如果收到的报文占据序列号，确保至少发送一个回应报文，以便更新**ackno**和**window_size**。

- `TCPConnection`中的`segment_received()`需要负责 **“keep-alive”** 报文。对等方可能会发送一个带无效序列号的报文，以确定TCP实现是否仍然活跃（如果活跃，还会获得当前窗口大小）。就算这些“keep-alive”报文不占据任何序列号，`TCPConnection`也需要接收。

#### 发送报文基本原则
- 任何时候`TCPSender`发送一个报文时，都需要设置**seqno,SYN,payload,FIN**位。
  
- 在发送报文前，`TCPConnection`会向`TCPReceiver`询问**ackno**和**window_size**。如果有**ackno**，就设置**ACK**位和其他的字段。

#### 时间流逝基本原则
操作系统会周期性调用`TCPConnection`中的`tick`函数，每当调用发生时，`TCPConnection`需要：

- 告诉`TCPSender`经过的时间。

- 如果连续重传次数超过了上限(`TCPConfig::MAX_RETX_ATTEMPTS`)，废弃连接并发送重置报文(只设置了`RST`标识的空报文)。

- 如果需要的话，干净地结束连接。

### FAQ
> 代码量大概多少？

`tcp_connection.cc`的代码量在100-150行左右。

> 我应该怎样开始？

首先通过`TCPSender`和`TCPReceiver`实现“ordinary”方法(`remaining outbound capacity()`, `bytes in flight()`, `unassembled bytes()`)，然后可以选择实现“writer”方法(`connect()`, `write()`,
, `end input stream()`)。其中一些方法可能需要对`TCPSender`中的`ByteStream`做修改。

> 应用怎样读取输入流数据？

`TCPConnection::inbound_stream()`已经在头文件中实现了，不需要再做其他改变了。

> `TCPConnection`需要任何特别的数据结构或算法吗？

不需要。大部分繁重的工作都已经在`TCPSender`和`TCPReceiver`中实现了。

> `TCPConnection`怎么发送一个报文？

跟`TCPSender`很像，将报文放入`_segment_out`队列。

> `TCPConnection`怎么知道时间的流逝？

跟`TCPSender`很像，会周期性调用`tick()`。

> 如果收到了**RST报文**，`TCPConnection`要怎么做？

将输入流和输出流都设置为`error`，接下来任何一次调用`TCPConnection::active()`时都返回false。

> 什么时候应该发送**RST报文**?

1. 失败的连续重传次数过多(超过了`TCPConfig::MAX_RETX_ATTEMPTS`)。

2. 当连接处于活跃状态时调用了`TCPConnection`的析构函数。

> **RST报文**的序列号应该怎样设置？

每一个发送出去的报文都应该有合适的序列号。你可以使用`send_empty_segment()`发送一个带有合适序列号的空报文，也可以通过`fill_window()`函数来填充窗口。

> ACK标识位有什么意义？难道不是每次都有确认号吗？

几乎所有的报文都有确认号并设置了`ACK`标识位，除了最开始建立连接的报文。

对于发出的报文，都尽可能设置确认好与`ACK`标识位。

对于收到的报文，只有当`ACK`标识位已设置时才会关注确认号，并将确认号和窗口大小发送给`TCPSender`。

> 怎样弄清楚各种“state”的名字？

看Lab2与Lab3讲义中的图。

> 如果`TCPReceiver`打算报告的窗口大小大于`TCPSegment::header().win`，我需要发送多大的窗口？

发送尽可能最大的值。

### 实验总结
这个Lab的讲义就老长老长的，光是FAQ就一大堆，特别劝退。

我已经把我能写的都写了，就这样吧，暂告一段落了。