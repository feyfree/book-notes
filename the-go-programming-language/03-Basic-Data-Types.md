# Basic Data Types

| 类型            | 备注     | 例子                                             |
| --------------- | -------- | ------------------------------------------------ |
| Basic types     | 基础类型 | 比如numbers， strings， booleans等等             |
| Aggregate types | 聚合类型 | 数组array， 结构体 struct， 或者是其他的组合类型 |
| Refrence types  | 引用类型 | 指针， 切片，maps， functions， channels         |
| Interface Types | 接口类型 |                                                  |

**关于引用类型**

they all refer to program variables or state in directly, so that the effect of an operation applied to one reference is observed by all copies of that reference

都是非直接引用程序中的变量或者状态， 所以对一个引用的修改， 会导致所有引用上面都能看到这个修改。

从表格中可以看到 数组是数组， 切片是切片， 实际是两种类型

## 1. Integers

无符号和有符号

int8 int16 int32 int64

对应

uint8 uint16 uint32 uint64



1. int 是最广泛使用的类型
2. rune 是 int32 的同义词， 字面value是通过 unicode 编码

2. byte 是 uint8 的同义词 ， 不会具体展示为 数字， 而是展示原始的数据
3. uintptr  底层编程会使用到， width 不确定， 但是能足够持有一个指针变量的位数



**注意类型之前的区别， 记住不是同一个类型**

1. 二进制数在内存中是以补码的形式存放的。
2. 补码首位是符号位，0表示此数为正数，1表示此数为负数。
3. 正数的补码、反码，都是其本身。
4. 负数的反码是：符号位为1，其余各位求反，但末位不加1 。
5. 负数的补码是：符号位不变，其余各位求反，末位加1 。
6. 所有的取反操作、加1、减1操作，都在有效位进行。



**位操作**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220510152216-go-bit-operations.png)

**Expression Initialization**

```go
```

