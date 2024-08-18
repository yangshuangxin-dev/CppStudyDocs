# 无锁消息队列

在多线程编程中无锁、无等待和阻塞是常常使用到的技术，并且使用的场景各有不同，首先需要区分这三个不同概念。

## 无锁、无等待和阻塞

- 无锁（lock-free）：软件系统作为一个整体无论如何都要向前移动，不保证每一个线程的前进的进度（可能出现线程饿死）。通常使用compare_exchange 原语实现，可以有循环，但是类似compare_exchange 实现的自旋锁不行。
- 无等待（wait-free）：系统向前移动，并且即使有其他线程争用、阻塞的情况下，每一个线程都会向前移动，每个操作都在有限步骤中执行。通常使用exchange、fetch_add等原语实现，并且不包含可能被其他线程影响的循环。
- 阻塞（blocking）：整个系统可能不会取得任何进展，阻塞、中断的线程可能无限的组织系统范围内的向前。

基于互斥锁，CPU执行一个线程时该线程没有获取到锁，就是阻塞的，通常也有一个线程向前推进，当时它可能是一个当前未运行的线程。

在一些不能使用基于锁的算法的情况下，可以使用无等待、无锁算法。使用无锁算法，可以保证当前运行的线程总是可以向前推进。例如：信号处理程序使用锁不安全，锁当前可以被抢占的线程获取，就会阻塞信号处理；在硬实时系统中，执行时间有严格的上限，应该使用无等待算法。

如果系统可以blocking，不需要任何特定的向前移动的保证，可以从blocking/LockFree/WaitFree中选择最快的算法；如果需要无锁向前进度保证，只能从LockFree/WaitFree中选择；如果需要无等待保证，只能选择WaitFree算法。

在实际的代码编程中，如何进行选择？首先应该先用有锁进行实现，然后进行分析，是否需要保证系统的向前移动，使用无锁进行优化，还需要考虑任务的执行实际，如果是耗时计算或耗时IO时，应该选择有锁。

## 队列的分类

在开发中，生产者-消费者是并发系统中最基本的组件之一，却没有一劳永逸完全通用的并发队列，可以按照不同的条件，分类成不同类型的队列。

按照生产者和消费者线程的数量，可以分为：

- MPMC：多生产者/多消费者队列
- SPMC：单生产者/多消费者队列
- MPSC：多生产者/单消费者队列
- SPSC：单生产者/单消费者队列

> 不同场景使用更适合的队列，才能更快。

按照底层数据结构，可以划分为

- 基于数组（Array-Based）：   基于数组的队列通常更快，但是通常不是严格无锁的。缺点是需要为最坏的情况预先分配内存。
- 基于链表（Linked-List-Based）：链表队列是动态增长的，不需要预先分配任何内存。
- 混合的（Hybrid）：固定大小的小数组的链表，结合了两者的优点。

按照链表队列是否可以侵入，可以划分为：

- 侵入式（Intrusive）：修改已经分配的链表节点数据。如果需要转换已经动态分配的数据，侵入式具有更好的性能，不需要额外的节点内存管理。
- 非侵入式（Non-intrusive）：新的数据，新分配内存，并且插入原来的队列中。

根据链表队列长度的最大的大小，可以划分：

- 有界（Bounded）: 队列长度是有限的。
- 无界（Unbounded）：队列长度是无固定的。

> 无界队列十分危险，通常需要一个有界队列，它强制执行你认为应该发生的事情（丢弃、覆盖），避免无法控制的事情（内存溢出、内存泄漏）。

## 有锁队列和无锁队列

无锁队列通常是锁和条件变量共同使用，在没有数据的时候条件变量会阻塞消费者线程，有数据时唤醒消费者线程。

无锁队列通常是支持所有的生产者消费者模型，多生产者多消费者、多生产者单消费者、单生产者多消费者、单生产者单消费者都可以。

### 普通有锁队列

- 最简单的有锁队列，适用于大多数场景
```c++
    template <class T, typename StorageType = std::deque<T> >
    class LockedQueue {
        // 互斥锁
        std::mutex queLock;
        //定义队列
        StorageType queueData;
    public:
        LockedQueue() {}
        virtual ~LockedQueue(){}
        // 增加元素到队列
        void Add(const T& item) {
            std::lock_guard<std::mutex> lock(queLock);
            queueData.push_back(item);
        }
    
        // 从队列中获取元素
        bool Next(T& result) {
            std::lock_guard<std::mutex> lock(queLock);
            if (queueData.empty()) {
                return false;
            }
            result = queueData.front();
            queueData.pop_front();
            return true;
        }
        // 按照Checker的条件出队列
        template<class Checker>
        bool Next(T& result, Checker& check) {
            std::lock_guard<std::mutex> lock(queLock);
            if (queueData.empty()) {
                return false;
            }
            result = queueData.front();
            if (!check.Process(result)) {
                return false;
            }
            queueData.pop_front();
            return true;
        }
    
        // 获取队列元素但是不出队列
        T& Peek() {
            std::lock_guard<std::mutex> lock(queLock);
            T& result = queueData.front();
            return result;
        }
    
        // 锁住队列禁止读写数据
        void Lock() {
            this->queLock.lock();
        }
    
        // 释放锁住的队列
        void Unlock() {
            this->queLock.unlock();
        }
    
        // 删除一个队列元素
        void popFront() {
            std::lock_guard<std::mutex> lock(queLock);
            queueData.pop_front();
        }
    
        // 检测队列是否为空
         bool empty() {
            std::lock_guard<std::mutex> lock(queLock);
            return queueData.empty();
        }
    };
```

