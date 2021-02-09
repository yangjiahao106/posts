# jetBrains IED关于正则搜索替换的高级用法。

## 使用正则表达式捕获分组和反向引用分组。

当我们使用正则表达式去替换一些东西时，我们还想使用正则匹配到的一些内容中，就可以使用分组然后在替换结果中引用它。

### 举个栗子：
替换前：
```html
<new product="ij" category="105" title="Multiline search and replace in the current file"/>
<new product="ij" category="105" title="Improved search and replace in the current file"/>
<new product="ij" category="105" title="Regexp shows replacement preview"/>
```

替换后：
```
<new product="ij" category="105"/><title>Multiline search and replace in the current file</title>
<new product="ij" category="105"/><title>Improved search and replace in the current file</title>
<new product="ij" category="105"/><title>Regexp shows replacement preview</title>

```

#### 1.使用快捷键Ctrl + R调出替换工具栏，选择正则表达式模式
#### 2.在搜索栏内输入正则表达式： 
```text
\stitle="(.*)?"\s*(/>*)
```
#### 3. 在替换栏内输入替换结果，使用$加上数字反向引用正则中匹配的分组，如何确定分组编号有一个小技巧，就是数'('的个数，内容在哪个'('内,就是第几个分组。
```text
$2<title>$1</title>
```
#### 4. 点击REPLACE All 大功告成


## 转换字符大小写
    语法也很简单，一共有四种：
    
    \l changes a character to lowercase until the next character in the string. 
    For example, Bar becomes bar.
    
    \u changes a character to uppercase until the next character in the string. 
    For example, bar becomes Bar.
    
    \L changes characters to lowercase until the end of the literal string (\E). 
    For example, BAR becomes bar.
    
    \U changes characters to uppercase until the end of the literal string (\E). 
    For example, bar becomes BAR.
    
    翻译过来就是：

    \l 首字母变小型，例如 Bar -> bar
    \u 首字母变大小，例如 bar -> Bar
    \L 字符串变小写，例如 BAR -> bar
    \U 字符串变大写，例如 bar -> BAR
    
### 举个栗子
下划线格式，变小驼峰格式

搜索栏：

    _([a-z]+)
替换栏：

    \u$1

    
#### 推荐一个jetbrains插件CamelCase, 安装后使用快捷键SHIFT + ALT + U 可以切换CamelCase, camelCase, snake_case, SNAKE_CASE 四种风格。

## 多行搜索

最近查找BUG时搜索代码用到了多行搜索，顺便做个笔记。

### 还是举个栗子

```
package awesomeProject

import "fmt"

func main()  {
	update := map[string]interface{}{
		"hello":"world",
		"name": "tom",
		"holy":"shit",
	}
	
	update = map[string]interface{}{
		"hello":"world",
		"holy":"shit",
	}
	update["name"] = "marry"
	fmt.Println(update)
}

```

假如代码中定义了许多的名为updage的map现在要搜索update中有添加键为name的代码，正则表达式可以这样写

    (update.*=.*\{[^}]*?name[^}]*?\})|(update\["name"])
    
要搜索出第一个update就需要使用多行搜索。[^}] 可以匹配除了 } 的所有字符，包括空格换行制表等等，然后使用} 进行结束匹配，避免匹配过多内容。搜索第二个update就简单多了。