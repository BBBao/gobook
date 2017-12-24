# TEXT

```text
文件基本操作
文件读写
文件压缩
文件哈希
```

#### 文件基本操作

- 创建和删除文件

```go
相关包：
os
相关函数：
func Create(name string) (*File, error)
func Remove(name string) error
举例：
func main(){
    newFile , err := os.Create("f1.txt")    //创建
    if err != nil{
        log.Fatal(err)
    }   
    newFile.Close()    
    err := os.Remove("f1.txt")  //删除文件或目录
    if err != nil {
        log.Fatal(err)
    }                                                                                             
}
说明：
1. 以默认权限(0666)，默认属性(O_RDWR)，创建并打开新文件，如果文件存在，则清空原文件内容
2. 关闭并删除文件
```

- 裁剪和清空文件

```go
相关包：
os
相关函数：
func Truncate(name string, size int64) error
func (f *File) Truncate(size int64) error
举例：
func main() {
    n ：= 100
    err := os.Truncate("f1.txt", n) //裁剪一个文件到100个字节
    if err != nil {
        log.Fatal(err)
    }   
}
说明：
1. 如果文件少于n个字节，则文件中原始内容保留，剩余的字节以null字节填充
2. 如果文件超过n个字节，则超过的字节会被抛弃
3. 传入n为0则会清空文件
```

- 重命名和移动文件

```go
相关包：
os
相关函数：
func Rename(oldpath, newpath string) error
举例：
func main() {
    originalPath := "old.txt"
    newPath := "new.txt"
    err := os.Rename(originalPath, newPath) // == move
    if err != nil {
        log.Fatal(err)
    }                                                                                                                     
}
说明：
1. 可以用来改名，也可以用来移动，如果newPath已经存在，则会被替换
```

- 打开文件

```
相关包：
os
相关函数：
func Open(name string) (*File, error)
func OpenFile(name string, flag int, perm FileMode) (*File, error)
举例：
func main(){
    file, err := os.Open("f1.txt") 
    if err != nil {
        log.Fatal(err)
    }
    file.Close()

    file, err = os.OpenFile("f1.txt", os.O_APPEND, 0666)
    if err != nil {
        log.Fatal(err)
    }
    file.Close()
}
说明：
os.Open：只读(O_RDONLY)属性打开文件
os.OpenFile：
    arg3:权限模式
    arg2:打开文件时的属性，可以结合使用，比如：os.O_CREATE | os.O_APPEND
属性描述：
    os.O_RDONLY：只读
    os.O_WRONLY： 只写
    os.O_RDWR： 读写
    os.O_APPEND：追加写
    os.O_CREATE：如果文件不存在则先创建
    os.O_TRUNC：文件打开时裁剪文件
    os.O_EXCL：和 O_CREATE一起使用，文件不能存在
    os.O_SYNC： 以同步I/O的方式打开
```

- 复制文件

```go
相关包：
io
相关函数：
func Copy(dst Writer, src Reader) (written int64, err error)
举例：
func main() {
    src, err := os.Open("src.txt")
    if err != nil {
        log.Fatal(err)
    } 
    defer src.Close()
    dest, err := os.Create("dest.txt")
    if err != nil {
        log.Fatal(err)
    }   
    defer dest.Close()

    bytesWritten, err := io.Copy(dest, src)
    if err != nil {
        log.Fatal(err)
    }   
    log.Printf("Copied %d bytes.", bytesWritten)
    err = dest.Sync()
    if err != nil {
        log.Fatal(err)
    }   
}
说明：
1. 只读方式打开源文件，并设置延迟关闭
2. 创建新文件，并设置延迟关闭
3. 调用Copy函数进行数据拷贝
4. 同步缓冲区数据到磁盘
```
#### 文件读写
- 读文件

