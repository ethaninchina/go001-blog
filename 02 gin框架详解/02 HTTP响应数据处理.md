#### Gin框架对HTTP响应数据的处理
```
我们在上一篇中，以一个简单的Gin实例来具体讲解它内部是如何创建一个Http服务，并且注册一个路由来接收用户的请求，在示例程序中我们使用了Context引用对象的String方法来处理HTTP服务的数据响应，所以在整个Gin框架中紧跟Router模型结构的就要属Context结构了，该结构体主要用来处理整个HTTP请求的上下文数据，也是我们在开发HTTP服务中相对比较重要的一个结构体了。

# 深入Gin框架内幕(一)中的示例
$ cat case1.go
package main
import (
    "net/http"

    "github.com/gin-gonic/gin"
)
func main() {
    ginObj := gin.Default()
    ginObj.Any("/hello",func(c *gin.Context){
        c.String(http.StatusOK,"Hello BGBiao.")
    })
    ginObj.Run("localhost:8080")
}复制代码

我们可以看到，在使用Gin框架后，我们只需要很简单的代码，即可以快速运行一个返回Hello BGBiao.的HTTP服务，而在ginObj.Any方法中，我们传入了一个参数为Context引用类型的匿名函数，并在该函数内部采用String(code,data)方法来处理HTTP服务的响应数据(返回Hello BGBiao字符串)，这个时候，你可能会想，我们在企业内部都是前后端分离，通常情况下后端仅会提供RESTful API，并通过JSON格式的数据和前端进行交互，那么Gin是如何处理其他非字符串类型的数据响应呢，这也是我们接下来要主要讲的Context结构模型。
```


#### Gin框架中的Context结构体
```
注意: 在Gin框架中由Router结构体来负责路由和方法(URL和HTTP方法)的绑定，内的Handler采用Context结构体来处理具体的HTTP数据传输方式，比如HTTP头部，请求体参数，状态码以及响应体和其他的一些常见HTTP行为。
Context结构体:
type Context struct {
    // 一个包含size,status和ResponseWriter的结构体
    writermem responseWriter
    // http的请求体(指向原生的http.Request指针)
    Request   *http.Request
    // ResonseWriter接口
    Writer    ResponseWriter

    // 请求参数[]{"Key":"Value"}
    Params   Params
    handlers HandlersChain
    index int8
    // http请求的全路径地址
    fullPath string
    // gin框架的Engine结构体指针
    engine   *Engine
    // 每个请求的context中的唯一键值对
    Keys map[string]interface{}
    // 绑定到所有使用该context的handler/middlewares的错误列表
    Errors errorMsgs
    // 定义了允许的格式被用于内容协商(content)
    Accepted []string
    // queryCache 使用url.ParseQuery来缓存参数查询结果(c.Request.URL.Query())
    queryCache url.Values
    // formCache 使用url.ParseQuery来缓存PostForm包含的表单数据(来自POST,PATCH,PUT请求体参数)
    formCache url.Values
}

```


