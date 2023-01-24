蓝桥杯嵌入式stm32HAL库-HC573与LED

## 板子上的硬件

- 需要注意的是，板子上的8个LED的部分引脚是与LCD屏的引脚公用的，因此在8个LED的引脚输入led前过了一个锁存器，如下原理图所示：

  ![HC573](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191154412.png)
  
- 查阅手册可知

  - 当锁存脚(PD2) 拉低时，输出电平将不再变化

  - 当锁存脚(PD2) 拉高时，输出电平受输入的8个引脚控制。

    即，当PD2拉高时，相当于正常LED灯。手册真值表截图如下：

    ![image-20230119115717697](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191157930.png)

## 对应驱动代码的编写

有两种方案，任选其一即可。

### 方案一：

**思想**：**只要一直保持HC573是锁住的状态，LCD操作时就不会对LED产生影响,只有在设置LED灯状态时才开启锁存**。

所以需要注意以下三点：

- 在初始化时，应该**配置所有LED灯全灭(全为高)** 且 **HC573一直是锁住的状态（锁存脚GPIOD2为低）**，但这样的设置，在初始时，HC573的锁存脚是锁住的，也就是初始的LED状态不一定有写入，

  因此建议在**GPIO初始化之后，LCD初始化之前，先设置一下LED的状态（`LED_ctrl()`）**

- 每次操作LED灯时，避免不必要的闪烁,**先将LED状态置位，再拉高锁存(D2),再拉低锁存(恢复锁住状态)** ,锁存拉高拉低**实测**期间可以不需要延时。

  由HC573的手册可以得知，Latch需要保持的最短时间为20ns，而使用库函数操作的时间实测是足够的。(HC573手册截图如下：)

  ![image-20230119165428820](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191654407.png)

- ==因为只有在LED灯操作时才打开锁存，所以，读LED灯状态不一定与LED当前显示的状态一致==。

  解决这个问题可以在每次LCD屏操作前先保存LED灯的状态，但因为一直锁存，LCD屏对LED灯的影响不大，如果不用到LED灯的不用保存。

驱动实现如下：(对应上面三点)

>初始化LED(CubeMX操作)
>
>![image-20230119121118544](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301191211393.png)
>
>同时建议在IO初始化后LCD屏初始化前，先设置LED灯的状态，如下：
>
>```c
>led_ctrl(0xFF);
>LCD_Init();//在屏幕初始化前设置，并锁存，防止屏幕操作干扰
>```
>
>

>LED驱动代码(同时设置8个LED、读取8个LED两个函数，LED_Latch设置的一个宏)
>
>`led.h` 文件
>
>```c
>#ifndef __led_H__
>#define __led_H__
>
>uint8_t led_read(void);
>void led_ctrl(uint8_t led_state);
>#define LED_LATCH(x) HAL_GPIO_WritePin(GPIOD,GPIO_PIN_2,(GPIO_PinState)x)
>
>#endif
>```
>
>`led.c` 文件
>
>```c
>#include "main.h"
>#include "led.h"
>
>GPIO_TypeDef * led_port[8] = {GPIOC,GPIOC,GPIOC,GPIOC,GPIOC,GPIOC,GPIOC,GPIOC};
>uint16_t led_pin[8] = {GPIO_PIN_8,GPIO_PIN_9,GPIO_PIN_10,GPIO_PIN_11,GPIO_PIN_12,GPIO_PIN_13,GPIO_PIN_14,GPIO_PIN_15};
>
>/**
>* @brief : uint8_t led_read(void)
>* @param : 返回8个按键当前的状态
>* @attention : None
>* @author : Sheep
>* @date : 23/01/19
>*/
>uint8_t led_read(void)
>{
>    uint8_t led_state = 0;
>    uint8_t i = 0;
>    for(i = 0;i<8;i++)
>    {
>        if(HAL_GPIO_ReadPin(led_port[i],led_pin[i]))
>            led_state |= (1<<i);
>        else led_state &= ~(1<<i);
>    }
>    return led_state;
>}
>
>/**
>* @brief : void led_ctrl(uint8_t led_state)
>* @param : uint8_t led_state 按键状态，注意0亮1灭
>* @attention : 实测可以不用延时LED_Latch 
>            HC573中标准最少需要维持高电平20ns，实测不需要延时也足够
>* @author : Sheep
>* @date : 23/01/19
>*/
>void led_ctrl(uint8_t led_state)
>{
>    uint8_t i = 0;
>    for(i = 0;i<8;i++)
>    {
>        HAL_GPIO_WritePin(led_port[i],led_pin[i],(GPIO_PinState)( led_state & (1<<i) ));
>    }
>    LED_LATCH(1); //解锁
>    // HAL_Delay(1); //延时，实测可以不用
>    LED_LATCH(0); //锁住
>}
>
>```

