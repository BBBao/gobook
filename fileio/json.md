# Json文件

```text
json基本操作
json文件操作
```
- json基本操作

json包使用map[string]interface{}和 []interface{} 储存任意的 JSON 对象和数组。
常用在结构体与json之间的互相转换，但是从结构体转json时需注意各字段名称的首写字符必须大些，否则转化将失败，比如：

```go
相关包：
import "encoding/json"
相关函数：
func Marshal(v interface{}) ([]byte, error)
func Unmarshal(data []byte, v interface{}) error
举例：
type Response1 struct {
    page   int 
    fruits []string
}

type Response2 struct {
    Page   int      `json:"page"`
    Fruits []string `json:"fruits"`
}

func main(){
    res1D := &Response1{
        page:   1,  
        fruits: []string{"apple", "peach", "pear"}}                                                                             
    res1B, _ := json.Marshal(res1D)
    fmt.Println(string(res1B))

    res2D := &Response2{
        Page:   1,  
        Fruits: []string{"apple", "peach", "pear"}}
    res2B, _ := json.Marshal(res2D)
    fmt.Println(string(res2B))
}
输出：
{}
{"page":1,"fruits":["apple","peach","pear"]}
```
> 注意：如果结构体变量定义时没有标签，json转换之后中的Key名称与变量名相同，均是大写的

- json文件操作

json除了作为系统之间通讯消息格式除外，为了让本地配置文件更具可读性和配置文件解析的便利性，通常使用json数据格式的
配置文件，比如：

```go
cfg.json文件：
{
	"enable":true,
	"hitcount":100,
	"seconds":1
}

type RateLimitConfig struct {
    Enable bool `json:"enable"`             //是否开启限速
    HitCount int    `json:"hitcount"`       //单位时间内执行次数
    Seconds  int    `json:"seconds"`        //单位时间，秒?
}

func main(){
    raw, err := ioutil.ReadFile("cfg.json")
    if err != nil {
        log.Fatal(err.Error())
    }   
    c := RateLimitConfig{}
    if err := json.Unmarshal(raw, &c); err != nil {
        log.Fatal(err.Error())
    }   
    log.Println(c.Enable, c.HitCount, c.Seconds)
}
输出：
true 100 1
```

json文件语法格式校验命令行工具jq,比如：

```json
{
  "enable": true,
  "hitcount": 100,
  "seconds": 1
}
校验：
# jq . cfg.json
parse error: Expected separator between values at line 3, column 15
```

在线json与go语言结构体转换工具：*[struct2json](http://www.unicode.org)*。

