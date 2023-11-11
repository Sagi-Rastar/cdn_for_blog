---
title: 【学习笔记】STM32中的GPIO框图解析与软件配置
date: 2023-11-07 10:57:34
tags: 
- stm32
- 嵌入式
---

# 【学习笔记】STM32中的GPIO框图解析与软件配置

## GPIO简介

GPIO全称为General-purpose I/O ports，一般译为“通用输入输出端口”，是STM32单片机中最基础的外设之一。
![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230106182513.png)  
>STM32芯片架构简图 参考<STM32库开发实战指南——基于野火MINI开发板>

GPIO有输出输入两种功能，可以实现与外部通讯、控制外部设备以及数据采集等功能。GPIO分为多组port，每一组port下一般有16个pin。
>GPIOA12，为A组第12号引脚。（关于port和pin的叫法属于个人习惯）

`输出功能`可以实现控制引脚输出高低电平，实现开关控制，也可以接入继电器# 【学习笔记】STM32中的GPIO框图解析与软件配置

## GPIO简介

GPIO全称为General-purpose I/O ports，一般译为“通用输入输出端口”，是STM32单片机中最基础的外设之一。
![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230106182513.png)  
>STM32芯片架构简图 参考<STM32库开发实战指南——基于野火MINI开发板>

GPIO有输出输入两种功能，可以实现与外部通讯、控制外部设备以及数据采集等功能。GPIO分为多组port，每一组port下一般有16个pin。
>GPIOA12，为A组第12号引脚。（关于port和pin的叫法属于个人习惯）

`输出功能`可以实现控制引脚输出高低电平，实现开关控制，也可以接入继电器与三极管，实现控制大功率电路通断。

`输入功能`可以实现检测外部电平、读取引脚状态、采集模拟信号等功能。

下面对GPIO基本框图进行分析，了解GPIO主要的工作原理。
>过一遍理解就可，有个印象

## GPIO基本框图
![GPIO基本框图](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230106151230.png)
虽然标注的有点花花绿绿（）但是大致上可以分为三个部分：

- 紫色为引脚与外部设备连接的端口。
- 以蓝色箭头为主线的一串冷色相关的，为输入模式相关的电路。
- 以红色箭头为主线的一串暖色相关的，为输出模式相关的电路。

---

首先是紫色的`保护二极管与上下拉电阻`。

引脚电压高于 $V_{DD}$ ，上方二极管导通。  
引脚电压低于 $V_{SS}$ ，下方二极管导通。

这样就可以防止不正常的电压引入芯片导致芯片烧毁。但要注意，纵使有这样的保护也不能直接外接大功率驱动器件。

---

其次是红色的`输出模式`这一条线，从左往右看。

由`输出数据寄存器GPIOx_ODR`开始，单片机可以对该寄存器修改值，从而给输出控制一个信号。  
除此之外对`置位/复位寄存器 GPIOx_BSRR`的值进行修改也可以影响”输出数据寄存器“的值。

由这两个寄存器给出信号之后，通过一个开关接入输出控制。该开关可以控制是由”输出数据寄存器“还是由其他片上外设作为”输出源“。
>例如要使用串口的时候，需要一个通讯发送引脚的时候，就可以将某个GPIO引脚配置为串口复用功能。

最关键的是由P-MOS与N-MOS两个管子组成的单元电路。
这个单元电路使得GPIO的输出模式有两个模式，分别是`推挽输出`和`开漏输出`。

`推挽输出`中  
输出源给高电平，经过反向，上方的P-MOS导通，下方的N-MOS关闭，对外输出高电平；  
输出源给低电平，经过反向，上方N-MOS管关闭，上方P-MOS导通，对外输出低电平；

`开漏输出`中，上方的P_MOS不工作  
当N-MOS导通的时候对外输出为低电平（接地）；  
当N-MOS关闭的时候对外为高阻态，正常使用的时候需要接上拉电阻；此时等效于开漏电路，具有“线与”特性，一般应用在I2C、SMBUS通讯等需要“线与”功能的总线电路中，或者需要输出电平不匹配的时候。
>比如需要输出 5 伏的高电平，就可以在外部接一个上拉电阻，上拉电源为 5 伏， 并且把GPIO设置为开漏模式，当输出高阻态时，由上拉电阻和电源向外输出 5 伏的电平。

