# 11蓝桥杯嵌入式stm32HAL库-DAC

## 板子上的硬件

- 板子上共有两个DAC，DAC1和DAC3，但能连接到IO扣输出的只有DAC1的两个通道，DAC3的两个通道只能连接到片内的外设。手册截图如下：

  ![image-20230303202050725](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202303032020143.png)

  从上图也可以看出，外部输出的引脚为 PA4、PA5和PA6。同时DAC1有两个通道，最高12bit分辨率。

## 具体使用

DAC输出的使用分为以下三类：

1. 输出**固定的直流电压**，即**数模转换直接输出**。
2. 输出**三角锯齿波**，利用DAC中自带的**逻辑控制器**，输出三角波。
3. 输出正弦波，实际上是通过**定时器触发DMA极短电平构成正弦波波形(查表法)**。

### 输出固定电压

- CUBEMX配置如下：

  注意：设置触发源 `Trigger` 为 `None` ，不是 `SoftWare Trigger` .

  ![image-20230303204815475](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202303032048719.png)

- 库函数使用

  >主要有两个函数 
  >
  >- `HAL_DAC_SetValue`  设置对应的电压值
  >-  `HAL_DAC_Start`  启用DAC输出，可以现在 `while(1)` 前使用，之后再根据输出需要调用 `HAL_DAC_SetValue` 改变DAC值。
  >
  >```c
  >HAL_DAC_Start(&hdac1,DAC_CHANNEL_1);//注意通道宏
  >HAL_DAC_SetValue(&hdac1,DAC_CHANNEL_1,DAC_ALIGN_12B_R,dac_val);
  >//设置dac的值，注意位对齐的宏
  >```

  输出的电压值的换算公式如下：(以 `12bitR` 12bit右对齐为例 )
  $$
  V_o = \frac{val_{set}}{4096}\times V_{ref} = \frac{val_{set}}{4096}\times 3.3
  $$

### 利用DAC的逻辑控制器输出三角波

#### CubeMX配置：

- DAC设置：

  注意：**触发源设置 `Trigger`  需为某个定时器** ,图中设置为 `tim2的触发输出事件`;

  这一项决定了**三角波的周期**，也只有选中这一项才会出现波形选择的选项。

  波形一般使用三角波，还有Noise波形的选项，不常用。

  需要设定**三角波的峰值**，这里设置的是12bit的最大值 `4095` 

  ![image-20230303211038130](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202303032110922.png)

- 对应定时器的设置：

  对应定时器中，**注意设置输出事件为那个中断事件** ，下图是设置tim2的溢出中断为定时器2的触发输出事件。

  ![image-20230303211841149](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202303032118342.png)

#### 库函数接口使用

>同样使用 `HAL_DAC_Start` 开启DAC输出。
>
>需要注意，由于使用TIM2做触发源，因此需要开启定时器2 `HAL_TIM_Base_Start(&htim2)` ，这样采用波形输出。
>
>可以不开启定时器2中断，因为只要有溢出标志做触发事件就够了，不需要中断也是可以的。
>
>可以通过 `HAL_DACEx_TriangleWaveGenerate(DAC_HandleTypeDef *hdac, uint32_t Channel, uint32_t Amplitude)` 这个库函数使能/失能DAC三角波的输出。

### 定时器触发+DMA查表法实现输出正弦波

不使用DAC自带的逻辑控制器，而是用触发信号来触发DMA传输，并在对应的DMA内存中设置好相应的电平值，这个内存(数组)便是波形函数的数值表。

#### cubeMX设置

- cubeMX的这是与上面三角波的设置类似，不过**波形模式需要失能**，同样的，也需要设置 **对应定时器的触发输出事件**

  ![image-20230303214351167](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202303032145952.png)

- 还需要**配置一个循环从内存发送到外设的DMA**

  ![image-20230303214845618](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202303032148666.png)

#### 代码编写

- 需要注意的是我们提供的数值表的长度决定了我们输出波形的平滑程度，即在 $2\pi$ 内的步长。

  我这里使用python生成这个数值表，不让单片机自己计算是为了节省单片机的运行时间。使用的公式如下：
  $$
  val_{i} = 2048+2048\times sin(\frac{2\pi}{len_{array}}\times i) \\
  其中，val_i表示数组的第i个值，len{array} 为数值长度
  $$
  生成sin数值表的pytho代码如下：

  ```python
  from math import *
  table_len = 64
  val_sin  = []
  for i in range(64):
      val_sin.append(2048 + round(2048*sin(2*pi/table_len*i)))
  print(val_sin)
  ```

  ```
  >[2048, 2249, 2448, 2643, 2832, 3013, 3186, 3347, 3496, 3631, 3751, 3854, 3940, 4008, 4057, 4086, 4096, 4086, 4057, 4008, 3940, 3854, 3751, 3631, 3496, 3347, 3186, 3013, 2832, 2643, 2448, 2249, 2048, 1847, 1648, 1453, 1264, 1083, 910, 749, 600, 465, 345, 242, 156, 88, 39, 10, 0, 10, 39, 88, 156, 242, 345, 465, 600, 749, 910, 1083, 1264, 1453, 1648, 1847]
  ```

- 代码编写

  ```c
  uint16_t sin_table[] = {2048, 2249, 2448, 2643, 2832, 3013, 3186, 3347,\
                          3496, 3631, 3751, 3854, 3940, 4008, 4057, 4086,\
                          4096, 4086, 4057, 4008, 3940, 3854, 3751, 3631,\
                          3496, 3347, 3186, 3013, 2832, 2643, 2448, 2249,\
                          2048, 1847, 1648, 1453, 1264, 1083, 910, 749,\
                          600, 465, 345, 242, 156, 88, 39, 10,\
                          0, 10, 39, 88, 156, 242, 345, 465,\
                          600, 749, 910, 1083, 1264, 1453, 1648, 1847};
  HAL_TIM_Base_Start(&htim2);
  HAL_DAC_Start_DMA(&hdac1,DAC_CHANNEL_1,(uint32_t*)sin_table,sizeof(sin_table)/2,DAC_ALIGN_12B_R);
  
  ```

  >使用 `HAL_DAC_Start_DMA` 设置数值表作为DAC DMA传输的发送缓冲，需要注意`sizeof` 计算出的数值是 以字节为单位，而数值表每个元素的大小为2个字节，因此计算时需要除以2.
  >
  >**同时注意需要开启定时器**，这样才会有触发波形输出。

