## 目录

### [引言](preface.md)

### 第一部分: 基础

#### 第一章 程序基础

<!-- 内存布局？ -->

#### 第二章 并发与分布式计算

<!-- - [并发] -->

#### 第三章 排队与调度理论

<!-- - [排队理论引导]()
- [工作窃取调度](papers/sched/work-steal-sched.md)
- [调度理论](4-sched/theory.md) -->

#### 第四章 内存管理工程

<!-- - 垃圾回收统一理论 -->

<!-- CPU 架构与操作系统? -->

<!-- - [Linux 系统调用]
- [Plan 9 汇编](appendix/asm.md) -->

#### [第五章 Go 程序生命周期](part1basic/ch05boot/readme.md)

- [5.1 程序引导](part1basic/ch05boot/boot.md)
    + 入口
    + 引导
      + 步骤1: runtime.check
      + 步骤2: runtime.args
      + 步骤3: runtime.osinit
      + 步骤4: runtime.schedinit
      + 步骤5: runtime.newproc
      + 步骤6: runtime.mstart
    + 总结
    + 进一步阅读的参考文献
- [5.2 初始化概览](part1basic/ch05boot/init.md)
    + CPU 相关信息的初始化
    + 运行时算法初始化
    + 模块链接初始化
    + 核心组件的初始化
    + 总结
- [5.3 主 goroutine](part1basic/ch05boot/main.md)
    + 概览
    + pkg.init 顺序
    + 何去何从？

### [第二部分：运行时机制](part2runtime/readme.md)

#### [第六章 调度器](part2runtime/ch06sched/readme.md)

- [6.1 基本结构](part2runtime/ch06sched/basic.md)
    + 工作线程的暂止和复始
    + 主要结构
      + M 的结构
      + P 的结构
      + G 的结构
      + 调度器 sched 的结构
      + 总结
      + 进一步阅读的参考文献
- [6.2 调度器初始化](part2runtime/ch06sched/init.md)
    + M 的初始化
    + P 的初始化
      + GOMAXPROCS
    + G 的初始化
      + 一些细节
    + 总结
- [6.3 调度循环](part2runtime/ch06sched/exec.md)
    + 执行前的准备
      + mstart 和 mstart1
      + M/P 的绑定
      + M 的暂止和复始
    + 核心调度
      + 偷取工作
      + M 的唤醒
      + M 的创生
        + 系统线程的创建 (darwin)
        + 系统线程的创建 (linux)
      + M/G 解绑
      + M 的死亡
    + 总结
    + 进一步阅读的参考文献
- [6.4 系统监控](part2runtime/ch06sched/sysmon.md)
    + 监控循环
    + 总结
- [6.5 线程管理](part2runtime/ch06sched/thread.md)
    + LockOSThread
    + UnlockOSThread
    + lockedg/lockedm 与调度循环
    + 模板线程
    + 总结
    + 进一步阅读的参考文献
- [6.6 信号处理机制](part2runtime/ch06sched/signal.md)
    + 信号与软中断
    + 初始化
    + 获取原始信号屏蔽字
    + 初始化信号栈
    + 初始化信号屏蔽字
    + 信号处理
    + Extra M
    + 总结
    + 进一步阅读的参考文献
- [6.7 执行栈管理](part2runtime/ch06sched/stack.md)
    + goroutine 栈结构
    + 执行栈初始化
    + G 的创生
    + 执行栈的分配
      + 小栈分配
      + 大栈分配
      + 堆上分配
    + 执行栈的伸缩
      + 分段标记
      + 栈的扩张
      + 栈的拷贝
      + 栈的收缩
    + 总结
- [6.8 协作与抢占](part2runtime/ch06sched/preemptive.md)
    + 协作式调度
      + 主动让权: Gosched
      + 主动弃权: 栈扩张与抢占标记
      + 被动弃权: 阻塞监控
    + 抢占式调度
    + 总结
    + 进一步阅读的参考文献
- [6.9 运行时同步原语](part2runtime/ch06sched/sync.md)
    + runtime.note
      + 结构
      + noteclear
      + notesleep, notesleep
      + notewakeup
      + notetsleepg
    + runtime.mutex
      + 结构
      + lock
      + unlock
    + semaphore
    + 总结
- [6.10 过去、现在与未来](part2runtime/ch06sched/history.md)
    + 演进史
    + 改进展望
      + 非均匀访存感知的调度器设计
    + 总结
    + 进一步阅读的参考文献

