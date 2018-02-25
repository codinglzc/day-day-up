# Intel Pin 介绍
Pin是Intel公司开发的动态二进制插桩框架，可以用于创建基于动态程序分析工具，支持IA-32和x86-64指令集架构，支持windows和linux。

简单说就是Pin可以监控程序的每一步执行，提供了丰富的API，可以在二进制程序运行过程中插入各种函数，比如说我们要统计一个程序执行了多少条指令，每条指令的地址等信息。显然，这样我们对程序完全掌握了以后是可以做很多事的。比如对程序的内存使用检测，对程序的性能评估等。

# 如何使用Pin插桩
### Pin
认识Pin的最好方法是认为它是一个"_just in time_"(**JIT**)编译器。这个编译器的输入不是字节码而是普通的可执行文件。Pin截获这个可执行文件的第一条指令，产生新的代码序列。接着将控制流程转移到这个产生的序列。产生的序列基本上跟原来的序列是一样的，但是Pin保证在一个分支结束后重新获得控制权。重新获得控制权之后，Pin为分支的目标产生代码并且执行。Pin通过将所有产生的代码放在内存中，以便于重新使用这些代码并且可以直接从一个分支跳到另一个分支，这提高了效率。

在JIT模式，执行的代码都是Pin生成的代码。原始代码仅仅是用来参考。当生成代码时，Pin给用户提供了插入自己代码的机会（插桩）。

Pin的桩代码都会被实际执行的，不论他们位于哪里。大体上，对于条件分支存在一些异常，比如，如果一个指令从不执行，它将不会被插入桩函数。

### Pintools
概念上说，插桩包括两个组件：

* 决定在哪里插入什么代码的机制
* 插入点执行的代码

这两个组件分别是桩(_instrumentation_)和分析(_analysis_)代码。两个组件都在一个单独的可执行体，即Pintool。Pintools可以认为是在Pin中的插件，它能够修改生成代码的流程。

Pintool在Pin中注册一些桩回调函数，每当Pin生成新的代码时就调用回调函数。桩回调函数例程代表桩(_instrumentation_)组件。它可以检查将要生成的代码，捕获它的静态属性，并且决定是否需要以及在哪里插入调用来分析(_analysis_)函数。

分析(_analysis_)函数收集关于程序的数据。Pin保证整数和浮点指针寄存器的状态，在必要时会被保存和恢复，且允许传递参数给(分析)函数。

Pintool也可以注册一些事件通知回调例程(_routines_)，比如线程创建或fork，这些回调大体上用于数据收集或者初始化与清理。

### 意见
由于Pintool类似插件一样工作，所以它必须运行在Pin和被插桩程序的地址空间。这样，Pintool就能够访问程序的所有数据。它也跟程序共享文件描述符和其它进程信息。

Pin和Pintool从第一条指令开始控制程序。对于与共享库一起编译的可执行文件，这意味着动态加载和共享库对Pintool可见。

**当编写tools时，最重要的是调整分析代码而不是桩代码。因为桩代码只执行一次，但分析代码执行许多次。**

### 插桩粒度
如上所诉，Pin的插桩是JIT（实时）的。插桩发生在一段代码序列执行之前。我们把这种模式叫做跟踪插桩(_trace instrumentation_)。

