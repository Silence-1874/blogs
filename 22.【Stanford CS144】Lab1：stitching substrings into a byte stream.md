### 实验大意
实现一个*流重组器(StreamReassembler)*，处理收到报文段的乱序、重复、重叠等问题。（丢包暂时不用考虑，因为本次实验重点在于接收方）

### 前期思考:
- 因为对C++的库不熟，一开始想直接用数组来充当哈希表，但index到后面会越来越大，数组空间肯定不够用，只好先去学一下C++的`map`了。

- 确定好使用`map`了，但`map`的键值对应该要存储什么呢？最开始我是打算学类似于操作系统磁盘空闲空间管理，键值对分别存储**空闲区域的起始下标**和**空闲区域的结束下标**，但是在实现的过程中遇到了很多困难。最主要的是每次`write()`有点不知所措，`write()`需要传入字符串，因此必须把每次接收到的字符串存储下来。参考了一些网上的文章，才发现自己想复杂了，`map`直接存储`每个字符的绝对索引`和`这个字符`即可。

- 接下来重点研究讲义里的这张图片。

  ![](https://s2.loli.net/2022/03/30/jl2a84Ag6ZzcbNL.png)

  最初笔者定义的成员变量里包括了`first_unread`和`first_unassembled`，但是头文件里并没有给出类似于`read`的函数，这样我就无从得知`first_unread`该如何变化了。
  
  重新看了一遍讲义，发现其中有一句

  > The owner can access and read from the ByteStream whenever it wants.

  也就是说，**上层**会**随时**read已经装配好的（绿色）部分。所以`first_unread`的值是不能明确得到的。

- 由上图可得，

  **first unread + capacity = first unacceptable**

  如果不确定`first_unread`，就无从得知`first_unacceptable`了，也就没法判断接收到的报文段是否超出范围，因此还得另辟蹊径。

- 突然发现还有一个熟悉的身影——`ByteStream`。

  我们在Lab0中已经实现了它。其中有一个成员变量，`_total_read`（这个变量名是笔者在Lab0中自己取的，对应的get函数为`bytes_read()`），表示字节流已经读取的字符数。再根据`index`的映射关系，就可以确定`first_unread`了。

  因为何时read是由上层决定的，所以`first_unread`应该由`ByteStream`类型的`_output`计算得来。

  同理，`first_unassembled`也可以通过`_total_written`（对应get函数`bytes_written()`)计算出来。

  现在已经可以确定图中的三个分界点了。

### 总体思路
- 用`_map`来管理为装配的（红色）部分，键为**绝对索引**，值为一个个**字符**。

- 对于新加入的字符串，将其字符**逐个**添加到`_map`中，因为`inisert()`方法会覆盖相同key的value，所以自然就解决了重复和重叠的问题。对于超出`first_unacceptable`的部分，直接丢弃。

- 从`first_unassembled`开始，将后面**连续的字符**组装成字符串，调用`_output`中的`write()`写入。注意，每组装一个字符，就从`_map`中将其键值对删除。

- 如果出现**eof消息**，不能立刻结束，而是要先记录下来，存入自定义的`_eof`中。因为这个eof消息可能是因乱序而提前到达的，也可能是因为超出了容量，导致不能写完整个字符串的。因此还需要记录**最后一个字符**的绝对索引，记为`_last`。

- 如果`_eof`为true，并且`_last`等于**未装配的第一个字符的绝对索引减1**，说明已经全部装配完毕，这时才真正结束，调用`_output`中的`end_input()`方法关闭流。

### 实验结果
```
[100%] Testing the stream reassembler...   
Test project /root/sponge/build                                                                                         
      Start 18: t_strm_reassem_single                                                                                   
 1/16 Test #18: t_strm_reassem_single ............   Passed    0.00 sec                                                 
      Start 19: t_strm_reassem_seq                                                                                      
 2/16 Test #19: t_strm_reassem_seq ...............   Passed    0.00 sec                                                 
      Start 20: t_strm_reassem_dup                                                                                      
 3/16 Test #20: t_strm_reassem_dup ...............   Passed    0.01 sec                                                 
      Start 21: t_strm_reassem_holes                                                                                    
 4/16 Test #21: t_strm_reassem_holes .............   Passed    0.00 sec                                                 
      Start 22: t_strm_reassem_many
 5/16 Test #22: t_strm_reassem_many ..............   Passed    1.32 sec
      Start 23: t_strm_reassem_overlapping
 6/16 Test #23: t_strm_reassem_overlapping .......   Passed    0.01 sec
      Start 24: t_strm_reassem_win
 7/16 Test #24: t_strm_reassem_win ...............   Passed    1.34 sec
      Start 25: t_strm_reassem_cap
 8/16 Test #25: t_strm_reassem_cap ...............   Passed    0.11 sec
      Start 26: t_byte_stream_construction
 9/16 Test #26: t_byte_stream_construction .......   Passed    0.00 sec
      Start 27: t_byte_stream_one_write
10/16 Test #27: t_byte_stream_one_write ..........   Passed    0.00 sec
      Start 28: t_byte_stream_two_writes
11/16 Test #28: t_byte_stream_two_writes .........   Passed    0.01 sec
      Start 29: t_byte_stream_capacity
12/16 Test #29: t_byte_stream_capacity ...........   Passed    0.60 sec
      Start 30: t_byte_stream_many_writes
13/16 Test #30: t_byte_stream_many_writes ........   Passed    0.01 sec
14/16 Test #53: t_address_dt .....................   Passed    0.03 sec
      Start 54: t_parser_dt
15/16 Test #54: t_parser_dt ......................   Passed    0.00 sec
      Start 55: t_socket_dt
16/16 Test #55: t_socket_dt ......................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 16

Total Test time (real) =   3.47 sec
[100%] Built target check_lab1 
```

### 实验总结
感觉自己总是想太多了，可能是算法题刷多了，连基本功能都还没实现，就老想着在时间和空间上优化，浪费了很多时间。

虽然在CSAPP的BombLab中用过GDB，但到现在早就忘光光了，只记得命令行调试很痛苦，于是花一天配置好了VS Code远程调试，舒服多了。

整体来说难度不是很大，但边界条件真的很折磨人，80%的时间都是在面向测试用例编程了。

写完以后成就感还是很大的。