# 蓝桥杯嵌入式stm32Hal库使用-按键使用

## 板子上的硬件

![image-20230120221257144](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301202229833.png)

可以看出连接的四个引脚是 `B0、B1、B2、A0` 分别对应板子上的按键1-4。需要注意的是**按下时是低电平** 

## 驱动代码的编写

按键的代码主要有两个要求，`一是实现消抖` `二是实现长按(按着) 与 短按 (按下的区别)` 。

按键这里没有使用外部中断，而是设置为一般的GPIO输入模式，通过前后台设置20ms(10ms)的轮询扫描。

### 按键消抖

消抖的实现只需要间隔20ms毫秒连续检测到低电平，即可认为按键是按下的，且此时20ms的间隔便是消抖的时间，**注意，不要使用`delay`，而是使用诸如之间前后台轮询中定时检查即可。**

因此，我们所编写的驱动的函数，只需要**连续检测两次低电平置1(表明是按下状态)，连续检测到高电平置0，表示松开状态**。 需要注意，**连续两次低电平对相应的位置1，与电平是相反的。**

如下图可以表明间隔20ms的连续两次间隔的消抖原理：

![image-20230120222854802](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301202228838.png)

主要需要实现两个函数(`CubeMX`的设置比较简单，不赘述)

>`key.c` 文件，**按键扫描函数**，需要**每隔20ms(或10ms)调用一次**
>
>```c
>uint8_t key_state = 0 ;
>GPIO_TypeDef * key_port[4] = {GPIOB,GPIOB,GPIOB,GPIOA};
>uint16_t key_pin[4] = {GPIO_PIN_0,GPIO_PIN_1,GPIO_PIN_2,GPIO_PIN_0};
>uint8_t key_scan(void)
>{
>    uint8_t i = 0;
>    for(i = 0;i < 4; i++)
>    {
>        if(HAL_GPIO_ReadPin(key_port[i],key_pin[i]) == GPIO_PIN_RESET)
>        {//检测到低
>            if(key_state & (1<<(i+4)) ) //第一次已经检测过低
>                key_state |= (1<<i);
>            else key_state |= (1<<(i+4)); //第一次低电平
>        }
>        else if(HAL_GPIO_ReadPin(key_port[i],key_pin[i]) == GPIO_PIN_SET)
>        {
>            if((key_state & (1<<(i+4))) == 0) 
>                //第一次已经检测过高,注意&的优先级没有==高
>                key_state &= (~(1<<i));
>            else key_state &= (~(1<<(i+4))); //第一次高电平
>        }
>    }
>    return key_state;
>}
>
>```
>
>还需要一个**同时还需要有一个按键读取函数，用于在其他地方异步读取结果，因为通常读取按键状态和按键检测不是同时完成的，有利于获取长按和短按状态和进行其他操作，如100ms(或者200ms调用一次),并根据结果编写对应的操作**
>
>以下是该函数在 `key.c` 文件中的定义
>
>```c
>uint8_t key_read(void)
>{
>    return key_state;
>}
>```

### 按下与按着

实际上上面检测出来的按键状态就是按键是否按着，因为其在按下时是1，松开时是0，所以按着就是1。如果使用一个变量在等于1时递增的话，就是长按递增。

我们使用上述按键消抖后的波形来分析按下与按着的不同

因此我们只需要在**检测本次按着与否**和**上次保存的电平状态是否是按着的**即可。示意图如下：

![image-20230120223729804](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301202237040.png)

而且对于按下的检测，通常放在需要检测按下的任务里，因为识别到短按后，需要立即进行对应的操作(加减某个变量等)，并及时将按下的标志清零，否则没有计数的话，多次按下只会记为一次按下。所以这些代码通常是在主函数中实现，不在按键的驱动中。