跟踪插桩(_trace instrumentation_)让Pintool在可执行代码每一次执行时都能进行监视和插桩。trace通常开始于选中的分支目标并结束于一个无条件分支，包括调用(call)和返回(return)。Pin能够保证trace只在最上层有一个入口，但是可以有很多出口。如果在一个trace中发生分支，Pin从分支目标开始构造一个新的trace。Pin根据基本块(basic blocks, BBLs)分割trace。一个基本块是一个有唯一入口和出口的指令序列。基本块中的分支会开始一个新的trace也即一个新的基本块。通常为每个基本块而不是每条指令插入一个分析调用。减少分析调用的次数可以提高插装的效率。trace插装通过[TRACE_AddInstrumentFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__TRACE__BASIC__API.html#g0261da0abe384db4e2f97cd31cf986f7) API call实现。

注意，虽然Pin从程序执行中动态发现执行流，Pin的BBL与编译原理中的BBL定义不同。如，考虑生成下面的`switch`语句：
```c
    switch(i)
    {
        case 4: total++;
        case 3: total++;
        case 2: total++;
        case 1: total++;
        case 0:
        default: break;
    }
```
它将会产生如下的指令（在IA-32架构上）
```
.L7:
        addl    $1, -4(%ebp)
.L6:
        addl    $1, -4(%ebp)
.L5:
        addl    $1, -4(%ebp)
.L4:
        addl    $1, -4(%ebp)
```
在经典的基本块中，每一个`addl`指令会成为一个单指令基本块。但是当不同的`switch cases`被执行时，Pin会产生不同的BBLs(当分支进入`.L7 case`时，BBL包含以上所有的四个指令；当分支进入`.L6 case`时，BBL包含后三个指令如此类推)。这就是说Pin的BBL与书上定义的BBL不一样。例如，这里当代码分支到`.L7`时，Pin只会产生1个BBL，但是有四个经典的基本块被执行。

Pin也会拆散其他指令为BBL，这些指令往往不被期望被拆散，比如cpuid,popf,和rep前缀的指令。因为rep前缀指令被视为隐循环，如果一个rep前缀指令不止迭代一次，在第一次迭代之后将会产生一个包含一个指令的BBL，所以这种情形会产生比你预期要多的基本块。

为了方便编写Pintool，Pin还提供了指令插桩模式(_instruction instrumentation_)，让Pintool可以监视和插桩每一条指令。本质上来说这两种模式是一样的，编写Pintool时不需要在为trace的每条指令反复处理。就像在trace插桩模式下一样，特定的基本块和指令可能会被生成很多次。指令插装通过 [INS_AddInstrumentFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INS__INST__API.html#gad5fd5cdd6c1cd37e57a1264e93b0435) API call实现。

有时，进行不同粒度的插桩比trace更有用。对此，Pin提供了两种模式：镜像(_image_)和例程(_routine_)插桩。这些模式是通过缓存(_caching_)插桩要求来实现的，因此需要额外的空间开销，这些模式也称作提前(_ahead-of-time_)插桩。

镜像插桩(_Image instrumentation_)让Pintool在image第一次导入的时候对整个image进行监视和插桩。Pintool的处理范围可以是镜像中的每个块(_section，SEC_)，块中的每个例程(_routine, RTN_)，例程中的每个指令(_instruction, INS_)。插装可以在一个例程或者一条指令开始执行之前或者结束执行之后执行。镜像插装通过 [IMG_AddInstrumentFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__IMG__BASIC__API.html#g4867c865fcf260951b7d750aaaaa0007) API call实现。镜像插装依靠符号信息来判断例程的边界，因此需要在[PIN_Init](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#g078bed712e6822f3b42a7b4815ba5c0a)之前调用[PIN_InitSymbols](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#ga6749650a8dce7151075fcc9345f7bd9)。

例程插桩(_Routine instrumentation_)让Pintool在image第一次加载的时候监视和插桩整个例程。Pintool的处理范围可以是例程里的每条指令。这里没有足够的信息把指令归并成BBLs。插桩可以在一个例程或者一条指令开始执行之前或者结束执行之后执行。例程插桩使Pintool编写更方便地在镜像插桩过程中遍历image的各个sections和routines。

例程插桩通过[RTN_AddInstrumentFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__RTN__BASIC__API.html#g50b3794c06a99774ed62abec8ad3b173) API call实现。插桩在例程结束后不一定能可靠地工作，因为当最后出现调用时无法判断何时返回。

注意：在镜像插桩和例程插桩中，不可能知道一个例程是否真的被执行(因为这些插桩发生在镜像被载入时)。在跟踪插桩和指令插桩中，通过识别例程开始执行的指令，可以遍历只有被执行的例程的指令。请参考Pintool _Tests/parse_executed_rtns.cpp_。

### 托管平台支持
Pin支持所有可执行文件包括托管的二进制文件。从Pin的角度来看，托管文件是一种自修改程序。有一种方法可以使Pin区分即时编译代码(_just-in-time compiled code, Jitted code_)和所有其他动态生成的代码，并且将Jitted代码与合适的管理函数联系在一起。为了支持这个功能，运行管理托管平台的JIT compiler必须支持[Jit Profiling API](https://software.intel.com/en-us/node/544211)。

必须支持下面的功能：
* [RTN_IsDynamic()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__RTN__BASIC__API.html#gb36b828dc06d754e79780245d938dc7b) API用来识别动态生成的代码。一个函数可以被Jit Profiling API标记为动态生成的。
* 一个Pintool可以使用[RTN_AddInstrumentFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__RTN__BASIC__API.html#g50b3794c06a99774ed62abec8ad3b173) API插入Jitted例程。更多信息查看[托管平台支持](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/index.html#JitApiTools)

为了支持托管平台，以下条件必须满足：
* 设置`INTEL_JIT_PROFILER32`和`INTEL_JIT_PROFILER64`环境变量，以便占用pinjitprofiling dynamic library
  1. For Windows
  ```
  set INTEL_JIT_PROFILER32=<The Pin kit full path>\ia32\bin\pinjitprofiling.dll
  set INTEL_JIT_PROFILER64=<The Pin kit full path>\intel64\bin\pinjitprofiling.dll
  ```
  2. For Linux
  ```
  setenv INTEL_JIT_PROFILER32 <The Pin kit full path>/ia32/bin/libpinjitprofiling.so
  setenv INTEL_JIT_PROFILER64 <The Pin kit full path>/intel64/bin/libpinjitprofiling.so
  ```
* 在Pin命令行为Pin tool添加knob support_jit_api选项
```
<Pin executable> <Pin options> -t <Pintool> -support_jit_api <Other Pintool options> -- <Test application> <Test application options>
```

### Symbols
Pin利用symbol对象（SYM）提供了对函数名字的访问。symbol对象仅仅提供了在程序中的关于函数symbol的信息。其他类型的符号（如数据符号）需要通过tool独立获取。

在Windows上，可以通过`dbghelp.dll`实现这个功能。注意在被插桩的进程中使用dbghelp.dll并不安全，可能会导致死锁。一个可能的解决方案是通过一个不同的未被插桩的进程得到符号。

在Linux上，`libefl.so`或者`libdwarf.so`可以用来获取符号信息。

为了通过名字访问函数必须先调用[PIN_InitSymbols](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#ga6749650a8dce7151075fcc9345f7bd9)。更多信息请查看[SYM: Symbol Object](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__SYM__BASIC__API.html)

### 在分析例程中浮点运算的支持
Pin在执行各种分析例程时保持着程序的浮点指针状态。

`IARG_REG_VALUE`不能作为浮点指针寄存器参数传给分析例程。

### 插桩多线程应用程序
给多线程程序插桩时，多个合作线程访问全局数据时必须保证tool是线程安全的。Pin试图为tool提供传统C++程序的环境，但是在一个Pintool是不可以使用标准库的。比如，Linux tool不能使用pthread，Windows tool不能使用Win32API管理线程。作为替代，应该使用Pin提供的锁和线程管理API。更多请查看[LOCK: Locking Primitives](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__LOCK.html)和[Pin Thread API](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__THREAD__API.html).

Pintools在插入桩函数时，不需要添加显示的锁，因为Pin是在得到内部锁(VM lock)之后执行这些函数的。然而，Pin并行执行分析和替代函数，所以Pintools如果这些例程访问全局变量，可能需要加锁。

Linux上的Pintools需要注意在分析函数或替代函数中使用C/C++标准库函数，因为链接到Pintools的C/C++标准函数不是线程安全的。一些简单C/C++函数本身是线程安全的，在调用时不需要加锁。但是，Pin没有提供一个线程安全函数的列表。如果有怀疑，需要在调用库函数的时候加锁。特别的，errno变量不是多线程安全的，所以使用这个变量的tool需要提供自己加锁。注意这些限制仅存在Unix平台，这些库函数在Windows上是线程安全的。

Pin可以在线程开始和结束的时候插入回调函数(see _PIN_AddThreadStartFunction_和_PIN_AddThreadFiniFunction_)。这为Pintool提供了一个比较方便的地方分配和操作线程局部数据。

Pin也提供了一个分析函数的参数(ARG_THREAD_ID)，用于传递Pin指定的线程ID给调用的线程。这个ID跟操作系统的线程ID不同，它是一个从0开始的小整数，可以作为线程数据或是Pin用户锁的索引。更多信息例子请看[Instrumenting Threaded Applications](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/index.html#MallocMT)。

除了Pin线程ID，Pin API提供了有效的线程局部存储(thread local storage, TLS），提供了分配新的TLS key并将它关联到指定数据的清理函数的选项。进程中的每个线程都能够在自己的槽中存储和取得对应key的数据。所有线程中key对应的初始化的value都是NULL。更多信息例子请看[Using TLS](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/index.html#InscountTLS)。

伪共享(False sharing)发生在多个线程访问相同的cache line的不同部分,至少其中之一是写。为了保持内存一致性，计算机必须将一个CPU的缓存拷贝到另一个，即使数据没有真正共享。可以通过将关键数据结构对其到一个cache line大小上或者重新排列数据结构避免伪共享。更多信息例子请看[Using TLS](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/index.html#InscountTLS)。

### 在多线程应用程序中避免死锁
因为Pin,Pintool和application可能都会要求或释放锁，Pin tool的开发者必须小心避免死锁。死锁经常发生在两个线程以不同的顺序要求同样的锁。例如，线程A要求lock L1，接着要求L2,线程B要求lock L2，接着要求L1。如果线程A得到了L1，等待L2，同时线程B得到了L2，等待L1，这就导致了死锁。为了避免死锁,Pin为必须获得的锁强加了一个层次结构。在Pintool获得任何锁之前，Pin先获得自己的内部锁(e.g. via [PIN_GetLock()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__LOCK.html#gc7b6e6ed4fb7e14452b85b96d5c42c10))。我们假设application将会在这个层次结构的顶端获得锁(i.e. 在Pin获得其内部锁之前)。下面的图展示了这种结构：
```
Application locks -> Pin internal locks -> Tool locks
```
Pin tool开发者在设计他们自己的锁时不应该破坏这个锁层次结构。下面是基本的指导原则：
* 如果Pintool在一个Pin回调中获得任何锁，它在从这个回调中返回时必须释放这些锁。从Pin内部锁看来，在Pin回调中占有一个锁违背了这个层次结构。
* 如果Pintool在一个分析函数中获得任何锁，它从这个分析函数中返回时必须释放这些锁。从Pin内部锁和程序自身看来，在分析函数中占有一个锁违背了这个层次结构。
* 如果Pintool在一个Pin回调或者分析函数中调用Pin API，它不应该在调用API的时候占有任何锁。一些Pin API使用了内部Pin锁，所以在调用这些API时占有一个tool锁违背了这个层次结构。
* 如果Pintool在分析函数里面调用了Pin API，它可能需要获得Pin客户锁时调用[PIN_LockClient()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#gc5d4cd777e5c34bb760ea6768e054f20)。这取决于API，需要查看特定API的更多信息。注意Pintool在调用[PIN_Lockclient()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#gc5d4cd777e5c34bb760ea6768e054f20)时，不能持有任何其他锁。

虽然上述的指导在大多数情况下已经足够，但是它们在某些特定的情形下显得比较严格。下面的指导解释了上述基本指导的放松情形：
* 在JIT模式下，Pintool可能在分析函数中获得锁而不释放它们，直到将要离开包含这个分析函数的trace时才释放锁。Pintool必须期望trace在程序还没有抛出异常的时尽早退出。任何被tool占有的锁L在程序抛出异常时，必须遵守以下规则：
  * tool必须建立一个当程序抛出异常时的处理回调，这个回调会释放之前得到的锁L。可以使用[PIN_AddContextChangeFunction()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#gfe475fc12b9060e8e45cbfccd7d592c8)建立这个回调。
  * 为了避免破坏这个层次结构，Pintool禁止在Pin回调中要求锁。
* 如果Pintool从一个分析函数中调用Pin API，如果在调用API时发生了下面情况，它需要获得并占有一个锁L：
  * 锁L不是从任何Pin回调中请求的。这避免了违背这个层次结构。
  * 被调用的Pin API不会导致程序代码被执行(e.g. [PIN_CallApplicationFunction()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#g5280354edd95efc19a837617ff63ba51))。这避免了违背这个层次结构。

# Examples
为了阐明如何编写Pintools，我们提供了一些简单的例子。

在手册上的所有例子可以在`source/tools/ManualExamples`目录下找到。

|    Pintool    |          功能          |  源文件包含知识点  |
| ------------: | :--------------------- | :--------------: |
| inscount0.cpp | 计算被执行的动态指令个数 | (**Instruction Instrumentation**)<br>如何为tool添加命令行参数，请看 [KNOB: Commandline Option Handling](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__KNOB__API.html) |
| itrace.cpp | 打印每条被执行的指令的地址(IPs) | (**Instruction Instrumentation**)<br>如何传递参数给分析函数，所有的参数类型请看 [IARG_TYPE](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INST__ARGS.html#g7e2c955c99fa84246bb2bce1525b5681) |
| pinatrace.cpp |  This tool generates a trace of all memory addresses referenced by a program. | (**Instruction Instrumentation**)<br>如何分类指令，see [here](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INS__BASIC__API.html) <br>Use [INS_InsertPredicatedCall](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INS__INST__API.html#g446df8cbefd4950b78cba7c9e7346053) instead of [INS_InsertCall](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INS__INST__API.html#g82d2ecc73dcd5e2af26fef5bd6ff7190) |
| imageload.cpp | 记录每当image加载和卸载的事件 | (**Image Instrumentation**)<br>注册回调函数使用的是[IMG_AddInstrumentFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__IMG__BASIC__API.html#g4867c865fcf260951b7d750aaaaa0007)和[IMG_AddUnloadFunction](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__IMG__BASIC__API.html#g8f27bbc6bbc60d15df64ca5e81b19c30)
| inscount1.cpp | 功能与inscount0一样，但比inscount0更高效 | (**Trace Instrumentation**)<br>分析函数每BBL才执行一次，而不是每条指令执行一次，所以开销小 |
| proccount.cpp | 计算routine被执行的次数，以及每个routine中被执行的指令个数 | (**Routine Instrumentation**)<br> |
| safecopy.cpp | copy the specified number of bytes from a source memory region to a destination memory region | Using [PIN_SafeCopy()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__PIN__CONTROL.html#g1e6d08632dccfcd10aec3fbdd2562899) allows the tool to read or write the application's values for these fields. |
| invocation.cpp | 分析函数的调用顺序取决于插入动作([IPOINT](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INST__ARGS.html#ga46cc1807fc61addd9afe69ee6736a21))和调用顺序([CALL_ORDER](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__INST__ARGS.html#g70f950adfb17bcd687fe356ed198bd2e)) | The example below illustrates this behavior by instrumenting all return instructions in three different ways. Additional examples can be found in source/tools/InstrumentationOrderAndVersion. |
| malloctrace.cpp | 获取某个函数的参数和返回值 | Using the [RTN_InsertCall()](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/group__RTN__BASIC__API.html#g006ef964b9e6e4d8e7880231e216344a) function, you can specify the arguments of interest. |
| malloc_mt.cpp | 给多线程应用程序插桩 | [Instrumenting Threaded Applications](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/index.html#MallocMT) |
| inscount_tls.cpp | 使用线程局部存储API | The example below demonstrates how to use **thread local storage (TLS)** APIs.

### 回调函数
[Callbacks](https://software.intel.com/sites/landingpage/pintool/docs/97503/Pin/html/index.html#CALLBACK)

