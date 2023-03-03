# 蓝桥杯嵌入式stm32HAL库-TIM与PWM

## 板子上的硬件

- stm32RBT6一共有11个定时器 (TIM1-8、TIM15-17)，其中，TIM6、7、16、17是普通定时器，只有溢出中断。
- 其他定时器均有通道可以输出PWM波，其中TIM1、8是高级定时器，还具有刹车区间输出等功能。

## 代码的编写

本次主要实现两个功能。

### 定时器溢出

#### CubeMX配置

以通用定时器TIM6为例，使用中断溢出，配置如下，一般默认定时器的时钟均为系统配置的时钟(其中经过不同总线的分频、倍频不再此叙述) 即80MHz。

![image-20230124142616338](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301241426193.png)

定时计算公式：
$$
每次溢出的时间 = \frac{(PSC+1)(ARR+1)}{定时器时钟} = \frac{(PSC+1)(ARR+1)}{80M} (单位：s) 
$$
因此上面配置的是100ms中断一次。

同时需要注意的是，**在 `NVIC` 中使能对应的中断**，如下：

![image-20230124143004779](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301241430998.png)

这样，在代码中的 `stm32g4xx_it.c` 文件中才有对应的中断函数。

#### 代码编写

- 思想：**HAL的中断逻辑仍然是，配置外设后默认关闭，并接管了对应的中断函数，在中断函数中清除了标志位，并关闭外设，同时调用回调函数**

- 因此，我们只需要注意以下几点

  >开启定时器的函数
  >
  >```c
  >__HAL_TIM_CLEAR_IT(&htim6,TIM_IT_UPDATE); 
  >//清楚中断标志位，避免已开启定时器就进入中断
  >HAL_TIM_Base_Start_IT(&htim6);
  >//开启定时器中断
  >```
  >
  >关闭定时器中断函数
  >
  >```c
  >HAL_TIM_Base_Stop_IT(&htim6); //关闭定时器中断
  >```
  >
  >完善定时器中断函数/回调函数
  >
  >定时器中断函数
  >
  >```c
  >void TIM6_DAC_IRQHandler(void)
  >{
  >  /* USER CODE BEGIN TIM6_DAC_IRQn 0 */
  >
  >  /* USER CODE END TIM6_DAC_IRQn 0 */
  >  HAL_TIM_IRQHandler(&htim6);
  >  /* USER CODE BEGIN TIM6_DAC_IRQn 1 */
  >    __HAL_TIM_CLEAR_IT(&htim6,TIM_IT_UPDATE); //清楚中断溢出标志位
  >    HAL_TIM_Base_Start_IT(&htim6); 
  >    //重新开启定时器，形成闭环,注意重新开启定时器需要在定时器接管函数后
  >  /* USER CODE END TIM6_DAC_IRQn 1 */
  >}
  >```
  >
  >定时器中断完成回调函数，这是一个 `__weak` 的函数，也就是说我们可以直接定义一个
  >
  >```c
  >void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
  >{
  >  //对应操作
  >  __HAL_TIM_CLEAR_IT(htim,TIM_IT_UPDATE); //清楚中断溢出标志位
  >  HAL_TIM_Base_Start_IT(htim); 
  >}
  >```

- 由于我查看HAL定时器中断函数中是没有关闭定时器的，因此，我们可以不用再重新开启定时器，只需要在需要关闭定时器时调用关闭函数即可。下图是 `stm32g4xx_hal_tim.c` 文件中的定时器中断接管函数 `HAL_TIM_IRQHandler()` 的部分内容。

  ![image-20230124144052970](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301241440100.png)

  可以看出，其只清楚了标志位，并调用回调函数，并没有关闭定时器。但网上的资料中说明这个定时器函数会关闭定时器中断，需要重新开启。我实测并不会(我是用的v1.3.0版本的HAL库) 所以，保险起见，**还是在中断函数或者中断完成回调函数中重新开启定时器，并在需要关闭的时候主动调用关闭定时器函数更为妥当.**

#### 注意定时器代码编写逻辑

如下使用定时器6实现灯0.1s翻转一次，持续5s钟，由于是在外部打开定时器，定时器中计次关闭定时器，因此，进入定时器的次数是循环的上限值，所经过的时间是`循环上限值*单次循环时间`。

