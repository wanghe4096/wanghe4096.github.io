---
title: 开发者零基础入门教程
---

# 学习内容大纲



## 计算机科学基础
### 命令行基础
- cd: change directory , 改变工作目录
- pwd: 查看当前所在的目录
- ls: list, 列出当前目录文件列表 
- open: 打开一个文件
- rm: remove 删除
- mv: move  
- mkdir: make directory 
- man: manual 
- touch: 创建一个文件 
- echo ： 回显

### 语言类
    - C 语言 : 偏向于系统底层的语言，常用来做底层开发，所谓底层开发是： 接近硬件的编程，比如操作系统，驱动软件，或者是对性能要求较为苛刻的场景
    - Java： 流行的，  较为常用的企业级应用开发语言， 或者是大数据相关领域较为常用的开发语言
    - Python: 人工智能，数据挖掘，以及科学计算
    - Go: 互联网公司常用

    
### 操作系统 
    - Unix / Linux 
    - Microsoft Window
    - MacOs : 基于FreeBSD Unix 操作系统深度定制的
    - iOS / Android
<!-- more -->

##### 理论
- 进程、线程
- 临界资源访问 
- 锁、信号量

### 网络基础
    - 计算机互联网络
    - 网络协议： 物理层、数据链路层、网络层、传输层、应用层, TCP/IP ： tcp, ip, udp, arp, darp . 
        - IP : Internal Protocol 网际传输协议
        - TCP: transfer control protocol 传输控制协议
        - UDP: user datagram protocol 用户数据报协议
        - http: 超文本传输协议
        - SMTP: 邮件发送协议
        - POP: 邮件接收协议

### 计算机体系结构 
    - 中央处理器
    - 内存储器
    - 外存储器
    - 输入设备
    - 输出设备

### 开发者工具
 #### IDE
    - Intelli IDEA 
    - Eclispe 

 #### 编辑器
    - visual studio code 
    - vim 
    - emacs 
    - sublime 
    - atom 


# Hello World 

```html
<html>
<body>
<h1> hello world </h1>
</body>
</html>
```

```PYTHON
print 'hello world'
```

```go
package main 
import (
    "fmt"
)
func main() {
    fmt.Println("hello world")
}

```

```c
#include <stdio.h> 
int main() 
{
    printf("hello world\n"); 
}
```

```js
    console.log("hello world");  
```



```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8 ">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<meta http-equiv="X-UA-Compatible" content="ie=edge">
	<title>Document</title>
	<style>
		body {
			background-color:red;
			color: white; 
		}
		
	</style>
	<script lang="javascript">
		function greet(){
			console.log("你好， 大璐!!!!")
			alert("hello")
			e = document.getElementsByTagName("body").
		}
	</script>
</head>
<body>
	<h1> hello world </h1>
	<p>
		The flutter-io.cn server is a provisional mirror for Flutter dependencies and packages maintained by GDG China. The Flutter team cannot guarantee long-term availability of this service. You’re free to use other mirrors if they become available. If you’re interested in setting up your own mirror in China, contact flutter-dev@googlegroups.com for assistance.
	</p>
	<button onclick="greet()">你好，大璐</button>
</body>
</html>
```
