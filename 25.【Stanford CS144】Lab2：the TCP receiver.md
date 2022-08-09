刚初始化项目就报错了：

```sh
$ make
--   NOTE: You can choose a build type by calling cmake with one of:
--     -DCMAKE_BUILD_TYPE=Release   -- full optimizations
--     -DCMAKE_BUILD_TYPE=Debug     -- better debugging experience in gdb
--     -DCMAKE_BUILD_TYPE=RelASan   -- full optimizations plus address and undefined-behavior sanitizers
--     -DCMAKE_BUILD_TYPE=DebugASan -- debug plus sanitizers
CMake Error: The following variables are used in this project, but they are set to NOTFOUND.
Please set them or make sure they are set and tested correctly in the CMake files:
LIBPCAP
    linked by target "tcp_parser" in directory /root/sponge/tests
    linked by target "tcp_parser" in directory /root/sponge/tests

-- Configuring incomplete, errors occurred!
See also "/root/sponge/build/CMakeFiles/CMakeOutput.log".
make: *** [cmake_check_build_system] Error 1
```

咱也不太懂Cmake的东西，猜想是之前用VS Code远程调试，可能不小心搞乱了什么文件，只好用万能的“remake”了。

把文件夹删了，重新`git clone`了Lab文件夹，然后把之前写好的代码直接复制过去，再编译Lab2，还是一样。

