# 反射
```
1. 类型
2. 值
3. 方法
4. 性能
5. 应用场景
```

### 类型
Go语言反射(reflect)是指在程序运行时，实现能够获取对象类型信息和内存结构的方法。
和C语言数据结构一样，Go对象头部也没有类型指针，通过其自身是无法在运行期
获取任何相关类型信息的，反射操作通过接口变量获取其自身类型外，还会保存实际
对象的类型数据。

```go
func TypeOf(i interface{}) Type
func ValueOf(i interface{}) Value
```
上面两个反射入口函数，会将传入的任意对象都转换为接口类型。

```go
type X32 int32
func main(){
    var a X32 = 100 
    t := reflect.TypeOf(a)
    fmt.Println(t.Name(), t.Kind())                                                   
}
输出：
X32 int32
```
需要注意的是Type和Kind的区别： Type表示对象的真实(静态)类型，Kind表示基础(底层)结构类型，所以在做类型
判断上需要明确需求，然后使用正确的方式。

也可以使用反射来构造一些基础复合类型：

```go
func main(){
    s := reflect.SliceOf(reflect.TypeOf(0))                                           
    m := reflect.MapOf(reflect.TypeOf(""), reflect.TypeOf(0))
    fmt.Println(s , m)
}
输出：
[]int map[string]int
```
针对传入参数的类型需要加以注意，比如：

```go
type X32 int32
{   
    var a X32 = 100
    tx ,tp := reflect.TypeOf(a), reflect.TypeOf(&a)
    fmt.Println(tx,tp)
    fmt.Println(tx.Kind(),tp.Kind())
    fmt.Println(tp.Elem())
}
```
因为普通基类型和指针类型是两种类型，使用Elem()方法可以返回指针、数组、切片、字典(值)、或
通道的基类型。比如：

```go
{
    s := reflect.TypeOf([]int32{}).Elem()                                         
    m := reflect.TypeOf(map[string]int{}).Elem()
    fmt.Println(s , m)
}
```

如何遍历结构体的字段呢？

```go
type data struct{
    username string
    password string
}

type http struct{
    host string
    agent string
    data
}
func main(){
    var h http
    t := reflect.TypeOf(&h)
    if t.Kind() == reflect.Ptr{
        t = t.Elem()
    }
    //t := reflect.TypeOf(h)
    for i:= 0;i < t.NumField();i++{
        field := t.Field(i)
        fmt.Println(field.Name, field.Type, field.Offset)
        if field.Anonymous {
            for x:= 0;x < field.Type.NumField();x++{
                subField := field.Type.Field(x)
                fmt.Println(subField.Name, subField.Type, subField.Offset)
            }
        }
    }
}
输出：
host string 0
agent string 16
data main.data 32
username string 0
password string 16
```
如果传入的是指针类型，那么必须获取基类型后，才能遍历它的字段，反之传入的
是普通对象，不需要调用Elem函数获取基类型就可以遍历它的字段。

对于匿名字段可以用多级索引或按照定义顺序直接访问，比如：

```go
{
    var h http
    t := reflect.TypeOf(h)
    username,ok := t.FieldByName("username")
    if ok{ 
        fmt.Println(username.Name, username.Type)
    }   
    password := t.FieldByIndex([]int{2,1})                                            
    fmt.Println(password.Name, password.Type)
}
输出：
username string
password string
```
遍历方法集时要区分基本类型与指针类型，比如：

```go
func (h *http) getHost()string{
    return h.host
}
func (h http) getAgent()string{
    return h.agent
}

func (d *data) getUser()string{
    return d.username
}
func (d data) getPass()string{
    return d.password                                                                        
}
func main(){
    var h http
    t := reflect.TypeOf(&h)
    s := []reflect.Type{t,t.Elem()}  
    //t := reflect.TypeOf(h)     
    //s := []reflect.Type{t,t.Elem()}     // Elem of invalid type

    for _, m := range s{
        fmt.Println(m,":")
        for i := 0;i<m.NumMethod();i++{
            fmt.Println(" ", m.Method(i))
        }   
    }   
}
输出：
*main.http :
  {getAgent main func(*main.http) string <func(*main.http) string Value> 0}
  {getHost main func(*main.http) string <func(*main.http) string Value> 1}
  {getPass main func(*main.http) string <func(*main.http) string Value> 2}
  {getUser main func(*main.http) string <func(*main.http) string Value> 3}
main.http :
  {getAgent main func(main.http) string <func(main.http) string Value> 0}
  {getPass main func(main.http) string <func(main.http) string Value> 1}
```

