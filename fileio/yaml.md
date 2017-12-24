# Yaml
```text
yaml读写
```
针对YAML操作，需要引入第三方包： 

```sh
#go get  gopkg.in/yaml.v1
```

yaml测试文件：test.yaml

```yaml
enable: true
hitcount: 100
servers:
  port: 80
  host: [192.168.1.1, 192.168.1.2]
seconds: 1
```

yaml文件解析代码：

```go
package main

import (
	"fmt"
	"log"
	"io/ioutil"
	"gopkg.in/yaml.v1"
)

type RateLimitConfig struct {
        Enable bool
        HitCount int
        Servers []struct{Port int,Host []string}
        Seconds  int
}

func main() {
	t := T{}
	data, err := ioutil.ReadFile("test.yaml")
	if err!= nil{
		log.Fatal(err)
	}
	err = yaml.Unmarshal([]byte(data), &t)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- t:\n%v\n\n", t)

	d, err := yaml.Marshal(&t)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- t dump:\n%s\n\n", string(d))

	m := make(map[interface{}]interface{})
	err = yaml.Unmarshal([]byte(data), &m)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- m:\n%v\n\n", m)

	d, err = yaml.Marshal(&m)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- m dump:\n%s\n\n", string(d))
}
说明：
1. 读取配置文件内容
2. 利用yaml.Unmarshal函数反序列化成目标结构体
3. 利用yaml.Marshal函数序列化数据
4. 可以将数据反序列化后存入字典中，再对其序列化输出
```
