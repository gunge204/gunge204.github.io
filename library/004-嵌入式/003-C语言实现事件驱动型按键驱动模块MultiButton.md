# C语言实现事件驱动型按键驱动模块MultiButton
>维护人员：**80后老男孩**  
>创建时间：2021-03-04  

> **修改日志：**

> - 【2020-03-04】  新增

> 本文已发表于csdn博客：[C语言实现事件驱动型按键驱动模块MultiButton](https://blog.csdn.net/weixin_41034400/article/details/114357718)
[TOC]

## 简介

在嵌入式系统尤其是单片机系统中经常用到按键检测和处理，这里提供一个标准的驱动函数模块*MultiButton*，能够提供按下、弹起、单击、双击、连击、长按等按键事件。

*MultiButton* 是一个小巧简单易用的事件驱动型按键驱动模块，可无限量扩展按键，按键事件的回调异步处理方式可以简化你的程序结构，去除冗余的按键处理硬编码，让你的按键业务逻辑更清晰。 

驱动模块代码的原作者是*Zibin Zheng*，发布于github上开源，代码写的非常精彩，可以拿来学习。

这里介绍其使用方法，并分析代码实现的原理，重要代码本文作者均额外加了中文注释，方便理解；

**github上原作者代码有bug，本文已经给予修订。无bug版的完整源代码和使用demo[请在这里下载](https://download.csdn.net/download/weixin_41034400/15557210)**，可以直接在项目中使用。

## 使用方法
1.先申请一个按键结构

```c
struct Button button1;
```
2.初始化按键对象，绑定按键的GPIO电平读取接口**read_button_pin()** ，后一个参数设置有效触发电平

```c
button_init(&button1, read_button_pin, 0);
```
3.注册按键事件

```c
button_attach(&button1, SINGLE_CLICK, Callback_SINGLE_CLICK_Handler);
button_attach(&button1, DOUBLE_CLICK, Callback_DOUBLE_Click_Handler);
...
```
4.启动按键

```c
button_start(&button1);
```
5.设置一个5ms间隔的定时器循环调用后台处理函数**button_ticks()**,  

如果是在嵌入式操作系统使用，则可以搞一个每5ms调用一次的按键任务去执行button_ticks()即可

```c
while(1) {
    ...
    if(timer_ticks == 5) {
        timer_ticks = 0;

        button_ticks();
    }
}
```

## 特性

*MultiButton* 使用C语言实现，基于面向对象方式设计思路，每个按键对象单独用一份数据结构管理：

```c
struct Button {
	uint16_t ticks;
	uint8_t  repeat: 4;
	uint8_t  event : 4;
	uint8_t  state : 3;
	uint8_t  debounce_cnt : 3;
	uint8_t  active_level : 1;
	uint8_t  button_level : 1;
	uint8_t  (*hal_button_Level)(void);
	BtnCallback  cb[number_of_event];
	struct Button* next;
};
```
这样每个按键使用单向链表相连，依次进入 **button_handler(struct Button* handle)** 状态机处理，所以每个按键的状态彼此独立。


## 按键事件

| 事件             | 说明                                 |
| ---------------- | ------------------------------------ |
| PRESS_DOWN       | 按键按下，每次按下都触发             |
| PRESS_UP         | 按键弹起，每次松开都触发             |
| PRESS_REPEAT     | 重复按下触发，变量repeat计数连击次数 |
| SINGLE_CLICK     | 单击按键事件                         |
| DOUBLE_CLICK     | 双击按键事件                         |
| LONG_PRESS_START | 达到长按时间阈值时触发一次           |
| LONG_PRESS_HOLD  | 长按期间一直触发                     |


## 使用方法举例

```c
#include "multi_button.h"

struct Button btn1;

uint8_t read_button1_GPIO()
{
	return HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin);
}
void BTN1_PRESS_DOWN_Handler(void* btn)
{
	//do something...
}

void BTN1_PRESS_UP_Handler(void* btn)
{
	//do something...
}

...
```
```c
int main()
{
	button_init(&btn1, read_button1_GPIO, 0);   
	button_attach(&btn1, PRESS_DOWN,      BTN1_PRESS_DOWN_Handler);
	button_attach(&btn1, PRESS_UP,         BTN1_PRESS_UP_Handler);
	button_attach(&btn1, PRESS_REPEAT,     BTN1_PRESS_REPEAT_Handler);
	button_attach(&btn1, SINGLE_CLICK,     BTN1_SINGLE_Click_Handler);
	button_attach(&btn1, DOUBLE_CLICK,     BTN1_DOUBLE_Click_Handler);
	button_attach(&btn1, LONG_PRESS_START, BTN1_LONG_PRESS_START_Handler);
	button_attach(&btn2, LONG_PRESS_HOLD,  BTN1_LONG_PRESS_HOLD_Handler);
	button_start(&btn1);

	//make the timer invoking the button_ticks() interval 5ms.
	//This function is implemented by yourself.
	__timer_start(button_ticks, 0, 5);

	while(1)
	{}
}
```

## 核心代码分析
### 头文件声明
```c
#include "stdint.h"
#include "string.h"

//According to your need to modify the constants.
#define TICKS_INTERVAL 5 //ms，系统节拍5ms
#define DEBOUNCE_TICKS 3 //MAX 8，去抖的次数
#define SHORT_TICKS (300 / TICKS_INTERVAL)	//单击短按键的时间是300ms以上
#define LONG_TICKS (1000 / TICKS_INTERVAL)	//长按键的时间是1000ms以上
#define LONG_HOLD_CYC (500 / TICKS_INTERVAL)	//处于长按键保持触发状态时，每500ms调用一次回调函数

typedef void (*BtnCallback)(void *);

//声明一个按键事件的枚举类型，包括了所有的按键操作类型
typedef enum
{
	PRESS_DOWN = 0,	//按键按下，每次按下都触发 
	PRESS_UP,		//按键弹起，每次松开都触发
	PRESS_REPEAT,	//重复按下触发，变量repeat计数连击次数
	SINGLE_CLICK,	//单击按键事件
	DOUBLE_CLICK,	//双击按键事件
	LONG_PRESS_START,	//达到长按时间阈值时触发一次
	LONG_PRESS_HOLD,	//长按期间一直触发
	number_of_event,	//事件数量
	NONE_PRESS		//没有任何按键事件
} PressEvent;

//声明一个按键结构体类型（链表），
typedef struct Button
{
	uint16_t ticks;			//系统节拍计数
	uint8_t repeat : 4;		//重复按键，双击、三击……
	uint8_t event : 4;		//当前按键事件
	uint8_t state : 3;		//当前按键状态
	uint8_t debounce_cnt : 3;	//去抖次数	
	uint8_t active_level : 1;	//按键按下的有效电平			
	uint8_t button_level : 1;	//按键电平状态，获取一次更新一次
	uint8_t (*hal_button_Level)(void);	//获取按键电平的函数指针
	BtnCallback cb[number_of_event];	//按键回调函数指针数组
	struct Button *next;	//指向链表的下一个
} Button;

#ifdef __cplusplus
extern "C"
{
#endif
	//按键结构体初始化，赋予按键获取电平的函数，并读出当前电平
	void button_init(struct Button *handle, uint8_t (*pin_level)(), uint8_t active_level);
	//按键注册（赋予特定的按键事件以回调函数）
	void button_attach(struct Button *handle, PressEvent event, BtnCallback cb);
	//获取按键事件返回到按键结构体
	PressEvent get_button_event(struct Button *handle);
	//启动一个按键，即新增一个按键结构体链表节点
	int button_start(struct Button *handle);
	//停止一个按键，从链表中删除节点
	void button_stop(struct Button *handle);
	//按键节拍，每一次节拍事件遍历按键列表中所有按键，调用驱动函数
	void button_ticks(void);

#ifdef __cplusplus
}
#endif

```
### 函数定义源代码  
模块的核心函数是**void button_handler(struct Button *handle)**，这是一个状态机程序，该函数流程如下图所示：  

![](assets/004/003-1.jpg)

代码实现如下：
```c
#define EVENT_CB(ev)    \
	if (handle->cb[ev]) \
	handle->cb[ev]((Button *)handle)

//button handle list head.
//按键结构体链表，初始化指针为NULL，空链表
static struct Button *head_handle = NULL;

/**
  * @brief  Initializes the button struct handle.
  * @param  handle: the button handle strcut.
  * @param  pin_level: read the HAL GPIO of the connet button level.
  * @param  active_level: pressed GPIO level.
  * @retval None
  */
 //按键结构体初始化，赋予按键获取电平的函数指针，并读出当前电平
void button_init(struct Button *handle, uint8_t (*pin_level)(), uint8_t active_level)
{
	memset(handle, 0, sizeof(struct Button));		   //结构体初始化为全0
	handle->event = (uint8_t)NONE_PRESS;			   //初始化为没有按键事件
	handle->hal_button_Level = pin_level;			   //读取按键电平状态函数指针
	handle->button_level = handle->hal_button_Level(); //读取一次当前按键电平
	handle->active_level = active_level;			   //按下的有效电平是1还是0
}

/**
  * @brief  Attach the button event callback function.
  * @param  handle: the button handle strcut.
  * @param  event: trigger event type.
  * @param  cb: callback function.
  * @retval None
  */
 //按键注册（赋予特定的按键事件以回调函数）
void button_attach(struct Button *handle, PressEvent event, BtnCallback cb)
{
	handle->cb[event] = cb; //赋予某特定按键事件的回调函数指针
}

/**
  * @brief  Inquire the button event happen.
  * @param  handle: the button handle strcut.
  * @retval button event.
  */
 //获取按键事件返回到按键结构体
PressEvent get_button_event(struct Button *handle)
{
	return (PressEvent)(handle->event); //返回按键事件
}

/**
  * @brief  Button driver core function, driver state machine.
  * @param  handle: the button handle strcut.
  * @retval None
  */
//按键驱动核心函数，驱动状态机
void button_handler(struct Button *handle)
{
	//读取当前电平状态
	uint8_t read_gpio_level = handle->hal_button_Level();

	//ticks counter working..
	//只要按键状态不是0，就递增记录系统节拍次数
	if ((handle->state) > 0)
		handle->ticks++;

	/*------------button debounce handle---------------*/
	//如果这一次获取的电平状态与上一次状态不符，则连读几次去抖才能赋值当前电平
	if (read_gpio_level != handle->button_level)
	{ //not equal to prev one
		//continue read 3 times same new level change
		if (++(handle->debounce_cnt) >= DEBOUNCE_TICKS)
		{
			handle->button_level = read_gpio_level;
			handle->debounce_cnt = 0;
		}
	}
	//否则不用去抖检测处理
	else
	{ //leved not change ,counter reset.
		handle->debounce_cnt = 0;
	}

	/*-----------------State machine-------------------*/
	switch (handle->state)
	{
	case 0: //如果前次按键状态是0，且当前按键电平等于有效电平，则认为是按键开始按下了
		if (handle->button_level == handle->active_level)
		{										 //start press down
			handle->event = (uint8_t)PRESS_DOWN; //有按键按下事件发生
			EVENT_CB(PRESS_DOWN);				 //如果存在回调函数则执行
			handle->ticks = 0;					 //节拍计数清零
			handle->repeat = 1;					 //事件重复次数为1
			handle->state = 1;					 //按键状态更新为1（按下）
		}
		else //如果不是有效电平，则无按键事件
		{
			handle->event = (uint8_t)NONE_PRESS;
		}
		break;

	case 1: //如果前次按键状态是1（按下了），且当前电平不是有效电平，则认为按键松开了
		if (handle->button_level != handle->active_level)
		{									   //released press up
			handle->event = (uint8_t)PRESS_UP; //有按键松开弹起事件发生
			EVENT_CB(PRESS_UP);				   //如果存在回调函数则执行
			handle->ticks = 0;				   //节拍计数清零
			handle->state = 2;				   //按键状态更新为2（松开）
		}
		//如果节拍计数超过了长按键阀值，则认为是一次长按键事件
		else if (handle->ticks > LONG_TICKS)
		{
			handle->event = (uint8_t)LONG_PRESS_START; //有长按键开始事件发生
			EVENT_CB(LONG_PRESS_START);				   //如果存在回调函数则执行
			handle->state = 5;						   //按键状态更新为5（长按开始）
		}
		break;

	case 2: //如果前次按键状态是2（松开了），且当前电平是有效电平，则认为按键又按下了
		if (handle->button_level == handle->active_level)
		{										 //press down again
			handle->event = (uint8_t)PRESS_DOWN; //有按键按下事件发生
			EVENT_CB(PRESS_DOWN);				 //如果存在回调函数则执行
			handle->repeat++;					 //又一次按下，重复次数加1
			EVENT_CB(PRESS_REPEAT);				 // repeat hit，如果存在回调函数则执行
			handle->ticks = 0;					 //节拍计数清零
			handle->state = 3;					 //按键状态更新为3（重复按键）
		}
		//如果不是有效电平，说明松开了，那么检查摁下的时间是否超过了短按键的节拍计数阀值
		else if (handle->ticks > SHORT_TICKS)
		{ //released timeout
			//超过阀值，且只按下了1次，则认为是一次单击事件
			if (handle->repeat == 1)
			{
				handle->event = (uint8_t)SINGLE_CLICK; //有单击事件发生
				EVENT_CB(SINGLE_CLICK);				   //如果存在回调函数则执行
			}
			//如果按下了2次，则认为是一次双击事件
			else if (handle->repeat == 2)
			{
				handle->event = (uint8_t)DOUBLE_CLICK; //有双击事件发生
				EVENT_CB(DOUBLE_CLICK);				   // repeat hit 如果存在回调函数则执行
			}
			handle->state = 0; //按键状态更新为0（未按下）
		}
		break;

	case 3: //如果前次按键状态是3（重复按下），且当前电平不是有效电平，则认为是按键弹起或短按键事件
		if (handle->button_level != handle->active_level)
		{									   //released press up
			handle->event = (uint8_t)PRESS_UP; //有按键弹起事件发生
			EVENT_CB(PRESS_UP);				   //如果存在回调函数则执行
			//如果记录的节拍时间小于300ms，
			if (handle->ticks < SHORT_TICKS)
			{
				handle->ticks = 0; //节拍计数清零
				handle->state = 2; //repeat press按键状态更新为2（松开）
			}
			else
			{
				handle->state = 0; //按键状态更新为0（未按下）
			}
		}
		break;

	case 5: //如果前次按键状态是5（长按键开始），且当前电平是有效电平，则认为处于长按键保持出发状态
		if (handle->button_level == handle->active_level)
		{
			//continue hold trigger
			handle->event = (uint8_t)LONG_PRESS_HOLD; //有按键弹起事件发生
			if (handle->ticks % LONG_HOLD_CYC == 0)	//每500ms调用一次回调函数
			{
				EVENT_CB(LONG_PRESS_HOLD); //如果存在回调函数则执行
			}
		}
		else								   //如果不是有效电平，则认为是按键弹起事件
		{									   //releasd
			handle->event = (uint8_t)PRESS_UP; //有按键弹起事件发生
			EVENT_CB(PRESS_UP);				   //如果存在回调函数则执行
			handle->state = 0;				   //reset按键状态复位
		}
		break;
	}
}

/**
  * @brief  Start the button work, add the handle into work list.
  * @param  handle: target handle strcut.
  * @retval 0: succeed. -1: already exist.
  */
 //启动一个按键，即新增一个按键结构体链表节点
int button_start(struct Button *handle)
{
	struct Button *target = head_handle;
	while (target)
	{
		if (target == handle)
			return -1; //already exist.
		target = target->next;
	}
	handle->next = head_handle;
	head_handle = handle;
	return 0;
}

/**
  * @brief  Stop the button work, remove the handle off work list.
  * @param  handle: target handle strcut.
  * @retval None
  */
 //停止一个按键，即从链表中删除该按键节点
void button_stop(struct Button *handle)
{
	struct Button **curr;
	for (curr = &head_handle; *curr;)
	{
		struct Button *entry = *curr;
		if (entry == handle)
		{
			*curr = entry->next;
			//			free(entry);
		}
		else
			curr = &entry->next;
	}
}

/**
  * @brief  background ticks, timer repeat invoking interval 5ms.
  * @param  None.
  * @retval None
  */
 //按键节拍，每一次节拍事件遍历按键列表中所有按键，调用驱动函数
void button_ticks()
{
	struct Button *target;
	for (target = head_handle; target; target = target->next)
	{
		button_handler(target);
	}
}

```

**无bug版的完整源代码和使用demo[请在这里下载](https://download.csdn.net/download/weixin_41034400/15557210)**


