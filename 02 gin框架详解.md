#### Gin框架介绍
```
Gin是一个用 Go (Golang) 编写的 web 框架。它是一个类似于martini但性能更好的API框架，不同于谢大主导的Beegoweb框架，后者更像是Python语言中的Django框架，内部包含了开发一个web程序所需的各种组件。
如果你是性能和高效的追求者，我相信你会像我一样爱上Gin。

同时，不同于其他Golang语言的API框架，该框架社区十分活跃，并且主程仍然在不断更新和改进，我们知道通常情况下在选择一个开源的软件或者相关库时，社区的活跃度以及项目的更新情况会非常重要(考虑到后期的维护和性能和特性问题)。
另外一方面，该框架官方提供了很多简单的示例来供我们快速运行一个期望的http服务，这对于一个刚转入Golang进行业务开发的同学来说是一个非常好的开始。

Gin官方提供的各种HTTP服务示例
https://github.com/gin-gonic/gin#api-examples

当然我在很长一段时间也仅是去看官方示例来快速熟悉并实现自己的业务需求，但当有一些特殊的需求时通常去查看官方的具体实现来满足需求，长期如此，不仅耗时且效率极低，因此我产生了将核心源码探究一番的兴趣，希望能通过源码方式来对Gin框架有个深入的学习。
```


#### Gin框架中的几个核心结构
```
我们都知道开发一个HTTP服务，首先需要启动一个TCP监听，然后需要有一些列的handler来处理具体的业务逻辑，最后在再将具体的业务逻辑通过HTTP协议约定和相关的Method和URL进行绑定，以此来对外提供具体功能的HTTP服务。那么在Gin框架对应的就是如下几个模型，我们将一起学习Gin的实现。

Gin框架中的几个重要的模型:
Engine: 用来初始化一个gin对象实例，在该对象实例中主要包含了一些框架的基础功能，比如日志，中间件设置，路由控制(组)，以及handlercontext等相关方法.源码文件
Router: 用来定义各种路由规则和条件，并通过HTTP服务将具体的路由注册到一个由context实现的handler中
Context: Context是框架中非常重要的一点，它允许我们在中间件间共享变量，管理整个流程，验证请求的json以及提供一个json的响应体. 通常情况下我们的业务逻辑处理也是在整个Context引用对象中进行实现的.
Bind: 在Context中我们已经可以获取到请求的详细信息，比如HTTP请求头和请求体，但是我们需要根据不同的HTTP协议参数来获取相应的格式化数据来处理底层的业务逻辑，就需要使用Bind相关的结构方法来解析context中的HTTP数据
```