代码和分析如下:(其中 `led_count` 是全局变量/静态变量)

```c
void TIM6_DAC_IRQHandler(void)
{
  /* USER CODE BEGIN TIM6_DAC_IRQn 0 */
  /* USER CODE END TIM6_DAC_IRQn 0 */
  HAL_TIM_IRQHandler(&htim6);
  /* USER CODE BEGIN TIM6_DAC_IRQn 1 */
  if(led_count < 50)  //注意这里上限是50
  {
    if(led_count%2)led_ctrl(0xFE);
    else led_ctrl(0xFF);
    led_count ++;
    __HAL_TIM_CLEAR_IT(&htim6,TIM_IT_UPDATE); //清楚中断溢出标志位
    HAL_TIM_Base_Start_IT(&htim6); //重新开启定时器，形成闭环
  } 
  else
  {
    led_count = 0;
    HAL_TIM_Base_Stop_IT(&htim6);
  }
  /* USER CODE END TIM6_DAC_IRQn 1 */
}
```

具体时间计算如下图：

![image-20230124152218373](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301241522468.png)

### PWM输出

#### CubeMX的配置

- 需要注意的有以下两点：

  - **是否开启了定时器的时钟源**，*因为通用定时器的时钟源默认是没有开启的*，**选择内部时钟源(Internal Clock)即可**
  - **是否开启了 `ARR` 寄存器的预装载** ，开启这个后，当ARR改变时不会马上装载进ARR, 而是需要等这个周期的波形完成后再装载ARR，这样，就不会出现使用ARR调节PWM波频率时，调节时刻的那个周期中PWM波出现其他的频率，因为在下一个周期开启时才装载ARR。
  - **`CCR`寄存器的自动重装载也是需要使能的** ，只有使能了这个一个PWM周期完成后CCR的值才会自动装载，但默认配置为PWM输出时，CubeMX已经开启了。
  
  CubeMX的配置可以参考下图：(TIM3CH1,PB4,1KHz,Duty 50%)
  
  ![image-20230125232148920](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301252321113.png)
  
  

#### 代码的编写

- 只需要注意**在 `while(1)` 前开启PWM的输出 `HAL_TIM_PWM_Start(<定时器句柄>, <通道宏>)`**即可。

- 同时需要**掌握几个调节PWM的函数和宏**即可。

  >`mian.c` 文件， `while(1)` 前开启PWM输出。
  >
  >```c
  >HAL_TIM_PWM_Start(&htim3,TIM_CHANNEL_1);
  >while(1)
  >{
  >    
  >}
  >```
  >
  >**几个调节PWM波的函数和宏**
  >
  >`__HAL_TIM_SET_AUTORELOAD(&htim3,arr_val)` 设置ARR寄存器的值，用于调节PWM波的频率。计算公式为：(PSC设置的初始值为79)
  >$$
  >freq_{PWM} = \frac{定时器时钟}{(ARR+1)(PSC+1)} = \frac{80M}{(ARR+1)(PSC+1)}
  >$$
  >`__HAL_TIM_SET_COMPARE(&htim3,TIM_CHANNEL_1,ccr_val)` 设置对应通道的CCR寄存器的值，用于调节PWM波的占空比。计算公式如下：(ARR初始值为999)
  >$$
  >Duty_{PWM} = \frac{CCR}{ARR+1}
  >$$
  >`__HAL_TIM_GET_AUTORELOAD(&htim3)` 获取定时器ARR寄存器的值
  >
  >`__HAL_TIM_GET_COMPARE(&htim3,TIM_CHANNEL_1)` 获取定时器指定通道CCR寄存器的值
  >
  >`HAL_TIM_PWM_Start(&htim3,TIM_CHANNEL_1)` 定时器指定通道的PWM波启动(使能)函数
  >
  >`  HAL_TIM_PWM_Stop(&htim3,TIM_CHANNEL_1)` 定时器指定通道的PWM波关闭(失能)函数
  >
  >注意：虽然HAL函数关于上面两个函数的注释都是`开启一次PWM波输出`  (和其他外设一样Start表示单次输出) 但由于我们使能了ARR的重装载和CCR的预装载 ，所以PWM波是会持续输出的。