>`main.c` 实现按键检测按下与按着的模板如下：需要使用前后台轮询调用该模板
>
>```c
>else if(task_flag[2])//100ms
>{
>    last_key_sta = key_sta; //保存上一次的状态
>    key_sta = key_read(); //获取当前状态
>    if((key_state & (1<<0)) && (!(key_last_state & (1<<0))))
>        {printf("press\r\n");}
>    //先高后低为一次下降沿，视为一次按下
>    else if((key_state & (1<<0)) && (key_last_state & (1<<0)))
>          { printf("long hold\r\n");}
>    //连续低电平视为按着，视为一直按着
>    else if((!(key_state & (1<<0))) && (key_last_state & (1<<0))
>            { printf("release\r\n");}
>    //先低后高一次上升沿，视为一次松开
>}
>```
>
>**注意`按下、按着` 和 `短按、长按` 的区别：**
>
>- **按着触发开始前一定会触发一次按下操作**
>- **按着触发结束时一定会触发一次松开操作**
>
>但如果按下与按着的操作是一致时，可以使用`按下、按着` 代替`短按、长按` 如 **按下按键+1，长按(按着)一直加的操作**

### 短按与长按

如果`短按、长按`的操作不同时，我们也就是说，在触发长按操作前不能触发短按的操作，这时，**我们就需要计数变量，来记录下按着状态持续的时间，以此来区别短按与长按** 

需要注意的时，**识别长按、短按的操作是在按键定时扫描的任务中完成的，因此如果有长耗时的操作(rom读取、LCD屏显示等)，就需要借助标志位来实现**

短按与长按识别示意图片如下：(使用消抖后的波形)

![image-20230121163200379](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301211633147.png)

>`main.c` 长按和短按识别的模板,注意这里保存按键扫描返回值的变量不能和按下和长按识别的变量重名，**该模板需要在10ms间隔的按键扫描中调用。**
>
>注意：==**短按的触发操作在高电平时检测，因为只有松手时，才知道低电平的时间是否符合短按的时长**==
>
>==**长按的触发操作每次循环都要检测，因为长按时间段内有保持的操作，否则，只有在松开那一刻才有长按触发**==
>
>```c
>#define KEY_SHORT_LOW_DLIMIT 15 
>//短按触发时间下限，150ms，需要下限，否则每次清零进入高电平都会触发短按
>#define KEY_SHORT_LOW_ULIMIT 50 //短按触发时间上限500ms,也可以是700ms
>#define KEY_LONG_LOW_DLIMIT 70 //长按触发时间下限，超过该时间视为长按
>uint8_t key_sta = 0;
>uint32_t key_down_count = 0; 
>//尽量使用uint32_t，避免太长时间没有清零溢出，如长按时间过久，可以使用标志位解决这个问题
>if(task_flag[0])//10ms
>{
>  key_sta = key_scan();
>  if(key_sta & 0x01) //低电平时
>  {
>      key_down_count ++; //低电平时一直加
>  }
>  else //高电平时
>  {
>    if(key_down_count>KEY_SHORT_LOW_DLIMIT && key_down_count < KEY_SHORT_LOW_ULIMIT) 
>        //松开时低电平时间250ms<x<500ms视为短按
>        printf("short\r\n");
>    key_down_count = 0; //低电平时间清零
>  }
>  if(key_down_count == KEY_LONG_LOW_DLIMIT) printf("long_press state\r\n"); 
>    // == 700ms触发长按时刻
>  if(key_down_count > KEY_LONG_LOW_DLIMIT) printf("long_press hold\r\n"); 			//>700ms 长按保持时触发
>  task_flag[0] = 0;
>}
>```

### (拓展)单击与双击的识别

这里的单击也就是上面的短按，要识别，双击，**我们需要增加一个变量用于高电平时间的计时，并且，只有在前面低电平时间较短的情况下开始高电平的计时**

注意以下三点：

- 我们这里在进入低电平时对高电平时间清零(在低电平操作结束后清零)；

- 进入高电平时对低电平时间清零(在高电平操作结束后清零)；
- 为了防止高电平计时只计时一次(因为高电平计时开启与否 与 低电平时间相关)，而进入高电平就会将低电平时间清零，所以只会进入一次，这里我们将**高电平计时变量复用为标志位，也就是说，进入高电平后判断到前面的低电平时间符合双击的要求，便将高电平计时变量++，同时，在高电平期间内，判断到计时变量不为0，就计时(自增)。** 而不是在判断低电平时间要求内自增。

