---
title: go如何等待所有的goroutine结束
date: 2019-06-02 16:58:40
tags:
---

golang中当main函数结束后，整个程序就退出了，不会等待其他的携程执行完毕，那么如何等待所有的goroutine结束？下面介绍两种方法。 

### 1.使用cnannel实现
```go
func main() {
	done := make(chan bool)
	urls := []string{
		"http://www.reddit.com/r/aww.json",
		"http://www.reddit.com/r/funny.json",
		"http://www.reddit.com/r/programming.json",
	}

	for _, url := range urls {
		go func(url string) {
			defer func() {
				done <- true
			}()
			fmt.Println(url)
			res, err := http.Get(url)
			if err != nil {
				log.Fatal(err)
				return
			}
			defer res.Body.Close()
			body, err := ioutil.ReadAll(res.Body)
			if err != nil {
				log.Fatal(err)
			} else {
				fmt.Println(string(body))
			}
		}(url)
	}

	for i:= 0 ; i< len(urls); i++{
		<-done
	}
}

```

### 2.使用sync包的 WaitGroup

```go
func main() {
	urls := []string{
		"http://www.reddit.com/r/aww.json",
		"http://www.reddit.com/r/funny.json",
		"http://www.reddit.com/r/programming.json",
	}
	
	wg := sync.WaitGroup{}
	
	wg.Add(len(urls))
	for _, url := range urls {
		go func(url string) {
			defer wg.Done()
			res, err := http.Get(url)
			if err != nil {
				log.Fatal(err)
				return
			}
			defer res.Body.Close()
			body, err := ioutil.ReadAll(res.Body)
			if err != nil {
				log.Fatal(err)
			} else {
				fmt.Println(string(body))
			}
		}(url)
	}

	wg.Wait()
}
```