# STM32G431关于TIM7HAL错误

## 现象

- 使用HAL库开启TIM7中断时卡死，根据正常步骤使用HAL库TIM7的定时溢出中断。发现当TIM7达到溢出中断时，整个单片机卡死。

- 定时器7的配置如下：

  ![image-20230222164029421](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302221640882.png)

- 使用的HAL库版本为1.2.0

  ![image-20230222164148654](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302221641201.png)

## 解释

原因：HAL中存在一个宏定义错误，导致没有设置相关的中断函数，致使中断触发后卡死在启动文件的中断向量表后。

1. 中断向量和中断函数

>如下，是CubeMX()中的中断函数位于 `stm32g4xx_it.c` 文件 
>
>![image-20230222163902606](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302221639184.png)
>
>但在在启动文件 `startup_stm32g431xx.s` 文件中
>
>对应的中断向量对应的虚函数应该是 `TIM7_DAC_IRQHandler` 
>
>![image-20230222164419219](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302221644980.png)
>
>也就是说，在`stm32g4xx_it.c` 文件中的所写中断函数不是TIM7溢出时会调用的。

2. 宏定义

>其实，HAL库使用宏定义，令 `TIM7_IRQHandler` 代替 `TIM7_DAC_IRQHandler`
>
>但库中将其定义反了，如下图的定义所示： `stm32g431xx.h`
>
>![image-20230222165752565](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302221657452.png)

**解决方法：是需要将上面的宏定义改为上述的形式即可**

3. 补充：上述宏定义的路径关系

>在 `stm32g431xx.h` 中include 了 `main.h` -> `stm32g4xx_ll_pwr.h` -> `stm32g4xx.h`->`stm32g431xx.h`