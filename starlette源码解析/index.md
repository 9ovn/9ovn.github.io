# Starlette源码解析


![background](/blog/starlette/starlette.png "starlette")

## 文件目录

```shell
starlette
├── applications.py ： Starlett类，启动的类
├── authentication.py： 验证相关的类，和AuthenticationMiddleware的实现还有permissions的实现在这里
├── background.py：封装后台任务， 会在返回响应后执行
├── \_compat.py： 用于Etag中md5，http缓存
├── concurrency.py： 对anyio的一些封装，包含了run_until_first_complete，run_in_threadpool方法。
├── config.py： 配置类环境变量
├── convertors.py： 用于路由参数转化到指定函数的方法
├── datastructures.py： 一些startette定义的数据类，Url,URLPath, Secret,State,UploadFile等。
├── endpoints.py:  包括 HTTPEndpoint 和 WebSocketEndpoint，它们提供基于类的视图模式来处理 HTTP 方法调度和 WebSocket 会话.
├── exceptions.py: 封装了两个错误类：HTTPException，WebSocketException。
├── formparsers.py：Form，File之类的解析类。
├── \_\_init\_\_.py：版本信息
├── middleware：中间件
├── requests.py： 请求数据核心为Request。
├── responses.py： 响应, 负责初始化Header和Cookies， 同时根据不同的Respnose类生成响应数据， 然后有个类ASGI调用接口， 该接口会发送ASGI协议到uvicorn服务， 发送完后如果有backgroud task, 则执行backgroud task， 直到执行完成， 该响应流程才结束。
├── routing.py： 路由
├── schemas.py： OpenApi相关的Schemas
├── staticfiles.py： 静态文件
├── status.py： HTTP状态码
├── templating.py：基于jinja的模板响应
├── testclient.py： 测试客户端
├── types.py： 类型
├── \_utils.py： 封装了一些工具类，判断是否为异步可调用对象，以及简单封装的异步管理器
└── websockets.py： 类型
```

## 生命周期

简化版的这里面还有一些细节，比如
![life-cycle](/blog/starlette/life.png "生命周期")

## applications.Starlette

- debug - 布尔值，指示是否在错误时返回调试追溯。
- routes - 一个路由列表，用于为传入的HTTP和WebSocket请求提供服务。
- middleware - 一个中间件列表，用于每个请求运行。Starlette应用程序始终会自动包括两个中间件类。ServerErrorMiddleware被添加为最外层的中间件，用于处理整个堆栈中任何未捕获的错误。ExceptionMiddleware被添加为最内层的中间件，用于处理路由或端点中发生的已处理异常情况。
- exception_handlers - 一个映射，将整数状态码或异常类类型映射到处理异常的可调用对象上。异常处理程序可调用对象应该是handler(request, exc) -> response的形式，并且可以是标准函数或异步函数。
- on_startup - 一个可调用对象列表，用于在应用程序启动时运行。启动处理程序可调用对象不需要任何参数，并且可以是标准函数或异步函数。
- on_shutdown - 一个可调用对象列表，用于在应用程序关闭时运行。关闭处理程序可调用对象不需要任何参数，并且可以是标准函数或异步函数。
- lifespan - 生命周期上下文函数，可用于执行启动和关闭任务。这是一种新的样式，取代了on_startup和on_shutdown处理程序。使用其中之一即可，不要同时使用。

#### applications.Starlette.\_\_call\_\_

Starlette:符合ASGI协议的，实现的方式在__call__方法。

![call](/blog/starlette/call.png "__call__方法")

执行__call__方法会先给scope[‘app’]绑定为self。然后判断middleware_stack是否为空，如果为空就调用build_middleware_stack方法。

#### applications.Starlette.build_middleware_stack

这个方法主要做了一下这些工作：

1. 将self.exception_handlers参数中错误码与错误处理函数放到方法内部参数error_handler和exception_handlers。处理500的单独取出放到error_handler中，其他的在exception_handlers中。
2. 然后将中间按按照 ServerErrorMiddleware -> 用户自定义中间件 -> ExceptionMiddleware中排好序后再逆序套在Router中。逆序是为了保证ServerErrorMiddleware再最外层调用保证所有的错误都能被这个中间键给捕获。

#### 之后

之后就交给各个中间键过一遍然后有router进行下一步操作。

## Middleware

每一个Middleware都是一个ASGI实例对象。

starlette内置了==AuthenticationMiddleware==，==CORSMiddleware==，==ServerErrorMiddleware==，==ExceptionMiddleware==，==GZipMiddleware==，==HTTPSRedirectMiddleware==，==SessionMiddleware==，==TrustedHostMiddleware==，==WSGIMiddleware==这些中间件，其中ServerErrorMiddleware和ExceptionMiddleware比较特殊，默认就会分别加载最外层和最内层。

ServerErrorMiddleware: 异常打印错误堆栈信息，debug展示错误
ExceptionMiddleware：异常处理层，处理路由执行时抛出所有异常，

### 用户自定义中间件

用户自定义的中间件一般都要继承==BaseHTTPMiddleware==并重写dispatch方法。实际内置的几个中间件并没有实现这些。

这是test文件中写的一个自定义中间件

```python
class CustomMiddleware(BaseHTTPMiddleware):  
    async def dispatch(self, request, call_next):  
     # 这里你可以对请求做一些操作

  # call_next作用是调用下一个中间件或者app执行
  # 视图函数逻辑
        response = await call_next(request)  
        
        # 这里看名可知你可以对返回给用的数据做一些操作。
        response.headers["Custom-Header"] = "Example"  
        return response
```

