# ESP32学习笔记（7）蓝牙GATT服务应用
>维护人员：**80后老男孩**  
>创建时间：2021-03-24  
>本文已发表于csdn博客[ESP32学习笔记（7）蓝牙GATT服务应用](https://blog.csdn.net/weixin_41034400/article/details/115189286)
[TOC]

**注：该部分的学习需要参考esp-idf下的demo程序**：  

1. ..\\esp-idf\\examples\\bluetooth\\bluedroid\\ble\\gatt_server

# 一、蓝牙4.0简介

## 概述
蓝牙4.0包括传统蓝牙和低功耗蓝牙BLE两部分，其中BLE（[Bluetooth Low Energy](http://en.wikipedia.org/wiki/Bluetooth_low_energy)）是蓝牙4.0的核心，主打功能是快速搜索，快速连接，超低功耗保持连接和传输数据，弱点是数据传输速率低，由于BLE的低功耗特点，因此普遍用于穿戴设备。Android 4.3才开始支持BLE API，所以需要在蓝牙4.0和Android 4.3及其以上的系统上进行开发、运行。比如一个Android系统pad和多个ble终端设备进行通讯，pad作为客户端，ble设备则为服务端。

## BLE通信原理

BLE分为Service（服务）、Characteristic（特征）、Descriptor（描述）三部分，这三部分都由UUID作为唯一标示符。一个蓝牙4.0的终端可以包含多个Service，一个Service可以包含多个Characteristic，一个Characteristic包含一个Value和多个Descriptor，一个Descriptor包含一个Value。一般来说，Characteristic是pad与BLE终端设备交换数据的关键，Characteristic有较多的跟权限相关的字段。为了找到Characteristic我们需要先理解一下几个概念：

**1)** **GATT**：可以把它看成pad与BLE终端设备建立通信的一个管道，有了这个管道，才有了通信的前提。

**2)** **Service（服务）**：蓝牙设备的服务，在这里我们把它比喻成班级。而将ble设备比喻成学校，一个学校里面可以有很多班级，也就是说每台BLE终端设备拥有多个服务，班级（各个服务）之间通过UUID（唯一标识符）区别。

**3)** **Characteristic（特征）**：蓝牙设备所拥有的特征，它就是我们所找的Characteristic，我们做的所有事情，目的就是为了得到它。在这里我们把它比喻成学生，一个班级里面有很多个学生，也就是说每个服务下拥有多个特征，学生（各个特征）之间通过UUID（唯一标识符）区别。

**4)** **Descriptor（描述）**：对Characteristic的描述，例如范围、计量单位等。

简而言之，当我们想要用pad与BLE设备进行通信时，实际上也就相当于我们要去找一个学生交流，首先我们需要搭建一个管道，也就是我们需要先获取得到一个GATT，其次我们需要知道这个学生在哪一个班级，学号是什么，这也就是我们所说的serviceUUID和charUUID，利用这两个信息可以找到他。

这里还需要注意一下，Characteristic（特征）就像一个中介一样，在pad和BLE终端设备之间帮助这两者传递着信息，pad需要经过它发送数据，由它传递到BLE设备上，而BLE设备上的返回信息，也是先传递到它那边，然后pad再从它那边进行读取。

## 蓝牙4.0通信实现过程

1. 扫描蓝牙BLE终端设备，对应esp32就是广播给大家供扫描  
2. 连接蓝牙BLE终端设备，pad扫描到后去连接  
3. 启动服务发现，连接到esp32后获取相应的服务。  
连接成功后，我们就要去寻找我们所需要的服务，这里需要先启动服务发现。
4. 获取Characteristic  
    之前我们说过，我们的最终目的就是获取Characteristic来进行通信，正常情况下，我们可以从硬件工程师那边得到serviceUUID和characteristicUUID，也就是我们所比喻的班级号和学号，以此来获得我们的characteristic。  
5. 开始通信  
    我们在得到Characteristic后，就可以开始读写操作进行通信了。  
    a. 对于读操作来说，读取BLE终端设备返回的数据会通过回调方法mGattCallback中的onCharacteristicChanged函数返回。  
    b. 对于写操作来说，可以通过向Characteristic写入指令以此来达到控制BLE终端设备的目的

# 二、ESP32 BLE的GATT服务

## GATT简介
GATT是用Attribute Protocal(属性协议)定义的一个service(服务)框架。这个框架定义了Services以及它们的Characteristics的格式和规程。规程就是定义了包括发现、读、写、通知、指示以及配置广播的characteristics。  

为实现配置文件(Profile)的设备定义了两种角色：Client(客户端)、Server(服务器)。esp32的ble一般就处于Server模式。

一旦两个设备建立了连接，GATT就开始发挥效用，同时意味着GAP协议管理的广播过程结束了。

## profile

profile 可以理解为一种规范，一个标准的通信协议中，存于从机(Server)中。蓝牙组织规定了一些标准的Profile。每个profile中包含多个Service，每个service代表从机的一种能力

一个GATT 服务器应用程序架构(由Application Profiles组织起来)如下: 

![](assets/004/002/007-1.png)

