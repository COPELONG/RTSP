# RTSP服务器

RTSP是一个**实时传输流协议**，是一个应用层的协议。通常说的RTSP 包括RTSP协议、RTP协议、RTCP协议，对于这些协议的作用简单的理解如下

 

RTSP协议：负责服务器与客户端之间的请求与响应

RTP协议： 负责服务器与客户端之间传输媒体数据

RTCP协议：负责提供有关RTP传输质量的反馈，就是确保RTP传输的质量，一般来说不需要使用RTCP服务传输，进行反馈时需要使用。

三者的关系： rtsp并不会发送媒体数据，只是完成服务器和客户端之间的信令交互，rtp协议负责媒体数据传输，rtcp负责rtp数据包的监视和反馈。rtp和rtcp并没有规定传输层的类型，可以选择udp和tcp。Rtsp的传输层则要求是基于tcp。

![image-20230723152900273](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230723152900273.png)

## 流程：

![image-20231211105901934](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211105901934.png)

解析来自客户端的发送报文的每一行数据，提取重要数据。所有行都解析完再回复请求。

![image-20231211110018697](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211110018697.png)

服务器端：回复客户端，根据接受到的报文信息进行reply。

![image-20231211110119809](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211110119809.png)

当收到客户端Describe请求时，回复需要带上SDP信息。

![image-20231211110305520](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211110305520.png)![image-20231211110346976](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211110346976.png)

当收到SetUp请求时，需要解析请求中的客户端的RTP和RTCP端口。

![image-20231211110503989](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211110503989.png)

如果是UDP传输，则需要分别创建RTP和RTCP通道：

```c++
        serverRtpSockfd = createUdpSocket();
        serverRtcpSockfd = createUdpSocket();
        bindSocketAddr(serverRtpSockfd, "0.0.0.0", SERVER_RTP_PORT)
        bindSocketAddr(serverRtcpSockfd, "0.0.0.0", SERVER_RTCP_PORT) < 0)
```
服务器回复客户端则需要回复服务器对应的RTP和RTCP端口。

![image-20231211110606105](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211110606105.png)





## RTP包结构：

### RtpHeader：

![image-20230724221049291](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221049291.png)

![image-20230724221238914](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221238914.png)

![image-20231207110019571](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231207110019571.png)

### H264格式解析：

H.264由一个一个的NALU组成，每个NALU之间使用`00 00 00 01`-->0001或`00 00 01`-->001分隔开，每个NALU的第一次字节都有特殊的含义,

Nalu Type：

![image-20230724221404336](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221404336.png)

![image-20230724221417409](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221417409.png)

![image-20230724221503386](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221503386.png)

![image-20230724221547742](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221547742.png)

![image-20230724221555789](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230724221555789.png)

```c++
struct RtpPacket
{
    struct RtpHeader rtpHeader;
    uint8_t payload[0];
    //柔性数组在结构体中的使用需要小心管理内存空间，确保在使用完毕后及时释放相应的内存。这样的数据结构在编写网络通信相关的程序时非常常见，用于处理变长的数据负载。
};

```



```c++
FILE 文件结构中 , 存在一个指针 , 每次调用文件的读写函数 , 该指针就会移动 ;
如 fgets / fputs , getc / putc , fscanf / fprintf , fread / fwrite 等函数 ;
默认情况下 , 指针是从前向后移动的 ;
该文件内部的指针指向的位置可以通过 fseek 函数进行改变 ;

        rSize = fread(frame, 1, size, fp);//将读取的数据size大小存储在frame缓冲区中。
        frameSize = (nextStartCode - frame);//相对于起始位置偏移了多少字节：frame的字节包括startcode
        fseek(fp, frameSize - rSize, SEEK_CUR);//退回操作：回到并指向下一个startcode位置
```



```c++
int pos = 1;
memcpy(rtpPacket->payload+2, frame+pos, RTP_MAX_PKT_SIZE);
pos += RTP_MAX_PKT_SIZE;
//frame+pos ：frame是一帧数据，包含Nalu Type（一个字节）.因为是分片发送，所以Nalu Type通过人工设置好了，不需要系统自带的Nalu Type，所以首先+1字节，跳过Nalu Type， pos += RTP_MAX_PKT_SIZE，frame+pos则指向一个位置数据。
```