---

最后是蓝色的`输入模式`这一条线，从右往左看。

通过引脚采集电压之后，首先通过上/下拉电阻，配置成上/下拉输入，随后接一个TTL肖特基触发器，进行模数转换。

最后将触发器给出的数字信号存储在“输入数据寄存器GPIOx_IDR”中，通过读取该寄存器就可以了解GPIO引脚的电平状态。

输入模式中的复用功能输入与复用功能输出类似，也只是不经过“输入数据寄存器”，直接连接其他的片上外设，让别的外设读取引脚状态。
>例如使用串口通讯，需要一个通讯接收引脚的时候，就可以将某个GPIO引脚配置为串口复用功能。此处与输出模式的复用功能输入所举的例子对应。

模拟输入与输出的通道是当引脚需要做ADC/DAC的时候，需要用原始信号时所提供的通道。  
`ADC`时需要采集原始的模拟信号，因此不经过触发器。  
`DAC`时需要输出原始的模拟信号，因此不经过双MOS管的结构。
>A for analog, D for digital. ADC-模数转换，DAC数模转换。

---

至此基本将GPIO的原理框图进行了一遍梳理，下面为在HAL库中如何配置GPIO的一些记录。

##  HAL库中GPIO的配置

>本节例程参考野火的<STM32 HAL库开发实战指南——F103系列>使用固件库点亮LED一章

首先明确一下软件上的层级关系：

LED灯的控制相关的代码不属于STM32HAL库的内容，一般将这样的控制代码封装在另外的文件中。本次例程中将文件名命名为`bsp_led.c`与`bsp_led.h`，这里的bsp是`Board Support Packet`的简写，即板级支持包。

>对bsp的个人理解：不同外设的控制代码。

本次使用的开发板是`野火的F103-MINI 开发板`，下面是本次例程所涉及的硬件原理图：

![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230111150900.png)

如图所示容易看出，若将这两个GPIO引脚设置为低电平即可点亮LED。

本次例程中关于引脚定义的宏：

>bsp_led.h
```c
//引脚定义
/*******************************************************/

#define LED1_PIN                  GPIO_PIN_2               
#define LED1_GPIO_PORT            GPIOC                    
#define LED1_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()


#define LED2_PIN                  GPIO_PIN_3              
#define LED2_GPIO_PORT            GPIOC                      
#define LED2_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()

/************************************************************/
```

```c
/** 控制LED灯亮灭的宏，
	* LED低电平亮，设置ON=0，OFF=1
	* 若LED高电平亮，把宏设置成ON=1 ，OFF=0 即可
	*/
#define ON  GPIO_PIN_RESET
#define OFF GPIO_PIN_SET

/* 带参宏，可以像内联函数一样使用 */
#define LED1(a)	HAL_GPIO_WritePin(LED1_GPIO_PORT,LED1_PIN,a)


#define LED2(a)	HAL_GPIO_WritePin(LED2_GPIO_PORT,LED2_PIN,a)


/* 直接操作寄存器的方法控制IO */
#define	digitalHi(p,i)			{p->BSRR=i;}			              //设置为高电平
#define digitalLo(p,i)			{p->BSRR=(uint32_t)i << 16;}		 //输出低电平
#define digitalToggle(p,i)		{p->ODR ^=i;}		            	//输出反转状态


/* 定义控制IO的宏 */
#define LED1_TOGGLE		digitalToggle(LED1_GPIO_PORT,LED1_PIN)
#define LED1_OFF		digitalHi(LED1_GPIO_PORT,LED1_PIN)
#define LED1_ON			digitalLo(LED1_GPIO_PORT,LED1_PIN)

#define LED2_TOGGLE		digitalToggle(LED2_GPIO_PORT,LED2_PIN)
#define LED2_OFF		digitalHi(LED2_GPIO_PORT,LED2_PIN)
#define LED2_ON			digitalLo(LED2_GPIO_PORT,LED2_PIN)


					
//黑(全部关闭)
#define LED_RGBOFF	\
		LED1_OFF;\
		LED2_OFF
```