每个Application Profile描述了一个方法来对为一个客户端应用程序设计的功能进行分组，例如在智能手机或平板电脑上运行的移动应用程序。

每个Profile定义为一个结构体，结构体成员依赖于该Application Profile 实现的services服务和characteristic特征。结构体成员还包括GATT interface(GATT 接口)、Application ID(应用程序ID)和处理profile事件的回调函数。

每个profile包括GATT interface(GATT 接口)、Application ID(应用程序ID)、 Connection ID(连接ID)、Service Handle(服务句柄)、Service ID(服务ID)、Characteristic handle(特征句柄)、Characteristic UUID(特征UUID)、ATT权限、Characteristic Properties、描述符句柄、描述符UUID。

如果Characteristic 支持通知(notifications)或指示(indicatons)，它就必须是实现CCCD(Client Characteristic  Configuration Descriptor)----这是额外的ATT。描述符有一个句柄和UUID。如：

```c
struct gatts_profile_inst {
    esp_gatts_cb_t gatts_cb;       //GATT的回调函数
    uint16_t gatts_if;             //GATT的接口
    uint16_t app_id;               //应用的ID
    uint16_t conn_id;              //连接的ID
    uint16_t service_handle;       //服务Service句柄
    esp_gatt_srvc_id_t service_id; //服务Service ID
    uint16_t char_handle;          //特征Characteristic句柄
    esp_bt_uuid_t char_uuid;       //特征Characteristic的UUID
    esp_gatt_perm_t perm;          //特征属性Attribute 授权
    esp_gatt_char_prop_t property; //特征Characteristic的特性
    uint16_t descr_handle;         //描述descriptor句柄
    esp_bt_uuid_t descr_uuid;      //描述descriptorUUID    
};
```

Application Profile存储在数组中，并分配相应的回调函数gatts_profile_a_event_handler() 和 gatts_profile_b_event_handler()。

**在GATT客户机上的不同的应用程序使用不同的接口，用gatts_if参数来表示。在初始化时，gatts-if参数初始化为ESP_GATT_IF_NONE，在注册客户端时(如注册profile A的客户端时gatt_if = 0,在注册profile B的客户端时gatt_if = 1)**

```c
/* One gatt-based profile one app_id and one gatts_if, this array will store the gatts_if returned by ESP_GATTS_REG_EVT */
static struct gatts_profile_inst gl_profile_tab[PROFILE_NUM] = {
    [PROFILE_A_APP_ID] = {
        .gatts_cb = gatts_profile_a_event_handler,
        .gatts_if = ESP_GATT_IF_NONE,       /* Not get the gatt_if, so initial is ESP_GATT_IF_NONE */
    },
    [PROFILE_B_APP_ID] = {
        .gatts_cb = gatts_profile_b_event_handler,/* This demo does not implement, similar as profile A */
        .gatts_if = ESP_GATT_IF_NONE,       /* Not get the gatt_if, so initial is ESP_GATT_IF_NONE */
    },
};
```

这是两个元素的数组。可以用Application ID来注册Application Profiles，Application ID是由应用程序分配的用来标识每个Profile。 通过这种方法，可以在一个Server中拥有多个Application Profile。

## demo程序gatt启动流程

一个gatt服务启动的主流程写在了demo的app_main中，注释理解如下：

```c
.........
//esp_bt_controller_config_t是蓝牙控制器配置结构体，这里使用了一个默认的参数
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    //初始化蓝牙控制器，此函数只能被调用一次，且必须在其他蓝牙功能被调用之前调用
    ret = esp_bt_controller_init(&bt_cfg);
    if (ret) {
        ESP_LOGE(GATTS_TAG, "%s initialize controller failed: %s\n", __func__, esp_err_to_name(ret));
        return;
    }

    //使能蓝牙控制器，mode是蓝牙模式，如果想要动态改变蓝牙模式不能直接调用该函数，
    //应该先用disable关闭蓝牙再使用该API来改变蓝牙模式
    ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
    if (ret) {
        ESP_LOGE(GATTS_TAG, "%s enable controller failed: %s\n", __func__, esp_err_to_name(ret));
        return;
    }
    //初始化蓝牙并分配系统资源，它应该被第一个调用
    /*
    蓝牙栈bluedroid stack包括了BT和BLE使用的基本的define和API
    初始化蓝牙栈以后并不能直接使用蓝牙功能，
    还需要用FSM管理蓝牙连接情况
    */
    ret = esp_bluedroid_init();
    if (ret) {
        ESP_LOGE(GATTS_TAG, "%s init bluetooth failed: %s\n", __func__, esp_err_to_name(ret));
        return;
    }
    //使能蓝牙栈
    ret = esp_bluedroid_enable();
    if (ret) {
        ESP_LOGE(GATTS_TAG, "%s enable bluetooth failed: %s\n", __func__, esp_err_to_name(ret));
        return;
    }

    //建立蓝牙的FSM（有限状态机）
    //这里使用回调函数来控制每个状态下的响应，需要将其在GATT和GAP层的回调函数注册
    /*gatts_event_handler和gap_event_handler处理蓝牙栈可能发生的所有情况，达到FSM的效果*/
    ret = esp_ble_gatts_register_callback(gatts_event_handler);
    if (ret){
        ESP_LOGE(GATTS_TAG, "gatts register error, error code = %x", ret);
        return;
    }
    ret = esp_ble_gap_register_callback(gap_event_handler);
    if (ret){
        ESP_LOGE(GATTS_TAG, "gap register error, error code = %x", ret);
        return;
    }

    //下面创建了BLE GATT服务A，相当于1个独立的应用程序
    ret = esp_ble_gatts_app_register(PROFILE_A_APP_ID);
    if (ret){
        ESP_LOGE(GATTS_TAG, "gatts app register error, error code = %x", ret);
        return;
    }
    //下面创建了BLE GATT服务B，相当于1个独立的应用程序
    ret = esp_ble_gatts_app_register(PROFILE_B_APP_ID);
    if (ret){
        ESP_LOGE(GATTS_TAG, "gatts app register error, error code = %x", ret);
        return;
    }
    /*
    设置了MTU的值(经过MTU交换，从而设置一个PDU中最大能够交换的数据量)。
    例如：主设备发出一个1000字节的MTU请求，但是从设备回应的MTU是500字节，那么今后双方要以较小的值500字节作为以后的MTU。
    即主从双方每次在做数据传输时不超过这个最大数据单元。
    */
    esp_err_t local_mtu_ret = esp_ble_gatt_set_local_mtu(500);
    if (local_mtu_ret){
        ESP_LOGE(GATTS_TAG, "set local  MTU failed, error code = %x", local_mtu_ret);
    }
.......
```