### AAC音频解析：

AAC音频格式有2种：

对AAC进行解码时：

ADIF(Audio Data Interchage Format)，音频数据交换格式：只有一个统一的头，必须得到所有数据后解码，适用于本地文件。

ADTS(Audio Data Transport Stream)，音视数据传输流：每一帧都有头信息，任意帧解码，适用于传输流。

adts的头部一共有15个字段，共7bytes，如果有校验位则会在尾部增加2bytesCRC校验。

![image-20230725101828005](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230725101828005.png)

![image-20231208121038875](D:/typora-image/image-20231208121038875.png)

![image-20231208122810536](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231208122810536.png)

![image-20230725112027325](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20230725112027325.png)

发送到AACData 不带ADTS头部

### **基于TCP协议传输和UDP协议传输的区别：**

  RTSP服务器不需要再对应创建RTP和RTCP的UDP连接通道，因为TCP版的RTP传输，客户端与服务器交互时，无论是RTSP信令还是RTP数据包或者是RTCP数据包，都是使用同一个tcp连接通道。只不过这个tcp连接通道在发送rtp数据包或者rtcp数据包时，需要在RTP头部前面加一些分隔字节。

使用TCP协议传输同时传输音视频数据需要做区分：标记符和通道说明

channel：两路：音视频RTP和RTCP通道0x00  0x01、  0x02   0x03

```c++
    char* tempBuf = (char *)malloc(4 + rtpSize);
    tempBuf[0] = 0x24;//$
    tempBuf[1] = channel;// 0x00;
    tempBuf[2] = (uint8_t)(((rtpSize) & 0xFF00) >> 8);
    tempBuf[3] = (uint8_t)((rtpSize) & 0xFF);// 3  4字节 表示长度
    memcpy(tempBuf + 4, (char*)rtpPacket, rtpSize);
```

![image-20231212122515887](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212122515887.png)

![image-20231211175109298](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231211175109298.png)

![image-20231212150115324](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212150115324.png)

当使用TCP协议承载RTSP/RTP时，所有的命令和媒体数据都将通过RTSP端口，通常是554，进行发送。同时，数据将经过二元交织格式化之后才能发送。

![image-20231213104538669](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231213104538669.png)



而基于UDP的实现：创建两个socket：RTP和RTCP。

```c++
serverRtpSockfd = createUdpSocket();
serverRtcpSockfd = createUdpSocket();
rtpSendAACFrame(serverRtpSockfd, clientIP, clientRtpPort,rtpPacket, frame, adtsHeader.aacFrameLength - 7);
```

在网络编程中，`send` 和 `sendto` 都是用于发送数据的函数，但它们在使用场景和功能上有一些区别。

1. `send` 函数：
   - `send` 函数是用于发送数据的通用函数，适用于面向连接的套接字（如 TCP 套接字）。
   - 在使用 `send` 函数发送数据时，需要先建立连接，通常通过 `connect` 函数建立连接，然后使用 `send` 发送数据。
   - `send` 函数的原型：`ssize_t send(int sockfd, const void *buf, size_t len, int flags);`
   - `sockfd` 是套接字描述符，`buf` 是指向要发送数据的缓冲区的指针，`len` 是要发送的数据大小，`flags` 是发送标志，一般设置为 0。
2. `sendto` 函数：
   - `sendto` 函数适用于无连接的套接字（如 UDP 套接字），不需要先建立连接，直接发送数据到指定的目标地址。
   - 在使用 `sendto` 函数发送数据时，需要指定目标地址信息，通常通过 `struct sockaddr` 结构体来表示目标地址信息。
   - `sendto` 函数的原型：`ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);`
   - `sockfd` 是套接字描述符，`buf` 是指向要发送数据的缓冲区的指针，`len` 是要发送的数据大小，`flags` 是发送标志，一般设置为 0，`dest_addr` 是指向目标地址信息的指针，`addrlen` 是目标地址结构体的大小。

总的来说，`send` 用于面向连接的套接字，需要先建立连接，而 `sendto` 适用于无连接的套接字，不需要先建立连接，直接发送数据到指定的目标地址。选择使用哪个函数取决于套接字的类型和应用场景的需求。













