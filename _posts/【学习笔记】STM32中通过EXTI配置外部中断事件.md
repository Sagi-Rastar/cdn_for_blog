---
title: 【学习笔记】STM32中通过EXTI配置外部中断事件
date: 2023-11-07 11:04:45
tags:
- stm32
- 嵌入式
---
# 【学习笔记】STM32中通过EXTI中断源配置中断事件

## 中断事件

一个中断事件中一般有很多的触发源，一般将其称作“中断源”。主程序在运行的过程中，一旦出现了中断源，其便会让CPU停止当前正在运行的程序，转而进行中断事件指定的程序，执行完所有的“中断程序”之后，便会返回原本主程序的断点处继续运行。  

EXTI (External interrupt/event controller) 是众多中断源之中的一个，一般译为“外部中断”。外部中断有很多的输入通道，一般来说有20个，其中GPIO占了16个，其余4个为不常用的PVD（电源电压监测）输出、RTC闹钟、USB唤醒、ETH以太网唤醒。

由于硬件与地址上的动态性（暂不展开），因此需要NVIC (Nested Vectored Interrupt Controller) 执行管理中断源与分配优先级两个工作，一般译为“嵌套中断向量控制器”。其中，中断源向NVIC输出的通道一般译为“中断通道”，需要注意的是一个中断源可能会占用多个中断通道，事实上中断源一般都占用多个中断通道。

本文将主要记录外部中断中GPIO的原理与配置过程，作为学习笔记记录。

## EXTI工作原理

![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/rect234.png)

EXTI作为检测外部中断的中断源，对GPIO进行监测是其最主要的功能。EXTI在对GPIO的检测上一般有16个输入通道（注意这里是EXTI的输入通道而不是“中断通道”），但是GPIO每一个port口都有0-15总计16个pin，如果要检测所有的GPIO明显16个输入通道不够用。  

于是在STM32上需要用到AFIO这个“选择器”。具体的工作原理就是让相同的Pin口对应到同一条输入通道，例如`PX0 => EXTI0`,`PX1 =>EXTI1`……以此类推，换句话说每个输入通道同时只能监测一个port口，这也就是为什么相同的pin口不能同时触发中断的原因。

解决了监测所有GPIO口的问题之后，所有的输入信号便可以通过EXTI被处理。在EXTI中做的处理通俗来讲就是通过一些寄存器，配置触发方式以及选择响应是“中断”还是“事件”。

在STM32中EXTI提供了一个新的响应，“事件”相对于软件上进入NVIC的“中断”而言有所不同，“事件”响应是硬件上的响应，事件响应是输出一个脉冲信号，以便用于与其他外设相互联动，例如触发ADC或DMA之类的。

> 具体工作框图可以参考：
![EXTI系统框图](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230105143832.png)

如上图所示，EXTI的输入通道经过“边沿检测”电路，此电路由两个寄存器控制，用于配置EXTI是`上升沿触发`还是`下降沿触发`或是`双边沿触发`。随后信号经过一个或门，此处或门的作用在于可以通过软件中断寄存器决定是否直接由软件触发中断。当或门的信号输出之后，分成`中断响应`和`事件响应`，两种响应都可以通过利用与门和“xx屏蔽寄存器”选择放行与否。

其中关于ST公司对EXTI中断通道的精简有必要提一嘴（……）

ST公司为了节省NVIC的中断资源，将EXTI的5-9通道合并为`EXTI9_5`，将10-15合并为`EXTI15_10`，因此例程中对于pin=13的这个GPIO口中断源选择为`EXTI15_10`。

```c
#define KEY2_INT_EXTI_IRQ                 EXTI15_10_IRQn
#define KEY2_IRQHandler                   EXTI15_10_IRQHandler
```

## EXTI具体配置

>本节例程参考野火的<STM32 HAL库开发实战指南——F103系列>。

本次使用的开发板是`野火的F103-MINI 开发板`，下面是本次例程所涉及的硬件原理图：

![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230111151132.png)

---

下面是中断编程要点：

- **使能外设某个中断，由每个外设的相关中断使能位控制。例如串口发送完成与接收完成中断，这两个中断都由串口控制寄存器的相关中断使能位控制。**

    ---

    按照自己目前的理解，这一步应该是说`​HAL_NVIC_EnableIRQ(IRQn)`这一个函数。具体做的事情就是使能中断源呗。
    > ？这个部分还不是太懂，GPIO的寄存器里有专门的“相关中断使能位”吗