## 几个需要注意的重要事项

### 1.GATT回调函数注册esp_ble_gatts_register_callback

其作用就是建立了蓝牙GATT的FSM（有限状态机），callback回调函数处理从BLE堆栈推送到应用程序的所有事件

回调函数的参数:
event: esp_gatts_cb_event_t  这是一个枚举类型，表示调用该回调函数时的事件(或蓝牙的状态)

gatts_if: esp_gatt_if_t (uint8_t) 这是GATT访问接口类型，通常在GATT客户端上不同的应用程序用不同的gatt_if(不同的Application profile对应不同的gatts_if) ，**调用esp_ble_gatts_app_register()时，注册Application profile 就会有一个gatts_if**。

param: esp_ble_gatts_cb_param_t 指向回调函数的参数，是个联合体类型，不同的事件类型采用联合体内不同的成员结构体。


### 2.注册创建GATT服务A：esp_ble_gatts_app_register(PROFILE_A_APP_ID);

当调用esp_ble_gatts_app_register()注册一个应用程序Profile(Application Profile)，将触发ESP_GATTS_REG_EVT事件，除了可以完成对应profile的gatts_if的注册,还可以调用esp_bel_create_attr_tab()来创建profile Attributes 表或创建一个服务esp_ble_gatts_create_service()

demo中的gatts_event_handler()回调函数---调用esp_ble_gatts_app_register(1)，触发ESP_GATTS_REG_EVT时，完成对每个profile 的gatts_if 的注册
```c
gl_profile_tab[param->reg.app_id].gatts_if = gatts_if;
```
如果gatts_if == 某个Profile的gatts_if时，调用对应profile的回调函数处理事情。

```c
if (gatts_if == ESP_GATT_IF_NONE||gatts_if == gl_profile_tab[idx].gatts_if) 			{
                if (gl_profile_tab[idx].gatts_cb) {
                    gl_profile_tab[idx].gatts_cb(event, gatts_if, param);
                }
            }
```
### GATT服务状态机的一般过程：

以gatts_server这个demo为例，讲解GATT状态机的一般过程:

**与客户机建立连接之前**
ESP_GATTS_REG_EVT---->ESP_GATTS_CREATE_EVT---->ESP_GATTS_START_EVT---->ESP_GATTS_ADD_CHAR_EVT--->ESP_GATTS_ADD_CHAR_DESCR_EVT

注册->创建->启动->添加特征->添加特征描述

**有Client客户端开始连接之后**

CONNECT_EVT---->ESP_GATTS_MTU_EVT--->GATT_WRITE_EVT--->ESP_GATTS_CONF_EVT-->GATT_READ_EVT

在Demo的ESP_GATTS_REG_EVT事件中，调用esp_ble_gap_set_device_name(char *)来设置蓝牙设备名字；  
调用esp_ble_gap_config_adv_data(esp_ble_adv_data_t *adv_data)来配置广播数据；  
最后调用esp_ble_gatts_create_service(esp_gatt_if_t gatts_if, esp_gatt_srvc_id_t *service_id, uint16_t num_handle)指定gatts_if和service_id来创建服务<实际调用btc_transfer_context()来完成服务的创建和调用回调函数>。

服务创建完成就会触发回调函数向profile报告状态和服务ID。Service_id对于后面添加included serivces 和 characteristics和descriptor都要用到。触发ESP_GATTS_CREATE_EVT事件

