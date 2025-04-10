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
# 二、实现RCU应该满足的条件
1. 必须有读端原语(比如rcu_read_lock()和 rcu_read_unlock())和宽限期原语(比如synchronize_rcu()和call_rcu())，
任何在宽限期开始前就存在的RCU读端临界区必须在宽限期结束前完毕。‌<br  />

2. RCU读端原语应该有最小的开销。特别是应该避免如高速缓存未命中、原子操作、内存屏障和分支之类的操作。‌<br  />

3. RCU读端原语应该有O(1)的时间复杂度，可以用于实时用途。这意味着读者可以与更新者并发运行。‌<br  />

4. RCU读端原语应该在所有上下文中都可以使用（在Linux内核中，只有空循环时不能使用RCU读端原语）。
一个重要的特例是RCU读端原语必须可以在RCU读端临界区中使用，换句话说，必须允许RCU读端临界区嵌套。‌<br  />

5. RCU读端原语不应该有条件判断，不会返回失败。这个特性十分重要，因为错误检查会增加复杂度，让测试和验证变得更复杂。‌<br  />

6. 除了静止状态以外的任何操作都能在RCU读端原语里执行。比如像I/O这样的不幂等（non-idempotent）的操作也该允许。‌<br  />

7. 应该允许在RCU读端临界区中执行的同时更新一个受RCU保护的数据结构。‌<br  />

8. RCU读端和更新端的原语应该在内存分配器的设计和实现上独立，换句话说，同样的RCU实现应该能在不管数据原语是分配还是释放的同时保护该数据元素。‌<br  />

9. RCU宽限期不应该被在RCU读端临界区之外阻塞的线程而阻塞。

---
# 三、C++实现的简单RCU举例

代码如下：
```
#include <atomic>
#include <memory>
#include <mutex>
#include <vector>
#include <thread>
#include <chrono>

template<typename T>
class SimpleRCU {
private:
    std::atomic<std::shared_ptr<T>> current_data;
    std::vector<std::shared_ptr<T>> garbage;
	
	std::mutex write_mutex;
	volatile long rcu_Idx;
	
	std::atomic<int> rcu_threads;		// 统计线程数
	
	thread_local int rcu_Refcnt[2];		// 防止读者让写者饥饿
	thread_local int rcu_nesting;		// 递归次数，允许嵌套lock()
    thread_local int rcu_readIdx;		// 写更新时使用

public:
    explicit SimpleRCU(std::shared_ptr<T> init) : current_data(init), rcu_threads(0) {
		rcu_Idx = 0；
		rcu_Refcnt[0] = rcu_Refcnt[1] = 0;
		rcu_nesting = 0;
		rcu_readIdx = 0；
	}
	
	void readThreadsAdd() {
		std::atomic_thread_fence(std::memory_order_acquire);
		rcu_threads.fetch_add(1, std::memory_order_relaxed);
	}
	
	void readThreadsSub() {
		std::atomic_thread_fence(std::memory_order_acquire);
		rcu_threads.fetch_sub(1, std::memory_order_relaxed);
	}
	
	void rcu_lock() {
		std::atomic_thread_fence(std::memory_order_acquire);
		if (rcu_nesting == 0) {
			rcu_readIdx = rcu_Idx & 0x1;
			rcu_Refcnt[rcu_readIdx]++;
		}
		
		rcu_nesting++;
		std::atomic_thread_fence(std::memory_order_release);
	}
	
	void rcu_unlock() {
		std::atomic_thread_fence(std::memory_order_acquire);
		if (rcu_nesting == 1) {
			rcu_Refcnt[rcu_readIdx]--;
		}
		
		rcu_nesting--;	
		std::atomic_thread_fence(std::memory_order_release);
	}
	
	void flip_counter_and_wait(int ctr) {
		rcu_Idx = ctr + 1;
		int i = ctr & 0x1;
		std::atomic_thread_fence(std::memory_order_acquire);
		// 对每一线程
		for (int m = 0; m < rcu_threads.load(); ++m) {
			while (rcu_Refcnt[i] != 0) {
				std::this_thread::sleep_for(std::chrono::milliseconds(10));
			}
		}
	}
	
	bool synchronize_rcu() {
		std::atomic_thread_fence(std::memory_order_acquire);
		int oldctr = rcu_Idx;
		
		// 最多只有3个线程在等待计数变为0
		if ((rcu_Idx - oldctr) >= 3) { 
			std::atomic_thread_fence(std::memory_order_release);
			return false;
		}
		
		flip_counter_and_wait(rcu_Idx);
		if (rcu_Idx - oldctr) < 2)
			flip_counter_and_wait(rcu_Idx + 1);
		
		std::atomic_thread_fence(std::memory_order_release);
		return true;
	}

    // 读者访问（无锁）
    std::shared_ptr<T> read() {
		rcu_lock();
		auto ptr = current_data.load(std::memory_order_relaxed);
		rcu_unlock();
		return ptr;
    }

    // 写者更新
    void write(auto&& updater) {
        // 1. 复制新数据
        auto new_data = std::make_shared<T>(*current_data);
        updater(*new_data);
        
        // 2. 原子替换指针
        auto old_data = current_data.exchange(new_data);
		
		std::lock_guard<std::mutex> lock(write_mutex);
		
        // 将旧的数据记录到队列中
		garbage.push_back(old_data);
		
        // 3. 等待当前读者退出
        if(!synchronize_rcu()) return;
  
        // 4. 安全回收旧数据
        garbage.erase(
            std::remove_if(garbage.begin(), garbage.end(),
                [](auto& ptr) { return ptr.use_count() == 1; }),
            garbage.end());
    }
};

// 使用示例
struct Config {
    int value = 0;
    std::string name = "default";
};

int main() {
    SimpleRCU<Config> rcu(std::make_shared<Config>());
    
    // 读者线程
    auto reader = [&] {
		rcu.readThreadsAdd();
        auto data = rcu.read();
        printf("Read value: %d, name: %s\n", 
               data->value, data->name.c_str());
		rcu.readThreadsSub();
    };
    
    // 写者线程
    auto writer = [&] {
        rcu.write([](Config& cfg) {
            cfg.value++;
            cfg.name = "updated_" + std::to_string(cfg.value);
        });
    };
    
    std::thread t1(reader);
    std::thread t2(reader);
    std::thread t3(writer);
    
    t1.join();
    t2.join();
    t3.join();
    
    return 0;
}
```