- **配置EXTI中断源、配置中断优先级。**

    ```c
    HAL_NVIC_SetPriority(IRQn, PreemptPriority, SubPriority)
    ```

    **IRQn：** 设置中断源。不同中断的中断源不可写错，写错了程序也不会报错，只会不响应。具体成员配置参考`stm32f103xe.h`里面的`IRQn_Type`结构体定义。

    **PreemptionPriority：** 抢占优先级。具体的值自己定。

    **SubPriority：** 子优先级。具体的值自己定。

    ---

    例程：

    >bsp_exit.h 
    ```c++
    //GPIO口定义
    #define KEY1_INT_GPIO_PORT                GPIOA
    #define KEY1_INT_GPIO_CLK_ENABLE()        __HAL_RCC_GPIOA_CLK_ENABLE();
    #define KEY1_INT_GPIO_PIN                 GPIO_PIN_0
    #define KEY1_INT_EXTI_IRQ                 EXTI0_IRQn
    #define KEY1_IRQHandler                   EXTI0_IRQHandler

    #define KEY2_INT_GPIO_PORT                GPIOC
    #define KEY2_INT_GPIO_CLK_ENABLE()        __HAL_RCC_GPIOC_CLK_ENABLE();
    #define KEY2_INT_GPIO_PIN                 GPIO_PIN_13
    #define KEY2_INT_EXTI_IRQ                 EXTI15_10_IRQn
    #define KEY2_IRQHandler                   EXTI15_10_IRQHandler    
    ```

    >bsp_exit.c
    ```c++
    void EXTI_Key_Config(void)
    {
        GPIO_InitTypeDef GPIO_InitStructure; 

        /*开启按键GPIO口的时钟*/
        KEY1_INT_GPIO_CLK_ENABLE();
        KEY2_INT_GPIO_CLK_ENABLE();

        /* 选择按键1的引脚 */ 
        GPIO_InitStructure.Pin = KEY1_INT_GPIO_PIN;
        /* 设置引脚为输入模式 */ 
        GPIO_InitStructure.Mode = GPIO_MODE_IT_RISING;	    		
        /* 设置引脚不上拉也不下拉 */
        GPIO_InitStructure.Pull = GPIO_NOPULL;
        /* 使用上面的结构体初始化按键 */
        HAL_GPIO_Init(KEY1_INT_GPIO_PORT, &GPIO_InitStructure); 
        /* 配置 EXTI 中断源 到key1 引脚、配置中断优先级*/
        HAL_NVIC_SetPriority(KEY1_INT_EXTI_IRQ, 0, 0);
        /* 使能中断 */
        HAL_NVIC_EnableIRQ(KEY1_INT_EXTI_IRQ);

        /* 选择按键2的引脚 */ 
        GPIO_InitStructure.Pin = KEY2_INT_GPIO_PIN;  
        /* 其他配置与上面相同 */
        HAL_GPIO_Init(KEY2_INT_GPIO_PORT, &GPIO_InitStructure);      
        /* 配置 EXTI 中断源 到key2 引脚、配置中断优先级*/
        HAL_NVIC_SetPriority(KEY2_INT_EXTI_IRQ, 0, 0);
        /* 使能中断 */
        HAL_NVIC_EnableIRQ(KEY2_INT_EXTI_IRQ);
    }
    ```
    
    GPIO的配置见其他博客。  
    这里主要是调用`HAL_NVIC_SetPriority(IRQn, PreemptPriority, SubPriority)`和`​HAL_NVIC_EnableIRQ(IRQn)`这两个函数，第一个设置中断优先级，第二个使能该中断。

- **编写中断服务函数。**

    所有的中断服务函数都写在`stm32f1xx_it.c`里面，并且函数名要注意不要写错。
    
    ---

    例程：
   
    >stm32f1xx_it.c
    ```c++
    void KEY1_IRQHandler(void)
    {
    //确保是否产生了EXTI Line中断
        if(__HAL_GPIO_EXTI_GET_IT(KEY1_INT_GPIO_PIN) != RESET) 
        {
            // LED1 取反		
            LED1_TOGGLE;
        //清除中断标志位
            __HAL_GPIO_EXTI_CLEAR_IT(KEY1_INT_GPIO_PIN);     
        }  
    }

    void KEY2_IRQHandler(void)
    {
    //确保是否产生了EXTI Line中断
        if(__HAL_GPIO_EXTI_GET_IT(KEY2_INT_GPIO_PIN) != RESET) 
        {
            // LED2 取反		
            LED2_TOGGLE;
        //清除中断标志位
            __HAL_GPIO_EXTI_CLEAR_IT(KEY2_INT_GPIO_PIN);     
        }  
    }
    ```
    
    >main.c
    ```c++
    int main(void)
    {
        /* 系统时钟初始化成72 MHz */
        SystemClock_Config();

        /* LED 端口初始化 */
        LED_GPIO_Config();

        /* 
        初始化EXTI中断，按下按键会触发中断，
        触发中断会进入stm32f4xx_it.c文件中的函数
        KEY1_IRQHandler和KEY2_IRQHandler，处理中断，反转LED灯。
        */
        EXTI_Key_Config(); 

        /* 等待中断，由于使用中断方式，CPU不用轮询按键 */
        while(1)l{}
    }
    ```

    这里的中断服务函数里面用到了两个函数：  
    `__HAL_GPIO_EXTI_GET_IT`：用来检查指定的EXTI线是否有产生中断（Checks whether the specified EXTI line is asserted or not.）  
    `__HAL_GPIO_EXTI_CLEAR_IT`：清除指定EXTI线的中断标志位（Clears the EXTI's line pending bits.）
    >留一个疑问，查HAL库的时候不太理解FLAG和IT这两个东西的具体区别。

## 总结

中断事件配置的过程大概就是这样，抓住上面三个要点就差不多。有的地方比较绕还是有一点翻译上的原因，比如EXTI线具体指什么（我的理解就是EXTI这个中断源向NVIC走的中断通道）之类的。

例程里面没有太过多涉及EXTI相关的一些寄存器的配置，这点还需要注意。

