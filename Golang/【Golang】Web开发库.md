# 【Golang】Web 开发库

* [【Golang】Web 开发库](#golangweb-开发库)
   * [Web 框架](#web-框架)
      * [Gin](#gin)
      * [Iris](#iris)
      * [Beego](#beego)
      * [Hertz](#hertz)
   * [数据存储](#数据存储)
      * [Gorm](#gorm)
   * [请求响应](#请求响应)
      * [Json](#json)
      * [Validator1](#validator1)
      * [Validator2](#validator2)
   * [工具](#工具)
      * [UUID](#uuid)

## Web 框架
### Gin
gin 包是一个简单易用的轻量级 Web 框架，专注于提供高性能的路由和中间件支持，并且具有最活跃的社区支持，适用于 Web API 开发。其安装路径为 `github.com/gin-gonic/gin`，[官方文档](https://gin-gonic.com/zh-cn/docs/introduction/)、[第三方文档](https://www.topgoer.com/gin%E6%A1%86%E6%9E%B6/)

简单的初始化和启动实例如下：

``` go
import "github.com/gin-gonic/gin"

r := gin.Default()

// 注册中间件
r.Use(func(c *gin.Context) {
    fmt.Print("middleware action")
})

// 注册路由和处理器
r.GET("/ping", func(c *gin.Context) {
	c.JSON(200, jsonObj)
})

// 基于 net/http 运行
router.Run(addr)
```

### Iris
iris 包是一个兼顾性能和功能的 Web 框架，旨在保持高性能的前提下，提供了更丰富的功能和中间件，包括模板引擎、WebSocket、HTTP/2、安全认证等，同时适配 MVC 和 Web API 开发。其安装路径为 `github.com/kataras/iris/v12`，[官方文档A](https://www.iris-go.com/docs)、[官方文档B](https://docs.iris-go.com/iris/)、[第三方文档](https://learnku.com/docs/iris-go/10)

简单的初始化和启动实例如下：

``` go
import "github.com/kataras/iris/v12"

app := iris.New()

// 注册中间件
app.Use(func(c iris.Context) {
    fmt.Print("middleware action")
})

// 注册路由和处理器
app.GET("/ping", func(c *gin.Context) {
	c.JSON(200, jsonObj)
})

// 基于 net/http 运行
app.Listen(addr)
```

### Beego
beego 包是一个全功能的传统 Web 框架，专注于提供高效快速的开发能力，以简单易用的方式提供了丰富的功能和中间件，额外包括管理后台、ORM、缓存管理、定时任务等，社区还提供工具链、i18n 等支持，适用于 MVC 开发。其安装路径为 `github.com/beego/beego/v2`，[官方文档](https://beego.gocn.vip/beego/zh/)，[第三方文档](https://git-books.github.io/books/beego/)

简单的初始化和启动实例如下：

``` go
import "github.com/beego/beego/v2/server/web"

// 定义中间件
func middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Print("middleware action")
		next.ServeHTTP(w, r)
	})
}

// 注册路由和处理器，既支持以下函数风格注册
// 还支持控制器风格注册，即基于控制器名和其方法名决定路由
web.Get("/hello", func(ctx *context.Context) {
    ctrl.TplName = "hello_world.html"
    ctrl.Data["name"] = "xxx"
    _ = ctrl.Render()
})

// 注册中间件，并基于 net/http 运行
web.Run(addr, middleware)
```

### Hertz
hertz 包是一个具有超高性能和扩展性的 Web 框架，旨在同时满足定制化要求和性能要求，采用了自身社区提供的高性能网络库，通过分层设计获得很高的功能和协议扩展，适用于 Web API 开发。其安装路径为 `github.com/cloudwego/hertz`，[官方文档](https://www.cloudwego.io/zh/docs/)

简单的初始化和启动实例如下：

``` go
import (
    "context"
    "github.com/cloudwego/hertz/pkg/app"
    "github.com/cloudwego/hertz/pkg/app/server"
    "github.com/cloudwego/hertz/pkg/protocol/consts"
)

h := server.Default()

// 注册中间件
h.Use(func(c context.Context, ctx *app.RequestContext) {
    fmt.Print("middleware action")
})

// 注册路由和处理器
h.GET("/ping", func(c context.Context, ctx *app.RequestContext) {
        ctx.JSON(consts.StatusOK, jsonObj)
})

// 基于采用了 netpoll 的 hertz http 运行
h.Spin()
```

## 数据存储
### Gorm
gorm 包是一个灵活好用的 ORM 库，使编程语言对象与数据库模型之间能关联起来，支持非常多的功能特性和主流数据库，其安装路径为 `gorm.io/gorm`，[官方文档](https://gorm.io/docs/)

gorm 通过抽象的数据库驱动，来连接数据库和建立会话，从而支持多种不同的数据库。其中官方提供了一系列主流的数据库驱动，包括 `sqlite`、`mysql`、`postgres`、`sqlserver` 和 `clickhouse`，其导入路径为 `gorm.io/driver/XXX`，除此之外还可以使用实现了相关接口的第三方数据库驱动或者自定义数据库驱动

通过数据库驱动 `XXXDriver` 连接数据库的方式如下：

``` go
var db *gorm.DB
db, err = gorm.Open(XXXDriver.New())
// 或者 db, err = gorm.Open(driver.Open())
if err != nil {
    panic(fmt.Errorf("cannot establish db connection: %w", err))
}
```

gorm 可将自定义的 `struct` 关联到数据库模型，并且 `struct` 中所定义的字段，可以是 Go 基本数据类型，或者是实现了 `Scanner` 和 `Valuer` 接口的自定义数据类型，可参考 [常用自定义字段](https://github.com/go-gorm/datatypes)，并且字段还支持通过 `gorm` 标签可以为字段设置各种选项

可以被关联到数据库模型的 `struct` 实例如下：

``` go
type XXXModel struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

## 请求响应
### Json
json 包提供了针对 JSON 的编解码功能，是 Go 标准库，[官方文档](https://pkg.go.dev/encoding/json)

JSON 表现为字节流 `[]byte`，任意类型都能编码为 JSON，且可以由 JSON 解码还原，编解码的用法示例如下：

``` go
import "encoding/json"

// 根据任意类型的 myObject 对象或指针，编码为 JSON 字节流
rst, err := json.Marshal(myObject)
if err != nil {
    panic(fmt.Errorf("encode json failed: %w", err))
}


// 将 JSON 字节流解码，保存到任意类型的 myObject 指针中
var myObject MyType
rst, err := Unmarshal(bytes, &myObject)
if err != nil {
    panic(fmt.Errorf("decode json failed: %w", err))
}
```

编解码都需要一定的规则，使得 Go 类型和 JSON 类型能够互相转换，默认的编码和解码规则如下：

```
----- 编码规则 -----

GO 类型           ->    JSON 类型
--------         ->    --------
布尔值 bool       ->    布尔值 boolean
所有数值型 int     ->    数字 number
字符串 string     ->    字符串 string (以 utf8 编码)
切片 []byte       ->    字符串 string (以 base64 编码）
其他切片 slice     ->    数组 array
数组 array        ->    数组 array
结构体 struct     ->    对象（结构体中可导出的字段才会被转换为对象的字符串索引）
映射 Map          ->    对象（映射必须为 map[string]interface{}
nil              ->    null

----- 解码规则 -----

JSON 类型         ->    GO 类型
--------         ->    --------
布尔值 boolean    ->    布尔值 bool
数字 number       ->    浮点数 float64
字符串 string     ->    字符串 string
数组 array        ->    []interface{}
对象 object       ->    map[string]interface{}
null             ->    nil
```

通过设置结构体的字段标签，自定义该字段的编码和解码规则：

``` go
type T struct {
    // 标签格式，key 表示字符串索引，flag 表示标志项
    f `json:"[key][,flag][,flag]..."`
    
    // `-` 表示不做编解码处理
    f1 `json:"-"`
    // 指定 key 作为编解码时匹配的字符串索引
    f2 `json:"name"`
    // 指定 `-` 作为编解码时匹配的字符串索引
    f3 `json:"-,"`
    // 表示当值为空时（长度为0或者为零值），不做编码处理    
    f4 `json:",omitempty"`
}
```

当编码的来源类型为结构体时，其字段会优先根据标签定义的 Key 作为 JSON 字符串索引，其次再直接使用字段名作为 JSON 字符串索引

``` go
type myType struct {
    AAAA string
    BBBB string `json:"bbbb"`
}

-> Marshal ->

{
    "AAAA": "",
    "bbbb": ""
}
```

当解码的目标类型为结构体时，每个 JSON 字符串索引要按照以下规则，按顺序来匹配目标结构体中的可导出字段：
1. 名字等于某个标签指定 Key 的字段不参与匹配
2. 每个字段可以通过标签指定 Key，否则使用字段名作为 Key
3. Key 完全等于该 JSON 字符串索引的字段
4. 名字不区分大小写时等于 Key，且位置较前的字段

若成功匹配，则将其对应 JSON 值解码为所匹配字段的值，否则丢弃该 JSON 值，如果两个 JSON 字符串索引都匹配了同一个字段，则第二个SON 字符串索引的对应值会覆盖掉第一个

``` go
type myType struct {
    AAAA string
    AAaa string
    BBBB string `json:"AAAA"`
    CCCC string
}

{
    "AAAA": "1",    // 匹配 BBBB
    "AAaa": "2",    // 匹配 AAaa
    "aaaa": "3",    // 匹配 AAaa，覆盖
    "CCCC": "4",    // 匹配 CCCC
    "DDDD": "5"     // 不匹配，丢弃
}

-> Unmarshal ->

myType {
    AAAA: ""
    AAaa: "3"
    BBBB: "1"
    CCCC: "4"
}
```

通过为自定义类型的声明以下方法，可以自定义该类型的编码和解码过程：

``` go
// 为类型 MyType 定义编码过程，额外增加前缀 `type-`
func (mt MyType) MarshalJSON() ([]byte, error) {
    var s string
    switch mt {
    default:
        s = "unknown"
    case "a":
        s = "type-a"
    case "b":
        s = "type-b"
    }
    return json.Marshal(s)
}

// 为类型 MyType 定义解码过程，额外去除前缀 `type-`
func (mt MyType) UnmarshalJSON(b []byte) error {
    var s string
    if err := json.Unmarshal(b, &s); err != nil {
		return err
    } 
    switch s {
    default:
        s = "unknown"
    case "type-a":
        s = "a"
    case "type-b":
        s = "b"
    }
    return nil
}
```

### Validator1
validator 提供基于字段标签的、针对结构的各个字段的值验证，并且支持跨字段进行验证、跨结构进行验证，多维度深度验证、多种格式匹配等丰富功能，其安装路径为 `github.com/go-playground/validator/v10`，[官方文档](https://pkg.go.dev/github.com/go-playground/validator/v10)

使用常用标签对结构进行字段验证的实例如下：

``` go
import "github.com/go-playground/validator/v10"

type MyStruct struct {
    // 必须有值，即非零值
    Name       string     `validate:"required"`
    // 大于等于 0 且小于等于 130
    Age        uint8      `validate:"gte=0,lte=130"`
    // 必需有值，且符合 email 格式
    Email      string     `validate:"required,email"`
    // 等于任一指定列表中的值
    Gender     string     `validate:"oneof=male female prefer_not_to“`
    // 必须有值，且其中的元素也必须有值
    Addresses  []*Address `validate:"required,dive,required"`
}

// 对指定结构进行验证
err := validate.Struct(MyStruct{})
if err != nil {
    validationErrors := err.(validator.ValidationErrors)
    panic(validationErrors)
}
```

### Validator2
validator 提供基于字段标签的、针对结构的各个字段的值验证，特点在于格式简单的表达式，满足常用功能之余保证了代码可读性，也支持跨字段、跨结构、多维度深度验证等常用功能，其安装路径为 `github.com/bytedance/go-tagexpr/v2/validator`，[官方文档](https://github.com/bytedance/go-tagexpr/blob/master/validator/README.md)

使用常用标签对结构进行字段验证的实例如下：

``` go
import "github.com/bytedance/go-tagexpr/v2/validator"

type MyStruct struct {
      // 不等于 Alice 或 (Age 为 18 时匹配指定正则)
		Name         string   `vd:"($!='Alice'||(Age)$==18) && regexp('\\w')"`
		// 大于 0 且小于 100
		Age          int      `vd:"$>0&&$<100"`
		// 满足邮箱格式
		Email        string   `vd:"email($)"`
		// 满足手机格式
		Phone1       string   `vd:"phone($)"`
		// 其中元素满足手机格式
		OtherPhones  []string `vd:"range($, phone(#v,'CN'))"`
}

vd := validator.New("vd")
// checkAll 为 true 表示中途错误也检查所有字段
err := vd.Validate(MyStruct{}, checkAll)
if err != nil {
    panic(err)
}
```

## 工具
### UUID
uuid 包根据 RFC/DEC 标准，提供了五种不同版本的 UUID，以一个 128 Bits 的正整数表示，常用于作为唯一标识。其安装路径为 `github.com/google/uuid`，[官方文档](https://pkg.go.dev/github.com/google/uuid)

各个版本 UUID 的生成函数使用实例如下：

``` go
import "github.com/google/uuid"

// 生成 v1 版本 UUID
// 基于时间戳和 MAC 地址 (RFC-4122)
id, err := uuid.NewUUID()

// 生成 v2 版本 UUID
// 基于时间戳、MAC 地址和操作系统 GID/UID (DCE 1.1)
gid, err := uuid.NewDCEGroup()
gid, err := uuid.NewDCEPerson()

// 生成 v3 版本 UUID
// 基于命名空间和指定值进行 MD5 哈希 (RFC-4122)
id, err := uuid.NewMD5(space, data) UUID

// 生成 v4 版本 UUID
// 基于随机数或伪随机数生成 (RFC-4122)
id, err := NewRandom()

// 生成 v5 版本 UUID
// 基于命名空间和指定值进行 SHA-1 哈希 (RFC-4122)
id := func NewSHA1(space, data) UUID

// ----- 快捷使用方式 -----

// 封装生成函数为安全模式（进行了错误 Panic），直接返回 UUID
id := uuid.Must(uuid.NewUUID())

// 在安全模式下，直接返回 v4 版本 UUID
id := uuid.New()

// 在安全模式下，直接返回 v4 版本 UUID 的格式化字符串
// 格式为 xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
idStr := NewString()
```

UUID 的格式化和解析方式如下：

``` go
var id uuid.UUID

// 格式化 UUID 为字符串
// 格式为 xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
idStr := id.String()

// 解析字符串为 UUID
// 支持四种格式：
// xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
// urn:uuid:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
// {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}
// xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
id, err := uuid.Parse(idStr)

// 在安全模式下，直接返回字符串解析出来的 UUID
id := uuid.MustParse(idStr)

// 自定义格式化，输出格式为不带 - 的字符串
func HexString(uuid UUID) String {
    var buf [32]byte
    return string(hex.Encode(buf[:], uuid[:]))
}
```
