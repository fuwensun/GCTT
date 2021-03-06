# Go 语言中的方法（第一部）

Golang程序中定义的类型有与其相关的方法。让我们来看一个例子：

```
type T struct {
    name string
}
func (t T) PrintName() {
    fmt.Println(t.name)
}
func main() {
    t := T{name: "Michał"}
    t.PrintName()
}
```

正如您可能怀疑程序输出名字。方法是一个函数，拥有附加的，单独元素的参数列表，称之为接收器。它被放在一个方法名之前。接收器的类型决定了如何使用方法。

## 接收器

方法被绑定到接收器的基类。小例子解释的很好:

```
type T struct {
    name string
}
func (T) F() {}
func (*T) F() {}
```

上面的代码无法编译. 第一个 F 方法被绑到 T 上. 第二各方法被邦到 *T 上. 对于单个基类型，方法名必须唯一，所以编译将报错:

```
> go install github.com/mlowicki/lab && ./spec/bin/lab
# github.com/mlowicki/lab
spec/src/github.com/mlowicki/lab/lab.go:103: method redeclared: T.F
        method(T) func()
        method(*T) func()
```

如果基类是一个 struct 类型，那么字段名不能和方法名重复。

```
type T struct {
    M int
}
func (t T) M() {}
```

编译失败信息如下:

```
type T has both field and method named M
```

Go 的类型系统限制了哪些类型可以作为接收器. 它不能是接口类型或指针，因此它不可能为一个空接口（interface{}）定义方法来满足所有类型。 只允许使用类型名称，所以，类型字面值会导致编译错误：

```
func (v map[string]float64) M() {}
```

错误信息 `invalid receiver type map[string]float64 (map[string]float64 is an unnamed type).`

## 方法集

调用一个 T 类型的变量上的 m 方法, 方法 m 必须在与 T 关联的方法集上. 方法集中的一个成员是什么意思?

### 接口类型

接口类型的方法集是接口自己 — 为实现接口需要定义方法列表:

```
type I interface {
    F()
    G()
}
type T struct{}
func (T) F() {}
func (T) G() {}
func (T) H() {}
func main() {
    var i I = T{}
    i.F()
    i.G()
    i.H() // error: i.H undefined (type I has no field or method H)
}
```

上面的例子中，不允许调用 T 类型的变量 i 上的方法。仅有来自接口的接口类型值方法属于接口类型的方法集。（方法集是静态的，当分配不同类型的值时不会改变）。 要调用 H 方法，需要先使用类型断言:

```
t, ok := i.(T)
if !ok {
    log.Fatal("Type assertion failed")
}
t.H()
```

### 非接口类型

对于非接口类型 T，方法集由接收器为 T 的方法组成:

```
type T struct{}
func (T) F() {}
func (T) G() {}
type U struct{}
func (U) H() {}
func main() {
    t := T{}
    t.F()
    t.G()
    t.H() // error: t.H undefined (type T has no field or method H)
}
```

当处理 T 类型的指针时, 它的方法集由接收器为 T 或 *T 的方法组成:

```
type T struct{}
func (T) F() {}
func (T) G() {}
func (*T) H() {}
func main() {
    t := &T{}
    t.F()
    t.G()
    t.H()
}
```

为什么 T 的方法集没有 *T 类型接收器的方法，而 *T 的方法集却包含 T 类型接收器的方法呢? 令人惊讶的是，分配给 t 不是地址，但无指针结构值仍执行的很好:

```
t := T{}
t.F()
t.G()
t.H()
```

Go 规范描述了明确的情况:

>如果 x 是可寻址的，并且 x 的方法集包含 m ，x.M() 是 (&x).M() 的速记。

何时使用值还是指针接收器的建议放在了官方的[FAQ](https://golang.org/doc/faq#methods_on_values_or_pointers)中。

### 唯一性

类型 T 定义的方法集不能有俩个名字一样的方法。 有相同的方法名但不同类型的参数，在 Go 中是不可能的。 ( Go 中没有 [ad hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism) )。

>类型 T 定义的方法集由 T 实现.

## 打印方法集

Go 有 [reflect](https://golang.org/pkg/reflect/) 包，对于看类型的方法集很有用:

```
func PrintMethodSet(val interface{}) {
    t := reflect.TypeOf(val)
    fmt.Printf("Number of methods: %d\n", t.NumMethod())
    for i := 0; i < t.NumMethod(); i++ {
        m := t.Method(i)
        fmt.Printf("Method %s\n", m)
        fmt.Printf("\tName: %s\n", m.Name)
        fmt.Printf("\tPackage path: %s\n", m.PkgPath)
    }
}
```

从[Method](https://golang.org/pkg/reflect/#Value.Method) 返回值类型，方法是 [Method](https://golang.org/pkg/reflect/#Method) 类型.

值得一提的是，文档有个bug, 因为 [NumMethod](https://golang.org/pkg/reflect/#Value.NumMethod) 从方法集返回了方法的数量，但只有那些被导出的方法 (名字以大写字母开头)。 归档在 [#17686](https://github.com/golang/go/issues/17686).

## 引入类型方法

方法只能被定义在有类型创建的包里。

github.com/mlowicki/lib/lib.go:

```
package lib
type T struct{}
```

github.com/mlowicki/lab/lab.go:

```
package main
import (
    "fmt"
    . "github.com/mlowicki/lib"
)
func (T) F()  {}
func (*T) F() {}
[...]
```

这会引起构建错误 `cannot define new methods on non-local type lib.T`.

## 不使用参数

函数/方法定义不强制命名所有参数或接收者。如果不使用它们，则只能指定类型。 Name can be eventually introduced later on when it’ll actually needed:

```
type T struct {
    name string
}
func (t T) F(name string) {}
func (T) G(string) {}
```

G 的声明省略无用标识符。

----------------

via: https://medium.com/golangspec/methods-in-go-part-i-a4e575dff860

作者：[Michał Łowicki](https://medium.com/@mlowicki)
译者：[themoonbear](https://github.com/themoonbear)
校对：[polaris1119](https://github.com/polaris1119)

本文由 [GCTT](https://github.com/studygolang/GCTT) 原创编译，[Go 中文网](https://studygolang.com/) 荣誉推出