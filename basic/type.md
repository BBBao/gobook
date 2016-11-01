#### 类型
1. 变量
2. 命名
3. 常量
4. 基本类型
5. 引用类型
6. 类型转换
7. 自定义类型


#### 变量

Go语言有两种方式定义变量：

```text
var 关键字
:= 短变量声明符
```

- var关键字

```go
var x int   //自动初始化为0
var y = false   //自动推断为bool类型
```
和C语言不同，类型被放在变量名之后,并且在运行时，为了避免出现不可预测行为，内存分配器会初始化变量为二进制零值。
如果显示初始化变量的值，可以省略变量类型，由编译器推断。

Go语言一次可以定义多个变量，并可以对其初始化成不同的类型，比如

```go
var x,y int             //相同类型多个变量
var a, s = 10010, "hello go"    //不同类型多个变量
```
按照Go语言的编码规范，建议以组的方式整理多行变量定义，比如:
```go
var (
    x,y int
    a,s = 10010, "hello go"
)
```
- 短变量声明符

在介绍词法的章节中已经看到过这种用法，比如:

```go
    x := 10010
    a , s := 10010, "hello go"
```
这种方式很简单，但日常开发中新手经常犯下的一个错误，比如:

```go
var x = 10010
func main(){
    println(&x, x)
    ...
    x := "hello go" //对全局变量重新定义并初始化
    println(&x, x)
}
```
可想而知最终酿成的后果;为了正确使用这种简单的短变量声明方式，请遵循如下限制：

```text
  1. 定义变量的同时，显示初始化
  2. 不能提供数据类型
  3. 只能在函数体内部使用
  4. 函数体内不能使用短变量重复声明
  5. 不能使用短变量声明这种方式来设置字段值
  6. 不能使用nil初始化一个未指定类型的变量
```

思考下面的代码片段，输出结果是什么？
```go
func test(){
    x := 1
    fmt.Println(x)
    {   
        fmt.Println(x)
        x = 3 
        x := 2
        fmt.Println(x)
        x = 5
    }
    fmt.Println(x)
}
```
如何检查你的代码中是否存在这样的声明？

```sh
# go tool vet -shadow yourfile.go
```
Go编译器会将未使用的局部变量当作错误，比如:

```go
func main() {
    x := 1
    var a int       //a declared and not used
    fmt.Println(x)
} 
```
那么思考一下下面的代码，会不会报错？

```go
func main() {
    x := 1
    const a = 10010   // ???                                                                                                                                                                                             
    fmt.Println(x)
} 
```
所以要记住函数体中存在未使用的常量，不会引发编译器报错;
#### 常量
常量表示运行时恒定不变的值，由const关键字声明，常量值必须是在编译期可确定的字符、字符串、数字或布尔值。
声明常量的同时可以指定其常量类型，或者由Go编译器通过初始化值推断，如果显示指定其常量类型，那么就必须保证常量左右值类型一致。
必要时要做类型转换。并且右值不能超出常量类型的取值范围，否则会造成溢出错误。比如：

```go
const(
    x , y = 100, -10010
    b byte = byte(x)    //类型不一致需要做类型转换
    c uint8 = uint8(y) //溢出
)
```
思考：修改一下上面的代码，只进行声明而没有对其初始化和指定类型，结果会怎样？

```go
const(
    x = 100
    y
    s string = "hello go"                                                                                                                                                                                      
    c
)
fmt.Printf("%T , %v\n",y,y)
fmt.Printf("%T , %v\n",c,c)
```
结论证明： 在常量组中如果不指定常量的类型和初始化值，则与上一行非空常量的右值相同

常量值可以是Go编译器能计算出结果的表达式，如unsafe.Sizeof, len,cap,iota等。

#### 枚举
Go语言没有明确上的定义枚举enum类型，不过可以通过iota关键字来实现一组递增常量的定义，比如:
```go
const (
    _ = iota
    KB = 1 << (10 * iota)   // 1<<(10 * 1)
    MB                      // 1<<(10 * 2)
    GB                      // 1<<(10 * 3)
)
```
多常量定义中也可以使用iota,它们各自单独计算，确保常量组中每行常量的列数量相等即可，比如：

```go
const(
    a ,b = iota,iota *10    //0 , 0 * 10
    c ,d                    //1 ， 1 * 10
)
```
如果中断iota自增,需要显示恢复，且后续的常量值将按行序递增，比如:
```go
const(
    a = iota
    b           //b = ???
    c = 100
    d           //d = ???
    e = iota    //c = ???
    f           //f = ???
)
```
自增常量可以指定常量类型，甚至可以自定义类型，比如：

```go
const(
    f float = iota //指定常量类型
)
type color byte
const(
    black color = iota  //自定义类型
    red
    blue
)
```

- 常量VS变量

```text
常量是“只读”属性
变量在运行期分配内存，而常量是在编译期间直接展开，作为指令数据使用
数字常量不会分配存储空间，不能像变量那样可以进行内存寻址
```
比如：

```go
const x = 10 
const y byte = x    // 相当于const y byte = 100
const a int = 10    //显示指定常量类型，编译器做强类型检查
const b byte = a    // cannot use a (type int) as type byte in const initializer
```