---
# 四、RCU宽限期

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
# 五、关键特性

#### 1. RCU是读/写锁的替代者
Linux内核中RCU最常见的用途就是在读占大多数时间的情况下替换读/写锁。<br  />
RCU和读/写锁最关键的相似之处在于两者都有可以并行执行的读端临界区。<br  />
RCU的优点在于性能、没有死锁，还有实时的延迟。RCU也有一点缺点，比如读者与更新者并发执行，比如低优先级RCU读者可
以阻塞正等待宽限期完毕的高优先级线程，还比如宽限期的延迟可以有好几毫秒。

#### 2. RCU是一种受限制的引用计数机制
因为宽限期不能在RCU读端临界区进行时完毕，所以RCU读端原语可以像受限的引用计数机制一样使用。

#### 3. RCU是一种可大规模使用的引用计数机制
传统的引用计数通常与某种或者一组数据结构有联系。然而，维护大量不同种类的数据结构的单一全局引用计数，通常会导致包含引用计数的缓存行来回“乒乓”。这种缓存行“乒乓”会严重影响系统性能。<br  />
相反，RCU的轻量级读端原语允许读端极其频繁地调用，却只带来微不足道的性能影响，这使得RCU可以作为一种几乎没有惩罚的“批量引用计数机制”（bulk reference-counting）。当某个任务需要在一系
列代码中持有引用时，可以使用可休眠RCU（SRCU）。

#### 4. RCU是穷人版的垃圾回收器（garbage collector）
RCU与GC有2点不同：<br  />
（1）程序员必须手动指示何时可以回收指定数据结构；<br  />
（2）程序员必须手动标出可以合法持有引用的RCU读端临界区。<br  />

#### 5. RCU是一种提供存在担保的方法
任何受RCU保护的数据元素在RCU读端临界区中被访问，那么数据元素在RCU读端临界区持续期间保证存在。

#### 6. RCU是一种提供类型安全内存的方法
很多无锁算法并不需要数据元素在被RCU读端临界区引用时保持
完全一致，只要数据元素的类型不变就可以了。换句话说，只要结构
类型不变，无锁算法可以允许某个数据元素在被其他对象引用时可以
释放并重新分配，但是决不允许类型上的改变。这种“保证”，在学术
文献中被称为“类型安全的内存”（type-safe meomry）。<br  />

类型安全的内存算法在Linux内核中的应用是slab缓存 ， 被SLAB_DESTROY_BY_RCU标记专门标出来的缓存通过RCU将已释放
的slab返回给系统内存。在任何已有的RCU读端临界区持续期间，使用RCU可以保证所有带有SLAB_DESTROY_BY_RCU标记且正在使用的
slab元素仍然在该slab中，类型保持一致。

#### 7. RCU是一种等待事物结束的方式
RCU的强大之处，其中之一就是允许你在等待上千个不同事物结束的同时，又不用显式地去跟踪其中每一个，因此也就无需担心性能
下降、扩展限制、复杂的死锁场景、内存泄露等显式跟踪机制自身的问题。

#### 8. RCU读端临界区必须提供的进度保证
1). RCU BH：读端临界区必须保证除了NMI和IRQ处理函数以外的进度保证，但不包括软（softirq）中断。RCU BH是全局的。<br  />
2). RCU Sched：读端临界区必须保证除了NMI和IRQ处理函数以外的进度保证，包括软中断。RCU Sched是全局的。<br  />
3). RCU（classic和realtime）：读端临界区必须保证除了NMI、IRQ、软中断处理函数和高优先级实时任务（在realtime RCU中）以外的进度保证，包括软中断。RCU是全局的。<br  />
4). SRCU和QRCU：读端临界区不提供进度保证，除非某些任务在等待对应的宽限期完成，此时这些读端临界区应该在不超过数秒内完成。SRCU和QRCU的范围由各自对应的srcu_struct或者qrcu_struct定义。

#### 9. 总结

|    特性	 |      实现方式	   |   性能影响    |
|------------|---------------------|---------------|
| 无锁读	 | 原子计数器+共享指针 | O(1) 无竞争   | 
| 写者互斥	 | 互斥锁保证单写者	   | 写操作串行化  | 
| 宽限期检测 |	等待读者计数器归零 | 潜在忙等	   |
| 旧数据回收 |	引用计数+定期清理  | 延迟回收	   |

---
# 六、适用场景
‌- **配置热更新‌**：需要频繁读取但极少修改的全局配置<br  />‌
‌- **统计计数器**‌：高频读取，允许短暂计数不一致的统计场景<br  />‌
‌- **只读缓存系统**‌：缓存版本切换时无需立即失效旧数据<br  />‌
