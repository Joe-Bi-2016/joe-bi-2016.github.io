---
title: "C++实现无锁队列"
collection: c-cplusplus
permalink: /c-cplusplus/C++-Lock-Free-Queue
excerpt: ' '
date: 2024-06-19
citation: 'Joe-Bi. (2024). &quot;C++ Lock-Free Queue.&quot; <i>GitHub Joe-Bi of blog</i>'
---
   
无锁队列（Lock-Free Queue）的原理是基于原子操作和内存模型，旨在解决高并发环境下的数据访问问题。其核心思想是通过复杂的原子操作来确保多线程环境下的数据一致性和正确性，从而避免使用传统的锁机制带来的性能开销和潜在的死锁问题。以下是关于无锁队列原理的详细解释：

## 一、设计目标
无锁队列的设计目标是在高并发场景下提供高性能的入队和出队操作。通过避免使用锁机制，无锁队列能够减少线程间的竞争和等待时间，从而提高程序的性能和可伸缩性。

## 二、实现原理
无锁队列的实现原理主要依赖于原子操作（如CAS，即Compare-and-Swap）来确保多线程环境下对队列的并发访问是线程安全的。以下是无锁队列实现的一些关键点：

1. 原子操作：无锁队列使用原子操作来更新队列的状态，如入队、出队等。原子操作是不可中断的操作，即在执行过程中不会被其他线程打断，从而保证了对共享数据的正确访问。
2. CAS操作：CAS是一种常用的原子操作，它包含三个参数：一个内存位置（V）、一个预期的原值（A）和一个新值（B）。当且仅当该位置的值与预期原值相同时，才将该位置的值更新为新值。CAS操作具有原子性，即在整个操作完成之前，不会被其他线程打断。
3. 队列结构：无锁队列通常使用链表或数组等数据结构来存储元素。在链表实现中，每个节点包含数据和指向下一个节点的指针；在数组实现中，则通过维护头部和尾部索引来实现队列的入队和出队操作。
4. 入队操作：当一个线程需要向队列插入数据时，它首先会尝试使用CAS操作将数据插入到队尾。如果CAS操作成功，则入队操作完成；否则，线程需要重新尝试或进行其他操作。
5. 出队操作：当一个线程需要从队列中取出数据时，它首先会尝试使用CAS操作将队列的头部指针更新为当前头部的下一个节点。如果CAS操作成功，则线程可以安全地读取原头部节点的数据，并完成出队操作；否则，线程需要重新尝试或进行其他操作。

## 三、性能优势
由于无锁队列避免了使用锁机制带来的性能开销和潜在的死锁问题，因此在高并发环境下具有更高的性能和更好的可伸缩性。具体来说，无锁队列的优势主要体现在以下几个方面：

1. 减少线程竞争：由于无锁队列不使用锁机制，因此可以避免线程间的竞争和等待时间，从而提高程序的并发性能。
2. 降低死锁风险：锁机制可能导致死锁问题，而无锁队列通过原子操作来确保线程安全，从而降低了死锁的风险。
3. 提高可扩展性：无锁队列的设计使得程序能够更好地利用多核处理器资源，提高程序的可扩展性。

## 四、概念性描述

无锁队列通常使用两个主要的组件：

节点（Node）：存储数据的结构，通常包含数据和指向下一个节点的指针。
队列头部（Head）和尾部（Tail）：两个原子指针，用于跟踪队列的开始和结束。

入队（Enqueue）操作
分配一个新的节点，并将数据写入节点。
使用原子操作尝试将新节点的地址设置为当前尾部的下一个节点。
如果成功，尝试将尾部指针更新为新节点。
注意：由于可能有多个线程同时尝试入队，因此可能需要一个循环来确保操作成功。

出队（Dequeue）操作
读取头部指针。
使用原子操作尝试将头部指针更新为当前头部的下一个节点。
如果成功，返回旧头部节点的数据。
同样，由于并发，出队操作也可能需要在一个循环中重试。

## 五、注意事项
虽然无锁队列具有很多优势，但在实际使用中还需要注意以下事项：

1. ABA问题：当节点被释放并重新分配时，可能会出现ABA问题。因此，在实现无锁队列时需要采取相应的策略来避免ABA问题。
2. 内存序：原子操作通常具有不同的内存序选项。正确选择内存序对于确保无锁队列的正确性至关重要。
3. 平台依赖：不同的硬件和编译器对原子操作的支持可能有所不同。因此，在实现无锁队列时需要考虑平台依赖性问题。  

综上所述，无锁队列是一种基于原子操作和内存模型的高性能并发数据结构，通过避免使用锁机制来提供高性能的入队和出队操作。在设计和实现无锁队列时需要注意上述关键点和注意事项，以确保程序的正确性和性能。

c++11开始支持原子锁atomic模板类型了，通过CAS函数atomic_compare_exchange_weak配合atomic可以实现无需锁mutex等同步操作的无锁队列。  

## 六、链表式无锁队列的c++实现

```
#ifndef __QueueCAS_h__
#define __QueueCAS_h__
#include <atomic>
#include <iostream>

template<typename ElemType>
class QueueCAS {
public:
    QueueCAS(void);
    ~QueueCAS(void);

    void enqueue(ElemType elem);
    ElemType dequeue(void);
    void dump(void);

private:
    typedef struct qNode {
        qNode(void) : next(nullptr) { }
        qNode(ElemType elem) : elem(elem), next(nullptr) { }
        ElemType       elem;
        std::atomic<struct qNode*> next;
    }Node;

private:
     std::atomic<Node*> head;
	 std::atomic<Node*> tail;
};

template<typename ElemType>
QueueCAS<ElemType>::QueueCAS(void) {
    head = tail = new Node();
}

template<typename ElemType>
QueueCAS<ElemType>::~QueueCAS(void) {
    while (head != nullptr)
    {
        Node* tempNode = head;
        head = head->next;
        delete tempNode;
    }
}

template<typename ElemType>
void QueueCAS<ElemType>::enqueue(ElemType elem) {
    Node* newNode = new Node(elem);
	Node* oldtail;
	Node* next;

	do {
		oldtail = tail.load();
		next = oldtail->next.load();
		if (oldtail != tail.load())
			continue;
		
		if (next != nullptr) {
			atomic_compare_exchange_weak(&tail, (std::atomic<Node*>*) (&oldtail), next);
			continue;
		}
	} while (atomic_compare_exchange_weak(&(oldtail->next), (std::atomic<Node*>*) (&next), newNode) != true);
	
	atomic_compare_exchange_weak((std::atomic<Node*>*)(&tail), (Node**)(&oldtail), newNode);
}

template<typename ElemType>
ElemType QueueCAS<ElemType>::dequeue(void) {
    Node* oldHead;
	Node* next;
	do {
		oldHead = head.load();
		Node* oldTail = tail.load();
		next = oldHead->next.load();
		if(oldHead != head.load())
			continue;
	
		if (oldHead == oldTail && next == nullptr)
			return  ElemType(0);
	
	} while (atomic_compare_exchange_weak(&head, (std::atomic<Node*>*)(&oldHead), next) != true);

	ElemType val = oldHead->elem;
	oldHead->next = nullptr;
	delete oldHead;

    return val;
}

template<typename ElemType>
void QueueCAS<ElemType>::dump(void) {
    Node* tempNode = head->next;

    if (tempNode == nullptr) {
        std::cout << "Empty" << std::endl;
        return;
    }

    while (tempNode != nullptr)
    {
        std::cout << tempNode->elem << " ";
        tempNode = tempNode->next;
    }
    std::cout << std::endl;
}

#endif // __QueueCAS_h__
```

<br />

  






