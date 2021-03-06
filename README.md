# Sinetlib: A High Perfermance C++ Network Library

[![Build Status](https://travis-ci.org/silence1772/Sinetlib.svg?branch=master)](https://travis-ci.org/silence1772/Sinetlib)
[![license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://opensource.org/licenses/MIT)

## Introduction
Sinetlib是一个仿照Muduo实现的基于Reactor模式的多线程网络库，附有异步日志，要求Linux 2.6以上内核版本。同时内嵌一个简洁HTTP服务器，可实现路由分发及静态资源访问。

## Feature
* 底层使用Epoll LT模式实现I/O复用，非阻塞I/O
* 多线程、定时器依赖于c++11提供的std::thread、std::chrono库
* Reactor模式，主线程accept请求后，使用Round Robin分发给线程池中线程处理
* 基于小根堆的定时器管理队列
* 双缓冲技术实现的异步日志
* 使用智能指针及RAII机制管理对象生命期
* 使用状态机解析HTTP请求，支持HTTP长连接
* HTTP服务器支持URL路由分发及访问静态资源，可实现RESTful架构

## Envoirment
* OS： Ubuntu 16.04
* Complier： g++ 5.4
* Build： CMake

## Tutorial
[C++网络编程实战项目--Sinetlib网络库（1）——概述](https://blog.csdn.net/silence1772/article/details/82916957)

[C++网络编程实战项目--Sinetlib网络库（2）——I/O复用与事件分发](https://blog.csdn.net/silence1772/article/details/83307716)

[C++网络编程实战项目--Sinetlib网络库（3）——事件循环与跨线程调用](https://blog.csdn.net/silence1772/article/details/83316765)

[C++网络编程实战项目--Sinetlib网络库（4）——线程池和整体框架](https://blog.csdn.net/silence1772/article/details/83377906)

[C++网络编程实战项目--Sinetlib网络库（5）——HTTP服务器设计与实现](https://blog.csdn.net/silence1772/article/details/83421085)

## Model
主线程负责连接的建立，并通过Round Robin方式将连接分配给工作线程处理

![1](https://img-blog.csdn.net/20181025132001790?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbGVuY2UxNzcy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![2](https://img-blog.csdn.net/20181025132600145?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbGVuY2UxNzcy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## Build
    
需先安装Cmake：

    $ sudo apt-get update
    $ sudo apt-get install cmake

开始构建

    $ git clone git@github.com:silence1772/Sinetlib.git
    $ cd Sinetlib
    $ ./build.sh

执行完上述脚本后编译结果在新生成的build文件夹内，示例程序在build/bin下。

库和头文件分别安装在/usr/local/lib和/usr/local/include，该库依赖c++11及pthread库，使用方法如下：

    $ g++ main.cpp -std=c++11 -lSinetlib -lpthread

在执行生成的可执行文件时可能会报找不到动态库文件，需要添加动态库查找路径，首先打开配置文件：

    $ sudo vim /etc/ld.so.conf

在打开的文件末尾添加这句，保存退出：

    include /usr/local/lib

使修改生效：

    $ sudo /sbin/ldconfig

## Usage
### net
Sinetlib的使用十分简单，用户只需设置四个回调即可。四个回调抽象的是TCP连接建立后、有消息到达、答复消息完成、连接关闭这四个状态，用户可以设置各个状态对应的执行函数。

在此之前，用户需要先创建Looper和Server，Server的第二和第三个参数分别是监听的端口号和线程池中线程数，一般情况下建议线程数与CPU核心数接近，以最大程度发挥多线程性能。
```c++
#include "server.h"
#include <iostream>

void OnConnection(const std::shared_ptr<Connection>& conn)
{
    std::cout << "OnConnection" << std::endl;
}
void OnMessage(const std::shared_ptr<Connection>& conn, IOBuffer* buf, Timestamp t)
{
    std::cout << "OnMessage" << std::endl;
}
void OnReply(const std::shared_ptr<Connection>& conn)
{
    std::cout << "OnReply" << std::endl;
}
void OnClose(const std::shared_ptr<Connection>& conn)
{
    std::cout << "OnClose" << std::endl;
}

int main()
{
    Looper loop;
    Server s(&loop, 8888, 4);

    s.SetConnectionEstablishedCB(OnConnection);
    s.SetMessageArrivalCB(OnMessage);
    s.SetReplyCompleteCB(OnReply);
    s.SetConnectionCloseCB(OnClose);

    s.Start();
    loop.Start();
}
```
可通过telnet测试上述程序，测试结果如下（左边为telnet程序，带有$号为用户输入内容，右边为服务器输出）：
```shell
$ telnet 127.0.0.1 8888
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.              
                                            OnConnection
$ test message
                                            OnMessage
$ ^]
telnet> quit
                                            OnClose
```

在提供给用户设置的各个回调函数中，都有一个指向当前连接的指针，在消息到达回调中还暴露了保存接收到的消息的IOBuffer缓冲区指针。

用户可以对这两个指针进行操作,通过读取缓冲区和调用Connection::Send()实现消息的收发。

Connection类：
```c++
// 发送数据到连接的对端
void Send(const void* data, size_t len);
void Send(const std::string& message);
void Send(IOBuffer& buffer);
// 关闭连接中自己的这一端，这会激发TCP的四次挥手进而关闭整个连接
void Shutdown();
// 获取连接描述符
const int GetFd()
// 获取输入输出缓冲区
const IOBuffer& GetInputBuffer()
const IOBuffer& GetOutputBuffer()
```

IOBuffer:
```c++
// 获取可读、可写、预留区的大小
size_t GetReadableSize()
size_t GetWritableSize()
size_t GetPrependSize()
// 获取可读、可写区的首指针
const char* GetReadablePtr()
const char* GetWritablePtr()
// 寻找回车换行符/r/n
const char* FindCRLF()
const char* FindCRLF(const char* start_ptr)
// 寻找换行符/n
const char* FindEOL()
const char* FindEOL(const char* start_ptr)
// 向后移动可读区指针到实际位置
void Retrieve(size_t len)
// 移动可读区指针到指定位置
void RetrieveUntil(const char* end)
// 将可读、可写区指针均移到初始位置，即重置缓冲区
void RetrieveAll()
// 添加数据进缓冲区
void Append(const char* data, size_t len)
void Append(const std::string& str)
// 添加数据进预留区
void Prepend(const void* data, size_t len)
```

### http
Sinetlib内嵌了一个HTTP服务器，设计上借鉴了golang的mux包实现路由分发的思想，同时还支持静态资源访问。

用户可以根据需要设置路由及匹配条件，目前支持URL、请求参数、请求头及请求方法的匹配，并且可以从中提取出参数。
一个请求必须满足所有条件才能匹配这个路由，得到它的Handler。

首先要设置相应的路由处理函数,该函数由用户实现，如果要实现静态资源访问，则需使用HttpServer的文件处理函数,该函数会将访问映射到用户指定的路径。
```
Route::SetHandler(YourHandler);
Route::SetHandler(HttpServer::GetFileHandler("/home/mys/"));
```
可以对url进行匹配,并且可以设置正则表达式匹配规则，比如下面第一句匹配对于/path/xxx的访问，并且可以通过key来获取到xxx，此处限定了key必须为字母。
也可以不设置正则匹配条件，如第二句的key可以匹配任何字符。
```
Route::SetPath("/path/{key:[a-zA-Z]+}");
Route::SetPath("/path/{key}");
```
匹配方法头、头部字段、参数：
```
Route::SetMethod("GET");
Route::SetHeader("Header-Name", "Header-Value");
Route::SetQuery("Filed", "Value");
```
匹配前缀，可通过“file_path"获取”/file/"后面的路径：
```
Route::SetPrefix("/file/");
```


完整的程序使用如下,该程序有两个路由，其中第二个为静态资源服务。

```c++
#include "httpserver.h"

void MyHandler(const HttpRequest& request, std::unordered_map<std::string, std::string>& match_map, HttpResponse* response)
{
    response->SetStatusCode(HttpResponse::OK);
    response->SetStatusMessage("OK");
    response->SetContentType("text/html");

    std::string body = "temp test page";
    response->AddHeader("Content-Length", std::to_string(body.size()));
    response->AppendHeaderToBuffer();
    response->AppendBodyToBuffer(body);
}

int main()
{
    Looper loop;
    HttpServer s(&loop, 8888, 4);

    s.NewRoute()
    ->SetPath("/path/{name:[a-zA-Z]+}")
    ->SetQuery("query", "t")
    ->SetHeader("Connection", "keep-alive")
    ->SetHandler(MyHandler);
    
    s.NewRoute()
    ->SetPrefix("/file/")
    ->SetHandler(s.GetFileHandler("/home/mys/"));

    s.Start();
    loop.Start();
}
```
该程序可匹配下面两个HTTP请求样例，第二个请求对应访问位于/home/mys/test.jpg的文件，可通过浏览器访问127.0.0.1:8888/path/myname?query=t和127.0.0.1:8888/file/test.jpg实现下列请求

    GET /path/myname?query=t HTTP/1.1
    Connection: keep-alive

    GET /file/test.jpg HTTP/1.1
    
用户使用时可通过HttpServer::NewRoute()创建一个新路由，并设置相应的匹配条件及处理函数。在有请求到来时会按照用户创建路由的顺序进行匹配，当有一个路由下的条件全部匹配，即可得到该路由的处理函数并执行。

对于静态资源访问，需要使用Route::SetPrefix("/prefix/")设置前缀，并配合使用HttpServer::GetFileHandler(“/map/path/")设置映射路径，那么对于/prefix/my/src的访问将会被映射到/map/path/my/src

成功匹配请求后即可执行对应的处理函数，这里暴露给了用户HttpRequest、map<string, string>、HttpResponse三个类，大概的使用就是从HttpRequest中取得请求的各项内容，同时可以从map中根据之前设置的key取得相应的value，比如上述程序的第一个路由就可取出‘name’的值。然后用户再把响应的内容写入HttpResponse即可。

需要注意的是HttpResponse的使用，HttpResponse内有一个发送缓冲区，用户需要先设置该类的头部信息，然后执行AppendHeaderToBuffer()把这些信息写入缓冲区，然后再直接往缓冲区中写入消息的主体。

HttpRequest接口如下：
```c++
// 获取方法头
const char* GetMethodStr()
// 获取HTTP版本
Version GetVersion()
// 获取URL
const std::string& GetPath()
// 获取参数
const std::string GetQuery(const std::string& field)
// 获取头部字段
std::string GetHeader(const std::string& field)
// 获取接收时间
Timestamp GetReceiveTime()
```
HttpResponse接口如下：
```c++
// 设置短连接
void SetCloseConnection(bool on)
// 设置响应状态码
void SetStatusCode(HttpStatusCode code)
// 设置状态信息
void SetStatusMessage(const std::string& message)
// 添加头部字段
void AddHeader(const std::string& field, const std::string& value)
// 设置Content-Type
void SetContentType(const std::string& content_type)
// 将响应头写入到缓冲区中
void AppendHeaderToBuffer();
// 将消息主体写入到缓冲区中
void AppendBodyToBuffer(std::string& body);
```
## Fix List

* 2018-11-15 修复对于短连接未写完数据就关闭连接的错误

Connection::Shutdown()没有判断当前数据是否发送完就直接关闭连接，导致当数据一次不能写完而连接又被shutdown时，一直在写一个已关闭的连接，产生死循环。

* 2018-11-15 修复文件句柄泄漏

当连接总数超过1000左右时程序异常终止，检查core文件定位到fopen()打开文件失败，查看errno发现打开的文件数超过系统限制1024。重新运行进程，cd 到 /proc/进程号/fd，查看目录内容即为打开的文件描述符，发现当新连接建立时新增socket描述符，当连接断开时却不减少，最终定位到connection.cpp中析构函数没有close掉socket。

* 2018-11-15 修复HTTP服务器中打开dir没有关闭导致泄漏的问题

* 2018-11-29 修复多线程下unordered_map不安全的错误

旧版本对于每个connection的解析器parser都是由主线程管理，存放在map里，当工作线程同时去map里去parser时就会产生错误，新版模仿muduo使用类似c++17的std::any，将parser直接嵌入到connection里，以避免出现不可重入现象

* 2018-11-29 修复cpu占用100%错误

当连接发生EPOLLERR时未能调用关闭连接回调，由于使用LT模式导致一直产生EPOLLERR事件，陷入死循环占用cpu；在eventbase的事件分发里对EPOLLERR事件调用关闭回调即可解决

## Contact
* Mail: silence1772@qq.com

## More

to be continued
