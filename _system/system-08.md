---
title: "几种读写锁的实现方式以及读写锁与RCU比较"
collection: system
permalink: /system/system-08
excerpt: ' '
date: 2025-03-13
citation: 'Joe-Bi. (2025). &quot;几种读写锁的实现方式以及读写锁与RCU比较.&quot; <i>GitHub Joe-Bi of blog</i>'
---


# 一、特点和目标

读写锁旨在解决‌读多写少场景下的并发效率问题‌。通过区分读操作（共享）和写操作（独占），允许多个读线程并行访问资源，而写线程独占资源，显著提升系统吞吐量‌。

---
# 二、实现原理
读写锁可以基于不同的底层机制来实现，常见的是使用自旋锁（Spin Lock）和条件变量（Condition Variable）的组合，也有使用普通锁（如互斥锁mutex）和条件变量（Condition Variable）的组合。

## 1. 基于自旋锁和条件变量实现

这种实现方式结合了自旋锁的忙等待特性和条件变量的阻塞等待特性。自旋锁用于在短时间内快速获取和释放锁，以减少线程切换的开销；条件变量用于在需要等待资源时让线程进入阻塞状态，避免无谓的CPU占用。。

实现代码：
```
#include <pthread.h>

typedef struct {
    int readers;	     // 记录当前读者的数量
    int writers;	     // 记录当前写者的数量（通常为 0 或 1）
    spinlock_t lock;	     // 自旋锁，用于保护共享数据
    struct condvar read_cv;  // 读者条件变量
    struct condvar write_cv; // 写者条件变量
} rwlock_t;

void rwlock_read_lock(rwlock_t *rwlock) {
    spin_lock(&rwlock->lock);
    // 如果有写者正在访问，读者需要等待
    while (rwlock->writers > 0) {
        cond_wait(&rwlock->read_cv, &rwlock->lock);
    }
    rwlock->readers++;
    spin_unlock(&rwlock->lock);
}

void rwlock_read_unlock(rwlock_t *rwlock) {
    spin_lock(&rwlock->lock);
    rwlock->readers--;
    // 如果没有读者了，唤醒一个等待的写者
    if (rwlock->readers == 0) {
        cond_signal(&rwlock->write_cv);
    }
    spin_unlock(&rwlock->lock);
}

void rwlock_write_lock(rwlock_t *rwlock) {
    spin_lock(&rwlock->lock);
    // 如果有读者或其他写者正在访问，写者需要等待
    while (rwlock->readers > 0 || rwlock->writers > 0) {
        cond_wait(&rwlock->write_cv, &rwlock->lock);
    }
    rwlock->writers++;
    spin_unlock(&rwlock->lock);
}

void rwlock_write_unlock(rwlock_t *rwlock) {
    spin_lock(&rwlock->lock);
    rwlock->writers--;
    // 唤醒所有等待的读者和一个等待的写者
    cond_broadcast(&rwlock->read_cv);
    cond_signal(&rwlock->write_cv);
    spin_unlock(&rwlock->lock);
}
```

## 2. 基于互斥锁和条件变量的实现
这种实现方式使用普通的互斥锁（Mutex）来保护共享数据，条件变量用于线程之间的同步。互斥锁在获取锁时可能会导致线程阻塞，适用于锁竞争较为激烈、持有锁时间较长的场景。

使用互斥锁和条件变量又有三种实现方式：‌<br  />
a) 两个mutex  
b) 一个mutex + 一个Condition Variable  
c) 一个mutex + 两个个Condition Variable  

### 2.1 两个mutex实现