网上找了一圈也没找到个解决方法，于是去查了查之前收集的别人的CS144学习笔记，在[这一篇](https://www.cnblogs.com/kangyupl/p/stanford_cs144_labs.html)中找到了解决方法，安装好`libpcap-devel`库，终于成功编译。

出师不利啊……

### Debug设置
用默认编译选项的时候，总是会自动优化，Debug时很多变量都看不到了，断点的位置也不对。

之前几次Lab也还好，变量不是很多，但这个实验我真的调不动了，查了一下解决方案，只要在cmake时命令行加一个参数就好了，如下：

```sh
cmake -DCMAKE_BUILD_TYPE=Debug ..
```

这样终于能显示完整的调试信息了。

之后再改回**Release**级别就好了。

```sh
cmake -DCMAKE_BUILD_TYPE=Release ..
```

### 3.1 Translating between 64-bit indexes and 32-bit seqnos
#### 实验大意
实现64位的**流索引(Stream Index)**、32位的**序列号(Sequence Numbers/Seqno)**、32位的**绝对序列号(Absolute Sequence Numbers/Absolute Seqno)** 之间的转换。

根据实验讲义，假设传输的数据为字符串`cat`，seqno从`2^32-2`开始，则三者的关系如下：

|element        |SYN   |c     |a|t|FIN|
|---------------|------|------|-|-|---|
|seqno          |2^32-2|2^32-1|0|1|2  |
|absolute seqno |0     |1     |2|3|4  |
|stream index   |  ——  |0     |1|2|—— |

本实验主要完成`seqno`与`absolute seqno`之间的相互转换。

首先要明确范围，`seqno`是32位无符号整数，超出2^32位后溢出，再次从0开始计数。

`absolute seqno`是64位无符号整数，范围相当大，讲义中说可以假设其不会溢出。

可以得出转换关系:
$$
seqno = (absolute\ seqno) mod (1 >> 32) + isn
$$

#### wrap
实现`absolute seqno`到`seqno`的转换。

由于`absolute seqno`的范围比较大，而且无符号32位整数溢出相当于自动模2^32，可以根据公式直接计算。

#### unwrap
实现`seqno`到`absolute seqno`的转换。

由较小范围的数转换为较大范围的数，转换结果不是唯一的，因此给出`checkpoint`，求出与`checkpoint`差值最小的`absolute seqno`的值。

首先求出`isn`到`seqno`的偏移量，记为`offset`，则

$$
offset = (seqno + 2^{32} -isn) mod (2^{32})
$$

由于是无符号整数类型，溢出后实际上相当于自动取余了，可以直接用`seqno-isn`来计算`offset`（当然，offset是`uint32_t`类型的）。

接下来，找出所有可能的`absolute seqno`，记为`x`。计算所有`x`与`offset`的差的绝对值，其中绝对值最小的`x`即为答案。

`absolute seqno`为了加快查找速度，不需要真的找出所有的`absolute seqno`，只要能找到`absolute seqno`前后最近的`x`即可，也就是恰好将`absolute seqno`夹在中间的两个`x`。

先将`checkpoint`整除2^32，再乘2^32，最后加上`offset`，此时得到的一定是两个`x`之一，再判断是比`checkpoint`大的`x`还是比它小的`x`，直接做差比较即可得到答案。

#### 小坑
- 计算2^32次时，使用左移运算符，直接用`1<<32`会溢出，看了测试代码才知道要加上`ul`，即`1ul<<32`，`ul`表示`unsigned long`，即无符号长整数。

- checkpoint比offset小时，直接返回offset。

#### 实验结果
```
Test project /root/sponge/build
    Start 1: t_wrapping_ints_cmp
1/4 Test #1: t_wrapping_ints_cmp ..............   Passed    0.00 sec
    Start 2: t_wrapping_ints_unwrap
2/4 Test #2: t_wrapping_ints_unwrap ...........   Passed    0.00 sec
    Start 3: t_wrapping_ints_wrap
3/4 Test #3: t_wrapping_ints_wrap .............   Passed    0.00 sec
    Start 4: t_wrapping_ints_roundtrip
4/4 Test #4: t_wrapping_ints_roundtrip ........   Passed    0.18 sec

100% tests passed, 0 tests failed out of 4

Total Test time (real) =   0.21 sec
```

### 3.2 Implementing the TCP receiver
实现一些函数。

#### segment_received()
如果收到**SYN**报文，初始化**ISN**，并将自定义的`_syn`成员变量置为true，表示已经初始化isn，供下面的方法使用。

`TCPSegment`类的成员中有一个`Buffer`类，`Buffer`类中又有两个成员：`_storage`和`_starting_offset`。

为了写入数据，需要使用在Lab1中实现的`StreamReassembler`类，回顾其中的`push_substring()`方法，需要三个参数。

```cpp
    void push_substring(const std::string &data, const uint64_t index, const bool eof);
```

`data`就是`Buffer`里的`_storage`（需要将`string_view`类型转换成`string`类型），`eof`根据标志位`FIN`来确定即可，接下来重点是`index`的确定。

为了得到`index`，需要用到上面的表格，因此只要得到`seqno`或者`absolute seqno`即可。

`absolute seqno`可以根据前面实现的`unwrap`函数得到。

还要注意，如果`_syn`为false，说明没有建立连接，函数直接返回，丢弃改报文。

#### ackno()
首先检查`_syn`，如果为false，说明还没有初始化**ISN**，直接返回`nullopt`。

**ACK**的计算也比较简单，只需要计算**初始序列号+已写入的字节数+1**，加1是最初的**SYN**报文会占一个序列号。另外，如果流已经关闭，说明连接已经关闭，已经收到了**FIN**报文，而**FIN**报文也会占一个序列号，需要+1。

#### window_size()
讲义已经提示了，窗口大小是**first unacceptable - first unassembled**，但是**first unacceptable**不好计算。

而程序中已经给出了**capacity**的值，根据Lab1的那张图片，可以得到窗口大小的另一个计算式**capacity - (first unassembled - first unread)**，接下来就需要确定**first unassembled - first unread**该如何表达。

在Lab0中，实现了`buffer_size()`函数，这个函数返回的是**可以读取的最大字节数**，正好就是**first unassembled - first unread**的意义。

窗口大小的表达式也就呼之欲出了：

$$
window\ size = capacity - buffer\ size
$$

#### 小坑
实验中的测试报文**SYN**和**FIN**标识位会同时开启，而且都可能携带数据，需要格外注意。

不过Lab1中已经处理了大部分的情况了，因此这里的实现会轻松不少。

#### 实验结果
```
[100%] Testing the TCP receiver...
Test project /root/sponge/build
      Start  1: t_wrapping_ints_cmp
 1/26 Test  #1: t_wrapping_ints_cmp ..............   Passed    0.00 sec
      Start  2: t_wrapping_ints_unwrap
 2/26 Test  #2: t_wrapping_ints_unwrap ...........   Passed    0.00 sec
      Start  3: t_wrapping_ints_wrap
 3/26 Test  #3: t_wrapping_ints_wrap .............   Passed    0.00 sec
      Start  4: t_wrapping_ints_roundtrip
 4/26 Test  #4: t_wrapping_ints_roundtrip ........   Passed    0.19 sec
      Start  5: t_recv_connect
 5/26 Test  #5: t_recv_connect ...................   Passed    0.00 sec
      Start  6: t_recv_transmit
 6/26 Test  #6: t_recv_transmit ..................   Passed    0.07 sec
      Start  7: t_recv_window
 7/26 Test  #7: t_recv_window ....................   Passed    0.00 sec
      Start  8: t_recv_reorder
 8/26 Test  #8: t_recv_reorder ...................   Passed    0.00 sec
      Start  9: t_recv_close
 9/26 Test  #9: t_recv_close .....................   Passed    0.00 sec
      Start 10: t_recv_special
10/26 Test #10: t_recv_special ...................   Passed    0.00 sec
      Start 18: t_strm_reassem_single
11/26 Test #18: t_strm_reassem_single ............   Passed    0.00 sec
      Start 19: t_strm_reassem_seq
12/26 Test #19: t_strm_reassem_seq ...............   Passed    0.00 sec
      Start 20: t_strm_reassem_dup
13/26 Test #20: t_strm_reassem_dup ...............   Passed    0.01 sec
      Start 21: t_strm_reassem_holes
14/26 Test #21: t_strm_reassem_holes .............   Passed    0.00 sec
      Start 22: t_strm_reassem_many
15/26 Test #22: t_strm_reassem_many ..............   Passed    1.10 sec
      Start 23: t_strm_reassem_overlapping
16/26 Test #23: t_strm_reassem_overlapping .......   Passed    0.00 sec
      Start 24: t_strm_reassem_win
17/26 Test #24: t_strm_reassem_win ...............   Passed    1.34 sec
      Start 25: t_strm_reassem_cap
18/26 Test #25: t_strm_reassem_cap ...............   Passed    0.11 sec
      Start 26: t_byte_stream_construction
19/26 Test #26: t_byte_stream_construction .......   Passed    0.00 sec
      Start 27: t_byte_stream_one_write
20/26 Test #27: t_byte_stream_one_write ..........   Passed    0.00 sec
      Start 28: t_byte_stream_two_writes
21/26 Test #28: t_byte_stream_two_writes .........   Passed    0.00 sec
      Start 29: t_byte_stream_capacity
22/26 Test #29: t_byte_stream_capacity ...........   Passed    0.67 sec
      Start 30: t_byte_stream_many_writes
23/26 Test #30: t_byte_stream_many_writes ........   Passed    0.01 sec
      Start 53: t_address_dt
24/26 Test #53: t_address_dt .....................   Passed    0.01 sec
      Start 54: t_parser_dt
25/26 Test #54: t_parser_dt ......................   Passed    0.00 sec
      Start 55: t_socket_dt
26/26 Test #55: t_socket_dt ......................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 26

Total Test time (real) =   3.55 sec
[100%] Built target check_lab2
```

### 实验总结
前前后后断断续续花了四五天才做完，其实这个实验难度并不大，但这段时间突发事情太多了，打乱了我的节奏。

之后尽量保持高效一点吧。