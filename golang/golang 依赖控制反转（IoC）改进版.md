# golang 依赖控制反转（IoC）改进版

　　最近在开发基于golang下的cqrs框架 <https://github.com/berkaroad/squat> （陆续开发中，最近断了半年，懒了。。。）。这个框架依赖ioc框架，因为之前写了一个ioc ，所以借此完善下，主要从灵活性、易用性、性能角度进行了优化。顺带也支持了go mod，并将源码文件合并为单文件，方便有直接移植源码的人（license信息请保留，尊重著作权）。

　　先来个直观的调用端对比（v1.2.0为新版，v0.1.1为旧版）：

```go
var container = ioc.NewContainer() // v0.1.1
var container = ioc.New() // v1.2.0，非必需，可以直接使用ioc.XXX 即使用内置全局Container

// register service to *struct
container.Register(&Class2{Name: "Jerry Bai"}, ioc.Singleton) // v0.1.1
ioc.AddSingleton[*Class2](&Class2{Name: "Jerry Bai"}) // v1.2.0
container.Register(&Class1{}, ioc.Transient) // v0.1.1，调用函数InitFunc()获取初始化函数来完成初始化
ioc.AddTransient[*Class1](func() *Class1 { // v1.2.0，直接通过传入的函数来完成
    var svc Class1
    // inject to *struct
    ioc.Inject(&svc) // v1.2.0，支持注入到结构体
}

// register service to interface.
container.RegisterTo(&Class2{Name: "Jerry Bai"}, (*Interface2)(nil), ioc.Singleton) // v0.1.1
ioc.AddSingleton[Interface2](&Class2{Name: "Jerry Bai"}) // v1.2.0
container.RegisterTo(&Class1{}, (*Interface1)(nil), ioc.Transient) // v0.1.1，调用函数InitFunc()获取初始化函数来完成初始化
ioc.AddTransient[Interface1](func() Interface1 { // v1.2.0，直接通过传入的函数来完成
    var svc Class1
    // inject to *struct
    ioc.Inject(&svc) // v1.2.0，支持注入到结构体
}

// inject to function
ioc.Inject(func(c1 *Class1, c2 *Class2, i1 Interface1, i2 Interface2, resolver ioc.Resolver) { // v1.2.0
    println("c1.C2Name=", c1.C2.Name)
    println("c2.Name=", c2.Name)
    println("i1.GetC2Name=()", i1.GetC2Name())
    println("i2.GetName=()", i2.GetName())
})
container.Invoke(func(c1 *Class1, c2 *Class2, i1 Interface1, i2 Interface2, roContainer ioc.ReadonlyContainer) { // v0.1.1
    println("c1.C2Name=", c1.C2Name)
    println("c2.Name=", c2.Name)
    println("i1.GetC2Name=()", i1.GetC2Name())
    println("i2.GetName=()", i2.GetName())
})
```

　　新版本，从功能角度，增加支持了结构体注入、泛型方式获取服务实例、替换已存在的服务。

```go
// get service from ioc（go1.18开始支持泛型）
c1 := ioc.GetService[*Class1]
c2 := ioc.GetService[*Class2]
i1 := ioc.GetService[Interface1]
i2 := ioc.GetService[Interface2]
```

```go
// override exists service（一般用于覆盖默认注入的对象）
c := ioc.New()
ioc.SetParent(c)
ioc.AddSingletonToC[Interface3](c, &Class3{Name: "Jerry Bai"}) // add service to parent's container
i3 := ioc.GetService[Interface3]() // *Class3, 'Interface3' only exists in parent's container
ioc.AddSingleton[Interface3](&Class4{Name: "Jerry Bai"}) // add service to global's container
i3 = ioc.GetService[Interface3]() // *Class4, 'Interface3' exists in both global and parent's container
```

　　对结构体初始化的函数定义（模拟构造函数），从固定获取函数的接口 `interface{InitFunc() interface{}}` 改为按函数名获取（默认为 `Initialize`）。

```go
type Class2 struct {
    Name     string
    resolver ioc.Resolver
}

func (c *Class2) Initialize(resolver ioc.Resolver) string {
    c.resolver = resolver
    return c.Name
}

type Class3 struct {
    Name     string
    resolver ioc.Resolver
}

// specific custom initialize method name
func (c *Class3) InitializeMethodName() string {
    return "MyInitialize"
}

// custom initialize method
func (c *Class3) MyInitialize(resolver ioc.Resolver) string {
    c.resolver = resolver
    return c.Name
}
```

以下是新版本的性能测试。带 “Native” 的为原生调用，具体测试代码，参见源码：<https://github.com/berkaroad/ioc/blob/master/benchmark_test.go>

```plain
go test -run=none -count=1 -benchtime=1000000x -benchmem -bench=. ./...

goos: linux
goarch: amd64
pkg: github.com/berkaroad/ioc
cpu: AMD Ryzen 7 5800H with Radeon Graphics         
BenchmarkGetSingletonService-4           1000000                26.16 ns/op            0 B/op          0 allocs/op
BenchmarkGetTransientService-4           1000000               370.9 ns/op            48 B/op          1 allocs/op
BenchmarkGetTransientServiceNative-4     1000000               131.9 ns/op            48 B/op          1 allocs/op
BenchmarkInjectToFunc-4                  1000000               659.5 ns/op           144 B/op          5 allocs/op
BenchmarkInjectToFuncNative-4            1000000                89.26 ns/op            0 B/op          0 allocs/op
BenchmarkInjectToStruct-4                1000000               311.7 ns/op             0 B/op          0 allocs/op
BenchmarkInjectToStructNative-4          1000000                87.64 ns/op            0 B/op          0 allocs/op
PASS
ok      github.com/berkaroad/ioc        1.686s
```

---------------------------------分割线-------------------------------------------------------------

我的[ioc项目: https://github.com/berkaroad/ioc](https://github.com/Berkaroad/ioc)，已经挂在github上，有兴趣的可以去了解下。

使用中有何问题，欢迎在github上给我提issue，谢谢！
