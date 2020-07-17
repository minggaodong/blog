# golang 编译和测试总结

## 编译
### go build
golang 中使用 go build 命令编译代码，编译时会搜索当前目录下的所有 .go 文件，并在当前目录下生成与目录同名的可执行文件。

golang 编译时会以源码形式编译所用 import 导入的依赖包。

#### 编译参数
```
go build -gcflags "-N -l -u"
```
参数说明
- -N：禁用优化
- -l：禁用函数内联
- -u：禁用unsafe代码
- -m：输出优化信息
- -S：输出汇报代码

#### 链接参数
```
go build -ldflags "-X main.VERSION=1.0.0 -X 'main.BUILD_TIME=`date`'"
```
参数说明
- -w：禁用调试信息，但不包括符号表；此时无法使用gdb调试
- -s：禁用符号表
- -X：修改程序中string类型的值，如上面例子中会修改main包中的VESION和BUILD_TIME两个字符串的值。

#### 竞态检测
```
go build -race 
```
设置 -race 标志可以开启竞态检测器，程序运行结束后，会将多个 goroutine 竞争访问的变量信息打印出来。

### GOPATH
GOPATH 是 golang 的一个环境变量，代表 golang 项目和依赖包的工作目录。

golang 通过 go get 或者 go build 安装查找依赖包的时候，都基于 GOPATH 指定的路径，GOPATH 路径可以有多个。

GOPATH 约定了三个子目录
- src: 存放源代码（比如：.go .c .h .s等）
- pkg: 编译后生成的文件（比如.a）
- bin: 编译后生成的可执行文件。

### go mod
go mod 是官方在 Go1.11 版本之后推出的版本管理工具，并且从 Go1.13 版本开始，成为Go语言默认的依赖管理工具。

go mod 编译的优点
- 项目存放目录可以随意指定，不必放在 GoPath 指定的 src 目录下。
- 执行build时，可以自动解决包依赖，会自动下载对应版本的依赖包，不需要手动 go get 安装。

#### 配置代理
go mod 下编译会自动下载第三方依赖包，很多国外网站因为被墙，可以设置go proxy代理
```
export GOPROXY=https://mirrors.aliyun.com/goproxy/
```

#### go mod 命令
##### go mod init
```
go mod init github.com/gin-gonic/gin
```
在项目目录下执行 go mod init xxx(module name)，会生成一个 go.mod 文件，用来保存 module 名称，版本以及依赖库。

module 名称使用 github 下的项目全路径，内部 import 其他依赖包时也要使用全路径。 

go mod 初始化完成后，执行 go test 或者 go build，会在 go.mod 内生成 requeire 信息，并下载对应的依赖包保存到 $GOPATH/pkt/mod 目录下。

require 信息的版本号可以手动修改，并重新编译
```
require (
	github.com/gin-contrib/sse v0.1.0
	github.com/go-playground/validator/v10 v10.2.0
	github.com/golang/protobuf v1.3.3
	github.com/json-iterator/go v1.1.9
	github.com/mattn/go-isatty v0.0.12
	github.com/stretchr/testify v1.4.0
	github.com/ugorji/go/codec v1.1.7
	gopkg.in/yaml.v2 v2.2.8
)
```

项目引用的某些库可能因为被墙无法访问，可以通过 replace 替换为 github 可访问的库，也可以修改成本地路径
```
replace (
    golang.org/x/text v0.3.0 => github.com/golang/text v0.3.0
    golang.org/x/text1 v0.3.0 => ./test1
)
```

##### go mod tidy
```
go mod tidy
```
代码中删除某些依赖包后，执行下面命令，可以更新go,mod中的依赖关系

##### go mod vendor
```
go mod vendor
go build -mod=vendor
```
该命令会将依赖包复制到本地 vendor 目录下，go build 的时候指定 vendor 模式进行构建。
由于外网访问的限制，很多情况下还是需要vendor目录存在的，并将vendor目录中的包一并提交到代码库中。

##### go clean -modcache
```
go clean -modcache
```

该命令会删除本地依赖包的缓存

## 测试
### go vet
go vet 是一个检查源码中静态错误的工具，可以检测出任何可疑，异常或者无用的代码。

go vet 是 go tool vet 的简单封装，go vet 只能检测当前目录下的 go 源文件，不能递归子目录。

### 单元测试
golang 的单元测试都是基于标准库 testing 进行的，编写单元测试时要遵循下面规则。

#### 用例规则
- 测试文件和源码文件放在一起，测试文件以 "_test.go" 结尾。
- 所有测试函数必须 "Test" 开头，Testxxx 或者 Test_xxx.xxx，首字母不能为小写。
- 调用 Error(), Errorf(), Fatal()，标识测试未通过，可以用Logf()输出调试信息。

```
func TestIdxInout(t *testing.T) {
	// 生成Context
	ctx, err := GetContext()
	if err != nil {
		t.Errorf("get context error: %s\n", err.Error())
	} else {
		t.Logf("%+v", ctx)
	}
}
```

#### 执行单元测试

```
// 测试整个包
go test -v xxx

// 测试1个文件
go test -v xxx_test.go xxx.go

// 测试1个函数
go test -v xxx_test.go xxx.go -test.run Test_Hello

// 传递参数
go test -v -args "xxxx"
```

执行 go test -v 命令运行单元测试用例，相同文件内按照函数定义的顺序执行，加 -v 参数可以显示执行成功的case。

### 基准测试
基准测试主要是通过测试CPU和内存的效率问题，来评估被测试代码的性能，进而找到更好的解决方案。

#### 用例规则
- 基准测试的函数名须以 Benchmark 开头
- 参数须为 *testing.B
- 循环次数由 b.N 参数指定，由系统根据测试时间自动生成，不用用户设定。

```
func BenchmarkSprintf(b *testing.B) {
	num := 10
	b.ResetTimer() // 重置计时器，这样可以避免for循环之前的初始化代码的干扰
	for i := 0; i < b.N; i++ {
		fmt.Sprintf("%d",num)
	}
}
```

#### 执行基准测试
```
go test -bench=. -benchtime=5s -run=none 
```
使用 -bench=. 运行所有的基准测试，使用 -run=none 可以忽略掉单元测试。

测试时间默认是1秒，可以通过-benchtime指定。

测试结果
```
// Benchmark 名字 - CPU     循环次数          平均每次执行时间 
BenchmarkSprintf-8      50000000               109 ns/op
PASS
//  哪个目录下执行go test         累计耗时
ok      flysnow.org/hello       5.628s
```