#### Context结构体常用的一些方法
```
基本方法:

Copy(): 返回当前正在使用的context的拷贝(context指针)，当这个context必须在goroutine中用时，该方法比较有用
HandlerName(): 返回当前主handler的名称(比如:handler为handleGetUsers(),该方法将返回"main.handleGetUsers")
HandlerNames(): 返回所有注册的handler的名称
Handler(): 返回当前的主handler(func (c *Context) Handler() HandlerFunc)
FullPath(): 返回一个匹配路由的全路径(uri: "/user/:id",c.FullPath() == "/user/:id" )



http常用方法:

ClientIP() string: 返回客户端ip(该方法会解析X-Real-IP,X-Forwarded-For)
ContentType() string: 返回HTTP的Content-Type头
IsWebsocket() bool: 返回是否为ws链接



流控相关的方法:

Next(): 该方法仅被使用在middleware中，它会在被调用的handler链内部执行pending handler
IsAborted(): 如果当前的context被终止了，该方法返回true
Abort(): 该函数可以从正在被调用中保护pending handler. 该方法停止后不会停止当前正在执行的handler. 比如我们有一个鉴权的中间件来验证请求是否有权限，如果认证失败了(用户信息异常等)，此时调用Abort()来确保后面的handler不再被调用
AbortWithStatus(code int): 同上，在会写入状态码。context.AbortWithStatus(401)即可表示上述的鉴权失败
AbortWithStatusJSON(code int, jsonObj interface{}): 同上，会再加响应数据.该方法会停止整个handler链，再写入状态码和json的响应体，同时也会设置Content-Type="application/json"
AbortWithError(code int, err error) *Error: 同上返回错误信息



错误管理:

Error(err error) *Error: 返回一些错误对象



元数据管理:

Set(key string, value interface{}): 给当前这个context设置一个新的键值对
Get(key string) (value interface{}, exists bool): 返回指定的key的值,以及是否存在
MustGet(key string) interface{}: 返回指定key的值，不存在则panic
GetString(key string) (s string): 以string类型返回指定的key
GetBool(key string) (b bool): 返回分配给该key的值(bool类型)
GetInt(key string) (i int):
GetStringSlice(key string) (ss []string): 返回key的slice类型
GetStringMap(key string) (sm map[string]interface{}): 返回interface{}类型的map结构
GetStringMapString(key string) (sms map[string]string): 返回string类型的map结构
GetStringMapStringSlice(key string) (smss map[string][]string): 同理



输入数据:

Param(key string) string: 返回URL的参数值(uri_patten: "/user/:id",url: "/user/john",c.Param("id") = "john")
Query(key string) string: 返回url中的查询参数值(url: "/path?id=1234&name=Manu&value=",c.Query("id")为1234,c.Query("name")为Manu,c.Query("value")为空)
DefaultQuery(key, defaultValue string) string: 返回url中的查询参数的默认值(同上，但是c.Query("value")就没有值，该方法可以设置默认值)
GetQuery(key string) (string, bool): 同Query()方法，并且会返回状态，如果对应的key不存在，返回("",false)
QueryArray(key string) []string: 返回指定key的对应的array(slice的长度取决于给定key的参数的数量)
GetQueryArray(key string) ([]string, bool): 同上，会返回状态
QueryMap(key string) map[string]string: 返回指定key对应map类型
GetQueryMap(key string) (map[string]string, bool): 同上，并且会返回状态
PostForm(key string) string: 该方法返回一个从POST 请求的urlencode表单或者multipart表单数据，不存在时返回空字符串
DefaultPostForm(key, defaultValue string) string: 同上，key不存在时返回默认值
GetPostForm(key string) (string, bool): 同PostForm()方法，并且会返回状态
PostFormArray(key string) []string: 该方法返回指定key的字符串类型的slice
GetPostFormArray(key string) ([]string, bool): 同上，并返回状态
PostFormMap(key string) map[string]string: 返回指定key的map类型
GetPostFormMap(key string) (map[string]string, bool): 同上，并返回状态
FormFile(name string) (*multipart.FileHeader, error): 返回指定key的第一个文件(用作文件上传)
MultipartForm() (*multipart.Form, error): 该方法解析multipart表单，包含file文件上传
SaveUploadedFile(file *multipart.FileHeader, dst string) error: 该方法用来上传指定的文件头到目标路径(dst)




Bind家族相关方法:

Bind(obj interface{}) error: 自动解析Content-Type并绑定到指定的binding引擎
BindJSON(obj interface{}) error: 同上，binding引擎为binding.JSON
BindXML(obj interface{}) error:
BindQuery(obj interface{}) error:
BindYAML(obj interface{}) error:
BindHeader(obj interface{}) error:
BindUri(obj interface{}) error: 使用binding.Uri来绑定传递的结构体指针
MustBindWith(obj interface{}, b binding.Binding) error: 使用指定的binding引擎来绑定传递的结构体指针(当有任何错误时，终止请求并返回400)




ShouldBind家族相关方法:

ShouldBind(obj interface{}) error: 同上述的Bind()方法，但是该方法在json结构无效时不会返回400
ShouldBindJSON(obj interface{}) error:
ShouldBindXML(obj interface{}) error:
ShouldBindQuery(obj interface{}) error:
ShouldBindYAML(obj interface{}) error:
ShouldBindHeader(obj interface{}) error:
ShouldBindUri(obj interface{}) error:
ShouldBindWith(obj interface{}, b binding.Binding) error: 等同于MustBindWith()方法
ShouldBindBodyWith(obj interface{}, bb binding.BindingBody) (err error): 和ShouldBindWith()方法相似，但是他会存储请求体到context中，当下次调用时可以重用(因为该方法是在binding之前读取body，因此在你只使用一次时，为了更好的性能还是使用ShouldBindWith会比较好)




HTTP响应相关的方法:

Status(code int): 设置http的响应码
Header(key, value string): 是c.Writer.Header().Set(key, value)的简单实现，在响应体重写入一个header，如果value为空，则相当于调用了c.Writer.Header().Del(key)
GetHeader(key string) string: 返回请求体重的header
GetRawData() ([]byte, error): 返回流式数据
SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool): 该方法将设置一个Set-Cookie到ResponseWriter的头中(注意:name必须是一个合法可用的名称,无效的coookie可能会被丢弃)
Cookie(name string) (string, error): 返回名称为name的cookie
Render(code int, r render.Render): 该方法写入响应头并调用render.Render去渲染数据
HTML(code int, name string, obj interface{}): 该方法使用指定文件模板名称去渲染http模板(同时会更新状态码并设置Content-Type as "text/html".)
IndentedJSON(code int, obj interface{}): 该方法会序列化对象obj为一个pretty JSON 数据到响应体中，同时设置Content-Type as "application/json"(pretty JSON需要消耗cpu和带宽，强烈建议生产使用Context.JSON())
SecureJSON(code int, obj interface{}): 同上，会序列化成 Secure Json
JSONP(code int, obj interface{}):
JSON(code int, obj interface{}): 序列化为JSON,并写Content-Type:"application/json"头
AsciiJSON(code int, obj interface{}):
PureJSON(code int, obj interface{}):
XML(code int, obj interface{}): 序列化成xml格式,并写Content-Type:"application/xml"
YAML(code int, obj interface{}): 序列化成yaml
ProtoBuf(code int, obj interface{}): 序列化成probuf
String(code int, format string, values ...interface{}): 将制定的string写入响应体
Redirect(code int, location string): 重定向
Data(code int, contentType string, data []byte): 写一些数据到响应体重，并更新响应码
DataFromReader(code int, contentLength int64, contentType string, reader io.Reader, extraHeaders map[string]string): 写一些制定模板的数据到响应体中，并更新状态码
File(filepath string): 以一种高效方式将制定文件写入响应体数据中
FileAttachment(filepath, filename string): 同上，但是在客户端文件会被直接下载下来
SSEvent(name string, message interface{}): 写Server-Sent Event到响应数据中
Stream(step func(w io.Writer) bool) bool: 发送一个流式的响应数据并返回状态
```


