# 接口
```
1. 定义
2. 实现机制
3. 类型转换
```

- 接口(interface)定义

接口是一组方法的集合，我们通过接口来定义对象的一组行为。在某些动态语言里，接口也被称作为(protocol)。
互相交互的双方，共同遵守事先规定的协议，使得交互双方在无须知道对方身份的情况下进行相互调用，而接口自身要实现的是做什么，而不关心
怎么做和谁来做，比如：

```go
type geometry interface {
    area() float64
    perim() float64
}

type circle struct {
    radius float64
}

type rect struct {
    width, height float64
}

func (r rect) perim() float64 {
    return 2 * r.width + 2*r.height
}

func (r rect) area() float64 {
    return r.width * r.height
}

func (c circle) perim() float64 {
    return 2 * math.Pi * c.radius
}

func (c circle) area() float64 {
    return math.Pi * c.radius * c.radius
}

func messure(g geometry) {
    fmt.Println(g)
    fmt.Println(g.area())
    fmt.Println(g.perim())
}

func main() {
    var r = rect{3, 2}
    var c = circle{3}
    messure(r)
    messure(c)
}
```
Go语言接口的实现机制很简洁，只要对象类型方法集包含接口声明的全部方法，就被视为实现了该接口，当然一个对象
可以实现多个接口，一个接口也可以对应多个对象。声明一个接口时需要注意以下几点：

```
- 接口中不能有字段声明
- 不能定义自己的方法
- 只能声明不能实现方法
- 可以嵌入其它接口类型
```
思考下面的代码片段，能否正常运行,为什么？

```go
type geometry interface {
    area() float64
    perim() float64
}

type rect struct {
    width, height float64
}

func (r *rect) perim() float64 {
    return 2 * r.width + 2 * r.height
}

func (r rect) area() float64 {
    return r.width * r.height
}

func messure(g geometry) {
    fmt.Println(g)
    fmt.Println(g.area())
    fmt.Println(g.perim())
}

func main() {
    var r = rect{3, 2}
    messure(r)
}
```
- 空接口

如果接口没有任何方法声明，则称该接口为空接口(interface{})，类似于C/C++中的void*，但是包含了类型信息，它可以被赋值为任何类型的对象。接口的默认值为nil，
如果实现接口的类型支持关系运算符，那么接口也可以进行相等运算，比如：

```go
{
    var a, b interface{}
    println(a == nil, b == nil,a == b)
    a ,b = 100,100
    println(a == nil, b == nil,a == b)
    a ,b = map[int]int{} ,map[int]int{}
    println(a == nil, b == nil,a == b)
}
输出：
true true true
false false true
panic: runtime error: comparing uncomparable type map[int]int
```
- 接口嵌入

接口同样支持嵌入，相当于将其它接口声明的方法集导入到该接口，所以对象类型方法集中必须拥有包含嵌入接口方法在内的全部方法才算实现了该接口，比如：

```go
type perimer interface{
    perim() float64
}
type geometry interface {
    area() float64
    perimer
}
```
既然支持了接口的嵌入，那么上述的两个interface之间可否相互转换呢？

```go
func main() {
    var r = rect{3, 2}
/*
    var t geometry = r
    var s  perimer = t                                                                                                                                
    fmt.Println(s.perim())
    fmt.Println(s.area()) //s.area undefined (type perimer has no field or method area)
*/
    var s perimer = r
    var t geometry = s //cannot use s (type perimer) as type geometry in assignment
    fmt.Println(s.perim())
    fmt.Println(s.area())
}
```
实践证明，可以将超级接口转换为其包含的子集接口，但是反过来操作是不可以的。

- interface 机制

上例messure函数中的参数是一个interface接口， 也就是geometry
的一个对象实例，这个对象实例就是interface value， 它的数据结构如下：

```go
type iface struct {
	tab  *itab          //itab 的结构体存储运行期所需的相关类型信息
	data unsafe.Pointer //data 是对应的实现该接口的类型的实例指针
}
type itab struct {
	inter  *interfacetype //inter字段表示这个interface value所属的接口元信息
	\_type  *_type  //_type字段表示具体实现类型的元信息
	link   *itab
	bad    int32
	unused int32
	fun    [1]uintptr // fun字段表示该interface的方法数组
}
```



- 类型转换

Go编译器根据类型推断，可以将接口变量还原为原始类型，或用来判断是否实现了某个更具体接口类型。

```go
type geometry interface {
    area() float64
    perim() float64
    String()string
}

type rect struct {
    width, height float64
}

func (r *rect) perim() float64 {
    return 2\*r.width + 2*r.height
}

func (r rect) area() float64 {
    return r.width * r.height
}
func (r rect) String() string{
    return fmt.Sprintf("%f:%f",r.width,r.height)
}

func main() {
    var r = rect{3, 2}
    var i interface{}
    i = r
    if n ,ok := i.(fmt.Stringer);ok{  //i.(fmt.Stringer)接口转换为具体的接口类型
        fmt.Println(n.String())
    }
    if r2 ,ok := i.(rect);ok{   //i.(rect) 转换回原始数据类型
        fmt.Println(r2)
    }
    e := i.(error) //interface conversion: main.rect is not error: missing method Error
    fmt.Println(e)
}
```
也可以使用switch语句在多种接口类型间做出推断匹配，并写出相应的处理逻辑，比如：

```go
func main() {
    var r = rect{3, 2}
    var i interface{}
    i = r
    switch v := i.(type){
        case fmt.Stringer:
            fmt.Println("fmt.stringer")
        case nil:
            fmt.Println("nil")
        default:
            fmt.Println("unknown",v)
    }
}
```
注意： type switch语句中不支持fallthrough关键字