在Demo的ESP_GATTS_CREATE_EVT中调用esp_ble_gatts_start_service(uint16_t service_handle)来启动服务；  
再调用esp_ble_gatts_add_char(uint16_t service_handle, esp_bt_uuid_t *char_uuid, esp_gatt_perm_t perm, esp_gatt_char_prop_t property, esp_attr_value_t *char_val, esp_attr_control_t *control)来添加特性(特征的UUID， 特征值描述符属性权限， 特征属性、特征值、属性响应控制字节)。

触发ESP_GATTS_START_EVT和ESP_GATTS_ADD_CHAR_EVT事件

在ESP_GATTS_ADD_CHAR_EVT事件中，获取特征值调用esp_err_tesp_ble_gatts_add_char_descr(uint16_t service_handle, esp_bt_uuid_t *descr_uuid, esp_gatt_perm_tperm, esp_attr_value_t *char_descr_val, esp_attr_control_t *control)来添加特征描述符  

### 3 建立连接之前的GATT状态机
### 3.1 创建服务 creating services
在触发ESP_GATTS_REG_EVT时，除了创建表还可以创建服务，调用esp_ble_gatts_create_service来创建服务。

```c
esp_err_t esp_ble_gatts_create_service(esp_gatt_if_t gatts_if,
                                       esp_gatt_srvc_id_t *service_id, uint16_t num_handle);
```
作用:创建一个service。当一个service创建成功(done)后，ESP_CREATE_SERVICE_EVT事件触发回调函数被调用，该回调函数报告了profile的stauts和service ID。当要添加include service和 characteristics//descriptors入服务service，Service ID在回调函数中用到。

参数:gatts_if：GATT 服务器访问接口

service_id: 服务UUID相关信息

**num_handle:该服务所需的句柄数 service、characteristic declaration、 characteristic value、characteristic description 的句柄数总和。Demo中用的是4（1+3），如果有两个特征，则为7（1+3+3）.**

service_id服务ID参数包括

is_primary参数当前服务是否是首要的；---------服务声明中的Attribute Type 0x2800---Primary/0x2801---Secondary

id参数UUID的信息(包括uuid 和服务实例instance id)

其中uuid是UUID的信息包括UUID的长度(16bit/32bit/128bit)及UUID具体值。

在Demo中，服务被定义为16bit的UUID的主服务。服务ID以实例ID为0，UUID为0x00FF来初始化，len为2个字节。

服务实例ID是用来区分在同一个Profile中具有相同UUID的多个服务。Application Profile中拥有相同UUID的两个服务，需要用不同的实例ID来引用。

如下示例展示了一个Service的创建。
```c
 switch (event) {
    case ESP_GATTS_REG_EVT:
         ESP_LOGI(GATTS_TAG, "REGISTER_APP_EVT, status %d, app_id %d\n", param->reg.status, param->reg.app_id);
         gl_profile_tab[PROFILE_B_APP_ID].service_id.is_primary = true;
         gl_profile_tab[PROFILE_B_APP_ID].service_id.id.inst_id = 0x00;
         gl_profile_tab[PROFILE_B_APP_ID].service_id.id.uuid.len = ESP_UUID_LEN_16;
         gl_profile_tab[PROFILE_B_APP_ID].service_id.id.uuid.uuid.uuid16 = GATTS_SERVICE_UUID_TEST_B;

         esp_ble_gatts_create_service(gatts_if, &gl_profile_tab[PROFILE_B_APP_ID].service_id, GATTS_NUM_HANDLE_TEST_B);
         break;
…
}
```
 一个profile有一个primary服务，服务有UUID以及服务实例的ID。

### 3.2 启动服务并创建Characteristics
 当一个服务service创建成功(done)后，由该profile GATT handler 管理的 ESP_GATTS_CREATE_EVT事件被触发，在这个事件可以启动服务和添加characteristics到服务中。调用esp_ble_gatts_start_service来启动指定服务。

Characteristic是在GATT规范中最小的逻辑数据单元，由一个Value和多个描述特性的Desciptior组成。**实际上，在与蓝牙设备打交道，主要就是读写Characteristic的value来完成**。同样的，Characteristic也是通过16bit或128bit的UUID唯一标识。

**我们根据蓝牙设备的协议用对应的Characteristci进行读写即可达到与其通信的目的**。

ESP_GATTS_CREATE_EVT事件中回调函数参数的类型为gatts_create_evt_param(包括操作函数、servic的句柄、服务的id<UUID+其他信息>) 如下所示。

 ```c
 /**

   * @brief ESP_GATTS_CREATE_EVT
     /
     struct gatts_create_evt_param {
     esp_gatt_status_t status;       /*!< Operation status */
     uint16_t service_handle;        /*!< Service attribute handle */
     esp_gatt_srvc_id_t service_id;  /*!< Service id, include service uuid and other information */
         } create;                           /*!< Gatt server callback param of ESP_GATTS_CREATE_EVT */
 ```

esp_err_t esp_ble_gatts_start_service(uint16_t service_handle)
 esp_ble_gatts_start_service()这个函数启动一个服务。

参数: service_handle-------要被启动的服务句柄。

首先，由BLE堆栈生成生成的服务句柄(service handle)存储在Profile表中，应用层将用服务句柄来引用这个服务。调用esp_ble_gatts_start_service()和先前产生服务句柄来启动服务。

