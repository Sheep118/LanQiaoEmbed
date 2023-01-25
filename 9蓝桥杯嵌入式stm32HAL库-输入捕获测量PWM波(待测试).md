# 蓝桥杯嵌入式stm32HAL库-输入捕获测量PWM波

## 板子上的硬件

- 板子上有两个XL555信号发生器，并且与(板子上标着的"频率输出"的)两个可调电阻相连，可以调节其输出方波信号的频率，输出的频率连接着stm32的PA15和PB4脚。

  其中，PA15内部是 `TIM2CH1` 或者是 `TIM8CH1` ；PB4内部是 `TIM3CH1` 或者是 `TIM16CH1` 

  原理图如下：

  ![image-20230125141047940](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251412660.png)

  可以使用通用定期或者高级定时的输入捕获功能，捕获高低电平，并计算占空比和频率。

## 驱动代码的编写

这里使用`TIM8CH1` 来测量PA15上的信号，用 `TIM3CH1`来测量PB4上的信号 。

测量方案主要有两种。一种是**单捕获通道+中断切换触发边沿** 另一种是 **单输入、双捕获通道+硬件从模式复位 + DMA数据传输** 。

### 需求计算(待检测)

从网上查阅得知：两个信号发生器产生的信号频率在`720Hz - 22.4kHz` 和 `700Hz - 24.0kHz` 所以我们可以得知，信号最长时间为 $\frac{1}{700} \approx 1.42ms$ ，信号最大频率为$24.0kHz$ 由采样定律可知，我们至少采样评频率至少为 $48.0kHz$ 而计时时间至少得满足$1.42ms$。

所以，将**定时器的 `psc` 设为 79**，即CNT每 $\frac{1}{1M} = 1us$ 加一，也就是每1us检测一次边沿，也就是采样频率为1MHz,是足够的。同时，**将 `arr` 设置到最大，为 65535(16bit)** ,所以可以检测最长的时间为 $65535us = 65.635ms$ 也是满足的。

如果需要检测更长的时间，就需要结合定时器的中断溢出进行输入捕获。

### 单捕获通道+中断切换触发边沿

思路：

1. 首先设置捕获上升沿，在第一次捕获后切换为下降沿；捕获下降沿后又切换为上升沿。
2. 利用标志位切换进入捕获中断(或回调函数)后执行函数。

下面以 `TIM8CH1` 为例子。

#### CubeMX的设置

- 设置**内部时钟源、PSC 、ARR** 

  ![image-20230125165514628](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251655710.png)



- 注意：需要在**NVIC设置开启中断**

  ![image-20230125150326784](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251503987.png)

#### 代码编写

需要注意的是，HAL库的回调函数是按功能分的，也就是说，所有输入捕获都是同一个回调函数，因此，在**回调函数中首先需要检测传入句柄是哪个定时器哪个通道** ，而且HAL中断函数中默认帮我们清楚了标志位和关闭中断，所以我们**想要持续触发就需要在回调函数中重新开启中断** 。

同时，我们也**需要在进入 `while(1)` 前，开启第一次中断捕获**。

