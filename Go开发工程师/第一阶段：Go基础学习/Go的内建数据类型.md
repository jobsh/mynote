Go 语言基础学习

## Go的内建数据类型

* bool	String

* (u)int	(u)int8	(u)int16	(u)int32	(u)int64	uintptr
* byte 	rune
* float32      float(64)        complex64      complex128

注意：1. go中没有像java中的double、long数据类型 ； 2. uintptr   ptr代表的是指针，Go语言中的指针要比C中的要好用多了；3. rune相当于java中的char，但又不一样，java中的char只能有1byte，而我们知道一个汉字至少两个字节才能表示，而这里的rune是32位的，就是为了更好的兼容更广的语言和编码格式。4. complex为复数类型，实部和虚部各占complex位数的一半。

* Go只能强制类型转换，不存在隐式转换。

## 常量

关键字：const

Go语言中常量不要大写

const定义的常量数值可以作为各种类型使用

## 特殊的常量——枚举

Go中没有枚举的特殊关键字 ，而是用一组const来表示

```go
func enums() {
    const(
    	cpp = 0
    	java = 1
    	python = 2 
    	golang = 3
    )
    fmt.Println(cpp, java, python, golang)
}
```

以上的const中的定义也可以如下表示

```
 const(
    	cpp = iota  // 表示从0开始自增
    	java
    	python
    	golang 
    )
```

## if

if 后面没有括号

if后面可以跟多个赋值语句，可以赋值完再去判断值

if条件中的赋值的变量的作用域就在if语句中

## switch

Go中会在csse结束后自动break，如果想要继续判断下面的case需要使用fallthrough

## for

for 后面不能加（）

初始表达式，结束条件，递增表达式都可省略

for {  }  —— 死循环    Go 	没有 while

## 结构体和方法

### 定义：

```go
type treenode struct {
	value int       // 后面直接换行，不用加，
  	left, right *treenode
}
```

### ps：函数补充：可以返回局部变量的地址

```
func creatNode(value int) *treenode {
   // 可以返回局部变量的地址
   return &treenode{value: value}
}
```

可以使用new（struct_name）来创建struct。

### 创建struct 函数

1. 函数依然为值传递
2. 只有使用指针才可以改变结构体的内容
3. nil指针也可以调用struct的方法
4. 是否需要nil判空，依据情况而定。

### 值接收者VS指针接收者

1. 要改变内容必须指针接收

2. 结构过大推荐指针接收

3. 一致性规范：如果有指针接收者，最好都是指针接收者。为了代码的易读性

4. 值接收者是GO特有，指针解释其他语言都有（比如C++、Java的引用、Python的self）

5. 值/指针接收者均可接受值/指针，编译器会做出判断是值还是指针。如下：

   ```go
   func (node *treenode) bianli() {
   	if node == nil {
   		return
   	}
   	node.left.bianli()
   	print(node.value)
   	node.right.bianli()
   }
   
   root := treenode{1,nil,nil}
   root.left = &treenode{2,nil, nil}
   root.left.right = &treenode{3, nil, nil}
   root.right = &treenode{4, nil, nil}
   root.bianli()
   ```

   root是一个值，但是仍然可以调用bianli()，而bianli方法要求的是一个treenode指针。