```go
相关包：
os
io/ioutil
bufio
相关函数：
func (f *File) Read(b []byte) (n int, err error)
func ReadFile(filename string) ([]byte, error)

func NewReader(rd io.Reader) *Reader
func (b *Reader) Peek(n int) ([]byte, error)
func (b *Reader) Read(p []byte) (n int, err error)
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
func (b *Reader) ReadString(delim byte) (string, error)
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)

func NewScanner(r io.Reader) *Scanner
func (s *Scanner) Split(split SplitFunc)
func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)

举例(正常读)：
func main() {
    file, err := os.Open("test.txt")
    if err != nil {
        log.Fatal(err)
    }   
    defer file.Close()
    byteSlice := make([]byte, 16) 
    bytesRead, err := file.Read(byteSlice)
    if err != nil {
        log.Fatal(err)
    }   
    log.Printf("Number of bytes read: %d\n", bytesRead)
    log.Printf("Data read: %s\n", byteSlice)                                                                              
}
说明：
1. 以只读方式打开文件，并设置延迟关闭
2. 创建一个适当大小的切片存放数据
3. 用file.Read函数从文件中进行数据读取
    返回值：正常情况返回读到的字节数，返回0说明已读到文件尾部，并错误值是io.EOF
举例(快速读)：
func main() {
    data, err := ioutil.ReadFile("test.txt")                                                                              
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Data read: %s\n", data)
}
说明：
1. 使用ReadFile函数读取文件里所有的数据存入data为名的字节数组中。
举例(缓存读)
func main() {
    file, err := os.Open("test.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    bufferedReader := bufio.NewReader(file)

    byteSlice := make([]byte, 5)
    byteSlice, err = bufferedReader.Peek(5)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Peeked at 5 bytes: %s\n", byteSlice)

    numBytesRead, err := bufferedReader.Read(byteSlice)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Read %d bytes: %s\n", numBytesRead, byteSlice)

    myByte, err := bufferedReader.ReadByte()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Read 1 byte: %c\n", myByte)

    dataBytes, err := bufferedReader.ReadBytes('\n')
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Read bytes: %s\n", dataBytes)

    dataString, err := bufferedReader.ReadString('\n')
    if err != nil {
        log.Fatal(err)
    }   
    fmt.Printf("Read string: %s\n", dataString)
    dataSlice, err := bufferedReader.ReadSlice('\n')
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Read slice: %s\n", dataSlice)

    dataLine,isPrefix ,err := bufferedReader.ReadLine()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Read line: %s, %v\n", dataLine,isPrefix)
}
说明：
1. 打开文件并设置延迟关闭
2. 创建用户态缓存读句柄
3. bufferedReader.Peek读取数据到缓存后恢复到原读位置
4. bufferedReader.Read读取数据到缓存
5. bufferedReader.ReadByte读取一个字节
6. bufferedReader.ReadBytes('\n')读取数据直到遇到'\n'字符，并包含'\n'字符一同以字符数组形式返回
7. bufferedReader.ReadString('\n')读取数据直到遇到'\n'字符，并包含'\n'字符一同以字符串形式返回
8. bufferedReader.ReadSlice('\n')读取数据直到遇到'\n'字符，并包含'\n'字符一同以字符切片形式返回
9. bufferedReader.ReadLine()读取一行数据

举例(Scanner)
func main() {
    file, err := os.Open("test.txt")
    if err != nil {
        log.Fatal(err)
    }   
    defer file.Close()

    scanner := bufio.NewScanner(file)
    scanner.Split(bufio.ScanLines)                                                                                      
    //scanner.Split(bufio.ScanWords)
    //scanner.Split(bufio.ScanBytes)
    for{
        success := scanner.Scan()
        if success == false {
            err = scanner.Err()
            if err == nil {
                log.Println("Scan completed and reached EOF")
                break
            } else {
                log.Fatal(err)
            }   
        }   
        fmt.Println("found:", scanner.Text())
    }   
}
说明：
1. 打开文件并设置延迟关闭 
2. 创建scanner
3. 设定切分格式函数,如果不指定splitFunc，默认采用行切分函数
4. scanner.Scan()按照split格式扫描数据，典型的操作是按行读取文件
```

