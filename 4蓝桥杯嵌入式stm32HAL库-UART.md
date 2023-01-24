# 蓝桥杯嵌入式stm32HAL库-UART

## 板子上的硬件

![image-20230120120944328](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301201209485.png)

板子上的USB口是DAP调试端口，因此其也与板子上stm32的串口1(PA9、PA10)相连。

## 驱动代码的编写

分为接收和发送两部分，==接收时需要注意在CubeMX中开启中断== 因为默认设置了串口时不开启的。

### 串口发送的实现

串口的发送主要有两种方式，一个是重定向 `fputc()` 函数，使用`printf()` 进行串口的打印；另一种是自己实现格式打印函数。这里没有重定向 `fgetc()` 和 `scanf()` 函数，因为，习惯对发送来的原始数据进行处理。

#### 重定向 `fputc()` 函数

需要注意的是  **`fputc()`  函数的格式(参数和返回值)**，不然会报错或者报警告，还需要一个C语言的标准头文件 `stdio.h`  `fputc()` 的函数声明在其中。 

>`uart1.c` 文件，重定向 `fputc()`
>
>```c
>#include <stdio.h>
>
>int fputc(int ch,FILE * f)
>{
>    HAL_UART_Transmit(&huart1,(uint8_t *)&ch,1,0xFFF); 
>    // 串口阻塞发送函数，最后一个参数是最长超时时间。
>    return ch;
>}
>```
>
>**同时需要注意的是CubeMX初始化后的`huart1`是在`main.c`文件中的，使用`extern`声明到自己的头文件中，方便使用。**
>
>`uart1.h` 文件参考
>
>```c
>#include "stm32g4xx_hal.h"
>extern UART_HandleTypeDef huart1;
>```

#### 自己实现格式打印函数

需要注意的是，使用到`可变长列表类型 va_list` 需要包含三个C语言标准头文件，标准库头文件 `stdlib.h` ，标准参数格式头文件 `stdarg.h` 标准字符串处理头文件 `string.h` 。

>`uart1.c` 文件，实现格式化打印函数。需要学习其中字段的意义，因为其他地方用到时自己才会写。
>
>```c
>#include "stdlib.h"
>#include "stdarg.h"
>#include "string.h"
>#include <stdio.h>
>
>#define USART1_SEND_BUFFER_SIZE 200  //发送缓冲区大小
>void usart1_printf(char * fmt,...) //可变长参数
>{
>    char buffer[USART1_SEND_BUFFER_SIZE] = {0};
>    va_list arg_ptr; //可变长列表类型
>    va_start(arg_ptr,fmt); //给可变长变量类型变量赋值它的的初始地址指针
>    vsnprintf(buffer,USART1_SEND_BUFFER_SIZE,fmt,arg_ptr); // 已指定格式、大小打印到buffer中
>    HAL_UART_Transmit(&huart1,(uint8_t *)buffer,strlen(buffer),0XFFF); //将数据发送
>    va_end(arg_ptr); //销毁变量指针
>}
>```



### CubeMX开启中断接收

通常异步串口都是使用**阻塞发送+中断接收**的方式进行配置，需要注意的是在CubeMX中NVIC处开启中断。

>`CubeMX` 配置异步串口和中断
>
>![Untitled picture](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301201215350.png)
>

### 中断优先级和波特率的修改
>`stm32g4xx_it.c` 文件，在CubeMX中开启中断后可以在该文件中发现，对应的中断函数已经被HAL接管。
>
>![Untitled picture](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301201217135.png)
>
>**`stm32g4xx_hal_msp.c`** 文件中的 `HAL_UART_MspInit(UART_HandleTypeDef* huart)` 函数会在串口初始化时被`main.c` 中的 ` MX_USART1_UART_Init()` 调用，如果需要修改串口中断的优先级，可以在这个函数中修改。
>
>![Untitled picture](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301201221699.png)
>
>在`main.c` 的`MX_USART1_UART_Init()` 可以修改串口的波特率。

### 中断回调函数的使用