#### [第七章 内存分配器](part2runtime/ch07alloc/readme.md)

- [7.1 基本知识](part2runtime/ch07alloc/basic.md)
    + 主要结构
    + Arena
      + heapArena
      + arenaHint
    + mspan
    + mcache
    + mcentral
    + mheap
    + 分配概览
    + 分配入口
      + 小对象分配
      + 微对象分配
      + 大对象分配
    + 总结
    + 进一步阅读的参考文献
- [7.2 组件](part2runtime/ch07alloc/component.md)
    + fixalloc
      + 结构
      + 初始化
      + 分配
      + 回收
    + linearAlloc
    + mcache
      + 分配
      + 释放
      + per-P? per-M?
    + 其他
      + memclrNoHeapPointers
    + 系统级内存管理调用
- [7.3 初始化](part2runtime/ch07alloc/init.md)
- [7.4 大对象分配](part2runtime/ch07alloc/largealloc.md)
    + 从堆上分配
    + 从操作系统申请
- [7.5 小对象分配](part2runtime/ch07alloc/smallalloc.md)
    + 从 mcache 获取
    + 从 mecentral 获取
    + 从 mheap 获取
- [7.6 微对象分配](part2runtime/ch07alloc/tinyalloc.md)
- [7.7 内存统计](part2runtime/ch07alloc/mstats.md)
- [7.8 过去、现在与未来](part2runtime/ch07alloc/history.md)

#### [第八章 垃圾回收器](part2runtime/ch08GC/readme.md)

- [8.1 基本知识](part2runtime/ch08GC/basic.md)
    + 并发三色回收一瞥
    + 内存模型
    + 编译标志 `go:nowritebarrier`、`go:nowritebarrierrec` 和 `go:yeswritebarrierrec`
    + 进一步阅读的参考文献
- [8.2 初始化](part2runtime/ch08GC/init.md)
- [8.3 标记清扫思想](part2runtime/ch08GC/vanilla.md)
    + 标记清扫算法
    + 进一步阅读的参考文献
- [8.4 屏障技术](part2runtime/ch08GC/barrier.md)
    + 三色不变性原理
      + 强、弱不变性
      + 赋值器的颜色
      + 新分配对象的颜色
    + 赋值器屏障技术
      + 灰色赋值器的 Dijkstra 插入屏障
      + 黑色赋值器的 Yuasa 删除屏障
    + 混合写屏障
      + 基本思想
      + 可靠性、完备性和有界性的证明
      + 实现细节
    + 总结
    + 进一步阅读的参考文献
- [8.5 并发标记清扫](part2runtime/ch08GC/concurrent.md)
    + 并发标记
    + 并发清扫
    + 并行三色标记
    + 进一步阅读的参考文献
- [8.6 标记过程](part2runtime/ch08GC/mark.md)
- [8.7 清扫过程](part2runtime/ch08GC/sweep.md)
- [8.8 存活与终结](part2runtime/ch08GC/finalizer.md)
    + SetFinalizer
    + KeepAlive
- [8.9 过去、现在与未来](part2runtime/ch08GC/history.md)

#### 第九章 调试

- [9.1 race 竞争检测](part2runtime/ch09debug/race.md)
- [9.2 trace 运行时调试](part2runtime/ch09debug/trace.md)

#### 第十章 兼容与契约

- [10.1 参与运行时的系统调用: Linux 篇](part2runtime/ch10abi/syscall-linux.md)
    + 案例研究 runtime.clone
    + 运行时实现的系统调用清单
    + 进一步阅读的参考文献
- [10.2 参与运行时的系统调用: Darwin 篇](part2runtime/ch10abi/syscall-darwin.md)
    + libcCall 调用
    + 案例研究: runtime.pthread_create
    + 运行时实现的 libc 调用清单
- [10.3 cgo](part2runtime/ch10abi/cgo.md)
    + 入口
    + cgocall
      + 原理概述: Go 调用 C
      + 实际代码
    + cgocallbackg
      + 原理概述: C 调用 Go
      + 实际代码
    + 总结
    + 进一步阅读的参考文献
- [10.4 WebAssembly](part2runtime/ch10abi/syscall-wasm.md)

### [第三部分：编译系统](part3compile/readme.md)

#### 第十一章 关键字

