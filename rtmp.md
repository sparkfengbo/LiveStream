rtmp

##1.概要
RTMP(Real-Time Messaging Protocol)刚开始是Macromedia针对FlashPlayer和服务器之间传递的网络音视频数据流开发的专利协议，Macromedia现在被Adobe收购，现在已经面向公众发布不完整的版本。


RTMP协议有多种变种：

 - RTMP    默认的基于TCP在端口1935运行的协议
 - RTMPS   TLS/SSL上的RTMP
 - RTMPE   加密的RTMP
 - RTMPT   封装在HTTP请求的RTMP
 - RTMFP   不适用TCP而使用UDP的RTMP

##2.握手
 RTMP是应用层协议，传输数据之前需要建立连接，这个过程是握手。
 ![](/Users/leegend/Desktop/tmppic/图片 1.png)
 
- 握手开始于客户端发送C0，C1块。
 
 具体握手过程的数据格式请参考RTMP文档。
 
##3.RTMP数据数据
###3.1总体
服务端和客户端通过在网络上发送RTMP消息（Message）进行通讯。消息可能包含音频，视频，数据，或其他的消息。


握手之后，连接开始复用一个或多个块流,将Message拆分成Chunk发送或组合。好处有两个：1.防止网络传输消息阻塞 2.通过分块可以减少冗余信息减少数据量

每个块流承载来自一个消息流的一类消息。每个被创建的块都关联到一个唯一的块流ID。所有的块都通过网络传输。在传输过程中，必须一个块发送完之后再发送下一个块。在接收端，每个块都根据块ID被收集成消息。

       


###3.2Message
消息头包含下面的内容：

 ![](/Users/leegend/Desktop/tmppic/图片2.png)

共11字节。

**消息负载**是消息中包含的真实数据。例如，它可以是音频样本或压缩的视频数据。

RTMP保留消息类型ID 在**1-7**之内的消息为**协议控制消息**。这些消息包含RTMP块流协议和RTMP协议本身要使用的信息。ID为1和2用于RTMP块流协议。ID在3-6之内用于RTMP本身。ID 7的消息用于边缘服务与源服务器。协议控制消息必须有**消息流ID 0**和**块流ID 2**，并且有最高的发送优先级。每个协议控制消息都有固定大小的负载。

- 1：设置块大小
- 2：取消消息，通知正在等待接收的对等端，丢弃一个块流中已经接收的部分并且取消对该消息的处理
- 3：确认（ACK）
- 4：用户控制消息 check
- 6：设置对等端带宽

- 8：音频消息
- 9：视频消息
- 22:聚合消息，是含有一个消息列表的一种消息。


###3.3Chunk
####3.3.1Chunk Format

 ![](/Users/leegend/Desktop/tmppic/图片3.png)

- 块基本头：1到3 字节
   块基本头包含**块流ID（Chunk Stream ID）**和**块类型（Chunk Type）**（在下图中用fmt表示）。
   
   **Chunk Type**决定**Chunk Msg Header**的

   **长度：**块基本头字段可能是1，2或3个字节。这取决于**块流ID（Chunk Stream ID）**。
ID 0、1、2作为保留。

- 0，代表Basic Header占用2字节，表示ID的范围是64-319（第二个字节+64）；
- 1，代表Basic Header占用3字节，表示ID范围是64-65599（第三个字节*256+第二个字节+64）；
- 2，表示低层协议消息。没有其他的字节来表示流ID。

**Chunk Type**长度固定为2位，因此CSID的长度是（6=8-2）、（14=16-2）、（22=24-2）中的一个。


- 3-63表示完整的流ID。3-63之间的值表示完整的流ID。没有其他的字节表示流ID。
 
##### **1字节**
 ![](/Users/leegend/Desktop/tmppic/图片4.png)
 当Basic Header为1个字节时，CSID占6位，6位最多可以表示64个数，因此这种情况下CSID在［0，63］之间，其中用户可自定义的范围为［3，63］。
 
 
#####  **2字节**
 ![](/Users/leegend/Desktop/tmppic/图片5.png)
 当Basic Header为2个字节时，CSID占14位，此时协议将与chunk type所在字节的其他位都置为0，剩下的一个字节来表示CSID－64，这样共有8个二进制位来存储CSID，8位可以表示［0，255］共256个数，因此这种情况下CSID在［64，319］，其中319=255+64。
#####  **3字节**
![](/Users/leegend/Desktop/tmppic/图片6.png)
当Basic Header为3个字节时，CSID占22位，此时协议将［2，8］字节置为1，余下的16个字节表示CSID－64，这样共有16个位来存储CSID，16位可以表示［0，65535］共65536个数，因此这种情况下CSID在［64，65599］，其中65599=65535+64，需要注意的是，Basic Header是采用小端存储的方式，越往f后的字节数量级越高，因此通过这3个字节每一位的值来计算CSID时，应该是:<第三个字节的值>x256+<第二个字节的值>+64