```
#include <mutex>

typedef struct {
    std::mutex read_mtx;    	// 保护读计数器和写锁请求状态
    std::mutex write_mtx;   	// 写操作的排他锁
    int readers = 0;        	// 当前活跃读线程数
    bool write_pending = false; // 是否有写线程等待
} rwlock_t；

// 读锁获取
void read_lock(rwlock_t *rwlock) {
    std::lock_guard<std::mutex> lock(rwlock->read_mtx);
    if (rwlock->readers == 0 && !rwlock->write_pending) {
        rwlock->write_mtx.lock();  // 第一个读线程获取写锁（阻塞写线程）
    }
    rwlock->readers++;
}

// 读锁释放
void read_unlock(rwlock_t *rwlock) {
    std::lock_guard<std::mutex> lock(rwlock->read_mtx);
    rwlock->readers--;
    if (rwlock->readers == 0 && rwlock->write_pending) {
        rwlock->write_mtx.unlock(); // 最后一个读线程释放写锁，允许写线程执行‌
    }
}

// 写锁获取
void write_lock(rwlock_t *rwlock) {
    std::lock_guard<std::mutex> lock(rwlock->read_mtx);
    rwlock->write_pending = true;   // 标记写线程等待
    rwlock->write_mtx.lock();       // 阻塞直到所有读线程释放‌
}

// 写锁释放
void write_unlock(rwlock_t *rwlock) {
    rwlock->write_mtx.unlock();
    std::lock_guard<std::mutex> lock(rwlock->read_mtx);
    rwlock->write_pending = false;  // 清除写等待标记‌
}

```
原理与特点‌
1. ‌核心机制‌‌<br  />
‌	 读锁流程‌：‌<br  />
	第一个读线程通过write_mtx.lock()阻止后续写线程‌。‌<br  />
	后续读线程直接增加readers计数器，共享读权限‌。‌<br  />
	最后一个读线程释放write_mtx，允许写线程获取锁‌。‌<br  />
‌	
	写锁流程‌：‌<br  />
	写线程先标记write_pending = true，阻止新读线程启动‌。‌<br  />
	通过write_mtx.lock()等待所有活跃读线程完成‌。‌<br  />

2. ‌特性‌‌<br  />
‌   读优先: 新读线程可抢占写锁请求（write_pending标记后仍允许读）。‌<br  />‌
‌   写线程饥饿风险: 高并发读场景下，写线程可能因write_pending无法及时生效而长期等待‌。‌<br  />
   ‌死锁风险‌: 必须严格保证锁顺序：read_mtx先于write_mtx‌‌。‌<br  />

3. ‌适用场景‌<br  />
‌   读多写极少‌：例如日志系统，允许读线程快速抢占资源。‌<br  />
   ‌低写优先级‌：不要求写操作必须及时执行。‌

### 2.2 一个mutex + 一个Condition Variable实现
```
#include <mutex>
#include <condition_variable>

typedef struct {
    std::mutex mtx;
    std::condition_variable cv;
    int readers = 0;          	// 当前活跃读线程数
    bool writer_active = false; // 写锁是否被持有
    int writers_waiting = 0;  	// 等待中的写线程数
}rwlock_t;

void read_lock(rwlock_t *rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    // 等待条件：无活跃写线程且无写线程等待（避免写线程饥饿）
    rwlock->cv.wait(lock, [this] { 
        return !rwlock->writer_active && rwlock->writers_waiting == 0; 
    });
    rwlock->readers++;
}

void read_unlock(rwlock_t *rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    rwlock->readers--;
    // 若无读线程且存在等待的写线程，唤醒一个写线程‌
    if (rwlock->readers == 0 && rwlock->writers_waiting > 0) {
        rwlock->cv.notify_one();
    }
}

void write_lock(rwlock_t *rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    rwlock->writers_waiting++;
    // 等待条件：无活跃读/写线程‌
    rwlock->cv.wait(lock, [this] { 
        return rwlock->readers == 0 && !rwlock->writer_active; 
    });
    rwlock->writers_waiting--;
    rwlock->writer_active = true;
}

void write_unlock(rwlock_t *rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    rwlock->writer_active = false;
    // 优先唤醒写线程（避免读线程持续抢占）或所有读线程
    if (rwlock->writers_waiting > 0) {
        rwlock->cv.notify_one();
    } else {
        rwlock->cv.notify_all();
    }
}

```
特性‌‌<br  />
    1. 唤醒策略‌: notify_all可能导致读/写线程竞争‌<br  />
    2. ‌公平性控制‌: 需依赖计数器逻辑判断，易导致写线程‌<br  />
    3. 性能损耗‌: 频繁无效唤醒增加上下文切换开销‌

