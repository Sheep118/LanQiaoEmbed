# 蓝桥杯嵌入式stm32HAL库-RTC时钟

## 板子上的硬件

### 关于LSE

- 需要注意的一点是，由于板子上**没有外置32KHz的低速外部晶振(LSE)** 因为连接外部晶振的引脚(PC14、PC15) 被用作GPIO了，同时板子上也没有用于低功耗供电的纽扣电池。因此，RTC是没有办法掉电运行的。

- 补充说明一点：通常都是使用LSE和纽扣电池的方式，因为维持HSE和LSI的运行都比较耗电。

- 因此，使用板子上的RTC只能用LSI或者HSE分配提供时钟。

### 关于RTC的分频器

- 以下是stm32G4系列参考手册的截图。

  可以看出RTC的时钟进入后会经过两个分频器(异步分频器、同步分频器)，经过异步分屏器后的信号会使毫秒寄存器自加，因此，**异步分频器决定了 `SubSeconds` 的最小刻度**。

  而经过异步分频器的信号 再经过同步分频器分频后才输入时间、日期寄存器，因此最终经过两次分频后的信号才是RTC时间寄存器自加的信号。因为是以秒为单位，因此，**经过两次分频后的信号一般设置为(1Hz)**

  ![](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301262055244.png)

- RTC时基的计算公式

  `SubSecond` 自增频率的计算公式为 $f_{SS} = \frac{f_{RTC}}{PSC\_A+1}$

  `Second` 自增频率的计算公式为 $f_S = \frac{f_{RTC}}{(PSC\_A+1)(PSC\_S+1)}$

  其中，$PSC\_A、PSC\_S$ 分别是异步、同步分频器的值。手册的说明截图如下：
  
  ![image-20230126210928938](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301262111214.png)

## 驱动代码

RTC主要用作两方面，一是实时时钟，而是闹钟中断(唤醒)。

### 实时时钟

#### CubeMX配置

- CubeMX配置方面需要注意：

  - **选择对应的时钟源、并将最终输出频率配置为 1Hz**
  - 注意内部寄存器中是BCD编码，但HAL库位我们做好了转换；因此**只需要注意日期格式(Data Format)的选择即可(==通常选择Binary==)**

- 通常情况下会这样设置

  | RTC时钟               | AsynchronousPSC | SynchronousPSC | 备注                  |
  | --------------------- | --------------- | -------------- | --------------------- |
  | LSI(32kHz)            | 32              | 1000           | Subseconds为1ms自增   |
  | LSE(32分频输入750kHz) | 75              | 10000          | Subseconds为0.1ms自增 |

  下图是以HSE的32分频(750kHz)做RTC时钟为例的配置

  ![image-20230126214527387](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301262145037.png)

  ![image-20230126220257805](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301262203006.png)

#### **BCD编码说明**

- BCD编码就是用8个bit(0-15)的内存储存 0-9的数据，并且每8bit构成十进制的一位数。

- HAL库中的 `BCD data format` **实际上就是将寄存器中的原始数据输出或者是设置数据直接输入寄存器。**因为RTC寄存器的日期、时钟都是BCD编码格式储存。

  HAL库中的 `Binary data format` 是指 内存储存是2进制储存，也就是数据的数值就是数据的大小。也就是说，**将BCD格式的数据换算成了数据大小输出(10进制)** 。

- 所以，**一般使用 `Binary data format` 格式进行读写即可(在HAL提供的接口中)**。这样就不需要我们自己换算。

- 同时由于BCD编码使用16进制打印时其显示出来的数字的值就是10进制对应的大小。因此可以用以下使用习惯。

  | 格式 | 目的                           | 设置格式                              | 读取格式 |
  | ---- | ------------------------------ | ------------------------------------- | -------- |
  | BCD  | 显示的数字以10进制读出是其大小 | 0xAA(AA就是其大小，如0x23,大小就是23) | %x       |
  | BIN  | 显示的数字以10进制读出是其大小 | AA(AA就是其大小，23,大小就是23)       | %d       |

#### 代码使用

- RTC配置完CubeMX后，代码编写比较简单。

