# 揭开golang中defer的坑

- defer执行顺序，后进先出，是个栈
- 函数在执行最后的RET返回指令前，会先检查是否存在defer语句，如果有从栈中依次取出
- 匿名返回值在return执行时被声明然后给他赋值，所以并不会返回在defer中修改后的值，有名返回值在函数声明时即被声明，在defer中修改即是修改返回值
- return内部有两个步骤，一是给返回值赋值（有名返回值直接赋值，匿名返回值则先声明再赋值），二是调用RET返回指令并传入返回值，RET会检查defer是否存在，执行完defer语句后携带返回值退出函数。

## 1、匿名返回值的情况
```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("return:", test()) // 打印结果为 return: 0
	return
}
func test() int{
	i := 0
	defer func (){
		i++
		fmt.Println("defer2:", i) // 打印结果为 defer2: 2
	}()
	defer func (){
		i++
		fmt.Println("defer1:", i) // 打印结果为 defer1: 1
	}()
	return i //假如返回值是a，此时a=i，defer中修改i的值不会影响返回值a，defer也根本访问不到a
}

输出结果：
defer1: 1
defer2: 2
return: 0
```

## 2、有名返回值的情况

```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("return:", test()) // 打印结果为 return: 7
	return
}
func test() (i int) {
	i= 5
	defer func() {
		i++
		fmt.Println("defer2:", i) // 打印结果为 b defer2: 7
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i) // 打印结果为 b defer1: 6
	}()
	return i // 返回值就是i，defer修改就是i的值
}

输出结果：
defer1: 6
defer2: 7
return: 7
```

## 3、匿名返回值是指针的情况

- return时将i的地址赋值给返回值后，返回值即指向改地址，当defer修改地址的值时，退出函数时RET携带的返回值依旧是i的地址，值当然会变化

```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("return:", *test()) // 打印结果为 return: 11
	return
}
func test() *int {
	i := 9
	defer func() {
		i++
		fmt.Println("defer2:", i, &i) // 打印结果为 defer2: 11 0xc042052088
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i, &i) // 打印结果为 defer1: 10 0xc042052088
	}()
	return &i
}

输出结果：
defer1: 10 0xc042052088
defer2: 11 0xc042052088
return: 11
```

## 4、defer声明时会计算确定参数的值，defer推迟执行的是函数体

```
package main

import (
	"errors"
	"fmt"
)

type people struct {
	Name string
	Age  int
}

func main() {
	a := new(int)
	*a = 10
	p := &people{
		Name: "hah",
		Age:  88,
	}
	defer fmt.Println(*a, *p) //打印 10 {hah 88}，*a和*p都是确定的参数
	defer fmt.Println(*a, p) //打印  10 &{jack 12}
	*a = 20
	p.Name = "jack"
	p.Age = 12
}

输出结果：
10 &{jack 12}
10 {hah 88}
```

## 5、defer作用域
- defer只对当前函数有效
- 当发生panic时，当前函数之前声明的defer会被执行，如果defer没有调用recover进行恢复，执行完所有defer后进程崩溃
- 若主动调用os.Exit(int)退出进程，defer不会执行