适用场景‌‌<br  />
‌	单条件变量适合低并发场景或读/写操作频率相近的应用（如低频配置更新）。

### 2.3 一个mutex + 两个Condition Variable实现
```
#include <mutex>
#include <condition_variable>

typedef struct {
private:
    std::mutex mtx;
    std::condition_variable read_cv, write_cv;
    int readers = 0;          		// 当前活跃读线程数
    bool writer_active = false; 	// 写锁是否被持有
    int writers_waiting = 0;  		// 等待中的写线程数
}rwlock_t;

void read_lock(rwlock_t* rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    // 等待条件：无活跃写线程‌
    rwlock->read_cv.wait(lock, [this] { 
        return !rwlock->writer_active; 
    });
    rwlock->readers++;
}

void read_unlock(rwlock_t* rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    rwlock->readers--;
    // 若当前无读线程且存在等待的写线程，唤醒一个写线程‌
    if (rwlock->readers == 0 && rwlock->writers_waiting > 0) {
        rwlock->write_cv.notify_one();
    }
}

void write_lock(rwlock_t* rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    rwlock->writers_waiting++;
    // 等待条件：无活跃读/写线程‌
    rwlock->write_cv.wait(lock, [this] { 
        return rwlock->readers == 0 && !rwlock->writer_active; 
    });
    rwlock->writers_waiting--;
    rwlock->writer_active = true;
}

void write_unlock(rwlock_t* rwlock) {
    std::unique_lock<std::mutex> lock(rwlock->mtx);
    rwlock->writer_active = false;
    // 优先唤醒读线程（若存在）或下一个写线程‌
    if (rwlock->writers_waiting > 0) {
        rwlock->write_cv.notify_one();
    } else {
        rwlock->read_cv.notify_all();
    }
}


```
特性‌<br  />
	‌1. 唤醒策略: 通过独立条件变量实现精确唤醒‌<br  />‌
    2. 公平性控制‌: 写线程优先唤醒，避免饥饿‌<br  />
    ‌3. 性能损耗‌: 减少无效唤醒，提升高并发性能‌

‌适用场景‌<br  />
	‌双条件变量‌适用于高并发读写场景（如数据库缓存），需严格避免写线程饥饿‌

---
# 三、差异比较

|       实现方式    |                    核心机制                             |                 性能                        |                      公平性                        |                       适用场景                           | 缺点                                     |
|-------------------|---------------------------------------------------------|---------------------------------------------|----------------------------------------------------|----------------------------------------------------------|------------------------------------------|
| 自旋锁            | 忙等循环检查锁状态，不释放CPU资源                       | 高吞吐（无上下文切换），但持续占用CPU       | 无公平性控制，可能饥饿                             | 临界区极短（如原子操作）、内核实时性要求高的场景         | CPU资源浪费，长时间等待导致系统效率下降  |
| 双互斥锁          | 读/写锁分离：读锁允许多线程共享，写锁独占               | 读操作无阻塞，写操作延迟高（需等待读锁释放）| 读优先，写线程易饥饿                               | 读多写极少（如日志系统），写操作优先级低                 | 写线程可能长期阻塞，无法保证写操作及时性 |
| 1互斥锁+1条件变量 | 通过单一条件变量同步读/写线程，依赖状态变量避免无效唤醒 | 频繁唤醒导致上下文切换开销，高并发性能较差  | 唤醒策略依赖逻辑判断，可能偏向读或写线程           | 低并发读写混合场景（如低频配置更新）                     | 无效唤醒频繁，公平性难以控制             |
| 1互斥锁+2条件变量 | 读/写线程使用独立条件变量队列，精准唤醒减少竞争         | 高并发下吞吐量高，无效唤醒少                | 支持写优先策略（通过writers_waiting计数器避免饥饿）| 高并发读写混合场景（如数据库缓存），需严格避免写线程饥饿 | 实现复杂度高，需精细控制状态变量         |