1. 写文件

```go
相关包：
os
io/ioutil
bufio
相关函数：
正常写：
func (f *File) Write(b []byte) (n int, err error)
快写：
func WriteFile(filename string, data []byte, perm os.FileMode) error
缓存写：
func NewWriter(w io.Writer) *Writer
func (b *Writer) Write(p []byte) (int,error)
func (b *Writer) WriteString(s string) (int, error)
举例(普通写)：
func main() {
    file, err := os.OpenFile("testwrite.txt",os.O_WRONLY|os.O_TRUNC|os.O_CREATE,0666)
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()
    byteSlice := []byte("Bytes!\n")
    bytesWritten, err := file.Write(byteSlice)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Wrote %d bytes.\n", bytesWritten)
}
说明：
1. 以写方式打开文件，文件不存在就创建文件
2. 因为write函数接收的字符数组，所以需要将字符串转成数组或切片
3. 写入文件时，如果写成功，则返回写入了字节数组的长度，否则返回错误值

举例(快写)：
func main() {
    err := ioutil.WriteFile("fast.txt", []byte("Hi\n"), 0666)
    if err != nil {
        log.Fatal(err)
    }                                                                                                                     
}
说明：
1. 如果文件不存在，则创建带有0666权限(可改)的新文件
2. 如果文件存在，则会清空文件，写入新的数据

举例(缓存写)：
func main() {
    file, err := os.OpenFile("bufwrite.txt", os.O_WRONLY|os.O_CREATE, 0666)
    if err != nil {
        log.Fatal(err)
    }   
    defer file.Close()

    bufferedWriter := bufio.NewWriter(file)
    bytesWritten, err := bufferedWriter.Write([]byte("Buffered bytes"))
    if err != nil {
        log.Fatal(err)
    }   
    log.Printf("Bytes written: %d\n", bytesWritten)

    bytesWritten, err = bufferedWriter.WriteString("Buffered string\n")
    if err != nil {
        log.Fatal(err)
    }   
    log.Printf("Bytes written: %d\n", bytesWritten)

    unflushedBufferSize := bufferedWriter.Buffered()
    log.Printf("Bytes buffered: %d\n", unflushedBufferSize)
    bytesAvailable := bufferedWriter.Available()
    if err != nil {
        log.Fatal(err)
    }   
    log.Printf("Available buffer: %d\n", bytesAvailable)
    bufferedWriter.Flush()

    bufferedWriter.Reset(bufferedWriter)
    bytesAvailable = bufferedWriter.Available()
    if err != nil {
        log.Fatal(err)
    }   
    log.Printf("Available buffer: %d\n", bytesAvailable)

    bufferedWriter = bufio.NewWriterSize(
        bufferedWriter,
        8000,
    )   

    bytesAvailable = bufferedWriter.Available()
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Available buffer: %d\n", bytesAvailable)
}
说明：
1. 用os.OpenFile函数打开或创建文件
2. 用bufio.NewWriter函数创建一个buffer writer，即用户态写缓存描述符
3. 用bufferedWriter.Write系列函数进行写入数据到缓存
4. 用bufferedWriter.Buffered函数获取当前缓存的使用量
5. 用bufferedWriter.Available函数获取当前缓存的剩余量
6. 用bufferedWriter.Flush函数刷新缓存的数据到磁盘
7. 用bufferedWriter.Reset重置缓存，丢弃未刷新的数据，缓存的默认大小为4096字节
8. 用bufio.NewWriterSize修改缓存容量，多用于扩容操作
```

- 文件读写偏移设置

