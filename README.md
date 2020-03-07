# Go语言环境配置与go mod
## 环境配置
### 1.GOROOT
   GOROOT其实就是golang的安装路径，也称为根目录，一般的默认途径为 /usr/go 或者 /usr/local/go

例如 我的电脑中GOROOT路径为 /usr/local/go
```bash
$ cd /usr/local/go
$ ls
AUTHORS		PATENTS		api		lib		src
CONTRIBUTING.md	README.md	bin		misc		test
CONTRIBUTORS	SECURITY.md	doc		pkg
LICENSE		VERSION		favicon.ico	robots.txt
```
### 2.GOPATH
  GOPATH环境变量表示go的工作目录，这个目录包含我们所有的源码和一些编译生成的文件。在这个目录中一般有3个文件夹：
```bash
goWorkSpace
   --bin //存放可执行文件
   --pkg //存放编译好的包文件，主要是*.a文件
   --src //源码路径，每个项目也是一个文件夹
```
几种命令的差别：
>go run : 编译并直接运行程序，不生成编译文件。

>go build : 用于测试编译包，主要检查是否有编译错误，不会产生结果文件，如果编译的是一个可执行文件的源码(即main包)，会在**执行目录下**生成可执行文件。

 > go install : 先编译导入的包文件，包文件编译完后再编译主程序，再将编译后的可执行文件放到bin目录下(`$`GOPATH/bin);如果编译的是某个依赖包，编译后的依赖放到pkg目录下(`$`GOPATH/pkg)。

>go get : git clone+go install (需要翻墙，一般直接clone)。

注意：
1.import后面的最后一个元素应该是路径，而非包名。

2.我们知道一个非main包在编译后会生成一个.a文件(除非go install安装到$GOPATH下，否则看不到)，用于后续可执行程序链接使用。当同时存在源码和.a文件的时候，编译器链接的是源码。也就是说，当别人要使用我们的源码时，我们要给的是源码而不是已经编译好的二进制文件。
### 3.GOBIN
GOBIN是程序生成的可执行文件的路径，可以为空，也可以自己设置。默认链接到$GOPATH/bin文件夹中。

#### 查看go环境：go env
```Go
$ go env
GO111MODULE="" //设置为on时，即开启go mod 模式，后面会讲
GOARCH="amd64" //64位的CPU
GOBIN="/Users/huanglingyun/go/bin" //编译器和链接器安装位置
GOCACHE="/Users/huanglingyun/Library/Caches/go-build"
GOENV="/Users/huanglingyun/Library/Application Support/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GONOPROXY=""
GONOSUMDB=""
GOOS="darwin"//目标机器操作系统
GOPATH="/Users/huanglingyun/go" //工作目录
GOPRIVATE=""
GOPROXY="https://goproxy.cn"
GOROOT="/usr/local/go" //根目录
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/darwin_amd64"
GCCGO="gccgo"
AR="ar"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=/var/folders/vl/kmnx_xg50tj36wvzrzkxbzr40000gn/T/go-build141219875=/tmp/go-build -gno-record-gcc-switches -fno-common"
```
或者打印某一个变量的值
```bash
$ go env GOPATH
/Users/huanglingyun/go
$ echo $GOPATH
/Users/huanglingyun/go
```
### 设置go环境变量
```bash
$ cd ~
$ ls -all //查看是否存在 .bash_profile文件
$ touch .bash_profile //若不存在该文件则手动创建
$ vim .bash_profile
```
 编辑.bash_profile，设置环境变量
```bash
  1 export GOPATH=/Users/huanglingyun/go
  2 
  3 export GOBIN=$GOPATH/bin
  4 
  5 export PATH=$PATH:$GOBIN:/usr/local/mysql/bin
```
保存后退出
```bash
$ source ~/.bash_profile //刷新一下
$ go env //检查一下设置后的环境
```
### Why GOPATH?
   golang的开发团队大部分是Google的成员，Google本身的开发方式就是用一个中央集中式的中心来管理软件包。使用，构建都从那里获取。因为外人无法访问google的中央，但是这种引用中央的开发模式被保留下来，抽象成我们本地的GOPATH。

这样的好处就是我们可以把一些开源包放在本地集中处理。处理依赖时比较方便，毕竟本地什么都有了。
# go module
Go语言的包管理经过了多种工具的演变，之前我们通过配置GOPATH来存放源代码进行包的管理，其实称不上包管理。Go Module 是Go官方为包依赖管理提供的一个解决方案。
### Why go mod ?
我们先来看一下之前GOPATH模式的一些缺点：

1.没法在GOPATH工作区以外的地方写代码

2.不能实现依赖的版本化管理

当我们在GOPATH之外创建项目并使用外部依赖时，运行时go会提醒我们找不到导入的包。那我们怎么解决这个问题呢？
我们可以使用一个特殊的文件，使用它指定仓库的规范名称。这个文件可以理解为是GOPATH的一个替代品，在它其中定义了仓库的规范名称，这样go就可以通过这个名称解析源码的导入包的位置。

