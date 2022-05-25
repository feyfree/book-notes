# Tests

关于 GO test 需要注意的是

1. go test 是一个tool， 在package 目录下， 以 _test.go 结尾的 不会参与go build， 但是会参与 go test

2. 有三种 go test 

   1. test
   2. benchmark
   3. example

   test function 以Test 开头， benchmark 需要以 Benchmark 开头， example 以 Example 开头

   go test 会扫描 *_test.go 的文件， 并且创建一个临时的 main package 去调用，build， 执行，显示测试结果， 以及做好clean up的操作

3. go 提供了 testing 的包 [testing介绍](https://pkg.go.dev/testing)