- [11.1 `go`](part3compile/ch11keyword/go.md)
- [11.2 `defer`](part3compile/ch11keyword/defer.md)
- [11.3 `panic` 与 `recover`](part3compile/ch11keyword/panic.md)
    + gopanic 和 gorecover
    + 总结
- [11.4 `map`](part3compile/ch11keyword/map.md)
- [11.5 `chan` 与 `select`](part3compile/ch11keyword/chan.md)
    + channel 的本质
      + 基本使用
      + channel 的创生
      + 向 channel 发送数据
      + 从 channel 接收数据
      + channel 的死亡
    + select 的本质
      + 随机化分支
      + 发送数据的分支
      + 接收数据的分支
    + 总结
    + 进一步阅读的参考文献

- [11.6 `interface`](part3compile/ch11keyword/interface.md)

#### 第十二章 泛型

- [12.1 泛型的历史及其演化]
- [12.2 泛型的实现]

#### 第十三章 模块链接器

- [初始化](part3compile/ch13link/init.md)
- [模块链接](part3compile/ch13link/link.md)

#### 第十四章 编译器

- [逃逸分析](part3compile/ch14gc/escape.md)
- [`unsafe`](part3compile/ch14gc/unsafe.md)
    + 任意类型 `ArbitraryType`
    + Go 指针 `Pointer`
    + 指针操作
- [词法与文法](part3compile/ch14gc/parse.md)
- [类型系统](part3compile/ch14gc/type.md)
- [编译后端 SSA](part3compile/ch14gc/ssa.md)
- [过去、现在与未来]

### [第四部分：标准库](part4lib/readme.md)

#### [第十五章 错误处理](part4lib/ch15errors/readme.md)

- [15.1 错误处理的历史及其演化](part4lib/ch15errors/error.md)
    + `error` 类型的历史形态
    + 错误处理的基本策略
      + 哨兵错误
      + 自定义错误
      + 隐式错误
    + `pkg/errors` 的错误处理原语
    + 争议
      + 对错误处理进行改进的反馈
      + check/handle 关键字
      + 内建函数 try
- [16.2 `errors` 包与错误检查](part4lib/ch15errors/errors.md)
    + 错误检查
      + Unwrap
      + As 与 Is
      + `%w`

#### [第十六章 sync 与 atomic 包](part4lib/ch16sync/readme.md)

- [16.1 `sync.Pool`](part4lib/ch16sync/pool.md)
    + 底层结构
    + Get
    + Put
    + 偷取细节
      + pin 和 pinSlow
      + getSlow
    + 缓存的回收
    + poolChain
      + poolChain 的 popHead, pushHead 和 popTail
      + poolDequeue 的 popHead, pushHead popTail
    + noCopy
    + 总结
    + 进一步阅读的参考文献
- [16.2 `sync.Once`](part4lib/ch16sync/once.md)
- [16.3 `sync.Map`](part4lib/ch16sync/map.md)
    + 结构
    + Store
    + Load
    + Delete
    + Range
    + LoadOrStore
    + 总结
- [16.4 `sync.WaitGroup`](part4lib/ch16sync/waitgroup.md)
    + 结构
    + Add/Done
    + Wait
- [16.5 `sync.Mutex`](part4lib/ch16sync/mutex.md)
- [16.6 `sync.Cond`](part4lib/ch16sync/cond.md)
    + 结构
    + copyChecker
    + Wait/Signal/Broadcast
    + notifyList
- [16.7 `sync/atomic.*`](part4lib/ch16sync/atomic.md)
    + 公共包方法
      + atomic.Value
      + atomic.CompareAndSwapPointer
    + 运行时实现

#### [第十七章 其他](part4lib/ch17other/readme.md)

- [17.1 `syscall.*`](part4lib/ch17other/syscall.md)
    + 由运行时提供支持的系统调用
    + 通用型系统调用
      + `runtime.entersyscall` 和 `runtime.exitsyscall`
      + 返回的错误处理
    + 进一步阅读的参考文献
- [17.2 `os/signal.*`](part4lib/ch17other/signal.md)
- [17.3 `reflect.*`](part4lib/ch17other/reflect.md)
- [17.4 `net.*`](part4lib/ch17other/net.md)
- [17.5 `time.*`](part4lib/ch17other/time.md)

### [结束语](finalwords.md)

### [参考文献](../bibliography/list.md)

### 附录

- [附录A: 源码索引](appendix/index.md)
- [附录B: 术语表](appendix/glossary.md)