---
# 四、std::shared_mutex
std::shared_mutex是C++17标准库中新增的一个类，它提供了读写锁机制，可以同时支持多个线程对同一个资源进行读操作，但只能支持一个线程对同一个资源进行写操作。这样可以避免多个线程同时写同一个资源而导致的数据竞争问题，提高程序的并发性能。 

应用举例：
```
#include <iostream>
#include <thread>
#include <shared_mutex>

std::shared_mutex mtx; 
int count = 0; 

// 读锁保护对象被销毁时，自动解读锁
void read() {
    std::shared_lock<std::shared_mutex> lock(mtx); // 创建一个读锁保护对象，自动加读锁
    std::cout << "count = " << count << std::endl; // 读取共享资源
} 

// 写锁保护对象被销毁时，自动解写锁
void write() {
    std::unique_lock<std::shared_mutex> lock(mtx); // 创建一个写锁保护对象，自动加写锁
    ++count; // 修改共享资源
} 

int main() {
    std::thread t1(read);
    std::thread t2(read);
    std::thread t3(write);
	
    t1.join();
    t2.join();
    t3.join();
	
    return 0;
}
```

# 五、 RCU与读写锁

RCU和读写锁的目标都是为了在读多写少的场景下大幅提升性能，大部分场景下，RCU是能取代读写锁的，但是两者的设计目标和实现机制又有些不同。

## 1. RCU的原理
#### 无锁读操作
1). 读者无需加锁：读操作直接访问共享数据，无需任何同步开销。‌<br  />
2).	写操作的延迟生效：写者先修改数据副本，通过指针原子切换使新数据生效，旧数据在所有读者离开临界区后才被释放。
#### 优雅周期
1). 写者更新数据后，需等待所有 CPU 完成一次上下文切换（即所有读者已离开临界区），确保旧数据不再被引用。‌<br  />
2). 内核通过维护每个 CPU 的状态（如rcu_gp机制）实现这一过程。

RCU适用于读极多、写极少的场景，如内核中的路由表、文件系统 inode 缓存等。

## 2. RCU与读写锁的性能对比

|   特性   |             RCU	        |           读写锁		   |
|----------|----------------------------|--------------------------|
| 读性能   |       无锁，性能极高	    |   需加锁，存在竞争开销   |
| 写性能   | 需复制数据，延迟释放旧数据	| 直接修改，无复制开销     |
| 适用场景 |           读多写少	        | 写操作较多或实时性要求高 |
| 内存开销 |  写操作需额外内存存储副本	|       无额外内存开销     |
| 实时性   |    优雅周期可能导致延迟	|       写操作立即生效     |


## 3. 读写锁的不可替代性
尽管RCU性能优异，但读写锁在以下场景中仍不可替代。

#### 1). 写操作频繁的场景
RCU的写操作需要复制数据并等待优雅周期，频繁写会导致内存占用增加和延迟累积。读写锁在写时直接阻塞读者，更高效。
#### 2). 实时性要求高
RCU 的优雅周期可能长达毫秒级（如等待CPU上下文切换），而读写锁的写操作可立即生效，适合实时系统。
#### 3). 数据结构不可复制
若数据结构庞大或难以复制（如链表、树结构），RCU的写操作会带来显著开销，此时读写锁更合适。
#### 4). 细粒度锁竞争
读写锁可用于保护更小的临界区，而RCU通常用于保护较大的数据结构。
#### 5). 简单性与维护成本
RCU的实现复杂（需跟踪优雅周期），而读写锁逻辑简单，更易于调试和维护。

## 4. 总结
- **RCU的优势**：极致读性能，适合读多写少的大规模数据场景。
- **读写锁的优势**：写操作高效、实时性强、内存占用低，适合写较多或对延迟敏感的场景。