这里中间件的设计很像golang中的gin的中间件,next之前就是request之后的就是reponse。

```go
func MiddleWare() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()
        fmt.Println("中间件开始执行了")
        // 设置变量到Context的key中，可以通过Get()取
        c.Set("request", "中间件")
        // 执行函数
        c.Next()
        // 中间件执行完后续的一些事情
        status := c.Writer.Status()
        fmt.Println("中间件执行完毕", status)
        t2 := time.Since(t)
        fmt.Println("time:", t2)
    }
}

// 使用gin.use()进行注册
gin.use(MiddleWare())

```

实际上在==BaseHTTPMiddleware==中的__call__方法进行了调度。call_next也是在这个函数中实现的。

#### BaseHTTPMiddleware

TODO: 需要继续做。

##### BaseHTTPMiddleware.\_\_call\_\_

主要将Scope用Request包装一层并调用dispatch函数，来再dispatch内部进行由用户自定义的一些对request和response的一些操作。

##### BaseHTTPMiddleware.\_\_call\_\_.call_next

会将控制权交给下一级的middleware调用直至返回response,将response返回给dispatch中，用户进一步操作。
在函数内部又实现了一些函数用来send和receive以及关闭链接，这部分的任务调度通过anyio实现，可以自己详细看一下。我对anyio了解不够深入，等后面看看anyio的用法然后把这部分给补充一下。

##### BaseHTTPMiddleware.dispatch

用户自定义的一些操作，再call_next之前我们能够对request进行操作，call_next之后就可以对response做一些操作。

## Routing

### Router

for循环路由匹配

![route-matches](/blog/starlette/route_matches.png "路由匹配")

> starlette 并没有向上文gin或者flask一样使用前缀树或者map为底层数据结构来处理路由问题。而是使用for循环然后进行正则匹配的方式来进行路由匹配。而mount会先匹配到mount然后再mount内部的router重复上文的逻辑进行再次匹配。所以当服务的路由数量不是很大的情况，性能并不会拖累整体的速度。但是当我们的路由超过百级别的情况下，尽量使用Mount来分好路由组，通过设计来实现一种跳表的数据结构，能够大大减缓路由匹配的原则。多少个路由才是使用mount或者mount的深度，没有测试过。后期会跑一下取一个值。

### Route

#### Route.matches

路由匹配规则，  
根据Scope中的path进行正则匹配，失败返回Match.None, {}。  
匹配成功：  
    将匹配到的结果用groupdicts转换为dict格式为 {“param1”:value, "param2": value}    key名为compile_path函数返回的path_regx中的\<params1\>/\<params2\>中的路径参数名称  
    value则是你传递过来的路径参数值。  
    再次判断请求方法是否存在且传递来的请求方法为methods参数中的。  
    如果不是则返回Match。Partial 和child_scope  
    是则返回Match.FULL 和 child_scope

### Mount

#### Mount.matches

Mount的匹配和Router的逻辑有一点不同  
Mount会在内部启用实例一个Router类（self.app ），  
当然你也可以直接实例化一个Router并通过app参数传递进来。  
将routers挂这个app中，然后在匹配到Mount后会按照router的match格式进行匹配。

### WebSocketRoute

#### request_response

将视图函数包装成一个ASGI函数
..待续

## Request

#### HTTPConnection

这里封装了一下property，将ASGI中的Scope的一些信息封装成一些方法用来调用

#### Request

这里有个小bug[starlette源码分析 - So1n blog](https://so1n.me/2021/11/15/starlette_source_code_analysis/#1-starlette%E7%9A%84%E5%BA%94%E7%94%A8)这篇博文提出了，主要是调用再中间件世界提前调用request实例的body导致。

Request中的receive会用来获取uvicorn中经过uvicorn.protocal序列化的http信息，而我们在dispatch方法调用前调用了body()方法，body会调用recive方法，导致后面的request实例都没法调用这个receive方法。(正常状态下应该从最深处返回receive，放到_body中,这样保证每个request都有uvicorn返回的数据)。

![route-matches](/blog/starlette/stream_receive.png "stream方法")

所以当我们在call_next方法中调用request.receive，控制权转到Uvicorn中的RequestResponseCycle的receive方法，由于之前已经调用过receive了导致self.message_event被clear重制， 然后再次调用的时候整个程序就在“await self.message_event.wait()”暂停直至超时。

![route-matches](/blog/starlette/wrap.png "stream方法")
![route-matches](/blog/starlette/await_stop.png "stream方法")

我们只需要将body取出来后再放进去一个recive放好scope和rec和send参数就行。(这部分我也没有吃太懂，先mark住等看看uvicorn.protocal这块的实现。)

```python
class DemoMiddleware(BaseHTTPMiddleware):

    async def proxy_get_body(self, request: Request) -> bytes:
        async def receive() -> Message:
            return {"type": "http.request", "body": body}
  
        body = await request.body()
        # 这里将recive放回去即可。
        request._receive = receive
        return body

  

    async def dispatch(
        self, request: Request, call_next: RequestResponseEndpoint
    ) -> Response:
        body = await self.proxy_get_body(request)
        return await call_next(request)
```

大体原因就是这个，如果不明白就注意在自定义中间价中一定要记得吧原先的scope保存，并声明一个新的recive函数，将body放到其中并传递给reques.\_receive参数。

## Response

#### Response

类有两个参数:

- media_type：放回的content-type
- charset: 返回用什么编码，默认是"utf-8"

代码很简单，自己看吧。

有一点很重要，就是返回数据后一个完整的请求并没有完成，还会再后台检查是否有后台任务，如果有的话会等待这个后台任务完成，然后整个请求周期才算全部完毕。