此处控制LED亮灭的操作是直接向`BSRR`写入控制指令来实现的，对`BSRR`低 16 位 写 1 输出高电平，对`BSRR`高 16 位写 1 输出低电平。同时，对`ODR`某位进行异或操作可反转位的状态。

>`BSRR`和`ODR`在前文详解GPIO框图的输出模式时候提到过  
>`BSRR`：置位/复位寄存器 GPIOx_BSRR，Bit Set/Reset Register  
>`ODR`：输出数据寄存器GPIOx_ODR，Output Data Register  
>![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230111155258.png)

>由`输出数据寄存器GPIOx_ODR`开始，单片机可以对该寄存器修改值，从而给输出控制一个信号。  
除此之外对`置位/复位寄存器 GPIOx_BSRR`的值进行修改也可以影响”输出数据寄存器“的值。

---

下面是本次例程的配置GPIO编程要点：

- **使能GPIO端口时钟**

    要使用这个外设，首先得使能时钟，没什么说的。
    
    ---
    
    >bsp_led.c
    ```c++
    /*开启LED相关的GPIO外设时钟*/
    LED1_GPIO_CLK_ENABLE();
    LED2_GPIO_CLK_ENABLE();
    ```
    这里使能时钟用的函数是宏定义的名字，实际是用`__HAL_RCC_GPIOx_CLK_ENABLE()`这个函数。
    >注意`GPIOx`改成实际的port口。  
    >这里的GPIO时钟宏`__HAL_RCC_GPIOx_CLK_ENABLE()`是 STM32HAL库定义的GPIO端口时钟相关的宏，它的作用与`GPIO_PIN_x`这类宏类似，是用于指示寄存器位的，方便库函数使用。

- **初始化GPIO目标引脚为推挽输出模式，配置GPIO初始化结构体**

    没有什么特殊需求，所以输出模式直接使用推挽输出模式

    ---

    >bsp_led.c
    ```c++
    /*选择要控制的GPIO引脚*/															   
    GPIO_InitStruct.Pin = LED1_PIN;	

    /*设置引脚的输出类型为推挽输出*/
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;  

    /*设置引脚为上拉模式*/
    GPIO_InitStruct.Pull = GPIO_PULLUP;

    /*设置引脚速率为高速 */   
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    ```

    这里配置了四个东西，分别是：  
    `GPIO_InitStruct.Pin`：pin号  
    `GPIO_InitStruct.Mode`：输出模式  
    `GPIO_InitStruct.Pull`：上下拉模式  
    `GPIO_InitStruct.Speed`：引脚速率

    其中HAL库中配置输出模式：  
    推挽输出`GPIO_MODE_OUTPUT_PP`；  
    开漏输出`GPIO_MODE_OUTPUT_OD`；  
    复用推挽输出`GPIO_MODE_AF_PP`；  
    复用开漏输出`GPIO_MODE_AF_OD`；  
    ……等等
    
    >其中的简写全称记录如下：  
    >PP for push-pull，推挽  
    >OD for open-drain，开漏  
    >AF for alternate function，复用功能

    关于HAL库中GPIO初始化结构体成员变量的配置，考虑到本次例程的篇幅，现简单记录如上，后面有时间做一个总结。

    下面是整个初始化函数的程序：

    >bsp_led.c
    ```c++
    void LED_GPIO_Config(void)
    {		
        /*定义一个GPIO_InitTypeDef类型的结构体*/
        GPIO_InitTypeDef  GPIO_InitStruct;

        /*开启LED相关的GPIO外设时钟*/
        LED1_GPIO_CLK_ENABLE();
        LED2_GPIO_CLK_ENABLE();

        /*设置引脚的输出类型为推挽输出*/
        GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;  

        /*设置引脚为上拉模式*/
        GPIO_InitStruct.Pull  = GPIO_PULLUP;

        /*设置引脚速率为高速 */   
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

        /*调用库函数，使用上面配置的GPIO_InitStructure初始化GPIO*/
        /*选择要控制的GPIO引脚*/
        GPIO_InitStruct.Pin = LED1_PIN;	
        HAL_GPIO_Init(LED1_GPIO_PORT, &GPIO_InitStruct);	

        /*选择要控制的GPIO引脚*/															   
        GPIO_InitStruct.Pin = LED2_PIN;	
        HAL_GPIO_Init(LED2_GPIO_PORT, &GPIO_InitStruct);	

        /*关闭RGB灯*/
        LED_RGBOFF;
    }
    ```

