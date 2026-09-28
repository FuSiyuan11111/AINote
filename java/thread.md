# Java Thread 类与线程控制

## 目录

- [Thread 是怎样的类：OS 线程的封装 + 任务/线程双重身份](#thread-是怎样的类os-线程的封装--任务线程双重身份)
- [Thread 能被任意多个类继承，单继承限制的是"你自己的类"](#thread-能被任意多个类继承单继承限制的是你自己的类)
- [线程优先级：1~10 的提示机制（MaxPriorityThread 不是 JDK 类）](#线程优先级110-的提示机制maxprioritythread-不是-jdk-类)
- [join() 阻塞的是调用者：join、sleep、yield 的对比与正确理解](#join-阻塞的是调用者joinsleepyield-的对比与正确理解)
- [BLOCKED 是抢锁失败者的状态：线程六状态速查](#blocked-是抢锁失败者的状态线程六状态速查)
- [Thread 没有 isStatus 方法，查状态用 getState 与 isAlive](#thread-没有-isstatus-方法查状态用-getstate-与-isalive)

---

## Thread 是怎样的类：OS 线程的封装 + 任务/线程双重身份
- 2026-09-21

**结论**：Thread 是 Java 对操作系统线程的面向对象封装（平台线程与内核线程 1:1 映射），它 `implements Runnable`，同时扮演"任务载体"和"线程控制句柄"两个角色。这个双重身份是历史设计，后来被 Executor 框架（任务/执行分离）和 JDK 21 虚拟线程修正。

**细节**：

```java
public class Thread implements Runnable {
    public synchronized void start() {
        start0();   // private native void start0(); 底层调 pthread_create / CreateThread
    }
    public void run() {          // 新线程由 OS 调度后，JVM 回调 run()
        if (target != null) { target.run(); }
    }
}
```

- 创建一个平台线程 = 消耗一个内核线程（默认约 1MB 栈 + 系统调用），成本高——这是线程池（复用）和虚拟线程（JVM 自调度）的动机；
- `run()` 是"做什么"（任务），`start()` 是"开一条新执行流去跑"——两者分离正是双重角色的体现；
- 推荐实现 Runnable 而非继承 Thread：不占继承名额、任务与执行分离、同一 task 可交给线程池；

生命周期六状态：`NEW → RUNNABLE ⇄ (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED`。

核心方法速查：

| 方法 | 作用 | 备注 |
|---|---|---|
| `start()` | 创建新线程并触发 run() | 只能调一次，再调抛 IllegalThreadStateException |
| `run()` | 任务体本身 | 直接调用 = 普通方法调用，不启新线程 |
| `interrupt()` | 设置中断标志 | 协作式通知，不强停 |
| `join()` | 等待目标线程结束 | 底层是 wait() 循环 |
| `sleep()` | 静态方法，当前线程休眠 | 不释放锁 |
| `setDaemon(true)` | 设守护线程 | 必须 start 前设置；JVM 不等守护线程 |
| `stop()/suspend()/resume()` | —— | 均已废弃 |

演进：JDK 1.5 juc 用 Executor 把"提交任务"与"管理线程"分离；JDK 21（JEP 444）虚拟线程 `Thread.ofVirtual().start(task)` 保留 Thread 外壳、替换底层实现，以兼容多年生态。

**坑**：`t.run()` 只是方法调用；`start()` 只能调一次；`sleep()` 不释放锁；守护线程的 finally 不保证执行。

## Thread 能被任意多个类继承，单继承限制的是"你自己的类"
- 2026-09-21

**结论**：Thread 不是 final，可以被任意多个类同时继承，互不冲突；Java 单继承限制作用在每个类自己身上——你的类一旦 `extends Thread`，就占用了唯一的继承名额，不能再继承其他类。

**细节**：

```java
class DownloadThread extends Thread { ... }
class LogThread extends Thread { ... }      // ✓ 任意多个类同时继承 Thread
class MyTask extends Thread { }             // ✓ 但 MyTask 不能再 extends 别的类
```

- Thread 自身声明是 `class Thread extends Object implements Runnable`，继承 Thread 的类天然"是一个" Runnable；
- Java 选单继承是为避免菱形继承问题，多态扩展能力由接口补偿——这是"实现 Runnable 优于继承 Thread"的深一层原因。

## 线程优先级：1~10 的提示机制（MaxPriorityThread 不是 JDK 类）
- 2026-09-21

**结论**：`MaxPriorityThread` / `MinPriorityThread` 不是 JDK 自带的类，是教材示例里自定义的 Thread 子类，用于演示优先级。优先级只是给 OS 调度器的**建议**：优先级高的线程**倾向于**分到更多 CPU 时间片，仅此而已。

**细节**：

| 常量 | 值 | 含义 |
|---|---|---|
| `Thread.MIN_PRIORITY` | 1 | 最低 |
| `Thread.NORM_PRIORITY` | 5 | 默认（新线程继承创建者的优先级） |
| `Thread.MAX_PRIORITY` | 10 | 最高 |

示例类的典型样子：

```java
class MaxPriorityThread extends Thread {
    public void run() {
        setPriority(Thread.MAX_PRIORITY);   // 10
        // ... 演示代码
    }
}
```

**坑**：优先级不可靠——跨平台行为不一致（不少 Linux 环境 JVM 直接忽略）；绝不能拿优先级做正确性逻辑，需要确定顺序就用锁、join、队列等同步机制。

## join() 阻塞的是调用者：join、sleep、yield 的对比与正确理解
- 2026-09-21

**结论**：`t.join()` = **调用它的线程**阻塞等待 t 执行结束（"插队"影响的是调用者，t 本身完全不受影响）；`Thread.sleep(ms)` 永远让当前正在执行这行代码的线程休眠（写 `t.sleep()` 睡的也是当前线程）；`yield()` 只是给调度器的礼貌建议，可能被完全无视。

| 方法 | 作用 | 调用后状态 |
|---|---|---|
| `t.join()` | 等别人：当前线程阻塞到 t 结束 | WAITING |
| `Thread.sleep(ms)` | 睡自己：当前线程休眠指定毫秒 | TIMED_WAITING |
| `Thread.yield()` | 让位子：提示调度器让出 CPU | 仍是 RUNNABLE |

**细节**：

```java
t.start();
t.join();   // main 在此阻塞，等 t 跑完才继续
```

- 重载：`join()` 无限等；`join(2000)` 最多等 2 秒，超时后调用方继续（t 可能没跑完）；
- 底层原理就是 wait/notify——join 是 synchronized 方法，核心是 `while (isAlive()) wait(0);`，线程终止时 JVM 自动 `notifyAll()`；
- 等待一组线程完成，现代更常用 `CountDownLatch`、`ExecutorService.awaitTermination()`、`Future.get()`；
- join"看起来不起作用"的两种情况：t 还没 start()（isAlive() 为 false 直接返回）、t 已结束。

**坑**：join 不能用于暂停/控制别的线程——只能"牺牲自己等它"；join 和 sleep 都响应 interrupt() 抛 InterruptedException；两个线程互相 join 对方会死锁；sleep 不释放锁（wait 才释放）；yield 几乎无生产用途。

## BLOCKED 是抢锁失败者的状态：线程六状态速查
- 2026-09-21

**结论**：BLOCKED 是**竞争 synchronized 锁失败、正在锁入口排队**的线程的状态；拿到锁在临界区里执行的是 RUNNABLE。注意两个例外：等 `Lock.lock()` 的线程是 WAITING（AQS 底层 park），不是 BLOCKED；阻塞在 IO 上的线程状态也是 RUNNABLE（JVM 未为 IO 等待单独建模）。

**细节**：

```
A: 拿到锁 ── RUNNABLE（执行临界区）── 释放锁
B: 尝试拿锁 ── BLOCKED（等锁）── 抢到锁 ── RUNNABLE
```

| 状态 | 进入原因 |
|---|---|
| NEW | 创建了还没 start |
| RUNNABLE | 运行 / 等 CPU / **阻塞在 IO** |
| BLOCKED | **等 synchronized 的锁** |
| WAITING | `wait()`、`join()`、`LockSupport.park()`（含 Lock.lock 等待） |
| TIMED_WAITING | `sleep(ms)`、`wait(ms)`、`join(ms)` |
| TERMINATED | run() 结束 |

补充：wait() 被唤醒后要**重新竞争锁**，抢到之前的间隙是 BLOCKED——线程可在 WAITING 和 BLOCKED 之间来回。jstack 排查技巧：等 synchronized 显示 `waiting for monitor entry`，等 ReentrantLock 显示 `parking to wait for`。

## Thread 没有 isStatus 方法，查状态用 getState 与 isAlive
- 2026-09-21

**结论**：JDK 的 Thread 类没有 `isStatus` 方法（教材/代码里见到必是自定义）。查状态用 `getState()`（返回 Thread.State 枚举），查活性用 `isAlive()`（已 start 且未死亡，等价于 state 不是 NEW 且不是 TERMINATED）。

**细节**：

| 方法 | 回答的问题 |
|---|---|
| `isAlive()` | 线程活着吗 |
| `isInterrupted()` | 中断标志被设置了吗（不清除标志） |
| `isDaemon()` | 是守护线程吗 |
| `Thread.holdsLock(obj)` | 当前线程持有 obj 的锁吗（静态） |
| `getState()` | 六种状态里处于哪一种 |

用途：监控诊断（jstack 本质就是批量查状态）、测试断言、验证并发现象。

**坑**：这些是只读观察器，返回的只是调用瞬间的快照——`if (t.getState() == TERMINATED)` 是 check-then-act 竞态。需要确定性用 join()、wait/notify 或 Condition；is/get 开头的方法只配用来看，不配用来做决定。
