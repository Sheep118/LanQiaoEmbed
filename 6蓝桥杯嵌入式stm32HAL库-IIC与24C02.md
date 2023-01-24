# 蓝桥杯嵌入式stm32HAL库-IIC与24C02

## 板子上的硬件

### IIC的驱动

- 官方给了IIC的驱动文件，分析下几个函数就好，其他的根据时序图看看就可以

  >`i2c.c`文件
  >
  >```c
  >/**
  >  * @brief I2C的短暂延时
  >  * @param None
  >  * @remark 80MHz,(以一条指令6个时间周期arm流水线)宏定义的延时是20次，
  >  *         因此，是20*6*1/80M=3/2M ≈1.5us,满足t_low(min) = 1.2us,t_high(min) = 0.6us的要求
  >  * @retval None
  >  */
  >static void delay1(unsigned int n)
  >{
  >    uint32_t i;
  >    for ( i = 0; i < n; ++i);
  >}
  >```
  >
  >```c
  >/**
  >  * @brief I2C等待确认信号
  >  * @param None
  >  * @attention 超时时间设置的是5个延时周期，不会一直死等
  >  * @retval None
  >  */
  >unsigned char I2CWaitAck(void)
  >{
  >    unsigned short cErrTime = 5;//超时时间是5次延时
  >    SDA_Input_Mode();
  >    delay1(DELAY_TIME);
  >    SCL_Output(1);
  >    delay1(DELAY_TIME);
  >    while(SDA_Input())
  >    {
  >        cErrTime--;
  >        delay1(DELAY_TIME);
  >        if (0 == cErrTime)
  >        {
  >            SDA_Output_Mode();
  >            I2CStop();
  >            return ERROR;
  >        }
  >    }
  >    SDA_Output_Mode();
  >    SCL_Output(0);
  >    delay1(DELAY_TIME);
  >    return SUCCESS;
  >}
  >```

  关于at24c02需要的延时时间要求，可以参考其手册：

  ![image-20230119172903511](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191729526.png)

- 同时，从上图可以看出IIC时序，需要注意的点：

  - 原则：高电平表示释放，即SCL高电平期间会释放SDA的所有权。所以，**不论发送还是接收，SCL高电平期间，需要保持SDA的数据不变化，变化的话就会被识别为开始或结束信号。**

  - 发送时，需要将数据先放上SDA，释放SLC让24c02接收，再拉低开始下一次，

    ```c
    /**
      * @brief I2C发送一个字节
      * @param cSendByte 需要发送的字节
      * @retval None
      */
    void I2CSendByte(unsigned char cSendByte)
    {
        unsigned char  i = 8;
        while (i--)
        {
            SCL_Output(0); //先拉低
            delay1(DELAY_TIME);
            SDA_Output(cSendByte & 0x80);
            delay1(DELAY_TIME);
            cSendByte += cSendByte; //乘2,相当于 << 1 ，把新数据推向最高位
            delay1(DELAY_TIME);
            SCL_Output(1); //放数据后再拉高
            delay1(DELAY_TIME); //高电平需要维持时间
        }
        SCL_Output(0);
        delay1(DELAY_TIME);
    }
    ```

  - 接收时，先拉高SCL，示意24c02需要将数据放上，在高电平期间(数据不会变化)读取SDA，再拉低SCL重新开始。

    ```c
    /**
      * @brief I2C接收一个字节
      * @param None
      * @retval 接收到的字节
      */
    unsigned char I2CReceiveByte(void)
    {
        unsigned char i = 8;
        unsigned char cR_Byte = 0;
        SDA_Input_Mode();
        while (i--)
        {
            cR_Byte += cR_Byte;
            SCL_Output(0);  //先拉低
            delay1(DELAY_TIME);
            delay1(DELAY_TIME);
            SCL_Output(1);  //再拉高，保证有上升沿这个动作
            delay1(DELAY_TIME); //延时等数据稳定
            cR_Byte |=  SDA_Input(); //高电平期间接收数据
        }
        SCL_Output(0);
        delay1(DELAY_TIME);
        SDA_Output_Mode();
        return cR_Byte;
    }
    ```

