# mongo

```text
Mongo数据库的CRUD操作
```

针对数据库操作，需要安装第三方数据库驱动包，比如：

```sh
go get "gopkg.in/mgo.v2"
```

- 连接数据库测试，比如：

```go
相关包：
"gopkg.in/mgo.v2"
"gopkg.in/mgo.v2/bson"
相关函数：
func Dial(url string) (*Session, error)
[mongodb://][user:pass@]host1[:port1][,host2[:port2],...][/database][?options]
func (s *Session) SetMode(consistency mode, refresh bool)
func (s *Session) Close()
举例：
type Person struct {
    ID        bson.ObjectId `bson:"_id,omitempty"`
    Name  string
    Phone string
    Timestamp   time.Time
}

func connDB()(*mgo.Session, error) {
    session, err := mgo.Dial("127.0.0.1")
    if err != nil {
        return session, err
    }
    //Optional. Switch the session to a monotonic behavior.
    session.SetMode(mgo.Monotonic, true)
    return session, err
}
func main() {
    s, err := connDB()
    if err != nil {
        log.Fatal(err)
    }
    defer s.Close()
}
说明：
BSON是一种类json的二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象，
MongoDB使用了BSON这种结构来存储数据和网络数据交换。
1. 应用mgo.Dial函数链接mongoDB服务器集群
2. 设置会话一致性访问模式
3. 连接成功后，设置会话关闭
```

- 插入数据，比如：

```go
相关包：
"gopkg.in/mgo.v2"
"gopkg.in/mgo.v2/bson"
相关函数：
func (c *Collection) Insert(docs ...interface{}) error
举例：
func inserGopher(s *mgo.Session,name string, phone string) error {
    c := s.DB("go").C("gopher")
    err := c.Insert(&Person{Name:name, Phone:phone,Timestamp:time.Now()})
    if err != nil {
        return err
    }
    return nil
}
说明：
1. 应用s.DB().C()函数选择相应的数据表
2. 应用c.Insert()函数对数据表进行插入数据操作
```

- 查询数据，比如：

```go
相关包：
"gopkg.in/mgo.v2"
"gopkg.in/mgo.v2/bson"
相关函数：
func (c *Collection) Find(query interface{}) *query
func (q *Query) All(result interface{}) error
举例：
查一条：
func findGopher(s *mgo.Session, name string)(Person , error){
    c := s.DB("go").C("gopher")
    result := Person{}
    err := c.Find(bson.M{"name": name}).One(&result)
    if err != nil {
        log.Println(err)
        return result,err
    }
    fmt.Println("Name:",result.Name, "Phone:", result.Phone)
    return result, nil
}
查多条：
func findAllGopher(s *mgo.Session, name string)([]Person , error){
    c := s.DB("go").C("gopher")
    results := []Person{}
    err := c.Find(bson.M{"name": name}).All(&results)
    //err := c.Find(nil).All(&results)  //select * records
    if err != nil {
        log.Println(err)
        return results,err
    }
    for _,  r := range results{
        fmt.Println("Name:",r.Name, "Phone:", r.Phone)
    }
    return results, nil
}
说明：
1. 应用s.DB().C()函数选择相应的数据表
2. 应用c.Insert()函数对数据表进行插入数据操作
```

- 修改数据，比如：

```go
相关包：
"gopkg.in/mgo.v2"
"gopkg.in/mgo.v2/bson"
相关函数：
func (c *Collection) Update(selector interface{}, update interface{}) error
举例：
func updateGopher(s *mgo.Session, name string) error {
    c := s.DB("go").C("gopher")
    selector := bson.M{"name": name}
    change := bson.M{"$set": bson.M{"phone": "+86 99 8888 7777", "timestamp": time.Now()}}
    err := c.Update(selector, change)
    if err != nil {
        return err
    }
    return nil
}
说明：
1. 应用s.DB().C()函数选择相应的数据表
2. 根据条件生成selector，也可以设置为nil，默认无条件
3. 应用c.Update()函数进行数据更新
```

- 删除数据，比如：

```go
相关包：
"gopkg.in/mgo.v2"
"gopkg.in/mgo.v2/bson"
相关函数：
func (c *Collection) Remove(selector interface{}) error
func (c *Collection) RemoveAll(selector interface{}) (info *ChangeInfo, err error)
举例：
func removeGopher(s *mgo.Session, name string)error {
    c := s.DB("go").C("gopher")
    selector := bson.M{"name": name}
    //err := c.Remove(selector)
    _, err := c.RemoveAll(selector)
    if err != nil {
        return err
    }
    return nil
}
说明：
1. 应用s.DB().C()函数选择相应的数据表
2. 根据条件生成selector，也可以设置为nil，默认无条件
3. 应用c.Remove或c.RemoveAll函数删除一条/多条数据
另外，可以通过API删除Collection和Database，比如：
func (c *Collection) DropCollection() error
func (db *Database) DropDatabase() error
```

可以通过(*[mgo](https://godoc.org/labix.org/v2/mgo#pkg-index)*)的官方文档查询下更多API应用;
