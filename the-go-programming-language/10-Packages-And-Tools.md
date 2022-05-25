# Packages and tools

## Packages

关于packages 需要注意的是

1. package 是一种封装， 变量名， 函数名等等， 可以避免和其他 package 冲突。同样， package 的存在也设定了 package level 相关的外部包的访问权限
2. package name 只是用于外部import 的一个类似 ID 的功能
3. 可执行的package 必须是 main 
4. directory 里面有 _test 结尾go 文件， 会创建一个 **external test package**
5. yaml 里面会定义 包的version， 但是 import 的时候，是看不到这个version 的
6. 可以通过 alternative ，对引入的包取别名
7. go 中显示引入的包，如果没有使用的话 （精简的道理），会报错， blank import 可以避免， blank import  会执行包里面的init方法。 看下面的代码示例

```go
import (
  "database/mysql"
  _ "github.com/lib/pq" // enable support for Postgres
  _ "github.com/go-sql-driver/mysql" // enable support for MySQL
)

db, err = sql.Open("postgres", dbname) // OK
db, err = sql.Open("mysql", dbname) // OK
db, err = sql.Open("sqlite3", dbname) // returns error:
unknown driver "sqlite3"
```

## Go tools

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220525104926-go-tools.png)

1. go path 的设定， 实际上相当于是 workspace如果不设定go path （为当前项目路径） 的话， 根据包名打包肯定是会失败的， 因为默认只会去 go root 下面去找

```bash
package books_learning/gopl/ch10 is not in GOROOT (/Users/feyfree/Develop/SDK/go1.17/src/books_learning/gopl/ch10)
➜  ch09 git:(main) ✗ go build go_demo/books_learning/gopl/ch10
package go_demo/books_learning/gopl/ch10 is not in GOROOT (/Users/feyfree/Develop/SDK/go1.17/src/go_demo/books_learning/gopl/ch10)
```

2. go的打包支持跨平台

```go
$ go build gopl.io/ch10/cross
$ ./cross
darwin amd64
$ GOARCH=386 go build gopl.io/ch10/cross
$ ./cross
darwin 386
```

