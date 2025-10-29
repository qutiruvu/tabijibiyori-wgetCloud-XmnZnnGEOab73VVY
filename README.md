[合集 - Gin笔记(2)](https://github.com)

[1.Gin笔记一之项目建立与运行10-23](https://github.com/hunterxiong/p/19161846):[nuts坚果](https://jianyong.org)

2.Gin笔记二之gin.Engine和路由设置10-29

收起

> 本文首发于公众号：Hunter后端
>
> 原文链接：[Gin笔记二之gin.Engine和路由设置](https://github.com)

这一篇笔记主要介绍 `gin.Engine`，设置路由等操作，以下是本篇笔记目录：

1. gin.Default() 和 gin.New()
2. HTTP 方法
3. 路由分组与中间件

### 1、gin.Default() 和 gin.New()

前面第一篇笔记介绍，创建一个 gin 的路由引擎使用的函数是 `gin.Default()`，返回的类型是 `*gin.Engine`，我们可以使用其创建路由和路由组。

除了这个函数外，还有一个 `gin.New()`，其返回的也是 `*gin.Engine`，但是不一样的是 `gin.Default()` 会对 `gin.Engine` 添加默认的 `Logger()` 和 `Recovery()` 中间件。

这两个函数大致内容如下：

```
func New(opts ...OptionFunc) *Engine {
    ...
}

func Default(opts ...OptionFunc) *Engine {
    ...
    engine := New()
    engine.Use(Logger(), Recovery())
    ...
}
```

我们使用第一篇笔记中使用 `debug` 模式运行系统后输出的信息可以再看一下：

```
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /test                     --> main.main.func1 (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on :9898
```

我们使用的是 `gin.Default()` 运行的系统，所以在第一行告诉我们创建了一个带有  `Logger and Recovery` 中间件的 `Engine`。

同时第三行输出路由信息的地方，标明了这个路由指向的处理函数，后面的括号里是 `3 handlers`，这个意思是除了我们处理路由的 `handler`，还有两个默认的中间件 `handler`，也就是这里的 `Logger()` 和 `Recovery()` 中间件。

下面介绍一下 `Logger()` 和 `Recovery()` 这两个 `handler` 的作用。

#### 1. Logger()

默认的 `Logger()` 会输出接口调用的信息，比如第一篇中我们定义了一个 `/test` 接口，当我们调用这个接口的时候，控制台会输出下面这条信息：

```
[GIN] 2025/08/19 - 23:15:26 | 200 |      36.666µs |       127.0.0.1 | GET      "/test"
```

可以看到日志中会包含请求时间、返回的 HTTP 状态码、请求耗时、调用方 ip、请求方式和接口名称等。

这条日志信息的输出就是 `Logger()` 这个中间件起的作用。

在其内部，会调用一个 `LoggerWithConfig()` 函数，获取到请求的 ip、记录调用时间、调用方式等信息，然后进行输出，下面是部分源码信息：

```
param.TimeStamp = time.Now()
param.Latency = param.TimeStamp.Sub(start)

param.ClientIP = c.ClientIP()
param.Method = c.Request.Method
param.StatusCode = c.Writer.Status()
param.ErrorMessage = c.Errors.ByType(ErrorTypePrivate).String()
```

#### 2. Recovery()

`Recovery()` 中间件则可以为我们捕获程序中未处理的 panic，记录错误信息并返回 500 状态码信息，比如我们在第一篇笔记中使用的 `TestHandler` 函数，我们在其中加一个除数为 0 的错误：

```
func TestHandler(c *gin.Context) {
    response := TestResponse{
        Code:    0,
        Message: "success",
    }
    a := 0
    fmt.Println(1 / a)
    c.JSON(http.StatusOK, response)
}
```

在接口调用的时候，如果我们使用的是 `gin.Default()`，那么客户端不会报错，而是会收到一个 HTTP 状态码为 500 的报错信息，而如果使用的是 `gin.New()`，客户端则会直接发生错误。

总的来说，`Logger()` 和 `Recovery()` 这两个的中间件是 `gin` 框架为我们默认添加的对于开发者来说较为友好的两个操作，在后面介绍中间件的时候，我们也可以手动实现这两个功能。

### 2、HTTP 方法

`gin.Engine` 支持配置 HTTP 多个方法，比如 GET、POST、PUT、DELETE 等。

以第一篇笔记中的代码为例，其设置方法如下：

```
r.GET("/test", TestHandler)
r.POST("/test", TestHandler)
r.PUT("/test", TestHandler)
r.DELETE("/test", TestHandler)
```

### 3、路由分组与中间件

除了设置单个路由，我们还可以对路由进行分组设置，比如需要控制版本，或者模块设置需要统一的前缀，又或者是需要统一设置中间件功能的时候。

其整体代码示例如下：

```
package main

import (
    "fmt"
    "net/http"
    "github.com/gin-gonic/gin"
)

type TestResponse struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func TestHandler(c *gin.Context) {
    response := TestResponse{
        Code:    0,
        Message: "success",
    }
    c.JSON(http.StatusOK, response)
}

func main() {
    r := gin.Default()

    v1 := r.Group("/v1")
    {
        v1.GET("/test", TestHandler)
    }

    err := r.Run(":9898")
    if err != nil {
        fmt.Println("gin run in 9898 error:", err)
    }
}
```

这里，我们设置了一个路由名称以 `v1` 为前缀的路由组，其下每个路由的访问都需要带有 `/v1`，这样就实现了统一设置路由前缀的功能。

而如果我们需要向其中添加中间件的时候，也可以不用挨个路由进行设置，而是在 `v1` 路由组的设置中就可以实现，比如：

```
v1 := r.Group("/v1", Middleware1, Middleware2)
```

这样，其下每个路由的 `handler` 函数在调用前就都会先调用 `Middleware1` 和 `Middleware2` 这两个中间件。

以上就是本篇笔记关于 `gin.Engine` 的全部内容，其实中间件的相关操作也应该属于 `gin.Engine` 的内容，但是那部分需要介绍的知识点和想要用于介绍的代码示例略多，所以就单独开一篇笔记在后面再介绍。
