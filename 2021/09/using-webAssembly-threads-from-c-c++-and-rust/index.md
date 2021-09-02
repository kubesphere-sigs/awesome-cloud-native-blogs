```
  
---
original: https://web.dev/webassembly-threads
author: Ingvar Stepanyan
translator: https://github.com/swartz-k
reviewer: 
title: Using WebAssembly threads from C, C++ and Rust
summary: 了解如何将用其他语言编写的多线程应用程序引入 WebAssembly。
categories: 译文
tags: webassembly
originalPublishDate: 2021-07-12
publishDate: 
---
```

## [Using WebAssembly threads from C, C++ and Rust](https://web.dev/webassembly-threads)

支持线程是 WebAssembly 很重要的特性之一。它能够让您在多核场景并行地运行部分代码，或者在输入数据中同时使用相同的代码处理部分数据，得益于能够扩展到与您的CPU内核数一样多的能力，它可以显著减少整体运行时间。

在本文中，您将学习到WebAssembly线程的工作原理以及C、C++ 和 Rust 等语言如何编写多线程应用。

## WebAssembly 线程如何工作
WebAssembly 线程不是一个单独的功能，而是多个组件的集合，它们使得 WebAssembly 应用可以使用传统的多线程模式。

### Web Workers

第一个组件是您在JavaScript 中经常遇到并且喜爱的[Worker](https://developer.mozilla.org/docs/Web/API/Web_Workers_API/Using_web_workers)。WebAssembly 线程使用`new Worker`函数来创建一个新的底层线程。每个线程加载一段JavaScript 胶水代码，然后主线程使用[Worker#postMessage](https://developer.mozilla.org/docs/Web/API/Worker/postMessage)方法与其他线程共享已编译的[WebAssembly.Module](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module)以及共享的[WebAssembly.Memory](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory)（见下文）。这样所有这些线程在相同的共享内存上运行相同的 WebAssembly 代码，而无需再次通过 JavaScript。

Web Workers 已经存在十多年了，得到了广泛的支持，并且不需要任何特殊的设置。

### SharedArrayBuffer

WebAssembly 内存由JavaScript API 中的[WebAssembly.Memory](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory)对象表示。默认情况下`WebAssembly.Memory`是围绕只能由单个线程访问的原始字节缓冲区[ArrayBuffer](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)的封装。

```
> new WebAssembly.Memory({ initial:1, maximum:10 }).buffer
ArrayBuffer { … }
```

为了支持多线程，`WebAssembly.Memory`也提供共享变量的功能。当通过 JavaScript API 或由 WebAssembly 二进制文件本身使用标志`shared`创建时，它由[SharedArrayBuffer](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)封装而成。与`ArrayBuffer`不同，它可以与其他线程共享，允许读取或修改操作。

```
> new WebAssembly.Memory({ initial:1, maximum:10, shared:true }).buffer
SharedArrayBuffer { … }
```

和通常用于主线程和 Web Workers 之间通信的[postMessage](https://developer.mozilla.org/docs/Web/API/Worker/postMessage)不同，[SharedArrayBuffer](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)不需要复制数据，甚至不需要等待事件循环来发送和接收消息。相比传统事件，所有线程几乎可以立即看到任何更改，使`SharedArrayBuffer`成为更好的选择。

`SharedArrayBuffer`有着复杂的历史。它最初在 2017 年中期在多个浏览器中支持，但由于发现[Spectre 漏洞](https://developers.google.com/web/updates/2018/02/meltdown-spectre)而不得不在 2018 年初被禁用。Spectre漏洞中的数据提取依赖于计时攻击——测量特定代码段的执行时间。为了使这种攻击更加困难，浏览器降低了标准计时 API 的精度，如`Date.now`和`performance.now`。但是，共享内存与在单独线程中运行的简单计数器循环相结合[也是获得高精度计时的一种非常可靠的方法](https://github.com/tc39/security/issues/3)，并且在不显着限制运行时性能的情况下更难解决。

Chrome 68（2018 年中）`SharedArrayBuffer`通过利用[站点隔离](https://developers.google.com/web/updates/2018/07/site-isolation)再次重新启用该功能，站点隔离将不同的网站置于不同的进程中，像 Spectre 这样的旁路攻击变得更加困难。但是，这种缓解措施仍然仅限于 Chrome 桌面，因为站点隔离是一项相当耗费资源的功能，无法默认为低内存移动设备上的所有站点启用，其他供应商也尚未实现。

快进到 2020 年，Chrome 和 Firefox 都实现了站点隔离，一个标准实现方式是网站可选择性的通过[COOP 和 COEP headers](https://web.dev/coop-coep/)  开启该功能。为所有网站都启用站点隔离成本会比较高，但是选择加入机制可以让这项特性在低功率设备上使用。要选择加入站点隔离，可以将以下标题添加到服务器配置的主文档中：

```
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

一旦您选择加入，您就可以访问`SharedArrayBuffer`（包括由`SharedArrayBuffer`支持的  `WebAssembly.Memory`）、精确计时器、内存测量和其他出于安全原因需要隔离的API。查看[使用 COOP 和 COEP 将您的网站“跨域隔离”](https://web.dev/coop-coep/)了解更多详细信息。

### **WebAssembly 原子[#](https://web.dev/webassembly-threads/#webassembly-atomics)**

虽然 SharedArrayBuffer 允许每个线程读取和写入同一块内存，但为了正确通信，您需要确保它们不会同时执行冲突操作。例如，一个线程可能开始从共享地址读取数据，而另一个线程正在写入数据，因此第一个线程现在将获得损坏的结果。此类错误称为竞争状态。为了防止竞争状态，您需要以某种方式顺序执行这些操作。这就是原子操作的用武之地。

[WebAssembly 原子](https://webassembly.github.io/threads/core/syntax/instructions.html#atomic-memory-instructions)是 WebAssembly 指令集的扩展，它允许“原子地”读取和写入小数据单元（通常是 32 位和 64 位整数）。也就是说，WebAssembly原子以某种方式保证没有两个线程同时读取或写入同一个单元，从而在低级别防止此类冲突。此外，WebAssembly 原子包含另外两种指令类型——“wait”和“notify”——它们允许一个线程在共享内存中的给定地址上休眠（“wait”），直到另一个线程通过“notify”将其唤醒。

所有更高级别的同步事件，包括通道、互斥锁和读写锁都建立在这些指令之上。

## 如何使用 WebAssembly 线程

### **特征检测[#](https://web.dev/webassembly-threads/#feature-detection)**

WebAssembly atomics 和`SharedArrayBuffer`是相对较新的功能，尚未在所有支持 WebAssembly 的浏览器中可用。您可以在[webassembly 路线图](https://webassembly.org/roadmap/)上找到哪些浏览器支持新的 WebAssembly 功能。

为了确保所有用户都可以加载您的应用程序，您需要通过构建两个不同版本的 Wasm 来实现渐进式增强——一个支持多线程，一个不支持多线程。然后根据特征检测结果加载支持的版本。要在运行时检测 WebAssembly 线程支持，请使用[wasm-feature-detect 库](https://github.com/GoogleChromeLabs/wasm-feature-detect)并像这样加载模块：

```
import { threads } from 'wasm-feature-detect';

const hasThreads = await threads();

const module = await (
  hasThreads
    ? import('./module-with-threads.js')
    : import('./module-without-threads.js')
);

// …now use `module` as you normally would
```

现在让我们来看看如何构建 WebAssembly 模块的多线程版本。

### **C**

在 C语言 中，尤其是在类 Unix 系统上，使用线程的常用方法是使用phread库提供的[POSIX 线程](https://en.wikipedia.org/wiki/POSIX_Threads)。Emscripten提供了一套构建在`pthread`、共享内存和原子操作之上的[API 兼容实现](https://emscripten.org/docs/porting/pthreads.html)，因此相同的代码可以在 Web 上运行而无需更改。

我们来看一个例子：

example.c：

```
#include <stdio.h>
#include <unistd.h>
#include<pthread.h>

void *thread_callback(void *arg)
{
    sleep(1);
    printf("Inside the thread: %d\n", *(int *)arg);
    return NULL;
}

int main()
{
    puts("Before the thread");

    pthread_t thread_id;
    int arg = 42;
pthread_create(&thread_id,NULL, thread_callback,&arg);

pthread_join(thread_id,NULL);

    puts("After the thread");

    return 0;
}
```

这里的`pthread`库通过`pthread.h`引用. 您还可以看到几个处理线程的关键函数。

[pthread_create](https://man7.org/linux/man-pages/man3/pthread_create.3.html)将创建一个后台线程。它需要一个目标来存储线程句柄、一些线程创建时的属性（这里没有传递任何属性，所以它只是`NULL`），要在新线程中执行的回调（这里是`thread_callback`），以及一个可选的参数指针来传递给该回调以供您可以从主线程共享一些数据使用——在这个例子中，我们共享一个指向变量的指针`arg`。

[pthread_join](https://man7.org/linux/man-pages/man3/pthread_join.3.html)可以在后面随时调用以等待线程执行完毕，并获取回调返回的结果。它接受先前分配的线程句柄以及存储结果的指针。在我们的例子中，没有任何结果，因此该函数将 `NULL`作为参数。

要使用 Emscripten 编译使用线程的样例代码，您需要调用`emcc`并传递`-pthread`参数，就像在其他平台上使用 Clang 或 GCC 编译相同的代码一样：

```
emcc -pthread example.c -o example.js
```

但是，当您尝试在浏览器或 Node.js 中运行它时，您会看到警告，然后程序将被挂起：

```
Before the thread
Tried to spawn a new thread, but the thread pool is exhausted.
This might result in a deadlock unless some threads eventually exit or the code
explicitly breaks out to the event loop.
If you want to increase the pool size, use setting `-s PTHREAD_POOL_SIZE=...`.
If you want to throw an explicit error instead of the risk of deadlocking in those
cases, use setting `-s PTHREAD_POOL_SIZE_STRICT=2`.
[…hangs here…]
```

发生了什么？这个问题是因为，Web 上大部分耗时的 API 都是异步的，并且依赖于事件循环来执行。与传统环境相比，这种限制是一个重要的区别，在传统环境中，应用程序通常以同步、阻塞的方式运行 I/O。如果您想了解更多信息，请查看关于[使用来自 WebAssembly 的异步 Web API](https://web.dev/asyncify/)的博客文章。

在我们的例子中，代码同步调用`pthread_create`以创建一个后台线程，然后通过另一个同步调用`pthread_join`来等待后台线程完成执行。但是，使用 Emscripten 编译此代码时使用的 Web Workers 是异步的。所以发生的事情是，`pthread_create`只会在下一次事件运行时创建一个新的 Worker 线程，但是`pthread_join`为了防止Worker被重复创建，会立即阻塞该事件直到Worker被创建。这是一个[死锁](https://en.wikipedia.org/wiki/Deadlock)的经典例子。

解决这个问题的一种方法是在程序开始之前创建一个Woker池。当`pthread_create`被调用时，它可以从池中取出一个随时可用的 Worker，在其后台线程上运行传入的回调，之后将 Worker 返还到池中。因此只要池足够大，所有这些都可以同步完成，而不会出现任何死锁。

这正是 Emscripten 提供的[-s PTHREAD_POOL_SIZE=...](https://emsettings.surma.technology/#PTHREAD_POOL_SIZE)选项所支持的。它允许指定多个线程、一个固定的数量或者是一个 JavaScript 表达式，比如[navigator.hardwareConcurrency](https://developer.mozilla.org/docs/Web/API/NavigatorConcurrentHardware/hardwareConcurrency)会创建与 CPU 内核数量一样多的线程。当您的代码可以扩展到任意数量的线程时，后一个选项很有用。

在上面的示例中，只创建了一个线程，因此无需保留所有内核，使用`-s PTHREAD_POOL_SIZE=1`就足够了

```
emcc -pthread -s PTHREAD_POOL_SIZE=1 example.c -o example.js
```

这一次，当您运行时，就成功了：

```
Before the thread
Inside the thread: 42
After the thread
Pthread 0x701510 exited.
```

但是还有另一个问题：代码示例中的`sleep(1)`看到了吗？它在线程回调中执行，意味着脱离主线程，所以应该没问题，对吧？然而，并不是。

当`pthread_join`被调用时，它必须等待线程执行完成，这意味着如果创建的线程正在执行长时间运行的任务——在我们的示例中，休眠 1 秒——那么主线程也将不得不阻塞相同的时间直到结果返回。在浏览器中执行此 JS 时，它将阻塞 UI 线程 1 秒，直到线程回调返回。这会导致很糟糕的用户体验。

有几个解决方案：

- `pthread_detach`
- `-s PROXY_TO_PTHREAD`
- 自定义 Worker 和 Comlink

### **pthread_detach [#](https://web.dev/webassembly-threads/#pthread_detach)**

首先，如果您只想要在主线程之外运行一些任务，而不需要等待结果，您可以使用[pthread_detach](https://man7.org/linux/man-pages/man3/pthread_detach.3.html)代替`pthread_join`。这将使线程回调在后台运行。如果您使用这个方案，可以使用参数 [-s PTHREAD_POOL_SIZE_STRICT=0](https://emsettings.surma.technology/#PTHREAD_POOL_SIZE_STRICT)关闭告警。

### **PROXY\_TO\_PTHREAD [#](https://web.dev/webassembly-threads/#proxy_to_pthread)**

其次，如果您正在编译 C 应用程序而不是函数库，则可以使用[-s PROXY_TO_PTHREAD](https://emsettings.surma.technology/#PROXY_TO_PTHREAD)选项，除了应用程序本身创建的任何嵌套线程之外，它还会将主应用程序代码分配到单独的线程。这样，主代码可以随时安全地阻塞，而不会冻结 UI。顺便说一句，当使用这个选项时，您也不必预先创建线程池——相反，Emscripten 可以利用主线程来创建新的底层 Worker，然后在`pthread_join` 中阻塞辅助线程，而不会死锁。

### Comlink**[#](https://web.dev/webassembly-threads/#comlink)**

如果您正在处理一个库并且仍然需要阻塞，您可以创建自己的 Worker，导入 Emscripten 生成的代码并使用[Comlink](https://github.com/GoogleChromeLabs/comlink)将其公开给主线程。这样主线程将能够调用公开的方法作为异步函数，这样也可以避免阻塞 UI。

在像前面的例子这样的简单应用程序中`-s PROXY_TO_PTHREAD`是最好的选择：

```
emcc -pthread -s PROXY_TO_PTHREAD example.c -o example.js
```

### **C++ [#](https://web.dev/webassembly-threads/#c++)**

相比于C，所有相同的警告和逻辑都以相同的方式适用于 C++。使用C++唯一获得的是访问更高级别的 API，如[std::thread](https://en.cppreference.com/w/cpp/thread/thread)和[std::async](https://en.cppreference.com/w/cpp/thread/async)，它们都使用前面提到的`pthread`库。

所以上面的例子可以用更常用的 C++ 重写，如下所示：

example.cpp：

```
#include <iostream>
#include<thread>
#include <chrono>

int main()
{
    puts("Before the thread");

    int arg = 42;
    std::threadthread([&](){
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "Inside the thread: " << arg << std::endl;
    });

    thread.join();

    std::cout << "After the thread" << std::endl;

    return 0;
}
```

当使用类似的参数编译和执行时，它的行为C的示例相同：

```
emcc -std=c++11 -pthread -s PROXY_TO_PTHREAD example.cpp -o example.js
```

输出：

```
Before the thread
Inside the thread: 42
Pthread 0xc06190 exited.
After the thread
Proxied main thread 0xa05c18 finished with return code 0. EXIT_RUNTIME=0 set, so
keeping main thread alive for asynchronous event operations.
Pthread 0xa05c18 exited.
```

### Rust**[#](https://web.dev/webassembly-threads/#rust)**

与 Emscripten 不同，Rust 没有专门的 Web 编译目标，而是提供了`wasm32-unknown-unknown`作为通用 WebAssembly 编译目标。

如果 Wasm 旨在用于 Web 环境，与 JavaScript API 的任何交互都留给外部库和工具，如[wasm-bindgen](https://rustwasm.github.io/docs/wasm-bindgen/)和[wasm-pack](https://rustwasm.github.io/docs/wasm-pack/)。不幸的是，这意味着标准库不会感知 Web Workers 和相关 API，例如[std::thread](https://doc.rust-lang.org/std/thread/)在编译为 WebAssembly 时将无法工作。

幸运的是，Rust生态系统的大部分都依赖于更高级别的库来处理多线程。在更高级别的视角，从各个有差异的平台中做抽象会比较简单。

值得一提的是，[Rayon](https://crates.io/crates/rayon)是 Rust 中最流行的数据并行方案。它允许您在常规迭代器上采用方法链，并且通常只需更改一行，就可以在所有可用线程上并行运行而不是按顺序运行的方式执行它们。例如：

```
pub fn sum_of_squares(numbers: &[i32]) -> i32 {
  numbers
  .iter()
  .map(|x| x * x)
  .sum()
}
```

`iter`修改为`par_iter`

```
pub fn sum_of_squares(numbers: &[i32]) -> i32 {
  numbers
  .par_iter()
  .map(|x| x * x)
  .sum()
}
```


通过这个小改动，代码将拆分输入数据，`x * x`在并行线程中计算部分求和，最后将这些部分结果加在一起。

为了适应不支持`std::thread`的平台，Rayon 允许定义用于生成和退出线程的自定义钩子。

[wasm-bindgen-rayon](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon)利用这些钩子来生成 WebAssembly 线程作为 Web Workers。要使用它，您需要将其添加为依赖项并按照[文档中](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon#setting-up)描述的配置步骤进行操作。上面的例子最终看起来像这样：

```
pubusewasm_bindgen_rayon::init_thread_pool;

#[wasm_bindgen]
pub fn sum_of_squares(numbers: &[i32]) -> i32 {
  numbers
  .par_iter()
  .map(|x| x * x)
  .sum()
}
```

完成后，生成的 JavaScript 将包含一个额外的`initThreadPool`函数。此函数将创建一个Worker 池，Rayon 生成的多线程操作可以在程序的整个生命周期中使用。

这个池机制与前面解释的 Emscripten 中的选项`-s PTHREAD_POOL_SIZE=...`类似，也需要在主代码之前进行初始化以避免死锁：

```
import init,{ initThreadPool, sum_of_squares}from'./pkg/index.js';

// 常规 wasm-bindgen 初始化.
await init();

// 使用给定的线程数量进行线程池初始化
// (如果您想使用所有内核可以传递这个参数 `navigator.hardwareConcurrency`)
awaitinitThreadPool(navigator.hardwareConcurrency);

// ...现在您可以像往常一样引用定义好的函数
console.log(sum_of_squares(new Int32Array([1, 2, 3]))); // 14
```

请注意，关于阻塞主线程的相同[警告](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon#caveats)也适用于此。即便是上文`sum_of_squares`的例子，仍然需要阻塞主线程直到其他线程的部分结果。

等待可能很短，也可能很长，这取决于迭代器的复杂性和可用线程的数量，但是，为了安全起见，浏览器引擎会主动阻止主线程完全阻塞，这样的代码会抛出错误。所以，您应该创建一个 Worker，在`wasm-bindgen`那里导入生成的代码，并使用[Comlink](https://github.com/GoogleChromeLabs/comlink) 库将其 API暴露到主线程。

这里有更详细的[wasm-bindgen-rayon示例](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/tree/main/demo)

- [线程的特征检测。](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/blob/e13485d6d64a062b890f5bb3a842b1fe609eb3c1/demo/wasm-worker.js#L27)
- [构建同一个 Rust 应用程序的单线程和多线程版本。](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/blob/e13485d6d64a062b890f5bb3a842b1fe609eb3c1/demo/package.json#L4-L5)
- [在 Worker 中加载 wasm-bindgen 生成的 JS+Wasm。](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/blob/e13485d6d64a062b890f5bb3a842b1fe609eb3c1/demo/wasm-worker.js#L28-L31)
- [使用 wasm-bindgen-rayon 初始化线程池。](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/blob/e13485d6d64a062b890f5bb3a842b1fe609eb3c1/demo/wasm-worker.js#L32)
- 使用Comlink将 [Worker 的 API](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/blob/e13485d6d64a062b890f5bb3a842b1fe609eb3c1/demo/wasm-worker.js#L44-L46)公开给[主线程](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon/blob/e13485d6d64a062b890f5bb3a842b1fe609eb3c1/demo/index.js#L25-L29)

## 真实场景
我们在[Squoosh.app](https://squoosh.app/) 使用 WebAssembly 线程进行客户端图像压缩——特别是对于 AVIF (C++)、JPEG-XL (C++)、OxiPNG (Rust) 和 WebP v2 (C++) 格式。多亏了多线程，我们可以得到 1.5-3倍 加速（每个编解码器的确切比率不同），并且能够通过将 WebAssembly 线程与[WebAssembly SIMD](https://v8.dev/features/simd)结合来进一步优化！

Google Earth 是另一个值得关注的服务，它在其[网络版本](https://earth.google.com/web/)中使用 WebAssembly 线程。

[FFMPEG.WASM](https://ffmpegwasm.github.io/)是流行的[FFmpeg](https://www.ffmpeg.org/)多媒体工具链的 WebAssembly 版本，它使用 WebAssembly 线程直接在浏览器中高效地编码视频。

还有更多令人兴奋的使用 WebAssembly 线程的示例。请查看这些示例并在您自己的多线程应用和库函数中使用起来吧！

