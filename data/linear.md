### 顺序查找

顺序查找是通过使用列表的索引从列表的开始移动到结尾，检查每个元素，如果与搜索项不匹配，
则检查下一个元素,直到匹配成功为止。

```go
package main

import "fmt"

func linearSearch(datalist []int, key int) bool {
	for _, item := range datalist {
		if item == key {
			return true
		}
	}
	return false
}

func main() {
	items := []int{1, 2, 3}
	fmt.Println(linearSearch(items, 2))
}
```