#### Gin框架中的Engine结构体
```
当我们在使用框架[Gin](https://github.com/gin-gonic/gin)来创建一个HTTP服务时，首先我们需要初始化一个实例，在Engine结构体中就包含了实例的一些基本属性和实例化的一些方法。


Engine结构体:
type Engine struct {
    // 路由组，在实际开发过程中我们通常会使用路由组来组织和管理一些列的路由. 比如: /apis/,/v1/等分组路由
    RouterGroup
    // 开启自动重定向。如果当前路由没有匹配到，但是存在不带/开头的handler就会重定向. 比如: 用户输入/foo/但是存在一个/foo 就会自动重定向到该handler，并且会向客户端返回301或者307状态码(区别在于GET方法和其他方法)
    RedirectTrailingSlash bool
    // 如果开启该参数，没有handler注册时，路由会尝试自己去修复当前的请求地址.
    // 修复流程:
    // 1.首位多余元素会被删除(../ or //); 2.然后路由会对新的路径进行不区分大小写的查找;3.如果能正常找到对应的handler，路由就会重定向到正确的handler上并返回301或者307.(比如: 用户访问/FOO 和 /..//Foo可能会被重定向到/foo这个路由上)
    RedirectFixedPath bool
    // 如果开启该参数，当当前请求不能被路由时，路由会自己去检查其他方法是否被允许.在这种情况下会响应"Method Not Allowed"，并返回状态码405; 如果没有其他方法被允许，将会委托给NotFound的handler
    HandleMethodNotAllowed bool
    // 是否转发客户端ip
    ForwardedByClientIP    bool
    // 如果开启将会在请求中增加一个以"X-AppEngine..."开头的header
    AppEngine bool
    // 如果开启将会使用url.RawPath去查找参数(默认:false)
    UseRawPath bool
    // 如果开启，请求路径将不会被转义. 如果UseRawPath为false，该参数实际上就为true(因为使用的是url.Path)
    UnescapePathValues bool
    // maxMemory参数的值(http.Request的ParseMultipartForm调用时的参数)
    MaxMultipartMemory int64
    // 是否删除额外的反斜线(开始时可解析有额外斜线的请求)
    RemoveExtraSlash bool
    // 分隔符(render.Delims表示使用HTML渲染的一组左右分隔符,具体可见html/template库)
    delims           render.Delims
    // 设置在Context.SecureJSON中国的json前缀
    secureJsonPrefix string
    // 返回一个HTMLRender接口(用于渲染HTMLProduction和HTMLDebug两个结构体类型的模板)
    HTMLRender       render.HTMLRender
    // html/template包中的FuncMap map[string]interface{} ,用来定义从名称到函数的映射
    FuncMap          template.FuncMap
    // 以下是gin框架内部定义的一些属性
    // HandlersChain 是一个HandlerFunc 的数组(HandlerFunc其实就是一个Context的指针,Context会在下一节讲解)
    allNoRoute       HandlersChain
    allNoMethod      HandlersChain
    noRoute          HandlersChain
    noMethod         HandlersChain
    // 这里定义了一个可以临时存取对象的集合(sync.Pool是线程安全的，主要用来缓存为使用的item以减少GC压力，使得创建高效且线程安全的空闲队列)
    pool             sync.Pool
    // methodTrees是methodTree的切片(methodTree是一个包含请求方法和node指针的结构体,node是一个管理path的节点树)
    trees            methodTrees
}

```

```
HandlerFunc定义:

// 定义了一个可以被中间件使用的handler
type HandlerFunc func(*Context)
```

```
初始化Engine的方式:

New(): 该函数返回一个默认的Engine引用实例(开启了自动重定向,转发客户端ip和禁止请求路径转义)
Default(): 内部调用New()函数，但是增加了Logger和Recovery两个中间件
```


```
Engine对外常用的方法:

Delims(left, right string) *Engine: 给创建好的gin实例指定模板引擎的左右分割符
SecureJsonPrefix(prefix string) *Engine: 给创建好的gin实例设置secureJsonPrefixi
SetHTMLTemplate(templ *template.Template): 该方法会gin实实例绑定一个模板引擎(内部其实是设置了engine的HTMLRender属性)
LoadHTMLGlob(pattern string): 该方法用来加载glob模式(类似于shell中的正则)的html模板文件，然后将结果和HTML模板引擎关联(内部调用SetHTMLTemplate方法将全部匹配到模板注册进去)
LoadHTMLFiles(files ...string): 该方法用上，需要指定一组模板文件名
SetFuncMap(funcMap template.FuncMap): 该方法会设置一个FuncMap给template.FuncMap使用(内部其实设置了engine的FuncMap)
NoRoute(handlers ...HandlerFunc): 该方法为NoRoute增加一些handler，它默认会返回404(通常在企业里，404我们会处理的比较优雅一些，比如给一些企业的静态页啥的)
NoMethod(handlers ...HandlerFunc): 同上，该方法用于给NoMethod增加handler，默认返回405
Use(middleware ...HandlerFunc) IRoutes: 该方法用于绑定一个全局的中间件给router. 通过该方法注册的中间件将包含在每个请求的handler chain中(比如可以在这里使用一些logger或者error相关的中间件). 在上面初始化实例的Default()函数中其实内部使用了engine.Use(Logger(), Recovery())来加载logger和recovery中间件
Routes() (routes RoutesInfo): 该方法用来返回一个路由列表信息RoutesInfo(一个路由信息RouteInfo中包含Method,Path,Handler,HandlerFunc)，该方法底层调用engine的trees来获取一些router必要的信息.
Run(addr ...string) (err error): 该方法会绑定router到http.Server中并开启一个http监听来接收http请求. 该方法其实是http.ListenAndServe(addr, engine)的简单实现. 注意:该方法除非出现错误，否则会无期限阻塞调用goroutine来接收请求(engine内部只要实现了http.ServeHTTP方法即可)
RunTLS(addr, certFile, keyFile string) (err error): 同上，以https方式运行服务
RunUnix(file string) (err error): 同Run(addr)方法，通过指定的unix socket文件运行服务
RunFd(fd int) (err error): 同Run(addr)方法，通过指定的文件描述符(fd)来运行服务
RunListener(listener net.Listener) (err error): 同Run(addr)，通过制定的net.Listener来运行服务
ServeHTTP(w http.ResponseWriter, req *http.Request): 该方法遵循了http.Handler的接口规范，可使gin内部调用http.ListenAndServe来启动一个http服务
HandleContext(c *Context): 该方法会重新确认一个被重写的context(可以通过c.Request.URL.Path来实现). 需要注意的是该方法可能造成context的循环使用(会绕死你,谨慎使用)

```