esp_err_t esp_ble_gatts_add_char(uint16_t service_handle, esp_bt_uuid_t *char_uuid, 

esp_gatt_perm_tperm, esp_gatt_char_prop_tproperty, esp_attr_value_t *char_val, 

esp_attr_control_t *control)

这样ESP_ATTS_START_EVT事件触发时，将打印输出信息启动的Service Handle之类的信息。

添加特征到service中，调用esp_ble_gatts_add_char()来添加characteristics连同characteristic 权限和property(属性)到服务service中。

**解释一下property**

Characteristic Properties这个域(bit控制)决定了Characteristic Value如何使用、Characteristic descriptors 如何访问。只要下标中对应bit被设备，那么对应描述的action就被允许。

在Demo中，特征uuid的len为2字节，uuid为0x00FF，权限是可读可写，属性的读、写、通知bit置1，且设置特征值，
```c
gl_profile_tab.char_uuid.len = ESP_UUID_LEN_16;
gl_profile_tab.char_uuid.uuid.uuid16 = GATTS_CHAR_UUID_TEST_A;
a_property = ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE | ESP_GATT_CHAR_PROP_BIT_NOTIFY;
        esp_err_t add_char_ret = esp_ble_gatts_add_char(gl_profile_tab.service_handle, &gl_profile_tab.char_uuid,
                                                        ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                                                        a_property,
                                                        &gatts_demo_char1_val, NULL);
```

#### 3.2.1 添加Characteristic Value declaration ATT
添加Characteristic的接口

```c
esp_err_t esp_ble_gatts_add_char(uint16_t service_handle, esp_bt_uuid_t *char_uuid, 

esp_gatt_perm_t perm, esp_gatt_char_prop_t property, esp_attr_value_t *char_val, esp_attr_control_t *control)
```


参数:service_handle-------Characteristic要添加到的服务的Service handler服务句柄,一个Characteristic至少包括2个属性ATT，一个属性用于characteristic declaration，另一个用于存放特征值(characteristic value declaration).

char_uuid-------Characteristic 的UUID;  属于Characteristic declaration 这个ATT

perm------特征值声明(Characteristic value declaration) 属性(Attribute)访问权限;

ATT具有一组与其关联的权限值，权限值指定了读/写权限、认证权限、授权许可

permission权限的可取值{可读、可加密读、可加密MITM读、可写、可加密写、可加密MITM写}
```c
/**
 * @brief Attribute permissions
 */
#define    ESP_GATT_PERM_READ                  (1 << 0)   /* bit 0 -  0x0001 */    /* relate to BTA_GATT_PERM_READ in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_READ_ENCRYPTED        (1 << 1)   /* bit 1 -  0x0002 */    /* relate to BTA_GATT_PERM_READ_ENCRYPTED in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_READ_ENC_MITM         (1 << 2)   /* bit 2 -  0x0004 */    /* relate to BTA_GATT_PERM_READ_ENC_MITM in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_WRITE                 (1 << 4)   /* bit 4 -  0x0010 */    /* relate to BTA_GATT_PERM_WRITE in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_WRITE_ENCRYPTED       (1 << 5)   /* bit 5 -  0x0020 */    /* relate to BTA_GATT_PERM_WRITE_ENCRYPTED in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_WRITE_ENC_MITM        (1 << 6)   /* bit 6 -  0x0040 */    /* relate to BTA_GATT_PERM_WRITE_ENC_MITM in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_WRITE_SIGNED          (1 << 7)   /* bit 7 -  0x0080 */    /* relate to BTA_GATT_PERM_WRITE_SIGNED in bta/bta_gatt_api.h */
#define    ESP_GATT_PERM_WRITE_SIGNED_MITM     (1 << 8)   /* bit 8 -  0x0100 */    /* relate to BTA_GATT_PERM_WRITE_SIGNED_MITM in bta/bta_gatt_api.h */
```
property-----Characteristic Properties (特征值声明属性的Properties)

char_val------属性值 Characteristic Value

control-------属性响应控制字节

characteristic declaration的Attribute Value  包括 property、characteristic Value declaration handle、char_uuid 三个段；其中property、char_uuid在添加Characteristic调用的函数的参数中已经指明，只有characteristic Value declaration handle尚未明确指出。

示例:

```c
gl_profile_tab[PROFILE_A_APP_ID].service_handle = param->create.service_handle;
gl_profile_tab[PROFILE_A_APP_ID].char_uuid.len = ESP_UUID_LEN_16;
gl_profile_tab[PROFILE_A_APP_ID].char_uuid.uuid.uuid16 = GATTS_CHAR_UUID_TEST_A;   
esp_ble_gatts_add_char(gl_profile_tab[PROFILE_A_APP_ID].service_handle,  
                            &gl_profile_tab[PROFILE_A_APP_ID].char_uuid,  
                            ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,  
                            a_property,  
                            &gatts_demo_char1_val,  
                            NULL);
```


 characteristic value------这个属性ATT怎么添加入Characteristic中，看gatts_demo_char1_val的具体部分

