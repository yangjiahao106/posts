
defer 的匿名函数中引用了外部变量a，a 的值在for循环结束最后会被设为"mouse", 所以defer中的输出都为"mouse"

```golang
func main() {
	animals := []string{"dog", "cat", "mouse"}
	for _, a := range animals{
		defer func() {
			fmt.Println(a)
		}()
	}
}

// output:
// mouse
// mouse
// mouse
```

// 下面的goroutine 同样引用了外部变量a，由于下面的三个goroutine是异步的，所以输出是不缺定的，次例的输出都为mouse。

```golang
func main(){
	animals := []string{"dog", "cat", "mouse"}
	for _, a := range animals {
		go func() {
			fmt.Println(a)
		}()
	}
	select {}
}

// output:
// mouse
// mouse
// mouse
```

如果我们想要分别输出 "dog", "cat","mouse" 改怎么写呢？

通过参数的传递做一次值拷贝就可以了，注意要传递值不要传递指针。

```go
fumc main(){
	animals := []string{"dog", "cat", "mouse"}
	for _, a := range animals {
		go func(a string) {
			fmt.Println(a)
		}(a)
	}
	select {}
}
// output
// dog
// mouse
// cat
```

下面再举两个个例子，第一段代码匿名函数中的err使用的外部变量，第二段代码匿名函数中的err是内部定义的变量。
```go
func main() {
	_, err := fmt.Println("hello world")
	_ = err
	go func() {
		_, err = fmt.Println("hello goroutine")
		time.Sleep(time.Second*2)
		if err != nil {
			fmt.Println("err is not nil")
		}else{
			fmt.Println("err is nil")
		}
	}()
	time.Sleep(time.Second )
	err = errors.New("error")
	select {}
}

// output:
// err is not nil
```
```go
func main() {
	_, err := fmt.Println("hello world")
	_ = err
	go func() {
		_, err := strconv.Atoi("22")
		time.Sleep(time.Second*2)
		if err != nil {
			fmt.Println("err is not nil")
		}else{
			fmt.Println("err is nil")
		}
	}()
	time.Sleep(time.Second )
	err = errors.New("error")

	select {}
}

// output:
// err is nil
```