#### Gin实例示例
##### 返回json格式的数据
```
为了解决我们在开头提到的问题，我们将使用context引用对象的JSON家族方法来处理该需求


# 使用context来返回json格式的数据
$ cat case2.go
package main

import (
    "github.com/gin-gonic/gin"
)

// 我们定义一个通用的格式化的响应数据
// 在Data字段中采用空接口类型来实际存放我们的业务数据
type restData struct {
    Data        interface{}   `json:"data"`
    Message     string        `json:"message"`
    Status      bool          `json:"status"`
}

func main() {
    // mock一个http响应数据
    restdata := &restData{"Hello,BGBiao","",true}
    restdata1 := &restData{map[string]string{"name":"BGBiao","website":"https://bgbiao.top"},"",true}

    // 使用Gin框架启动一个http接口服务
    ginObj := gin.Default()
    ginObj.GET("/api/test",func(c *gin.Context){
       // 我们的handlerFunc中入参是一个Context结构的引用对象c
       // 因此我们可以使用Context中的JSON方法来返回一个json结构的数据
       // 可用的方法有如下几种，我们可以根据实际需求进行选择
       /*
          IndentedJSON(code int, obj interface{}): 带缩进的json(消耗cpu和mem)
          SecureJSON(code int, obj interface{}): 安全化json
          JSONP(code int, obj interface{})
          JSON(code int, obj interface{}): 序列化为JSON,并写Content-Type:"application/json"头
       */
       c.JSON(200,restdata)
    })
    ginObj.GET("/api/test1",func(c *gin.Context){
        c.IndentedJSON(200,restdata1)
    })

    ginObj.Run("localhost:8080")
}


# 实例运行(这里成功将我们写的两个api接口进行对外暴露)
$ go run case2.go
....
....
[GIN-debug] GET    /api/test                 --> main.main.func1 (3 handlers)
[GIN-debug] GET    /api/test1                --> main.main.func2 (3 handlers)

# 接口测试访问
$ curl localhost:8080/api/test
{"data":"Hello,BGBiao","message":"","status":true}
$ curl localhost:8080/api/test1
{
    "data": {
        "name": "BGBiao",
        "website": "https://bgbiao.top"
    },
    "message": "",
    "status": true
}%


当然上面我们仅以JSON格式来示例，类似的方式我们可以使用XML,YAML,ProtoBuf等方法来输出指定格式化后的数据。
```


##### 其他常用的基本方法
```
注意:在其他基本方法中我们仍然使用上述示例代码中的主逻辑，主要用来测试基本的方法.


# 我们在/api/test这个路由中增加如下两行代码
// 设置响应体中的自定义header(通常我们可以通过自定义头来实现一个内部标识)
c.Header("Api-Author","BGBiao")
// GetHeader方法用来获取指定的请求头，比如我们经常会使用请求中的token来进行接口的认证和鉴权
// 这里由于我们使用的restdata的指针，通过GetHeader方法获取到token赋值给Message
// ClientIP()方法用于获取客户端的ip地址
restdata.Message = fmt.Sprintf("token:%s 当前有效，客户端ip:%s",c.GetHeader("token"),c.ClientIP())

# 访问接口示例(我们可以看到在响应体中多了一个我们自定义的Api-Author头，并且我们将请求头token的值)
$ curl -H 'token:xxxxxxxx' localhost:8080/api/test -i
HTTP/1.1 200 OK
Api-Author: BGBiao
Content-Type: application/json; charset=utf-8
Date: Sun, 12 Jan 2020 14:41:01 GMT
Content-Length: 66

{"data":"Hello,BGBiao","message":"token:xxxxxxxx 当前有效，客户端ip:127.0.0.1","status":true}
```


