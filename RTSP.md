# RTSP/RTP/RTCP/SDP

![image-20231106171531566](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106171531566.png)

![image-20231106173244320](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106173244320.png)

# RTSP协议：

![image-20231107163507875](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107163507875.png)

![image-20231107164148741](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107164148741.png)

![image-20231107164304360](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107164304360.png)

![image-20231107165405118](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107165405118.png)

![image-20231107165637013](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107165637013.png)

![image-20231107165710673](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107165710673.png)

 ![image-20231107165822207](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107165822207.png)

![image-20231107170022157](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107170022157.png)

UDP传输注意音视频端口不同，在setup中建立两次传输。

![image-20231107170957914](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107170957914.png)

# SDP协议：

![image-20231107185236745](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107185236745.png)

![image-20231107185607117](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107185607117.png)

![image-20231107185653445](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107185653445.png)

![image-20231107190351259](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107190351259.png)

![image-20231107190923010](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107190923010.png)

# RTCP协议：

![image-20231107205227782](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107205227782.png)

![image-20231107205140547](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107205140547.png)













# RTP协议：

![image-20231106180436109](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106180436109.png)

![image-20231106180446108](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106180446108.png)

---------------

![image-20231106180708906](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106180708906.png)

![image-20231106181527306](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106181527306.png)

--------

----------

# RTP封包H264：

![image-20231106182127842](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106182127842.png)

![image-20231106182411833](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106182411833.png)

![image-20231106182716594](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106182716594.png)

 ![image-20231106183324949](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106183324949.png)

### 打包方式：

Single NAL Unit：

![image-20231106191617184](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106191617184.png)

FU-A：

![image-20231106193114256](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106193114256.png)

F和NRI就是NALU头的前⾯三个bit位，后⾯的TYPE就是NALU的FU-A类型28

![image-20231106193146297](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106193146297.png)

![image-20231106193205080](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106193205080.png)

![image-20231106203857636](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231106203857636.png)

-----------

--------------

### 代码实现：

![image-20231107094701274](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231107094701274.png)































