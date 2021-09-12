---
original: https://dev.to/fabriciopashaj/understanding-webassembly-better-by-learning-webassembly-text-50bj
author: Fabricio Pashaj
translator: https://github.com/lxy96Scorpion
reviewer: 
title: 通过学习 WebAssembly-Text 更好地理解 WebAssembly
summary: 通过 WebAssembly-Text 理解 WebAssembly
categories: 译文
tags: ["WebAssembly", "WebAssembly-Text"]
originalPublishDate: 2021-08-15
publishDate: 2021-09-08
---

# 通过学习 WebAssembly-Text 更好地理解 WebAssembly
<p align='right'>2021 年 8 月 15 日 法布里西奥 · 帕沙伊</p>

## 通过学习 WebAssembly Text 更好地理解 WebAssembly

WebAssembly 是一场真正的技术革命，不仅在前端领域，而且由于 WASI （ WebAssembly System Interface ）和相关的其他技术， WebAssembly 正在变得无处不在。
WebAssembly 提供的最好的东西之一是成为编译目标，而不仅仅是另一种编程语言。这有可能帮助许多非 JS 开发者参与 Web 开发。WebAssembly 也有文本版本，称为...你懂的， WebAssembly Text ，或简称 WAT！（[此处为](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format) MDN 文档）。可以使用[ WABT ](https://github.com/WebAssembly/wabt)将其编译为二进制文件。

## 准备事项（跟着了解）
+ 您知道如何使用 WABT 来组装 WAT。
+ 您知道如何运行 WebAssembly 二进制文件。

## 理解语法
WAT 提供了两种编写代码的方式：
传统的汇编风格

```
    local.get 0
    local.get 1
    i32.add
```

和更 LISPy 的方式（称为 S-Expression 格式）

```
    (i32.add
        (local.get 0)
        (local.get 1))
```

汇编程序将从这两个程序中输出相同的结果，但前者以更清楚的方式显示指令是如何放在二进制文件中。我们将在本文中使用这种编码风格。最基础、有效的（虽然是无用的） WAT 文件包含以下内容：

```
(module)
```

## WebAssembly 的堆栈如何工作

堆栈只不过是一个 LIFO（后进先出）线性数据结构。把他想象成一个数组，你只能 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">push</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">pop</span> 不能以任何其他方式访问这一数据结构。
WebAsembly 的堆栈没有什么特别的，但有一些特性使他更酷、更安全。
其中之一是堆栈拆分/成帧。这个名字可能看起来很高深莫测，但他比看起来更简单。他只是在堆栈顶部的项目被拆分时所在的位置放置一个标记。标记是栈帧的底部，他只是说：“嘿，这是一个自己的堆栈， 这里的代码块对其进行操作。只有这个代码块可以访问他，其他代码块无法访问。新堆栈的生命周期仅限于完全执行此代码块所需的时间”。我们将调用拆分父帧的堆栈和拆分子帧的新堆栈。

一个<strong>块</strong>是一个之间的部分 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">block</span> /  <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if</span> /  <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">loop</span> 指令
和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">end</span> 指令。每个块都有结果，也就是说当块的堆栈帧到达末尾时他的生命周期，最后一个堆栈帧被弹出。然后该帧被破坏（即，标记被移除) 并且弹出的栈帧被推送到父栈帧。这也是函数的工作方式，但是他们可以有参数并且可以随时执行。

## 你好，世界！嗯，有点。
WebAssembly Text 文件总是以模块定义开始，其他所有内容都放在 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">module</span> 单词和最后一个括号之间。让我们看看如何写一个简单的 “ Hello, world ! ” WAT 中的程序。
```
(module
    (func $hello_world (param $lhs i32) (param $rhs i32) (result i32)
        local.get $lhs
        local.get $rhs
        i32.add)
    (export "helloWorld" (func $hello_world)))
```

好吧，你可能会说“这是什么鬼东西？我以为这是一个‘你好，世界！’ 的例子！”。嗯，重点是 WASM 不是为了打印字符串和与 API 交互而产生的，他的目的是通过提供一个快速、简单指令的接口来帮助 JavaScript 处理繁重的计算。

## 但是代码有什么作用呢？
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 是 WebAssembly 的四种主要类型之一，他是一种 32 位整数类型。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">func</span> 声明一个函数
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$hello_world</span> 是我们为函数提供的编译时名称/标签（稍后我们将看到更多相关信息）
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">(param $lhs i32)</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">(param $rhs i32)</span> 告诉该函数接受两个参数，第一个标记为左侧的 $lhs（注意 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">\$</span> ），类型为 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span>，第二个标记为右侧的 $rhs，类型为 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span>。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">(result i32)</span> 说函数返回类型是一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span>。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">local.get $lhs</span> 将标记为 $lhs 的参数值压入堆栈。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">local.get $rhs</span> 与上述相同，但他会推送 $rhs 的值。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.add</span> <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 从堆栈中弹出两个类型的值，并将他们相加的结果压回到堆栈中。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">(export "helloWorld" (func $hello_world))</span> 将标记为 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$hello_world</span> 的函数导出到名为“helloWorld”的模块。

## <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">return</span> 声明 在哪里？

WebAssembly 有一条 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">return</span> 指令，但只有在您需要立即返回并停止进一步执行该函数时才使用他。否则，在函数的末尾会有一个隐式返回，他弹出堆栈上的最后一个值并返回他。

## “标签”究竟意味什么？
所有的函数调用、参数和本地访问都是通过索引完成的。标签只是编译时注释，使代码更易于阅读、编写和理解。

## <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">local.get</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.add</span> 是唯一的说明吗？
当然不是。WebAssembly 指令集非常庞大。仅 MVP（最小可行产品）就有大约 120 条指令。他们大多以类型开头，只能是 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> , <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i64</span> , <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f32</span> , <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f64</span> ，后跟一个点和操作名称。还有其他用途的说明，例如控制流（决策、分支/跳转和循环）。可以在[此处](https://github.com/sunfishcode/wasm-reference-manual/blob/master/WebAssembly.md#instructions)找到所有说明及其解释的列表。

## 写点有用的东西
下面的示例展示了如何编写使用勾股定理计算两点之间距离的函数。

```
(module
    (func $distance (param $x1 i32) (param $y1 i32) (param $x2 i32) (param $y2 i32) (result f64)

        local.get $x1
        local.get $x2
        i32.sub ;; calculate the X axis distance (a)
        call $square ;; (a ^ 2)

        local.get $y1
        local.get $y2
        i32.sub ;; calculate the Y axis distance (b)
        call $square ;; (b ^ 2)

        i32.add ;; (c^2 = a^2 + b^2)
        f64.convert_s/i32 ;; convert to f64 so we can square root
        f64.sqrt)
    (export "distance" (func $distance))
    (func $square (param $i32) (result i32)
        local.get 0
        local.get 0
        i32.mul))
```

## 分解代码（再次）
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px"> ; ;</span> 是单行注释的开始
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.sub</span> <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 从堆栈中弹出两个类型的数字并推回他们的减法结果
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">call $square</span> 调用函数
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f64.convert_s/i32</span> 从堆栈中弹出一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 并将其值作为转换为一个有符号的 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f64</span> 数返回（如果是 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f64.convert_u/i32</span> 这样，将会把他从无符号形式转换为有符号形式）。如果您不了解有符号数和无符号数之间的区别，我建议您阅读[此内容](https://dev.to/aidiri/signed-vs-unsigned-bit-integers-what-does-it-mean-and-what-s-the-difference-41a3)
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f64.sqrt</span> 从堆栈中弹出一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">f64</span> 数字类型并压入数字的平方根。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.mul</span> 从堆栈中弹出两个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 类型的数字并将他们的乘法结果返回。（我们将一个数与他自己相乘以得到他在 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$square</span> 函数中的 2 次方）。

## 全局变量
全局变量只是每个函数都可以访问的“变量”。他们可以被导出，但只有在他们被声明为可变的情况下才能被改变。下面的代码显示了如何使用他们：

```
(module
    (;
      declare a mutable global with type `i32` and initial value 0
    ;)
    (global $counter (mut i32) (i32.const 0))
    (export "counter" (global $counter)) ;; export it with the name "counter"
    (func $countUp (result i32)
        global.get $counter ;; push the value of $counter into the stack
        i32.const 1
        i32.add ;; increment the value by 1
        global.set $counter ;; assign $counter to the incremented value
        global.get $counter) ;; return the new incremented value
    (export "countUp" (func $countUp)))
```

## 做决定
尽管 WebAssembly 是一种顶层字节码格式，但他支持更高级的概念，如 if 语句和循环。下面的代码显示了一个函数，他接收两个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 作为参数并返回其中最大的一个。

```
(module
    (func $largest (param $0 i32) (param $1 i32) (result i32)
        local.get $0 ;; pushing $0's value into the stack
        local.get $1 ;; pushing $1's value into the stack
        i32.gt_s ;; comparing if $0 is greater than $1
        if (result i32)
            local.get $0
        else
            local.get $1
        end)
    (export "largest" (func $largest)))
```
前 3 条指令很简单，可以用注释进行解释，但如果您仍然不明白他们的作用：
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">local.get $0</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">local.get $1</span> 推送标记为 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$0</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$1</span> 的参数的值
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.gt_s</span> <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 从堆栈中弹出两个 type 值并将他们的比较结果推回（的值 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">param $0</span> 大于 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">param $1</span> 的值）。 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">1</span> 如果是 true ， <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">0</span> 如果是 false 。他有 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">_s</span> 后缀是因为他将数字作为有符号的数字进行比较（即他们可以是负数）。

## 返回 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$0</span> 或返回 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$1</span>
当一条 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if</span> 指令发生时，栈中的最后一帧（条件）被弹出。他必须是一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span>。如果条件不是 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">0</span> ，则执行块内的指令 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if..else/end</span> ，否则 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">else..end</span> 执行 。如果没有 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">else</span> ，则什么都不会发生。您可能会注意到 if “语句”有结果。 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if</span> 是一个块，这意味着他能够返回结果。

## 选择
对于简单的决定，比如选择一个数字，<span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if</span> 可能有点矫枉过正。还有一条 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">select</span> 指令像三元运算符一样工作。要使用 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">select</span> 而不是 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if</span> ，我们会这样做。
```
local.get $0
local.get $1 ;; push value of $0 and $1 into stack for selection
local.get $0
local.get $1 ;; push value of $0 and $1 into stack for comparing
i32.gt_s ;; doing the comparison between the two last `i32` items on the stack
select
```
该 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">select</span> 指令从堆栈中弹出 3 个值。他根据第三个值（条件）决定推回前两个值中的哪一个。（如果条件不是 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">0</span>，则推送第一个值，否则推送第二个值）。

## 循环和分支
WebAssembly 支持循环，但不是您可能想到的那种循环。以这段代码为例：
```
(module
    (func $fac (param $0 i32) (result i32) (local $acc i32)
        (local $num i32)

        local.get $0
        local.tee $acc
        local.set $num
        block $outer
            loop $loop
                local.get $num
                i32.const 1
                i32.lt_u ;; we check if $num is lower than 1
                br_if $outer
                local.get $acc
                local.get $num
                i32.const 1
                i32.sub
                local.tee $num
                i32.add
                local.set $acc
                br $loop
            end
        end
        local.get $acc)
    (export "factorial" (func $fac)))
```
此代码显示了如何编写一个函数，该函数使用循环查找数字的阶乘。使用递归可能更容易，但这里的重点是理解循环和分支。

## 分解代码
这段代码中的大部分内容都已经解释过了。唯一的新东西这里有 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">(local $acc i32)</span>， <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">block</span> ， <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">loop</span>， <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">br_if $outer</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">$br $loop</span>。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">(local $acc i32)</span> 类似于他们在高级语言中所说的局部变量。他的访问方式与参数相同。第一个本地的索引比最后一个参数的索引多 1。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">block $outer</span>没有什么特别的，他只是封装了他和相应<span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">end</span>指令之间的代码。当您需要在不同级别进行分支时使用他。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">loop $loop</span> 是一种特殊的<strong>块</strong>，如果你在循环上做一个分支，你不会 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">end</span> 去到循环的开始，你会去到循环的开头。（分支相当于 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">break</span> 上的 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">block</span> 与 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">if</span> 和像 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">continue</span> 上的 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">loop</span> ）
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">br $loop</span> 是无条件分支。标签操作数是一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">loop</span>，因此他将跳转到 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">loop</span> 指令顶部发生。如果您了解 C/C++，您就知道使用 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">goto</span> 跳转，通过使用 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">goto</span> 您可以从代码中的任何地方跳转到任何地方。WebAssembly 的限制性更强，您只能向外和按标签/索引进行分支。最里面的<strong>块</strong>具有最小的索引( <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">0</span> )，最外面的<strong>块</strong>具有最大的索引。我们执行分支以便我们可以继续循环而不是退出。
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">br_if $outer</span> 是条件分支/跳转指令。他将从堆栈中弹出最后一项（条件必须是一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> )，如果他与 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">0</span> 不同，将会执行分支，否则不会。

## 线性内存
WebAssembly 提供了另一种存储数据的方式，而不是堆栈，线性内存。他可以被看作是一个可调整大小的 JavaScript <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">TypeArray</span>。其主要目的是存储复杂 and / or 连续的数据。有 14 条加载指令和 9 条存储指令，以及 2 条其他用于操作和获取其大小的指令。有了到目前为止我们学到的知识，让我们实现一个生成斐波那契数列的函数。
```
(module
    (memory 1)
    (export "memory" (memory 0))
    (func $fib (param $length i32) (local $offset i32)
        i32.const 8
        local.set $offset ;; assign offset to 8 (see below)
        i32.const 0
        i32.const 1
        i32.store offset=4 ;; store 1 at offset 0 eith static offset 4
        block $block
            loop $loop
                local.get $offset
                i32.const 4
                i32.div ;; divide by 4(read below)
                local.get $length
                i32.gt_u ;; compare the requsted length
                br_if $block ;; break out if false (`9`)
                local.get $offset ;; get the offset for storing
                local.get $offset
                ;; ---------
                i32.const 8
                i32.sub
                i32.load
                local.get $offset
                i32.const 4
                i32.sub
                i32.load
                i32.add ;; load the two previous numbers from memory and add them
                ;; ---------
                i32.store ;; store the number at the current offset
                local.get $offset
                i32.const 4
                i32.add
                local.set $offset ;; increment the offset by 4
                br $loop
            end
        end)
    (export "fibonacci" (func $fib)))
```
## 解释几件事
上面的代码使用了您在阅读本文时获得的几乎所有知识。需要注意的几点：
+ <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">store offset=4</span> 存储一个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 。他从堆栈中弹出两个栈帧。第一个是地址/偏移量，第二个是将要存储的值。 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">offset=4</span> 是静态偏移量，这意味着他将从堆栈中得到的偏移量加起来，没有你需要做 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.const 4</span> 和 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32.add</span> 上偏移。在此示例中， <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">offset</span> 仅用于演示目的。
+ 我们使用偏移量而不是索引，因为我们没有像数组这样的花哨的东西。每个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span> 占用 4 个字节，我们必须将偏移量增加 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">4</span> 而不是 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">1</span> 。
+ 偏移变量指向内存中的地址，该地址将存储前两个数字相加后所得的数字，这就是他最初为 8 的原因。两个 <span style="background: #E5E5E5; color: #333;border-radius: 5px; padding: 0 3px">i32</span>s 在内存中占用 8 个字节。
+ 比较时，我们将偏移量除以 4，使其像“索引”一样。

## 结束。
我希望这篇文章能让您更深入地了解 WebAssembly 的工作原理。非常感谢[ rom ](https://github.com/romdotdog)，他多次编辑和更正了我的文章