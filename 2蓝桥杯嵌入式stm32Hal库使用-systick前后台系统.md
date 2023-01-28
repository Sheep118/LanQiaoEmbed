# 蓝桥杯嵌入式stm32Hal库使用-systick前后台系统

## 板子上的硬件

- systick是一个24bit的向下计数器，stm32G4使用HAL库默认配置时钟为1ms进入一次中断。 `__IO uint32_t uwTick;` 是HAL库中用于systick中用于自增计数的变量，用于实现1ms左右的延时。由于其是32位的变量1ms自增1，大概需要49天左右才会溢出。

- 因此，可以使用如下的方式，临时进行前后台系统的构建，但不建议，因为一旦systick溢出，前后台系统的秩序将完全混乱。

  ```c
  uint32_t led_point = 0; //注意需要使用32位的变量
  void led_task()
  {
      if(uwTick - led_point < 1000)return; //1s才能进入一次
      led_point = uwTick;
  }
  
  void main()
  {
      led_point = uwtick;//给初值
      while(1)
      {
          led_task();
      }
  }
  ```

## 前后台系统的基本代码

### 前后台系统说明

- 前后台系统是利用
  - systick中对变量进行计数，并且置相应的标志位，称为后台
  - `mian` 函数中，在`while(1)` 中对标志位进行检测，以达到定时运行不同的任务的目的。

### 基本代码实现

- 首先是后台，在 `stm32g4xx_it.c` 文件中实现 `SysTick_Handler` 中断 函数对任务计数变量 `task_tick` 的自增 和 `task_flag` 标志位的置位。

  >`stm32g4xx_it.c` 文件，`SysTick_Handler` 函数
  >
  >```c
  >uint16_t task_tick[TASK_MAX_SIZE] = {0}; //任务计数数组
  >uint16_t task_limit[TASK_MAX_SIZE] = {10,50,100};
  >//任务定时运行的时间间隔
  >uint8_t task_flag[TASK_MAX_SIZE] = {0};
  >//任务标志位
  >void SysTick_Handler(void)
  >{
  >  /* USER CODE BEGIN SysTick_IRQn 0 */
  >    uint8_t i = 0;
  >  /* USER CODE END SysTick_IRQn 0 */
  >  HAL_IncTick();
  >  /* USER CODE BEGIN SysTick_IRQn 1 */
  >    for (i = 0; i < TASK_MAX_SIZE ; i++)
  >    {
  >        if(task_flag[i] == 0) //任务还在定时计数
  >        {
  >            if(task_tick[i] < task_limit[i])
  >                task_tick[i]++;  //时间间隔不满足标准，继续自增
  >            else
  >            {
  >                task_flag[i] = 1; 
  >                //时间间隔运行结束，标志可以开始任务函数的运行
  >                task_tick[i] = 0; //清零，准备下一次计时
  >            }
  >        }    
  >    }
  >  /* USER CODE END SysTick_IRQn 1 */
  >}
  >```
  >
  >其中，任务总数宏 `TASK_MAX_SIZE`  和 任务标志位数组`task_flag` 的声明在`main.h` 中
  >
  >```c
  >/* USER CODE BEGIN EM */
  >#define TASK_MAX_SIZE 3
  >/* USER CODE END EM */
  >/* USER CODE BEGIN Private defines */
  >extern uint8_t task_flag[TASK_MAX_SIZE];
  >/* USER CODE END Private defines */
  >```

- 在`mian.c` 文件的 实现对应任务即可。

  任务模板如下：(分三步)

  - 检测对应的标志位
  - 任务主体函数
  - **任务执行结束，清楚标志位，重新开始时间间隔计时**

  >`mian.c` 以按键检测任务函数为例(上面的时间间隔为10ms一次)
  >
  >```c
  >uint8_t key_state = 0;
  >uint16_t key_count[4] = {0};
  >//需要在不同函数之间使用的变量，建议使用全局变量，
  >//如果只是自己的函数使用，建议使用静态变量，避免进入函数时被初始化没有保留之前的值
  >void key_task(void) //10ms
  >{
  >    if(task_flag[0])
  >    {
  >        /*********对应的任务主体操作
  >        key_state = key_scan();
  >			******/
  >        task_flag[0] = 0; //清楚标志位，重新开始时间间隔计时
  >    }
  >}
  >```
  >
  >需要注意的是，**记得在 `main.c` 的主循环中调用该函数**
  >
  >```c
  >  while (1)
  >  {
  >    /* USER CODE END WHILE */
  >    /* USER CODE BEGIN 3 */
  >      key_task();
  >      uartrx_task();
  >      LCD_task();
  > 	 //注意需要写在这两段用户注释3中，重新配置cubemx时才不会被清理掉
  >  }
  >  /* USER CODE END 3 */
  >```

## 蓝桥杯建议

- 使用三个数组构造前后台系统的方式必须掌握。而且建议使用多个数组的方式进行构造，使用结构体数组可能会比较复杂，在比赛时比较浪费时间。