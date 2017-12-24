# 方法

```
1. 定义
2. 匿名字段
3. 方法集
4. 表达式
```

- 定义
方法是与对象实例绑定的特殊函数，用于维护和展示对象的自身状态。 与函数的区别是方法有前置实例接收参数(receiver)，编译器根据
receiver来判断该方法属于哪个实例。receiver可以是基础类型，也可以是指针类型，这会关系到是否需要有
可以修改对象实例的能力。在调用方法时，可以使用对象实例值或指针，编译器会根据receiver类型自动在基础类型
和指针类型之间转换，比如：

```go
type rect struct {
    width, height, area int
}

func (r *rect) pointer() {
    r.width += 2
    r.area = r.width * r.height
}
func (r rect) value() {
    r.width += 4
    r.area = r.width * r.height
}

func main() {
    r := rect{width: 10, height: 5}
    r.value()
    fmt.Println(r)
    r.pointer()
    fmt.Println(r)
/*
    r := &rect{width: 10, height: 5}
    r.value()
    fmt.Println(r)
    r.pointer()
    fmt.Println(r)
*/                                                                                                                                                    
}
输出：
{10 5 0}
{12 5 60}
```
如果使用指针调用方法时，需要注意不能使用多级指针，且必须使用合法的指针(包括nil)或能取实例地址，比如：

```go
type X struct{}                                                                                                                                       

func (x *X)test(){
    fmt.Println("hello gopher")
}

func main(){
    var x *X
    fmt.Println(x)
    x.test()
    &X{}.test() //cannot take the address of X literal
}
```

如何选择方法的receiver类型 ？

```
- 如果要修改实例状态，用*T
- 如果不需要修改实例状态的小对象或固定值，建议用T
- 如果是大对象，建议用*T，可以减少复制成本
- 引用类型、字符串、函数等指针包装对象，用T
- 如果对象实例中包含Mutex等同步字段，用*T, 以免因复制造成锁无效
- 无法确定需求的情况，都用*T
```

- 匿名字段
可以像访问匿名字段成员那样调用其方法，由编译器负责查找，比如：

```go
type person  struct{}

type Man struct{
    person
}
func (p person)toWork()string{
    return "Tom"
}
func main(){
    var m Man 
    fmt.Println(m.toWork())                                                                                                              
}
输出：
Tom
```
如果Man结构体也有个同名的toWork方法，此时调用逻辑如下，比如:

```go
type person  struct{}

type Man struct{
    person
}

func (p person)toWork()string{
    return "Tom to work"
}

func (m Man)toWork()string {
    return "me to work"
}
func main(){
    var m Man
    fmt.Println(m.toWork())         //me to work
    fmt.Println(m.person.toWork())  //Tom to work                                                                                                      
}
```
- 方法集
Go规范中提到了一个与类型相关的方法集(method set)，这决定了它是否实现了某个接口。

```
- 类型T方法集包含所有receiver T方法
- 类型*T方法集包含所有receiver T + *T的方法
- 匿名嵌入S，T方法集包含所有receiver S方法
- 匿名嵌入*S,T方法集包含所有receiver S + receiver *S方法
- 匿名嵌入\*S或匿名嵌入S，*T方法集包含所有receiver S + receiver *S方法
```

```go
type S struct{}

type T struct{
    S   
}

func (S) SVal()  {}  
func (*S) SPtr() {}
func (T) TVal()  {}  
func (*T) TPtr() {}

func methodSet(a interface{}) {
    t := reflect.TypeOf(a)
    for i, n := 0, t.NumMethod(); i < n; i++ {
        m := t.Method(i)
        fmt.Println(m.Name, m.Type)
    }   
}
                                                                                                                                                      
func main() {
    var t T 
    methodSet(t)
    println("--------------")
    methodSet(&t)
}
输出：
SVal func(main.T)
TVal func(main.T)
--------------
SPtr func(*main.T)
SVal func(*main.T)
TPtr func(*main.T)
TVal func(*main.T)
```
很显然， 匿名字段就是为扩展方法集准备的。否则没有必要少写个字段而大费周章。这种组合没有父子依赖关系，
整体与局部松耦合，可以任意增加来实现扩展。各单元互无关联，实现与维护更加简单。

- 方法表达式
方法是一种特殊的函数，除了可以直接调用之外，还可以进行赋值或当作参数传递，下面是Go语言的方法定义格式，比如：

```go
func (p mytype) funcname(q type) (r,s type) { return 0,0}
```
本质上这就是一种语法糖，方法调用如下：

```go
instance.method(args) -> (type).func(instance, args)
```
instance 就是Reciever，左边的称为Method Value;右边则是Method Expression，Go推荐使用左边形式。
Method Value是包装后的状态对象，总是与特定的对象实例关联在一起(类似闭包)，
而Method Expression会被还原成普通的函数，将Receiver作为第一个显式参数，调用时需额外传递。
二者本质上没有区别，只是Method Value 看起来更像面向对象的格式,且编译器会自动进行类型转换；
Method Expression更直观，更底层，编译器不会进行类型转换，会按照实际表达意义去执行，更易于理解。

- Method Expression

```
type Person struct {
    Age int 
    Name string
}

func (p Person) GetAge() int{
    return  p.Age
}

func (p *Person) SetAge(i int){
    p.Age =  i
}

func main(){
    p := Person{20, "Tom"}
    setAge := (*Person).SetAge   //(\*Person)必须用括号，整体相当于func(p *Person)                                                                                                
    setAge(&p,50)                  //编译器不会进行类型转换
    getAge := Person.GetAge
    fmt.Println(getAge(p))
}
输出：
50
```
- Method Value

```go
func main(){
    p := Person{20, "Tom"}
    setAge := p.SetAge      //编译器会自动进行类型转换
    setAge(50)                                                                                                                                        
    getAge := p.GetAge
    fmt.Println(getAge())
}
```
只要receiver参数类型正确，使用nil同样可以执行，比如：

```go
type N int 

func (n N) value(){
    println(n)
}
func (n *N) pointer(){
    println(n)
}

func main(){
    var n *N
    n.pointer()
    (*N)(nil).pointer()
    (*N).pointer(nil)                                                                                                                                 
}
```
这样写程序并没有什么意义，只是希望你能理解并安全使用Method Value和Method Expression