- **编写简单测试程序，控制GPIO引脚输出高低电平**
    
    main函数

    ---
    >main.c
    ```c++
    int main(void)
    {
        /* 系统时钟初始化成72 MHz */
        SystemClock_Config();

        /* LED 端口初始化 */
        LED_GPIO_Config();

        /* 控制LED灯 */
        while (1)
        {
            LED1( ON );			 // 亮 
            HAL_Delay(1000);
            LED1( OFF );		  // 灭
        
            LED2( ON );			// 亮 
            HAL_Delay(1000);
            LED2( OFF );		  // 灭  
        }
    }
    ```
    >   控制LED使用的宏定义
    >    ```c++  
    >    /* 直接操作寄存器的方法控制IO */
    >    #define	digitalHi(p,i)			{p->BSRR=i;}			              //设置为高电平
    >    #define digitalLo(p,i)			{p->BSRR=(uint32_t)i << 16;}		 //输出低电平
    >    #define digitalToggle(p,i)		{p->ODR ^=i;}		            	//输出反转状态
    >
    >
    >    /* 定义控制IO的宏 */
    >    #define LED1_TOGGLE		digitalToggle(LED1_GPIO_PORT,LED1_PIN)
    >    #define LED1_OFF		digitalHi(LED1_GPIO_PORT,LED1_PIN)
    >    #define LED1_ON			digitalLo(LED1_GPIO_PORT,LED1_PIN)
    >
    >    #define LED2_TOGGLE		digitalToggle(LED2_GPIO_PORT,LED2_PIN)
    >    #define LED2_OFF		digitalHi(LED2_GPIO_PORT,LED2_PIN)
    >    #define LED2_ON			digitalLo(LED2_GPIO_PORT,LED2_PIN)
    >                        
    >    //黑(全部关闭)
    >    #define LED_RGBOFF	\
    >            LED1_OFF;\
    >            LED2_OFF
    >    ```

    大概就是这样。

## 总结

关于GPIO的分析和HAL库的配置大概就是这样。
与三极管，实现控制大功率电路通断。

`输入功能`可以实现检测外部电平、读取引脚状态、采集模拟信号等功能。

下面对GPIO基本框图进行分析，了解GPIO主要的工作原理。
>过一遍理解就可，有个印象

## GPIO基本框图
![GPIO基本框图](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230106151230.png)
虽然标注的有点花花绿绿（）但是大致上可以分为三个部分：

- 紫色为引脚与外部设备连接的端口。
- 以蓝色箭头为主线的一串冷色相关的，为输入模式相关的电路。
- 以红色箭头为主线的一串暖色相关的，为输出模式相关的电路。

---

首先是紫色的`保护二极管与上下拉电阻`。

引脚电压高于 $V_{DD}$ ，上方二极管导通。  
引脚电压低于 $V_{SS}$ ，下方二极管导通。

这样就可以防止不正常的电压引入芯片导致芯片烧毁。但要注意，纵使有这样的保护也不能直接外接大功率驱动器件。

---

其次是红色的`输出模式`这一条线，从左往右看。

由`输出数据寄存器GPIOx_ODR`开始，单片机可以对该寄存器修改值，从而给输出控制一个信号。  
除此之外对`置位/复位寄存器 GPIOx_BSRR`的值进行修改也可以影响”输出数据寄存器“的值。

