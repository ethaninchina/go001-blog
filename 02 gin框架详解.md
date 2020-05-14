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
