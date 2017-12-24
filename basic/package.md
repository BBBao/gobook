# 包管理

```
1. 工作空间
2. 导入包
3. 组织结构
4. 依赖管理
```

- 工作空间

工作空间(workspace)是一个由src,bin,pkg三个目录组成的目录，通常情况下需要把该路径加入到GOPATH环境变量中，
以便相关工具能够正常工作。

```
- src：工程源代码存放路径
- bin：可执行文件的安装路径
- pkg：包安装路径，按照操作系统和平台进行隔离
```
比如：

```
workspace/
├── bin
│   └── server
├── pkg
│   └── linux_amd64
│       └── service.a
└── src
    ├── server
    │   ├── main.go
    └── service
        └── utils.go
```
在工作空间里，bin和pkg目录主要是受go install/get的命令影响，它们会把
编译结果(可执行文件/静态库)安装到这两个目录下，以实现增量编译。

- 环境变量：

GOPATH：编译器等相关工具会按照GOPATH环境变量中设置的路径进行目标搜索，比如在
导入目标库时，排在GOPAH环境变量列表前面的路径比当前工作空间优先级高，go
get将下载的第三方包保存到GOPAH环境变量列表中第一个工作空间内。
GOROOT：用于指定工具链以及标准库的存放位置。 

- 导入包
在使用标注库和第三方包之前，首先需要使用import导入，参数是以src目录为
始的绝对路径。编译器会先从标准库路径开始搜索，然后对GOPATH环境变量列表中的
各个路径进行搜索，比如：

```go
import "path/filepath"  //实际路径为/root/data/go/src/path/filepath/
```
为了避免引入的包名称冲突，可以使用包别名进行区分，比如：

```go
import(
	"net/http"
	"text/template"
	"html/template" //template redeclared as imported package name
)
import(
	"net/http"
	"text/template"
	htemplate "html/template" //alias
)

```
总结四种包导入方式：

```
import "github.com/sanrenxing/service"  //默认方式
import S "github.com/sanrenxing/service" //别名方式
import . "github.com/sanrenxing/service" //简便方式
import _ "github.com/sanrenxing/service" //初始化方式，不引用，只用于初始化目标包
```
- 相对路径：

在开发过程中，也可以使用相对路径进行包的引用，这并不影响进行项目的编译、运行和测试，
如果项目不在工作空间里的话，会影响执行go install安装。
反而如果在设置了GOPATH的工作空间中，相对路径将会导致编译失败。

- 组织结构

包(package) 是组织结构中最小单位，它是由一个或多个保存在同一个目录下的(不含子目录)的
源码文件组成。包的用法类似名字空间(namespace)，它也是成员做用域和访问权限的边界。
比如：

```go
#src/mycluster
package cluster
type ClusterInfo struct{}

func NewCluster() *ClusterInfo{}
```
相同目录下所有源码文件必须使用相同的包名称，并且包名与目录名并无关系，不要求保持一致因为导入时使用绝对路径，所以在搜索路径下，包
必须有唯一路径，但无须是唯一的名字，比如：

```
# go list net/...
net
net/http
net/http/cgi
net/http/cookiejar
net/http/fcgi
net/http/httptest
net/http/httptrace
net/http/httputil
net/http/internal
net/http/pprof
net/internal/socktest
```

- 权限

```
所有成员在相同包内都可以相互访问，无论是否在同一个源码文件中。
首写字母为大写的全局变量、常量、类型、结构体字段、函数和方法等，可以在包外访问。
```
不过可以使用指针绕开限制，比如：

```go
/src/meta/data.go 
package meta
type data struct{
	x int
	Y int
}
func New()*data{
	return new(data)
}
---------------------------------------------
func main(){
    m := meta.New()
    m.Y = 100
    p := (*struct{x int})(unsafe.Pointer(m))
    p.x = 200
    fmt.Printf("%+v\n",*m)
}
```
- 初始化(init)

源码包内每个源码文件都可以定义一个或多个init初始化函数，所有这些初始化函数以及标准库和导入第三方包都由编译器
自动生成的一个包装函数进行调用，因此可保证在单一线程上执行，且只执行一次，
但是编译器不能保证这些初始化函数的执行顺序。所以初始化函数中不应该有逻辑关联，最好只处理
当前文件的初始化操作。

编译器执行次序：首先确保所有全局变量初始化，然后执行初始化函数，最后进入main.main入口函数。

讨论如下代码执行情况：

```go
//main.go
func init(){
    println("main")
}

func main(){
    init()
}
```

- 内部包(internal)

内部包机制相当于增加了新的访问控制权限，比如，所有保存在internal目录下的包，只能
被其父目录下的包访问。比如：

```go
service/
├── internal
│   ├── a
│   │   └── a.go
│   └── b
│       └── b.go
└── utils.go
----------------------------------------------
/service/utils.go
package service
import "service/internal/a"
import "service/internal/b"

func SayHello(){
	println("i am service")
	a.A()
	b.B()
}
-----------------------------------------------
/server/main.go
package main
import "service"
import "service/internal/a" 

func main(){
	service.SayHello()
	a.A()
}
编译时输出：
imports service/internal/a: use of internal package not allowed
```

- 依赖管理

在项目中常常会引入第三方依赖包，而第三方包都放到一个独立的工作空间中，那么就有可能导致
版本冲突问题，但如果都放到项目目录中，久而久之，又可能会把项目目录搞得很乱，为此，Go
引入名为vendor的机制，专门存放和管理第三方包，实现将源码与依赖整体打包分发。

```go
# cat server/vendor/github.com/sanrenxing/test/test.go
package test
func Test(){
	println("test")
}
----------------------------------
# cat server/main.go
package main
import "service"
import "github.com/sanrenxing/test"

func main(){
	service.SayHello()
	test.Test()
}
```
在main.go中引入"github.com/sanrenxing/test"时，优先搜索vendor目录。

如果引入的第三方包中也带有vendor目录，那么编译器该如何搜索包呢？

```
server/
├── main.go
├── server
└── vendor
    ├── github.com
    │   └── sanrenxing
    │       └── test
    │           ├── test.go
    │           └── vendor
    │               └── github.com
    ├── p
    │   └── p.go
    └── q
        ├── test.go
        └── vendor
            └── p
                └── p.go
```
```go
vendor/p/p.go 
package p
func PP(){
	println("pp")
}
------------------------
vendor/q/vendor/p/p.go 
package p
func PP(){
	println("PPP")
}
------------------------
main.go
package main
import "q"
import "p"

func main(){
	p.PP()
	q.QQ()
}
```

在main.go和test.go中分别import "p", 则程序运行结果会怎样？



