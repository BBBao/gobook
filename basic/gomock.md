#### gomock

https://github.com/golang/mock

在开发过程中往往需要配合单元测试，但是很多时候，单元测试需要依赖一些比较复杂的准备工作，比如需要依赖数据库环境，需要依赖网络环境，单元测试就变成了一件非常麻烦的事情。举例来说，比如我们需要请求一个网页，并将请求回来的数据进行处理。在刚开始的时候，我通常都会先启动一个简单的http服务，然后再运行我的单元测试。可是这个单元测试测起来似乎非常笨重。甚至在持续集成过程中，我还为了能够自动化测试，特意写了一个脚本自动启动相应的服务。事情似乎需要进行一些改变。

mock对象就是为了解决上面的问题而诞生的，mock(模拟)对象能够模拟实际依赖对象的功能，同时又不需要非常复杂的准备工作，你需要做的，仅仅就是定义对象接口，然后实现它，再交给测试对象去使用。

go-mock是专门为go语言开发的mock库，该库使用方式简单，支持自动生成代码，可以说是不可多得的好工具。下面我就简单地展示一下go-mock是如何工作的:
第一个是代码依赖，第二个是命令行工具
```bash
go get github.com/golang/mock/gomock
go get github.com/golang/mock/mockgen
```
我在$GOPATH/src目录下新建一个项目：hellomock，在$GOPATH/src/hellomock目录下新建hellomock.go，并定义一个接口Talker:
```go
package hellomock

type Talker interface {
    SayHello(word string)(response string)
}
```

然后我们需要一个实现了Talker功能的结构体，假设我们有这样的场景，我们现在有一个迎宾的岗位，需要一个能够迎宾的迎宾员，当然这个迎宾员可以是一个人，或者是一只鹦鹉。那么我们需要做的是，定义一个Persion结构体（或者是鹦鹉或者是别的什么东西），并实现Talker接口：
```go
package hellomock
import "fmt"
type Person struct{
  name string
}
func NewPerson(name string)*Person{
  return &Person{
      name:name,
  }
}
func (p *Person)SayHello(name string)(word string) {
  return fmt.Sprintf("Hello %s, welcome come to our company, my name is %s",name,p.name)    
}
```
现在我们的Person已经实现了Talker接口，现在我们让他发挥作用吧！
现在假设，我们有一个公司，公司有一个迎宾员，也就是我们的前台妹子，这个妹子实现了Talker接口.她能够自动向来的客人SayHello:
company.go
```go
package hellomock
type Company struct {
  Usher Talker
}
func NewCompany(t Talker) *Company{
  return &Company{
    Usher:t,
  }
}
func ( c *Company) Meeting(gusetName string)string{
  return c.Usher.SayHello(gusetName)
}
```
我们的场景已经设计好了，那么我们传统的话，会如何实现单元测试呢？
company_test.go
```go
package hellomock

import "testing"

func TestCompany_Meeting(t *testing.T) {
    person := NewPerson("王尼美")
    company := NewCompany(person)
    t.Log(company.Meeting("王尼玛"))
}
```
```bash
go test -v hellomock -run ^TestCompany_Meeting
```
现在我们构造一个王尼美还是很简单的，但是我们现在要用mock对象进行模拟,这时mockgen就登场了：
```bash
mkdir mock_hellomock
mockgen -source=hellomock.go > mock_hellomock/mock_Talker.go
```
这个时候，将会生成mock/mock_Talker.go文件：
需要注意的是，自动生成的文件同源文件在不同的包下，需要新建一个目录存放
我们并不需要在意生成文件的内容，我们只需要知道如何去使用即可

接下来看看如何去使用这个mock对象，新建一个单元测试：
```go
func TestCompany_Meeting2(t *testing.T) {
    ctl := gomock.NewController(t)
    mock_talker := mock_hellomock.NewMockTalker(ctl)
    mock_talker.EXPECT().SayHello(gomock.Eq("王尼玛")).Return("这是自定义的返回值，可以是任意类型。")

    company := NewCompany(mock_talker)
    t.Log(company.Meeting("王尼玛"))
    //t.Log(company.Meeting("张全蛋"))
}
```
```bash
go test -v hellomock -run ^TestCompany_Meeting2
```

需要说明的一点是，mock对象的SayHello可以接受的参数有gomock.Eq(x interface{})和gomock.Any()，前一个要求测试用例入餐严格符合x，第二个允许传入任意参数。比如我们在注释掉的测试中传入了"张全蛋"，结果报错，测试失败：