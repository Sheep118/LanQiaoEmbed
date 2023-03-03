# 蓝桥杯嵌入式stm32HAL库-UART+DMA

## 问题

### 问题简述

- 串口的处理方式通常都是 `阻塞发送+异步接收` 之前串口的简单使用中。


- 发送方面使用 `重定向fputc` 或者 `重新串口printf` 都可以，底层调用的都是 阻塞发送的函数 `HAL_UART_Transmit` 函数。


- 接收方面，之前使用的是 50ms轮询中断接收的方式，这样的方式在数据量较小、数据处理较简单的情况下是适用的。但一旦数据量过大，或对数据响应要求较高(如12届的赛题中要求串口响应时间<10ms，就不太适用了)

### 解决方案

解决方案有以下三种：

- 串口中断回调接收
- **DMA+中断回调接收**
- **DMA+串口空闲中断**

## HAL库的回调机制

实际上HAL库中有一套通用的回调机制: (下面以串口回调为例)

- **对于有多种回调函数,像串口、DMA等，回调函数会以函数指针的形式赋值给对应对象中的函数成员。**
- 而调用时，是直接调用该成员。

### 串口接收回调机制

#### 回调函数的设置和调用

由于串口有多种格式的接收 (FIFO失能/使能，16/8/7bit数据位) 所以，串口接收的回调函数，是以函数指针的方式赋值给 `huart->RxISR` 该结构体成员。

而设置该成员变量的字段就是在 `HAL_UART_Receive_IT()` 或其他的开启接收函数中。

![回调函数设置](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302071248454.png)

回调函数的调用，在中断函数中调用：

![回调函数的调用](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302071250968.png)

#### *HAL_UART_RxCpltCallback 的调用关系*

有了上面的函数成员的设置，我们就可以知道回调函数最终的调用关系，下面以8bit数据位FIFO关闭为例。

开启接收中断 `HAL_UART_Receive_IT()` 设置了接收缓冲后，当有串口接收到一个数据时：

![串口接收完成回调函数机制](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302071352924.png)

同时可以看出，调用 `HAL_UART_Receive_IT()` 函数，如果没有接收到要求数量的数据是不会进入接收完成回调函数的。

其在 `HAL_UART_Receive_IT()` 函数中也保存了缓冲区指针和数量，如下：

![接收回调函数设置参数部分截图](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302071240806.png)

HAL库数据处理部分代码截图如下：

![HAL库数据处理部分代码截图](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302071356791.png)

### 串口DMA接收回调机制

#### DMA回调函数的设置

同样的DMA的回调函数，也是在设置串口接收时 `HAL_UART_Receive_DMA` 赋值给结构体成员，通过结构体成员调用。

![image-20230207221352363](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302072214376.png)

![设置成员函数](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302072222712.png)

#### *串口+DMA数据接收回调函数调关系*

在开启串口接收与DMA绑定后，开启DMA发送(搬运)完成中断后，再中断函数中的调用关系如下：

![串口DMA数据回调流程](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302072249416.png)



## 方案一：串口接收中断闭环

### 方案实现

分为以下几个步骤：

- 开启串口中断函数
- 在主循环前调用第一次中断接收 `HAL_UART_Receive_IT()` 开始中断接收。
- 中断完成回调函数中调用 `HAL_UART_Receive_IT()` 重新开始中断接收(闭环)。
- 有两点需要注意：
  - 适用于接收定长的数据，不定长的数据很有可能使接收中断终止(下方有解释)。因此，第一次调用和中断中调用时接收数据长度最好一致。
  - 由于中断完成回调函数 `HAL_UART_RxCpltCallback` 处理的是上一次接收的数据，因此需要使用静态变量。

### **存在的问题**

使用串口中断闭环接收时，存在一个很严重的问题：当接收的数据长度与我们设置接收的数据长度不同时，串口中断会接收到缓冲区满时才调用接收函数，并且如果此时还有数据接收，实测发现，会导致串口中断接收终止，`RxISR` 函数为NULL。

