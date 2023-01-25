# 蓝桥杯嵌入式板ADC使用

## ADC简介

### ADC类型：

- 并联比较型：通过多个电阻分压，再进行多个电阻端的比较，对M位ADC需要2^M^-1 次方个电阻和比较器

  - 优点：精度高、速度快
  - 缺点：价格昂贵，成本高

  ![image-20230117134555294](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171346362.png)

- 逐次比较型：通过DAC输出的电压和读入的电压进行比较,只需要一个比较器。

  - 优点：成本低
  - 缺点：识别时间长，精度于识别速度成反比

  ![image-20230117134629481](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171346640.png)

## 蓝桥杯上的硬件

### stm32G431RB

- G431有5个ADC，每个ADC的有18个通道，但分为注入通道和规则通道，个数各不相同，注入通道可以使用中断打断规则通道。

- 需要知道每个ADC提供了多个通道一起使用时的扫描模式，并且我们可以改变扫描的顺序。但由于，板子上是一个ADC连接一个滑动变阻器，因此多通道扫描模式在这儿并没有用到。

- 需要知道的还有这块芯片的ADC都是逐次比较型ADC，也就是**识别精度受转换速度(转换周期)设置的影响**

  ![image-20230117140907482](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171409690.png)

- 同时，需要了解ADC转换周期的公式计算：(下图是stm32参考手册中的截图)

  ![image-20230117141858752](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171419388.png)

  上述的公式主要关注第一条：
  $$
  单次转换所需时间 = (所设置的转换周期数 + 12.5)\times ADC时钟周期
  $$

### 板子上的硬件

- ADC在板子上的应用主要是：用于读取板子上两个模拟输出的引脚输入的电压值，如下图

  (另一个可编程电阻在IIC时再说)

  ![image-20230117140040935](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171400689.png)

- **其对应关系如下**：

  | 引脚 | ADC通道   | 板子丝印  |
  | ---- | --------- | --------- |
  | PB12 | ADC1_IN11 | 电压采集1 |
  | PB15 | ADC2_IN15 | 电压采集2 |

  在数据手册中截图如下：

  ![image-20230117140249766](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171402547.png)

  ![image-20230117140305606](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171405143.png)

## 编程设置

常用的ADC读取方式有两种

### 单次转换+软件触发+轮询读取

- 每开启一次ADC就读取一次，因为让ADC一直读取是十分浪费资源的。

- 且，由于每次开启转换后需要CPU阻塞去等待ADC读取完成才能拿到数据，所以，**单次转换时通常会设置较短的转换周期，以便快速读取到ADC值**

#### CubeMX的设置(以ADC2IN15为例)

- 对每一通道都有以下除了使能外有以下两种模式(有些只有一种,如IN15)

  ![image-20230117143349969](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171433060.png)

  解释一下就是：

  - Differential：差分模式(通常需要两个引脚做电压输入，一个作为参考电压，板子上用不到)
  - Single-ended: **单引脚转换**(用内部的参考电压做标准，**通常用这个，直接测出后的值对位数转换到0-3.3之间**)

- 其他的ADC配置主要如下：

  ![image-20230117143958359](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171440687.png)

  需要在意的是**不连续转换失能，还有软件触发**。

  通常情况下精度选择12bit，因此转换公式为
  $$
  实际电压值 = \frac{ADC读取值(16bit长)*3.3}{2^{精度位数}}=\frac{ADC读取值*3.3}{4096}(精度12bit时)
  $$
  同时，为了使CPU阻塞时间尽量短，ADC通道中的转换时间周期设置为最短的2.5个周期

  ![image-20230117144622276](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171446324.png)

#### 代码编写

- 由于读取一次ADC需要开启一次ADC转换，因此可以编写如下函数

  ```c
  float adc_read_signle(ADC_HandleTypeDef *hadc)
  {
      float ret = 0;
      HAL_ADC_Start(hadc); //开启一次ADC转换
      HAL_ADC_PollForConversion(hadc,10); //阻塞等待，设置超时时间
      ret = HAL_ADC_GetValue(hadc)*3.3/4096; //获取读取的值
      return ret;
  }
  ```

### ==循环转换+DMA数据传输==

通常在多个通道转换时，上面的轮询读取可能不及时，导致读取出的ADC结果被覆盖，因为一个ADC只有一个寄存器。而且使用CPU阻塞等待ADC读取完成也十分浪费。因此通常都是使用ADC循环转换+DMA数据传输。使用循环转换的原因是我们不想要每次读取完成都需要再次手动开启，因为此时有DMA帮我们搬运数据，不需要担心读出的数据被覆盖。

#### CubeMX设置(ADC1_IN11为例)

上面已经提过的*精度*、*右对齐*、*使能规则通道*、*软件触发*不再次说明了。

![image-20230117150008850](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171500961.png)

同时需要添加一个DMA请求：

![image-20230117150327845](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301171503027.png)

