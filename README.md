# STM32_Project
关于stm32的工程文件
## OLED显示屏编程

### I2C通信协议

- #### 简介：串行通信协议，一根时钟线SCL，一根数据线SDA；IIC将==SCL处于高时SDA拉低==的动作作为开始信号，==SCL处于高时SDA拉高==的动作作为结束信号；

  ```c++
  void OLED_I2C_Start(void)		//开始
  {
  	OLED_W_SDA(1);
  	OLED_W_SCL(1);
  	OLED_W_SDA(0);
  	OLED_W_SCL(0);
  }
  
  void OLED_I2C_Stop(void)		//结束
  {
  	OLED_W_SDA(0);
  	OLED_W_SCL(1);
  	OLED_W_SDA(1);
  }
  ```

  

- #### 传输数据时，SDA在SCL低电平时改变数据，在SCL高电平时读取数据，每个SCL脉冲的高电平传递1位数据。

- #### 字节格式：

  ![image-20240220102950762](https://image-jerry-1324385193.cos.ap-chongqing.myqcloud.com/Typora/image-20240220102950762.png?imageSlim)

- #### 应答信号：

  > 发送器在应答时钟周期内释放对SDA总线的控制，这样接收器可以通过将==SDA线拉低==告知发送器：数据已被成功接收。

  应答信号分为两种：

      1）当第9位(应答位)为 低电平 时，为 ACK  （Acknowledge）   信号

      2）当第9位(应答位)为 高电平 时，为 NACK（Not Acknowledge）信号

    主机发送数据，从机接收时，ACK信号由从机发出。当在SCL第9位时钟高电平信号期间，如果SDA仍然保持高电平，则主机可以直接产生STOP条件终止以后的传输或者继续ReSTART开始一个新的传输

    从机发送数据，主机读取数据时，ACK信号由主机给出。主机响应ACK表示还需要再接收数据，而当主机接收完想要的数据后，通过发送NACK告诉从机读取数据结束、释放总线。随后主机发送STOP命令，将总线释放，结束读操作。

- #### 数据传输：

  ​	主机对从机发送数据时，主机对从机发送一个开始字节，然后即可一直发送数据。

  ​	主机对向从机读取数据时，方式同发送数据有所不同，要多一次通信过程。
      主机需要先向从机发送一次信号，告诉从机”我要读取数据“，然后==重开一次通信==，等待从机主动返回数据。





### 代码讲解

1. 引脚配置

   ```c++
   #define OLED_W_SCL(x)		GPIO_WriteBit(GPIOB, GPIO_Pin_8, (BitAction)(x))
   #define OLED_W_SDA(x)		GPIO_WriteBit(GPIOB, GPIO_Pin_9, (BitAction)(x))
   ```

   > **注：**这是一种新的定义方式

2. 写数据

   ```c++
   void OLED_I2C_SendByte(uint8_t Byte)
   {
   	uint8_t i;
   	for (i = 0; i < 8; i++)
   	{
   		OLED_W_SDA(Byte & (0x80 >> i));		
   		OLED_W_SCL(1);		//通过添加一个时钟周期，确保数据逐个发送，同时从机在SCL为高时读取数据
   		OLED_W_SCL(0);
   	}
   	OLED_W_SCL(1);	//额外的一个时钟，不处理应答信号，确保在数据传输的最后一个时钟周期后，总线能够正常						释放，从而使得总线处于一个稳定状态。
   	OLED_W_SCL(0);
   }
   ```

   > 注：(0x80 >> i)表示按位向右移动，即：0x10000000——》0x01000000···
   >
   > Byte & (0x80 >> i)表示按位与（==都为1才是1，否则为0==），for循环即检查Byte的每一位，确保数据正确
