# 蓝桥杯嵌入式stm32HAL库-MCP4017

## 板子上的硬件

MCP4017也是连接着板子上PB6、PB7的IIC设备，IIC在24c02中有，不再介绍，不过MCP4017中给出的时序图比24c02更直观些，因此下面代码下也给出了对应的时序图。

![image-20230119230344287](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192304645.png)

从原理图中可以看出，MCP4017有两个输出口，一个是GND的B，另一个就是可变阻值的W。而且W与PB14相连，再经过R17(10k)连到VDD(3.3V)。可以通过分压公式换算PB14上的电压。

![MCP4017的说明](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192309404.png)

## 对应驱动代码

### 设备地址及寄存器

因为是IIC器件，找到其设备地址和寄存器说明如下：

![image-20230119231013339](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192310547.png)

因此其读写的设备地址可以宏定义为：

>`MCP4017.c` 文件，宏定义其读写地址
>
>```c
>#define MCP_WRITE_ADDR 0x5E
>#define MCP_READ_ADDR 0x5F
>```

寄存器只有一个7位的寄存器。因此我们也不需要发寄存器地址，直接发送设备地址后读写的数据便是MCP4017的阻值分配。

![image-20230119231304028](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192313171.png)

### 读写函数(设置、读出阻值)

>`MCP4017.c` 文件,设置阻值函数(写函数)，需要注意的是只能设置0-127之间的整数，分配到0-100k欧之间。
>
>```c
>/**
>* @brief : void MCP4017_write(uint8_t res_val)
>* @param : 4017可调电阻写函数，可以设置其阻值
>* @attention : res_val阻值应在0-127之间，超过该值时只有低7位有效
>* @author : Sheep
>* @date : 23/01/19
>*/
>void MCP4017_write(uint8_t res_val)
>{
>    I2CStart();
>    I2CSendByte(MCP_WRITE_ADDR);
>    I2CWaitAck();
>
>    I2CSendByte(res_val);
>    I2CWaitAck();
>    I2CStop();
>}
>```
>
>对应时序图：(MCP4017比24c02的时序图更好理解)
>
>![image-20230119231613209](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192318408.png)

>`MCP4017.c` 文件,读出阻值的分配大小函数(读函数)
>
>同样的读出的数值是0-127之间的整数，对应的阻值分配到0-100k欧之间。
>
>```c
>/**
>* @brief : uint8_t MCP4017_read(void)
>* @param : 读4017阻值函数，读出的值在0-127之间，
>            4017的总阻值为100k欧，分成的每一份阻值为787.402欧，
>            使用头文件中的宏 MCP4017_CAL_RES(x)可以计算其阻值
>            使用头文件中的宏 MCP4017_CAL_PB14_VOL(x)可以计算其经过原理图分压后PB14上的电压
>* @attention : None
>* @author : Sheep
>* @date : 23/01/19
>*/
>uint8_t MCP4017_read(void)
>{
>    uint8_t res = 0;
>    I2CStart();
>    I2CSendByte(MCP_READ_ADDR);
>    I2CWaitAck();
>    res = I2CReceiveByte();
>    // I2CSendNotAck(); //不应答
>    I2CWaitAck(); //延时也可以
>    I2CStop();
>    return res;
>}
>```
>
>对应的时序图：
>
>![image-20230119231943606](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192319862.png)

### 宏定义计算阻值和分压值

MCP4017中的阻值是将100k的阻值分为127份(每一份为$R_{step} = 787.402 \Omega$ )，读取和写入的数值便是多少份 $R_{step}$ 

所以计算式和计算的宏如下：

>`MCP4017.h` 文件，计算MCP4017阻值和PB14上分压值的两个值
>
>```c
>#ifndef __MCP4017_H__
>#define __MCP4017_H__
>
>#include "i2c.h"
>
>#define Rstep 787.402
>
>void MCP4017_write(uint8_t res_val);
>uint8_t MCP4017_read(void);
>
>//计算MCP4017的阻值
>#define MCP4017_CAL_RES(x) (Rstep*x/127)
>//计算在MCP4017上分压后在PB14上的电压值
>// \frac{(x/127*100k)}{(x/127*100k+10k)} = V/3.3 ==> V = x/(x+12.7)*3.3
>#define MCP4017_CAL_PB14_VOL(x) (x/(x+12.7)*3.3)
>
>#endif
>```
>
>计算阻值的宏：
>$$
>\begin{matrix}
> R = \frac{MCP4017读出值}{127}\times 100k 
>\\ = {MCP4017读出值}\times R_{step} 
>\\ ={MCP4017读出值}\times {787.402}
>\end{matrix}
>$$
>由于在PB14分压的电阻为10k,所以PB14上的分压值如下式计算：
>$$
>\begin{matrix}
> \frac{\frac{MCP4017读出值}{127}\times 100k }{\frac{MCP4017读出值}{127}\times 100k +10k} = \frac{V_{PB14}}{3.3}  
> \\ \Downarrow 
> \\ \frac{{MCP4017读出值}}{{MCP4017读出值}+12.7} = \frac{V_{PB14}}{3.3}
> \\ \Downarrow 
> \\ V_{PB14} = \frac{{MCP4017读出值}}{{MCP4017读出值}+12.7} \times 3.3
>\end{matrix}
>$$
>手册中公式的说明
>
>![image-20230119234228608](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301192342877.png)

## 蓝桥杯使用建议

实现**读写两个函数和两个计算转换宏**即可，需要注意`IICInit()` 初始化和at24c02的初始化函数是一致的，不需要重复调用。同时编写函数时**注意官方给的IIC驱动函数命名由 `IIC+驼峰命名法`** 组成。

同时可以使用ADC1_IN5通道读取PB14上的电压值( 电压值的区间为 $[0,\frac{10}{11}\times 3.3] \Rightarrow[0,3]$ 实测最大只能到2.9V)。