由这两个寄存器给出信号之后，通过一个开关接入输出控制。该开关可以控制是由”输出数据寄存器“还是由其他片上外设作为”输出源“。
>例如要使用串口的时候，需要一个通讯发送引脚的时候，就可以将某个GPIO引脚配置为串口复用功能。

最关键的是由P-MOS与N-MOS两个管子组成的单元电路。
这个单元电路使得GPIO的输出模式有两个模式，分别是`推挽输出`和`开漏输出`。

`推挽输出`中  
输出源给高电平，经过反向，上方的P-MOS导通，下方的N-MOS关闭，对外输出高电平；  
输出源给低电平，经过反向，上方N-MOS管关闭，上方P-MOS导通，对外输出低电平；

`开漏输出`中，上方的P_MOS不工作  
当N-MOS导通的时候对外输出为低电平（接地）；  
当N-MOS关闭的时候对外为高阻态，正常使用的时候需要接上拉电阻；此时等效于开漏电路，具有“线与”特性，一般应用在I2C、SMBUS通讯等需要“线与”功能的总线电路中，或者需要输出电平不匹配的时候。
>比如需要输出 5 伏的高电平，就可以在外部接一个上拉电阻，上拉电源为 5 伏， 并且把GPIO设置为开漏模式，当输出高阻态时，由上拉电阻和电源向外输出 5 伏的电平。

---

最后是蓝色的`输入模式`这一条线，从右往左看。

通过引脚采集电压之后，首先通过上/下拉电阻，配置成上/下拉输入，随后接一个TTL肖特基触发器，进行模数转换。

最后将触发器给出的数字信号存储在“输入数据寄存器GPIOx_IDR”中，通过读取该寄存器就可以了解GPIO引脚的电平状态。

输入模式中的复用功能输入与复用功能输出类似，也只是不经过“输入数据寄存器”，直接连接其他的片上外设，让别的外设读取引脚状态。
>例如使用串口通讯，需要一个通讯接收引脚的时候，就可以将某个GPIO引脚配置为串口复用功能。此处与输出模式的复用功能输入所举的例子对应。

模拟输入与输出的通道是当引脚需要做ADC/DAC的时候，需要用原始信号时所提供的通道。  
`ADC`时需要采集原始的模拟信号，因此不经过触发器。  
`DAC`时需要输出原始的模拟信号，因此不经过双MOS管的结构。
>A for analog, D for digital. ADC-模数转换，DAC数模转换。

---

至此基本将GPIO的原理框图进行了一遍梳理，下面为在HAL库中如何配置GPIO的一些记录。

##  HAL库中GPIO的配置

>本节例程参考野火的<STM32 HAL库开发实战指南——F103系列>使用固件库点亮LED一章

首先明确一下软件上的层级关系：

LED灯的控制相关的代码不属于STM32HAL库的内容，一般将这样的控制代码封装在另外的文件中。本次例程中将文件名命名为`bsp_led.c`与`bsp_led.h`，这里的bsp是`Board Support Packet`的简写，即板级支持包。

>对bsp的个人理解：不同外设的控制代码。

本次使用的开发板是`野火的F103-MINI 开发板`，下面是本次例程所涉及的硬件原理图：

![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230111150900.png)

如图所示容易看出，若将这两个GPIO引脚设置为低电平即可点亮LED。

本次例程中关于引脚定义的宏：

>bsp_led.h
```c
//引脚定义
/*******************************************************/

#define LED1_PIN                  GPIO_PIN_2               
#define LED1_GPIO_PORT            GPIOC                    
#define LED1_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()


#define LED2_PIN                  GPIO_PIN_3              
#define LED2_GPIO_PORT            GPIOC                      
#define LED2_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()

/************************************************************/
```