- 同时，由于我们没有开启中断(单纯PWM输出是可以不用开启中断的)，如果需要计算输出的脉冲个数，可以开启对应的比较中断或者溢出中断。

### 单定时器多通道输出频率、占空比不同的PWM波

- 由于定时器中ARR决定了CNT的上限，因此像上面一样调整ARR的值，会对定时器的所有通道的频率产生影响。
- 以下是通过**比较输出翻转+定时器中断** 来实现单定时器多通道输出频率、占空比不同的PWM波。

#### cubemx的设置

需要注意的有以下几点：

- *ARR的值尽量设置大一些，这样CNT不易溢出，且如果产生的频率已知，可以将ARR的值设为对应周期的倍数*，修改占空比时CCR的值会周期性变化。

- **通道模式应设置为 `Toggle on match`** 即当CNT值自增到与对应通道的CCR匹配时翻转输出的电平。

  设置参考：

  ![定时器比较输出Cubemx设置参考](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302081446859.png)

#### 代码编写

##### 理论分析：

- 思想：

  在比较输出中断中，给CCR赋值控制下一段电平的时间，通过标志位翻转轮流对CCR赋值高低电平的时间，形成个数、频率、占空比可控的PWM输出。

- 注意：

  需要注意的是CNT一直在自增，因此下一次的CCR的值需要在上一次CCR的值的基础上自增。同时，CNT自增到ARR后会重新归零，因此，所设置的CCR的值需要对ARR取余。

- 例子：以11届中要求PA6 100Hz, PA7 200Hz，且需要占空比可控为例。

  因为PA6是TIM3CH1,PA6是TIM3CH2，频率不同，因此，不能直接用PWM模式输出，两个通道都配置成比较输出模式。

  参考下图：

  ![](D:\typora\local notebook\stm32\蓝桥杯嵌入式\assets\inputpwm.drawio.png)

  

##### 代码完成

主要代码在 `main.c` 中完成 `HAL_TIM_OC_DelayElapsedCallback` 输出比较溢出的回调函数即可。(也可以使用 `HAL_TIM_PWM_PulseFinishedCallback` 函数，效果一致 )

>`main.c`  以下的代码可以通过改变pa6_T 和pa6_duty来调节周期和占空比。
>
>```c
>void HAL_TIM_OC_DelayElapsedCallback(TIM_HandleTypeDef *htim)
>{
>    if(htim->Instance == TIM3)
>    {
>        if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
>        {
>            ccr = __HAL_TIM_GetCompare(htim,TIM_CHANNEL_1); //得到上一次的CCR值
>            pa6_sta = !pa6_sta;
>            if(pa6_sta == 0)
>                __HAL_TIM_SetCompare(htim,TIM_CHANNEL_1,(ccr + pa6_T - pa6_duty)%arr);
>            else
>                __HAL_TIM_SetCompare(htim,TIM_CHANNEL_1,(ccr + pa6_duty)%arr);
>//            HAL_TIM_OC_Start_IT(&htim3,TIM_CHANNEL_1);
>        }
>        else if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_2)
>        {
>            ccr = __HAL_TIM_GetCompare(htim,TIM_CHANNEL_2);
>            pa7_sta = !pa7_sta;
>            if(pa7_sta == 0)
>                __HAL_TIM_SetCompare(htim,TIM_CHANNEL_2,(ccr +pa7_T - pa7_duty)%arr);
>            else
>                __HAL_TIM_SetCompare(htim,TIM_CHANNEL_2,(ccr + pa7_duty)%arr);
>            //HAL_TIM_OC_Start_IT(&htim3,TIM_CHANNEL_2);
>            //可以不需要再次开启
>        }
>    }
>}
>```
>
>使用 `__HAL_TIM_SetCompare()` 和 `__HAL_TIM_GetCompare` 两个宏设置和读取CCR的值。

>需要记得在 主循环前开启 输出比较中断。
>
>```c
>HAL_TIM_OC_Start_IT(&htim3,TIM_CHANNEL_1);
>HAL_TIM_OC_Start_IT(&htim3,TIM_CHANNEL_2);
>```

## 蓝桥杯建议

需要注意 单定时器+双通道PWM输出中网上有个问题称ARR设为65535(最大) CCR作为16位自增时，不取余，无法正常运行, **待验证**。