### AT24c02

实际上，不需要我们写IIC的驱动，因此重点放在at24c02的时序上。

上述时序图给出一个参数，是我们需要注意的一点，便是at24c02一次写周期为5ms(比较长)，也就是**==我们两次写操作的时间间隔至少为5ms==** ,下图是手册中的截图：

![image-20230119174350350](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191743598.png)

## 需要编写的驱动和对应的时序图

一共需要两个函数(实际上有5个，连续读写的函数可以不用，但**最好实现连续写的函数，因为读函数的两次之间需要5ms间隔**)

调用官方给的驱动时需要注意：所有函数都是**大写I2C开头 + 驼峰命名法**

注意：==看时序图时，上半部分是单片机端的信号，下半部分是at24c02端的信号==

​		因此当24c02端给出一个ACK时，单片机就需要等待Ack `I2CWaitAck()`

### AT24c02的器件读写地址

从手册截图和原理可以看出其IIC读写地址，截图如下：

![image-20230119181929876](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191819147.png)

![image-20230119182052966](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191820042.png)

>`at24c02.c` 文件，宏定义读写地址，方便阅读和自己查错。
>
>```c
>#define WRITE_ADDER 0xA0 //器件写地址
>#define READ_ADDER 0xA1 //器件读地址
>```
>
>宏定义写在.c文件中因为外部不会用到，且可以防止与IIC可变电阻读写地址重复宏定义报错。

### 写入单字节函数

>`at24c02.c` 文件，写入单字节函数
>
>```c
>/**
>* @brief : void at24c02_write_byte(uint8_t address,uint8_t data)
>* @param : uint8_t address, 要写入的数据地址
>           uint8_t data要写入的一字节数据
>* @attention : None
>* @author : Sheep
>* @date : 23/01/19
>*/
>void at24c02_write_byte(uint8_t address,uint8_t data)
>{
>   I2CStart();
>   I2CSendByte(WRITE_ADDER);
>   I2CWaitAck();
>
>   I2CSendByte(address);
>   I2CWaitAck();
>
>   I2CSendByte(data);
>   I2CWaitAck();
>
>   I2CStop();
>}
>```
>
>对应时序图：
>
>![image-20230119175527319](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191755329.png)

### **读取单字节函数**

>`at24c02.c` 文件,直接读并不常用，这里的单字节读函数对应的时序图是手册中的随机读函数。
>
>```c
>/**
>* @brief : uint8_t at24c02_read_byte(uint8_t address)
>* @param : uint8_t address 要读取一字节的地址
>* @attention : 最后要不要应答均可，官方给的实例中是用WaitAck进行延时
>* @author : Sheep
>* @date : 23/01/19
>*/
>uint8_t at24c02_read_byte(uint8_t address)
>{
>   uint8_t ret = 0;
>   //假写地址
>   I2CStart();
>   I2CSendByte(WRITE_ADDER);
>   I2CWaitAck();
>
>   I2CSendByte(address);
>   I2CWaitAck();
>
>   //开始读
>   I2CStart();
>   I2CSendByte(READ_ADDER);
>   I2CWaitAck();
>   ret = I2CReceiveByte();
>   // I2CWaitAck(); //等待中默认超时时间是五次等待，相当于延时
>   // I2CSendAck(); //不应答也行
>   I2CStop();
>
>   return ret;
>}
>```
>
>对应的随机读时序：
>
>![image-20230119180738872](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191807679.png)

### **写入多字节函数**

- 需要注意的是at24c02**一次最多只能写一页，也就是8个字节**，**连续写时多写入的字节会从==这一页开始的位置==重新开始覆盖**。 **以8个字节地址为一页(0-7,8-15....为一页)**

  手册中说明的截图如下：

  ![image-20230119181205513](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191812738.png)

