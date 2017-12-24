# redis

```text
Redis数据库的CRUD操作
```

针对数据库操作，需要安装第三方数据库驱动包，比如：

```sh
go get "gopkg.in/redis.v5"
```

- 连接数据库测试，比如：

```go
相关包：
"gopkg.in/redis.v5"
相关函数：
func NewClient(opt *Options) *Client
func (c *Client) Close() error
举例：
func createClient() (*redis.Client,error) {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",   //no password set
        DB:       0,    //use default DB
    })
    _, err := client.Ping().Result()
    if err != nil{
        return nil, err
    }
    return client, nil
}
func main() {
    cli, err := createClient()
    if err != nil{
        log.Fatal(err)
    }   
    defer cli.Close()
}
说明：
1. 应用redis.NewClient函数建立与Redis服务器的连接，默认连接数据库0
2. 连接后通过 cient.Ping函数来验证是否成功连接Redis服务器
3. 设置连接关闭
```

- 数据读写操作，比如：

```go
相关包：
"gopkg.in/redis.v5"
相关函数：
func (c *Client) Set(key string, value interface{}, expiration time.Duration) *StatusCmd
func (cmd *StatusCmd) Err() error
func (c *Client) Get(key string) *StringCmd
func (cmd *StringCmd) Result() (string, error)
func (c *Client) Del(keys ...string) *IntCmd
写数据举例：
func setRedis(client *redis.Client,key,value string,  expiration  time.Duration) error{
    err := client.Set(key, value, expiration).Err()
    if err != nil {
        return err 
    }
    return nil 
}
说明：
1. 应用client.Set()函数进行数据存储，expiration是过期时间，如果是0的话，将永久不会过期
2. 应用Err()函数获取数据存储结果信息
3. 应用client.Del(key)函数可以删除key
读数据举例：
func getRedis(client *redis.Client, key string) (string, error) {
    return client.Get(key).Result()
}
说明：
1. 应用client.Get()函数进行数据获取，并通过Result()函数对返回结果进行格式化
2. 获取数据时如果发生错误，则可以通过错误值是否为redis.Nil来判定key是否存在
```
可以通过(*[go-redis](https://godoc.org/gopkg.in/redis.v5)*)的官方文档查询下更多API应用;

