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

8. 