>如果需要保存LED灯状态时，LCD屏操作时的基本模板
>
>```c
>uint8_t led_state = 0;
>led_state = led_read();
>//LCD屏的操作
>led_ctrl(led_state);
>```

### 方案二：

**因为只有LCD屏的操作会影响LED灯的状态**，因此，只需要在CubeMX中确定好LED灯的初始状态，并且**在每次LCD操作前，保存LED状态并锁住**；**在每次LCD操作后，恢复LED状态并解锁**。

与方案一的区别是，LED默认是不锁住的；并且`LED_ctrl()`函数只包含对LED引脚的操作，同时需要注意的一点时，==所有LCD屏的操作都需要加上LED状态保存并恢复这样的操作装饰，包括LCD屏的初始化==。

同样分作三点：

- 初始状态设置，LED的锁存为开(**GPIOD2为高,灯为默认状态（全灭、全高）**,方便`LED_Ctrl()`的设置)
- LED的驱动(**设置LED状态、读取LED状态两个函数，LED锁存控制的一个宏**)
- **LCD操作前，保存LED状态并锁住**；**在每次LCD操作后，恢复LED状态并解锁**

>初始化LED(CubeMX操作)
>
>![image-20230119123339568](C:\Users\20mxxiao\AppData\Roaming\Typora\typora-user-images\image-20230119123339568.png)
>
>LED驱动代码(同时设置8个LED、读取8个LED两个函数，LED_Latch设置的一个宏)
>
>`led.h`与方案一的led.h一致
>
>```c
>#ifndef __led_H__
>#define __led_H__
>
>uint8_t led_read(void);
>void led_ctrl(uint8_t led_state);
>#define LED_LATCH(x) HAL_GPIO_WritePin(GPIOD,GPIO_PIN_2,(GPIO_PinState)x)
>
>#endif
>```
>
>`led.c` 与方案一有些不同，没有开关锁存的操作
>
>```c
>#include "main.h"
>#include "led.h"
>GPIO_TypeDef * led_port[8] = {GPIOC,GPIOC,GPIOC,GPIOC,GPIOC,GPIOC,GPIOC,GPIOC};
>uint16_t led_pin[8] ={GPIO_PIN_8,GPIO_PIN_9,GPIO_PIN_10,GPIO_PIN_11,GPIO_PIN_12,GPIO_PIN_13,GPIO_PIN_14,GPIO_PIN_15};
>uint8_t led_read(void)
>{
>    uint8_t led_state = 0;
>    uint8_t i = 0;
>    for(i = 0;i<8;i++)
>    {
>        if(HAL_GPIO_ReadPin(led_port[i],led_pin[i]))
>            led_state |= (1<<i);
>        else led_state &= ~(1<<i);
>    }
>    return led_state;
>}
>
>void led_ctrl(uint8_t led_state)
>{
>    uint8_t i = 0;
>    for(i = 0;i<8;i++)
>    {
>        HAL_GPIO_WritePin(led_port[i],led_pin[i],(GPIO_PinState)( led_state & (1<<i) ));
>    }
>}
>```

>LCD屏操作模板：
>
>```c
>uint8_t led_state = LED_state(); //保存当前led状态
>LED_latch(0); //拉低锁存
>  //对应的LCD屏操作
>LED_ctrl(led_state); //恢复led状态
>LED_latch(1); //拉高锁存
>```

## 蓝桥杯使用建议

建议使用第一种，因为其LED_Latch在平常时是低(锁存)的状态，可以保护LED状态不受其他操作影响，而且可以不用在每次LCD屏的操作前后都保存LED灯的状态和恢复LED的状态。

但第一种使用时需要注意：**因为平时LED灯都是锁住的状态，所以读取得到的状态就不一定是LED灯显示的状态** 可以通过在LCD屏操作前，保存LED灯的状态解决。

实际上第二种也可以设计为平时锁存的状态，但在每次LED操作前就需要解锁。

