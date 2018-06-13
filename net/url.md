### HTTP URL 解析
URL示例如下：
```text
    scheme://[userinfo@]host:port/path[?query][#fragment]
    http://user:pass@host.com:5432/path?k=v#f
1. scheme：指定协议部分，该URL的协议部分为"http"。在Internet中可以使用多种协议，如HTTP，FTP等等,本例中使用的是HTTP协议。在"HTTP"后面的“//”为分隔符
2. userinfo:代表用户认证信息，比如git clone http://userName:password@git.xxxx.git，ftp://joe:joepasswd@ftp.prep.edu/pub/name
3. host: 域名部分,也可以用ip表示
4. port: 端口部分，与域名之间用冒号 ":" 分隔,
5. path: 路径部分，用于说明资源位于服务器的哪个目录
6. query: 参数部分，通过指定参数查询正确数据
7. fragment: 片段部分，用于指定网页中的某个位置，指定的值就代表该位置的标志符
```

应用net包练习解析上述url

```golang
func(cmd *cobra.Command, args []string) {
		u, err := url.Parse(target)
		if err != nil {
			fmt.Println(err.Error())
		}
		fmt.Println(u.Scheme)
		fmt.Println(u.User.String())
		fmt.Println(u.Host)
	}
```

