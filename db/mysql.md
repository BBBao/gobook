# mysql

```text
Mysql数据库的CRUD操作
```

针对数据库操作，需要安装第三方数据库驱动包，比如：

```sh
go get "github.com/go-sql-driver/mysql"
```

- 连接数据库测试，比如：

```go
相关包：
"database/sql"
_ "github.com/go-sql-driver/mysql"
相关函数：
func (d MySQLDriver) Open(dsn string) (driver.Conn, error)
[username[:password]@][protocol[(address)]]/dbname[?param1=value1&...&paramN=valueN]
func (db *DB) Close() error
举例：
import (
    "log"
    "database/sql"
    _ "github.com/go-sql-driver/mysql"

)

func connDB() (*sql.DB, error)  {
    db , err := sql.Open("mysql", "root:root@tcp(localhost:3306)/go?charset=utf8")
    if err != nil {
        log.Println(err)
        return nil,err
    }   
    return db,nil
}
func main() {
    db , err := connDB()
    if err!=nil{
        log.Fatal(err)
    }
    defer db.Close()
    log.Println("conn mysql db success")
}
说明：
1. 应用sql.Open函数连接数据库，参数描述可见官方文档
2. 连接成功之后，要记得关闭数据库链接
```

- 创建表，比如：

```go
相关包：
"database/sql"
_ "github.com/go-sql-driver/mysql"
相关函数：
func (db *DB) Prepare(query string) (*Stmt, error)
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
举例：
func createTable(db *sql.DB,tableName string)error{
    create := fmt.Sprintf("create table if not exists %s(id int UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,name varchar(
64),status char DEFAULT 'u')",tableName)
    stmt, err := db.Prepare(create) 
    if err!= nil{
        return err
    }
    _, err = stmt.Exec()
    if err != nil {
        return err
    }
    stmt.Close()
    return nil
}
func main() {
    var tableName = "gopher"  
    db , err := connDB()
    if err!=nil{
        log.Fatal(err)
    }
    defer db.Close()
    log.Println("conn mysql db success")
    err = createTable(db, tableName)
    if err != nil{
        log.Fatal(err)
    }
    log.Println("create table success")
}
说明：
1. 延续链接数据库的例子，进而编辑创建表的sql语句
2. 调用Exec函数执行sql语句
```

- 插入数据，比如：

```go
相关包：
"database/sql"
_ "github.com/go-sql-driver/mysql"
相关函数：
func (db *DB) Prepare(query string) (*Stmt, error)
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
func (s *Stmt) Close() error
举例：
func insertGopherTable(db *sql.DB,name string, status string)error{
    stmt, err := db.Prepare(`INSERT gopher(name,status) values (?,?)`)
    if err!= nil{
        return err
    }
    _, err = stmt.Exec(name,status)
    if err != nil {
        return err
    }
    stmt.Close()
    return nil
}
说明：
1. 延续链接数据库的例子，进而编辑创建表的sql语句
2. 调用Exec函数执行sql语句
```

- 修改数据，比如：

```go
相关包：
"database/sql"
_ "github.com/go-sql-driver/mysql"
相关函数：
func (db *DB) Prepare(query string) (*Stmt, error)
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
func (s *Stmt) Close() error
举例：
func updateGopherTable(db *sql.DB,name string, status string,id int)error{
    stmt, err := db.Prepare(`update gopher SET name=?,status=? WHERE id=?`)
    if err!= nil{
        return err
    }
    _, err = stmt.Exec(name,status,id)
    if err != nil {
        return err
    }
    stmt.Close()
    return nil
}
说明：
1. 延续链接数据库的例子，进而编辑创建表的sql语句
2. 调用Exec函数执行sql语句
```

- 查询数据，比如：

```go
相关包：
"database/sql"
_ "github.com/go-sql-driver/mysql"
相关函数：
func (db *DB) Prepare(query string) (*Stmt, error)
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
func (s *Stmt) Query(args ...interface{}) (*Rows, error)
func (s *Stmt) QueryRow(args ...interface{}) *Row
func (s *Stmt) Close() error
func (rs *Rows) Close() error
查询一条数据：
func queryGopherTableOne(db *sql.DB,id int) (string, error) {
    var name string
    stmt, err := db.Prepare("select name from gopher where (id=?)")
    if err!= nil{
        return name, err
    }
    err = stmt.QueryRow(id).Scan(&name)
    if err != nil {
        return name, err
    }
    stmt.Close()
    return name ,nil 
}
也可以简化成如下：
func queryGopherTableOne(db *sql.DB,id int) (string, error) {
    var name string
    err := db.QueryRow("select name from gopher where id = ?",id).Scan(&name)
    if err !=nil{
        return name, err
    }
    return name, nil
}

查询多条数据(一)：
func queryGopherTable(db *sql.DB) error {
    rows, err := db.Query("SELECT name,status FROM gopher")
    if err != nil{
        return err
    }
    defer rows.Close()
    for rows.Next(){
        var(
            name string
            status string
        )
        err = rows.Scan(&name, &status)
        if err != nil{
            continue
        }
        log.Println(name, status)
    }
    return nil
}
查询多条数据(二)：
func queryGopherTable2(db *sql.DB) error {
    rows, err := db.Query("SELECT name,status FROM gopher")
    if err != nil{
        return err
    }
    columns, _ := rows.Columns()
    scanArgs := make([]interface{}, len(columns))
    values := make([]interface{}, len(columns))
    
    for i := range values {
            scanArgs[i] = &values[i]
    
    }
    for rows.Next() {
        err = rows.Scan(scanArgs...)
        record := make(map[string]string)
        for i, col := range values {
            if col != nil {
                record[columns[i]] = string(col.([]byte))
            }
        }
        fmt.Println(record)
    }
    return nil
}

```
- 删除数据，比如：

```go
func deleteGopherTable(db *sql.DB,id int) error{
    stmt, err := db.Prepare(`delete from gopher where id=?`)
    if err != nil{
        return err
    }
    _, err = stmt.Exec(id)
    if err != nil {
        return err
    }   
    stmt.Close()
    return nil
}
说明：
逻辑与其它数据操作逻辑相同
```

可以通过(*[mysql](https://godoc.org/github.com/go-sql-driver/mysql)*)的官方文档查询下更多API应用;