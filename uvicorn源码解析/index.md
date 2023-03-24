# Uvicorn源码解析


![background](/blog/uvicorn/uvicorn.png "starlette")

## 目录结构

```shell
uvicorn
├── config.py
├── importer.py
├── __init__.py
├── lifespan
├── logging.py
├── loops
├── __main__.py
├── main.py
├── middleware
├── protocols
├── py.typed
├── server.py
├── \_subprocess.py
├── supervisors
├── \_types.py
└── workers.py
```

这些文件都有对应的作用

- lifespan：启动和结束时候和ASGI应用通信，在启动的时候调用on.py中的LifespanOn类，发送信息给ASGI应用实例{“type”:"lifspan.startup"}, 等待ASGI应用实例发送complete后Uvicorn继续运行，报错则终止运行。
- loops: 加载事件循环模型，优先uvloop
- protocols：里面存放着读取连接数据和解析消息体的协议， 如HTTP和WebSockets, 可以把他认为是一个序列化器。比较核心的部分
- middleware：中间件，ASGI2Middleware， MessageLoggerMiddleware，WSGIMiddleware和ProxyHeadersMiddleware。在调用config.load方法的时候套在ASGI应用实例外层。
- supervisors：存放Uvicorn几种启动方式。内部有Multiprocess，BaseReload，StatReload，WatchFilesReload，WatchGodReload。对外暴漏的只有Multiprocess和ChangeReload。ChangeReload对应几种热加载方法。优先级WatchFilesReload > WatchGodReload > StatReload.
- config.py: 配置类，lifespan，loops，protocols，log，绑定socket等。
- importer.py: 主要是一个import_from_string方法
- logging.py：日志配置
- main.py 入口文件，代码运行和命令行 运行 使用click来写的命令行
- server.py 核心服务
- subprocess.py： 给`supervisors/multiprocess.py`使用的， 可能是为了以后拓展需要， 才放在一级目录
- workers.py： 其他工作模式的`Uvicorn`， 比如里面有个`UvicornWorker`, 就是用于`gunicorn`启动`uvicorn`

## 启动流程

![life-cycle](/blog/uvicorn/life.png "生命周期")

在uvicorn中main.py为如何文件，其中的main函数为入口函数，main函数调用了run函数。

### main.run

run函数主要做以下几件事

1. 初始化config
2. 将初始化的config作为参数传递给Server函数进行初始化
3. 根据config中参数加载ChangeReload 和 Multiprocess 并绑定socket
4. 调用server.run

### Config

Config类主要的作用是按需加载类。

- 判断是否为ssl如果是则创建ssl上下文
- 加载http解析库，优先httptools，如果没有就使用h11
- 加载websocket解析库
- 加载lifespan模块
- 根据启动命令加载ASGi实例
- 判断使用的是WSGI，ASGI2还是ASGI3协议，然后如果是wsgi则app包一层WSGIMiddleware，ASGI2则是ASGI2Middleware
- 加载log中间件和代理头中间件,这一不会把app实列分别包裹MessageLoggerMiddleware和ProxyHeadersMiddleware。
 上述加载完毕后会将config中的loaded参数设置为True标记已经进行过加载。

### Server

![server_code](/blog/uvicorn/server_code.png "server代码展示")
server.run调用Server.server方法启用服务，主要做以下工作：

1. 从config中加载lifespan模块。
2. 监控ctrl+c和kill \<pid\>信号。
3. 调用server.startup函数。startup调用完毕会检查是否启动成功，失败直接推出函数。
4. 调用main_loop保持程序运行。
5. 停止运行调用shutdown函数。

#### Server.startup

startup主要做以下工作:

1. 调用config.lifespan中startup方法，startup方法用来通知ASGI实列进行启动初始化传送的信息为{"type": "lifespan.shutdown"}，ASGI应用实例启动成功就给Uvicorn返回{"type": "lifespan.startup.complete"}来告知启动成功。如果失败则将should_exit参数更新为True。
2. 调用内部create_protocol函数。创建HTTP_PROTOCOLS中对应的类(在代码protocols中后面详细解释，这些类继承asycnio.Protocol用来从socket获取数据和写入数据， 同时也有一些TCP相关的调用。)这些类作为socket和ASGi实例中间的通信中间层,将http数据和ASGI数据进行互转。如：   b\'xxxxx\r\n\' >>> {"type":"http.lifespan.start", "status":200,....}
3. 随后根据config中的变量启动服务。启动完毕后将started标记为True，表明已经启动服务。
1. 当用户传socket过来的时候： 基于该scoket和`create_protocol`创建服务， 如果是多进程且是Windows系统， 则要显示的共享socket。
2. 当用户传文件描述符的时候： 基于该文件描述符获取scoket， 并通过该socket和`create_protocol`创建服务。
3. 当用户传unix domain socket的时候: 基于unix domain socket和`create_protocol`创建服务。
4. 当用户传host和port参数的时候: 基于host和port和`create_protocol`创建服务。

#### Server.main_loop

mian_loop主要做了以下功能：

1. 声明一个counter来记录0.1s记录一次
2. 调用on_tick方法获取should_exit状态
3. 当should_exit为True的时候推出循环。下面是should_exit什么场景下会为True。
1. startup失败会把should_exit置为True
2. 监听到ctrl+c和kill\<pid\>也会置为True
3. 请求次数大于config中设置的限制请求次数。

#### Server.shutdown

 shutdown主要做了以下工作:

- 拒绝新的请求
- 关闭所有现存的链接
- 等待所有的链接返回结果
- 等待现有的任务完成
- 调用lifespan.shutdown,等待应用关闭

## uvicorn.protocols

这是Uvicorn中比较核心的代码，但是由于我对socket的了解不够深入就等有时间继续写。!