```c
/**
  * @brief set the attribute value type
    */
typedef struct
{
    uint16_t attr_max_len;                      /*!<  attribute max value length */     
    uint16_t attr_len;                          /*!<  attribute current value length */ 
    uint8_t  *attr_value;                       /*!<  the pointer to attribute value */
} esp_attr_value_t;

uint8_t char1_str[] = {0x11,0x22,0x33};
esp_attr_value_t gatts_demo_char1_val = 
{
    .attr_max_len = GATTS_DEMO_CHAR_VAL_LEN_MAX,
    .attr_len     = sizeof(char1_str),
    .attr_value   = char1_str,
};
```

 在这个结构体里面包括了属性值最大长度、当前长度、当前值。由这个来指定 characteristic value declaration的。有这些值就足以构成一个 characteristic  value declaration ATT了。

### 3.3 添加Characteristic descriptor 描述符
当特征添加到service中成功(done)时，触发ESP_GATTS_ADD_CHAR_EVT事件。触发ESP_GATTS_ADD_CHAR_EVT事件时，回调函数参数param的结构体为gatts_add_char_evt_param，包括操作状态、特征ATT的handle()、service_handle(服务句柄)、characteristc uuid(服务的UUID)。
```c
    /**
     * @brief ESP_GATTS_ADD_CHAR_EVT
     */
    struct gatts_add_char_evt_param {
        esp_gatt_status_t status;       /*!< Operation status */
        uint16_t attr_handle;           /*!< Characteristic attribute handle */
        uint16_t service_handle;        /*!< Service attribute handle */
        esp_bt_uuid_t char_uuid;        /*!< Characteristic uuid */
    } add_char;                         /*!< Gatt server callback param of ESP_GATTS_ADD_CHAR_EVT */
```
还可以通过调用esp_ble_gatts_get_attr_value()来获取跟具体的Characteristic Value declartation 属性的具体信息。

下面是调用的例子，输入参数是特征句柄；输出参数是length和prf_char

```c
esp_ble_gatts_get_attr_value(param->add_char.attr_handle,  &length, &prf_char);
```

还可在ESP_GATTS_ADD_CHAR_EVT事件的回调函数中，给characteristic添加characteristic description ATT。

下面是添加char_descr的例子

```c
gl_profile_tab[PROFILE_A_APP_ID].char_handle = param->add_char.attr_handle;
gl_profile_tab[PROFILE_A_APP_ID].descr_uuid.len = ESP_UUID_LEN_16;
gl_profile_tab[PROFILE_A_APP_ID].descr_uuid.uuid.uuid16 = ESP_GATT_UUID_CHAR_CLIENT_CONFIG; 
esp_err_t add_descr_ret = esp_ble_gatts_add_char_descr(
gl_profile_tab[PROFILE_A_APP_ID].service_handle, &gl_profile_tab[PROFILE_A_APP_ID].descr_uuid,
ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE, NULL, NULL);
case ESP_GATTS_ADD_CHAR_DESCR_EVT:
        gl_profile_tab[PROFILE_A_APP_ID].descr_handle = param->add_char_descr.attr_handle;
        ESP_LOGI(GATTS_TAG, "ADD_DESCR_EVT, status %d, attr_handle %d, service_handle %d\n",
                 param->add_char_descr.status, param->add_char_descr.attr_handle, param->add_char_descr.service_handle);
        break;
```


 看看esp_ble_gatts_add_char_descr()这个函数的原型

esp_err_t esp_ble_gatts_add_char_descr (uint16_t service_handle,
                                        esp_bt_uuid_t   *descr_uuid,
                                        esp_gatt_perm_t perm, esp_attr_value_t *char_descr_val,
                                        esp_attr_control_t *control);
参数:service_handle:这个characteristic descriptor要添加的service handle。

perm: 描述符访问权限

descr_uuid:描述符UUID

char_descr_val:描述符值

control:ATT 应答控制字节

这个函数被用来添加Characteristic descriptor。当添加完成时，BTA_GATTS_ADD_DESCR_EVT 回调函数被调用去报告它的状态和ID。

gatt_server例子中：一共建立两个profile。A profile中包括的service UUID为0x00FF， characteristic UUID为0xFF01；

B profile中包括的service UUID为0x00EE， characteristic UUID为0xEE01

### 4 .建立连接后MTU大小确定
 当有手机(client客户端)连上server时，触发ESP_GATTS_MTU_EVT事件，其打印如下图所示

ESP_GATTS_MTU_EVT事件对应的回调函数中参数param的结构体为gatts_mtu_evt_param（包括连接id和MTU大小）

    /**
     * @brief ESP_GATTS_MTU_EVT
     */
    struct gatts_mtu_evt_param {
        uint16_t conn_id;               /*!< Connection id */
        uint16_t mtu;                   /*!< MTU size */
    } mtu;                              /*!< Gatt server callback param of ESP_GATTS_MTU_EVT */
在例子中设置本地的MTU大小为500，代码如下所示：

