#### 介绍
etcd是随着CoreOS项目一起成长起来的，随着Golang和CoreOS等项目在开源社区日益火热， etcd作为一个高可用、(raft)强一致性的分布式Key-Value存储系统，被越来越多的开发人员关注和使用；
这篇文章全方位介绍了etcd的应用场景，这里简单摘要如下：
```
服务发现（Service Discovery）
消息发布与订阅
负载均衡
分布式通知与协调
分布式锁
分布式队列
集群监控与Leader竞选
```
本文重点介绍如何利用 ectd 实现一个分布式锁。 锁的概念大家都熟悉，当我们希望某一事件在同一时间点只有一个线程(goroutine)在做，或者某一个资源在同一时间点只有一个服务能访问，这个时候我们就需要用到锁。 例如我们要实现一个分布式的id生成器，多台服务器之间的协调就非常麻烦。分布式锁就正好派上用场。

#### 安装&启动
```bash
./etcd -listen-client-urls "http://0.0.0.0:2379"  -advertise-client-urls "http://0.0.0.0:2379"
```

#### 测试

```go
import (
	"context"
	"fmt"
	"time"

	"github.com/coreos/etcd/clientv3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2379", "localhost:22379", "localhost:32379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		fmt.Println("connect failed, err:", err)
		return
	}

	fmt.Println("connect succ")
	defer cli.Close()
	//设置1秒超时，访问etcd有超时控制
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	//操作etcd
	_, err = cli.Put(ctx, "/reboot/conf/", "sample_value")
	//操作完毕，取消etcd
	cancel()
	if err != nil {
		fmt.Println("put failed, err:", err)
		return
	}
	//取值，设置超时为1秒
	ctx, cancel = context.WithTimeout(context.Background(), time.Second)
	resp, err := cli.Get(ctx, "/reboot/conf/")
	cancel()
	if err != nil {
		fmt.Println("get failed, err:", err)
		return
	}
	for _, ev := range resp.Kvs {
		fmt.Printf("%s : %s\n", ev.Key, ev.Value)
	}
}
```

#### 验证
```bash
export ETCDCTL_API=3
ETCDCTL_API=3 ./etcdctl get /logagent/conf/
```