#### 基本类型

| 类型 | 长度 | 默认值 | 说明 |
| :---: |:---:|:---:| :---:|
| bool | 1 | false | |
| byte | 1 | 0 | alias for uint8|
| rune | 4 | 0 | alias for int32|
|int, uint| 4 或 8| 0 | 32 或 64 位|
|int8,uint8| 1 | 0 | -128 ~ 127, 0 ~ 255 |
|int16, uint16| 2 | 0 |-32768 ~ 32767, 0 ~ 65535|
|int32, uint32| 4 | 0 |-21亿 ~ 21 亿, 0 ~ 42 亿|
|int64, uint64| 8 | 0 |
|float32| 4 | 0.0 | |
|float64 |8 |0.0 | 默认浮点数类型|
|complex64 | 8 | | |
|complex128 |16| | |
|uintptr| 4 或 8 | 0 |足以存储指针的uint|
|string|| "" | UTF-8 字符串|
|struct || |结构体|
|interface| |nil |接⼝|
|function ||nil |函数|
|slice || nil|切片:引用类型|
|map ||nil|字典:引⽤用类型 |
|channel|| nil|通道:引⽤用类型|
|array|||数组类型就是值类型，声明方式 [n]T，声明n个大小的T类型数组|

#### 引用类型
特指slice,map和channel这三种预定义类型，他们具有比数字、数组等更复杂的存储结构;
引用类型的创建必须是用make函数来创建，因为除了分配内存之外，还有一系列属性需要初始化，比如指针, 长度，甚至包括哈希分布,数据队列等。
而内置函数new只是按照指定类型长度分配零值内存，返回指针。比如：

```go
p := new(map[string]int) //new函数返回指针
m := *p
m["go"] = 1     //panic: assignment to entry in nil map(运行时)
```

#### 自定义类型
使用关键字type可以定义用户自定义类型，包括基于基本数据类型创建，或者结构体和函数等类型，比如：

```go
type flags byte
const (
    exec flags = 1 << iota
    write 
    read
)
```
与var、const类似，多个type定义也可以合并成组，比如：

```go
    type (
        user struct{
            name string
            age int
        }

        f func(user)
    )                       //自定义一组类型

    u := user{"Tom",20}
    var say f = func(s user){
        fmt.Println(s.name, s.age)
    }
    say(u)
```
自定义类型与基础类型相比，它们具有相同的数据结构，但它们不存在任何关系，除操作符外，自定义类型不会继承基础类型的其它信息，它们属于完全不同的两种类型，
即它们之间不能做隐式转换，不能视为别名甚至不能用于比较表达式。比如:

```go
    type(
        data int
    )
    var d data = 10
    var x int = d       //cannot use d (type data) as type int in assignment
    println(x == d)     //invalid operation: x == d (mismatched types int and data)
```
#### 未命名类型
与有明确标志符的bool, int, string等类型相比，数组、切片、字典、通道等类型与具体元素类型或长度等属性有关，所以称作
未命名类型(unnamed type)。当然可以用type为其提供具体名称，将其改变为命名类型(named type)。当然可以用type为其提供具体名称，将其改变为命名类型

具有相同声明的未命名类型视作同一类型，比如：

```text
1. 具有相同基类型的指针
2. 具有相同元素类型和长度的数组(array)
3. 具有相同元素类型的切片(slice)
4. 具有相同键值类型的字典(map)
5. 具有相同数据类型及操作方向的通道(channel)
6. 具有相同字段序列(字段名、字段类型、标签以及字段顺序)的结构体(struct)
7. 具有相同签名(参数类型，参数顺序和返回值列表，不包括参数名)的函数(func)
8. 具有相同方法集(方法名、方法签名、不包括顺序)的接口(interface)
```
结构体的tag经常被初学者忽视，它也属于结构体类型组成的一部分，而不仅仅是元数据描述。

#### 类型转换
类型转换在日常开发中是不可避免的问题，Go语言属于强类型语言，除常量、别名以及未命名类型之外，其它不同类型之间的转换都需要显示类型转换;好处是我们至少能知道语句及表达式的确切含义，比如：

```go
a := 10         //int
b := byte(a)    //byte
c := a + int(b) //确保类型一致性
d := b + byte(a)//确保类型一致性
```
如果类型转换的目标是指针、单向通道、没有返回值的函数类型，那么为了避免语法歧义带来的类型转换错误，必须使用括号，比如

```go
x := 100
p := *int(&x) //cannot convert &x (type *int) to type int
                //invalid indirect of int(&x) (type int)
pc := <-chan int(c) //invalid operation: pc <- 3 (send to non-chan type int)

```
正确做法如下:

```go
p := (\*int)(&x) // (\*int)(&x)
(<-chan int)(c) //单向通道
(func()) (f)     //无返回值的函数
(func()int) (f) //有返回值的函数，但是也可以不用括号:func()int(f),但是为了影响程序可读性，建议加上括号
```

完善上面的代码片段，继续验证和扩展你的想法。

下一节课我们会继续讲解Go语言表达式，包括：关键字、运算符、初始化和流程控制。