>`main.c` (其他文件也可以，因为这是一个HAL库弱定义的函数） 中断回调函数处理：
>
>```c
>uint8_t cap_sta = 0;
>uint16_t cap_val = 0;
>double cap_high_time[2] = {0}; //最好是用double
>double cap_all_time[2] = {0};//最好是用double
>double cap_duty[2] = {0};//最好是用double
>double cap_fre[2] = {0};//最好是用double
>
>void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
>{
>
>    if(htim->Instance == TIM8)
>    {
>        if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
>        {          
>          switch (cap_sta)
>          {
>          case 0: //上升沿时
>              cap_all_time[0] = __HAL_TIM_GET_COMPARE(htim,TIM_CHANNEL_1); //保留整个周期时间
>              if(cap_all_time !=0) //避免除0错误
>              {
>                cap_duty[0] = cap_high_time[0]/cap_all_time[0]; 
>                cap_fre[0] = 1000000.0/cap_all_time[0];
>              }
>              cap_sta = 0;cap_val = 0; //取出数据后便可以将数据清零
>              __HAL_TIM_SetCounter(htim,0); //上升沿开始时将所有数据清零
>              cap_sta = 1; //标记捕获上升沿
>              __HAL_TIM_SET_CAPTUREPOLARITY(htim,TIM_CHANNEL_1,TIM_INPUTCHANNELPOLARITY_FALLING); 
>              //更改为下降沿
>            break;
>          case 1:
>              cap_sta = 0; //标记捕获到下降沿
>              cap_val = __HAL_TIM_GET_COMPARE(htim,TIM_CHANNEL_1);
>              cap_high_time[0] = cap_val; //储存高电平
>              __HAL_TIM_SET_CAPTUREPOLARITY(htim,TIM_CHANNEL_1,TIM_INPUTCHANNELPOLARITY_RISING);
>              //更换为上升沿
>            break;          
>          default:
>            break;
>          }
>          // HAL_TIM_Base_Start_IT(htim); //有些需要开启定时器，测试不用
>          HAL_TIM_IC_Start_IT(htim,TIM_CHANNEL_1); 
>          //重新开启通道捕获中断，形成闭环，因为HAL库中断函数关闭了中断
>        }
>    }
>}
>```
>
>上述代码：需要注意：标志位的使用，而且需要在定时器清零前读出并计算数值。
>
>需要掌握以下几个函数和宏：
>
>`__HAL_TIM_GET_COMPARE(htim,TIM_CHANNEL_1)` 获得捕获寄存器的值，也就是CCR的值
>
>`__HAL_TIM_SetCounter(htim,0)` 设置CNT寄存器的值，计算完成后，开启下一次捕获计时计算时用于清零CNT
>
>`__HAL_TIM_SET_CAPTUREPOLARITY(htim,TIM_CHANNEL_1,TIM_INPUTCHANNELPOLARITY_FALLING)` 更改输入捕获通道的触发边沿
>
>`HAL_TIM_Base_Start_IT(htim)` 重新使能定时器，我的版本(HAL1.3.0版本)中断函数中没有关闭定时器
>
>`HAL_TIM_IC_Start_IT(htim,TIM_CHANNEL_1)` 重新开始中断捕获
>
>`TIM_CHANNEL_1` 设置时的通道宏，要与 `HAL_TIM_ACTIVE_CHANNEL_1` 检测时的通道宏相区别。 

>  `mian.c` 初始化后，进入 `while(1)` 开启第一次中断捕获
>
>```c
>  HAL_TIM_IC_Start_IT(&htim8,TIM_CHANNEL_1);
>  while(1)
>  {
>      
>  }
>```

上述代码的检测原理可以表示为下图

![image-20230125153238229](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251533309.png)

由于CNT是1us增加一次，所以Duty(D) 和Frequency(F) 的计算如下：
$$
D = \frac{high\_time}{all\_time} \\
F = \frac{1}{1(us) \times all\_time} = \frac{1}{\frac{1}{1M} \times all\_time} = \frac{1M}{all\_time}
$$

### **单输入、双捕获通道+硬件从模式复位 + DMA数据传输**

#### 定时器模式设置(CubeMX)

- 参考网上一个[博客]([蓝桥杯嵌入式基于STM32G4的模块总结【HAL库】【省赛】_蓝桥杯嵌入式用什么板子_Starrism丶的博客-CSDN博客](https://blog.csdn.net/qq_51573030/article/details/123618862)) 

  使用双捕获通道(IC1、IC2): 

  - 一个通道(IC1)直接(Direct)接入输入通道(TI1),上升沿捕获

  - 一个通道(IC2)非直接(Indirect)接入输入通道(TI1)，下降沿捕获

  同时使用输入通道1捕获信号(TI1TP1)作为定时器的从模式的重启信号，也就是说

  - 定时器设置为从模式(Slave Mode)，从模式为复位模式(Reset Mode)，即 定时器会接收一个信号进行复位(CNT清零)
  - 并将从模式的触发源(Trigger Source) 设为输入通道1的捕获信号 (TI1FP1)
  
  这样设置后，我们便可以从定时器的CCR1(通道1比较寄存器)读出两次高电平间的时间间隔(all_time);从CCR2(通道2比较寄存器)读出下降沿到上次上升沿间的时间间隔(high_time),以此可以计算Duty和Frequency。
  
  信号的触发流程可以参考下图：
  
  ![image-20230125164709065](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251647807.png)
  
  **CubeMX的定时器配置**如下：(以TIM3CH1为例)
  
  ![image-20230125165333720](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251653076.png)
  
    

#### DMA数据传输设置(CubeMX)

- 实测上述的数据，如果使能中断，并在中断处理两个通道的数据，当接入的信号频率较高时频繁的中断，使得单片机卡死，因此使用 `双捕获通道+硬件从模式复位`时开启DMA来传输数据，并注意 **将CubeMX中的DMA传输完成中断关闭，定时器的通道捕获中断也全关闭**

  ![image-20230125181451574](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251814840.png)

- 由于只需要传输16bit的CCR中的数据，因此使能两个通道的DMA，地址都不自增，并设置循环接收。

  ![image-20230125181508058](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251815548.png)

#### 代码编写

- 使用`单输入、双捕获通道+硬件从模式复位 + DMA数据传输` 方式进行信号测量，我们就不需要编写较为复杂的代码，只需要在 `main.c` 文件中主循环前开启输入捕获的DMA接收即可。

  > `mian.c` DMA接收输入捕获的数据并计算(计算公式与上文一致)
  >
  >```c
  >uint16_t cap_dam_high_buff = 0;
  >uint16_t cap_dam_all_buff = 0;
  >
  >  HAL_TIM_IC_Start_DMA(&htim3,TIM_CHANNEL_1,(uint32_t *)&cap_dam_all_buff,1);
  >	//注意，数据接收buff需要强转成 uint32_t *的数据类型，但我们已设置过半字传输，因此接收的数据只有半字
  >  HAL_TIM_IC_Start_DMA(&htim3,TIM_CHANNEL_2,(uint32_t *)&cap_dam_high_buff,1);
  >
  >while(1)
  >{
  >      cap_duty[1] = ((double)cap_dam_high_buff)/((double)cap_dam_all_buff);
  >    //注意，由于DMA接收的buff均是uint16_t类型，在相除需强转成double
  >      cap_fre[1] = 1000000.0/((double)cap_dam_all_buff);
  >}
  >```

- ==待测试！！！！！== 因为使用DMA传输测试出的频率，最高有30000Hz,与查到的24kHz,还有中断计算出的25KHz不符，待使用逻辑分析仪测试。

## 蓝桥杯使用建议

- 需要掌握单通道捕获方案的代码，因为捕获更长的时间是需要使用（注意HAL提供的函数和宏接口的名称）。
- **但需要注意，实测当两个输入捕获都使用 `单捕获通道方案` 时，输入频率较高时，25KHz左右，单片机会因为频繁中断而卡住**
- 建议在多个输入捕获时使用`双捕获通道+硬件从模式复位+DMA方案`