```c
/** 控制LED灯亮灭的宏，
	* LED低电平亮，设置ON=0，OFF=1
	* 若LED高电平亮，把宏设置成ON=1 ，OFF=0 即可
	*/
#define ON  GPIO_PIN_RESET
#define OFF GPIO_PIN_SET

/* 带参宏，可以像内联函数一样使用 */
#define LED1(a)	HAL_GPIO_WritePin(LED1_GPIO_PORT,LED1_PIN,a)


#define LED2(a)	HAL_GPIO_WritePin(LED2_GPIO_PORT,LED2_PIN,a)


/* 直接操作寄存器的方法控制IO */
#define	digitalHi(p,i)			{p->BSRR=i;}			              //设置为高电平
#define digitalLo(p,i)			{p->BSRR=(uint32_t)i << 16;}		 //输出低电平
#define digitalToggle(p,i)		{p->ODR ^=i;}		            	//输出反转状态


/* 定义控制IO的宏 */
#define LED1_TOGGLE		digitalToggle(LED1_GPIO_PORT,LED1_PIN)
#define LED1_OFF		digitalHi(LED1_GPIO_PORT,LED1_PIN)
#define LED1_ON			digitalLo(LED1_GPIO_PORT,LED1_PIN)

#define LED2_TOGGLE		digitalToggle(LED2_GPIO_PORT,LED2_PIN)
#define LED2_OFF		digitalHi(LED2_GPIO_PORT,LED2_PIN)
#define LED2_ON			digitalLo(LED2_GPIO_PORT,LED2_PIN)


					
//黑(全部关闭)
#define LED_RGBOFF	\
		LED1_OFF;\
		LED2_OFF
```

此处控制LED亮灭的操作是直接向`BSRR`写入控制指令来实现的，对`BSRR`低 16 位 写 1 输出高电平，对`BSRR`高 16 位写 1 输出低电平。同时，对`ODR`某位进行异或操作可反转位的状态。