#### Gin框架中的Router
```
使用Engine结构体中提供的相关方法，我们就可以快速的启动一个HTTP服务了，但是如何对外暴露一个URL来简单实现一个HTTP的数据传输呢，这个时候就需要使用Router中的方法了。
```

```
Gin框架中Router相关的结构体:

RouterGroup: 该结构体被用来在Gin内部配置一个路由，一个RouterGroup被用来关联URL前缀和一组具体的handler业务逻辑
IRoutes: IRoutes是一个定了了所有路由处理的接口(包含一些常用的HTTP方法)
IRouter: IRouter则是一个包含单个路由和路由组的所有路由处理的接口


// RouterGroup 结构体
type RouterGroup struct {
    Handlers HandlersChain
    basePath string
    engine   *Engine
    root     bool
}

// IRoutes 接口
type IRoutes interface {
    Use(...HandlerFunc) IRoutes

    Handle(string, string, ...HandlerFunc) IRoutes
    Any(string, ...HandlerFunc) IRoutes
    GET(string, ...HandlerFunc) IRoutes
    POST(string, ...HandlerFunc) IRoutes
    DELETE(string, ...HandlerFunc) IRoutes
    PATCH(string, ...HandlerFunc) IRoutes
    PUT(string, ...HandlerFunc) IRoutes
    OPTIONS(string, ...HandlerFunc) IRoutes
    HEAD(string, ...HandlerFunc) IRoutes

    StaticFile(string, string) IRoutes
    Static(string, string) IRoutes
    StaticFS(string, http.FileSystem) IRoutes
}

// IRouter接口
type IRouter interface {
    IRoutes
    Group(string, ...HandlerFunc) *RouterGroup
}
```

```
还记得在上一节中我们的Engine结构体中有一个RouterGroup字段吗，该字段会在我们创建一个Engine的gin实例后帮助我们初始化一个默认的RouterGroup实例。
比如在Engine结构体的中的New()函数，会初始化一个带有如下RouterGroup的gin实例，并将gin实例注册到RouterGroup的engine字段.源码文件
// https://github.com/gin-gonic/gin/blob/master/gin.go#L129
        RouterGroup: RouterGroup{
            Handlers: nil,
            basePath: "/",
            root:     true,
        },

```


