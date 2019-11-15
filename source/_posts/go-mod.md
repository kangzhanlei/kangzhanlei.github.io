---
title: go mod 学习笔记
date: 2019-011-15 16:21:12
tags:
	- go
categories:
	- go
---



# 建立一个库

在 github 上申请一个仓库：https://github.com/kangzhanlei/gotest, 克隆到本地。然后建立目录结构如下：

* gotest
  * hello
    * hello.go

然后在 gotest 目录下，输入命令`go mod init github.com/kangzhanlei/gotest` ，自动生成`go.mod`文件。

文件内容为：

```go
module github.com/kangzhanlei/gotest

go 1.13

```



编写代码hello.go内容为:

```go
package hello

import "fmt"

func Hello(){
        fmt.Println("hello world")
}
```



完毕后，提交到 github 上

```bash
git add .
git commit -m "v1"
git push
```



到此为止，一个完整的库已经做完了。



# 使用库

在本地建立一个测试用的项目，取名为`kzl`，写一个 main 函数如下：

```go
package main

import "github.com/kangzhanlei/gotest/hello"

func main(){
        hello.Hello()
}
```



然后执行`go run main.go` ，可以看到如下的输出：

```bash
kzl@CebTechKzl kzl % go run main.go
go: finding github.com/kangzhanlei/gotest latest
go: downloading github.com/kangzhanlei/gotest v0.0.0-20191115083704-26e07495dead
go: extracting github.com/kangzhanlei/gotest v0.0.0-20191115083704-26e07495dead
hello world

```



从输出上来看，自动从 github 上下载了 gotest 这个依赖库，然后编译输出 helloworld。再观察看 go 的仓库

```bash
kzl@CebTechKzl kangzhanlei % pwd
/Users/kzl/go/pkg/mod/github.com/kangzhanlei
kzl@CebTechKzl kangzhanlei % ls
gotest@v0.0.0-20191115083704-26e07495dead
kzl@CebTechKzl kangzhanlei %

```

在仓库中已经有一个v0.0.0版本的库。





# 使用更新后的库

当前项目依赖的`gotest`这个库如果进行了更新，例如hello.go修改为：

```go

package hello

import "fmt"

func Hello(){
        fmt.Println("v2 hello world")
}
```



然后提交到 github 上之后，再次运行 main.go， 发现main 会自动下载最新的 gotest 的版本。输出 `v2 hello world` 。如果不想使用最新版本的 gotest，那么可以在 main 这个工程中，指定版本。如下：

```bash
kzl@CebTechKzl kangzhanlei % ll
total 0
drwxr-xr-x   5 kzl  staff   160B 11 15 16:47 .
drwxr-xr-x  79 kzl  staff   2.5K 11 15 15:11 ..
dr-x------   3 kzl  staff    96B 11 15 16:39 gotest@v0.0.0-20191115083704-26e07495dead
dr-x------   3 kzl  staff    96B 11 15 16:45 gotest@v0.0.0-20191115084455-0c668973981a
dr-x------   3 kzl  staff    96B 11 15 16:47 gotest@v0.0.0-20191115084604-c15e57a874df
```



目前有三个版本，我们选择一个版本来使用,在 kzl 工程下写命令

```bash
go mod init kzl	
```

自动生成了go.mod文件，修改为： 

```bash
module kzl

go 1.13

require github.com/kangzhanlei/gotest v0.0.0-20191115083704-26e07495dead
```

再次运行`go run main.go`，得到的输出如下：

```go
hello world
```



可以看到，强制使用了gotest 的第一个版本。





# 库的大版本升级

如果第三方库有较大版本的升级，可以直接使用库的版本号来进行依赖。例如 gotest 修改为：

```go
package hello

import "fmt"

func Hello(){
        fmt.Println("这是 gotest 的较大版本的升级 v2 版本")
}
```



然后给这个版本打一个 tag 标签，注意:

> 标签的样式必须是： major.minor.patch ，例如v1.0.5 表示大版本是 1，小版本是 0，补丁是 5



然后修改 gotest 的`go.mod`，将module 修改为：`module github.com/kangzhanlei/gotest/v2`

现在打个 v2 的标签如下：

```bash
kzl@CebTechKzl gotest % git add .
kzl@CebTechKzl gotest % git commit -m "upgrade"
[master a15764f] upgrade
 2 files changed, 4 insertions(+), 1 deletion(-)
 create mode 100644 go.mod
kzl@CebTechKzl gotest % git tag v2.0.0
kzl@CebTechKzl gotest % git push --tags
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (5/5), 471 bytes | 471.00 KiB/s, done.
Total 5 (delta 0), reused 0 (delta 0)
To https://github.com/kangzhanlei/gotest
 * [new tag]         v2.0.0 -> v2.0.0
```

到此为止，gotest 的库的 v2.0.0 版本制作完毕。下面用 kzl 工程来使用它。

使用方需要做的事情很简单，直接修改代码，增加版本号即可，例如：

```go 
package main

import "github.com/kangzhanlei/gotest/hello"
import kzl "github.com/kangzhanlei/gotest/v2/hello"

func main(){
        hello.Hello()
        kzl.Hello()
}
```



注意： 增加了 v2 的版本，为了区别旧版本，使用别名代替。注意一个工程中就可以引用两个不同版本的库函数了。

再次执行`go run main.go` ，得到输出如下： 

```bash

kzl@CebTechKzl kzl % go run main.go
go: finding github.com/kangzhanlei/gotest latest
go: finding github.com/kangzhanlei/gotest/v2 v2.0.0
go: downloading github.com/kangzhanlei/gotest/v2 v2.0.0
go: extracting github.com/kangzhanlei/gotest/v2 v2.0.0
hello world
这是 gotest 的较大版本的升级 v2 版本
```



可以看到，在当前工程中，确实使用到了 2 个版本的库函数。



最后来看一下，go 本地仓库里的目录结构：

```bash
kzl@CebTechKzl kangzhanlei % ll
total 0
drwxr-xr-x   6 kzl  staff   192B 11 15 17:02 .
drwxr-xr-x  79 kzl  staff   2.5K 11 15 15:11 ..
drwxr-xr-x   3 kzl  staff    96B 11 15 17:02 gotest
dr-x------   3 kzl  staff    96B 11 15 16:39 gotest@v0.0.0-20191115083704-26e07495dead
dr-x------   3 kzl  staff    96B 11 15 16:45 gotest@v0.0.0-20191115084455-0c668973981a
dr-x------   3 kzl  staff    96B 11 15 16:47 gotest@v0.0.0-20191115084604-c15e57a874df
```

在 gotest 目录下，有 v2.0.0 的版本：

```
kzl@CebTechKzl gotest % pwd
/Users/kzl/go/pkg/mod/github.com/kangzhanlei/gotest
kzl@CebTechKzl gotest % ll
total 0
drwxr-xr-x  3 kzl  staff    96B 11 15 17:02 .
drwxr-xr-x  6 kzl  staff   192B 11 15 17:02 ..
dr-x------  4 kzl  staff   128B 11 15 17:02 v2@v2.0.0
```



这就是 go mod 的原理。