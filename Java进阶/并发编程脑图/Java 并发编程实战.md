# Java 并发编程实战

## 15.原子变量与非阻塞同步机制

### 锁的劣势

- 非阻塞算法

	- 底层的原子机器指令代替锁来确保数据再并发访问中的一致性
	- 应用场景

		- 操作系统和 JVM 中实现线程/进程调度机制
		- 垃圾回收机制
		- 锁和其他并发数据结构

- 原子变量

	- 提供了与 volatile 类型变量相同的内存语义，此外还支持原子的更新操作
	- 适用场景

		- 计数器
		- 序列发生器
		- 统计数据收集

- 锁的劣势

	- 多线程请求锁，一些线程会被挂起。即使线程恢复也需要等待其他线程执行完其对应的时间片。挂起和恢复线程存在很大的性能开销
	- volatile 的局限

		- 虽然提供可见性的保证，但是不能用于原子的复合操作
		- 当新值依赖于旧值时，就不能使用 volatile

	- 当一个线程正在等待锁时，他不能做任何其他事情，如果持有锁的线程被永久阻塞，等待这个锁的线程就永远无法执行下去
	- 锁定方式对于细粒度的操作仍然是一种高开销的机制
	- 优先级反转

		- 如果被阻塞的线程优先级比较高，而持有锁的线程优先级比较低，这样会导致高优先级线程降级至低优先级的级别

### 硬件对并发的支持

- CAS（比较并交换）

	- 冲突检查机制

		- 判断在更新的过程中是否存在来自其他线程的干扰，如果存在操作置为失败，并且可以重试或者放弃

	- 几乎所有线代处理器都包含了某种形式的原子「读-改-写」指令

		- 比较并交换（CAS）
		- 关联加载/条件存储

	- CAS 的三个操作数

		- 需要读写的内存位置 V
		- 进行比较的值 A
		- 拟写入的新值 B

	- 当且仅当 V 的值等于 A 时，CAS 才会通过原子方式用新值 B 来更新 V 的值，（比较并设置，Compare and Set）否则不执行任何操作，直接返回 V 原有的值

		- 我认为 V 的值应该为 A，如果是，那么将 V 的值更新为 B，否则不修改并告诉 V 的值实际为多少。

	- 多个线程尝试 CAS 的时候，只有其中一个线程能更新变量的值，其他线程都将失败，但是不会线程挂起

- 非阻塞的计数器

	- 当竞争程度不高时，基于 CAS 的计数器在性能上远远超过了基于锁的计数器
	- CAS 的缺点

		- 让调用者自己处理竞争问题（重试、回退、放弃）
		- 锁能自动处理竞争问题（阻塞、挂起）

- JVM 对 CAS 的支持

	- 最坏的情况下，如果 JVM 不支持 CAS 指令，则会使用自旋锁
	- 原子变量类（java.util.concurrent.atomic）

		- JVM 支持为数字类型和引用类型提供一种高效的 CAS 操作
		- java.util.concurrent 中大多数类在实现时直接或者间接使用了原子变量类

### 原子变量类

- 原子变量是一种「更好的 volatile」，不想基本类型的包装类不可修改，原子变量类是可以修改的

	- 原子变量比锁的粒度更细，量级更轻 
	- 原子变量类分组

		- 标量类

			- AtomicInteger
			- AtomicLong
			- AtomicBoolean
			- AtomicReference

		- 更新器类

			- AtomicReferenceFieldUpdater

		- 数组类

			- AtomicIntegerArray
			- AtomicLongArray
			- AtomicReferenceArray

		- 复合变量类

- 性能比较：锁与原子变量

	- 高度竞争

		- 锁的性能超过原子变量的性能，因为锁的机制是将线程挂起，所以会降低 CPU 的使用率

	- 更真实的竞争

		- 原子变量的性能超过锁的性能

	- 中低程度的竞争

		- 原子变量提供更高的可伸缩性

### 非阻塞算法

- 非阻塞的栈

	- 非阻塞算法

		- 如果在某种算法中，一个线程的失败或挂起不会导致其他线程也失败或挂起，这种算法就是「非阻塞算法」

	- 无锁算法

		- 如果在非阻塞的算法中每个步骤都存在某个线程能够执行下去，那么就是「无锁算法」

	- 非阻塞算法中通常不会产生死锁和优先级反转问题

