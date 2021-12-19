---
title: 重新学习Go语言
---

# 重新学习Go语言

## 写在前面

&emsp;&emsp;go语言可以说是我学习过的编程语言中最操蛋的，初见是在实习期间（2019年），当时`go vendor`还是主流，`go mod`项目还在起步，实习期第一个需求就是迁移一个老的`vendor`到`mod`，当时作为一名`javaer`，在`maven`和`gradle`的洗礼下，实在是无法理解go作为后期之秀，有这么多现成成熟的方案不去借鉴，结果搞出来的这么一堆垃圾玩意，着实是把我恶心了一把。

&emsp;&emsp;现在（2021年）的我已不再是当年的实习新手，因秋季换组的工作需要再次学习起了go。相比于实习期那时，go在生态上已经好了太多。虽然工具链还是那么难用:material-emoticon-sad:，至少感觉继续迭代上几年，可能会有天翻地覆的变化
## 聊聊Go的语言特性

&emsp;&emsp;首先需要说明的是——我并不是一名go吹，相反我极度讨厌go的某些特性，但同时也不得不承认go的的确确有着自己的优点。这里我只想聊聊我在实际工作开发中遇到的语言特性和问题，这些特性并不是

### 访问控制
&emsp;&emsp;封装是对现代编程语言必不可少的特性，为的是暴露给用户访问接口的同时隐藏实现的相关细节。不同的编程语言对封装的实现也不尽相同：

!!! note

    - Java提供了`private`,`protect`, `default`, `public`以及模块化的`export`关键字控制
    - Python并没有提供相关机制，仅依靠约定俗成：`_xx`, `__xx`前缀下划线告知使用者不要使用

&emsp;&emsp;Go为了节省关键字数量，使用命名首字母大小写控制，大写字母开头才会导出：
```go
package sample

type unexport struct {
    Field1 string
}

type Export struct {
    ExportField string
    hiddenField string
}
```
&emsp;&emsp;虽然这样做可以在阅读源码的时候可以通过判断首字母轻易判断是否是导出字段，但是在实际开发过程中会造成一些诡异的情况————当一个函数返回了一个未导出结构（返回一个未导出的结构是为了避免用户直接用结构定义构造数据）：
```go
package sample

func Fn() unexport {
    return unexport {
        Field1: "field"
    }
}

// ==================

package hello

import "sample"

e := sample.Fn()
e.Field1               // ok

var e sample.unexport  // not ok
e = sample.Fn()
```
&emsp;&emsp;可能有些用户认为并不会造成实际的使用影响——全都用`:=`就好了，但是现实的情况是，很多时候不得补考虑使用`var`——尤其是多返回值和重复定义的问题。

&emsp;&emsp;此外，早期的go是无法避免用户使用依赖库中其他其他实现模块的方法，如utils等，为了实现模块化的隔离，1.14版本新增了一项`import`补丁约束：[internal packages](https://golang.org/doc/go1.4#internalpackages)

## 方法（receiver method）

&emsp;&emsp;go在语言机制上提供了一种类似OOP的method的语法，为了封装实现细节，给`Person`添加**getter**方法:
```go
type Person struct {
    name string
    age  int
}

func (p Person) GetName() string {
    return p.name
}
func (p *Person) GetAge() int {
    return p.age
}
```
&emsp;&emsp;本质上这种receiver method的效果与直接传参数没有什么区别，可以看成是go的一种调用语法糖。
```go
func GetName(p Person) func()string {
    return func() {
        return p.name
    }
}

func GetAge(p *Person) func()int {
    return func() {
        return p.age
    }
}
```
&emsp;&emsp;比较有意思的是，不同于`Java`的方法调用，这种method的实现方式在方法中没有使用到recevier时是不会造成空指针异常，也就是说`nil`的方法调用也可能是安全有意义的：
```go
func (p *Person) Whatever() string {
    return "whatever"
}

var person = new(Person)  // person == nil // true
_ = person.Whatever()     // return "whatever", 无nil pointer deref error
```

&emsp;&emsp;开始学习的时候，我总是下意识的将这种机制与`kotlin`的`extension method`联系在一起，期望可以通过这种机制实现一些方便功能，如链式调用。但可惜的是，在go中对于定义receiver method限制非常强：**receiver method必须和receiver定义在同一个模块中**，这也就意味着你没有办法给外部接口定义增加新的receiver method，也就是说下面的方法是行不通的：
```go
func (i int) Hour() time.Duration {
    return time.Hour * i
}
// 3.Hour()
```
&emsp;&emsp;虽然golang中{==类型别名==}和{--方便的--}{==隐式类型转换==}算是缓解(?)这种情况带来的不便，但实际使用下来实现相当繁琐，希望后续语言的演化过程中可以解除这种限制：
```
type Int int64
func (i Int) Hour() time.Duration {
    return time.Duration(int64(time.Hour) * int64(i))
}

// Int(3).Hour()
```

## 3. interface

&emsp;&emsp;go语言中提供的唯一的抽象方式

```go
type Graph interface {
    AddNode(id int64)
    AddEdge(src, end int64, weight float64)
    Nodes() []int64
    Neighbors(src int64) []int64
}
```

```go
type graphImpl struct {}

func NewGraph() Graph {
    return &graphImpl{}
}

func (impl *graphImpl) AddNode(id int64) {
    panic("not implemented")
}
func (impl *graphImpl) AddEdge(src, end int64, weight float64) {
    panic("not implemented")
}
func (impl *graphImpl) Nodes() []int64 {
    panic("not implemented")
}
func (impl *graphImpl) Neighbors(src int64) []int64 {
    panic("not implemented")
}
```

&emsp;&emsp;由于Go没有明确的`implement`关系，为了能在编译期保证约束关系，在go项目中会经常看到类似下面的变量声明，如果改动接口导致实现缺失相关实现会在编译期直接报错：
```go
// 模拟implements语法
// type GraphImpl struct implements Graph {
// }
var _ Graph = (*graphImpl)(nil)
// 下面的方式也是类似
// var _ Graph = &GraphImpl{} 
```

&emsp;&emsp;这里提供了Graph的构造器的同时，利用访问控制不导出具体的实现————依赖接口而不是具体实现，在实际项目中是一个好的最佳实践。

## 反射

&emsp;&emsp;go语言不提倡使用反射机制，反射的实现性能也是堪忧，但是为了高级抽象，不得不使用反射实现相关功能。
   
    TODO