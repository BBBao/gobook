# 流程控制

Go语言的流程控制主要分为三大类：

```text
1. 条件判断
2. 循环控制
3. 无条件跳转
```

#### 条件判断
Go语言的条件判断由if ... else if ... else 语句实现，条件表达式值必须是布尔类型，可省略圆括号，但是花括号不能省略且左花括号不能另起一行，比如：

```go
    if 7%2 == 0 {
        fmt.Println("7 is even")
    } else {
        fmt.Println("7 is odd")
    }

    if 8%4 == 0 {   //可以没有else只有if语句
        fmt.Println("8 is divisible by 4")
    }
```
Go语言比较特别的是在条件判读语句中支持初始化语句,允许定义局部变量，但是这个变量的作用域仅限于该条件逻辑块内，比如：

```go
    if num := 9; num < 0 {
        fmt.Println(num, "is negative")
    } else if num < 10 {
        fmt.Println(num, "has 1 digit")
    } else {
        fmt.Println(num, "has multiple digits")
    }
```
最典型的应用就是字典的查找：

```go
    if v, ok := map1[key1];ok{
        ...
    }
```
优化建议： 对于那些过于复杂的组合条件判断，建议独立出来封装成函数，将流程控制和实现细节分离，使你的代码更加模块化，提高可读性。
有些时候你需要写很多的if-else来实现一些逻辑处理，这个时候代码看上去就很丑很冗长，而且也不易于以后的维护，这个时候switch就能很好的解决这个问题。
- switch
switch与if类似，也用于选择执行，执行的过程从上至下，直到找到匹配项，基本用法如下：

```go
switch sExpr {
    case expr1:
        some instructions
    case expr2:
        some other instructions
    case expr3:
        some other instructions
    default:
        other code
}
```

switch的表达式可以不是常量或者字符串，也可以沒有表达式，没有表达式则匹配true, 如同if-else if-else,和
其它语言的switch用法基本一样，不同的是在每个case中隐藏了break,不过你也可以显示加入break。
当匹配case成功后，处理当前case逻辑，处理完成后跳出整个switch，但这不是绝对的，可以使用fallthrough强迫程序执行后面的case， 
反之没有匹配到任何case，则执行default逻辑，比如:

```go
{
    i := 2
    switch i {              //将i 与case条件匹配
    case 1:
        fmt.Println("one")
        //break             //没有实际意义
    case 2:
        fmt.Println("two")  //直接跳出switch
    case 3:
        fmt.Println("three")
    }
}                           //基本的switch

{
    switch time.Now().Weekday() {
    case time.Saturday, time.Sunday:
        fmt.Println("it's the weekend")
    default:
        fmt.Println("it's a weekday")
    }
}                           //在同一个case中可以写多个表达式，它们之间是逻辑或的关系

{
    t := time.Now()
    switch {            //没有表达式，相当于if-else
    case t.Hour() < 12:
        fmt.Println("it's before noon")
    default:
        fmt.Println("it's after noon")
    }
}
{
    i := 2
    switch i {
    case 1:
        fmt.Println("one")
    case 2:
        fmt.Println("two")                                                                                                                                                                                         
        fallthrough             //使用fallthrough强迫程序执行后面的case
    case 3:
        fmt.Println("three")
    }
}
```

#### 循环控制
for是Go语言中仅有的一种循环语句，下面是常见的三种用法：

```go
    for j := 7; j <= 9; j++ { //典型的用法 initial/condition/after
        fmt.Println(j)
    }

    i := 1
    for i <= 3 {            //类似while(i <= 3){} 或for ; i<=3 ;;{}
        fmt.Println(i)
        i = i + 1
    }

    for {                   //类似while(true){} 或for true {}
        fmt.Println("loop")
        break
    }
```
初始化语句只会被执行一次，并且初始化表达式支持函数调用或定义局部变量;

for循环在日常开发中经常与range搭配，被用来遍历字符串、数组、切片、字典和通道，返回索引和键值数据，比如:

```go
{
    for i, c := range "go" {            //range string
        fmt.Println(i, c)
    }

    nums := []int{2, 3, 4}
    for i, num := range nums {          //range array/slice
        fmt.Println(i, ":", num)
    }

    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {         //range map
        fmt.Printf("%s -> %s\n", k, v)
    }
    
    chs := make(chan int,10)    //range channel
    chs <- 1
    chs <- 2
    close(chs)                                                                                                                                                                                                     
    for v := range chs {
        fmt.Println(v)
    }
}
```
总结如下：

|data type|first value|second value|
| :---: |:---:|:---:|
|string|index|s[index]|
|array/slice|index|v[index]|
|map|key|value|
|channel|element||

#### 无条件跳转
Go语言有三种跳转类型，分别是 continue, break, goto

```text
continue 仅用于for循环内部 ，终止本次循环，立即进入下一轮循环
break 用于for循环, switch,select语句，终止整个语句块的执行
goto 定点跳转，但不能跳转到其他函数或内层代码块内
```
看下面的例子：

```go
func over(){
    over:   //label over defined and not used
    println("call over")
}

func main(){
    rand.Seed(time.Now().UnixNano())

    for {
        loop:
        x := rand.Intn(100)     //获取一个100以内的随机数
        if x % 2 == 0{
            continue        //终止本次循环，立即进入下一轮循环
        }
        if x > 50{
            break           //终止整个语句块的执行
        }
        println(x)
    }
    goto over       //label over not defined
    goto loop       //goto loop jumps into block starting
}
```
goto 的习惯用法

```go
{
    f, err := os.Open("/tmp/dat")
    if err != nil{
        ....
        goto exit
    }
    ...
    n1, err := f.Read(b1)
    if err != nil{
        goto end
    }
    ...
                                                                                                                                                                                                                   
end:
    println("clear resource and exit")
    f.close()
exit:
    return
}
```

完善上面的代码，验证和扩展所有功能

下一节 函数