- 在线程池、多生产者和多消费者的场景中，在没有数据时常常需要阻塞消费者线程，直到生产者有数据后唤醒消费者线程来进行数据处理。
```c++
    #include <condition_variable>
    #include <mutex>
    #include <queue>
    #include <atomic>
    #include <type_traits>
    template <typename T>
    class ProducerConsumerQueue {
    private:
        // 队列
        std::queue<T> queueData;
        // 锁和条件变量
        std::mutex queueLock;
        std::condition_variable queueCV;
        // 销毁队列标识
        std::atomic<bool> shutdown;
    public:
        ProducerConsumerQueue<T>() : shutdown(false) { }
        // 数据入队列
        void Push(const T& value)
        {
            std::lock_guard<std::mutex> lock(queueLock);
            queueData.push(std::move(value));
            // 通知消费者线程
            queueCV.notify_one();
        }
        // 检测是队列是否为空
        bool Empty()
        {
            std::lock_guard<std::mutex> lock(queueLock);
            return queueData.empty();
        }
        // 获取队列的长度
        size_t Size() const
        {
            return queueData.size();
        }
        // 无阻塞出队列
        bool Pop(T& value)
        {
            std::lock_guard<std::mutex> lock(queueLock);
            if (queueData.empty() || shutdown) {
                return false;
            }
            value = queueData.front();
            queueData.pop();
            return true;
        }
        // 条件变量阻塞消费者线程，直到有数据才返回
        void WaitAndPop(T& value) {
            std::unique_lock<std::mutex> lock(queueLock);
            queueCV.wait(lock, [&]()-> bool { return  !(queueData.empty() && !shutdown)});
            if (queueData.empty() || shutdown) {
                return;
            }
            value = queueData.front();
            queueData.pop();
        }
    
        void Cancel() {
            std::unique_lock<std::mutex> lock(queueLock);
            while (!queueData.empty()) {
                T& value = queueData.front();
                // 如果队列中的元素是new的对象指针，需要释放内存
                DeleteQueuedObject(value);
                queueData.pop();
            }
            shutdown = true;
            // 通知所有的消费者线程
            queueCV.notify_all();
        }
    
    private:
        // 条件编译，指针类型释放内存函数
        template<typename E = T>
        typename std::enable_if<std::is_pointer<E>::value>::type DeleteQueuedObject(E& obj) { delete obj; }
        // 非指针类型不做处理
        template<typename E = T>
        typename std::enable_if<!std::is_pointer<E>::value>::type DeleteQueuedObject(E const& /*packet*/) { }
    };
```

### 性能优化有锁队列

上面的两个队列在多生产者多消费者的应用场景下，由于使用同一个锁，虽然锁读写是原子操作，但是随着线程的增加，其效率会急剧降低。原因在于锁的碰撞非常频繁，具体为生产者和生产者时刻发生碰撞，生产者和消费者发生碰撞，消费者和消费者也会发生碰撞，就需要进行性能优化。生产者和消费者进行分离，减少由于线程之间锁的碰撞造成的性能损失，从而提升效率。

