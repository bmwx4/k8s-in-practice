# Go程序 Debug
作为go语言开发者，debug 一个go 程序，都应该是每个工程师应该具备的基本能力，借此文介绍一下使用dlv来debug go程序的姿势；

### 安装调试工具
Devle 是一个基于源码级别的 golang 调试工具, ，支持多种调试方式，可以直接运行调试，或者 attach 到一个正在运行中的 golang 程序，进行调试。
线上golang服务出现问题时，Devle 是必不少的在线调试工具，如果使用 docker，也可以把Devle打进docker镜像里，调试代码。安装方式如下:
```bash
$ go get -u github.com/go-delve/delve/cmd/dlv
```
### dlv 进行调试Golang 程序
可以使用 dlv 直接在命令行调试一个go程序，查看 dlv 的用法:
```bash
Available Commands:
  attach      Attach to running process and begin debugging.
  connect     Connect to a headless debug server.
  core        Examine a core dump.
  debug       Compile and begin debugging main package in current directory, or the package specified.
  exec        Execute a precompiled binary, and begin a debug session.
  help        Help about any command
  run         Deprecated command. Use 'debug' instead.
  test        Compile test binary and begin debugging program.
  trace       Compile and begin tracing program.
  version     Prints version.
```
直接debug, 我们准备一段go的 http server 小程序:
```go
func main() {
	suffix := ".docker.xx"
	hostname := "a-bdocker-0docker.docker.xx"
	fmt.Println(strings.TrimSuffix(hostname, suffix))
	router := mux.NewRouter().StrictSlash(true)
	router.HandleFunc("/", Index)
	log.Fatal(http.ListenAndServe(":9090", router))
}

func Index(w http.ResponseWriter, r *http.Request) {
	var url = html.EscapeString(r.URL.Path)
	fmt.Fprintf(w, "Hello, %q", url)
}
```
以 debug 模式来运行 :
```bash
$ dlv debug main.go
Type 'help' for list of commands.
设置断点, 函数级别:
(dlv) b main.main
Breakpoint 3 set at 0x13468bb for main.main() ./main.go:22
(dlv) b main.Index
Breakpoint 2 set at 0x1346d78 for main.Index() ./main.go:33
设置断点, 文件的行级别:
(dlv) b main.go:25
Breakpoint 2 set at 0x1346908 for main.main() ./main.go:25
输入c , 让服务跑起来:
(dlv) c
> main.main() ./main.go:22 (hits goroutine(1):1 total:1) (PC: 0x13468bb)
=>  22:	func main() {
输入 n 回车，单步执行调试:
(dlv) n
> main.main() ./main.go:23 (PC: 0x13468d2)
=>  23:		suffix := ".docker.py"
继续输入n
=>  24:		hostname := "a-bdocker-0docker.docker.py"
输入 print（别名p）输出变量信息:
(dlv) p suffix
".docker.py"
输入 locals 打印所有的本地变量:
(dlv) locals
hostname = "a-bdocker-0docker.docker.py"
suffix = ".docker.py"
输入c，继续让程序运行，直到下一个断点:
(dlv) c
```

这是我们在另一个终端用curl 命令访问一下服务:

```bash
$ curl localhost:9090/
正常情况会卡住，因为我们开启了debug:
(dlv) c
> main.Index() ./main.go:33 (hits goroutine(35):1 total:1) (PC: 0x1346d78)
    28:		router.Methods("DELETE").Path("/todos").HandlerFunc(TodoIndexDelete)
    29:		router.HandleFunc("/", Index)
    30:		log.Fatal(http.ListenAndServe(":9090", router))
    31:	}
    32:
=>  33:	func Index(w http.ResponseWriter, r *http.Request) {
    34:		var url = html.EscapeString(r.URL.Path)
    35:		fmt.Fprintf(w, "Hello, %q", url)
    36:	}
    37:
    38:	func TodoIndexDelete(w http.ResponseWriter, r *http.Request) {
(dlv)
此时我们的 debug 这边已经走到了之前bp的位置了, 往下你可以继续单步调试了
输入 args 打印出所有的方法参数信息:
(dlv) args
r = ("*net/http.Request")(0xc00015e200)
w = net/http.ResponseWriter(*net/http.response) 0xc0000b7998
输出请求内容:
(dlv) p r
*net/http.Request {
	Method: "GET",
	URL: *net/url.URL {
		Scheme: "",
		Opaque: "",
		User: *net/url.Userinfo nil,
		Host: "",
		Path: "/",
		RawPath: "",
		ForceQuery: false,
		RawQuery: "",
		Fragment: "",}
此时哗哗的会把客户端请求内容都打印出来的;
清空断点:
(dlv) clearall
Breakpoint 1 cleared at 0x13468bb for main.main() ./main.go:22
Breakpoint 2 cleared at 0x1346908 for main.main() ./main.go:25
```

### 使用 dlv 附加到运行的golang服务进行调试
使用这种方式 debug 的时候，必须注意的是，编译时要带符号信息的debug版本
```bash
$ go build -gcflags="all=-N -l" main.go
```
### 使用 dlv 调试core文件
```bash
$ dlv core <executable> <core>
```

到此，就完成了使用 dlv debug 来go程序的热身，
