---
title: go 反射 reflect.md
date: 2021-01-31 19:11:50
tags:
- golang
---



## go 反射reflect 

go 的反射是通过reflect包来实现的

### reflect.Type
函数 reflect.TypeOf 接受任意的 interface{} 类型，并以 reflect.Type 形式返回其动态类型：


```go 

t := reflect.TypeOf(3)  // a reflect.Type
fmt.Println(t.String()) // "int"
fmt.Println(t)          // "int"


```
### reflect.Value

reflect.Value 可以装载任意类型的值。函数 reflect.ValueOf 接受任意的 interface{} 类型，并返回一个装载着其动态值的 reflect.Value。和 reflect.TypeOf 类似，reflect.ValueOf 返回的结果也是具体的类型，但是 reflect.Value 也可以持有一个接口值。
```go 
v := reflect.ValueOf(3) // a reflect.Value
fmt.Println(v)          // "3"
fmt.Printf("%v\n", v)   // "3"
fmt.Println(v.String()) // NOTE: "<int Value>"

t := v.Type()           // a reflect.Type
fmt.Println(t.String()) // "int"
```

reflect.ValueOf 的逆操作是 reflect.Value.Interface 方法。它返回一个 interface{} 类型，装载着与 reflect.Value 相同的具体值：
```go 
    
    v := reflect.ValueOf(3) // a reflect.Value
    x := v.Interface()      // an interface{}
    i := x.(int)            // an int
    fmt.Printf("%d\n", i)   // "3"

```

 reflect.Value 的 Kind 方法来替代之前的类型 switch。虽然还是有无穷多的类型，但是它们的 kinds 类型却是有限的：Bool、String 和 所有数字类型的基础类型；Array 和 Struct 对应的聚合类型；Chan、Func、Ptr、Slice 和 Map 对应的引用类型；interface 类型；还有表示空值的 Invalid 类型。（空的 reflect.Value 的 kind 即为 Invalid。）
 
 ```go
 // Any formats any value as a string.
func Any(value interface{}) string {
    return formatAtom(reflect.ValueOf(value))
}

// formatAtom formats a value without inspecting its internal structure.
func formatAtom(v reflect.Value) string {
    switch v.Kind() {
    case reflect.Invalid:
        return "invalid"
    case reflect.Int, reflect.Int8, reflect.Int16,
        reflect.Int32, reflect.Int64:
        return strconv.FormatInt(v.Int(), 10)
    case reflect.Uint, reflect.Uint8, reflect.Uint16,
        reflect.Uint32, reflect.Uint64, reflect.Uintptr:
        return strconv.FormatUint(v.Uint(), 10)
    // ...floating-point and complex cases omitted for brevity...
    case reflect.Bool:
        return strconv.FormatBool(v.Bool())
    case reflect.String:
        return strconv.Quote(v.String())
    case reflect.Chan, reflect.Func, reflect.Ptr, reflect.Slice, reflect.Map:
        return v.Type().String() + " 0x" +
            strconv.FormatUint(uint64(v.Pointer()), 16)
    default: // reflect.Array, reflect.Struct, reflect.Interface
        return v.Type().String() + " value"
    }
}
 ```

### 通过reflect.Value 修改值

有一些reflect.Values是可取地址的；其它一些则不可以。

```go 
x := 2                   // value   type    variable?
a := reflect.ValueOf(2)  // 2       int     no
b := reflect.ValueOf(x)  // 2       int     no
c := reflect.ValueOf(&x) // &x      *int    no
d := c.Elem()            // 2       int     yes (x)

```
实际上，所有通过reflect.ValueOf(x)返回的reflect.Value都是不可取地址的。但是对于d，它是c的解引用方式生成的，指向另一个变量，因此是可取地址的。我们可以通过调用reflect.ValueOf(&x).Elem()，来获取任意变量x对应的可取地址的Value。

我们可以通过调用reflect.Value的CanAddr方法来判断其是否可以被取地：

```go
fmt.Println(a.CanAddr()) // "false"
fmt.Println(b.CanAddr()) // "false"
fmt.Println(c.CanAddr()) // "false"
fmt.Println(d.CanAddr()) // "true"

```

slice的索引表达式e[i]将隐式地包含一个指针，它就是可取地址的，即使开始的e表达式不支持也没有关系。以此类推，reflect.ValueOf(e).Index(i)对应的值也是可取地址的，即使原始的reflect.ValueOf(e)不支持也没有关系。

```go
s := []int{1,2,3}
v = reflect.ValueOf(s)

fmt.Println(v.CanAddr())                // false
fmt.Println(v.Index(0).CanAddr())       // true
reflect.ValueOf(s).Index(0).SetInt(0)
fmt.Println(s)                          // [0, 2, 3]
```

修改变量

```go

x := 2
d := reflect.ValueOf(&x).Elem()   // d refers to the variable x
// 方法一
px := d.Addr().Interface().(*int) // px := &x
*px = 3                           // x = 3
fmt.Println(x)                    // "3"

// 方法二
d.Set(reflect.ValueOf(4))
fmt.Println(x) // "4"
// 方法三
d.SetInt(5)
fmt.Println(x) // "5"

```

注意： 要确保该变量可以接受对应的值；

对于一个引用interface{}类型的reflect.Value调用SetInt会导致panic异常，即使那个interface{}变量对于整数类型也不行。

``` go 
x := 1
rx := reflect.ValueOf(&x).Elem()
rx.SetInt(2)                     // OK, x = 2
rx.Set(reflect.ValueOf(3))       // OK, x = 3
rx.SetString("hello")            // panic: string is not assignable to int
rx.Set(reflect.ValueOf("hello")) // panic: string is not assignable to int

var y interface{}
ry := reflect.ValueOf(&y).Elem()
ry.SetInt(2)                     // panic: SetInt called on interface Value
ry.Set(reflect.ValueOf(3))       // OK, y = int(3)
ry.SetString("hello")            // panic: SetString called on interface Value
ry.Set(reflect.ValueOf("hello")) // OK, y = "hello"

```

通过反射机制可以读取结构体中未导出的成员， 但不可修改

```go
stdout := reflect.ValueOf(os.Stdout).Elem() // *os.Stdout, an os.File var
fmt.Println(stdout.Type())                  // "os.File"
fd := stdout.FieldByName("fd")
fmt.Println(fd.Int()) // "1"
fd.SetInt(2)          // panic: unexported field
fmt.Println(fd.CanAddr(), fd.CanSet()) // "true false"

```

应用：下面代码表示如果传入的结构体中有ImageUrl字段则加上一个后缀

```go

func compressImage(req interface{}) {
	modelType := reflect.TypeOf(req).Elem()
	modelValue := reflect.ValueOf(req).Elem()
	_, ok = modelType.FieldByName("ImageUrl")
	if ok {
		value := modelValue.FieldByName("ImageUrl")
		imageUrl := value.String()
		if strings.HasSuffix(imageUrl, ".jpg"){
			value.SetString(imageUrl + "!compress")
		}
	}
}
```

## 参考 
[通过reflect.Value修改值
](https://books.studygolang.com/gopl-zh/ch12/ch12-05.html)