---
title: "RCU同步机制"
collection: system
permalink: /system/system-09
excerpt: ' '
date: 2025-03-13
citation: 'Joe-Bi. (2025). &quot;RCU同步机制.&quot; <i>GitHub Joe-Bi of blog</i>'
---


RCU是一种同步机制，适用于读多写少的场景，读者不需要加锁，而写者需要处理数据的更新和旧数据的回收。‌<br  />

RCU的核心是读者在读取数据时不需要锁，写者更新数据时生成新副本，并在所有读者不再引用旧数据后回收它。这需要一种机制来跟踪读者的访问状态。常见的方法是使用“宽限期”（grace period）来确定何时可以安全回收旧数据。<br  />

为了实现宽限期，可能需要为每个读者线程维护一个计数器或标记，记录它们是否处于临界区（即正在读取数据）。写者在更新数据后，等待所有当前正在执行的读者退出临界区，然后回收旧数据。这里的关键是高效地跟踪读者的状态，而不会引入太大的开销。<br  />

# 一、执行流程
### 1. 写操作开始
当写线程发起写操作时，它不会直接对原数据进行修改，而是先创建一个新的数据副本，把新的数据写入该副本，接着通过原子操作将共享指针指向这个新的数据副本。在这个过程中，读线程依旧可以无障碍地访问旧数据，因为共享指针的更新是原子操作，读线程不会察觉到数据的变更。
### 2. 宽限期等待
写线程完成新数据的替换之后，会进入宽限期。在宽限期内，写线程需要等待所有正在执行的读操作都结束，也就是要等到所有读线程都离开了它们的临界区。这是通过对每个 CPU 上的读线程状态进行跟踪来实现的。例如，Linux 内核里会使用 rcu_read_lock() 和 rcu_read_unlock() 来标记读线程的临界区，内核会监测每个 CPU 上是否还有处于临界区内的读线程。
### 3. 旧数据释放
一旦宽限期结束，也就意味着所有读线程都已经完成了对旧数据的访问，写线程就可以安全地释放旧数据所占用的内存空间。

---
# 二、C++实现的简单RCU举例

代码如下：
```
#include <iostream>
#include <thread>
#include <atomic>
#include <vector>
#include <mutex>
#include <chrono>

// 定义一个简单的数据结构
struct Data {
    int value;
    Data(int v) : value(v) {}
};

// 全局共享数据指针
std::atomic<Data*> sharedData;

// 写锁和读锁计数器
std::mutex writeMutex;
std::atomic<int> readCount(0);

// 模拟 RCU 的优雅周期
void synchronizeRCU() {
    // 等待所有读者完成
    while (readCount.load() > 0) {
        std::this_thread::yield();
    }
}

// 读操作
void reader() {
    // 增加读者计数
    readCount.fetch_add(1, std::memory_order_relaxed);

    // 读取共享数据
    Data* data = sharedData.load(std::memory_order_acquire);
    if (data) {
        std::cout << "Reader read value: " << data->value << std::endl;
    }

    // 减少读者计数
    readCount.fetch_sub(1, std::memory_order_relaxed);
}

// 写操作
void writer(int newValue) {
    // 加写锁
    std::lock_guard<std::mutex> lock(writeMutex);

    // 创建新的数据副本
    Data* newData = new Data(newValue);

    // 原子地更新共享数据指针
    Data* oldData = sharedData.exchange(newData, std::memory_order_release);

    // 等待所有读者完成（优雅周期）
    synchronizeRCU();

    // 释放旧数据
    delete oldData;
}

int main() {
    // 初始化共享数据
    sharedData.store(new Data(0), std::memory_order_release);

    // 创建多个读者和写者线程
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(reader);
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    threads.emplace_back(writer, 10);

    // 等待所有线程完成
    for (auto& thread : threads) {
        thread.join();
    }

    // 释放最后一个共享数据
    delete sharedData.load();

    return 0;
}
```

---
# 三、RCU宽限期

RCU是一种以牺牲一定数据一致性来换取高并发性能的同步机制。它并不保证所有读线程在同一时刻看到相同的数据状态。在写操作更新共享数据后，由于指针的原子更新，后续开始的读操作会看到新数据，而在写操作更新指针之前就已经开始的读操作会继续使用旧数据。这种 “不一致性” 是有意为之的，因为它避免了读写锁机制中读操作和写操作之间的互斥等待，使得读操作可以无锁进行，从而提高了系统的并发性能。<br  />

宽限期的主要作用是确保旧数据在不再被任何读线程引用之后才被释放，避免悬空指针问题。即使在宽限期之前有新的读线程读取到了新数据，宽限期仍然会等待所有在写操作之前就已经开始读旧数据的线程完成操作，然后才释放旧数据，以此避免悬空指针问题，这样可以保证数据的一致性和程序的稳定性，保证数据的安全性，不会因为提前释放旧数据而导致读线程访问到无效的内存。

### 宽限期的意义
#### 1. 数据一致性保障
宽限期确保了读线程在访问数据的过程中，数据不会突然被释放或者修改。即使写操作提前开始，只要读线程还在访问旧数据，宽限期就会阻止写线程释放旧数据，从而保证读线程能够完整、正确地读取到数据。
#### 2. 避免悬空指针
如果没有宽限期，写线程在旧数据还被读线程引用时就释放了旧数据，那么读线程后续再访问该数据时就会产生悬空指针，进而引发程序崩溃或者产生不可预期的结果。宽限期的存在有效地避免了这种情况的发生。
#### 3. 并发性能优化
宽限期允许读线程和写线程在一定程度上并发执行。读线程不需要等待写操作完成，写线程也不需要等待所有读线程都开始执行。只要在宽限期内所有读线程都能完成对旧数据的访问，就可以保证数据的安全性和一致性，从而提高了系统的并发性能。

---
# 四、关键特性

|    特性	 |      实现方式	   |   性能影响    |
|------------|---------------------|---------------|
| 无锁读	 | 原子计数器+共享指针 | O(1) 无竞争   | 
| 写者互斥	 | 互斥锁保证单写者	   | 写操作串行化  | 
| 宽限期检测 |	等待读者计数器归零 | 潜在忙等	   |
| 旧数据回收 |	引用计数+定期清理  | 延迟回收	   |

---
# 五、适用场景
‌- **配置热更新‌**：需要频繁读取但极少修改的全局配置<br  />‌
‌- **统计计数器**‌：高频读取，允许短暂计数不一致的统计场景<br  />‌
‌- **只读缓存系统**‌：缓存版本切换时无需立即失效旧数据<br  />‌