需要注意的是中断完成回调函数 `HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)` 是**在串口中断接收完成时HAL库调用的，并非是中断接收函数(一有数据接收就调用的函数)** 

另一个需要注意的点是 ==串口中断函数在接收完成后会关闭中断，在调用上面的接收完成回调函数，因此，**下一次要重新接收就需要重新开启中断**== 开启中断接收的函数为 **`HAL_UART_Receive_IT(<串口句柄>,<接收缓冲句柄>,<接收缓冲数量>);`**

因此，有以下三种方案：

#### 方案一：轮询中断接收启动函数

因为从中断接收启动函数**`HAL_UART_Receive_IT(<串口句柄>,<接收缓冲句柄>,<接收缓冲数量>);`**中可以获得接收的数据。因此只需要轮询该函数，便能持续处理接收数据。

>`mian.c` 文件中使用前后台系统进行50ms一次的轮询即可。简单模板示意如下：
>
>实测轮询时接收多个字节也是可以的。
>
>```c
>while(1)
>{
>   if(task_flag[0])//50ms
>    {
>      HAL_UART_Receive_IT(&huart1,&uart_buffer,1);
>       //缓存，可以不只接收一个字节，接收多个字节也是可以的
>      task_flag[0] = 0;
>    } 
>}
>	
>```

#### 方案二：中断完成回调函数中重新开启接收形成闭环 

这个方案使用时需要注意以下两点：

1. 在 `main函数主循环`  的`while(1)` 前**==需要开启一次接收中断(第一次进入中断)==，否则默认初始化后时关闭串口中断的** 。

   >`mianc.c` 文件，开启第一次进入中断
   >
   >```c
   > HAL_UART_Receive_IT(&huart1,&temp,1);
   >  while (1)
   >	{
   >	}
   >```

2. 在**中断完成回调函数中重新开启中断**，使下一次接收也能进入中断完成回调函数，形成闭环。

   >建立`uart1.c` 文件，完成回调函数的闭环
   >
   >```c
   >uint8_t buffer = 0; //使用静态变量或全局变量
   >void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
   >{
   >    if(huart->Instance == USART1) //检测是不是串口1的回调
   >    {        
   >        printf("buffer = %c\r\n",buffer); //串口回显
   >        HAL_UART_Receive_IT(&huart1,&buffer,1); //重新开启中断
   >    }
   >}
   >```
   >
   >同时需要注意两点，
   >
   >- ==完成回调函数中接收到的数据，是上一次调用 `HAL_UART_Receive_IT`得到的数据，**回调函数中对数据的处理需要使用静态变量，因为时上一次结束回调时读取的变量**==
   >- ==**实测在串口接收完成回调函数中尽量不要接收多字节**，接收多字节时会出现数据混乱==

#### 方案三：在串口中断函数中重新开启中断接收，形成闭环

另一种可取的做法是将 `HAL_UART_Receive_IT(&huart1,&temp,1);` 作为**中断使能函数**放在中断服务函数中，将中断服务函数改写为如下：

>``stm32g4xx_it.c` 文件,在中断中重新开启中断接收
>
>```c
>void USART1_IRQHandler(void)
>{
>  /* USER CODE BEGIN USART1_IRQn 0 */
>  /* USER CODE END USART1_IRQn 0 */
>  HAL_UART_IRQHandler(&huart1); //HAL库的标准中断回调函数，其中会失能中断，并调用发送完成回调函数
>  /* USER CODE BEGIN USART1_IRQn 1 */
>  HAL_UART_Receive_IT(&huart1,&temp,1);//重新开始中断 
>  /* USER CODE END USART1_IRQn 1 */
>}
>```
>
>之后**需要在中断完成回调函数中，处理接收的数据 `temp`。**

## 蓝桥杯的建议

快速使用时，只需要重定向 `fputc()` 函数，并且开启串口中断接收，50ms轮询 一次 `HAL_UART_Receive_IT(&huart1,&temp,1);` 函数，即方案一。但自己构造格式化打印函数需要掌握。