**也就是说，如果发送过来的数据`不是定长的`，`且与我们设置的数据接收长度不成倍数关系`的话，串口中断在缓冲区满后便无法再进入 接收完成回调函数`HAL_UART_RxCpltCallback`。**

如果，我们是在接收完成回调函数 `HAL_UART_RxCpltCallback` 重新使能串口接收的话，上述的情况发送时，串口中断会直接终止。

具体分析和猜测如下：

![串口中断终止的猜想](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202302072204262.png)

## **方案二：串口DMA接收定长数据**

### *优点*

因为使用中断闭环接收时，经常有数据丢失的现象，而容易受串口数据影响而导致中断终止。

因此，使用DMA进行数据传输，会更加可靠，实测`使用DMA和 接收完成回调函数`进行处理时，不定长的数据超过缓冲区时只会对数据进行循环覆盖，不会导致数据接收处理终止。

猜测是因为数据缓冲区满时产生的中断不会对DMA数据搬运产生影响，而串口中断时则会被打断。

### **==方案实现==**

主要步骤（CUBEMX的设置不在此展示）：

- 设置串口DMA，**循环接收，内存地址自增**

- **关闭串口中断**  并 **开启DMA传输完成中断**
- **完成串口接收完成回调函数 `HAL_UART_RxCpltCallback` ** 

> `串口+DMA传输时 ` 接收完成回调函数示例`HAL_UART_RxCpltCallback`
>
> ```c
> void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
> {
>     if(huart->Instance == USART1) //串口1
>     {
>         rx_buff[22] = '\0';
>         printf("rx_buff = %s\t len =%d\r\n",rx_buff,strlen((char*)rx_buff));
> 		//可以通过strlen()函数取出接收到的字串大小，但没有什么意义，因为会进入中断接收，必然时接收到我们要求的数据大小。
>         memset(rx_buff,0,23);
>         //清空缓冲区
>     }
> }
> ```

### *缺陷*

虽然使用DMA接收避免了出现串口中断接收不定长数据时终止的情况。但只有在数据接收到要求的数据量大小时才会进行处理。缺陷有以下两个：

- 对于不定长的数据，较短的数据不能及时处理，较长数据会产生覆盖。
- 设置的接收数量过大，接收处理不及时。

## **方案三：串口空闲中断+DMA处理不定长数据**

### *优点：*

串口的空闲中断(IDLE) 时串口在接收到第一数据后，第一次出现数据断流，即出现空闲帧（8数据1停止0检验时10bit全高）时产生的中断。可以很好的解决上述不定长度数据接收问题，且在发送停止时可以及时处理。

### **==方案实现==**

#### cubeMX配置

- 配置串口DMA，**循环接收、内存地址自增**
- **关闭DMA传输完成中断** （因为不需要） 同时， **开启串口中断**

代码中需要实现以下三步

#### 代码编写

为了和HAL库的逻辑一致我们在**串口中断函数 `USART1_IRQHandler`** 中 调用我们的**用户串口中断接管函数 `USER_UART_IRQHandler`** ，在这个函数在，我们坚持对应的标志位，清除对应的标志位，并调用我们自己定义的**串口空闲回调函数`USER_UART_IDLEcb`**

三个函数的示例代码如下：

>`stm32g4xx_it.c` 串口中断函数在这个文件，我们在里面定义`USER_UART_IRQHandler` 函数。
>
>```c
>void USART1_IRQHandler(void)
>{
>  /* USER CODE BEGIN USART1_IRQn 0 */
>
>  /* USER CODE END USART1_IRQn 0 */
>  HAL_UART_IRQHandler(&huart1);
>  /* USER CODE BEGIN USART1_IRQn 1 */
>  USER_UART_IRQHandler(&huart1); //调用自己的串口接管函数
>  /* USER CODE END USART1_IRQn 1 */
>}
>```
>
>```c
>//两种检测标志位和清楚标志位的方法，均可，需要注意参数宏不是同一个
>void USER_UART_IRQHandler(UART_HandleTypeDef *huart)
>{
>//    if(__HAL_UART_GET_FLAG(huart,UART_FLAG_IDLE))
>//    {
>    if(__HAL_UART_GET_IT(huart,UART_IT_IDLE))
>    {
>        __HAL_UART_CLEAR_IT(huart,UART_CLEAR_IDLEF);
>        //注意，两个标志位不是同一个宏
>//        __HAL_UART_CLEAR_IDLEFLAG(huart); //清除中断标志位，避免重复进入
>        USER_UART_IDLEcb(huart);
>    }
>}
>```
>
>在 `main.h` 中声明 `USER_UART_IDLEcb`  便于在 `main.c` 中编写回调函数相关的处理代码。而 `USER_UART_IRQHandler` 这个函数声明在 `stm32g4xx_it.c` 中即可。
>
>`main.h` 中的函数声明：
>
>```c
>/* USER CODE BEGIN EFP */
>void USER_UART_IDLEcb(UART_HandleTypeDef *huart);
>/* USER CODE END EFP */
>```