esp_err_t local_mtu_ret = esp_ble_gatt_set_local_mtu(500);
**如上所述，设置了MTU的值(经过MTU交换，从而设置一个PDU中最大能够交换的数据量)。例如：主设备发出一个150字节的MTU请求，但是从设备回应的MTU是23字节，那么今后双方要以较小的值23字节作为以后的MTU。即主从双方每次在做数据传输时不超过这个最大数据单元**。 MTU交换通常发生在主从双方建立连接后。MTU比较小，就是为什么BLE不能传输大数据的原因所在。

参照一分钟读懂低功耗(BLE)MTU交换数据包https://blog.csdn.net/viewtoolsz/article/details/76177465 这篇文章就可以了解MTU交换过程。

MTU交换请求用于client通知server关于client最大接收MTU大小并请求server响应它的最大接收MTU大小。

Client的接收MTU 应该大于或等于默认ATT_MTU(23).这个请求已建立连接就由client发出。这个Client Rx MTU参数应该设置为client可以接收的attribute protocol PDU最大尺寸。

MTU交换应答发送用于接收到一个Exchange MTU请求

这个应答由server发出，server的接收MTU必须大于或等于默认ATT_MTU大小。这里的Server Rx MTU应该设置为 服务器可以接收的attribute protocol PDU 最大尺寸。 

Server和Client应该设置ATT_MTU为Client Rx MTU和Server Rx MTU两者的较小值。

这个ATT_MTU在server在发出这个应答后，在发其他属性协议PDU之前生效；在client收到这个应答并在发其他属性协议PDU之前生效。

### 5. 发送应答数据
```c
#if (GATTS_INCLUDED == TRUE)
esp_err_t esp_ble_gatts_send_response(esp_gatt_if_t gatts_if, uint16_t conn_id, uint32_t trans_id,
                                      esp_gatt_status_t status, esp_gatt_rsp_t *rsp);


    btc_msg_t msg;
    btc_ble_gatts_args_t arg;
     
    ESP_BLUEDROID_STATUS_CHECK(ESP_BLUEDROID_STATUS_ENABLED);
    
    msg.sig = BTC_SIG_API_CALL;
    msg.pid = BTC_PID_GATTS;
    msg.act = BTC_GATTS_ACT_SEND_RESPONSE;
    arg.send_rsp.conn_id = BTC_GATT_CREATE_CONN_ID(gatts_if, conn_id);
    arg.send_rsp.trans_id = trans_id;
    arg.send_rsp.status = status;
    arg.send_rsp.rsp = rsp;
     
    return (btc_transfer_context(&msg, &arg, sizeof(btc_ble_gatts_args_t),
                                 btc_gatts_arg_deep_copy) == BT_STATUS_SUCCESS ? ESP_OK : ESP_FAIL);
}
#endif
```
这个函数用于发送应答给对应请求。

参数: gatts_if-------GATT server 访问接口

conn_id-----连接ID

trans_id-----传输ID

status----应答状态

rsp-----应答数据 gatt attribute value

### 6.最重要的几个事件的参数
将ESP_GATTS_READ_EVT、ESP_GATTS_WRITE_EVT和ESP_GATTS_EXEC_WRITE_EVT三类事件param参数的类型如下

  ```c
/**

   * @brief ESP_GATTS_READ_EVT
*/
     struct gatts_read_evt_param {
     uint16_t conn_id;               /*!< Connection id */
     uint32_t trans_id;              /*!< Transfer id */
     esp_bd_addr_t bda;              /*!< The bluetooth device address which been read */
     uint16_t handle;                /*!< The attribute handle */
     uint16_t offset;                /*!< Offset of the value, if the value is too long */
     bool is_long;                   /*!< The value is too long or not */
     bool need_rsp;                  /*!< The read operation need to do response */
         } read;                             /*!< Gatt server callback param of ESP_GATTS_READ_EVT */


    /**
     * @brief ESP_GATTS_WRITE_EVT
     */
    struct gatts_write_evt_param {
        uint16_t conn_id;               /*!< Connection id */
        uint32_t trans_id;              /*!< Transfer id */
        esp_bd_addr_t bda;              /*!< The bluetooth device address which been written */
        uint16_t handle;                /*!< The attribute handle */
        uint16_t offset;                /*!< Offset of the value, if the value is too long */
        bool need_rsp;                  /*!< The write operation need to do response */
        bool is_prep;                   /*!< This write operation is prepare write */
        uint16_t len;                   /*!< The write attribute value length */
        uint8_t *value;                 /*!< The write attribute value */
    } write;                            /*!< Gatt server callback param of ESP_GATTS_WRITE_EVT */
     
    /**
     * @brief ESP_GATTS_EXEC_WRITE_EVT
     */
    struct gatts_exec_write_evt_param {
        uint16_t conn_id;               /*!< Connection id */
        uint32_t trans_id;              /*!< Transfer id */
        esp_bd_addr_t bda;              /*!< The bluetooth device address which been written */

#define ESP_GATT_PREP_WRITE_CANCEL 0x00 /*!< Prepare write flag to indicate cancel prepare write */
#define ESP_GATT_PREP_WRITE_EXEC   0x01 /*!< Prepare write flag to indicate execute prepare write */
        uint8_t exec_write_flag;        /*!< Execute write flag */
    } exec_write; 
  ```


