# 包管理

```
1. 工作空间
2. 导入包
3. 组织结构
4. 依赖管理
```

- 环境变量：

GOPATH：编译器等相关工具会按照 GOPATH 环境变量中设置的路径进行目标搜索，比如在
导入目标库时，排在 GOPAH 环境变量列表前面的路径比当前工作空间优先级高，go
get将下载的第三方包保存到GOPAH环境变量列表中第一个工作空间内。

GOROOT 不是必须要设置的。因为默认go会安装在/usr/local/go下，但也允许自定义安装位置，GOROOT 的目的就是告知go当前的安装位置，编译的时候从GOROOT去找SDK的system libariry，用于指定工具链以及标准库的存放位置


- 工作空间

GOPATH 工作空间(workspace)是一个由src,bin,pkg三个目录组成的目录，通常情况下需要把该路径加入到 GOPATH 环境变量中，
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
在工作空间里，bin和pkg目录主要是受go install/get的命令影响，它们会把编译结果(可执行文件/静态库)安装到这两个目录下，以实现增量编译。


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
相同目录下所有源码文件必须使用相同的包名称，并且包名与目录名并无关系，不要求保持一致因为导入时使用绝对路径，所以在搜索路径下，包必须有唯一路径，但无须是唯一的名字，比如：

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



测试 

vendor
依赖GOPATH来解决go import有个很严重的问题：如果项目依赖的包做了修改，或者干脆删掉了，会影响我的项目。因此在1.5版本以前，为了规避这个问题，通常会将当前使用的依赖包拷贝出来。

为了能让项目继续使用这些依赖包，有这么几个办法：

将依赖包拷贝到项目源码树中，然后修改import
将依赖包拷贝到项目源码树中，然后修改GOPATH
在某个文件中记录依赖包的版本，然后将GOPATH中的依赖包更新到对应的版本(因为依赖包实际是个git库，可以切换版本)
go作为一个现代化的语言，居然要用这么复杂不直观而又不标准的方法来管理依赖，难怪在早期会有很多人非常不看好go的前景。

为了解决这个问题，go在1.5版本引入了vendor属性(默认关闭，需要设置go环境变量GO15VENDOREXPERIMENT=1)，并在1.6版本中默认开启了vendor属性。

简单来说，vendor属性就是让go编译时，优先从项目源码树根目录下的vendor目录查找代码(可以理解为切了一次GOPATH)，如果vendor中有，则不再去GOPATH中去查找。

以kube-keepalived-vip为例。该项目会调用k8s.io/kubernetes的库(Client)，但如果你用1.5版本的kubernetes代码来编译keepalived，会编译不过：

./controller.go:107: undefined: "k8s.io/kubernetes/pkg/client/unversioned".Client
查下代码会发现1.5版本中代码有变化，已经没有这个Client了。这就是前面说的依赖GOPATH来解决go import所带来的问题，代码不对上了。

kube-keepalived-vip项目用vendor目录解决了这个问题：该项目把所有依赖的包都拷贝到了vendor目录下，对于需要编译该项目的人来说，只要把代码从github上clone到$GOPATH/src以后，就可以进去go build了(注意，必须将kube-keepalived-vip项目拷贝到$GOPATH/src目录中，否则go会无视vendor目录，仍然去$GOPATH/src中去找依赖包)。

但是vendor目录又带来了一些新的问题：

vendor目录中依赖包没有版本信息。这样依赖包脱离了版本管理，对于升级、问题追溯，会有点困难。
如何方便的得到本项目依赖了哪些包，并方便的将其拷贝到vendor目录下？ Manual is fxxk.
社区为了解决这些(工程)问题，在vendor基础上开发了多个管理工具，比较常用的有godep, govendor, glide。go官方也在开发官方dep，目前还是Alpha状态。

下面来看看使用的比较多的gode, glide和 govendor。

godep
godep的使用者众多，如docker，kubernetes， coreos等go项目很多都是使用godep来管理其依赖，当然原因可能是早期也没的工具可选。

godep早期版本并不依赖vendor，所以对go的版本要求很松，go 1.5之前的版本也可以用，只是行为上有所不同。在vendor推出以后，godep也改为使用vendor了。

godep使用很简单：当你的项目编写好了，使用GOPATH的依赖包测试ok了的时候，执行：

$ godep save
以hcache为例，执行go save，会做2件事：

扫描本项目的代码，将hcache项目依赖的包及该包的版本号(即git commit)记录到Godeps/Godeps.json文件中
将依赖的代码从GOPATH/src中copy到vendor目录(忽略原始代码的.git目录)。对于不支持vendor的早期版本，则会拷贝到Godeps/_workspace/里
一个Godeps.json的例子。

如果要增加新的依赖包：

Run go get foo/bar
Edit your code to import foo/bar.
Run godep save (or godep save ./…).
如果要更新依赖包：

Run go get -u foo/bar
Run godep update foo/bar. (You can use the … wildcard, for example godep update foo/…).
godep还支持godep restore，可以将vendor下的代码反向拷贝到$GOPATH下。不过我没想到这个功能在什么情况下可以用到。

glide
glide也是在vendor之后出来的。glide的依赖包信息在glide.yaml和glide.lock中，前者记录了所有依赖的包，后者记录了依赖包的版本信息(合成一个多好)。

glide使用也不麻烦：
glide create  # 创建glide工程，生成glide.yaml
glide install # 生成glide.lock，并拷贝依赖包
work, work, work
glide update  # 更新依赖包信息，更新glide.lock

glide install会根据glide.lock来更新包的信息，如果没有则会走一把glide update生成glide.lock

最终一个使用glide管理依赖的的工程会是这样：


glide的功能更丰富一些。

glide tree可以很直观的看到vendor中的依赖包(以后会被移除掉，感觉没啥用)
glide list可以列出vendor下所有包
glide支持的Version Control Systems更多，除了支持git，还支持 SVN, Mercurial (Hg), Bzr
最重要的，glide.yaml可以指定更多信息，例如依赖包的tag、repo、本package的os, arch。允许指定repo可以解决package名不变，但使用的是fork出来的的工程。
govendor
govendor是在vendor之后出来的，功能相对godep多一点，不过就核心问题的解决来说基本是一样的。govendor生成vendor目录的时候需要2条命令：

govendor init生成vendor/vendor.json，此时文件中只有本项目的信息
govendor add +external更新vendor/vendor.json，并拷贝GOPATH下的代码到vendor目录中。
govendor还可以直接指定依赖包版本来获取包，这也有了点版本管理的影子了。