通过反射能否得知其它包的非导出的结构成员呢？

```go
import nhp "net/http"       //alias
type data struct{
    username string
    password string
}
type http struct{
    host string
    agent string
    data
}

func main(){
    var s nhp.Server
    t := reflect.TypeOf(s)
    for i:= 0;i < t.NumField();i++{
        fmt.Println(t.Field(i).Name)
    }
}
输出：
Addr
......
disableKeepAlives
nextProtoOnce
nextProtoErr
```
利用反射获取结构体字段tag信息，这种做法常用于ORM映射或数据格式验证的场景下，比如：

```go
type Person struct{
    username string `field:"name" type:"varchar(60)"`
    password string `field:"passmd5" type:"varchar(128)"`                                    
}

func main(){
    var p Person
    t := reflect.TypeOf(p)
    for i:= 0;i < t.NumField();i++{
        f := t.Field(i)
        fmt.Println(f.Name,f.Tag.Get("field"), f.Tag.Get("type"))
    }
}
```
常见应用tag的场景是json，和db或者field，那么思考如下代码，s中是否有值，为什么？
```go
type S struct {
	DiskReadIOPS string `json:"disk-ReadIOPS"`
}
type D struct {
	DiskReadIOPS string `db:"disk-ReadIOPS"`
}
{
	var d = D{
		DiskReadIOPS: "30",
	}
    var s = S{}
	js1, _ := json.Marshal(&d)
    json.Unmarshal(js1, &s)
	fmt.Println(s)	
}
```
总结：
Unmarshal 是怎么找到结构体中对应的值呢？比如给定一个 JSON key Filed
```
    首先查找json tag 名字为 Field 的字段
    然后查找名字为 Field 的字段
    最后再找名字为 FiElD 等大小写不敏感的匹配字段
    如果都没有找到，就直接忽略这个 key，也不会报错
```
再比如：
```go
type peerInfo struct {  
    HTTPPort int  
    TCPPort  int  
    versiong string  
}  
  
func main() {  
    var v peerInfo  
    data := []byte(`{"http_port":80,"tcp_port":3306}`)  
    err := json.Unmarshal(data, &v)  
    if err != nil {  
        fmt.Println(err)  
    }  
    fmt.Printf("%+v\n", v)  
}
```

#### 值
重温一下Go反射的两个反射入口函数：

```go
func TypeOf(i interface{}) Type     //Type获取类型信息
func ValueOf(i interface{}) Value   //Value获取对象实例数据信息
```
因为在获取接口变量的时候是复制对象操作，并且接口变量是unaddressable的，所以要想修改对象，必须使用指针。

```go
{
    var i = 100 
    vi, vp, vpe := reflect.ValueOf(i), reflect.ValueOf(&i),reflect.ValueOf(&i).Elem()
    fmt.Println(vi.CanAddr(),vi.CanSet())
    fmt.Println(vp.CanAddr(),vp.CanSet())
    fmt.Println(vpe.CanAddr(),vpe.CanSet())
    //如何修改
    v := int64(7)
	rv := reflect.ValueOf(v)
	vpe.Set(rv)
	fmt.Println(i)
}
输出：
false false
false false
true true
```
说明只有通过Elem函数获取目标对象才能够对其进行取址和设置操作。

对无论是当前包还是其它包中结构体的非导出字段是不能直接进行设置操作的，比如：

```go
type person struct{
    Username string
    password string
}
func main(){
    var h person
    v := reflect.ValueOf(&h).Elem()
    name := v.FieldByName("Username")
    pass := v.FieldByName("password")
    code := v.FieldByName("code")           //？什么鬼 IsValid()
    fmt.Println(name.CanAddr(), name.CanSet())
    fmt.Println(pass.CanAddr(), pass.CanSet())
}
输出：
true true
true false
```
从上面的例子中看到，不能对非导出的字段进行设置，但是可以取址，那么是否可以通过其它渠道对其设置呢？

```go
{
    var h person
    v := reflect.ValueOf(&h).Elem()
    name := v.FieldByName("Username")
    pass := v.FieldByName("password")
    fmt.Println(name.CanAddr(), name.CanSet())
    fmt.Println(pass.CanAddr(), pass.CanSet())
    if name.CanSet(){
        name.SetString("Kite")
    }
    if pass.CanAddr(){
        *(*string)(unsafe.Pointer(pass.UnsafeAddr())) = "123123"                             
    }
    fmt.Println(h)
}
输出：
true true
true false
{Kite 123123}
```
UnsafeAddr返回该字段自身的地址，而Pointer返回该字段所保存的地址。

