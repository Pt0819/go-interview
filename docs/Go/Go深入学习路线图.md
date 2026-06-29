# Go 深入学习路线图

> 从应用到原理，系统性地深入 Go 语言底层机制。路线分为 6 个阶段，由浅入深，每个阶段包含必读文档、关键概念、实践建议和自检题目。

---

## 目录

- [第一阶段：语言内功 — 常用数据结构的底层](#第一阶段语言内功--常用数据结构的底层)
- [第二阶段：并发基础 — CSP 模型与 goroutine/channel](#第二阶段并发基础--csp-模型与-goroutinechannel)
- [第三阶段：同步原语 — 锁与无锁数据结构](#第三阶段同步原语--锁与无锁数据结构)
- [第四阶段：运行时调度 — GPM 模型与 goroutine 生命周期](#第四阶段运行时调度--gpm-模型与-goroutine-生命周期)
- [第五阶段：内存与 GC — 分配、回收、排查](#第五阶段内存与-gc--分配回收排查)
- [第六阶段：工程实战 — 网络、调试、序列化](#第六阶段工程实战--网络调试序列化)
- [缺失主题汇总](#缺失主题汇总)

---

## 第一阶段：语言内功 — 常用数据结构的底层

> 你每天用的 slice、interface、struct，底层到底长什么样？

### 阅读顺序

| 序号 | 文档 | 核心内容 |
|------|------|----------|
| 1.1 | `slice实践以及底层实现.md` | ptr/len/cap 三元组、共享底层数组、append 扩容策略（< 1024 翻倍，>= 1024 约增长 25%） |
| 1.2 | `结构体是否可以比较.md` | 字段对齐与内存布局、`==` 运算符的前提条件、不可比较类型（slice/map/function） |
| 1.3 | `Interface内部实现的理解.md` | iface/eface 结构、数据指针 + 类型指针、接口判 nil 的经典陷阱 |
| 1.4 | `常用包.md` | fmt/strings/strconv/sync 等标准库的高效用法与设计模式 |

### 关键概念提要

- **slice**：`reflect.SliceHeader` 三元组（Data, Len, Cap）。append 超出 cap 时分配新底层数组，旧数组可能被 GC 回收
- **string vs []byte**：string 是不可变的，底层是 `reflect.StringHeader`；与 []byte 互转有内存拷贝开销
- **struct 对齐**：字段排列顺序影响内存占用，`unsafe.Sizeof` / `unsafe.Alignof` 可验证
- **interface 装箱**：当一个具体类型被赋值给 interface，Go 会创建一个 `iface`（有方法的接口）或 `eface`（空接口），存储动态类型和动态值
- **nil 接口陷阱**：`var p *MyErr = nil; var e error = p; e != nil` 因为接口的 `_type` 字段不为空

### 实践建议

- 用 `unsafe.Sizeof` 观察不同结构体的内存占用，尝试重排字段减少 padding
- 手写一个动态扩容的 slice 实现（用数组 + 指针模拟）
- 故意触发 interface nil 判断陷阱，用 `fmt.Printf("%#v\n", e)` 和反射确认类型信息
- 用 `go build -gcflags="-m"` 观察变量逃逸到堆的情况

### 自检问题

1. `slice = append(slice, x)` 后，原来的底层数组什么时候会被替换？
2. 两个 nil 的 `interface{}` 变量比较，一定相等吗？
3. 什么类型的字段会导致 struct 不可用 `==` 比较？
4. 为什么 `var nilErr error = (*MyErr)(nil)` 判断 `nilErr == nil` 为 false？

### 缺失主题

| 主题 | 说明 |
|------|------|
| **string 与 []byte 的底层与互转** | string 的不可变性原因，`unsafe` 零拷贝转换的风险与场景，`strings.Builder` 内部优化 |
| **map 的底层实现** | hmap 结构体、bucket 链、哈希冲突解决、扩容与渐进式驱逐（evacuate）、不可寻址的原因 |
| **defer 的实现与执行顺序** | defer 链表（link struct）、参数快照时机、返回值命名的影响、Go 1.13 栈分配优化、1.14 open coded defer |

---

## 第二阶段：并发基础 — CSP 模型与 goroutine/channel

> 并发编程的 "正确姿势" 和底层通信机制。

### 阅读顺序

| 序号 | 文档 | 核心内容 |
|------|------|----------|
| 2.1 | `CSP并发模型.md` | CSP 哲学："通过通信来共享数据，而不是通过共享内存来通信" |
| 2.2 | `goroutine什么情况下会阻塞.md` | 六大阻塞场景：channel 收发、系统调用、time.Sleep、GC、锁争用、network I/O |
| 2.3 | `gofunc过程.md` | `go func(){}()` 的背后：创建 g 结构体 → 初始化栈 → 加入 P 本地队列 |
| 2.4 | `chan底层原理.md` | hchan 核心字段（buf/sendq/recvq/lock）、缓冲与非缓冲、FIFO 唤醒 |
| 2.5 | `Context的使用场景.md` | 超时取消、链式传递、goroutine 生命周期管理、最佳实践 |

### 关键概念提要

- **CSP vs Actor**：Go 的 channel 是匿名的、同步的、有缓冲的；CSP 强调通过 channel 编排并发
- **goroutine 栈**：初始 2KB，动态增长（栈拷贝），最大可达 1GB
- **hchan 的等待队列**：sendq 和 recvq 都是 FIFO 双向链表，存放 sudog 结构
- **select 随机性**：多个 case 同时就绪时，Go runtime 伪随机打乱 case 顺序，防止饥饿
- **Context 树**：`Done()` 返回只读 channel，WithCancel/WithTimeout/WithDeadline 级联取消

### 实践建议

- 实现一个 "扇出-扇入" 管道：多个 worker 从同一 channel 消费，结果汇聚到一个 channel
- 用 select 实现：超时控制、非阻塞收发、心跳检测三种并发模式
- 故意造成 goroutine 泄露（如向一个无人接收的 channel 发送），用 `/debug/pprof/goroutine` 观察泄露
- 用 Context 实现一个带超时取消的 HTTP 请求链，验证子 context 被 cancel 后能否被父 context 感知

### 自检问题

1. 无缓冲 channel 和有缓冲 channel，发送方和接收方谁先阻塞？
2. `select { case ch <- x: default: }` 加上 default 后语义有什么变化？
3. `context.WithCancel` 派生出的子 context 被 cancel 后，父 context 的 Done channel 会被关闭吗？
4. goroutine 被阻塞在 channel 上后，所在的 P 会怎样？

### 缺失主题

| 主题 | 说明 |
|------|------|
| **select 底层实现** | 多 case 的随机打乱（scramble）、sudog 的入队与出队、`runtime.selectgo` 的完整流程 |
| **channel 关闭的完整语义** | 读已关闭 channel 立即返回零值、写已关闭 channel panic、优雅关闭的 producer-only 模式、`sync.WaitGroup` + close 组合 |
| **goroutine 泄露的模式** | chan 阻塞泄露、循环创建未退出、锁未释放、HTTP response body 未关闭——每种的定位和修复 |

---

## 第三阶段：同步原语 — 锁与无锁数据结构

> mutex 家族、sync.Map、sync.Pool 的原理与选型。

### 阅读顺序

| 序号 | 文档 | 核心内容 |
|------|------|----------|
| 3.1 | `mutex怎么使用，乐观和悲观锁的实现.md` | sync.Mutex 基础用法 + 乐观锁（CAS 自旋）vs 悲观锁（挂起等待）的哲学 |
| 3.2 | `乐观锁与悲观锁与Golang.md` | Go 中 CAS 的实践场景：atomic、自旋锁、乐观更新数据库记录 |
| 3.3 | `互斥锁实现原理剖析.md` | sync.Mutex 源码：KEY 值、正常/饥饿模式、自旋条件、W 状态 |
| 3.4 | `读写锁的实现及底层原理.md` | sync.RWMutex：读锁可重入设计、写者饥饿问题、RLock/RUnlock 的性能路径 |
| 3.5 | `读写分离 sync.Map.md` | read map（readOnly）+ dirty map（entry.p 指针）、miss 计数与晋升 |
| 3.6 | `sync.Pool.md` | per-P localPool、private/shared、victim cache 的 GC 延迟清空策略 |

### 关键概念提要

- **sync.Mutex 两种模式**：正常模式（自旋等待，新来的 goroutine 抢锁有优势）→ 饥饿模式（FIFO 队列保证公平，防止尾部延迟过高）
- **RWMutex**：写锁互斥，读锁共享；大量读的情况下写者可能"饿死"
- **sync.Map 适用场景**：读多写少且 key 集合相对稳定（entry.p 指针复用）
- **sync.Pool victim cache**：GC 时先移到 victim，再下一次 GC 才清空——给"跨 GC 轮次复用"留机会
- **锁选型决策树**：纯读不写 → 无需锁；读多写少 → RWMutex 或 sync.Map；写多 → Mutex；单次操作极快 → 自旋锁或 atomic

### 实践建议

- 实现一个并发计数器：分别用 sync.Mutex、atomic.Int64、sync.Map 的 LoadOrStore，benchmark 对比不同并发度（1/4/16/64 goroutine）
- 实现一个简化版 RWMutex（mutex + atomic + channel），体会读锁计数和写者排队
- 用 sync.Pool 优化 `bytes.Buffer` 的创建回收，pprof 对比优化前后的 `alloc_space`
- 用 race detector（`go run -race`）检测锁遗漏问题

### 自检问题

1. sync.Mutex 在什么条件下从正常模式切换到饥饿模式？
2. sync.Map 的 `Load` 操作为什么在大多数情况下不需要加锁？
3. sync.Pool 里的对象在 GC 时一定会被回收吗？victim cache 给了几次 "幸存机会"？
4. 什么场景下用 `atomic.CompareAndSwapInt32` 自旋比 sync.Mutex 更好？（提示：临界区极短）
5. sync.RWMutex 的 `RLock` 是否可重入？同一个 goroutine 多次 `RLock` 会死锁吗？

### 缺失主题

| 主题 | 说明 |
|------|------|
| **atomic 与内存顺序** | atomic.Load/Store/Add/Swap/CAS、Go 1.19+ 的 atomic 泛型类型、内存屏障（memory barrier）与 happens-before 语义 |
| **sync.Cond 使用场景** | 条件变量的 Signal/Broadcast 模式、什么时候用 Cond 而不是 channel（如：多条件等待同一个锁） |
| **errgroup 与并发错误处理** | `golang.org/x/sync/errgroup`：多 goroutine 并发，任一失败即取消全组；与 `context.Context` 的集成 |

---

## 第四阶段：运行时调度 — GPM 模型与 goroutine 生命周期

> Go 如何调度百万 goroutine？g0、gopark、栈扩容是什么？

### 阅读顺序

| 序号 | 文档 | 核心内容 |
|------|------|----------|
| 4.1 | `go runtime 简析.md` | 运行时全景：调度器、内存分配器、GC 三大子系统的职责边界 |
| 4.2 | `线程的实现模型.md` | 1:1、N:1、M:N 三种线程模型，解释 Go 为什么选择 M:N |
| 4.3 | `GPM调度模型.md` | G/P/M 三元组关系、本地队列（runq）+ 全局队列（runqhead）、work-stealing |
| 4.4 | `Go协程的栈内存管理.md` | 栈 2KB 起步、栈拷贝（stack copying）、栈的写屏障、缩容策略 |
| 4.5 | `gopark函数和goready函数原理分析.md` | gopark→goready 对偶：挂起当前 g → 唤醒目标 g → 加入 runq |
| 4.6 | `特权 Goroutine g0.md` | 每个 M 有两个 g（g0 管理栈 + gsignal 信号栈），schedule() 必须跑在 g0 上 |
| 4.7 | `如何回收goroutine.md` | goroutine 退出路径：goexit → 写屏障 → gFree 链表复用 g 对象 |

### 关键概念提要

- **M:N 调度**：N 个 goroutine 映射到 M 个 OS 线程，runtime 负责映射（不依赖内核调度器）
- **P 的角色**：P 是"执行资源"，数量 = GOMAXPROCS。P 有 runq（容量 256）+ runnext（单 slot 优先级插队）
- **work-stealing 顺序**：本地 runq → runnext → 全局 runq → 其他 P 的 runq（偷一半）
- **栈扩容**：栈拷贝时用写屏障保证所有指针指向新栈；goroutine 当前活跃帧以上的区域做 **stackmap** 精确定位
- **g0 的特殊性**：g0 有自己的栈（8KB 起，可扩容），调度器代码（schedule、findrunnable、park_m）必须在 g0 栈上执行
- **gopark/goready**：gopark 设置 g 的状态为 `_Gwaiting` 然后调用 schedule()；goready 设置 g 为 `_Grunnable` 然后加入 runq

### 实践建议

- 设置 `GODEBUG=schedtrace=1000` 运行程序，观察每 1 秒的调度状态（idle P、全局队列长度、偷任务次数）
- 用 dlv 打断点 `runtime.schedule` 和 `runtime.gopark`，单步跟踪一个 goroutine 从阻塞到唤醒的全流程
- 写一个 CPU 密集型程序（如循环计算 hash），对比 GOMAXPROCS=1/2/4/8/16 的吞吐量变化
- 用 `GODEBUG=scheddetail=1` 观察每个 P 的 M 绑定状态和 runq 深度

### 自检问题

1. P 的 runq 有容量上限吗？满了之后新创建的 goroutine 放哪里？
2. g0 和普通 goroutine 的栈是物理隔离的吗？为什么调度循环必须在 g0 上运行？
3. goroutine 栈扩容的触发条件是什么？栈拷贝过程中如何保证旧栈的指针不悬空？
4. goroutine 退出后，它的 g 结构体会被彻底释放吗？（提示：gFree 链表）
5. 当一个 goroutine 进入系统调用时，P 会被怎样处理？

### 缺失主题

| 主题 | 说明 |
|------|------|
| **系统调用包装（entersyscall/exitsyscall）** | goroutine 进入系统调用时解绑 P → handoffp 把 P 给别的 M → exitsyscall 后重新找 P 或入队 |
| **netpoller** | epoll/kqueue 集成，Go 如何把网络 I/O 阻塞转化为 goroutine 挂起/唤醒，`internal/poll.FD` 的实现 |
| **抢占式调度** | Go 1.14 前：协作式抢占（只在函数调用点检查）；1.14 后：基于信号的异步抢占（sysmon → SIGURG） |
| **runtime.Gosched() 与 runtime.Goexit()** | 主动让出 CPU 的正确姿势，为什么大多数场景下不需要手动调用 |

---

## 第五阶段：内存与 GC — 分配、回收、排查

> GC 算法、STW 时间线、内存泄露定位。

### 阅读顺序

| 序号 | 文档 | 核心内容 |
|------|------|----------|
| 5.1 | `GC 是怎样监听你的应用的.md` | GC 触发条件：GOGC 参数、目标堆大小、定时强制 GC（2 分钟 max interval） |
| 5.2 | `GC垃圾回收算法.md` | 三色标记 + 混合写屏障、并发标记的四个阶段、白色对象 = 垃圾 |
| 5.3 | `Stop the World (STW).md` | STW 的时间线演进、哪些阶段需要 STW、Go 1.14+ 的 STW 优化 |
| 5.4 | `内存泄露的发现与排查.md` | pprof heap 差分分析、`--base` 对比、持续增长 vs 峰值不降两种泄露模式 |

### 关键概念提要

- **GC 触发三条件**：① 堆达到上次 GC 后存活量的 (1 + GOGC/100) 倍；② `runtime.GC()` 手动触发；③ 距上次 GC 超过 2 分钟
- **三色标记**：白（未扫描）→ 灰（正在扫描其引用）→ 黑（已扫描完成）。并发标记结束时白色 = 可回收
- **混合写屏障**：Go 1.8 引入，融合 Dijkstra 插入屏障（黑引用白 → 标灰）和 Yuasa 删除屏障（删除引用 → 标灰），栈上不加屏障
- **STW 时间线**：标记准备（STW ~10μs）→ 并发标记（0 STW）→ 标记终止（STW 0.05~0.5ms）→ 并发清扫（0 STW）
- **内存泄露排查三步骤**：`go tool pprof -base=before.prof after.prof` → `top` → `list <function>`

### 实践建议

- 写一个"快速分配/快速释放"的循环程序，用 `GODEBUG=gctrace=1` 观察每轮 GC 的触发时机、STW 耗时和回收量
- 构造三种内存泄露场景：① 全局 map 只增不删；② goroutine channel 永久阻塞；③ time.Ticker 未 Stop。用 pprof 逐一定位
- 分别设置 GOGC=50 / 100 / 200 / off 运行同一个程序，用 benchstat 对比吞吐量和内存峰值
- 用 `go tool trace` 观察 GC 事件和 goroutine 执行的时间线重叠

### 自检问题

1. 三色标记中，写屏障解决的核心问题是什么？插入屏障和删除屏障各防范哪种 "漏标" 情况？
2. GOGC=100 时，从 GC 刚结束到下一次触发，堆内存大约会增长多少？
3. pprof heap profile 里 `inuse_space` 和 `alloc_space` 的本质区别是什么？排查持续泄露该用哪个？
4. 什么场景下 GC 的 STW（标记终止阶段）时间会特别长？（提示：大 object 的 finalizer？）
5. 并发标记阶段中，黑色对象引用了白色对象，如何保证白色不被误回收？

### 缺失主题

| 主题 | 说明 |
|------|------|
| **内存分配器（TCMalloc 风格）** | mcache（per-P 无锁）→ mcentral（span 缓存，轻锁）→ mheap（全局，大锁）；size class 分 67 级（1B～32KB）；大于 32KB 直接走 mheap |
| **逃逸分析** | `-gcflags="-m"` 逐变量分析、"逃逸"的本质是生命周期超出栈帧；common cases：返回指针、闭包捕获、interface 装箱、大对象 |
| **unsafe 与内存操纵** | unsafe.Pointer 的三条铁律、uintptr 的临时性（GC 不追踪）、与 C 互操作的内存布局保证 |
| **pprof 深度使用** | goroutine profile（阻塞/等待分析）、mutex profile（锁争用排行）、block profile、火焰图解读、`-http=:` 启动交互模式 |
| **go tool trace** | 跟 pprof 互补：goroutine 执行甘特图、GC 事件时间线、调度延迟可视化、`MMU`（最小互斥时间）图 |

---

## 第六阶段：工程实战 — 网络、调试、序列化

> HTTP 连接池、dlv 调试、ProtoBuf、gin 路由树——日常开发的硬通货。

### 阅读顺序

| 序号 | 文档 | 核心内容 |
|------|------|----------|
| 6.1 | `长连接和短连接的学习.md` | TCP 连接管理的开销模型：三次握手 + 四次挥手、TIME_WAIT |
| 6.2 | `HTTP Client大量长连接保持.md` | Transport 连接池：MaxIdleConns、MaxIdleConnsPerHost、IdleConnTimeout、Keep-Alive |
| 6.3 | `gin路由树.md` | radix tree 路由匹配、`:param` 与 `*filepath` 通配符、节点优先级 |
| 6.4 | `详解通信数据协议ProtoBuf.md` | varint/zigzag 编码、wire type、字段编号 1-15 vs 16+、向后兼容原理 |
| 6.5 | `dlv分析golang进程.md` | dlv debug / attach / core 三种模式、断点、goroutine 切换、条件断点 |

### 关键概念提要

- **HTTP 连接池核心参数**：`MaxIdleConnsPerHost`（默认 2，太小！）、`IdleConnTimeout`（默认 90s）、`DisableKeepAlives`
- **gin radix tree**：每个节点存储 `path` 前缀 + `indices`（子节点首字符）+ `children`；匹配时先走公共前缀，再按优先级（静态 > 参数 > 通配）
- **ProtoBuf 编码**：varint 用 7bit 数据 + 1bit 终止标志；字段编号 1-15 只需 1 字节 tag（field_number + wire_type）；sint32 用 zigzag 编码使负数也紧凑
- **dlv 三个模式**：`dlv debug`（编译+调试）、`dlv attach <pid>`（附加运行中进程，需 ptrace 权限）、`dlv core <binary> <core>`（core dump 事后分析）

### 实践建议

- 对比 gin 和 net/http 在 100 条路由下的匹配性能，用 `go test -bench=. -benchmem` 量化
- 用 dlv attach 到一个运行的 Go HTTP 服务，执行 `goroutines` 命令，分析哪些 goroutine 最多、哪些在阻塞
- 定义一个复杂的 proto3 文件（含嵌套、repeated、oneof），生成 Go 代码，写测试验证 varint 编码后各类型的字节占用
- 用 Wireshark 抓包（localhost 用 rawcap/tshark），直观对比 ProtoBuf 二进制和 JSON 的尺寸差异

### 自检问题

1. HTTP Client `Transport` 的默认 `MaxIdleConnsPerHost` 是多少？这个默认值在生产环境够用吗？
2. gin 路由 `/user/new` 和 `/user/:name` 同时注册会冲突吗？优先级如何决定？
3. ProtoBuf 的字段编号 1-15 和 16-2047 编码后 tag size 差多少？为什么建议频繁字段用 1-15？
4. dlv 中 `next`（n）、`step`（s）、`step-instruction`（si）三者的区别是什么？
5. HTTP 长连接在防火墙/NAT 环境下有什么风险？如何用 `IdleConnTimeout` 和 Keep-Alive 缓解？

### 缺失主题

| 主题 | 说明 |
|------|------|
| **net/http Server 请求处理全流程** | Listener.Accept → 分配 goroutine per conn → ServeHTTP → ResponseWriter 的 Hijack 和 Flush |
| **Go module 与依赖管理** | go.mod/go.sum 结构、最小版本选择（MVS）、replace/retract/vendor 策略、workspace（Go 1.18+） |
| **Go 测试体系深度** | table-driven test、子测试（t.Run/t.Parallel）、testify/suite、benchmark（`-benchmem`/`-cpuprofile`）、fuzzing（Go 1.18+） |
| **cgo 的边界与开销** | Go→C 调用的 goroutine 固定到 M、栈切换成本；什么场景该用 cgo（已有 C 库/性能关键路径），什么场景不该（可替代的 Go 库/简单逻辑） |

---

## 学习进度建议

### 时间安排

| 阶段 | 预计学习时间 | 难度 |
|------|-------------|------|
| 第一阶段：语言内功 | 1-2 周 | ⭐⭐ |
| 第二阶段：并发基础 | 2-3 周 | ⭐⭐⭐ |
| 第三阶段：同步原语 | 2-3 周 | ⭐⭐⭐ |
| 第四阶段：运行时调度 | 3-4 周 | ⭐⭐⭐⭐ |
| 第五阶段：内存与 GC | 3-4 周 | ⭐⭐⭐⭐ |
| 第六阶段：工程实战 | 2-3 周 | ⭐⭐⭐ |

### 学习方法论

1. **先读文档，再读源码** — 每篇文档提供了高层框架，理解后用 `dlv` 或直接读 `runtime/` 源码验证
2. **带着问题写代码** — 每个阶段都有实践建议，动手写比眼睛看效果好 10 倍
3. **用 pprof/trace/dlvv 验证理解** — Go 的工具链是理解运行时的窗口，每学一个机制就用对应工具观察
4. **做笔记，画流程图** — GPM 调度、三色标记、chan 唤醒链，这些流程画出来才真正内化
5. **交叉复习** — 阶段间有依赖关系：学 GC 时回顾 goroutine 栈（写屏障），学 netpoller 时回顾 gopark

### 推荐的源码阅读顺序

```
runtime/
├── runtime2.go       # g/m/p 结构体定义（从这里开始）
├── proc.go           # schedule、findrunnable、work-stealing
├── stack.go          # 栈拷贝、扩缩容
├── chan.go           # hchan、send/recv/select
├── mutex.go          # sync.Mutex runtime 部分（sema）
├── mheap.go          # 堆管理
├── mcentral.go       # 中央缓存
├── mcache.go         # per-P 缓存
├── mgc.go            # GC 入口
├── mbarrier.go       # 写屏障
└── netpoll.go        # 网络轮询器
```

---

## 缺失主题汇总

以下是当前文档库未覆盖但各阶段推荐补充的 21 个主题：

| # | 阶段 | 缺失主题 | 重要程度 |
|---|------|----------|----------|
| 1 | 语言内功 | string 与 []byte 的底层与互转 | ⭐⭐⭐ |
| 2 | 语言内功 | map 的底层实现 | ⭐⭐⭐ |
| 3 | 语言内功 | defer 的实现与执行顺序 | ⭐⭐⭐ |
| 4 | 并发基础 | select 底层实现 | ⭐⭐⭐ |
| 5 | 并发基础 | channel 关闭的完整语义 | ⭐⭐ |
| 6 | 并发基础 | goroutine 泄露的模式与排查 | ⭐⭐⭐ |
| 7 | 同步原语 | atomic 与内存顺序 | ⭐⭐⭐ |
| 8 | 同步原语 | sync.Cond 使用场景 | ⭐⭐ |
| 9 | 同步原语 | errgroup 与并发错误处理 | ⭐⭐ |
| 10 | 运行时调度 | 系统调用包装（entersyscall/exitsyscall） | ⭐⭐⭐ |
| 11 | 运行时调度 | netpoller | ⭐⭐⭐ |
| 12 | 运行时调度 | 抢占式调度 | ⭐⭐⭐ |
| 13 | 运行时调度 | runtime.Gosched/Goexit | ⭐⭐ |
| 14 | 内存与 GC | 内存分配器（TCMalloc 风格） | ⭐⭐⭐ |
| 15 | 内存与 GC | 逃逸分析 | ⭐⭐⭐ |
| 16 | 内存与 GC | unsafe 与内存操纵 | ⭐⭐ |
| 17 | 内存与 GC | pprof 深度使用 | ⭐⭐⭐ |
| 18 | 内存与 GC | go tool trace | ⭐⭐ |
| 19 | 工程实战 | net/http Server 请求处理全流程 | ⭐⭐⭐ |
| 20 | 工程实战 | Go module 与依赖管理 | ⭐⭐ |
| 21 | 工程实战 | Go 测试体系深度 | ⭐⭐⭐ |
| 22 | 工程实战 | cgo 的边界与开销 | ⭐⭐ |

---

## 参考资料

- [Go 官方博客](https://go.dev/blog/) — 深入理解 Go 的权威来源
- [Go Runtime 源码](https://github.com/golang/go/tree/master/src/runtime) — 所有运行时机制的最终答案
- [The Go Memory Model](https://go.dev/ref/mem) — happens-before 关系的权威定义
- [Diagnostics - Go](https://go.dev/doc/diagnostics) — pprof、trace、GODEBUG 的官方指南
- [Effective Go](https://go.dev/doc/effective_go) — Go 编程惯例与最佳实践
- [Go Scheduler: Ms, Ps, & Gs](https://www.youtube.com/watch?v=KBZlN0izeiY) — Kavya Joshi 的 GPM 调度经典演讲
- [Getting to Go: The Journey of Go's Garbage Collector](https://go.dev/blog/ismmkeynote) — Rick Hudson 的 GC 演进史