#### 6.1 使能通知
使能notify并读取蓝牙发过来的数据，开启这个后我们就能实时获取蓝牙发过来的值了。

使能通知(notify enable)的打印如下所示，打开通知实际上的一个WRITE。

 如果write.handle和descr_handle相同，且长度==2，确定descr_value描述值，根据描述值开启/关闭 通知notify/indicate。

```c
//the size of notify_data[] need less than MTU size
esp_ble_gatts_send_indicate(gatts_if, param->write.conn_id, gl_profile_tab[PROFILE_A_APP_ID].char_handle,
sizeof(notify_data), notify_data, false);
```

该函数将notify或indicate发给GATT的客户端；

need_confirm = false,则发送的是notification通知；

==true，发送的是指示indication。

其他参数: 服务端访问接口；连接id； 属性句柄，value_len; 值 

#### 6.2 读写数据
看看Write 和Read 事件触发的回调函数；

```c
ESP_LOGI(GATTS_TAG, "GATT_WRITE_EVT, conn_id %d, trans_id %d, handle %d", param->write.conn_id, param->write.trans_id, param->write.handle);
		if (!param->write.is_prep){
			ESP_LOGI(GATTS_TAG, "GATT_WRITE_EVT, value len %d, value :", param->write.len);
			esp_log_buffer_hex(GATTS_TAG, param->write.value, param->write.len);
example_write_event_env(gatts_if, &a_prepare_write_env, param);
		break;
	}
```


深入example_write_event_env(),其核心代码如下 esp_ble_gatts_send_response函数

```c
if (param->write.need_rsp){

.....
esp_err_t esp_ble_gatts_send_response(esp_gatt_if_t gatts_if, uint16_t conn_id, uint32_t trans_id,
                                      esp_gatt_status_t status, esp_gatt_rsp_t *rsp);
}
```


根据param->write.need_rsp是否需要应答来决定，是否调用esp_ble_gatts_send_response来应答。 

而esp_ble_gatts_send_response()函数实际调用了btc_transfer_context，将信息发送出去。

**need_rsp可以初始化为true或者false，以决定是否需要应答**，应答的话会影响数据传输速度，在需要大数据量的场合是否合适需要试验？

先看看write的事件参数

```c
     struct gatts_write_evt_param {
     uint16_t conn_id;               /*!< Connection id */
     uint32_t trans_id;              /*!< Transfer id */
     esp_bd_addr_t bda;              /*!< The bluetooth device address which been written */
     uint16_t handle;                /*!< The attribute handle */
     uint16_t offset;                /*!< Offset of the value, if the value is too long */
     bool need_rsp;                  /*!< The write operation need to do response */
     bool is_prep;                   /*!< This write operation is prepare write */
     uint16_t len;                   /*!< The write attribute value length */
     uint8_t *value;                 /*!< The write attribute value */
         } write;  
```

**这里的is_prep 是针对单独一个Write Request Attribute Protocol 信息太长了，放不下的情况，即Write Long Characteristic；一般先prepare再excute； 由这个参数确定参数的是长包还是短包。**

offset 是这包数据相对长数据的偏移，第一包是0x0000；

need_rsp是针对该写操作是否需要应答response。

value就是待写入的Characteristic Value的值。

接下去看看Read的事件参数

```c
struct gatts_read_evt_param {
    uint16_t conn_id;               /*!< Connection id */
    uint32_t trans_id;              /*!< Transfer id */
    esp_bd_addr_t bda;              /*!< The bluetooth device address which been read */
    uint16_t handle;                /*!< The attribute handle */
    uint16_t offset;                /*!< Offset of the value, if the value is too long */
    bool is_long;                   /*!< The value is too long or not */
    bool need_rsp;                  /*!< The read operation need to do response */
} read;                             /*!< Gatt server callback param of ESP_GATTS_READ_EVT */
```
 is_long针对客户端知道特征值句柄和特征值长度大于单独一个Read Response ATT 信息大小时，表示传输的是长数据，就用Read Blob Request。

handle 就是要读的Characteristic Value的句柄；

offset 就是要读的Characteristic Value偏移位置，第一包Read Blob Request时，offset为0x00；

对应的是Read Blob Response ATT 以一部分的Characteristic Value作为ATT Value 参数。

在看看ESP_IDF中关于read Response 的结构体。

```c
/// Gatt attribute value 
typedef struct {
    uint8_t           value[ESP_GATT_MAX_ATTR_LEN];         /*!< Gatt attribute value */
    uint16_t          handle;                               /*!< Gatt attribute handle */
    uint16_t          offset;                               /*!< Gatt attribute value offset */
    uint16_t          len;                                  /*!< Gatt attribute value length */
    uint8_t           auth_req;                             /*!< Gatt authentication request */
} esp_gatt_value_t;

/// GATT remote read request response type
typedef union {
    esp_gatt_value_t attr_value;                            /*!< Gatt attribute structure */
    uint16_t            handle;                             /*!< Gatt attribute handle */
} esp_gatt_rsp_t;
```