这个特殊文件就是```go.mod```，这个文件中规范名称表示的新实体称为Moduel(模块)。
### 设置方式
因为没有将go mod设置为默认的包管理方式，增加了一个环境变量```GO111MODULE```来设置：
默认值是auto。
```go
GO111MODULE=off 不使用go mod，go 会从 GOPATH 和 vendor 文件夹寻找包。
GO111MODULE=on 使用go mod，go会忽略GOPATH设置，只根据 go.mod 下载依赖。
GO111MODULE=auto 在 $GOPATH/src 外面且根目录有 go.mod 文件时，开启模块支持。
```
直接在命令行中设置
```bash
go env -w GO111MODULE=on
```

## 一个简单的例子
下面我们通过一个完整的例子熟悉一下go mod的使用方法：

随意创建一个目录```example```,并用```go mod init```初始化
```bash
$ mkdir example
$ cd example
$ go mod init example//初始化
go: creating new go.mod: module example//初始化时生成一个go.mod文件
``` 
我们发现初始化时生成了一个```go.mod```文件，来看看这个文件里面有什么
```go
//go.mod
  1 module example //声明了module的名字
  2 
  3 go 1.13//golang的版本
```
创建```main.go```
```go
  1 package main
  2 
  3 import (
  4     "github.com/spf13/viper"//引用了一个本地没有的依赖
  5     "fmt"
  6 )
  7 
  8 func main() {
  9     viper.SetConfigFile("")
 10     fmt.Println("This is an example.")
 11 }
```
执行```go mod tidy```，我们看到在go mod模式下自动下载了项目所依赖的相关包
```go
$ go mod tidy
go: finding github.com/spf13/viper v1.3.1
go: downloading github.com/spf13/viper v1.3.1
go: finding github.com/hashicorp/hcl v1.0.0
go: downloading github.com/BurntSushi/toml v0.3.1
...

$ ls
go.mod	go.sum	main.go
```
增加了一个```go.sum```文件,文件记录的是依赖包和它自身的依赖包的版本记录
```go
//go.sum
github.com/BurntSushi/toml v0.3.1 h1:WXkYYl6Yr3qBf1K79EBnL4mak0OimBfB0XUf9Vl28OQ=
github.com/BurntSushi/toml v0.3.1/go.mod h1:xHWCNGjB5oqiDr8zfno3MHue2Ht5sIBksp03qcyfWMU=
...
```
我们再来看下```go.mod```文件
```go
  1 module example
  2 
  3 go 1.13
  4 
  5 require github.com/spf13/viper v1.6.2
```
go会检测main.go里的import语句，并尝试根据go.mod的依赖关系导入第三包。如果发现本地没有，就会从远程拉取，像go get一样。当go module下载了远程包后，同时会自动更新go.mod。

如上面追加了```require github.com/spf13/viper v1.6.2```。require后面跟**依赖包名**和**版本号**

### go.mod文件
go.mod文件还可以指定要替换和排除的版本，命令行会自动根据go.mod文件来维护需求声明中的版本。如果想获取更多的有关go.mod文件的介绍，可以使用命令```go help go.mod```。
文件的每行都有一条指令，由一个动作加上参数组成。例如：
```go
//go.mod
module my/example

require other/thing 	        v1.0.2
require new/thing 		v2.3.4
exclude old/thing 		v1.2.3
replace bad/thing 		v1.4.5 	=> good/thing v1.4.5
```
上面的三个动词```require```,```exclude```,```replace```分别表示：项目需要的依赖包及版本、排除某些包的特定版本、取代当前项目中的依赖包。

相同动作的命令可以放到括号中，例如：
```go
require (
    new/thing v2.3.4
    old/thing v1.2.3
)
```
注意：一些国内无法访问的依赖，可以用```replace```替换成其他库：
```go
replace golang.org/x/text v0.3.0 => github.com/golang/text v0.3.0
```
## go mod命令的使用
```
download    下载依赖包到本地缓存目录
edit        编辑go.mod文件
graph       打印模块依赖图
init        在当前文件夹下初始化一个新的module, 创建go.mod文件
tidy        增加丢失的module，去掉未使用的module
vendor      将依赖复制到vendor下
verify      校验依赖
why         解释为什么需要依赖
 ```
简单介绍一下几个重要的命令：

**```go mod init```**:上面的例子我们知道这个命令会初始化目录并创建一个```go.mod```文件，当然也可以手动创建```go.mod```文件，然后包含一些```module```声明，这样就比较麻烦。使用这条命令时，```go.mod```文件不能提前存在。

**```go mod download```**:可以下载所需的依赖，自动下载到```$GOPATH/pkg/mod```中，并且会把所有的包都打上版本号。其他项目也可以用缓存的module。

**```go mod tidy```**:它会添加确实的模块并移除不需要的模块，执行后生成go.sum文件。```go mod tidy```还可以用来解决依赖包冲突问题。

**```go mod vendor```**:此命令会将build阶段需要的所有依赖包放到主模块所在的vendor目录中，并且测试所有主模块的包。

**```go mod verify```**:这个命令会检查当前模块的依赖是否已经存储在本地下载的源代码缓存中，以及检查自从下载下来是否有修改。如果所有的模块都没有修改，那么会打印```all modules verified```，否则会打印变化的内容。








