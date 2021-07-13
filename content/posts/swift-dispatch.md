---
title: "Swift Dispatch"
date: 2021-01-08T14:30:41+08:00
type: "posts"
draft: false
---

本文探讨一下 Swift 中的方法调度。

# 1. 派发机制

函数派发(Method Dispatch)又名方法调度，简单讲就是编程语言在源代码中判断用哪种方式调用函数的机制。一般来讲方法派发的方式分为两种：

- 静态派发([Static Dispatch](https://en.wikipedia.org/wiki/Static_dispatch))

    又被称为直接派发，系统直接按照方法的具体实现地址进行调用，在编译期间就确定了调用的方法的具体实现。甚至编译器能够进行内联优化，因此，当调用该方法时，调用的指令集非常少，执行速度快，但相应的缺点也很明显，没有动态性。我们常见的C语言就使用直接派发、C++默认也使用直接派发。

- 动态派发([Dynamic Dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch))

    动态派发是指需要在运行时选择方法的具体实现。动态调度是多态的具体实现过程，被广泛的应用于 OOP 语言(object-oriented programing)中。一般实现动态派发的方式有以下几种：

    - 函数表派发

        函数表派发是动态派发最常见的一种实现方式。每个类中会存在一个函数表，存储了每个函数实现的指针。当调用方法时，先找到类的函数表，然后再函数表中找到要调用的方法指针，最后根据该指针找到方法的具体实现。C++、Java 等语言就是通过函数表的方式来实现多态。

    - 消息派发

        方法调用就是给对象发送消息。以 Objective-C 举例， 调用方法需要在相应的类中遍历其方法列表，如果能找到相匹配的方法就调用该方法的实现，如果找不到，就沿着继承链的顺序继续向上查找。

# 2. SIL

[SIL](https://github.com/apple/swift/blob/main/docs/SIL.rst) 是 Swift 编译器中间语言，通过 `swiftc` 将 Swift 转化为 SIL 可以方便的查看 Swift 的方法派发方式。通过SIL文档中对[动态调度](https://github.com/apple/swift/blob/main/docs/SIL.rst#dynamic-dispatch)的描述，我们可以查看动态调度所用的指令。这里简单总结下判断方法是动态调度还是静态调度的方式：

- `class_method` 和 `super_method` 使用 `vtable` 派发
- `witness_method` 使用 `witness_table` 派发(Swift 协议中函数表派发的实现)
- `objc_method` 和 `objc_super_method` 使用 `Objective-C message` 派发，并使用 `foreign` 标记
- 其他诸如 `function_ref` 等指令使用的是静态派发

# 3. Swift 中的派发方式

Swift 中支持以下三种派发方式

1. 静态派发
2. 函数表派发
3. 消息派发

Swift 中的派发机制主要与以下四点有关

- 数据类型
- 声明位置
- 指定派发方式
- 编译器优化

## 3.1 数据类型

- 值类型

    先来分析一下 Swift 结构体中的函数派发方式

    ```swift
    struct MyStruct {
        func myMethod() {}
    }

    let myStruct = MyStruct()
    myStruct.myMethod()
    ```

    利用 swiftc 将上述代码转换成 SIL，命令为 `swiftc -emit-silgen main.swift > main.sil`

    ```swift
    // 截取 myStruct 的创建和调用 myMethod 方法部分
    // function_ref MyStruct.init()
    %5 = function_ref @$s4main8MyStructVACycfC : $@convention(method) (@thin MyStruct.Type) -> MyStruct // user: %6
    %6 = apply %5(%4) : $@convention(method) (@thin MyStruct.Type) -> MyStruct // user: %7
    store %6 to [trivial] %3 : $*MyStruct           // id: %7
    %8 = load [trivial] %3 : $*MyStruct             // user: %10
    // function_ref MyStruct.myMethod()
    %9 = function_ref @$s4main8MyStructV8myMethodyyF : $@convention(method) (MyStruct) -> () // user: %10
    %10 = apply %9(%8) : $@convention(method) (MyStruct) -> ()
    ```

    从上述代码中我们可以看到在调用了 `MyStruct.init()` 方法后，直接调用了 `MyStruct.myMethod()`，同时调用该方法前使用了 `function_ref` 指令，使用的是静态派发。利用同样的方法，我们可以验证 enum 中的方法同样使用静态调度。因此我们可以得出结论，Swift 中值类型的方法采用静态派发。

- 引用类型

    ```swift
    class MyClass {
        func myMethod() {}
    }

    let myClass = MyClass()
    myClass.myMethod()
    ```

    转换成 SIL 后，我们查看以下关键代码

    ```swift
    // main
    sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
    bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
      // 省略无关代码...
      %9 = class_method %8 : $MyClass, #MyClass.myMethod : (MyClass) -> () -> (), $@convention(method) (@guaranteed MyClass) -> () // user: %10
      %10 = apply %9(%8) : $@convention(method) (@guaranteed MyClass) -> ()
      // 省略无关代码...
    } // end sil function 'main'

    // ...
    // 省略无关代码...
    // ...

    sil_vtable MyClass {
      #MyClass.myMethod: (MyClass) -> () -> () : @$s4main7MyClassC8myMethodyyF	// MyClass.myMethod()
      // 省略无关代码...
    }
    ```

    我们可以看到 `MyClass.myMethod` 方法出现在了 MyClass 的 `sil_vtable` 表中，同时调用 MyClass 中的 `myMethod` 方法前使用的指令为 `class_method`，根据 [SIL 文档](https://github.com/apple/swift/blob/main/docs/SIL.rst#dynamic-dispatch) 解释，此处使用的是 `vtable dispatch`(函数表派发)。

- 协议

    ```swift
    protocol MyProtocol {
        func myMethod()
    }

    class MyClass: MyProtocol {
        func myMethod() {}
    }

    let myProtocol: MyProtocol = MyClass()
    myProtocol.myMethod()
    ```

    转换成 SIL

    ```swift
    // main
    sil [ossa] @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
    bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
    	// 省略无关代码...
      %10 = witness_method $@opened("1C1A86E8-E2F3-11EB-B18F-ACDE48001122") MyProtocol, #MyProtocol.myMethod : <Self where Self : MyProtocol> (Self) -> () -> (), %9 : $*@opened("1C1A86E8-E2F3-11EB-B18F-ACDE48001122") MyProtocol : $@convention(witness_method: MyProtocol) <τ_0_0 where τ_0_0 : MyProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %9; user: %11
      %11 = apply %10<@opened("1C1A86E8-E2F3-11EB-B18F-ACDE48001122") MyProtocol>(%9) : $@convention(witness_method: MyProtocol) <τ_0_0 where τ_0_0 : MyProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %9
      // 省略无关代码...
    } // end sil function 'main'

    // protocol witness for MyProtocol.myMethod() in conformance MyClass
    sil private [transparent] [thunk] [ossa] @$s4main7MyClassCAA0B8ProtocolA2aDP8myMethodyyFTW : $@convention(witness_method: MyProtocol) (@in_guaranteed MyClass) -> () {
    // %0                                             // user: %1
    bb0(%0 : $*MyClass):
      %1 = load_borrow %0 : $*MyClass                 // users: %5, %3, %2
      %2 = class_method %1 : $MyClass, #MyClass.myMethod : (MyClass) -> () -> (), $@convention(method) (@guaranteed MyClass) -> () // user: %3
      %3 = apply %2(%1) : $@convention(method) (@guaranteed MyClass) -> ()
      %4 = tuple ()                                   // user: %6
      end_borrow %1 : $MyClass                        // id: %5
      return %4 : $()                                 // id: %6
    } // end sil function '$s4main7MyClassCAA0B8ProtocolA2aDP8myMethodyyFTW'

    sil_vtable MyClass {
      #MyClass.myMethod: (MyClass) -> () -> () : @$s4main7MyClassC8myMethodyyF	// MyClass.myMethod()
      // 省略无关代码...
    }

    sil_witness_table hidden MyClass: MyProtocol module main {
      method #MyProtocol.myMethod: <Self where Self : MyProtocol> (Self) -> () -> () : @$s4main7MyClassCAA0B8ProtocolA2aDP8myMethodyyFTW	// protocol witness for MyProtocol.myMethod() in conformance MyClass
    }
    ```

    我们可以看到 `myMethod` 出现在了 `sil_witness_table` 中，调用时用 `witness_method` 指令，所以 protocol 中的方法也采用动态派发。此外，我们通过 `MyProtocol.myMethod` 的具体实现 `@$s4main7MyClassCAA0B8ProtocolA2aDP8myMethodyyFTW` 可以看到，其内部还是调用了 `MyClass.myMethod` 方法。

通过上述分析我们可以得出结论：值类型的函数采用静态派发，引用类型的函数采用动态派发(class 采用 `vtable`，protocol 采用 `witness_table`)。

## 3.2 声明位置

函数声明位置的不同使用的派发方式也可能不相同，我们接着3.1中的分析继续验证在各个数据结构的扩展中声明的函数派发方式

- 值类型

    ```swift
    struct MyStruct {}

    extension MyStruct {
        func myMethod() {}
    }

    let myStruct = MyStruct()
    myStruct.myMethod()
    ```

    转换成 SIL 后函数调用部分代码

    ```swift
    // function_ref MyStruct.myMethod()
    %9 = function_ref @$s4main8MyStructV8myMethodyyF : $@convention(method) (MyStruct) -> () // user: %10
    %10 = apply %9(%8) : $@convention(method) (MyStruct) -> ()
    ```

    我们看到对于 struct 来说，在 extension 中声明的函数并不会改变派发方式，依然采用静态派发。同理我们可验证在 enum extension 中依然采用静态派发。

- 引用类型

    将函数声明放在 class extension 中的 SIL 代码如下

    ```swift
    // main
    sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
    bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
      // 省略无关代码...
      // function_ref MyClass.myMethod()
      %9 = function_ref @$s4main7MyClassC8myMethodyyF : $@convention(method) (@guaranteed MyClass) -> () // user: %10
      %10 = apply %9(%8) : $@convention(method) (@guaranteed MyClass) -> ()
      // 省略无关代码...
    } // end sil function 'main'

    sil_vtable MyClass {
      #MyClass.init!allocator: (MyClass.Type) -> () -> MyClass : @$s4main7MyClassCACycfC	// MyClass.__allocating_init()
      #MyClass.deinit!deallocator: @$s4main7MyClassCfD	// MyClass.__deallocating_deinit
    }
    ```

    我们可以看到，MyClass 的 `vtable` 中已经没有了 `myMethod` 方法。同时，调用 `myMethod` 时采用了静态派发的方式，这说明 class extension 中方法的派发方式为静态派发。

- 协议

    我们将函数的声明位置放到 protocol extension 中，生成的 SIL如下

    ```swift
    // main
    sil [ossa] @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
    bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
      // 省略无关代码...
      // function_ref MyProtocol.myMethod()
      %10 = function_ref @$s4main10MyProtocolPAAE8myMethodyyF : $@convention(method) <τ_0_0 where τ_0_0 : MyProtocol> (@in_guaranteed τ_0_0) -> () // user: %11
      %11 = apply %10<@opened("0BF36FCA-E38C-11EB-96BB-ACDE48001122") MyProtocol>(%9) : $@convention(method) <τ_0_0 where τ_0_0 : MyProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %9
      // 省略无关代码...
    } // end sil function 'main'

    // ...
    // 省略无关代码...
    // ...

    sil_witness_table hidden MyClass: MyProtocol module main {
    }
    ```

    我们可以看到 `sil_witness_table` 中并没有 `myMethod` 方法，且调用时直接代用静态派发。

通过以上分析我们可以得出结论：值类型、引用类型和协议三者的扩展中声明的方法采用静态派发。

## 3.3 指定派发方式

我们只讨论会改变其原本派发方式的关键字

- `final`

    `final` 是用于 classes 或者 class members 前的关键字，用来防止被重写，当类的函数使用 `final` 关键字修饰后，生成的 SIL

    ```swift
    // main
    sil [ossa] @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
    bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
      // 省略无关代码...
      // function_ref MyClass.myMethod()
      %9 = function_ref @$s4main7MyClassC8myMethodyyF : $@convention(method) (@guaranteed MyClass) -> () // user: %10
      %10 = apply %9(%8) : $@convention(method) (@guaranteed MyClass) -> ()
      // 省略无关代码...
    } // end sil function 'main'

    // ...
    // 省略无关代码...
    // ...

    sil_vtable MyClass {
      #MyClass.init!allocator: (MyClass.Type) -> () -> MyClass : @$s4main7MyClassCACycfC	// MyClass.__allocating_init()
      #MyClass.deinit!deallocator: @$s4main7MyClassCfD	// MyClass.__deallocating_deinit
    }
    ```

    可以看出函数的派发方式变成了静态派发，而且 `sil_vtable` 中已没有了 `myMethod` 方法。同样的，用 `final` 关键字修饰 class 后，自定义的函数同样会被自动定义为 `final`，所以派发方式也为静态派发。

- `static`

    通过在方法的 `func` 关键字前写上 `static` 关键字来表示类型方法，Swift 中的所有类型(结构体、枚举、类、协议)都可以定义类型方法。

    struct、struct extension、enum、enum extension、class、class extension、protocol、protocol extension 等类型中的方法用 `static` 修饰后都采用静态派发方式，这里不做验证，有兴趣可自行转换成 SIL 查看。

- `class`

    在 Swift 类中声明类型方法时 `static` 关键字可用 `class` 来代替，并且如果没有 `final` 修饰的话，允许子类覆盖父类对该方法的实现。对应的 SIL 为

    ```swift
    // main
    sil [ossa] @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
    bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
      // 省略无关代码...
      %3 = class_method %2 : $@thick MyClass.Type, #MyClass.myMethod : (MyClass.Type) -> () -> (), $@convention(method) (@thick MyClass.Type) -> () // user: %4
      %4 = apply %3(%2) : $@convention(method) (@thick MyClass.Type) -> ()
      // 省略无关代码...
    } // end sil function 'main'

    //
    // 省略无关代码...
    //

    sil_vtable MyClass {
      #MyClass.myMethod: (MyClass.Type) -> () -> () : @$s4main7MyClassC8myMethodyyFZ	// static MyClass.myMethod()
      #MyClass.init!allocator: (MyClass.Type) -> () -> MyClass : @$s4main7MyClassCACycfC	// MyClass.__allocating_init()
      #MyClass.deinit!deallocator: @$s4main7MyClassCfD	// MyClass.__deallocating_deinit
    }
    ```

    我们可以看到，用 `class` 来修饰的类方法出现在了 `sil_vtable` 中，且采用的是动态派发。此外，在类的扩展中的方法用 `class` 修饰后依然采用静态派发，且子类不可重写(子类重写会得到一个编译时错误)。

- `@objc`

    class extension 中的方法用 `@objc` 标记后派发方式改为了消息派发。至于原因，目前还没有找到相关的文档说明

    ```swift
    %9 = objc_method %8 : $MyClass, #MyClass.myMethod!foreign : (MyClass) -> () -> (), $@convention(objc_method) (MyClass) -> () // user: %10
    %10 = apply %9(%8) : $@convention(objc_method) (MyClass) -> ()
    ```

- `@objc dynamic`

    ```swift
    %9 = objc_method %8 : $MyClass, #MyClass.myMethod!foreign : (MyClass) -> () -> (), $@convention(objc_method) (MyClass) -> () // user: %10
    %10 = apply %9(%8) : $@convention(objc_method) (MyClass) -> ()
    ```

    class 或者 class extension 中的方法被标记为 `@objc dynamic` 会变成消息派发

## 4. 效率对比

我们来对比一下 Swift 中各种方法调度所耗费的时间

```swift
protocol TestProtocol {
    func witnessTableDispatch()
}

struct TestStruct {
    func staticDispatch() {}
}

class TestClass: TestProtocol {
    final func staticDispatch() {}
    func vTableDispatch() {}
    func witnessTableDispatch() {}
    @objc dynamic func messageDispatch() {}
}

var testStruct = TestStruct()
var testClass = TestClass()

var startDate = Date()
var maxTimes = Int(1e7)

(1...maxTimes).forEach { _ in testStruct.staticDispatch() }
print("static dispatch in struct duration: \(Date().timeIntervalSince(startDate))")

startDate = Date()
(1...maxTimes).forEach { _ in testClass.staticDispatch() }
print("static dispatch in class duration: \(Date().timeIntervalSince(startDate))")

startDate = Date()
(1...maxTimes).forEach { _ in testClass.vTableDispatch() }
print("vtable dispatch duration: \(Date().timeIntervalSince(startDate))")

startDate = Date()
(1...maxTimes).forEach { _ in testClass.witnessTableDispatch() }
print("witness_table dispatch duration: \(Date().timeIntervalSince(startDate))")

startDate = Date()
(1...maxTimes).forEach { _ in testClass.messageDispatch() }
print("objc_message dispatch duration: \(Date().timeIntervalSince(startDate))")
```

经过多次运行，输出结果

![swift_dispatch_performance](/images/swift_dispatch_performance.png)

我们可以发现，struct 中的静态派发要比 class 中的快，而 class 中的静态派发、vtable 派发 和 witness_table 派发耗时基本相同，objc_message 派发最慢。从而我们可以得出 Swift 中派发方式执行效率排名：

静态派发 > 函数派发 > 消息派发

## 总结

Swift 中支持静态派发和动态派发的方式，而且实现动态派发的方式还不止一种(`vtable`、`witness_table`、`objc_message`)。数据类型、方法声明位置、方法关键字的不同，使用的派发方式也不同。汇总在一起可以得出如下表格：

![swift_dispatch_table](/images/swift_dispatch_table.svg)

不同派发方式的执行效率也不同。此外，Swift 编译器还会优化编译方式，甚至会使用内联优化，从而使代码执行效率更高。

而至于为什么效率对比中的输出结果中显示，结构体中的静态派发和类中的静态派发时间不一致？类中的静态派发时间和函数表派发时间差不多甚至还要更多一点？我们以后再探索。