```go
相关包：
os
相关函数：
func (f *File) Seek(offset int64, whence int) (ret int64, err error)
举例：
func main() {
    file, _ := os.Open("test.txt")
    defer file.Close()

    var offset int64 = 5 
    var whence int = 0 

    newPosition, err := file.Seek(offset, whence)
    if err != nil {
        log.Fatal(err)                                                                                                    
    }   

    whence = 1
    newPosition, err = file.Seek(-2, whence)
    if err != nil {
        log.Fatal(err)
    }   
    currentPosition, err := file.Seek(0,whence)     //获取当前位置
    fmt.Println("Current position:", currentPosition)
    whence = 0
    newPosition, err = file.Seek(0, whence)
    if err != nil {
        log.Fatal(err)
    }   
    fmt.Println("Position after seeking 0,0:", newPosition)
}
说明：
offset：设置偏移的值，正负值都可以
whence：设置相对位置，可以取的值为：
    0: 文件开始位置
    1: 文件当前位置
    2: 文件结尾位置
```

#### 解压缩

- 目录归档

```go
相关包：
archive/zip
os
io/ioutil
path/filepath
相关函数：
func NewWriter(w io.Writer) *Writer
func (w *Writer) Close() error
func (w *Writer) Create(name string) (io.Writer, error)
举例：
const(
    dirName = "sum"
)

func main() {
    outFile, err := os.Create("test.zip")
    if err != nil {
        log.Fatal(err)
    }   
    defer outFile.Close()

    zipWriter := zip.NewWriter(outFile)

    dirAbs,err := filepath.Abs(dirName)
    if err != nil{
        log.Fatal(err)
    }   
    println(dirAbs)
    fileInfos, err := ioutil.ReadDir(dirAbs)
    if err != nil {
        log.Fatal(err)
    }   

    for _, file := range fileInfos{
        src := filepath.Join(dirAbs,file.Name())
        dest := filepath.Join(dirName,file.Name())

        f,err := zipWriter.Create(dest)
        if err != nil{
            continue
        }   
        content, err := ioutil.ReadFile(src)
        if err != nil{
            continue
        }   
        f.Write(content)
    }

    err = zipWriter.Close()
    if err != nil {
            log.Fatal(err)
    }
}
说明：
1. 创建并打开目标压缩文件，设置延迟关闭
2. 使用zip.NewWriter函数创建写压缩文件的指针zipWriter
3. 使用ioutil.ReadDir函数搜集被压缩目录下的文件信息fileInfos
4. 使用zipWriter.Create函数创建压缩文件中的文件指针f
5. 读取原文件数据，并通过f.Write函数写入到压缩文件中
6  关闭压缩文件的指针zipWriter
```

- 目录抽取

```go
相关包：
archive/zip
os
io
path/filepath
相关函数：
func OpenReader(name string) (*ReadCloser, error)
func (rc *ReadCloser) Close() error
举例：
func main() {
    zipReader, err := zip.OpenReader("test.zip")
    if err != nil {
        log.Fatal(err)
    }   
    defer zipReader.Close()

    for _, file := range zipReader.Reader.File {
        zippedFile, err := file.Open()
        if err != nil {
            log.Println(err)
            continue
        }   
        defer zippedFile.Close()

        targetDir := "./"
        fields := strings.Split(file.Name,"/")
        if len(fields) >1{ 
            parent := strings.Join(fields[:len(fields)-1],"/")
            os.MkdirAll(parent , 0755)
        }
        extractedFilePath := filepath.Join(
            targetDir,
            file.Name,
        )
        if file.FileInfo().IsDir() {
            os.MkdirAll(extractedFilePath, file.Mode())
        } else {
            outputFile, err := os.OpenFile(
                extractedFilePath,
                os.O_WRONLY|os.O_CREATE|os.O_TRUNC,
                file.Mode(),
            )
            if err != nil {
                log.Println(err)
                continue
            }
            defer outputFile.Close()
            _, err = io.Copy(outputFile, zippedFile)
            if err != nil {
                log.Println(err)
                continue
            }
        }
    }
}
说明：
1. 用zip.OpenReader函数创建zipReader指针，用于获取归档中文件
2. 循环读取归档中的文件或目录
3. 创建存放抽取后的文件目录
4. 如果抽取的目录，则需创建目录
5. 如果是文件，就利用io.Copy函数将数据拷贝到目标文件
```

