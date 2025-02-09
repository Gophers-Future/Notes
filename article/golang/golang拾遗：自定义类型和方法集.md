golang拾遗主要是用来记录一些遗忘了的、平时从没注意过的golang相关知识。

很久没更新了，我们先以一个谜题开头练练手：

```golang
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

type MyTime time.Time

func main() {
    myTime := MyTime(time.Now()) // 假设获得的时间是 2022年7月20日20：30：00，时区UTC+8
    res, err := json.Marshal(myTime)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(res))
}
```

请问上述代码会输出什么：

1. 编译错误
2. 运行时panic
3. {}
4. "2022-07-20T20:30:00.135693011+08:00"
 
很多人一定会选4吧，然而答案是3：

```bash
$ go run customize.go

{}
```

是不是很意外，`MyTime`就是`time.Time`，理论上应该也实现了`json.Marshaler`，为什么输出的是空的呢？

实际上这是最近某个群友遇到的问题，乍一看像是golang的bug，但其实还是没掌握语言的基本规则。

在深入下去之前，我们先问自己两个问题：

1. MyTime 真的是 Time 类型吗？
2. MyTime 真的实现了 `json.Marshaler` 吗？

对于问题1，只需要引用spec里的说明即可：

> A named type is always different from any other type.

<https://go.dev/ref/spec#Type_identity>

意思是说，只要是type定义出来的类型，都是不同的（type alias除外），即使他们的underlying type是一样的，也是两个不同的类型。

那么问题1的答案就知道了，显然`MyTime`不是`time.Time`。

既然MyTime不是Time，那它是否能用Time类型的method呢？毕竟MyTime的基底类型是Time呀。我们写段代码验证下：

```golang
package main

import (
    "fmt"
    "time"
)

type MyTime time.Time

func main() {
    myTime := MyTime(time.Now()) // 假设获得的时间是 2022年7月20日20：30：00，时区UTC+8
    res, err := myTime.MarsharlJSON()
    if err != nil {
            panic(err)
    }
    fmt.Println(string(res))
}
```

运行结果：

```bash
# command-line-arguments
./checkoutit.go:12:24: myTime.MarsharlJSON undefined (type MyTime has no field or method MarsharlJSON)
```

现在问题2也有答案了：`MyTime`没有实现`json.Marshaler`。

那么对于一个没有实现`json.Marshaler`的类型，json是怎么序列化的呢？这里就不卖关子了，文档里有写，对于没实现`Marshaler`的类型，默认的流程使用反射获取所有非export的字段，然后依次序列化，我们再看看time的结构：

```golang
type Time struct {
        // wall and ext encode the wall time seconds, wall time nanoseconds,
        // and optional monotonic clock reading in nanoseconds.
        //
        // From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
        // a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
        // The nanoseconds field is in the range [0, 999999999].
        // If the hasMonotonic bit is 0, then the 33-bit field must be zero
        // and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
        // If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
        // unsigned wall seconds since Jan 1 year 1885, and ext holds a
        // signed 64-bit monotonic clock reading, nanoseconds since process start.
        wall uint64
        ext  int64

        // loc specifies the Location that should be used to
        // determine the minute, hour, month, day, and year
        // that correspond to this Time.
        // The nil location means UTC.
        // All UTC times are represented with loc==nil, never loc==&utcLoc.
        loc *Location
}
```

里面都是非公开字段，所以直接序列化后整个结果就是`{}`。当然，Time类型自己重新实现了`json.Marshaler`，所以可以正常序列化成我们期望的值。

而我们的MyTime没有实现整个接口，所以走了默认的序列化流程。

所以我们可以得出一个重要的结论：**从某个类型A派生出的类型B，B并不能获得A的方法集中的任何一个**。

想要B拥有A的所有方法也不是不行，但得和`type B A`这样的形式说再见了。

方法一是使用type alias：

```diff
- type MyTime time.Time
+ type MyTime = time.Time

func main() {
-   myTime := MyTime(time.Now()) // 假设获得的时间是 2022年7月20日20：30：00，时区UTC+8
+   var myTime MyTime = time.Now() // 假设获得的时间是 2022年7月20日20：30：00，时区UTC+8
    res, err := json.Marshal(myTime)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(res))
}
```

类型别名自如其名，就是创建了一个类型A的别名而没有定义任何新类型（注意那两行改动）。现在MyTime就是Time了，自然也可以直接利用Time的MarshalJSON。

方法二，使用内嵌类型：

```diff
- type MyTime time.Time
+ type MyTime struct {
+     time.Time
+ }

func main() {
-   myTime := MyTime(time.Now()) // 假设获得的时间是 2022年7月20日20：30：00，时区UTC+8
+   myTime := MyTime{time.Now}
    res, err := myTime.MarsharlJSON()
    if err != nil {
            panic(err)
    }
    fmt.Println(string(res))
}
```

通过将Time嵌入MyTime，MyTime也可以获得Time类型的方法集。更具体的可以看我之前写的另一篇文章：[golang拾遗：嵌入类型](golang拾遗：嵌入类型.md)

如果我实在需要派生出一种新的类型呢，通常在我们写一个通用模块的时候需要隐藏实现的细节，所以想要对原始类型进行一定的包装，这时该怎么办呢？

实际上我们可以让MyTime重新实现`json.Marshaler`：

```golang
type MyTime time.Time

func (m MyTime) MarshalJSON() ([]byte, error) {
    // 我图方便就直接复用Time的了
    return time.Time(m).MarshalJSON()
}

func main() {
    myTime := MyTime(time.Now()) // 假设获得的时间是 2022年7月20日20：30：00，时区UTC+8
    res, err := myTime.MarsharlJSON()
    if err != nil {
            panic(err)
    }
    fmt.Println(string(res))
}
```

这么做看上去违反了DRY原则，其实未必，这里只是示例写的烂而已，真实场景下往往对派生出来的自定义类型进行一些定制，因此序列化函数里会有额外的一些操作，这样就和DRY不冲突了。

不管哪一种方案，都可以解决问题，根据自己的实际需求做选择即可。

## 总结

总结一下，一个派生自A的自定义类型B，它的方法集中的方法只有两个来源：

- 直接定义在B上的那些方法
- 作为嵌入类型包含在B里的其他类型的方法

而A的方法是不存在在B中的。

如果是从一个匿名类型派生的自定义类型B（`type B struct {a, b int}`），那么B的方法集中的方法只有一个来源:

- 直接定义在B上的那些方法

还有最重要的，**如果两个类型名字不同，即使它们的结构完全相同，也是两个不同的类型**。

这些边边角角的知识很容易被遗忘，但还是有机会在工作中遇到的，记牢了可以省很多事。
