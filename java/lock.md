# Java 锁的原理与使用

## 目录

- [synchronized 与 Lock 的区别：语法、功能、底层实现与设计理念](#synchronized-与-lock-的区别语法功能底层实现与设计理念)
- [synchronized 的锁是任何实例，Lock 的锁是专门的类](#synchronized-的锁是任何实例lock-的锁是专门的类)
- [Mark Word：每个对象天生的锁依据，兼有三重用途](#mark-word每个对象天生的锁依据兼有三重用途)
- [Mark Word 如何实现锁：CAS 抢写与偏向轻量级重量级升级链](#mark-word如何实现锁cas抢写与偏向轻量级重量级升级链)
- [锁的身份是"同一个实例"（==），而不是同一个类或内容相等](#锁的身份是同一个实例而不是同一个类或内容相等)
- [synchronized(this) 放进括号里是什么情况](#synchronizedthis放进括号里是什么情况)
- [自己设计锁：从专用锁对象到 AQS](#自己设计锁从专用锁对象到-aqs)
- [notify() 与 wait/notify 机制：铃声不带信息，条件活在 while 里](#notify-与-waitnotify-机制铃声不带信息条件活在-while-里)

---

## synchronized 与 Lock 的区别：语法、功能、底层实现与设计理念
- 2026-09-21

**结论**：`synchronized` 是 JVM 内置的隐式锁（自动加锁释放、块结构、不可中断、非公平），`Lock` 是 juc 包里的显式锁接口（手动 `lock()/unlock()`、可尝试/超时/可中断/公平、多 Condition 精确唤醒）。JDK 1.6 之后两者性能差距已很小，**默认选 synchronized，只在需要 Lock 的独有能力时才用它**。

**背景**：讨论 Java 中两种加锁方式的区别，涵盖语法、功能、底层实现与设计理念。

**细节**：

### 1. 语法对比

synchronized——隐式锁，JVM 自动管理：

```java
// 用法1：修饰方法
public synchronized void transfer() { ... }

// 用法2：修饰代码块，指定锁对象
private final Object lock = new Object();
public void transfer() {
    synchronized (lock) {
        // 临界区
    }
}
```

Lock——显式锁，最常用实现是 `ReentrantLock`：

```java
private final Lock lock = new ReentrantLock();

public void transfer() {
    lock.lock();          // 必须显式加锁
    try {
        // 临界区
    } finally {
        lock.unlock();    // 必须在 finally 中显式释放，否则异常时锁永远不释放
    }
}
```

| 对比点 | synchronized | Lock |
|---|---|---|
| 层面 | 语言关键字，JVM 层实现 | 接口 + 实现类，纯 Java 类库 |
| 加锁/解锁 | 自动（退出代码块或抛异常时自动释放） | 手动调用 `lock()` / `unlock()` |
| 忘记释放的后果 | 不可能发生 | 死锁（所以必须放 `finally`） |
| 锁的作用范围 | 必须是块结构（方法或 `{}` 内） | 可以跨方法、甚至跨对象持有 |
| 读写分离 | 不支持 | `ReentrantReadWriteLock` 支持 |

### 2. Lock 的独有功能

```java
// 1. 非阻塞尝试：拿不到锁立即返回 false，不阻塞
if (lock.tryLock()) {
    try { ... } finally { lock.unlock(); }
}

// 2. 超时等待：最多等 3 秒
if (lock.tryLock(3, TimeUnit.SECONDS)) { ... }

// 3. 可中断等待：等待锁的过程中可以响应 interrupt()
lock.lockInterruptibly();

// 4. 公平锁：按线程排队顺序获取锁，而非抢占式
Lock fairLock = new ReentrantLock(true);
```

synchronized 的等待是不可中断、非公平、无限期的——一旦拿不到锁就只能死等。

等待-通知机制上，synchronized 只能配合 Object 的 `wait/notify/notifyAll`（一个锁一把条件，`notifyAll()` 会唤醒所有等待线程）；Lock 可以创建多个 Condition，按条件精确唤醒：

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notFull  = lock.newCondition();
private final Condition notEmpty = lock.newCondition();
// 生产者唤醒 notEmpty，消费者唤醒 notFull，互不干扰
// 典型应用如 ArrayBlockingQueue
```

### 3. 底层实现

- **synchronized**：编译后依赖字节码指令 `monitorenter` / `monitorexit`（代码块）或方法标志位 `ACC_SYNCHRONIZED`，底层关联对象的 monitor（管程）。JDK 1.6 后做了偏向锁、轻量级锁、适应性自旋、锁消除、锁粗化等优化，无竞争和低竞争场景开销非常小。
- **Lock（ReentrantLock）**：基于 **AQS**（`AbstractQueuedSynchronizer`）实现，核心是 `volatile int state` + CAS 修改状态 + FIFO 双向队列管理阻塞线程。锁逻辑全在 Java 类库里，可阅读、可扩展（AQS 还派生出读写锁、信号量、倒计时器等）。

### 4. 设计理念

- **synchronized：让正确的并发简单易用**。锁是语言原语，不可能忘记释放，块结构强制加解锁严格嵌套，追求"默认安全"，符合 Java 减少犯错机会的语言哲学。代价是灵活性受限。
- **Lock：把控制权交给专家**。Doug Lea 认为块结构锁无法覆盖所有并发场景，把锁抽象成普通对象——可跨方法传递（hand-over-hand 锁）、可尝试获取后放弃、可限时、可中断、可多条件队列。代价是把释放锁的责任转移给开发者。

一句话：**synchronized 是保守但安全的"编译器保证"，Lock 是灵活但自律的"开发者负责"**。

### 5. 选择建议

- 默认用 synchronized：更简单、不会出错，且 JVM 会持续优化；
- 只有明确需要以下能力时才用 Lock：尝试/超时获取锁、可中断等待、公平队列、多条件精确唤醒（Condition）、读写分离、非块结构加锁。

**坑**：

- `unlock()` 忘了放 `finally` 里，临界区抛异常后锁永远不释放，导致后续所有线程死锁；
- synchronized 的锁等待不响应中断，等待中的线程无法被取消，需要可取消的等待必须换 `lockInterruptibly()`；
- 性能不是选 Lock 的理由——JDK 1.6 后 synchronized 有偏向锁/轻量级锁路径，低竞争时反而更快。

## synchronized 的锁是任何实例，Lock 的锁是专门的类
- 2026-09-21

**结论**：synchronized 的锁 = 任何对象内置的 monitor（机制只碰对象头，与类型、字段、内容无关）；Lock 的锁 = 基于 AQS 的 `ReentrantLock` 等专门类的实例。任何实例都能当 synchronized 的锁，但任何实例都变不出 `lock()` 方法。

**细节**：

```java
Object lock1 = new Object();
Order order = new Order();
int[] arr = new int[10];

synchronized (lock1) { }              // ✓
synchronized (order)   { }            // ✓ 任何实例都行
synchronized (arr)     { }            // ✓ 数组也是对象
synchronized (order.getClass()) { }   // ✓ Class 对象也是对象（"类锁"）
```

证据在 API 设计里：`wait()/notify()/notifyAll()` 定义在 `java.lang.Object` 上——正因每个对象内部都关联 monitor，任何对象才能既当锁用、又当条件队列用。

| | synchronized | Lock |
|---|---|---|
| 锁是什么 | 任何对象内置的 monitor | ReentrantLock 等专门类的实例 |
| 任何实例能当锁？ | 能（自带 Mark Word/monitor） | 不能（必须 new 一个 Lock） |

**坑**：null 不能当锁（NPE）；基本类型编译不过（没有对象头）；String 常量池/Integer 缓存实例会被意外共享；标准写法是专用的 `private final Object lock = new Object()`。

## Mark Word：每个对象天生的锁依据，兼有三重用途
- 2026-09-21

**结论**：每个对象在 JVM 分配内存时自带对象头，其中 Mark Word（64 位 JVM 上 8 字节）是**锁、GC、hashcode 三个子系统共享的复用字段**——不当锁的对象，它的 Mark Word 也在为 GC（分代年龄）和 hashCode() 缓存工作。"与生俱来"指的是机制出厂自带，不是出生就挂着重量级锁（ObjectMonitor 是竞争加剧后才惰性创建的）。

**细节**：

```
+---------------------------+
|       对象头 (Header)      |
|  +---------------------+  |
|  |     Mark Word       |  | ← 多功能档案柜
|  +---------------------+  |
|  |  类型指针 (Klass)    |  | ← 指向类元数据，方法调用/instanceof/反射用，与锁无关
+---------------------------+
|     实例数据 (你的字段)     |
+---------------------------+
```

| 用途 | 具体内容 | 何时使用 |
|---|---|---|
| **锁** | 线程 ID / Lock Record 指针 / ObjectMonitor 指针 | 被 synchronized 时 |
| **GC** | 分代年龄 age（4 位）、GC 标记 | **每个对象都用**——熬过 Minor GC 次数，攒到阈值（默认 15）晋升老年代 |
| **hashcode** | identity hashcode（31 位） | 调 `hashCode()` 时缓存（未重写的类） |

三用途复用同一块地方的直接后果：轻量级锁加锁要把原 Mark Word 拷贝到 Lock Record（Displaced Mark Word）保管，否则 hashcode 和 GC 年龄丢失；算过 hashcode 的对象与偏向锁放不下（31 位 hashcode 与 54 位线程 ID 冲突），会跳过偏向。

## Mark Word 如何实现锁：CAS 抢写与偏向轻量级重量级升级链
- 2026-09-21

**结论**：加锁 = 各线程用 CAS 抢着改写同一个 Mark Word（末尾 2 位锁标志位标记当前状态，各状态存的内容互相冲突，改写成功者定义锁状态）。竞争程度决定代价：无竞争近零成本（偏向，已废弃），轻度竞争用户态自旋，重度竞争膨胀为 ObjectMonitor 内核挂起——**按竞争强度逐级付费，升级单向不可逆**。

**细节**：

### Mark Word 布局（64 位）

```
| unused:25 | hashcode:31 | unused:1 | age:4 | biased:1 | lock:2 |  无锁     |
| threadId:54 | epoch:2   | unused:1 | age:4 | biased:1 | lock:2 |  偏向锁   |
|         pointer_to_lock_record:62                    | lock:2 |  轻量级锁 |
|         pointer_to_ObjectMonitor:62                   | lock:2 |  重量级锁 |
```

### 三级机制

- **偏向锁**（JDK 15 废弃、18 移除）：第一个线程 CAS 写入自己线程 ID，之后本人再进入只比对 ID、连 CAS 都不做；第二线程竞争时撤销升级；
- **轻量级锁**：加锁 = 栈帧里建 Lock Record、CAS 把 Mark Word 改写为指向它的指针（标志位 00）；失败则自旋重试；解锁 = CAS 把 Displaced Mark Word 塞回；
- **重量级锁**：自旋超阈值后膨胀，Mark Word 指向 ObjectMonitor（C++ 结构）：

```cpp
ObjectMonitor {
    _owner      // 当前持锁线程
    _EntryList  // 抢锁失败被 park 挂起的线程队列（BLOCKED！）
    _WaitSet    // 调用 wait() 的线程队列
    _recursions // 重入计数
}
```

抢不到的线程经 OS mutex（pthread_mutex_lock）挂起，用户态→内核态切换，这是"重量级"的来源。释放时从 _EntryList 唤醒线程继续竞争。

### 升级链

```
无锁 ──CAS写线程ID──> 偏向锁 ──竞争──> 轻量级锁（CAS 指向 Lock Record）──自旋失败──> 重量级锁（ObjectMonitor 挂起排队）
```

### 已验证的心智模型（重要）

多个线程的 synchronized(aaa) 抢的是**同一个 Mark Word**，CAS 改写成功者进临界区，走完恢复原值，所有人再抢（非公平，新来的可能赢过等久的）。两处精化：
- CAS 的"读旧值、比对、写新值"是一条 CPU 原子指令（cmpxchg）内完成的，不存在"读完还没写被插手"的窗口——原子性是锁机制可信的根基；
- 抢不到 ≠ 立刻阻塞：先**自旋**重试（轻度竞争常态，自旋期间就能抢到，从未阻塞），自旋无果才膨胀挂起（此时才是 BLOCKED）。

### 术语速查

| 术语 | 含义 |
|---|---|
| CAS / cmpxchg | 比较并交换 / x86 上对应的原子指令 |
| Lock Record / Displaced Mark Word | 栈帧里的锁记录 / 被拷贝保管的原 Mark Word |
| ObjectMonitor / 管程(monitor) | 重量级锁真身（C++ 结构）/ 其理论原语 |
| monitorenter / monitorexit | synchronized 代码块编译出的字节码指令 |
| mutex / pthread_mutex_lock | OS 互斥量 / POSIX 线程库的加锁函数 |
| park / unpark | 线程挂起/唤醒原语（LockSupport） |
| 用户态/内核态 | CPU 特权级；挂起线程需切入内核，慢 |
| 自旋 | 抢锁失败后空转循环重试，烧 CPU 换免切换 |
| 重入 | 同线程可反复加同一把锁，记层数（_recursions） |

**坑**：升级单向不可逆；调 wait() 立刻膨胀为重量级（_WaitSet 只存在于 ObjectMonitor）；算过 identity hashcode 的对象跳过偏向；偏向锁 JDK 15 已废弃（JDK 18+ 直接走轻量级 CAS 路径）。

## 锁的身份是"同一个实例"（==），而不是同一个类或内容相等
- 2026-09-21

**结论**：是不是同一把锁，看的是 `==`（同一个对象、同一个对象头），不随类走、不随内容走、不随 equals 走。同一个类的两个不同实例 = 两个独立的 Mark Word = 两把互不知晓的锁，互斥失效。想让锁起作用，先问：所有线程摸到的是不是同一个 `==` 的对象。

**细节**：

```java
class Task implements Runnable {
    private final Object lock = new Object();      // ✗ 每个线程各 new 一个，互斥失效
    private static final Object LOCK = new Object(); // ✓ static：全类共享同一实例
}
```

原则：保护**静态数据** → 锁必须 static（或类锁 `XXX.class`，Class 对象全 JVM 唯一）；保护**实例数据** → 锁随实例走，此时"每实例一把锁"反而是正确的细粒度设计（与事故性的身份分裂区分开）。

四个经典变体（同一原则的马甲）：

1. 实例方法 `synchronized(this)`：两个实例调各自的同步方法不互斥，保护 static 数据失守；跨实例互斥要类锁（`static synchronized` 或 `synchronized(XXX.class)`）；
2. 锁引用被重新赋值：`lock = new Object()` 前后是两把锁——铁律 `private final`；
3. 内容相同的陷阱：`new String("lock")` 与常量池实例不是同一把锁；Integer 缓存池内（-128~127）`==` 为 true、池外为 false——用包装类当锁，互斥与否取决于数值范围；
4. 跨类加载器：同类被不同 ClassLoader 加载，实例属于两个不同的类，不可能构成同一把锁（多 ClassLoader 的 Web 容器偶发）。

## synchronized(this) 放进括号里是什么情况
- 2026-09-21

**结论**：锁"当前实例"，与 synchronized 实例方法完全等价（字节码细节不同：方法用 ACC_SYNCHRONIZED 标志、代码块用 monitorenter/monitorexit，锁语义一样）。同一实例上所有 synchronized 实例方法和 synchronized(this) 块共用一把锁互相互斥；不同实例不互斥；与类锁互不相干。

**细节**：

```java
public synchronized void deposit() { ... }   // 等价于 synchronized(this) { ... }

class Account {                              // 对象状态是一个整体时，锁 this 合理
    private int balance;
    public synchronized void deposit(int x)  { balance += x; }
    public synchronized void withdraw(int x) { balance -= x; }
}
```

三个特有问题：

1. **身份公开**：this 是暴露给外界的引用，任何拿到实例的代码都能 `synchronized(acc)` 锁你——无法阻止、无法审计，甚至与外部代码形成意外死锁。专用锁 `private final Object lock` 换来的是私有可控的身份；
2. **粒度粗**：整个实例一把锁，操作不相干字段的方法也互斥。可拆成多个专用锁（balanceLock / logLock）；
3. **与类锁互斥失效**：实例锁与 static synchronized / synchronized(Class) 是两把锁，互不阻塞。

合理场景：对象可变状态高度耦合本该整体互斥、实例不泄露给不信任代码。JDK 老集合（Vector、Hashtable、StringBuffer）全是 synchronized(this)，后被 juc 专用锁设计取代。

## 自己设计锁：从专用锁对象到 AQS
- 2026-09-21

**结论**："费心机设计专职锁实例"正是 ReentrantLock 的诞生思路，但分两条路：在 synchronized 的世界里，锁实例只能提供身份 + 外围信息（命名、监控），锁行为是 JVM 黑盒改不了；想要可编程的锁，就把锁的本质——**状态 + CAS + 等待策略**——自己写成类，这条路走到头是 AQS 和整个 juc 锁家族。锁设计还有第三维：粒度/分段的架构设计。

**细节**：

路线 A——给 synchronized 设计锁对象（只得到外围功能）：

```java
class NamedLock {              // 命名、业务标注可以，但 synchronized 只读对象头
    final String name;         // 类里的字段对锁行为零影响
    NamedLock(String name) { this.name = name; }
}
```

路线 B——自己造锁（行为完全可编程）：

```java
class SpinLock {               // 10 行的货真价实的锁：状态 + CAS + 自旋等待
    private final AtomicBoolean locked = new AtomicBoolean(false);
    void lock()   { while (!locked.compareAndSet(false, true)) { } }
    void unlock() { locked.set(false); }
}
```

把"排队管理"的通用苦工抽象出来就是 AQS——骨架是 `volatile int state` + CAS + FIFO 队列，你只需回答"什么叫获取成功"：

```java
class Mutex extends AbstractQueuedSynchronizer {   // JDK 文档经典示例
    protected boolean tryAcquire(int ignored) {
        return compareAndSetState(0, 1);    // 规则：state 0→1 即获得
    }
    protected boolean tryRelease(int ignored) {
        setState(0);
        return true;
    }
}
```

ReentrantLock（重入计数）、Semaphore（多名额）、CountDownLatch（倒计数）、读写锁（两种获取规则）= AQS 骨架 + 不同 tryAcquire 规则。

路线 C——锁的架构设计（锁什么、锁几个）：全局锁 → 每数据一把锁（细粒度）→ 锁分段（`Object[16] stripes`，ConcurrentHashMap 旧版思路）。并发度常靠减少锁碰撞面赢得，而非换更高级的锁。

## notify() 与 wait/notify 机制：铃声不带信息，条件活在 while 里
- 2026-09-23

**结论**：notify() 定义在 Object 上（不是 Thread），随机唤醒**一个**在这个对象上 wait() 的线程；notifyAll() 才唤醒全部。wait/notify 双方对"条件"一无所知——条件只存在于调用方写的 while 循环里。notify 结束的是"睡眠"（把线程移出 _WaitSet 去抢锁），能否结束"等待"由醒来者抢到锁后自己重查 while 决定，不满足就再睡回去。

**背景**：Object 的 wait/notify 是 synchronized 世界的条件队列（一把锁配一个混杂等待队列），Lock 的 Condition 是它的升级版（一把锁多个条件队列、精确唤醒）。

**细节**：

守卫块范式（等待方与通知方必须用**同一个锁对象**）：

```java
// 消费者：等条件
synchronized (lock) {
    while (queue.isEmpty()) {      // 必须 while，不能 if
        lock.wait();               // 释放锁，进 _WaitSet 睡觉
    }
    var item = queue.remove();
}
// 生产者：改条件 + 发信号
synchronized (lock) {
    queue.add(item);
    lock.notify();                 // 唤醒一个在 lock 上等待的线程
}
```

- 底层动作：notify 把 _WaitSet 里的一个线程移出、送去竞争锁（WAITING → 抢锁的 BLOCKED → 拿到锁后才从 wait() 返回）；醒来的第一件事是排队抢回当初放手的那把锁；
- join() 底层就是这套机制：`while (isAlive()) wait(0);`，线程终止时 JVM 自动 notifyAll()。

notify vs notifyAll：

| | notify() | notifyAll() |
|---|---|---|
| 唤醒数量 | 随机一个（调度器定，不保证先来先服务） | 全部 |
| 风险/成本 | 信号丢失 | 惊群（多余唤醒+抢锁开销） |

- **信号丢失**：一个 _WaitSet 混居着等不同条件的线程，notify 叫醒的人发现条件不满足又睡回去，真正该醒的线程没被叫到——元素明明到位却永远等下去。notify 安全的前提：所有等待者等**同一个条件**且每次只需一个线程干活（如空闲线程池取任务）；
- 经验法则：拿不准就用 notifyAll()——宁付惊群成本，不冒丢信号风险。

已验证的两处理解修正：

1. "没有指定条件的 wait，notify 会直接结束等待" → **wait() 全都不带条件**（签名里没有条件参数，条件由 while 循环表达），notify 的铃声里也不带任何信息，语义只是"条件可能变了"。"结束睡眠"≠"结束等待"：醒来 → 抢锁 → while 重查 → 满足才真正结束等待，否则睡回去（醒错了人/条件被先醒者消费掉时的正常现象）；
2. "notify 一定会结束所有正在 wait 的线程" → 那是 notifyAll 的行为；notify 只结束**一个**随机线程的睡眠。

**坑**：不在 synchronized 块内调用（或锁对象不是调用目标）抛 IllegalMonitorStateException；没人等时 notify 白调（无记账，先 notify 后 wait 信号凭空消失）；必须 while 不能 if（notify 只表示条件可能变了、notifyAll 醒一批先醒者会消费条件、JVM 允许虚假唤醒）；wait 释放锁而 sleep 不释放；wait 期间可被 interrupt() 中断。