日常开发中，经常会遇到利用interface方法进行类型推断和转换的场景，比如：

```
type person struct{
    Username string
    Password string
}

func main(){
    h:= person{"tom","123123"}
    fmt.Println(h)
    v := reflect.ValueOf(&h)
    p,ok := v.Interface().(*person)
    if !ok {
        fmt.Println("v.Interface fail.")
        return
    }
    p.Password = "123456"
    fmt.Println(h)
}
```
如果操作对象是chanenl类型的话如何设置呢？

```go
{
    c := make(chan string, 5)
    v := reflect.ValueOf(c)
    if v.TrySend(reflect.ValueOf("hello gopher")){                                           
        fmt.Println(v.TryRecv())
    }
}
输出：
hello gopher true
```
注意trySend和TryRecv都是非阻塞操作函数

对于interface有两个nil状态，这给开发带来了麻烦，需要使用IsNil来判断值是否为nil，比如：

```go
{
    var a interface{} = nil
    var b interface{} = (*int)(nil)
    fmt.Println(a == nil)                                                                    
    fmt.Println(b == nil, reflect.ValueOf(b).IsNil())
}
输出：
true
false true
```

#### 方法

利用反射进行动态方法调用时，只需要按in参数列表准备好参数即可，对于变参来说，就是用CallSlice函数，比如：

```go
func (v Value) Call(in []Value) []Value
func (v Value) CallSlice(in []Value) []Value

type X int
func (X) Sum(x,y int)int {
    return x+y 
}

func (X) SumSlice(y ...interface{})int {
    sum := 0
    for i:= 0;i < len(y);i++{
        sum += y[i].(int)
    }
    return sum
}

func main(){
    var i X
    v := reflect.ValueOf(&i)
    m := v.MethodByName("Sum")

    args := []reflect.Value{
        reflect.ValueOf(1),
        reflect.ValueOf(2),
    }
    out := m.Call(args)                                                                      
    for _ , v := range out{
        fmt.Println(v)
    }

    m = v.MethodByName("SumSlice")
    args = []reflect.Value{
        reflect.ValueOf([]interface{}{1,2,3}),
    }
    out = m.CallSlice(args)
    for _ , v := range out{
        fmt.Println(v)
    }
}
```
比如：

```go
func (X) SumSlice(y ...interface{})int {
    sum := 0
    for i:= 0;i < len(y);i++{
        sum += y[i].(int)
    }
    return sum
}
```
```
注意无法利用反射调用未导出的方法，甚至无法对其进行取址
```

#### 性能
Go语言反射带来方便的他同时，也造成了很大性能上的损失，损失到很多人对其避之不及，比如：

赋值测试：

```go
type Person struct{
    Age int 
}
var p Person
func set(a int){
    p.Age = a 
}
func rset(a int){
    v := reflect.ValueOf(&p).Elem()
    f := v.FieldByName("Age")
    f.Set(reflect.ValueOf(a))
}
func BenchmarkSet(b *testing.B){
    for i:=0;i <b.N;i++{
        set(1)
    }   
}
func BenchmarkRset(b *testing.B){
    for i:=0;i <b.N;i++{
        rset(1)                                                                              
    }   
}
输出：
go test -run None -bench . -benchmem file_test.go
BenchmarkSet-2 	2000000000	         0.45 ns/op	       0 B/op	       0 allocs/op
BenchmarkRset-2	10000000	       160 ns/op	      16 B/op	       2 allocs/op
```
方法调用测试：

```go
func (p *Person)Add(){
    p.Age++
}
func do(){
    p.Add()
}

var v = reflect.ValueOf(&p)
var m = v.MethodByName("Add")
func rdo(){
    m.Call(nil)
}

func BenchmarkDo(b *testing.B){
    for i:=0;i <b.N;i++{
        do()
    }
}

func BenchmarkRdo(b *testing.B){
    for i:=0;i <b.N;i++{
        rdo()
    }   
}
输出：
# go test -run None -bench . -benchmem file_test.go
BenchmarkDo-2 	1000000000	         2.08 ns/op	       0 B/op	       0 allocs/op
BenchmarkRdo-2	10000000	       170 ns/op	       0 B/op	       0 allocs/op
```
实验证明反射机制对程序的性能还是有一定影响的，谨慎使用。
