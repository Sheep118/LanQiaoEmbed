# 蓝桥杯嵌入式stm32HAL库ex-PWM高级

## pwm波互补输出

- 在高级定时器(TIM1/8)中，每个通道都有两个输出(如 `CH1和CH1N`) 。这两个通道同时使用时，输出的pwm波是互补的。下面使用 `TIM8CH1N` 配置，单独输出一路PWM波2kHz。

- 同时开启`CH1和CH1N` 时，输出的两路PWM波是互补的，但如果仅仅单独使用一路(不论是`CH1` 还是 `CH1N`) 都是可以配置PWM模式，占空比，和高低电平的。

### CUBEMX配置

![image-20230221230057498](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302212301163.png)

- 单纯输出PWM波的话，不需要使能中断。

### 代码编写

>HAL使用上只是调用的库不同。
>
>- 一般的通道1(CH1)使用时，
>
>  启动PWM调用：`HAL_TIM_PWM_Start(TIM_HandleTypeDef *htim, uint32_t Channel)`
>
>  关闭时使用：`HAL_TIM_PWM_Stop(TIM_HandleTypeDef *htim, uint32_t Channel)`
>
>- 互补通道1(CH1N)使用时，
>
>  启动PWM调用：`HAL_TIMEx_PWMN_Start(TIM_HandleTypeDef *htim, uint32_t Channel)`
>
>  关闭时使用：`HAL_TIMEx_PWMN_Stop(TIM_HandleTypeDef *htim, uint32_t Channel)`