```
RouterGroup结构体对外暴露的常用方法:

Use(middleware ...HandlerFunc) IRoutes: 注册一个中间件并返回Iroutes接口
Group(relativePath string, handlers ...HandlerFunc) *RouterGroup: Group方法会创建一个新的路由组。通常我们会创建一个公共的中间件或者是具有相同前缀的路由，来归并到一个路由组
BasePath() string: 该方法用来返回一个路由组初始路径(比如 v := router.Group("/rest/n/v1/api")，则v.BasePath()就是"/rest/n/v1/api")
Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes: 该方法会使用给定的HTTP方法和URL来注册一个新的handler。(最后一个handler应该是真正的处理程序，其他的应该是在不同的路由之间共享的中间件)。注意:内部调用了一个handle(httpMethod, relativePath string, handlers HandlersChain)的私有方法来处理核心逻辑
POST(relativePath string, handlers ...HandlerFunc) IRoutes: 该方法是router.Handle("POST", path, handle)的快速实现，
GET(relativePath string, handlers ...HandlerFunc) IRoutes: 同上
DELETE(relativePath string, handlers ...HandlerFunc) IRoutes: 同上
PATCH(relativePath string, handlers ...HandlerFunc) IRoutes : 同上
PUT(relativePath string, handlers ...HandlerFunc) IRoutes: 同上
OPTIONS(relativePath string, handlers ...HandlerFunc) IRoutes: 同上
HEAD(relativePath string, handlers ...HandlerFunc) IRoutes: 同上
Any(relativePath string, handlers ...HandlerFunc) IRoutes: 同上，会将HTTP的所有方法都注册上去
StaticFile(relativePath, filepath string) IRoutes: 该方法用来注册一台路由来服务本地文件系统的单个文件，比如:router.StaticFile("favicon.ico", "./resources/favicon.ico")
Static(relativePath, root string) IRoutes: 该方法用来提供一个指定文件系统根路径的的路由，内部调用group.StaticFS(path,Dir(root,false))来提供服务
StaticFS(relativePath string, fs http.FileSystem) IRoutes: 指定文件系统(http.FileSystem)来创建一个服务
```


#### Gin实例示例
```
有了上面两个核心模型Engine和RouteGroup的了解，此时我们就可以通过Gin框架快速来创建一个简单HTTP服务了。
```

```
默认路由
# 测试示例
$ cat case1.go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    // 使用Default()函数来初始化一个gin实例(engine结构体的引用对象)
    // Default函数内部调用New()函数来初始化一个gin实例，
    // 同时使用Use(middleware ...HandlerFunc) IRoutes 方法注册了Logger和Recovery两个中间件
    // 在New()初始化gin实例的过程中还默认初始化了一个bathPath为"/"的RouterGroup，其实就是一个router实例
    ginObj := gin.Default()
    // 由于RouterGroup在engine结构体中是一个匿名对象，因此实例化的engine引用对象就可以直接操作RouterGroup结构体里对外暴露的所有方法
    // 这里我们尝试注册一个包含所有HTTP方法的路由
    // https://github.com/gin-gonic/gin/blob/master/routergroup.go#L133
    // 而在RouterGroup的各种对外暴露的方法中，底层调用了 Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes方法，后面可以传入多个handler来处理具体的业务逻辑，当handler有多个时最后一个处理实际的业务请求，前面的handler来处理中间件和共享的组件

    // 而HandlerFunc 其实就是一个func(*Context)的匿名函数.Context会在下一节具体分析
    ginObj.Any("/hello",func(c *gin.Context){
        // context结构体相关的方法下一节会具体分析，这里是一个简单的示例
        c.String(http.StatusOK,"Hello BGBiao.")
    })


    // 当所有的路由注册之后，我们可以使用gin的结构体方法(engine结构体的引用对象)来实际运行HTTP服务，以接收用户的http请求
    // 我们前面说过该方法除非出现错误，否则会无期限阻塞调用goroutine来接收请求
    ginObj.Run("localhost:8080")
}




# 运行实例
$ go run case1.go
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] POST   /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] PUT    /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] PATCH  /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] HEAD   /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] OPTIONS /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] DELETE /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] CONNECT /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] TRACE  /hello                    --> main.main.func1 (3 handlers)
[GIN-debug] Listening and serving HTTP on localhost:8080

# 模拟请求(因为我们注册了全部的HTTP方法的路由)
$ curl localhost:8080/hello
Hello BGBiao.%
$ curl -X POST  localhost:8080/hello
Hello BGBiao.%
$ curl -X DELETE   localhost:8080/hello
Hello BGBiao.%
$ curl -X TRACE   localhost:8080/hello
Hello BGBiao.%

```