```c++
// 定义消息队列
struct Msgqueue {
	size_t msgMax;  // 队列的最大长度
	size_t msgCnt;	// 当前队列的长度
	int linkoff;	// 队列节点的指针偏移量
	int nonblock;	// 设置是否阻塞
	void *head1;	// 两个头节点指针，分离生产者和消费者
	void *head2;
	void **getHead; // 消费者链表头指针
	void **putHead;	// 生产者队列的头指针和尾指针
	void **putTail;
	pthread_mutex_t getMutex; // 消费者锁和条件变量
	pthread_cond_t getCond;
	pthread_mutex_t putMutex; // 生产者锁和条件变量
	pthread_cond_t putCond;
};
typedef struct Msgqueue msgqueue_t;
// 设置非阻塞模式
void MsgqueueSetNonblock(msgqueue_t *queue) {
	queue->nonblock = 1;
	pthread_mutex_lock(&queue->putMutex);
	pthread_cond_signal(&queue->getCond);
	pthread_cond_broadcast(&queue->putCond);
	pthread_mutex_unlock(&queue->putMutex);
}
// 设置阻塞模式
void MsgqueueSetBlock(msgqueue_t *queue) {
	queue->nonblock = 0;
}
// 消费者链表和生产者队列交换，会发生消费者锁和生产者锁碰撞
static size_t MsgqueueSwap(msgqueue_t *queue) {
	void **getHead = queue->getHead;
	size_t cnt;
	// 读链表指针指向写队列的头指针
	queue->getHead = queue->putHead;
	pthread_mutex_lock(&queue->putMutex);
	// 生产者队列也是空，如果是阻塞模式，对消费者线程进行阻塞
	while (queue->msgCnt == 0 && !queue->nonblock) {
		pthread_cond_wait(&queue->getCond, &queue->putMutex);
	}
	cnt = queue->msgCnt;
	// 唤醒达到最大队列长度的生产者线程
	if (cnt > queue->msgMax - 1) {
		pthread_cond_broadcast(&queue->putCond);
	}
	// 生产者队列的头尾指针指向原来的消费者链表
	queue->putHead = getHead;
	queue->putTail = getHead;
	queue->msgCnt = 0;
	pthread_mutex_unlock(&queue->putMutex);
	return cnt;
}
// 生产者写数据到队列中
void MsgqueuePut(void *msg, msgqueue_t *queue) {
	// link 为链表下一个节点的指针
	void **link = (void **)((char *)msg + queue->linkoff);
	*link = NULL;
	pthread_mutex_lock(&queue->putMutex);
	// 队列到达最大值，如果是阻塞模式，进行阻塞，直到消费者取走数据
	while (queue->msgCnt > queue->msgMax - 1 && !queue->nonblock) {
		pthread_cond_wait(&queue->putCond, &queue->putMutex);
	}
	*queue->putTail = link;
	queue->putTail = link;
	queue->msgCnt++;
	pthread_mutex_unlock(&queue->putMutex);
	// 条件变量通知消费者线程
	pthread_cond_signal(&queue->getCond);
}
// 消费者取数据
void *MsgqueueGet(msgqueue_t *queue) {
	void *msg;
	pthread_mutex_lock(&queue->getMutex);
	// 链表有数据取走数据，无数据进行与生产者队列交换
	if (*queue->getHead || MsgqueueSwap(queue) > 0) {
		msg = (char *)*queue->getHead - queue->linkoff;
		*queue->getHead = *(void **)*queue->getHead;
	}
	else {
		msg = NULL;
	}
	pthread_mutex_unlock(&queue->getMutex);
	return msg;
}

msgqueue_t *msgqueue_create(size_t maxlen, int linkoff) {
	// 分配队列内存
	msgqueue_t *queue = (msgqueue_t *)malloc(sizeof (msgqueue_t));
	int ret;
	if (!queue) {
		return NULL;
	}
	// 初始化锁和条件变量
	ret = pthread_mutex_init(&queue->getMutex, NULL);
	if (ret == 0) {
		ret = pthread_mutex_init(&queue->putMutex, NULL);
		if (ret == 0) {
			ret = pthread_cond_init(&queue->getCond, NULL);
			if (ret == 0) {
				ret = pthread_cond_init(&queue->putCond, NULL);
				if (ret == 0)
				{
					queue->msgMax = maxlen;
					queue->linkoff = linkoff;
					queue->head1 = NULL;
					queue->head2 = NULL;
					// 消费者链表头指针
					queue->getHead = &queue->head1;
					// 生产者队列头、尾指针
					queue->putHead = &queue->head2;
					queue->putTail = &queue->head2;
					queue->msgCnt = 0;
					queue->nonblock = 0;
					return queue;
				}
				pthread_cond_destroy(&queue->getCond);
			}
			pthread_mutex_destroy(&queue->putMutex);
		}
		pthread_mutex_destroy(&queue->getMutex);
	}
	errno = ret;
	free(queue);
	return NULL;
}
// 销毁所有的数据
void msgqueue_destroy(msgqueue_t *queue) {
	pthread_cond_destroy(&queue->putCond);
	pthread_cond_destroy(&queue->getCond);
	pthread_mutex_destroy(&queue->putMutex);
	pthread_mutex_destroy(&queue->getMutex);
	free(queue);
}
```



### 无锁消息队列(wait-free模式)

无锁队列最重要的方式就是使用原子变量。原子变量的操作是原子的，但是内存序有多种方式，不同的内存序会影响原子变量的**可见性**（同步性）和**顺序性**。

**可见性**就是另一个线程能不能看到最新的数据，当前线程发布的数据什么时候可以被被人看到；

