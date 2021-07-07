# ESP32学习笔记（5）HTTP应用
>维护人员：**80后老男孩**  
>创建时间：2021-03-24  
>本文已发表于csdn博客[ESP32学习笔记（5）HTTP应用](https://blog.csdn.net/weixin_41034400/article/details/115164336)

[TOC]

**注：该部分的学习需要参考esp-idf下的demo程序**：  

1. ..\\esp-idf\\examples\\protocols\\esp_http_client

# 一.HTTP的简单介绍

HTTP--Hyper Text Transfer Protocol，超文本传输协议，是一种建立在TCP上的无状态连接，整个基本的工作流程是客户端发送一个HTTP请求，说明客户端想要访问的资源和请求的动作，服务端收到请求之后，服务端开始处理请求，并根据请求做出相应的动作访问服务器资源，最后通过发送HTTP响应把结果返回给客户端。其中一个请求的开始到一个响应的结束称为事务，当一个事物结束后还会在服务端添加一条日志条目

## HTTP请求  

HTTP请求是客户端往服务端发送请求动作，告知服务器自己的要求。其中信息由三部分组成：

请求方法，URI协议/版本：包括请求方式Method、资源路径URL、协议版本Version
请求头：包括一些访问的域名、用户代理、Cookie等信息
请求正文：就是HTTP请求的数据
备注：请求方式Method一般有GET、POST、PUT、DELETE，含义分别是获取、修改、上传、删除，其中GET方式仅仅为获取服务器资源，方式较为简单，因此在请求方式为GET的HTTP请求数据中，请求正文部分可以省略，直接将想要获取的资源添加到URL中。下图所示就是GET的请求，没有请求正文。详细的说明在下边。

现在大多数协议版本为http/1.1  
![](assets/004/002/005-1616551211481.png)  
下图所示为POST请求的格式，有请求方法（状态行）、请求头、请求正文三部分  
![](assets/004/002/005-1616551263434.png)  
## HTTP响应  

服务器收到了客户端发来的HTTP请求后，根据HTTP请求中的动作要求，服务端做出具体的动作，将结果回应给客户端，称为HTTP响应。数据主要由三部分组成：HTTP 应答格式

1. 协议状态：包括协议版本Version、状态码Status Code、回应短语
2. 响应头：包括搭建服务器的软件，发送响应的时间，回应数据的格式等信息
3. 响应正文：就是响应的具体数据

备注：我们主要关心并且能够在客户端浏览器看得到的是三位数的状态码，不同的状态码代表不同的含义，其中

1xx：表示HTTP请求已经接受，继续处理请求  
2xx：表示HTTP请求已经处理完成  
3xx：表示把请求访问的URL重定向到其他目录  
4xx：表示客户端出现错误  
5xx：表示服务端出现错误  

具体HTTP响应实例如下图：  
![](assets/004/002/005-1616551348402.png)  

常见状态码的含义：

200---OK/请求已经正常处理完毕  
301---/请求永久重定向  
302---/请求临时重定向  
304---/请求被重定向到客户端本地缓存  
400---/客户端请求存在语法错误  
401---/客户端请求没有经过授权  
403---/客户端的请求被服务器拒绝，一般为客户端没有访问权限  
404---/客户端请求的URL在服务端不存在  
500---/服务端永久错误  
503---/服务端发生临时错误  

## HTTP报文格式

HTTP报文是HTTP应用程序之间传输的数据块，HTTP报文分为HTTP请求报文和HTTP响应报文，但是无论哪种报文，他的整体格式是类似的，大致都是由起始、首部、主体三部分组成，起始说明报文的动作，首部说明报文的属性，主体则是报文的数据。接下来具体说明。

### HTTP请求报文  
![](assets/004/002/005-1616551423137.png)  
请求报文的起始由请求行构成（有些资料称为状态行，名字不一样而已，都是指的一个东西），用来说明该请求想要做什么，由\<Method>、\<URL>、\<Version> 三个字段组成，注意每个字段之间都有一个空格。

其中<Method>字段有不同的值：
GET   --- 访问服务器的资源  
POST  --- 向服务器发送要修改的数据  
HEAD  --- 获取服务器文档的首部  
PUT   --- 向服务器上传资源  
DELETE--- 删除服务器的资源  
\<URL>字段表示服务器的资源目录定位  
\<Version>字段表示使用的http协议版本  

首部部分由多个请求头（也叫首部行）构成，那些首部字段名有如下，不全：  
Accept     指定客户端能够接收的内容格式类型  
Accept-Language 指定客户端能够接受的语言类型  
Accept-Ecoding  指定客户端能够接受的编码类型  
User-Agent      用户代理，向服务器说明自己的操作系统、浏览器等信息  
Connection      是否开启持久连接（keepalive）  
Host            服务器域名  
 ...  

主体部分就是报文的具体数据。  

### HTTP响应报文  
![](assets/004/002/005-1616551456590.png)  
响应报文的起始由状态行构成，用来说明服务器做了什么，由\<Version>、\<Status-Code>、\<Phrase>三个字段组成，同样的每个字段之间留有空格；

\<Status-Code> 上边已经说明； 

首部由多个响应头(也叫首部行)组成， 首部字段名如下，不全：  
Server    服务器软件名，Apache/Nginx  
Date      服务器发出响应报文的时间  
Last-Modified   请求资源的最后的修改时间  
 ...

主体部分是响应报文的具体数据。

# 二.esp_http_client的demo

**注：demo的例程仍然用到了公共组件：ESP_ERROR_CHECK(example_connect())，用于wifi或者以太网网络的连接，在前面的学习中已经讲过;**

## 参数结构体简介

先来看一下http的配置参数结构体，在初始化过程中对照填写应用指定的参数
```c
 typedef struct {
     const char                  *url; //url请求接口必须配置               /*!< HTTP URL, the information on the URL is most important, it overrides the other fields below, if any */
     const char                  *host; //服务器域名或ip地址              /*!< Domain or IP as string */
     int                          port; //端口 http默认80 https 默认443                /*!< Port to connect, default depend on esp_http_client_transport_t (80 or 443) */
     const char                  *username;  //用户名，认证使用         /*!< Using for Http authentication */
     const char                  *password;  //用户密码，认证使用         /*!< Using for Http authentication */
     esp_http_client_auth_type_t auth_type; //认证方式          /*!< Http authentication type, see `esp_http_client_auth_type_t` */
     const char                  *path;  //路径             /*!< HTTP Path, if not set, default is `/` */
     const char                  *query;  //请求参数            /*!< HTTP query */
     const char                  *cert_pem;    //证书       /*!< SSL server certification, PEM format as string, if the client requires to verify server */
     const char                  *client_cert_pem;    /*!< SSL client certification, PEM format as string, if the server requires to verify client */
     const char                  *client_key_pem;     /*!< SSL client key, PEM format as string, if the server requires to verify client */
     esp_http_client_method_t    method;  //请求方式 post get                 /*!< HTTP Method */
     int                         timeout_ms;  //请求超时             /*!< Network timeout in milliseconds */
     bool                        disable_auto_redirect;    /*!< Disable HTTP automatic redirects */
     int                         max_redirection_count;    /*!< Max number of redirections on receiving HTTP redirect status code, using default value if zero*/
     int                         max_authorization_retries;    /*!< Max connection retries on receiving HTTP unauthorized status code, using default value if zero. Disables authorization retry if -1*/
     http_event_handle_cb        event_handler;  //可注册回调           /*!< HTTP Event Handle */
     esp_http_client_transport_t transport_type;  // 传输方式 tcp ssl        /*!< HTTP transport type, see `esp_http_client_transport_t` */
     int                         buffer_size; //接收缓存大小             /*!< HTTP receive buffer size */
     int                         buffer_size_tx; //发送缓存大小          /*!< HTTP transmit buffer size */
     void                        *user_data;  //http用户数据             /*!< HTTP user_data context */
     bool                        is_async;  //同步模式               /*!< Set asynchronous mode, only supported with HTTPS for now */
     bool                        use_global_ca_store;       /*!< Use a global ca_store for all the connections in which this bool is set. */
     bool                        skip_cert_common_name_check; //跳过证书   /*!< Skip any validation of server certificate CN field */
 } esp_http_client_config_t;
```
## 核心代码注释理解

### http的事件回调函数
```c
esp_err_t _http_event_handler(esp_http_client_event_t *evt)
{
    switch (evt->event_id)
    {
    case HTTP_EVENT_ERROR: //错误事件
        ESP_LOGD(TAG, "HTTP_EVENT_ERROR");
        break;
    case HTTP_EVENT_ON_CONNECTED: //连接成功事件
        ESP_LOGD(TAG, "HTTP_EVENT_ON_CONNECTED");
        break;
    case HTTP_EVENT_HEADER_SENT: //发送头事件
        ESP_LOGD(TAG, "HTTP_EVENT_HEADER_SENT");
        break;
    case HTTP_EVENT_ON_HEADER: //接收头事件
        ESP_LOGD(TAG, "HTTP_EVENT_ON_HEADER, key=%s, value=%s", evt->header_key, evt->header_value);
        break;
    case HTTP_EVENT_ON_DATA: //接收数据事件
        ESP_LOGD(TAG, "HTTP_EVENT_ON_DATA, len=%d", evt->data_len);
        if (!esp_http_client_is_chunked_response(evt->client))
        {
            // Write out data
            // printf("%.*s", evt->data_len, (char*)evt->data);
        }

        break;
    case HTTP_EVENT_ON_FINISH: //会话完成事件
        ESP_LOGD(TAG, "HTTP_EVENT_ON_FINISH");
        break;
    case HTTP_EVENT_DISCONNECTED: //断开事件
        ESP_LOGI(TAG, "HTTP_EVENT_DISCONNECTED");
        int mbedtls_err = 0;
        esp_err_t err = esp_tls_get_and_clear_last_error(evt->data, &mbedtls_err, NULL);
        if (err != 0)
        {
            ESP_LOGI(TAG, "Last esp error code: 0x%x", err);
            ESP_LOGI(TAG, "Last mbedtls failure: 0x%x", mbedtls_err);
        }
        break;
    }
    return ESP_OK;
}
```
### http的各种“方法”（使用url的方式）的使用举例
```c
static void http_rest_with_url(void)
{
    esp_http_client_config_t config = {
        .url = "http://httpbin.org/get",      //请求url
        .event_handler = _http_event_handler, //注册事件回调
    };
    //开始HTTP会话之前必须首先调用这个初始化函数esp_http_client_init。
    //它的返回值esp_http_client_handle_t型变量作为其他HTTP API的实参用。
    esp_http_client_handle_t client = esp_http_client_init(&config);

    // GET
    /*  esp_http_client_perform
    该API以阻塞或非阻塞方式执行整个请求。默认情况下，API以阻塞的方式执行请求，并在完成时返回，
    或者在以非阻塞的方式下遇到EAGAIN/EWOULDBLOCK/EINPROGRESS返回。
    在非阻塞状态下的请求可以调用这个API多次除非这个请求&应答完成或失败。
    使能非阻塞的方法是设置esp_http_client_config_t配置项中的is_async.这样，你就可以对同一个esp_http-client_handle_t变量进行多次调用。
    如果服务器允许，底层连接可以保持为打开状态，我们鼓励传输多个文件。
    esp_http_client重用相同的连接，可以使操作更快、CPU更少，使用的网络资源更少。
    注意:永远不要同时对同一个client句柄调用这个API。要想并行运行，请调用不同的client句柄。
    这个函数包括如下操作:esp_http_client_open->esp_http_client_write->esp_http_client_fetch_headers->esp_http_client_read
    或者esp_http_client_close
    */
    esp_err_t err = esp_http_client_perform(client);
    if (err == ESP_OK)
    {
        ESP_LOGI(TAG, "HTTP GET Status = %d, content_length = %d",
                 esp_http_client_get_status_code(client),     //状态码
                 esp_http_client_get_content_length(client)); //数据长度
    }
    else
    {
        ESP_LOGE(TAG, "HTTP GET request failed: %s", esp_err_to_name(err));
    }

    // POST
    const char *post_data = "field1=value1&field2=value2";
    esp_http_client_set_url(client, "http://httpbin.org/post");
    esp_http_client_set_method(client, HTTP_METHOD_POST);
    esp_http_client_set_post_field(client, post_data, strlen(post_data));
    err = esp_http_client_perform(client);
    if (err == ESP_OK)
    {
        ESP_LOGI(TAG, "HTTP POST Status = %d, content_length = %d",
                 esp_http_client_get_status_code(client),
                 esp_http_client_get_content_length(client));
    }
    else
    {
        ESP_LOGE(TAG, "HTTP POST request failed: %s", esp_err_to_name(err));
    }

    ....以下各种方法雷同，忽略  
    ......

    esp_http_client_cleanup(client); //在所有操作完成后用esp_http_client_cleanup()来清除
}
```

### 应用实例1：get一下百度首页
```c
void http_client_test(void)
{
    char pReadBuf[513];
    //http client配置
    esp_http_client_config_t config = {
        .method = HTTP_METHOD_GET,           //get请求
        .url = "https://www.baidu.com/",     //请求url
        .event_handler = _http_event_handler, //注册时间回调
    };
    esp_http_client_handle_t client = esp_http_client_init(&config); //初始化配置
    esp_err_t err = esp_http_client_perform(client);                 //执行请求

    if (err == ESP_OK)
    {
        ESP_LOGI(TAG, "Status = %d, content_length = %d",
                 esp_http_client_get_status_code(client),            //状态码
                 esp_http_client_get_content_length(client));        //数据长度
        esp_err_t ret = esp_http_client_read(client, pReadBuf, 512); //读取512数据内容
        if (ret > 0)
        {
            ESP_LOGI(TAG, "recv data = %d %s", ret, pReadBuf); //打印数据
        }
    }
    esp_http_client_cleanup(client); //断开并释放资源
}
```
### 应用实例2：get一下天气数据
```c
void https_client_test(void)
{
    char pReadBuf[513];
    //http client配置
    esp_http_client_config_t config = {
        .method = HTTP_METHOD_GET,           //get请求
        .url = "https://api.thinkpage.cn/v3/weather/now.json?key=g3egns3yk2ahzb0p&location=jinan&language=en",     //请求url
        .event_handler = _http_event_handler, //注册时间回调
    };
    esp_http_client_handle_t client = esp_http_client_init(&config); //初始化配置
    esp_err_t err = esp_http_client_perform(client);                 //执行请求

    if (err == ESP_OK)
    {
        ESP_LOGI(TAG, "Status = %d, content_length = %d",
                 esp_http_client_get_status_code(client),            //状态码
                 esp_http_client_get_content_length(client));        //数据长度
        esp_err_t ret = esp_http_client_read(client, pReadBuf, 512); //读取512数据内容
        if (ret > 0)
        {
            ESP_LOGI(TAG, "recv data = %d %s", ret, pReadBuf); //打印数据
        }
    }
    esp_http_client_cleanup(client); //断开并释放资源
}
```