>`BSRR`和`ODR`在前文详解GPIO框图的输出模式时候提到过  
>`BSRR`：置位/复位寄存器 GPIOx_BSRR，Bit Set/Reset Register  
>`ODR`：输出数据寄存器GPIOx_ODR，Output Data Register  
>![](https://cdn.jsdelivr.net/gh/Sagi-Rastar/SagiImg@main/img/20230111155258.png)

>由`输出数据寄存器GPIOx_ODR`开始，单片机可以对该寄存器修改值，从而给输出控制一个信号。  
除此之外对`置位/复位寄存器 GPIOx_BSRR`的值进行修改也可以影响”输出数据寄存器“的值。

---

下面是本次例程的配置GPIO编程要点：

- **使能GPIO端口时钟**

    要使用这个外设，首先得使能时钟，没什么说的。
    
    ---
    
    >bsp_led.c
    ```c++
    /*开启LED相关的GPIO外设时钟*/
    LED1_GPIO_CLK_ENABLE();
    LED2_GPIO_CLK_ENABLE();
    ```
    这里使能时钟用的函数是宏定义的名字，实际是用`__HAL_RCC_GPIOx_CLK_ENABLE()`这个函数。
    >注意`GPIOx`改成实际的port口。  
    >这里的GPIO时钟宏`__HAL_RCC_GPIOx_CLK_ENABLE()`是 STM32HAL库定义的GPIO端口时钟相关的宏，它的作用与`GPIO_PIN_x`这类宏类似，是用于指示寄存器位的，方便库函数使用。

- **初始化GPIO目标引脚为推挽输出模式，配置GPIO初始化结构体**

    没有什么特殊需求，所以输出模式直接使用推挽输出模式

    ---

    >bsp_led.c
    ```c++
    /*选择要控制的GPIO引脚*/															   
    GPIO_InitStruct.Pin = LED1_PIN;	

    /*设置引脚的输出类型为推挽输出*/
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;  

    /*设置引脚为上拉模式*/
    GPIO_InitStruct.Pull = GPIO_PULLUP;

    /*设置引脚速率为高速 */   
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    ```

    这里配置了四个东西，分别是：  
    `GPIO_InitStruct.Pin`：pin号  
    `GPIO_InitStruct.Mode`：输出模式  
    `GPIO_InitStruct.Pull`：上下拉模式  
    `GPIO_InitStruct.Speed`：引脚速率

    其中HAL库中配置输出模式：  
    推挽输出`GPIO_MODE_OUTPUT_PP`；  
    开漏输出`GPIO_MODE_OUTPUT_OD`；  
    复用推挽输出`GPIO_MODE_AF_PP`；  
    复用开漏输出`GPIO_MODE_AF_OD`；  
    ……等等
    
    >其中的简写全称记录如下：  
    >PP for push-pull，推挽  
    >OD for open-drain，开漏  
    >AF for alternate function，复用功能

    关于HAL库中GPIO初始化结构体成员变量的配置，考虑到本次例程的篇幅，现简单记录如上，后面有时间做一个总结。

    下面是整个初始化函数的程序：

    >bsp_led.c
    ```c++
    void LED_GPIO_Config(void)
    {		
        /*定义一个GPIO_InitTypeDef类型的结构体*/
        GPIO_InitTypeDef  GPIO_InitStruct;

        /*开启LED相关的GPIO外设时钟*/
        LED1_GPIO_CLK_ENABLE();
        LED2_GPIO_CLK_ENABLE();

        /*设置引脚的输出类型为推挽输出*/
        GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;  

        /*设置引脚为上拉模式*/
        GPIO_InitStruct.Pull  = GPIO_PULLUP;

        /*设置引脚速率为高速 */   
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

        /*调用库函数，使用上面配置的GPIO_InitStructure初始化GPIO*/
        /*选择要控制的GPIO引脚*/
        GPIO_InitStruct.Pin = LED1_PIN;	
        HAL_GPIO_Init(LED1_GPIO_PORT, &GPIO_InitStruct);	

        /*选择要控制的GPIO引脚*/															   
        GPIO_InitStruct.Pin = LED2_PIN;	
        HAL_GPIO_Init(LED2_GPIO_PORT, &GPIO_InitStruct);	

        /*关闭RGB灯*/
        LED_RGBOFF;
    }
    ```

- **编写简单测试程序，控制GPIO引脚输出高低电平**
    
    main函数

    ---
    >main.c
    ```c++
    int main(void)
    {
        /* 系统时钟初始化成72 MHz */
        SystemClock_Config();

        /* LED 端口初始化 */
        LED_GPIO_Config();

        /* 控制LED灯 */
        while (1)
        {
            LED1( ON );			 // 亮 
            HAL_Delay(1000);
            LED1( OFF );		  // 灭
        
            LED2( ON );			// 亮 
            HAL_Delay(1000);
            LED2( OFF );		  // 灭  
        }
    }
    ```
    >   控制LED使用的宏定义
    >   ```c++  
    >    /* 直接操作寄存器的方法控制IO */
    >    #define	digitalHi(p,i)			{p->BSRR=i;}			              //设置为高电平
    >    #define digitalLo(p,i)			{p->BSRR=(uint32_t)i << 16;}		 //输出低电平
    >    #define digitalToggle(p,i)		{p->ODR ^=i;}		            	//输出反转状态
    >
    >
    >    /* 定义控制IO的宏 */
    >    #define LED1_TOGGLE		digitalToggle(LED1_GPIO_PORT,LED1_PIN)
    >    #define LED1_OFF		digitalHi(LED1_GPIO_PORT,LED1_PIN)
    >    #define LED1_ON			digitalLo(LED1_GPIO_PORT,LED1_PIN)
    >
    >    #define LED2_TOGGLE		digitalToggle(LED2_GPIO_PORT,LED2_PIN)
    >    #define LED2_OFF		digitalHi(LED2_GPIO_PORT,LED2_PIN)
    >    #define LED2_ON			digitalLo(LED2_GPIO_PORT,LED2_PIN)
    >                        
    >    //黑(全部关闭)
    >    #define LED_RGBOFF	\
    >            LED1_OFF;\
    >            LED2_OFF
    >   ```

    大概就是这样。

## 总结

关于GPIO的分析和HAL库的配置大概就是这样。