##### 用户数据输入
```
当然到这里后，你可能还会有新的疑问，就是通常情况下，我们开发后端接口会提供一些具体的参数，通过一些具体数据提交来实现具体的业务逻辑处理，这些参数通常会分为如下三类:

使用HTTP GET方法获取到的url中的一些查询参数来执行更具体的业务逻辑(比如我们查询数据的指定条数之类的)
使用HTTP POST GET等其他方式以form表单方式提交的数据来验证和处理用户数据
在URL中获取一些可变参数(比如通常我们的url会定义为"/api/uid/:id"来表示用户id相关的接口，这个时候通常需要获取到url中的id字段)



以上的基本需求，几乎都可以在Context结构体的输入数据中找到响应的方法.
# 接下来，我们依然在上述的代码中进行修改，增加如下路由
$ cat case2.go
....
....
    // 比如我们该接口时用来获取全部数据，但是我们希望在url中增加参数来限制数据条数
    datas := []string{"Golang","Python","Docker","Kubernetes","CloudNative","DevOps"}
    ginObj.GET("/api/testdata",func(c *gin.Context){
        limit := c.Query("limit")
        // 其实既然这里我们已经确定需求了，当用户没有输入limit参数时我们就可以设置默认值
        // DefaultQuery("limit","1")
        // 同时我们其实也可以使用GetQuery方法来获取参数解析状态，即是否有对应的参数
        // 还有QueryArray和GetQueryArray类似的方法
        if limit != "" {
            num,_ := strconv.Atoi(limit)
            restdata1.Data = datas[:num]
        }else {
            restdata1.Data = datas
        }
        c.IndentedJSON(200,restdata1)

    })

    // 使用form表单方式提交数据
    ginObj.POST("/api/testdata",func(c *gin.Context){
        // 使用c.PostForm方法来提交一个data数据
        // 同时我们可以使用DefaultPostForm方法来给提交数据一个默认值，比如我们有些参数是希望有默认值的
        // 当然也可以使用GetPostForm，PostFormArray，PostFormArray方法来获取多个数据和状态
        // data := c.PostForm("data")
        // datas = append(datas,data)
        /* 这里可能会有个问题就是同时提交多个数据时，使用PostForm方法就会不那么好使了
        通常情况下回使用PostFormArray方法
        */
        data := c.PostFormArray("data")
        datas = append(datas,data...)
        restdata1.Data = datas
        c.IndentedJSON(200,restdata1)
    })

    // 获取url中的路径参数
    ginObj.GET("/api/testdata/:data",func(c *gin.Context){
        data := c.Param("data")
        for _,rawData := range datas {
            if data == rawData {
                restdata1.Data = data
                break
            }
        }
        if restdata1.Data != data {
            restdata1.Data = ""
            restdata1.Message = fmt.Sprintf("%v 不存在",data)
            restdata1.Status = false
        }
        c.IndentedJSON(200,restdata1)
    })
....
....



# 请求示例接口
# 我们可以看到使用GET方法默认会获取到全部数据，但是如果有了limit参数后，我们就可以限制数据的条数
$ curl -H 'token:xxxxxxxx' localhost:8080/api/testdata
{
    "data": [
        "Golang",
        "Python",
        "Docker",
        "Kubernetes",
        "CloudNative",
        "DevOps"
    ],
    "message": "",
    "status": true
}%
$ curl -H 'token:xxxxxxxx' "localhost:8080/api/testdata?limit=2"
{
    "data": [
        "Golang",
        "Python"
    ],
    "message": "",
    "status": true
}%

# 当我们使用post接口往服务提交数据时，就可以让服务端按照需求进行数据处理
curl -X POST  -d data="vue" "localhost:8080/api/testdata"
{
    "data": [
        "Golang",
        "Python",
        "Docker",
        "Kubernetes",
        "CloudNative",
        "DevOps",
        "vue"
    ],
    "message": "",
    "status": true
}%

# 当我们同时需要提交多份数据时，可以使用PostFormArray方法，同时提交多份数据(可以理解为批量提交)
$ curl -X POST -d data="vue" -d data="Rust" "localhost:8080/api/testdata"

# 获取URL中的参数值
$ curl "localhost:8080/api/testdata/Golang"
{
    "data": "Golang",
    "message": "",
    "status": true
}%

$ curl "localhost:8080/api/testdata/Java"
{
    "data": "",
    "message": "Java 不存在",
    "status": false
}%

```