- 非阻塞的链表
- 原子的域更新器
- ABA 问题

	- 如果 V 的值首先由 A 变成 B，再由 B 变成 A，那么仍然要认为发生了变化
	- 解决方案

		- 加版本号

## 16.Java 内存模型

对某个线程的内存操作在哪些情况下对于其他线程是可见的说明

### 什么是内存模型，为什么需要它

- 平台的内存模型

	- 使得线程无法看到变量的最新值的几种情况

		- 缺少同步
		- 指令重排序，编译器还会把变量保存在寄存器而不是内存中
		- 处理器可以采用乱序或者并行的方式执行指令
		- 缓存改编将写入变量提交到主内存的次序
		- 保存在处理器本地缓存中的值对于其他处理器是不可见的

	- 内存栅栏

		- 在架构定义的内存模型中将告诉应用程序可以从内存系统中获得怎样的保证，此外还订一块了一些特殊的指令
		- 这些特殊的指令就是「内存栅栏」，当需要数据共享时，这些指令就能实现额外的存储协调放置出现一些奇怪的情况。

- 重排序

	- 各种使操作延迟或者看似乱序执行的不同原因，都可以归为重排序
	- 内存及的重排序会使程序的行为变得不可预测

- Java 内存模型简介

	- Java 的内存模型是通过各种操作来定义的

		- 对变量的读写操作
		- 监视器的加锁与释放操作
		- 线程的启动和合并操作

	- Happens-Before 关系

		- 一种偏序关系，保证执行操作 B 的线程看到操作 A 的结果

	- 程序顺序规则

		- 如果在程序中操作 A 先于操作 B，那么线程中 A 操作也要先于操作 B

	- 监视器锁规则

		- 在监视器锁上的操作必须在用一个监视器锁上的加锁操作之前执行

	- volatile 变量规则

		- 对 volatile 变量的写入操作必须在对该变量的读操作之前执行

	- 线程启动规则

		- 在线程上对 Thread.start 的调用必须在该线程中执行任何操作之前执行

	- 线程结束规则

		- 线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从 Thread.join 中成功返回，或者在调用 Thread.isAlive 时返回 false

	- 中断规则

		- 当一个线程在另一个线程调用 interrupt 时，必须在被中断线程监测到 interrupt 调用之前执行

	- 终结器规则

		- 对象的构造函数必须在启动该对象的终结器之前执行完成

	- 传递性

		- 如果操作 A 先于操作 B，并且操作 B 先于操作 C，那么操作 A 必须先于操作 B

- 借助同步

	- 安全发布

		- 如果一个线程将对象置入队列并且另一个线程随后获取这个对象

	- Java 类库提供的 Happens-Before 排序规则

		- 将一个元素放入一个线程安全容器的操作将在另一个线程从该容器中获得这个元素的操作之前执行
		- 在 CountDownLatch 上的倒数操作将在线程从闭锁上的 await 方法中返回之前执行
		- 释放 Semaphore 许可的操作将在从该 Semaphore 上获得一个许可之前执行
		- Future 表示的任务的所有操作将在从 Future.get 中返回之前执行
		- 向 Excutor 提交一个 Runnable 或 Callable 的操作将在任务开始执行之前执行
		- 一个线程到达 CycliicBarrier 或 Exchanger 的操作姜在其他到达该栅栏或交换点的仙鹤草呢个被释放之前执行。

### 发布

- 不安全的发布

	- 当缺少 happens-Before 关系时，就可能出现重排序的问题
	- 被部分构造对象，错误的延迟初始化，双重检查锁问题，多线程环境下的懒汉式单例。

- 安全的发布

	- 使用一个锁保护共享变量
	- 使用共享的 volatile 类型变量

- 安全初始化模式

	- 提前初始化

		- 参考饿汉式单例

	- 延迟初始化占位类模式，也可以叫做按需初始化，参考单例的「IoDH」写法

- 双重检查加锁

	- 在没有同步的情况下读取一个共享对象时，可能发生最糟糕的事情只是看到一个失效值
	- 对共享变量声明 volatile，解决该问题，但是性能开销变大

### 初始化过程中的安全性

- 通过 final 域科大的值从构造过程完成时开始的可见性

