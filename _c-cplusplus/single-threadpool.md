---
title: "一种简单的C++线程池实现方案"
collection: c-cplusplus
permalink: /c-cplusplus/single-threadpool
excerpt: ' '
date: 2024-06-19
citation: 'Joe-Bi. (2024). &quot;single threadpool.&quot; <i>GitHub Joe-Bi of blog</i>'
---
   
线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。线程池线程都是后台线程。每个线程都使用默认的堆栈大小，以默认的优先级运行，并处于多线程单元中。如果某个线程在托管代码中空闲（如正在等待某个事件），则线程池会插入另一个辅助线程来使所有处理器保持繁忙。如果所有线程池线程都始终保持繁忙，但队列中包含挂起的工作，则线程池会在一段时间后创建另一个辅助线程但线程的数目永远不会超过最大值。超过最大值的线程可以排队，但他们要等到其他线程完成后才启动。

线程池的主要优点包括：

资源复用：通过复用已存在的线程而不是创建新的线程，可以明显减少线程创建和销毁的开销。
提高响应速度：当任务到达时，任务可以不需要等到线程创建就能立即执行。
提高线程的可管理性：线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。


## c++实现代码

```
#ifndef __fixed_thread_pool_h__
#define __fixed_thread_pool_h__

#include <mutex>
#include <condition_variable>
#include <functional>
#include <queue>
#include <thread>
#include <unordered_map>
#include <iostream>

class fixed_thread_pool {
public:
	explicit fixed_thread_pool(size_t thread_count)
		: data_(std::make_shared<data>()) {
		for (size_t i = 0; i < thread_count; ++i) {
			std::thread([data = data_] {
				std::unique_lock<std::mutex> lk(data->mtx_);
				for (;;) {
					if (!data->tasks_.empty()) {
						auto func = data->tasks_.front();
						auto iter = std::find_if(data->funParams.begin(), data->funParams.end(),
							[func](const auto& item) {
								void (* const* ptrf)(void*) = func.target<void(*)(void*)>();
								void (* const* ptrf1)(void*) = item.second.target<void(*)(void*)>();
								if (ptrf && ptrf1 && *ptrf == *ptrf1)
									return true;
								return false;
							}
						);
						if (iter != data->funParams.end()) {
							auto currentfunc = std::move(func);
							auto param = iter->first;
							data->funParams.erase(iter);
							data->tasks_.pop();
							lk.unlock();
							currentfunc(param);
							lk.lock();
						}
					}
					else if (data->is_shutdown_) {
						std::cout << std::flush << "thread " << std::this_thread::get_id() << " exit" << std::endl;
						break;
					}
					else {
						data->cond_.wait(lk);
					}
				}
			}).detach();
		}
	}

	fixed_thread_pool() = default;
	fixed_thread_pool(fixed_thread_pool&&) = default;

	~fixed_thread_pool() {
		if ((bool)data_) {
			{
				std::lock_guard<std::mutex> lk(data_->mtx_);
				data_->is_shutdown_ = true;
			}
			data_->cond_.notify_all();
		}
	}

	template <class F>
	void execute(F&& task, void* arg) {
		{
			std::lock_guard<std::mutex> lk(data_->mtx_);
			data_->funParams[arg] = task;
			data_->tasks_.emplace(std::forward<F>(task));
		}
		data_->cond_.notify_one();
	}

private:
	struct data {
		std::mutex mtx_;
		std::condition_variable cond_;
		bool is_shutdown_ = false;
		std::queue<std::function<void(void*)>> tasks_;
		std::unordered_map<void*, std::function<void(void*)>> funParams;
	};
	std::shared_ptr<data> data_;
};

struct param {
	param(void) = default;
	virtual ~param(void) = default;

	virtual void callbackFunc(void) {
		std::cout << std::flush << "Parameter object " << this << " running callback function" << std::endl;
	}
};

void threadFunc(void* args) {
	if (args) {
		struct param* cl = (struct param*)args;
		
		// do something

		cl->callbackFunc();
		delete cl;
	}
};

//example:
//fixed_thread_pool* threadpoolPtr = new fixed_thread_pool(10);
//threadpoolPtr->execute(std::function<void(void*)>(threadFunc), new struct param());
//delete threadpoolPtr;

#endif /*__fixed_thread_pool_h__*/

```

## 注意事项
使用线程池时，需要注意以下几点：

合理地设置线程池大小，避免过多或过少的线程导致资源浪费或任务处理不及时。
注意线程池中的任务队列，避免队列过长导致内存溢出。
合理地设置线程池的拒绝策略，当线程池无法处理新任务时，可以采取不同的策略来处理。
在使用完线程池后，及时关闭线程池，避免资源浪费。


  






