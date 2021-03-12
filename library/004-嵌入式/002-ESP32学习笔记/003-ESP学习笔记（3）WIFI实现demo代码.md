# ESP32学习笔记（3）WIFI实现demo代码
>维护人员：**80后老男孩**  
>创建时间：2021-03-11  
>本文已发表于csdn博客[ESP32学习笔记（3）WIFI实现demo代码](https://blog.csdn.net/weixin_41034400/article/details/114691945)



[TOC]

Wi-Fi 库支持配置及监控 ESP32 Wi-Fi 连网功能。

相关内容参考乐鑫的文档https://docs.espressif.com/projects/esp-idf/zh_CN/release-v4.1/api-reference/network/esp_wifi.html

# 支持配置：

- 基站模式（即 STA 模式或 Wi-Fi 客户端模式），此时 ESP32 连接到接入点 (AP)。此时**相当于ESP在蹭网**
- AP 模式（即 Soft-AP 模式或接入点模式），此时基站连接到 ESP32。此时**相当于ESP开热点**
- AP-STA 共存模式（ESP32 既是接入点，同时又作为基站连接到另外一个接入点）。此时**相当于ESP连着隔壁wifi开热点给自家用**
- 上述模式的各种安全模式（WPA、WPA2 及 WEP 等）。可以理解成**安全蹭网**
- 扫描接入点（包括主动扫描及被动扫描）。**主动找别人家网蹭**
- 使用混杂模式监控 IEEE802.11 Wi-Fi 数据包。**可以理解成ESP能看到你上了什么不可描述的网站**

# STA模式demo代码分析

我们把esp32当成一个终端设备，让他用wifi去连接我们家里的路由器，下面分析demo代码，给出代码的中文解释方便理解。  
初次接触新东西的最好和最快的办法就是详细学习和分析官方给的demo，官方写的东西一般也比较正规，可读性很强，**阅读高手的代码对我们代码能力的提高也有很大帮助。**

demo代码在我们下载安装的\\esp-idf\\examples\\wifi\\getting_started\\station

**其中有几个函数和变量类型的概念需要首先熟悉一下，以方便下面代码的阅读：**

## 与FreeRTOS相关的

### 事件组

事件组是一种将事件传递给任务的方式，与通过信号量与队列传递的方式不同，他们主要特征为：
1，事件组允许任务进入阻塞态等待多个事件之一的混合发生。
2，当事件发生时，事件组使所有等待该事件的任务进入阻塞态。
这种方式对于多重同步任务有很大帮助，并且可降低RAM的适用，因为它可减少大量二进制信号量的适用。当需要使用事件组时，需要编译源文件event_groups.c。

**事件组的特性**

**事件组，事件标志和事件比特（event bits）**  
一个事件标志就是一个布尔型1或0的量来表示一个事件是否发生。一个事件组就是一系列事件标志的集合。
一个事件标志只能为1或0，并且存储在一个数据位中，因此我们可以用一个类型为EventBits_t的变量中的不同位来表示不同事件，不同事件的标志对应变量中的不同位。当该位为1时表示事件发生，为0时表示没发生。

**关于EventBits_t的数据类型**  
该数据的位数与FreeRTOSConfig.h中的configUSE_16_BIT_TICKS有关，当configUSE_16_BIT_TICKS置为1时，每个事件组包含8个事件位，当置为0时，每个事件组包含24个事件位。

**任务对于事件组的权限**  
所有直到该事件组存在的任务和中断都可获取该事件组，任一数量的任务都可对事件组进行读写操作。

**使用事件组管理事件**  

**xEventGroupCreate()**  
该函数用于创建一个事件组，事件组的类型为EventGroupHandle_t，其函数原型如下所示：

```c
EventGroupHandle_t xEventGroupCreate( void );
```
返回值：当返回NULL表示内存不足创建失败，否则创建成功。

**xEventGroupSetBits()**   
该函数用于将一个事件组中一个或多个事件位置位，它表明相关事件的发生。
***注意***中断函数中有专门函数xEventGroupSetBitsFromISR()。
函数原型如下：

```c
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup,
								const EventBits_t uxBitsToSet );
```
参数：
xEventGroup：事件组名称。
uxBitsToSet 需要置位的事件的位置，例如0x04，表示bit3需要置位。可以一次性置位多个，例如0x05。
返回值：事件组的值。
注意 使用uxBitsToSet 置位的值再返回后并不一定为1，因为可能存在其他任务在该函数执行过程中打断了该函数然后再次将事件组内的值复位了。

与之功能相反的函数是**EventGroupClearBits()**，即清除标志位

**xEventGroupWaitBits()**  
该函数用于任务阻塞读取事件组中的一位或几位置位，只有当所读取的置位后才退出阻塞态。函数原型如下：

```c
EventBits_t xEventGroupWaitBits( const EventGroupHandle_t xEventGroup,
								const EventBits_t uxBitsToWaitFor,
								const BaseType_t xClearOnExit,
								const BaseType_t xWaitForAllBits,
								TickType_t xTicksToWait );
```
uxBitsToWaitFor：表示需要测试那些位。
xWaitForAllBits：表示是只要有一位符合就退出阻塞态，还是uxBitsToWaitFor中的位都置位再退出阻塞态。可选pdTRUE（等全部）或者pdFALSE（有一个就行）
xEventGroup：事件组名称。
xClearOnExit：若为pdTRUE，则在该函数退出前会将相关位置为0，若为pdFALSE则不对时间组操作。（ 也可通过EventGroupClearBits()清除标志位。）
xTicksToWait ：阻塞等待最长时间。
返回值：若相关位已置位，则返回值为事件组复位前的值（xClearOnExit为pdTRUE）。若超时，则返回的值为在阻塞状态花费的时间长。

## espressif相关的

**ESP_ERROR_CHECK**  
用来检查返回值是否为ESP_OK。

**ESP_LOGI**  
是espressif的格式化输出信息，其他还包括ESP_LOGE（错误信息）、ESP_LOGD（调试信息）。

### 事件循环

esp32提供了esp_event库,用于替代freertos中的事件。并且Wi-Fi、以太网、IP等事件会被发送到此库提供的默认事件循环中。

事件和信号其实是同样的道理，来了一个事件（或者信号）去执行一个函数去处理一些事情，类似于单片机机的中断。

使用事件和事件处理函数解决问题的方式就成为事件处理机制。

ESP32的一些组件在状态发生改变时使用事件去通知应用程序，比如wifi的连接和断开事件，下面是ESP32的事件处理机制。

**Legacy Event Loop  传统的事件循环**

在esp_event Library事件库引入之前，wifi驱动，以太网，tcp/ip协议栈的事件都是通过传统的事件循环处理的，事件循环可以看作一个while(1) ;循环。传统的事件循环只支持系统已经定义好的事件id和事件信息结构，对于用户自定义的事件和蓝牙mesh事件是无法处理的。传统事件只支持一个事件处理函数，而且应用程序组件不能独自处理wifi和ip事件，需要应用程序将这些事件抛出。

**esp_event Library Event Loop esp事件库提供的事件循环**

事件库用来取代传统的事件循环，并提供了一个默认的事件循环来处理wifi、以太网和ip事件。

事件库的能力：

事件库允许组件声明一个事件到其他组件注册的处理函数中，当事件发生时执行相应的处理函数。这允许松散耦合的组件间不需要通过应用程序的调用在其他组件状态发生变化时另一个组件执行相应的操作。例如，将所有的连接处理放在一个高级的库中，订阅其他组件的连接事件，在其他组件连接状态发生变化时，高级的处理进行相应的操作。这种机制以序列化的方式简化了事件的处理并且在另一个上下文中推迟了代码的执行。简单来说，就是将事件的发生地点和事件的处理地点分来，事件处理地点去订阅事件发生地点要发生的事件，且经整个系统的事件处理放在同一个地点处理，简化了系统的设计。

esp_event Library API使用步骤：  
1. 创建事件循环 ；如果创建一个系统默认的则为：**esp_event_loop_create_default()**，后台程序自动处理

2. 注册事件和事件处理函数到事件循环中 ；
    注册事件的函数原型为：
```c
esp_err_tesp_event_handler_register(esp_event_base_tevent_base, int32_t event_id, esp_event_handler_tevent_handler, void *event_handler_arg)
```
>此函数可用于注册以下各项的处理程序：（1）特定事件，（2）某个事件基础的所有事件，或（3）系统事件循环已知的所有事件。
>
>特定事件：指定确切的event_base和event_id
>特定基准的所有事件：指定确切的event_base并使用ESP_EVENT_ANY_ID作为event_id
>循环已知的所有事件：将ESP_EVENT_ANY_BASE用作event_base，将ESP_EVENT_ANY_ID用作event_id
>可以将多个处理程序注册到事件。将单个处理程序注册到多个事件也是可能的。但是，将同一处理程序多次注册到同一事件将导致以前的注册被覆盖。
>
>注意
>事件循环库不维护event_handler_arg的副本，因此用户应确保在调用处理程序时event_handler_arg仍指向有效位置
>返回
>ESP_OK：成功
>ESP_ERR_NO_MEM：无法为处理程序分配内存
>ESP_ERR_INVALID_ARG：事件库和事件ID的无效组合
>其他：失败
>参量
>event_base：要为其注册处理程序的事件的基本ID
>event_id：要为其注册处理程序的事件的ID
>event_handler：在调度事件时调用的处理函数
>event_handler_arg：除事件数据外，在调用时传递给处理程序的数据

    事件处理函数（回调函数）写法为：
```c
    void handle(void* event_handler_arg, esp_event_base_t event_base,  int32_t event_id, void* event_data)
```
    >event事件传入四个参数，分别为
    >arg：注册时传入的参数
    >event_base：事件的基础名
    >event_id：事件的id
    >event_data：事件的信息

3. 触发事件，执行事件处理函数；

**库中的wifi和ip事件包含的内容：**

```c
const char * WIFI_EVENT = "wifi_event";
const char * IP_EVENT = "ip_event";

typedef enum {
    WIFI_EVENT_WIFI_READY = 0,           /**< ESP32 WiFi ready */
    WIFI_EVENT_SCAN_DONE,                /**< ESP32 finish scanning AP */
    WIFI_EVENT_STA_START,                /**< ESP32 station start */
    WIFI_EVENT_STA_STOP,                 /**< ESP32 station stop */
    WIFI_EVENT_STA_CONNECTED,            /**< ESP32 station connected to AP */
    WIFI_EVENT_STA_DISCONNECTED,         /**< ESP32 station disconnected from AP */
    WIFI_EVENT_STA_AUTHMODE_CHANGE,      /**< the auth mode of AP connected by ESP32 station changed */

    WIFI_EVENT_STA_WPS_ER_SUCCESS,       /**< ESP32 station wps succeeds in enrollee mode */
    WIFI_EVENT_STA_WPS_ER_FAILED,        /**< ESP32 station wps fails in enrollee mode */
    WIFI_EVENT_STA_WPS_ER_TIMEOUT,       /**< ESP32 station wps timeout in enrollee mode */
    WIFI_EVENT_STA_WPS_ER_PIN,           /**< ESP32 station wps pin code in enrollee mode */

    WIFI_EVENT_AP_START,                 /**< ESP32 soft-AP start */
    WIFI_EVENT_AP_STOP,                  /**< ESP32 soft-AP stop */
    WIFI_EVENT_AP_STACONNECTED,          /**< a station connected to ESP32 soft-AP */
    WIFI_EVENT_AP_STADISCONNECTED,       /**< a station disconnected from ESP32 soft-AP */

    WIFI_EVENT_AP_PROBEREQRECVED,        /**< Receive probe request packet in soft-AP interface */
} wifi_event_t;

typedef enum {
    IP_EVENT_STA_GOT_IP,               /*!< ESP32 station got IP from connected AP */
    IP_EVENT_STA_LOST_IP,              /*!< ESP32 station lost IP and the IP is reset to 0 */
    IP_EVENT_AP_STAIPASSIGNED,         /*!< ESP32 soft-AP assign an IP to a connected station */
    IP_EVENT_GOT_IP6,                  /*!< ESP32 station or ap or ethernet interface v6IP addr is preferred */
    IP_EVENT_ETH_GOT_IP,               /*!< ESP32 ethernet got IP from connected AP */
} ip_event_t;
```



**注意：**
**Freertos的事件组标志位和espressif中的WIFI的事件组并不是一个东西，事件组标志位类似于原子操作，防止因为顺序错乱而导致程序问题，需要我们在用户程序中去置位或者清除。而WIFI事件组只是WIFI在连接过程中的某种状态，是后台自动更新标志，无须用户程序参与。**



### wifi_config_t结构体

wifi_config_t这个结构体用于wifi连接前的配置信息，它的内容如下所示

```c
typedef struct {
    uint8_t ssid[32];//SSID
    uint8_t password[64];//密码
    uint8_t ssid_len;//SSID长度，若设为0则会自动查找到终止字符；否则会在规定长度处截断
    uint8_t channel;//AP的通道
    wifi_auth_mode_t authmode;//授权模式
    uint8_t ssid_hidden;//是否广播SSID，默认为0-广播；设为1则不广播
    uint8_t max_connection;//能连接的最大节点数量，默认为4，最大为4
    uint16_t beacon_interval;//信标间隔，默认100ms，应设置在100-60000ms内
} wifi_ap_config_t;
```

## 部分使用到的库函数

```c
1.初始化与设置
esp_wifi_init(const wifi_init_config_t *config)//WiFi功能初始化，config为初始化结构体句柄
esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf)//使能设置
esp_wifi_set_mode(wifi_mode_t mode)//模式设置
    
//可如下配置
WIFI_MODE_NULL=0
WIFI_MODE_STA//STA模式
WIFI_MODE_AP//软AP模式
WIFI_MODE_APSTA//混合模式
WIFI_MODE_MAX
    
esp_wifi_get_mode(wifi_mode_t *mode)//获取当前模式
esp_wifi_get_config(wifi_interface_t interface, wifi_config_t *conf)//获取当前设置
2.关闭wifi
esp_wifi_stop()//STA模式下断开wifi连接，AP模式下关闭热点并释放内存，共用模式下断开连接并关闭热点
esp_wifi_deinit()//释放曾在esp_wifi_init中申请的资源并停止WiFi工作，不需要wifi功能时可以使用    
3.连接/断开wifi
/* 用于STA模式 */
esp_wifi_connect()//连接WiFi
esp_wifi_disconnect()//断开WiFi

/* 用于AP模式 */
esp_wifi_deauth_sta(uint16_t aid)//停止对接入设备的授权——不让别人蹭网
esp_wifi_ap_get_sta_aid(const uint8_t mac[6], uint16_t *aid)//获取当前接入的设备信息
esp_wifi_ap_get_sta_list(wifi_sta_list_t *sta)//获取当前接入的设备列表
```

## 下面是demo代码的分析

```c
/* WiFi station Example

   This example code is in the Public Domain (or CC0 licensed, at your option.)

   Unless required by applicable law or agreed to in writing, this
   software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
   CONDITIONS OF ANY KIND, either express or implied.
*/
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"

#include "lwip/err.h"
#include "lwip/sys.h"

/* The examples use WiFi configuration that you can set via project configuration menu

   If you'd rather not, just change the below entries to strings with
   the config you want - ie #define EXAMPLE_WIFI_SSID "mywifissid"
*/
#define EXAMPLE_ESP_WIFI_SSID "hw123456"         //wifidssid，就是家里路由器的名字
#define EXAMPLE_ESP_WIFI_PASS "hwhwH9871"     //wifi密码，家里路由器的密码
#define EXAMPLE_ESP_MAXIMUM_RETRY 3 //尝试连接wifi的最大次数，若超过这个数则停止连接

/* FreeRTOS event group to signal when we are connected*/
static EventGroupHandle_t s_wifi_event_group; //声明一个事件组，事件组主要是考虑到很多网络操作需要在我们连接到路由器并拿到IP之后才能进行

/* The event group allows multiple bits for each event, but we only care about two events:
 * - we are connected to the AP with an IP
 * - we failed to connect after the maximum amount of retries */
#define WIFI_CONNECTED_BIT BIT0 //定义的事件组的0位，标志连接成功
#define WIFI_FAIL_BIT BIT1      //定义的事件组的1位，标志连接失败
/*
Note.
事件组标志位和espressif中的WIFI的事件组并不是一个东西，事件组标志位类似于原子操作，防止因为顺序错乱而导致程序问题。
而WIFI事件组只是WIFI在连接过程中的某种状态。
*/

static const char *TAG = "wifi station";

static int s_retry_num = 0;

/*WIFI在esp32中的连接是通过几组不同的事件处理来实现的，下述代码就是在作为STA连接WIFI中需要进行处理的事件。
后续如果有需要在拿到IP后才进行的操作只需要调用xEventGroupWaitBits就可以避免在没有拿到IP之前进行网络交互操作。
在WIFI因为某些原因断开后，事件标志位变为SYSTEM_EVENT_STA_DISCONNECTED，此时需要调用esp_wifi_connect()重新连接，并清除事件组标志位
*/
static void event_handler(void *arg, esp_event_base_t event_base,
                          int32_t event_id, void *event_data)
{
    //WIFI_EVENT_STA_START为初始事件状态，在此事情状态中，调用esp_wifi_connect()操作会连接到路由器（AP）上，
    //连接成功后，事件状态位会变为WIFI_EVENT_STA_CONNECTED，此时会调用DHCP进行请求IP，
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START)
    {
        esp_wifi_connect();
    }
    //如果连接失败，事件状态位会变为WIFI_EVENT_STA_DISCONNECTED，继续连接直到连接次数超过了最大允许值
    else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED)
    {
        if (s_retry_num < EXAMPLE_ESP_MAXIMUM_RETRY)
        {
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "retry to connect to the AP");
        }
        else
        {
            //超过了最大连接次数还连不上，则置位事件组的WIFI_FAIL_BIT标志
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
        ESP_LOGI(TAG, "connect to the AP fail");
    }
    //在拿到IP之后事件状态会变更为IP_EVENT_STA_GOT_IP。在这里我们使用了xEventGroupSetBits，设置事件组标志，
    else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP)
    {
        //将事件传递过来的数据格式化为ip格式
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        //将IP格式华为字符串打印出来
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        //成功获取了IP，置位事件组中的WIFI_CONNECTED_BIT标志
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

/*
下述代码是实现初始化WIFI的核心代码，
注：ESP_ERROR_CHECK用来检查返回值是否为ESP_OK。
    ESP_LOGI是espressif的格式化输出信息，其他还包括ESP_LOGE（错误信息）、ESP_LOGD（调试信息）。
*/
void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();   //FreeRTOS创建事件组，返回其句柄

    ESP_ERROR_CHECK(esp_netif_init());  //初始化底层TCP/IP堆栈

    ESP_ERROR_CHECK(esp_event_loop_create_default());   //创建默认事件循环，能够默认处理wifi、以太网、ip等事件
    esp_netif_create_default_wifi_sta();    //创建默认的 WIFI STA. 如果出现任何初始化错误，此API将中止。

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();    //指定需要初始化wifi底层的默认参数信息
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));   //用上述信息来初始化WIFI硬件。

	//向上面创建的的事件循环中注册事件和事件处理函数
    //注册一个事件句柄到WIFI_EVENT事件，如果发生任何事件（ESP_EVENT_ANY_ID），则执行回调函数event_handler，无额外参数传递
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    //注册一个事件句柄到IP_EVENT事件，如果发生获取了IP事件（IP_EVENT_STA_GOT_IP），则执行回调函数event_handler，无额外参数传递
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL));

    /*
    wifi_config是WIFI连接时的配置信息，作为STA时只需要考虑sta的参数信息，下述代码只是制定了最基本的ssid和password信息，
    除此之外，还可以指定bssid和channel等相关信息。
    */
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS,
            /* Setting a password implies station will connect to all security modes including WEP/WPA.
             * However these modes are deprecated and not advisable to be used. Incase your Access point
             * doesn't support WPA2, these mode can be enabled by commenting below line */
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,

            .pmf_cfg = {
                .capable = true,
                .required = false},
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));  //设置wifi模式为sta
    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config));    //设置wifi的配置参数
    ESP_ERROR_CHECK(esp_wifi_start());  //使用当前配置启动wifi工作

    ESP_LOGI(TAG, "wifi_init_sta finished.");

    /* Waiting until either the connection is established (WIFI_CONNECTED_BIT) or connection failed for the maximum
     * number of re-tries (WIFI_FAIL_BIT). The bits are set by event_handler() (see above) */
    //FreeRTOS事件组等待wifi连接完成（WIFI_CONNECTED_BIT置位）或者超过最大尝试次数后连接失败（WIFI_FAIL_BIT置位）
    //两种状态的置位都是在上边的event_handler()回调函数中执行的
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
                                           WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
                                           pdFALSE,
                                           pdFALSE,
                                           portMAX_DELAY);

    /* xEventGroupWaitBits() returns the bits before the call returned, hence we can test which event actually
     * happened. */
    //测试一下回调函数把哪一个状态置位了
    if (bits & WIFI_CONNECTED_BIT)
    {
        ESP_LOGI(TAG, "connected to ap SSID:%s password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    }
    else if (bits & WIFI_FAIL_BIT)
    {
        ESP_LOGI(TAG, "Failed to connect to SSID:%s, password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    }
    else
    {
        ESP_LOGE(TAG, "UNEXPECTED EVENT");
    }

    //注销一个事件句柄到WIFI_EVENT事件，如果发生任何事件（IP_EVENT_STA_GOT_IP），不再执行回调函数event_handler，无参数传递
    ESP_ERROR_CHECK(esp_event_handler_unregister(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler));
    //注销一个事件句柄到WIFI_EVENT事件，如果发生任何事件（ESP_EVENT_ANY_ID），不再执行回调函数event_handler，无参数传递
    ESP_ERROR_CHECK(esp_event_handler_unregister(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler));
    //FreeRTOS销毁wifi事件组
    vEventGroupDelete(s_wifi_event_group);
}

//void app_main()是espressif工程的入口，类似于C语言中的main
void app_main(void)
{
    //Initialize NVS
    esp_err_t ret = nvs_flash_init();   //始化NVS存储（非易失性flash）
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND)
    {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_LOGI(TAG, "ESP_WIFI_MODE_STA");
    wifi_init_sta();    //按照sta模式初始化wifi
}
```

