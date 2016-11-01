# 数据
1. 字符串
2. 数组
3. 切片
4. 字典
5. 结构

- 字符串
Go语言中的字符串是由一组不可变的字节(byte)序列组成，其本身是一个复合结构：

```go
string.go 文件中
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
字符串中的每个字节都是以UTF-8编码存储的Unicode字符，字面量里可以使用十六进制、八进制和UTF编码格式，字符串的头部指针指向字节数组的开始，但是没有NULL或'\0'结尾标志。
表示方式很简单，用双引号("")或者反引号(``)，它们的区别就是双引号之间的转义符会被转义，而反引号之间的字符保持不变;并且反引号支持跨行，而双引号则不可以。比如：

```go
{
    println("hello\ngo")    //输出hello	go
    println(`hello\ngo`)    //输出hello\tgo
}

{
    println( "hello 
        go" )                   //syntax error: unexpected semicolon or newline, expecting comma or )

    println(`hello                                                                                                                                                                                                 
        go`)
}
输出：
hello 
	go
```

在类型的章节中讲过字符串的默认值是""，而不是nil,比如：

```go
var s string
println( s == "" )  //true
println( s == nil ) //invalid operation: s == nil (mismatched types string and nil)
```
Go字符串支持"+ , += , == , != , < , >" 六种运算符

Go字符串允许以索引号访问字节数组(非字符)，但不能获取元素的地址：比如

```go
{
    var a = "hello"
    println(a[0])       //输出 104
    println(&a[1])      //cannot take the address of a[1]
}
```
Go字符串允许以切片的语法返回子串(起始和结束索引号)

```go
    var a = "0123456"                                                                                                                                                                                              
    println(a[:3])      //0,1,2
    println(a[1:3])     //1,2
    println(a[3:])      //3,4,5,6
```

如何遍历字符串？

```
{
    var a = "Go语言"
    for i:=0;i < len(a);i++{                //以byte方式按字节遍历
        fmt.Printf("%d: [%c]\n", i, a[i])
    }
    for i, v := range a{                    //以rune方式遍历                                                                                               
        fmt.Printf("%d: [%c]\n", i, v)
    }
}
输出：    
0: [G]
1: [o]
2: [è]
3: [¯]
4: [­]
5: [è]
6: [¨]
7: []
0: [G]
1: [o]
2: [语]
5: [言]
```
在Go语言中，字符串的底层使用byte数组存的，并且是不可以改变的，所以byte方式每次得到的只有一个byte，而中文字符是占3个byte的。
rune采用计算字符串长度的方式与byte方式也不同，比如：

```go
println(utf8.RuneCountInString(a)) // 结果为  4
println(len(a)) // 结果为 8
```
如果想要获得我们想要的那种情况的话，需要先将a字符串转换为rune切片，再使用内置的len函数，比如:

```go
{
    r := []rune(a)
    for i:= 0;i < len(r);i++{
        fmt.Printf("%d: [%c]\n", i, r[i])
    }
}
```
所以，在遍历或处理的字符串中存在中文的情况下，尽量使用rune方式处理。

更多关于字符串处理的函数，请参见[strings](https://golang.org/pkg/strings/)