- 这里需要注意的有三点，**ADC连续转换**、**ADC中开启DMA的连续请求**，**MDA循环接收DMA请求**

  因为连续转换，每次转换结束都会进行DMA请求(就是连续DMA请求)，而且DMA需要连续响应该请求，不然会出错，下面时库函数的注释说明(stm32g4xxx_hal_adc.h中关于`hadc.Init.DMAContinuousRequests` 的注释)：

  ```c
  FunctionalState DMAContinuousRequests; 
  /*!< Specify whether the DMA requests are performed in one shot mode (DMA transfer stops when number of conversions is reached)                             or in continuous mode (DMA transfer unlimited, whatever number of conversions).
  This parameter can be set to ENABLE or DISABLE.
  Note: In continuous mode, DMA must be configured in circular mode. Otherwise an overrun will be triggered when DMA buffer maximum pointer is reached. */
  
  ```

- 还有两点需要特别注意的是

  **==通道转换周期尽量设置长一些(上图设置是47.5个周期)== *因为循环转换时如果转换周期太短，会过快进行DMA请求，DMA数据完成又会触发中断，会导致不停的进出中断，导致单片机卡死***

  还有就是，由于不需要CPU阻塞等待，设置长一点的转换周期有利于读取到更为精确的ADC结果，图中的47.5是在没有其他中断情况下，我测试出的最小不卡死的转换周期，但设置更长些会更好。

  实际上，**==可以关闭CubeMX中的强制DMA中断选项==**， 这样就不会每次完成MDA传输时都触发中断导致单片机卡死。(如下图)
  
  ![image-20230125120920394](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301251209320.png)
  
- **另一个就是使用多通道时，需要开始扫描模式，确定扫描模式的通道顺序【并且最好开启分组扫描，将ADC多个通道分成多个组进行扫描】，并且开启DMA内存地址的自增**，因为一个ADC数据寄存器只有一个，多个通道都会往里面写值，当使用内存地址自增时，每次ADC一个通道转换完成就会触发一次DMA，内存地址自增时，就可以将不同通道的ADC值都读到不同的内存地址中，防止覆盖。

#### 代码编写

- DMA的代码编写比较简单

  ```c
  uint16_t adc1_buffer[2] = {0}; //多少个通道就需要多大的数组
  HAL_ADCEx_Calibration_Start(&hadc1,ADC_SINGLE_ENDED);//ADC矫正，需要在start前，不用也可以
  HAL_ADC_Start_DMA(&hadc1,(uint32_t* )adc1_buffer,2); //DMA每次搬运的数量也是通道决定
  while(1)
  {
      //读adc1_buffer即可
  }
  ```