- 文件压缩

```
相关包：
compress/gzip
相关函数：
func NewWriter(w io.Writer) *Writer
func (z *Writer) Write(p []byte) (int, error)
func (z *Writer) Close() error
举例：
func main() {
    outputFile, err := os.Create("test.gz")
    if err != nil {
        log.Fatal(err)
    }   
    defer outputFile.Close()

    gzipWriter := gzip.NewWriter(outputFile)
    defer gzipWriter.Close()

    _, err = gzipWriter.Write([]byte("Hello Gophers!\n"))
    if err != nil {
        log.Fatal(err)
    }   
    log.Println("Compressed data to file success.")
} 
说明：
1. 创建并打开压缩文件
2. 使用gzip.NewWriter函数创建压缩文件写指针gzipWriter
3. 使用gzipWriter。Write函数写入数据
```


- 文件解压缩

```go
相关包：
compress/gzip
相关函数：
func NewReader(r io.Reader) (*Reader, error)
func (z *Reader) Close() error
举例：
func main() {
    gzipFile, err := os.Open("test.gz")
    if err != nil {                                                                                                       
        log.Fatal(err)
    }   
    defer gzipFile.Close()
    gzipReader, err := gzip.NewReader(gzipFile)
    if err != nil {
        log.Fatal(err)
    }   
    defer gzipReader.Close()

    outFileWriter, err := os.Create("gunzip.txt")
    if err != nil {
        log.Fatal(err)
    }   
    defer outFileWriter.Close()

    _, err = io.Copy(outFileWriter, gzipReader)
    if err != nil {
        log.Fatal(err)
    }   
}
说明：
1. 只读方式打开原始压缩文件
2. 使用gzip.NewReader创建压缩文件的读指针
3. 使用os.Create创建并打开一个目标文件，用来存解压后的数据
4. 使用io.Copy拷贝数据
```

#### 文件哈希
- 小文件快速计算哈希

```go
相关包：
crypto/md5
crypto/sha1
crypto/sha256
crypto/sha512
io/ioutil
相关函数：
func New() hash.Hash
func Sum(data []byte) [Size]byte
举例：
func main() {
    data, err := ioutil.ReadFile("smallfile.txt")
    if err != nil {
        log.Fatal(err)
    }                                                                                          

    fmt.Printf("Md5: %x\n", md5.Sum(data))
    fmt.Printf("Sha1: %x\n", sha1.Sum(data))
    fmt.Printf("Sha256: %x\n", sha256.Sum256(data))
    fmt.Printf("Sha512: %x\n", sha512.Sum512(data))
}
说明：
1. 读取文件内容到data字符数组
2. 通过各类哈希算法计算哈希值
```
- 大文件分段计算哈希

```go
相关包：
crypto/md5
os
io
bufio
相关函数：
func New() hash.Hash
func Sum(data []byte) [Size]byte
func WriteString(w Writer, s string) (n int, err error)
举例：
const(
    trunkSize = 4096
)
func main() {
    file, err := os.Open("bigfile.txt")
    if err != nil {
        log.Fatal(err)
    }   
    defer file.Close()
    hasher := md5.New()
    bufferedReader := bufio.NewReader(file)
    for{
        byteSlice := make([]byte, trunkSize)                                                   
        numRead, err := bufferedReader.Read(byteSlice)
        if err != nil {
            log.Println(err)
            break
        }
        log.Println(numRead)
        log.Println(string(byteSlice))
        io.WriteString(hasher,string(byteSlice))
    }   
    log.Printf("%x\n",hasher.Sum(nil))
}
说明：
1. 以只读方式打开文件，并设置延迟关闭
2. 用md5.New函数新建md5函数接口实例hasher
3. 设置缓存读bufferedReader
4. 按段获取数据并追加写入hasher
5. 用hasher.Sum函数计算哈希值
```

以上是日常开发中经常用到的文件操作需求，标准库还有很多其它的文件操作函数，可以查阅官方文档，进一步学习。
