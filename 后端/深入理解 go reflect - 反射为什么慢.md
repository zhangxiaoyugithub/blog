> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7186859098661453884)

go 的性能测试
--------

在开始之前，有必要先了解一下 go 的性能测试。在 go 里面进行性能测试很简单，只需要在测试函数前面加上 `Benchmark` 前缀， 然后在函数体里面使用 `b.N` 来进行循环，就可以得到每次循环的耗时。如下面这个例子：

```
func BenchmarkNew(b *testing.B) {
   b.ReportAllocs()
   for i := 0; i < b.N; i++ {
      New()
   }
}
复制代码
```

我们可以使用命令 `go test -bench=. reflect_test.go` 来运行这个测试函数，又或者如果使用 goland 的话，直接点击运行按钮就可以了。

说明：

*   在 `*_test.go` 文件中 `Benchmark*` 前缀函数是性能测试函数，它的参数是 `*testing.B` 类型。
*   `b.ReportAllocs()`：报告内存分配次数，这是一个非常重要的指标，因为**内存分配相比单纯的 CPU 计算是比较耗时的操作**。在性能测试中，我们需要关注内存分配次数，以及每次内存分配的大小。
*   `b.N`：是一个循环次数，每次循环都会执行 `New()` 函数，然后记录下来每次循环的耗时。

> go 里面很多优化都致力于减少内存分配，减少内存分配很多情况下都可以提高性能。

输出：

```
BenchmarkNew-20    1000000000    0.1286 ns/op   0 B/op   0 allocs/op
复制代码
```

输出说明：

*   `BenchmarkNew-20`：`BenchmarkNew` 是测试函数名，`-20` 是 CPU 核数。
*   `1000000000`：循环次数。
*   `0.1286 ns/op`：每次循环的耗时，单位是纳秒。这里表示每次循环耗时 0.1286 纳秒。
*   `0 B/op`：每次循环内存分配的大小，单位是字节。这里表示每次循环没有分配内存。
*   `0 allocs/op`：每次循环内存分配的次数。这里表示每次循环没有分配内存。

go 反射慢的原因
---------

> 动态语言的灵活性是以牺牲性能为代价的，go 语言也不例外，go 的 interface{} 提供了一定的灵活性，但是处理 interface{} 的时候就要有一些性能上的损耗了。

