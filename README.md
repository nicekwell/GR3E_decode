# GR3E_decode
GR3E三通道接收机解调，PPM转换成串口数字信号。

# 功能
GR3E是富斯（flysky）三通道航模遥控器的接收机，其三个通道输出PPM信号。  
本工程使用单片机提取其三个通道的信号，解码后以数字信号通过UART串口输出。

# GR3E特性
关于GT2B遥控器和GR3E接收机的详细介绍可访问 <http://nicekwell.net/>。  
关于PPM信号的详细介绍可访问 <http://nicekwell.net/>。

三通道：

通道 | 功能 | 备注
:-: | :-: | :-:
CH1 | 方向 | 连续变化，**精度20us**
CH2 | 油门 | 连续变化，**精度20us**
CH3 | 按钮 | 在1000us和2000us间切换

由于输出精度是20us，所以单片机在采集信号时精度达到20us即可。

**所有通道周期大约是16ms，刷新率60Hz。**

每个通道的高电平脉宽范围大约在1000us~2000us之间。  
在一个周期内，各个通道的高电平脉冲一个紧接着一个依次输出，三个通道信号全部输出结束最多约占用2ms*3=6ms，周期内剩余约10ms的时间所有通道输出全为低电平。

# 硬件
【单片机】STC12C5A60S2  
【晶振】24MHz  
注：此晶振可产生精确地定时器中断，方便监测各个通道，但串口波特率会有0.16%的误差，不会影响使用。  
【引脚连接】  
CH1：P1.2  
CH2：P1.1  
CH3：P1.0  
TXD：P3.1

# 输出格式
【波特率】115200  
【数据格式】  
每个周期内，当采集完三个通道的高电平后（最长约6ms）会立刻通过串口发送3个通道的数据信息。  
每个周期的数据为一帧，一帧数据有4个字节：  
第一字节固定为0x01，标志一帧数据开始。（后面三个字节不可能为这个值）  
后面三个字节一次表示CH1、CH2、CH3的脉宽，单位是10us。如输出150表示脉宽为1500us。  
注：  
1、接收机输出的脉宽范围大约在1000us~2000us之间，所以三个脉宽的数据范围大约在100~200之间。  
2、解码后输出的数据单位是10us，但实际接收机输出的精度是20us，单片机程序也是按照20us的精度采样的。

# 程序结构介绍
两个进程：定时器中断和主循环。

定时器20us一次中断，有两个状态：  
1、信号采集中：  
&emsp;&emsp;1、采集各个通道高电平时间。  
&emsp;&emsp;2、判断当前所有通道是否采集完成（所有通道信号结束后，所有通道都会输出低电平。  
&emsp;&emsp;&emsp;&emsp;如果连续100us（5个周期）检测到所有通道都是低电平，则认为一帧信号结束，通知主进程发送数据，并进入状态2。  
2、本周期信号已结束，等待下一周期：  
&emsp;&emsp;任意通道采集到高电平则进入状态1。

主循环进程只干一件事，等待定时器进程发送指令，接收到指令后发送数据。  
但主循环会忽略第一帧数据，因为第一帧数据可能采集不完整。