- 下面给出CubeMX配置的代码段在工程中的位置，方便自己修改。

  >`main.c`的adc初始化函数(hadc各个成员的意思看注释,以单次读取的adc1为例)

  ```c
  static void MX_ADC1_Init(void)
  {
  
    /* USER CODE BEGIN ADC1_Init 0 */
  
    /* USER CODE END ADC1_Init 0 */
  
    ADC_MultiModeTypeDef multimode = {0};
    ADC_ChannelConfTypeDef sConfig = {0};
  
    /* USER CODE BEGIN ADC1_Init 1 */
  
    /* USER CODE END ADC1_Init 1 */
    /** Common config
    */
    hadc1.Instance = ADC1; //adc实例基地址
    hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV2; //adc时钟预分频
    hadc1.Init.Resolution = ADC_RESOLUTION_12B;  //精度
    hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT; //左右对齐
    hadc1.Init.GainCompensation = 0; //增益
    hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;  //是否扫描
    hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;   //转换结束的标志位是单次转换还是序列转换
    hadc1.Init.LowPowerAutoWait = DISABLE;  //低电平等待
    hadc1.Init.ContinuousConvMode = ENABLE;  //是否连续转换
    hadc1.Init.NbrOfConversion = 1;  //每次转换的通道数
    hadc1.Init.DiscontinuousConvMode = DISABLE;  //间断模式(基本不用到)
    hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;  // (通常都是)软件开启或者是硬件定时器、中断等触发
    hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;  //外部触发时的触发边缘
    hadc1.Init.DMAContinuousRequests = ENABLE;  // 是否使能连续的DMA请求
    hadc1.Init.Overrun = ADC_OVR_DATA_PRESERVED;  // 数据是否覆盖
    hadc1.Init.OversamplingMode = DISABLE;  // 过采样，在实现高精度如16bit时会用到
    if (HAL_ADC_Init(&hadc1) != HAL_OK)
    {
      Error_Handler();
    }
    /** Configure the ADC multi-mode
    */
    multimode.Mode = ADC_MODE_INDEPENDENT;
    if (HAL_ADCEx_MultiModeConfigChannel(&hadc1, &multimode) != HAL_OK)
    {
      Error_Handler();
    }
    /** Configure Regular Channel
    */
    sConfig.Channel = ADC_CHANNEL_11; //通道号
    sConfig.Rank = ADC_REGULAR_RANK_1;  //分组中这个通道的排序
    sConfig.SamplingTime = ADC_SAMPLETIME_47CYCLES_5; //采样周期，也就是我们能设置的转换周期
    sConfig.SingleDiff = ADC_SINGLE_ENDED; //单引脚adc还是差分
    sConfig.OffsetNumber = ADC_OFFSET_NONE; //设置偏移量
    sConfig.Offset = 0; //偏移量的值，也就是读出的原始数据每次会加上多少，可以与矫正函数配合，一般不需要
    if (HAL_ADC_ConfigChannel(&hadc1, &sConfig) != HAL_OK) //对ADC的通道进行设置
    {
      Error_Handler();
    }
    /* USER CODE BEGIN ADC1_Init 2 */
  
    /* USER CODE END ADC1_Init 2 */
  
  }
  ```

  >`stm32g4xx_it.c` DMA中断函数
  
  ```c
  void DMA1_Channel1_IRQHandler(void)
  {//对应DMA通道的中断函数
    /* USER CODE BEGIN DMA1_Channel1_IRQn 0 */
  
    /* USER CODE END DMA1_Channel1_IRQn 0 */
    HAL_DMA_IRQHandler(&hdma_adc1);
    /* USER CODE BEGIN DMA1_Channel1_IRQn 1 */
  
    /* USER CODE END DMA1_Channel1_IRQn 1 */
  }
  ```
  
  >`stm32g4xx_hal_msp.c`
  >
  >DMA的具体配置函数在msp.c文件中，因为不同的芯片的DMA不同，所以DMA的配置在msp.c文件中
  
  ```c
  void HAL_ADC_MspInit(ADC_HandleTypeDef* hadc)
  {
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    if(hadc->Instance==ADC1)
    {
    /* USER CODE BEGIN ADC1_MspInit 0 */
  
    /* USER CODE END ADC1_MspInit 0 */
      /* Peripheral clock enable */
      HAL_RCC_ADC12_CLK_ENABLED++;
      if(HAL_RCC_ADC12_CLK_ENABLED==1){
        __HAL_RCC_ADC12_CLK_ENABLE();
      }
  
      __HAL_RCC_GPIOB_CLK_ENABLE();
      /**ADC1 GPIO Configuration
      PB12     ------> ADC1_IN11
      */
      GPIO_InitStruct.Pin = GPIO_PIN_12;
      GPIO_InitStruct.Mode = GPIO_MODE_ANALOG; //引脚模拟输入模式
      GPIO_InitStruct.Pull = GPIO_NOPULL;
      HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
  
      /* ADC1 DMA Init */
      /* ADC1 Init */
      hdma_adc1.Instance = DMA1_Channel1;   // DMA通道
      hdma_adc1.Init.Request = DMA_REQUEST_ADC1;  // 请求方
      hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;  //方向外设->内存
      hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;  //外设地址不自增
      hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;  // 内存地址自增
      hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;  //外设地址对齐方式
      hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;  //内存地址对齐方式，也就是地址自增时增加的大小
      hdma_adc1.Init.Mode = DMA_CIRCULAR;  //DMA循环接收
      hdma_adc1.Init.Priority = DMA_PRIORITY_LOW;  //DMA多个通道中的优先级
      if (HAL_DMA_Init(&hdma_adc1) != HAL_OK)
      {
        Error_Handler();
      }
  
      __HAL_LINKDMA(hadc,DMA_Handle,hdma_adc1); //将DMA与外设绑定
  
    /* USER CODE BEGIN ADC1_MspInit 1 */
  
    /* USER CODE END ADC1_MspInit 1 */
    }
    else if(hadc->Instance==ADC2)
    {
    /* USER CODE BEGIN ADC2_MspInit 0 */
  
    /* USER CODE END ADC2_MspInit 0 */
      /* Peripheral clock enable */
      HAL_RCC_ADC12_CLK_ENABLED++;
      if(HAL_RCC_ADC12_CLK_ENABLED==1){
        __HAL_RCC_ADC12_CLK_ENABLE();
      }
  
      __HAL_RCC_GPIOB_CLK_ENABLE();
      /**ADC2 GPIO Configuration
      PB15     ------> ADC2_IN15
      */
      GPIO_InitStruct.Pin = GPIO_PIN_15;
      GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
      GPIO_InitStruct.Pull = GPIO_NOPULL;
      HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
  
    /* USER CODE BEGIN ADC2_MspInit 1 */
  
    /* USER CODE END ADC2_MspInit 1 */
    }
  
  }
  
  ```
  
  

## 蓝桥杯使用建议

通常蓝桥杯的板子两个ADC都设置为*循环接收* + *DMA数据传输* 就可以了，同时记得关闭CubeMX中的强制DMA中断的选项，同时循环接收采集的周期可以设置得长一些