识别单击与双击原理示意图如下：

![image-20230121163404126](https://sheep-photo.oss-cn-shenzhen.aliyuncs.com/img/202301211634247.png)

>`mian.c` 短按、长按、双击的识别模板；
>
>```c
>#define KEY_SHORT_LOW_DLIMIT 15 
>//短按触发时间下限，150ms，需要下限，否则每次清零进入高电平都会触发短按
>#define KEY_SHORT_LOW_ULIMIT 50 //短按触发时间上限500ms,也可以是700ms
>#define KEY_LONG_LOW_DLIMIT 70 //长按触发时间下限，超过该时间视为长按
>#define KEY_DOUBLE_1LOW_DLIMIT 5 //双击第一次按下的低电平持续时间下限50ms
>#define KEY_DOUBLE_1LOW_ULIMIT 15 //双击第一次按下的低电平持续时间上限150ms
>#define KEY_DOUBLE_1HIGH_DLIMIT 5 //双击第一次松开后高电平持续时间下限50ms
>#define KEY_DOUBLE_1HIGH_ULIMIT 15 //双击第一次松开后高电平持续时间上限150ms
>
>
>uint8_t key_sta = 0; //按键状态储存
>uint32_t key_down_count = 0,key_up_count = 0; 
>//低\高电平持续时间计数变量 (单位扫描时间间隔10ms)
>
>
>/*while(1) 10ms一次的前后台任务中*/
>if(task_flag[0])//10ms
>{
>  key_sta = key_scan(); //按键扫描
>  if(key_sta & 0x01) //低电平时
>  {
>    if(key_up_count > KEY_DOUBLE_1HIGH_DLIMIT && \
>       key_up_count < KEY_DOUBLE_1HIGH_ULIMIT) 
>          //满足双击高电平时间长后的第二次按下时触发操作
>          printf("double press\r\n");    
>      key_down_count ++;
>      key_up_count = 0; //低电平操作结束，高电平时间清零
>  }
>  else //高电平时
>  {
>    if(  key_down_count > KEY_DOUBLE_1LOW_DLIMIT \
>       && key_down_count<KEY_DOUBLE_1LOW_ULIMIT )
>    	{key_up_count ++; } //当成标志位，置为1
>    if(key_up_count != 0)key_up_count ++; 
>      //满足低电平达到双击的第一次低电平时间长度才计时高电平时间，置为1后会持续计时
>      //不使用这样复用标志位的话，
>      //在第一次高电平后低电平时间被清零，高电平持续时间内便没有持续计时
>    if(key_down_count>KEY_SHORT_LOW_DLIMIT && \
>       key_down_count < KEY_SHORT_LOW_ULIMIT) 
>        //松开时低电平时间150ms<x<500ms视为短按
>        printf("short\r\n");
>    key_down_count = 0; //高电平操作结束，低电平时间清零
>  }
>    
>  if(key_down_count == KEY_LONG_LOW_DLIMIT) printf("long_press state\r\n"); 
>    // == 700ms触发长按时刻
>  if(key_down_count > KEY_LONG_LOW_DLIMIT) printf("long_press hold\r\n"); 	  //>700ms 长按保持时持续触发
>  task_flag[0] = 0;
>}
>```
>
>需要注意：**==双击操作的触发是在低电平(前方低电平时间和高电平时间均满足时的一次下降沿) 时触发。==** 因此上面的代码，在快速单击后长按也会触发一次双击。
>
>**同样的因为与按键扫描是在同一个定时任务中运行，对应操作不能有长耗时操作(ROM读写如:at24c02,LCD屏操作等)**

## 蓝桥杯建议

无论是`按下、按着` 、 `短按、长按` 还是 `单击、双击` 的识别，都需要**实现10ms一次的按键扫描函数** 和**按键读取函数** 。

同时需要掌握 `按下、按着、释放` `短按、长按` 的识别模板，注意由于**短按长按识别与按键扫描在同个定时任务中，因此，不可以有长耗时操作。**