####3.3.3 Chunk Msg Header

包含了要发送的实际信息（可能是完整的，也可能是一部分）的描述信息。

Chunk Message Header的格式和长度取决于Basic Header的chunk type，共有4种不同的格式，由上面所提到的Basic Header中的fmt字段控制。

其中第一种格式可以表示其他三种表示的所有数据，但由于其他三种格式是基于对之前chunk的差量化的表示，因此可以更简洁地表示相同的数据，实际使用的时候还是应该采用尽量少的字节表示相同意义的数据。以下按照字节数从多到少的顺序分别介绍这4种格式的Message Header。

#####  **类型0**
![](/Users/leegend/Desktop/tmppic/图片7.png)
0类型的块长度为**11**字节。在一个块流的开始和时间戳返回（即值与上一个chunk相比减小，通常在回退播放的时候会出现这种情况）的时候必须有这种块。

- 1.时间戳：3字节 
- 2.message length（消息数据的长度）：占用3个字节，表示实际发送的消息的数据如音频帧、视频帧等数据的长度，单位是字节。注意这里是Message的长度，也就是chunk属于的Message的总数据长度，而不是chunk本身Data的数据的长度。
- 3.message type id(消息的类型id)：占用1个字节，表示实际发送的数据的类型，如8代表音频数据、9代表视频数据。
- 4.msg stream id（消息的流id）：占用4个字节，表示该chunk所在的流的ID，和Basic Header的CSID一样，它采用小端存储的方式，

#####  **类型1**
![](/Users/leegend/Desktop/tmppic/图片8.png)

类型1的块占**7**个字节长，省去了表示msg stream id的4个字节，**消息流 ID不包含在本块中**。表示此chunk和上一次发的chunk所在的流相同。具有可变大小消息的流，在第一个消息之后的每个消息的第一个块应该使用这个格式。

- timestamp delta：占用3个字节，注意这里和type＝0时不同，存储的是和上一个chunk的时间差。类似上面提到的timestamp，当它的值超过3个字节所能表示的最大值时，三个字节都置为1，实际的时间戳差值就会转存到Extended Timestamp字段中，接受端在判断timestamp delta字段24个位都为1时就会去Extended timestamp中解析时机的与上次时间戳的差值。


#####  **类型2**
![](/Users/leegend/Desktop/tmppic/图片9.png)

类型2的块占**3**个字节，相对于type＝1格式又省去了表示消息长度的3个字节和表示消息类型的1个字节。既不包含**流ID**也不包含**消息长度**。本块使用的流ID和消息长度与先前的块相同。具有固定大小消息的流，在第一个消息之后的每个消息的第一个块应该使用这个格式。

#####  **类型3**
类型3的块没有头。流ID，消息长度，时间戳都不出现。
这种类型的块使用与先前块相同的数据。当一个消息被分成多个块，除了第一块以外，所有的块都应使用这种类型。


####3.3.4 Extended Time Stamp
![](/Users/leegend/Desktop/tmppic/图片10.png)
只有当块消息头中的普通时间戳设置为0x00ffffff时，本字段才被传送。如果普通时间戳的值小于0x00ffffff，那么本字段一定不能出现。如果时间戳字段不出现本字段也一定不能出现。类型3的块一定不能含有本字段。

扩展时间戳占4个字节。能表示的最大数值就是0xFFFFFFFF＝4294967295。当扩展时间戳启用时，timestamp字段或者timestamp delta要全置为1，表示应该去扩展时间戳字段来提取真正的时间戳或者时间戳差。注意扩展时间戳存储的是完整值，而不是减去时间戳或者时间戳差的值。

###4 示例
参考[带你吃透RTMP](http://mingyangshang.github.io/2016/03/06/RTMP%E5%8D%8F%E8%AE%AE/)或[rtmp 协议规范 中文版](http://download.csdn.net/download/leixiaohua1020/6563059)中的示例


--------

 
参考

- [维基百科](https://en.wikipedia.org/wiki/Real-Time_Messaging_Protocol#Packet_structure)
- [带你吃透RTMP](http://mingyangshang.github.io/2016/03/06/RTMP%E5%8D%8F%E8%AE%AE/)
- [rtmp 协议规范 中文版](http://download.csdn.net/download/leixiaohua1020/6563059)
- [ [总结]RTMP流媒体技术零基础学习方法](http://blog.csdn.net/leixiaohua1020/article/details/15814587)
- [RTMP规范简单分析](http://blog.csdn.net/leixiaohua1020/article/details/11694129)
- [RTMP多媒体视频服务器（RTMP系列三）](http://www.jianshu.com/p/a77156c60868#comment-14811838)