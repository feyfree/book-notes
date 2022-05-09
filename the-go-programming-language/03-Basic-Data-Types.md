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