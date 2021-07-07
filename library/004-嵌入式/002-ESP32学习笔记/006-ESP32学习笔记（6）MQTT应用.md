# ESP32学习笔记（6）MQTT应用
>维护人员：**80后老男孩**  
>创建时间：2021-03-24  
>本文已发表于csdn博客[ESP32学习笔记（6）MQTT应用](https://blog.csdn.net/weixin_41034400/article/details/115166413)

[TOC]

**注：该部分的学习需要参考esp-idf下的demo程序**：  

1. ..\\esp-idf\\examples\\protocols\\mqtt\\tcp

#  MQTT原理和工作过程讲解  
MQTT(消息队列遥测传输)是ISO 标准(ISO/IEC PRF 20922)下基于发布/订阅范式的消息协议。它工作在 TCP/IP协议族上，是为硬件性能低下的远程设备以及网络状况糟糕的情况下而设计的发布/订阅型消息协议，为此，它需要一个消息中间件 （服务器）。

通过MQTT协议，目前已经扩展出了数十个MQTT服务器端程序，可以通过PHP，JAVA，Python，C，C#等系统语言来向MQTT发送相关消息。

此外，国内很多企业都广泛使用MQTT作为Android手机客户端与服务器端推送消息的协议。MQTT由于开放源代码，耗电量小等特点。在物联网领域，传感器与服务器的通信，信息的收集，MQTT都可以作为考虑的方案之一。在未来MQTT会进入到我们生活的各个方面。

**所以，如果物联网设备想要联网，MQTT是不二选择。**

## MQTT特点
MQTT协议是为大量计算能力有限，且工作在低带宽、不可靠的网络的远程传感器和控制设备通讯而设计的协议，它具有以下主要的几项特性：

使用发布/订阅消息模式，提供一对多的消息发布，解除应用程序耦合；
对负载内容屏蔽的消息传输；
使用 TCP/IP 提供网络连接；
有三种消息发布服务质量：
“至多一次”，消息发布完全依赖底层 TCP/IP 网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。
“至少一次”，确保消息到达，但消息重复可能会发生。
“只有一次”，确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。
小型传输，开销很小（固定长度的头部是 2 字节），协议交换最小化，以降低网络流量；

这就是为什么在介绍里说它非常适合物联网领域，要知道嵌入式设备的运算能力和带宽都相对薄弱，使用这种协议来传递消息再适合不过了。

## MQTT实现方式　  　
实现MQTT协议需要客户端和服务器端通讯完成，在通讯过程中，MQTT协议中有三种身份：发布者（Publish）、代理（Broker）（服务器）、订阅者（Subscribe）。其中，消息的发布者和订阅者都是客户端，消息代理是服务器，消息发布者可以同时是订阅者。

MQTT传输的消息分为：主题（Topic）和负载（payload）两部分：  
Topic，可以理解为消息的类型，订阅者订阅（Subscribe）后，就会收到该主题的消息内容（payload）；  
payload，可以理解为消息的内容，是指订阅者具体要使用的内容。  
![](assets/004/002/005-1616553837322.png)  
MQTT服务器的主要工作是数据分发，没有数据保存功能。  
可以订阅自己发布的主题，服务器就是回发测试。  
MQTT让逻辑变得更清晰，需要什么订阅什么。  
走标准化流程，解放了私有协议制定、实现、调试、测试一整套复杂的流程。  

## Windows搭建本地MQTT服务器[参考原文](https://blog.csdn.net/weixin_41572450/article/details/82558774)

为了方便测试和调试，我们可以在电脑上搭建一个MQTT的服务，目前主流的Broker有以下3个：

Mosquitto：https://mosquitto.org/  
VerneMQ：https://vernemq.com/  
EMQTT：http://emqtt.io/    
凭喜好选择一个搭建即可。

# MQTT的DEMO注释理解  
MQTT是基于TCP的，关于是如何封装的，乐鑫已经整理好了，留给我们应用层使用的接口相对较少，内部如何封装，乐鑫怎么封装，标准怎么封装，有时间可以仔细阅读乐鑫源码和标准文档。只说demo。

## MQTT配置参数结构体  
首先要理解和分析mqtt配置参数结构体的成员变量的定义，用于初始化mqtt时根据用户需求配置  
```c
typedef struct {
mqtt_event_callback_t event_handle; /*回调*/
const char *host; /*!< MQTT 服务器域名(ipv4 as string)*/
const char *uri; /*!< MQTT 服务器域名 */
uint32_t port; /*!< MQTT服务器端口*/
const char *client_id; /*MQTT Client的名字默认是ESP32_加上MAC后3hex*/
const char *username; /*MQTT用户名*/
const char *password; /*MQTT密码*/
const char *lwt_topic; /*!< LWT主题，默认为空*/
const char *lwt_msg; /*!< LWT信息，默认为空*/
int lwt_qos; /*!< LWT消息质量*/
int lwt_retain; /*!< LWT保留消息标志*/
int lwt_msg_len; /*!< LWT消息长度*/
int disable_clean_session; /*!< mqtt clean session,默认为真*/
int keepalive; /*MQTT心跳，默认120秒 */
bool disable_auto_reconnect; /*错误，断开后重连，true不连*/
void *user_context; /*用户信息 */
int task_prio; /*!< MQTT任务优先级，默认为5,可以在make menuconfig中修改*/
int task_stack; /*!< MQTT 任务堆栈大小,默认6144 bytes,可以在make menuconfig中修改*/
int buffer_size; /*!< MQTT收发缓存，默认1024 */
const char *cert_pem; /*指向用于服务器验证（使用SSL）的PEM格式的证书数据的指针，默认值为空，不需要验证服务器 */
const char *client_cert_pem; /*指向用于SSL相互身份验证的PEM格式的证书数据的指针，默认值为空，如果不需要相互身份验证，则不需要。如果不为空，还必须提供“客户机密钥”。*/
const char *client_key_pem; /*指向用于SSL相互身份验证的PEM格式的私钥数据的指针，默认值为空，如果不需要相互身份验证，则不需要。如果不为空，还必须提供“client-cert-pem”。*/
esp_mqtt_transport_t transport; /*覆盖URI传输*/
} esp_mqtt_client_config_t;
```
## 声明MQTT配置
```c
static const char *Host = "mqtt://mqtt.eclipse.org";  //mqtt服务器主机mqtt://mqtt.eclipse.org
static const int Port = 1883;    //mqtt服务器主机端口
static const char *ClientId = "f1612828";   //本设备id
static const char *Username = "user";  //接入用户名
static const char *Password = "pass";   //接入密码
static const char *Substopic = "device/";   //订阅的主题
static const char *Pubstopic = "device/server";  //发布的主题
```
## 核心代码注释理解  
### MQTT的事件处理回调函数
```c
static esp_err_t mqtt_event_handler_cb(esp_mqtt_event_handle_t event)
{
    esp_mqtt_client_handle_t client = event->client;
    int msg_id;

    // your_context_t *context = event->context;
    switch (event->event_id)
    {
    case MQTT_EVENT_CONNECTED:  //MQTT连上事件
        ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
        //MQTT Client发布主题函数，服务质量qos1，发布的数据是data-3
        //msg_id = esp_mqtt_client_publish(client, "/topic/qos1", "data_3", 0, 1, 0);
        msg_id = esp_mqtt_client_publish(client, Pubstopic, "hello mqtt from esp32!", 0, 1, 0);
        ESP_LOGI(TAG, "sent publish successful, msg_id=%d", msg_id);
    
        //msg_id = esp_mqtt_client_subscribe(client, "/topic/qos0", 0);   //MQTT Client订阅主题函数，服务质量qos0
        msg_id = esp_mqtt_client_subscribe(client, Substopic, 0);   //MQTT Client订阅主题函数，服务质量qos0
        ESP_LOGI(TAG, "sent subscribe successful, msg_id=%d", msg_id);
    
        // msg_id = esp_mqtt_client_subscribe(client, "/topic/qos1", 1);   //MQTT Client订阅主题函数，服务质量qos1
        // ESP_LOGI(TAG, "sent subscribe successful, msg_id=%d", msg_id);
    
        // msg_id = esp_mqtt_client_unsubscribe(client, "/topic/qos1");    //MQTT Client取消订阅主题函数
        // ESP_LOGI(TAG, "sent unsubscribe successful, msg_id=%d", msg_id);
        break;
    case MQTT_EVENT_DISCONNECTED:   //MQTT断开连接事件
        ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
        break;
    
    case MQTT_EVENT_SUBSCRIBED: //MQTT发送订阅成功事件
        ESP_LOGI(TAG, "MQTT_EVENT_SUBSCRIBED, msg_id=%d", event->msg_id);
        //回个消息，表示成功了
        //msg_id = esp_mqtt_client_publish(client, "/topic/qos0", "data", 0, 0, 0);
        msg_id = esp_mqtt_client_publish(client, Pubstopic, "hello mqtt，now subscribed ok!", 0, 0, 0);
        ESP_LOGI(TAG, "sent publish successful, msg_id=%d", msg_id);
        break;
    case MQTT_EVENT_UNSUBSCRIBED:   //MQTT取消订阅事件
        ESP_LOGI(TAG, "MQTT_EVENT_UNSUBSCRIBED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_PUBLISHED:  //MQTT发布成功事件
        ESP_LOGI(TAG, "MQTT_EVENT_PUBLISHED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_DATA:   //MQTT接收数据事件
        ESP_LOGI(TAG, "MQTT_EVENT_DATA");
        printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
        printf("DATA=%.*s\r\n", event->data_len, event->data);
        break;
    case MQTT_EVENT_ERROR:  //MQTT错误事件
        ESP_LOGI(TAG, "MQTT_EVENT_ERROR");
        break;
    default:
        ESP_LOGI(TAG, "Other event id:%d", event->event_id);
        break;
    }
    return ESP_OK;
}
```
### 启动MQTT应用
```c
static void mqtt_app_start(void)
{
    esp_mqtt_client_config_t mqtt_cfg = {
        .host = Host,   //MQTT服务器地址
        //.uri = Host,    //MQTT服务器域名
        .port = Port,   //MQTT服务器端口
        .username = Username,   //接入用户名
        .password = Password,   //接入密码
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);  //MQTT Client初始化
    //注册mqtt事件，mqtt事件句柄是client
    //如果发生任何事件（ESP_EVENT_ANY_ID），则执行回调函数mqtt_event_handler，额外参数传递client句柄的内容
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
    esp_mqtt_client_start(client);  //MQTT Client启动
}
```