- 主要有以下几个结构体和函数

  > `RTC_DateTypeDef` `RTC_TimeTypeDef` 日期结构体、时间结构体,其成员分别如下：
  >
  >```c
  >typedef struct
  >{
  >    uint8_t WeekDay; 
  >    uint8_t Month;   
  >    uint8_t Date;   
  >    uint8_t Year;
  >} RTC_DateTypeDef;
  >typedef struct
  >{
  >  uint8_t WeekDay;
  >  uint8_t Month;  
  >  uint8_t Date;   
  >  uint8_t Year;   
  >} RTC_DateTypeDef;
  >```
  >
  > 获取时间和日期的函数
  >
  >```c
  >RTC_DateTypeDef sDate;
  >RTC_TimeTypeDef sTime;
  >
  >HAL_RTC_GetTime(&hrtc,&sTime,RTC_FORMAT_BIN); //注意BIN格式读出
  >HAL_RTC_GetDate(&hrtc,&sDate,RTC_FORMAT_BIN);
  >```
  >
  >设置时间和日期的函数
  >
  >```c
  >RTC_TimeTypeDef sTime = {0};
  >sTime.Hours = 23;
  >sTime.Minutes = 59;
  >sTime.Seconds = 50;
  >sTime.SubSeconds = 0;
  >HAL_RTC_SetTime(&hrtc, &sTime, RTC_FORMAT_BIN)
  >```
  >
  >```c
  >RTC_DateTypeDef sDate = {0};
  >sDate.WeekDay = RTC_WEEKDAY_FRIDAY;
  >sDate.Month = RTC_MONTH_FEBRUARY;
  >sDate.Date = 28;
  >sDate.Year = 21;
  >HAL_RTC_SetDate(&hrtc, &sDate, RTC_FORMAT_BIN)
  >```

- 临时关闭RTC(暂停或者从新启动RTC)可以使用通过开关RTC的时钟来实现

  `__HAL_RCC_RTC_ENABLE()` 使能RTC时钟

  `__HAL_RCC_RTC_DISABLE()` 失能RTC时钟。

### 闹钟中断

#### CubeMX配置

- **首先需要根据上文配置好实时时钟**

- **需要注意的是要在NVIC钟开启闹钟中断，才会进入对应的中断函数。**

  下图是闹钟配置的CubeMX的截图:(注意：使用 `Data Weekday` 屏蔽可以设置出一个每天都响的闹钟)

  ![image-20230126222302071](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301262223764.png)

#### 代码编写

- 闹钟中断的代码编写也比较简单，需要注意的是在HAL的中断函数中是将其对应中断关闭了的，因此，**如果需要重复中断，就需要重新开启闹钟中断**。

  >闹钟中断回调函数：
  >
  >```c
  >void HAL_RTC_AlarmAEventCallback(RTC_HandleTypeDef *hrtc)
  >{
  >    if(hrtc->Instance == RTC)
  >    {//也可以不用检测该实例，因为只有一个RTC
  >		//对应的操作
  >    }
  >}
  >```
  >
  >值得注意的是我们设置闹钟时需要通过结构体初始化闹钟，因此，结构的每一个成员，尽量都初始化赋值，不然默认为0时可能无法开启闹钟，如 **`sAlarm.AlarmMask` `sAlarm.Alarm` 这两个成员决定了是否屏蔽日期和工作日，还有闹钟实例是哪个(`RTC_ALARM_A`还是`RTC_ALARM_B`) 如果这两个成员不赋值的话，闹钟开启后就不可能正常运行进入中断。**如下是，闹钟初始化的模板：
  >
  >```c
  >RTC_AlarmTypeDef sAlarm = {0};
  >sAlarm.AlarmTime.Hours = 0;
  >sAlarm.AlarmTime.Minutes = 0;
  >sAlarm.AlarmTime.Seconds = 0;
  >sAlarm.AlarmTime.SubSeconds = 0;
  >sAlarm.AlarmMask = RTC_ALARMMASK_DATEWEEKDAY; //每日闹钟
  >sAlarm.AlarmSubSecondMask = RTC_ALARMSUBSECONDMASK_ALL;
  >sAlarm.AlarmDateWeekDaySel = RTC_ALARMDATEWEEKDAYSEL_DATE;
  >sAlarm.AlarmDateWeekDay = 1;
  >sAlarm.Alarm = RTC_ALARM_A; //闹钟A
  >if (HAL_RTC_SetAlarm_IT(&hrtc, &sAlarm, RTC_FORMAT_BIN) != HAL_OK)
  >{
  >Error_Handler();
  >}
  >```

## 蓝桥杯使用建议

- **建议使用HSE的32分频作为RTC的时钟源，**

  因为实测使用LSI作为时钟源时，用LCD屏每隔100ms刷新一次时间，会出现秒钟偶尔停止的现象，但使用HSE时则不会。

- 同时注意 **设置闹钟中断的结构体赋值时不要忘了 `Alarm` 成员**，他决定了是闹钟A还是闹钟B。

  
