# ESP32学习笔记（4）wifi配网、参数保存、socket连接
>维护人员：**80后老男孩**  
>创建时间：2021-03-22  
>本文已发表于csdn博客[ESP32学习笔记（4）wifi配网、参数保存、socket连接](https://blog.csdn.net/weixin_41034400/article/details/115089342)

[TOC]

**注：该部分的学习需要参考esp-idf下的2个demo程序**：  

1. ..\\esp-idf\examples\\wifi\smart_config：这里展示了如何智能配网
2. ..\\esp-idf\examples\\protocols\sockets\\tcp_client：这里展示了如何用socket套接字做客户端的tcp连接，并收发数据

## 一.Smartconfig与Airkiss用于智能配网

上一章给模块wifi联网的例程中，模块连接的ssid和key是预先设定好的，但在实际的产品中不可能采用这种方式。能不能像手机一样，展示所有的wifi列表，然后点击输入密码连接，这种方式当然式可以的，问题就在于会增加产品的成本，而且在有些场景下不一定适合。所以这时候就引用了“空中配网技术”，比如smartconfig,airkiss,蓝牙辅助配网，阿里的零配等等。这里介绍ESP32如何通过smartconfig和airkiss两种方式连接上网络。

  ### 1.Smartconfig

智能硬件处于混杂模式下，监听网络中的所有报文；手机APP将当前连接的ssid和key编码到UDP报文中，通过广播或者组播的方式发送报文；智能硬件接收到UDP报文后解码，得到ssid和key,然后使用该组ssid和key去连接网络。这里需要用到esptouch手机app，下载地址在[乐鑫科技官网](https://www.espressif.com/zh-hans/products/software/esp-touch/resources)上有展示
### 2.Airkiss

  AirKiss是微信硬件平台提供的一种WIFI设备快速入网配置技术，要使用微信客户端的方式配置设备入网，需要设备支持AirKiss技术。Aiskiss的原理和smartconfig很类似，设备工作在混杂模式下，微信客户端发送包含ssid和key的广播包，设备收到广播包解码得到ssid和password。详细的可以参考[微信官方的介绍](https://iot.weixin.qq.com/wiki/document-7_1.html)。demo所用微信二维码如下（esp32进入智能配网模式后，直接用手机扫微信二维码即可完成对esp32的配网）：  
![](assets/004/002/004-1616406721924.png)



### 3.demo代码分析

esp32的wifi智能配网有两种模式，在代码中找到如下语句进行修改即可：
```c
1、ESP_ERROR_CHECK(esp_smartconfig_set_type(SC_TYPE_ESPTOUCH));    //仅能使用esptouch的手机app配置
2、ESP_ERROR_CHECK(esp_smartconfig_set_type(SC_TYPE_ESPTOUCH_AIRKISS));    //可以使用微信小程序配置
```
核心代码注释和分析如下
```c
//事件处理句柄
static void event_handler(void *arg, esp_event_base_t event_base,
                          int32_t event_id, void *event_data)
{
    //初始状态，启动智能配网任务
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START)
    {
        xTaskCreate(smartconfig_example_task, "smartconfig_example_task", 4096, NULL, 3, NULL);
    }
    //如果连接ap失败，事件状态位会变为WIFI_EVENT_STA_DISCONNECTED，继续连接，并清除连接标志
    else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED)
    {
        esp_wifi_connect();
        xEventGroupClearBits(s_wifi_event_group, CONNECTED_BIT);
    }
    //在拿到IP之后事件状态会变更为IP_EVENT_STA_GOT_IP。在这里我们使用了xEventGroupSetBits，设置事件组标志，
    else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP)
    {
        xEventGroupSetBits(s_wifi_event_group, CONNECTED_BIT);
    }
    //扫描完成
    else if (event_base == SC_EVENT && event_id == SC_EVENT_SCAN_DONE)
    {
        ESP_LOGI(TAG, "Scan done");
    }
    //扫描后发现wifi信道
    else if (event_base == SC_EVENT && event_id == SC_EVENT_FOUND_CHANNEL)
    {
        ESP_LOGI(TAG, "Found channel");
    }
    //获取了wifi的ssid和密码，则尝试连接ap
    else if (event_base == SC_EVENT && event_id == SC_EVENT_GOT_SSID_PSWD)
    {
        ESP_LOGI(TAG, "Got SSID and password");

        //格式化事件传递过来的参数，以便获取ssid和密码
        smartconfig_event_got_ssid_pswd_t *evt = (smartconfig_event_got_ssid_pswd_t *)event_data;
        wifi_config_t wifi_config; //定义一个wifi配置变量
        uint8_t ssid[33] = {0};
        uint8_t password[65] = {0};

        bzero(&wifi_config, sizeof(wifi_config_t));                                        //wifi配置全部清除
        memcpy(wifi_config.sta.ssid, evt->ssid, sizeof(wifi_config.sta.ssid));             //复制ssid到wifi配置变量
        memcpy(wifi_config.sta.password, evt->password, sizeof(wifi_config.sta.password)); //复制密码到wifi配置变量
        wifi_config.sta.bssid_set = evt->bssid_set;                                        //复制是否需要设置目标ap的mac地址到wifi配置变量
        if (wifi_config.sta.bssid_set == true)                                             //如果需要则复制
        {
            memcpy(wifi_config.sta.bssid, evt->bssid, sizeof(wifi_config.sta.bssid));
        }

        memcpy(ssid, evt->ssid, sizeof(evt->ssid));
        memcpy(password, evt->password, sizeof(evt->password));
        ESP_LOGI(TAG, "SSID:%s", ssid);
        ESP_LOGI(TAG, "PASSWORD:%s", password);

        ESP_ERROR_CHECK(esp_wifi_disconnect());                              //断开wifi
        ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config)); //设置wifi的sta参数
        ESP_ERROR_CHECK(esp_wifi_connect());                                 //连接wifi

        //保存wifi配网参数到nvs中
        saveWifiConfig(&wifi_config);
    }
    //获取后应答完成
    else if (event_base == SC_EVENT && event_id == SC_EVENT_SEND_ACK_DONE)
    {
        xEventGroupSetBits(s_wifi_event_group, ESPTOUCH_DONE_BIT); //事件组职位表示智能配网完成
    }
}

//智能配网任务----------------------------------------
static void smartconfig_example_task(void *parm)
{
    EventBits_t uxBits;
    //ESP_ERROR_CHECK(esp_smartconfig_set_type(SC_TYPE_ESPTOUCH));    //仅能使用esptouch的手机app配置
    ESP_ERROR_CHECK(esp_smartconfig_set_type(SC_TYPE_ESPTOUCH_AIRKISS)); //可以使用微信小程序配置
    smartconfig_start_config_t cfg = SMARTCONFIG_START_CONFIG_DEFAULT(); //智能配网用默认的参数
    ESP_ERROR_CHECK(esp_smartconfig_start(&cfg));                        //智能配网启动
    while (1)
    {
        //等待连接成功或配网结束的事件到来
        uxBits = xEventGroupWaitBits(s_wifi_event_group, CONNECTED_BIT | ESPTOUCH_DONE_BIT, true, false, portMAX_DELAY);
        if (uxBits & CONNECTED_BIT) //连接成功
        {
            ESP_LOGI(TAG, "WiFi Connected to ap");
        }
        if (uxBits & ESPTOUCH_DONE_BIT) //配网结束
        {
            ESP_LOGI(TAG, "smartconfig over");
            esp_smartconfig_stop(); //结束配网
            vTaskDelete(NULL);      //自删除任务
        }
    }
}
```

以上的过程就完成了智能配网。

## 二.保存和读取参数

智能配网结束后，参数并没有保存，下次启动又得再次配网，那么就需要我们保存一下，保存在esp32的nvs存储中即可，保存和再次读取的核心代码如下：

```c
//保存wifi配置参数结构体变量wifi_config到nvs------------------------------------------------
static void saveWifiConfig(wifi_config_t *wifi_config)
{
    nvs_handle my_handle;
    nvs_open("WIFI_CONFIG", NVS_READWRITE, &my_handle); //打开

    ESP_ERROR_CHECK(nvs_set_blob(my_handle, "wifi_config", wifi_config, sizeof(wifi_config_t)));

    ESP_ERROR_CHECK(nvs_commit(my_handle)); //提交
    nvs_close(my_handle);                   //退出
}

//从nvs中读取wifi配置到给定的sta_config结构体变量
static esp_err_t readWifiConfig(wifi_config_t *sta_config)
{
    nvs_handle my_handle;
    nvs_open("WIFI_CONFIG", NVS_READWRITE, &my_handle); //打开

    // sta_config = {
    //     .sta = {
    //         .ssid = "ssid",
    //         .password = "password",
    //         .bssid_set = false}}

    uint32_t len = sizeof(wifi_config_t);
    esp_err_t err = nvs_get_blob(my_handle, "wifi_config", sta_config, &len);

    nvs_close(my_handle); //退出

    return err;
}
```

按照以上操作保存了以后，下次再上电开机，直接读取就是用就可以了，不用再次配网。

假如我们想清除网络配置，比如我们到了一个新的地方，网络环境变了，需要配置，可以采用长按键的方式清除，重新配置，清除nvs保存参数的核心代码如下：

```c
ESP_ERROR_CHECK(nvs_flash_erase());
```

## 三.socket连接（tcp client）

在联网操作中，最常用的就是tcp连接的客户端模式。

这里直接参看历程即可，过程比较明了，其中需要注意其中用到了一个公共组件example_connect()：

```c
    /* This helper function configures Wi-Fi or Ethernet, as selected in menuconfig.
     * Read "Establishing Wi-Fi or Ethernet Connection" section in
     * examples/protocols/README.md for more information about this function.
     */
    //注意去读example_connect的代码，里边有启动时wifi的一些配置，比如连接的wifi、用户名密码之类的
    //这些配置在工程配置里有选项，比如选择wifi还是以太网之类的
    ESP_ERROR_CHECK(example_connect());
```

这个公共组件位于：..\\esp-idf\\examples\\common_components\\protocol_examples_common文件夹之下，经过简单配置就可以选择我们要使用的网络、比如wifi还是以太网，是否ipv6及一些参数，这些东西就不再需要我们编写代码，其中如果你希望wifi的ssid和密码使用保存好的而不是配置的话，进入其中源代码文件connect.c下的esp_err_t example_connect(void)和void start(void)函数进行改写即可，建议改写这两个函数成为能够接收传递参数的形式进行联网，参数中直接把wifi配置传递过来。