**顺序性**由于CPU和编译器会对不相关的代码优化，前后代码的执行顺序不一定就是写的代码的顺序，不相关的代码进行**错序执行**。在C++11中的6种内存序种，只有relax模式不保证可见性，也不保证顺序性，其他模式都在保证可见性的条件下，有不同的内存顺序优化方案。

下面为无锁消息队列的实现，只支持**多生产者单消费者**。

#### 非侵入式队列

```c++
template<typename T>
class MPSCQueueNonIntrusive {
public:
    MPSCQueueNonIntrusive() : headPtr(new Node()), tailPtr(headPtr.load(std::memory_order_relaxed)) {
        // 队列的第一个头节点
        Node* front = headPtr.load(std::memory_order_relaxed);
        front->Next.store(nullptr, std::memory_order_relaxed);
    }

    ~MPSCQueueNonIntrusive()
    {
        T* output;
        while (Dequeue(output)) {
            delete output;
        }
        Node* front = headPtr.load(std::memory_order_relaxed);
        delete front;
    }

    // wait-free 的入队列，多线程安全
    void Enqueue(T* input)
    {
        Node* node = new Node(input);
        Node* prevHead = headPtr.exchange(node, std::memory_order_acq_rel);
        prevHead->Next.store(node, std::memory_order_release);
    }
    // 只能单线程，多线程可能取到同一个数据
    bool Dequeue(T*& result) {
        Node* tail = tailPtr.load(std::memory_order_relaxed);
        Node* next = tail->Next.load(std::memory_order_acquire); // 依赖传递，下面的代码不会优化到上面执行
        if (!next) {
            return false;
        }
        result = next->Data;
        tailPtr.store(next, std::memory_order_release); // 上面的代码不会优化到下面
        delete tail;
        return true;
    }

private:
    // 链表的节点 
    struct Node {
        Node() = default;
        explicit Node(T* data) : Data(data) {
            Next.store(nullptr, std::memory_order_relaxed);
        }

        T* Data;
        std::atomic<Node*> Next;
    };
    // 队列的头尾指针
    std::atomic<Node*> headPtr;
    std::atomic<Node*> tailPtr;
    // 禁止拷贝函数
    MPSCQueueNonIntrusive(MPSCQueueNonIntrusive const&) = delete;
    MPSCQueueNonIntrusive& operator=(MPSCQueueNonIntrusive const&) = delete;
};
```

#### 侵入式队列

```c++
template<typename T, std::atomic<T*> T::* IntrusiveLink>
class MPSCQueueIntrusive
{
public:
    MPSCQueueIntrusive() : dummyPtr(reinterpret_cast<T*>(std::addressof(dummy))), headPtr(dummyPtr), tailPtr(dummyPtr) {
        std::atomic<T*>* dummyNext = new (&(dummyPtr->*IntrusiveLink)) std::atomic<T*>();
        dummyNext->store(nullptr, std::memory_order_relaxed);
    }

    ~MPSCQueueIntrusive() {
        T* output;
        while (Dequeue(output)) {
            delete output;
        }
    }

    void Enqueue(T* input) {
        (input->*IntrusiveLink).store(nullptr, std::memory_order_release);
        T* prevHead = headPtr.exchange(input, std::memory_order_acq_rel);
        (prevHead->*IntrusiveLink).store(input, std::memory_order_release);
    }

    bool Dequeue(T*& result) {
        T* tail = tailPtr.load(std::memory_order_relaxed);
        T* next = (tail->*IntrusiveLink).load(std::memory_order_acquire);
        if (tail == dummyPtr) {
            if (!next) {
                return false;
            }
            tailPtr.store(next, std::memory_order_release);
            tail = next;
            next = (next->*IntrusiveLink).load(std::memory_order_acquire);
        }

        if (next) {
            tailPtr.store(next, std::memory_order_release);
            result = tail;
            return true;
        }

        T* head = headPtr.load(std::memory_order_acquire);
        if (tail != head) {
            return false;
        }
        Enqueue(dummyPtr);
        next = (tail->*IntrusiveLink).load(std::memory_order_acquire);
        if (next) {
            tailPtr.store(next, std::memory_order_release);
            result = tail;
            return true;
        }
        return false;
    }

private:
    std::aligned_storage_t<sizeof(T), alignof(T)> dummy;
    T* dummyPtr;
    std::atomic<T*> headPtr;
    std::atomic<T*> tailPtr;

    MPSCQueueIntrusive(MPSCQueueIntrusive const&) = delete;
    MPSCQueueIntrusive& operator=(MPSCQueueIntrusive const&) = delete;
};
```

两个消息队列进行归一，使用条件编译，编译器自动选择队列类型

```c++
template<typename T, std::atomic<T*> T::* IntrusiveLink = nullptr>
using MPSCQueue = std::conditional_t<IntrusiveLink != nullptr, MPSCQueueIntrusive<T, IntrusiveLink>, MPSCQueueNonIntrusive<T>>;
```