我们都知道，go 是一门静态语言，这意味着我们在编译的时候就知道了所有的类型，而不是在运行时才知道类型。 但是 go 里面有一个 `interface{}` 类型，它可以表示任意类型，这就意味着我们可以在运行时才知道类型。 但本质上，`interface{}` 类型还是静态类型，只不过它的类型和值是动态的。 在 `interface{}` 类型里面，存储了两个指针，一个指向类型信息，一个指向值信息。具体可参考[《go interface 设计与实现》](https://juejin.cn/post/7173965896656879630 "https://juejin.cn/post/7173965896656879630")。

### go interface{} 带来的灵活性

有了 `interface{}` 类型，让 go 也拥有了动态语言的特性，比如，定义一个函数，它的参数是 `interface{}` 类型， 那么我们就可以传入任意类型的值给这个函数。比如下面这个函数（做任意整型的加法，返回 `int64` 类型）：

```
func convert(i interface{}) int64 {
   typ := reflect.TypeOf(i)
   switch typ.Kind() {
   case reflect.Int:
      return int64(i.(int))
   case reflect.Int8:
      return int64(i.(int8))
   case reflect.Int16:
      return int64(i.(int16))
   case reflect.Int32:
      return int64(i.(int32))
   case reflect.Int64:
      return i.(int64)
   default:
      panic("not support")
   }
}

func add(a, b interface{}) int64 {
   return convert(a) + convert(b)
}
复制代码
```

说明：

*   `convert()` 函数：将 `interface{}` 类型转换为 `int64` 类型。对于非整型的类型，会 panic。（当然不是很严谨，还没涵盖 `uint*` 类型）
*   `add()` 函数：做任意整型的加法，返回 `int64` 类型。

相比之下，如果是确定的类型，我们根本不需要判断类型，直接相加就可以了：

```
func add1(a, b int64) int64 {
   return a + b
}
复制代码
```

我们可以通过以下的 benchmark 来对比一下：

```
func BenchmarkAdd(b *testing.B) {
   b.ReportAllocs()
   for i := 0; i < b.N; i++ {
      add(1, 2)
   }
}

func BenchmarkAdd1(b *testing.B) {
   b.ReportAllocs()
   for i := 0; i < b.N; i++ {
      add1(1, 2)
   }
}
复制代码
```

结果：

```
BenchmarkAdd-12         179697526                6.667 ns/op           0 B/op          0 allocs/op
BenchmarkAdd1-12        1000000000               0.2353 ns/op          0 B/op          0 allocs/op
复制代码
```

我们可以看到非常明显的性能差距，`add()` 要比 `add1()` 慢了非常多，而且这还只是做了一些简单的类型判断及类型转换的情况下。

### go 灵活性的代价（慢的原因）

通过这个例子我们知道，go 虽然通过 `interface{}` 为我们提供了一定的灵活性支持，但是使用这种动态的特性是有一定代价的，比如：

*   我们在运行时才知道类型，那么我们就需要在运行时去做类型判断（也就是通过**反射**），这种判断会有一定开销（本来是确定的一种类型，但是现在可能要在 20 多个类型中匹配才能确定它的类型是什么）。同时，判断到属于某一类型之后，往往需要转换为具体的类型，这也是一种开销。
*   同时，我们可能需要去做一些属性、方法的查找等操作（`Field`, `FieldByName`, `Method`, `MethodByName`），这些操作都是在运行时做的，所以会有一定的性能损耗。
*   另外，在做属性、方法之类的查找的时候，查找性能取决于属性、方法的数量，如果属性、方法的数量很多，那么查找性能就会相对慢。**通过 index (`Field`, `Method`)查找相比通过 name (`FieldByName`, `MethodByName`)查找快很多，后者有内存分配的操作**
*   在我们通过反射来做这些操作的时候，多出了很多操作，比如，简单的两个 `int` 类型相加，本来可以直接相加。但是通过反射，我们不得不先根据 `interface{}` 创建一个反射对象，然后再做类型判断，再做类型转换，最后再做加法。

**总的来说，go 的 interface{} 类型虽然给我们提供了一定的灵活性，让开发者也可以在 go 里面实现一些动态语言的特性，** **但是这种灵活性是以牺牲一定的性能来作为代价的，它会让一些简单的操作变得复杂，一方面生成的编译指令会多出几十倍，另一方面也有可能在这过程有内存分配的发生（比如 `FieldByName`）。**

慢是相对的
-----

从上面的例子中，我们发现 go 的反射好像慢到了让人无法忍受的地步，然后就有人提出了一些解决方案， 比如：**通过代码生成的方式避免运行时的反射操作，从而提高性能。比如 easyjson**

但是这类方案都会让代码变得繁杂起来。我们需要权衡之后再做决定。为什么呢？因为反射虽然慢，但我们要知道的是，如果我们的应用中有网络调用，**任何一次网络调用的时间往往都不会少于 1ms，而这 1ms 足够 go 做很多次反射操作了**。这给我们什么启示呢？如果我们不是做中间件或者是做一些高性能的服务，而是做一些 web 应用，那么我们可以考虑一下性能瓶颈是不是在反射这里，如果是，那么我们就可以考虑一下代码生成的方式来提高性能，如果不是，那么我们真的需要牺牲代码的可维护性、可读性来提高反射的性能吗？优化几个慢查询带来的收益是不是更高呢？

go 反射性能优化
---------

如果可以的话，**最好的优化就是不要用反射**。

### 通过代码生成的方式避免序列化和反序列化时的反射操作

这里以 `easyjson` 为例，我们来看一下它是怎么做的。假设我们有如下结构体，我们需要对其进行 json 序列化/反序列化：

```
// person.go
type Person struct {
   Name string `json:"name"`
   Age  int    `json:"age"`
}
复制代码
```

使用 `easyjson` 的话，我们需要为结构体生成代码，这里我们使用 `easyjson` 的命令行工具来生成代码：

```
easyjson -all person.go
复制代码
```

这样，我们就会在当前目录下生成 `person_easyjson.go` 文件，里面包含了 `MarshalJSON` 和 `UnmarshalJSON` 方法，这两个方法就是我们需要的序列化和反序列化方法。不同于标准库里面的 `json.Marshal` 和 `json.Unmarshal`，这两个方法是不需要反射的，它们的性能会比标准库的方法要好很多。

```
func easyjsonDb0593a3EncodeGithubComGinGonicGinCEasy(out *jwriter.Writer, in Person) {
   out.RawByte('{')
   first := true
   _ = first
   {
      const prefix string = ","name":"
      out.RawString(prefix[1:])
      out.String(string(in.Name))
   }
   {
      const prefix string = ","age":"
      out.RawString(prefix)
      out.Int(int(in.Age))
   }
   out.RawByte('}')
}

// MarshalJSON supports json.Marshaler interface
func (v Person) MarshalJSON() ([]byte, error) {
   w := jwriter.Writer{}
   easyjsonDb0593a3EncodeGithubComGinGonicGinCEasy(&w, v)
   return w.Buffer.BuildBytes(), w.Error
}
复制代码
```

我们看到，我们对 `Person` 的序列化操作现在只需要几行代码就可以完成了，但是也有很明显的缺点，生成的代码会很多。

性能差距：

```
goos: darwin
goarch: amd64
pkg: github.com/gin-gonic/gin/c/easy
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
BenchmarkJson
BenchmarkJson-12            3680560          305.9 ns/op      152 B/op         2 allocs/op
BenchmarkEasyJson
BenchmarkEasyJson-12       16834758           71.37 ns/op         128 B/op         1 allocs/op
复制代码
```

我们可以看到，使用 `easyjson` 生成的代码，序列化的性能比标准库的方法要好很多，好了 4 倍以上。

### 反射结果缓存

> 这种方法适用于需要根据名称查找结构体字段或者查找方法的场景。

假设我们有一个结构体 `Person`，其中有 5 个方法，`M1`、`M2`、`M3`、`M4`、`M5`，我们需要通过名称来查找其中的方法，那么我们可以使用 `reflect` 包来实现：

```
p := &Person{}
v := reflect.ValueOf(p)
v.MethodByName("M4")
复制代码
```

这是很容易想到的办法，但是性能如何呢？通过性能测试，我们可以看到，这种方式的性能是非常差的：

```
func BenchmarkMethodByName(b *testing.B) {
   p := &Person{}
   v := reflect.ValueOf(p)

   b.ReportAllocs()
   for i := 0; i < b.N; i++ {
      v.MethodByName("M4")
   }
}
复制代码
```

结果：

```
BenchmarkMethodByName-12         5051679               237.1 ns/op           120 B/op          3 allocs/op
复制代码
```

相比之下，我们如果使用索引来获取其中的方法的话，性能会好很多：

```
func BenchmarkMethod(b *testing.B) {
   p := &Person{}
   v := reflect.ValueOf(p)

   b.ReportAllocs()
   for i := 0; i < b.N; i++ {
      v.Method(3)
   }
}
复制代码
```

结果：

```
BenchmarkMethod-12              200091475                5.958 ns/op           0 B/op          0 allocs/op
复制代码
```

我们可以看到两种性能相差几十倍。那么我们是不是可以通过 `Method` 方法来替代 `MethodByName` 从而获得更好的性能呢？答案是可以的，我们可以缓存 `MethodByName` 的结果（就是方法名对应的下标），下次通过反射获取对应方法的时候直接通过这个下标来获取：

> 这里需要通过 reflect.Type 的 MethodByName 来获取反射的方法对象。

```
// 缓存方法名对应的方法下标
var indexCache = make(map[string]int)

func methodIndex(p interface{}, method string) int {
   if _, ok := indexCache[method]; !ok {
      m, ok := reflect.TypeOf(p).MethodByName(method)
      if !ok {
         panic("method not found!")
      }

      indexCache[method] = m.Index
   }

   return indexCache[method]
}
复制代码
```

性能测试：

```
func BenchmarkMethodByNameCache(b *testing.B) {
   p := &Person{}
   v := reflect.ValueOf(p)

   b.ReportAllocs()
   var idx int
   for i := 0; i < b.N; i++ {
      idx = methodIndex(p, "M4")
      v.Method(idx)
   }
}
复制代码
```

结果：

```
// 相比原来的 MethodByName 快了将近 20 倍
BenchmarkMethodByNameCache-12           86208202                13.65 ns/op            0 B/op          0 allocs/op
BenchmarkMethodByName-12                 5082429               235.9 ns/op           120 B/op          3 allocs/op
复制代码
```

> 跟这个例子类似的是 Field/FieldByName 方法，可以采用同样的优化方式。这个可能是更加常见的操作，反序列化可能需要通过字段名查找字段，然后进行赋值。

### 使用类型断言代替反射

在实际使用中，如果只是需要进行一些简单的类型判断的话，比如判断是否实现某一个接口，那么可以使用类型断言来实现：

```
type Talk interface {
   Say()
}

type person struct {
}

func (p person) Say() {
}

func BenchmarkReflectCall(b *testing.B) {
   p := person{}
   v := reflect.ValueOf(p)

   for i := 0; i < b.N; i++ {
      idx := methodIndex(&p, "Say")
      v.Method(idx).Call(nil)
   }
}

func BenchmarkAssert(b *testing.B) {
   p := person{}

   for i := 0; i < b.N; i++ {
      var inter interface{} = p
      if v, ok := inter.(Talk); ok {
         v.Say()
      }
   }
}
复制代码
```

结果：

```
goos: darwin
goarch: amd64
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
BenchmarkReflectCall-12          6906339               173.1 ns/op
BenchmarkAssert-12              171741784                6.922 ns/op
复制代码
```

在这个例子中，我们就算使用了缓存版本的反射，性能也跟类型断言差了将近 25 倍。

> 因此，在我们使用反射之前，我们需要先考虑一下是否可以通过类型断言来实现，如果可以的话，那么就不需要使用反射了。

总结
--

*   go 提供了性能测试的工具，我们可以通过 `go test -bench=.` 这种命令来进行性能测试，运行命令之后，文件夹下的测试文件中的 `Benchmark*` 函数会被执行。
*   性能测试的结果中，除了平均执行耗时之外，还有内存分配的次数和内存分配的字节数，这些都是我们需要关注的指标。其中内存分配的次数和内存分配的字节数是可以通过 `b.ReportAllocs()` 来进行统计的。内存分配的次数和内存分配的字节数越少，性能越好。
*   反射虽然慢，但是也带来了一定的灵活性，它的慢主要由以下几个方面的原因造成的：
    *   运行时需要进行类型判断，相比确定的类型，运行时可能需要在 20 多种类型中进行判断。
    *   类型判断之后，往往需要将 `interface{}` 转换为具体的类型，这个转换也是需要消耗一定时间的。
    *   方法、字段的查找也是需要消耗一定时间的。尤其是 `FieldByName`, `MethodByName` 这种方法，它们需要遍历所有的字段和方法，然后进行比较，这个比较的过程也是需要消耗一定时间的。而且这个过程还需要分配内存，这会进一步降低性能。
*   慢不慢是一个相对的概念，如果我们的应用大部分时间是在 IO 等待，那么反射的性能大概率不会成为瓶颈。优化其他地方可能会带来更大的收益，同时也可以在不影响代码可维护性的前提下，使用一些时空复杂度更低的反射方法，比如使用 `Field` 代替 `FieldByName` 等。
*   如果可以的话，尽量不使用反射就是最好的优化。
*   反射的一些性能优化方式有如下几种（不完全，需要根据实际情况做优化）：
    *   使用生成代码的方式，生成特定的序列化和反序列化方法，这样就可以避免反射的开销。
    *   将第一次反射拿到的结果缓存起来，这样如果后续需要反射的话，就可以直接使用缓存的结果，避免反射的开销。（**空间换时间**）
    *   如果只是需要进行简单的类型判断，可以先考虑一下类型断言能不能实现我们想要的效果，它相比反射的开销要小很多。

反射是一个很庞大的话题，这里只是简单的介绍了一小部分反射的性能问题，讨论了一些可行的优化方案，但是每个人使用反射的场景都不一样，所以需要根据实际情况来做优化。
