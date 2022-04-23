# 在 synchronized 和 ReentrantLock 之间的选择
#Java #Multithreading 

ReentrantLock 虽然在加锁和内存上提供的语义和 synchronized 相同的同时还提供了额外的定时、可中断、公平锁等特性，并且性能还似乎优于 synchronized。

但与 ReentrantLock 相比，内置锁仍然具有以下优势：

1. 内置锁被大多数开发人员所熟悉，并且简洁紧凑，现有大多数程序都使用了内置锁。
2. ReentrantLock 由于需要手动在 finally 中释放锁，相比内置锁的危险性要高。
3. 在线程转储中能给出在哪些调用帧中获得了哪些锁，并能够检测和识别发生死锁的线程。

Java 6.0 之后，ReentrantLock 可通过提供的一个管理和调试的接口，锁可以通过该接口进行注册，从而与 ReentrantLock 相关的加锁消息就能出现在线程转储中，并通过其他的管理接口和调试接口来访问。

线程转储上 ReentrantLock 的非块结构特性仍然意味着，获取锁的操作不能与特定的栈帧关联起来，而内置锁可以。

在一些内置锁无法满足需求的场景下，ReentrantLock 可以作为一种提供更多特性的高级工具。
