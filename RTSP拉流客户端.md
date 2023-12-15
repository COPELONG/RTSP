# RTSP拉流客户端



![image-20231210163300514](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231210163300514.png)

拉流客户端处理服务端发来请求，并且向服务端发送报文数据。

当有多路流存在时，SetUp需要建立多次请求，首先需要解析describe后reply的请求，里面包含SDP媒体信息

通过解析SDP信息，获取每一路流的具体信息并且建立连接进行play

# 解析SDP：

![image-20231212115648575](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212115648575.png)

首先解析服务器回复的RTSP部分，然后根据服务器回复的状态码和CSeq去发送报文或者解析SDP。

![image-20231212115925883](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212115925883.png)

一行一行解析：解析关键行并获取对应数据

![image-20231212114640246](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212114640246.png)

![image-20231212113151132](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212113151132.png)

![image-20231212113609816](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212113609816.png)

![image-20231212113708268](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212113708268.png)

然后解析里面的关键数据赋值定义好的SDP结构体中。

![image-20231212115106333](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231212115106333.png)

存储每一种流的SDPTrack数据

-----------

-------------



![image-20231212122804037](D:/typora-image/image-20231212122804037.png)

![image-20231212122818146](D:/typora-image/image-20231212122818146.png)

TCP和UDP传输时，SETUP发送到报文数据有区别！

---------

TCP传输RTP包：

每个RTP包前面都有4个字节：￥，channel，len：2. 所以要解析出这4个字节在进行解析RTP包