>`at24c02.c` 文件，多字节写入函数，对应是手册中的页写时序，注意每次最多只能写一页。
>
>```c
>/**
>* @brief : void at24c02_write_bytes(uint8_t start_addr,uint8_t * data_bytes,uint8_t data_num)
>* @param : uint8_t start_addr,要写入的首地址
>           uint8_t * data_bytes,数据数组的首地址
>           uint8_t data_num，要写入数据的数量
>* @attention : 当写入的数量超过8个时会从这一页重新开始覆盖写入(0-7,8-15这样8个字节为一页)
>* @author : Sheep
>* @date : 23/01/19
>*/
>void at24c02_write_bytes(uint8_t start_addr,uint8_t * data_bytes,uint8_t data_num)
>{
>    uint8_t i = 0;
>    I2CStart();
>    I2CSendByte(WRITE_ADDER);
>    I2CWaitAck();
>    I2CSendByte(start_addr);
>    I2CWaitAck();
>    for(i = 0;i<data_num;i++)
>    {
>        I2CSendByte(data_bytes[i]);
>        I2CWaitAck();
>    }
>    I2CStop();
>}
>
>```
>
>对应的页写时序：
>
>![image-20230119181530007](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191815510.png)

### 读取多字节函数

需要注意的时与写入多字节只能写如一页不同，**读取多字节函数可以从起始地址一直自增，直到达到整块内存的限制(2K)，才会重新从头(0地址)覆盖读**。 手册的说明如下：

![image-20230119182340257](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191823237.png)

>`at24c02.c` 文件，读取多字节函数，手册中的时序图有一部分省略，因此，完整的时序是随机读+连续读的拼接,注意这个函数中第二个参数是传指针进行取值输出。
>
>```c
>/**
>* @brief : void at24c02_read_bytes(uint8_t start_addr,uint8_t * data_bytes,uint8_t data_num)
>* @param : uint8_t start_addr,要读取数据的首地址
>            uint8_t * data_bytes,[out]数据缓存的首地址
>            uint8_t data_num，要读取的数量
>* @attention : None
>* @author : Sheep
>* @date : 23/01/19
>*/
>void at24c02_read_bytes(uint8_t start_addr,uint8_t * data_bytes,uint8_t data_num)
>{
>    uint8_t i = 0;
>    I2CStart();
>    I2CSendByte(WRITE_ADDER);
>    I2CWaitAck();
>    I2CSendByte(start_addr);
>    I2CWaitAck();
>
>    I2CStart();
>    I2CSendByte(READ_ADDER);
>    I2CWaitAck();
>    for(i = 0;i<data_num;i++)
>    {
>        data_bytes[i] = I2CReceiveByte();
>        if(i != data_num -1)
>            I2CSendAck(); //最后一次不发应答
>    }
>    I2CStop();
>}
>
>```
>
>对应的时序在手册中的中一部分：
>
>![image-20230119182648854](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191826118.png)

### 对应的头文件

>注意头文件中需要实现复用`IICInit()` 函数即可，或者直接调用也可以。
>
>```c
>#ifndef __AT24C02_H__
>#define __AT24C02_H__
>
>#include "stm32g4xx_hal.h"
>#include "stdint.h"
>#include "i2c.h"
>
>__STATIC_INLINE void at24c02_init(void) //内联函数，编译时展开
>{
>    I2CInit();
>}
>
>void at24c02_write_byte(uint8_t address,uint8_t data);
>uint8_t at24c02_read_byte(uint8_t address);
>void at24c02_write_bytes(uint8_t start_addr,uint8_t * data_bytes,uint8_t data_num);
>void at24c02_read_bytes(uint8_t start_addr,uint8_t * data_bytes,uint8_t data_num);
>
>#endif
>```

## 蓝桥杯使用建议

蓝桥杯中实现两个函数即可，**连续写入多字节函数**、**读取一个字节的函数** 

因为写入单字节可以使用写入多字节代替，但读取单字节可以直接将结果返回，不需要像读取多字节一样使用外部定义的内存的指针进行读取，而且读取周期并没有像写入有要求5ms时间间隔，因此可以使用单字节读取函数进行循环即可。

**==需要注意的是，多字节写入时只能在一页内进行写入，注意写入首地址和写入数量的设置==。==还有多次写入c操作之间要求时间间隔至少5ms==**