回调函数的编写：

>`main.c` 串口空闲中断回调函数模板示例
>
>```c
>void USER_UART_IDLEcb(UART_HandleTypeDef *huart)
>{
>    if(huart->Instance == USART1)
>    {
>        HAL_UART_DMAStop(huart);
>        //暂停DMA传输，避免数据覆盖
>//        rx_buff[22] = '\0'; //给缓冲区加字符串结尾符，可用于打印
>        strlen((char*)rx_buff); //可用strlen获取收到的字符数量
>        __HAL_DMA_GET_COUNTER(&hdma_usart1_rx); 
>        //也可以读取dma的计数器，计数器初始值是我们设置的读入数量，每传输完成一个字符便-1.
>        buff_size - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx); 
>        //因此，可以计算接收到的字符数量。
>        
>        /*
>        数据处理部分
>        */
>        memset(rx_buff,0,23); //清空缓冲区
>        HAL_UART_Receive_DMA(huart,rx_buff,22); //重新开始接收
>    }
>}
>```
>
>`main.c` 在主循环前使能串口空闲中断，并开始DMA传输
>
>```c
>__HAL_UART_ENABLE_IT(&huart1,UART_IT_IDLE); //使能空闲中断
>HAL_UART_Receive_DMA(&huart1,rx_buff,128); //开始DMA传输
>while(1)
>{
>    
>}
>```
>
>

#### 总结：

- 需要掌握空闲中断标志位检测和清楚的函数和宏

  >检测标志位(建议使用第一种，**注意参数宏名称**)
  >
  >`__HAL_UART_GET_FLAG(huart,UART_FLAG_IDLE);`
  >
  >`__HAL_UART_GET_IT(huart,UART_IT_IDLE);`
  >清除标志位(建议使用第一种，**注意参数宏名称**)
  >
  >` __HAL_UART_CLEAR_IDLEFLAG(huart);`
  >
  >`__HAL_UART_CLEAR_IT(huart,UART_CLEAR_IDLEF);`

- 回调处理的基本步骤

  - 检查实例(是否是串口1，可省略)

  - **暂时关闭DMA `HAL_UART_DMAStop(huart);` ** 避免数据覆盖

  - 获取数据长度(可省略)，掌握一个宏 `__HAL_DMA_GET_COUNTER`

    实际上是访问这个寄存器`huart->hdmarx->Instance->CNDTR`

  - **数据处理**

  - **重新启动DMA接收 `HAL_UART_Receive_DMA(huart,rx_buff,22)` **

- 在**主循环前开启空闲中断和DMA传输**。

- 同时，由于空闲中断可以及时处理不定长数据，因此，**可以将缓冲区大小和每次启动DMA接收的数据量设置得尽量大些** 。

### *缺陷*

如果两个数据帧之间的空闲间隔很短，即很密集的间断数据。此时，因为上面开关DMA的操作，可能会导致后一帧数据有数据缺失。解决方案是，使用空闲中断+DMA+双缓冲区，在第一次接收完成后将数据搬运到新缓冲区中。

## 蓝桥杯建议

**对于定长数据,使用串口+DMA接收完成中断即可** 。

**如果需要处理不定长数据(或者是需要数据传输错误判断的)，使用串口空闲中断+DMA传输**

