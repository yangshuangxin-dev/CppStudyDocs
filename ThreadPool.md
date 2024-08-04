# 线程池

## 什么是线程池

在项目中常常使用池式结构，比如内存池、连接池、对象池、线程池。本文介绍线程池的实现。

线程池是维持和管理固定数量线程的池式结构，由于CPU的核心数量是固定的，比如8核CPU，每一个核心在一个时刻只能处理一个线程，所以线程数量的增加，由系统CPU资源的限制，不会带来性能的提升，反而会带来负担。

线程池目的就是避免线程的频繁创建和销毁，维持固定数量的线程，并且对线程进行管理：有任务的时候时执行任务，无任务时不执行，并且线程处于休眠状态。

## 线程池解决的问题

在程序的生产者消费者模型中，例如Reactor，在事件循环中，如果某一类任务特别耗时，这会严重影响生产者线程处理其他任务，这就需要把这类任务放到其他线程异步执行，通常的做法是把该任务的上下文和执行函数进行打包，推入线程池进行处理。

线程池复用了线程的资源，可以更加充分的利用系统资源，并且异步的执行耗时任务。

## 线程池的组成结构

线程池的组成通常是：固定数量的线程+队列。

固定数量的线程，一个程序的线程池应该创建多少才能最大化的利用CPU资源，并且减少线程的负担？可以从该程序处理任务的类型进行分析。在CPU密集型的程序中，线程池的处理一般为CPU的核心数；在IO密集型程序中（网络IO密集、磁盘IO密集），常常使用 2 * CPU核心数 + 2 线程。这是因为在IO密集的程序中，read/write读写操作时，程序会陷入内核态，并且从内核态进行数据的拷贝，由于是阻塞IO，所以当一个线程陷入内核态进行数据操作时，另一个线程也可以进行数据操作，所以通常分配2 * CPU数量的线程。

由于是进行多线程操作，所以队列需要加锁。依据操作临界资源（公共资源）的时长，锁按照不同的细粒度从小到大为：原子操作、自旋锁（spin lock）、互斥锁(mutex lock)。在进行队列的不同操作时，按照操作的复杂程度，需要使用不同的锁进行多线程保护。

队列的数据结构是先进先出，这也保证了按照先后到达的次序进行组织任务，保证任务调度的先后顺序。

## 线程池的解决方案

线程池管理和维持消费者线程，通过管理队列，改变队列的状态来决定消费者线程的状态：如果队列中的任务从有->无，线程进行休眠；如果任务从无->有，就唤醒消费者线程进行处理。

线程池内部应该管理任务的生命周期，在操作队列时需要加锁，所以锁放入队列中，根据队列的状态使用条件变量进行管理消费者线程。

## 线程池的实现

1. 定义线程池对外接口

    ```c++
    // 定义传入的任务处理函数
    typedef void (*handler_pt)(void * /* ctx */);
    // 线程池的创建
    void *thrdpool_create(int thrd_count);
    // 线程池的销毁
    void thrdpool_terminate(void *pool);
    // 线程池增加任务
    int thrdpool_post(void *pool, handler_pt func, void *arg);
    // 等待所有的任务结束
    void thrdpool_waitdone(void *pool);
    ```

2. 定义任务、队列和线程池

    ```c++
    typedef struct spinlock spinlock_t; // 自旋锁
    // 定义任务
    typedef struct task_s {
        void *next;	// 执行下一个任务
        handler_pt func; // 任务的函数指针
        void *arg;	// 任务的参数
    } task_t;
    // 任务队列
    typedef struct task_queue_s {
        void *head;	// 队列的头指针
        void **tail; // 队列尾指针
        int block;	// 队列是否阻塞消费者线程
        spinlock_t lock;	// 自旋锁
        pthread_mutex_t mutex;// 互斥锁
        pthread_cond_t cond;// 条件变量
    } task_queue_t;
    // 线程池
    struct thrdpool_s {
        task_queue_t *task_queue; // 任务队列
        atomic_int quit;	// 原子操作，标记是否退出
        int thrd_count;		// 线程数量
        pthread_t *threads;	// 管理的消费者线程
    };
    ```

3. 创建、设置和销毁任务队列

    ```c++
    // 创建队列
    static task_queue_t *TaskqueueCreate()
    {
        int ret;
        task_queue_t *queue = (task_queue_t *)malloc(sizeof(task_queue_t));
        if (queue)
        {
            ret = pthread_mutex_init(&queue->mutex, NULL);
            if (ret == 0)
            {
                ret = pthread_cond_init(&queue->cond, NULL);
                if (ret == 0)
                {
                    spinlock_init(&queue->lock);
                    queue->head = NULL;
                    queue->tail = &queue->head;
                    queue->block = 1;
                    return queue;
                }
                pthread_mutex_destroy(&queue->mutex);
            }
            free(queue);
        }
        return NULL;
    }
    // 设置是否阻塞
    static void Setblock(task_queue_t *queue, int block)
    {
        pthread_mutex_lock(&queue->mutex);
        queue->block = block;
        pthread_mutex_unlock(&queue->mutex);
        pthread_cond_broadcast(&queue->cond);
    }
    // 销毁线程池
    static void TaskqueueDestroy(task_queue_t *queue)
    {
        task_t *task;
        while ((task = PopTask(queue)))
        {
            free(task);
        }
        spinlock_destroy(&queue->lock);
        pthread_cond_destroy(&queue->cond);
        pthread_mutex_destroy(&queue->mutex);
        free(queue);
    }
    ```

4. 向任务队列增加和获取任务

    ```c++
    // 任务入队列
    static inline void AddTask(task_queue_t *queue, void *task)
    {
        // 不限定任务类型，只要该任务的结构起始内存是一个用于链接下一个节点的指针
        void **link = (void **)task;
        *link = NULL;
    
        spinlock_lock(&queue->lock);
        /* 等价于 queue->tail->next */
        *queue->tail = link;
        queue->tail = link;
        spinlock_unlock(&queue->lock);
        pthread_cond_signal(&queue->cond);
    }
    // 任务出队列
    static inline void *PopTask(task_queue_t *queue)
    {
        spinlock_lock(&queue->lock);
        if (queue->head == NULL)
        {
            spinlock_unlock(&queue->lock);
            return NULL;
        }
        task_t *task;
        task = queue->head;
    
        void **link = (void **)task;
        queue->head = *link;
    
        if (queue->head == NULL)
        {
            queue->tail = &queue->head;
        }
        spinlock_unlock(&queue->lock);
        return task;
    }
    // 获取一个任务
    static inline void *GetTask(task_queue_t *queue)
    {
        task_t *task;
        // 使用while防止虚假唤醒
        while ((task = PopTask(queue)) == NULL)
        {
            pthread_mutex_lock(&queue->mutex);
            // 是否阻塞
            if (queue->block == 0)
            {
                pthread_mutex_unlock(&queue->mutex);
                return NULL;
            }
            pthread_cond_wait(&queue->cond, &queue->mutex);
            // pthread_cond_wait的操作
            // 1. 先 unlock(&mtx)
            // 2. 在 cond 休眠，阻塞在这
            // 3. AddTask 时 在 cond 唤醒
            // 4. 加上 lock(&mtx);
            pthread_mutex_unlock(&queue->mutex);
        }
        return task;
    }
    ```

5. 消费者（工作）线程处理任务

    ```c++
    static void * ThrdpoolWorker(void *arg)
    {
        thrdpool_t *pool = (thrdpool_t *)arg;
        task_t *task;
        void *ctx;
    
        while (atomic_load(&pool->quit) == 0)
        {
            task = (task_t *)GetTask(pool->task_queue);
            if (!task) {
                continue;
            }
            handler_pt func = task->func;
            ctx = task->arg;
            free(task);
            func(ctx);
        }
        return NULL;
    }
    ```

6. 线程池的消费线程创建和销毁

    ```c++
    // 创建线程
    static int ThreadsCreate(thrdpool_t *pool, size_t thrd_count)
    {
        pthread_attr_t attr;
        int ret;
        ret = pthread_attr_init(&attr);
        if (ret == 0)
        {
            pool->threads = (pthread_t *)malloc(sizeof(pthread_t) * thrd_count);
            if (pool->threads)
            {
                int i = 0;
                for (; i < thrd_count; i++)
                {
                    if (pthread_create(&pool->threads[i], &attr, ThrdpoolWorker, pool) != 0)
                    {
                        break;
                    }
                }
                pool->thrd_count = i;
                pthread_attr_destroy(&attr);
                if (i == thrd_count)
                    return 0;
                ThreadsTerminate(pool);
                free(pool->threads);
            }
            ret = -1;
        }
        return ret;
    }
    // 销毁线程
    static void ThreadsTerminate(thrdpool_t *pool)
    {
        atomic_store(&pool->quit, 1);
        Setblock(pool->task_queue, 0);
        for (int i = 0; i < pool->thrd_count; i++)
        {
            pthread_join(pool->threads[i], NULL);
        }
    }
    ```

7. 线程池的创建和销毁

    ```c++
    // 线程池的创建
    thrdpool_t *thrdpool_create(int thrd_count) {
        thrdpool_t *pool;
    
        pool = (thrdpool_t*)malloc(sizeof(*pool));
        if (pool) {
            task_queue_t *queue = TaskqueueCreate();
            if (queue) {
                pool->task_queue = queue;
                atomic_init(&pool->quit, 0);
                if (ThreadsCreate(pool, thrd_count) == 0){
                    return pool;
                }
                TaskqueueDestroy(queue);
            }
            free(pool);
        }
        return NULL;
    }
    
    void thrdpool_terminate(thrdpool_t * pool) {
        atomic_store(&pool->quit, 1);
        Setblock(pool->task_queue, 0);
    }
    ```

    

8. 线程池增加任务和等待所有任务结束

    ```c++
    // 线程池增加任务
    int thrdpool_post(thrdpool_t *pool, handler_pt func, void *arg)
    {
        if (atomic_load(&pool->quit) == 1) {
            return -1;
        }
        //创建任务
        task_t *task = (task_t *)malloc(sizeof(task_t));
        if (!task) {
             return -1;
        }
        task->func = func;
        task->arg = arg;
        AddTask(pool->task_queue, task);
        return 0;
    }
    
    // 等待任务结束
    void thrdpool_waitdone(thrdpool_t *pool)
    {
        int i;
        for (i = 0; i < pool->thrd_count; i++)
        {
            pthread_join(pool->threads[i], NULL);
        }
        TaskqueueDestroy(pool->task_queue);
        free(pool->threads);
        free(pool);
    }
    ```

## 附录

### 原子操作

```c++
#ifndef SKYNET_ATOMIC_H
#define SKYNET_ATOMIC_H

#ifdef __STDC_NO_ATOMICS__

#include <stddef.h>
#include <stdint.h>

#define ATOM_INT volatile int
#define ATOM_POINTER volatile uintptr_t
#define ATOM_SIZET volatile size_t
#define ATOM_ULONG volatile unsigned long
#define ATOM_INIT(ptr, v) (*(ptr) = v)
#define ATOM_LOAD(ptr) (*(ptr))
#define ATOM_STORE(ptr, v) (*(ptr) = v)
#define ATOM_CAS(ptr, oval, nval) __sync_bool_compare_and_swap(ptr, oval, nval)
#define ATOM_CAS_ULONG(ptr, oval, nval) __sync_bool_compare_and_swap(ptr, oval, nval)
#define ATOM_CAS_SIZET(ptr, oval, nval) __sync_bool_compare_and_swap(ptr, oval, nval)
#define ATOM_CAS_POINTER(ptr, oval, nval) __sync_bool_compare_and_swap(ptr, oval, nval)
#define ATOM_FINC(ptr) __sync_fetch_and_add(ptr, 1)
#define ATOM_FDEC(ptr) __sync_fetch_and_sub(ptr, 1)
#define ATOM_FADD(ptr, n) __sync_fetch_and_add(ptr, n)
#define ATOM_FSUB(ptr, n) __sync_fetch_and_sub(ptr, n)
#define ATOM_FAND(ptr, n) __sync_fetch_and_and(ptr, n)

#else

#if defined(__cplusplus)
#include <atomic>
#define STD_ std::
#define atomic_value_type_(p, v) decltype((p)->load())(v)
#else
#include <stdatomic.h>
#define STD_
#define atomic_value_type_(p, v) v
#endif

#define ATOM_INT STD_ atomic_int
#define ATOM_POINTER STD_ atomic_uintptr_t
#define ATOM_SIZET STD_ atomic_size_t
#define ATOM_ULONG STD_ atomic_ulong
#define ATOM_INIT(ref, v) STD_ atomic_init(ref, v)
#define ATOM_LOAD(ptr) STD_ atomic_load(ptr)
#define ATOM_STORE(ptr, v) STD_ atomic_store(ptr, v)

static inline int ATOM_CAS(STD_ atomic_int *ptr, int oval, int nval)
{
	return STD_ atomic_compare_exchange_weak(ptr, &(oval), nval);
}

static inline int ATOM_CAS_SIZET(STD_ atomic_size_t *ptr, size_t oval, size_t nval)
{
	return STD_ atomic_compare_exchange_weak(ptr, &(oval), nval);
}

static inline int ATOM_CAS_ULONG(STD_ atomic_ulong *ptr, unsigned long oval, unsigned long nval)
{
	return STD_ atomic_compare_exchange_weak(ptr, &(oval), nval);
}

static inline int ATOM_CAS_POINTER(STD_ atomic_uintptr_t *ptr, uintptr_t oval, uintptr_t nval)
{
	return STD_ atomic_compare_exchange_weak(ptr, &(oval), nval);
}

#define ATOM_FINC(ptr) STD_ atomic_fetch_add(ptr, atomic_value_type_(ptr, 1))
#define ATOM_FDEC(ptr) STD_ atomic_fetch_sub(ptr, atomic_value_type_(ptr, 1))
#define ATOM_FADD(ptr, n) STD_ atomic_fetch_add(ptr, atomic_value_type_(ptr, n))
#define ATOM_FSUB(ptr, n) STD_ atomic_fetch_sub(ptr, atomic_value_type_(ptr, n))
#define ATOM_FAND(ptr, n) STD_ atomic_fetch_and(ptr, atomic_value_type_(ptr, n))
#endif
#endif
```

### 自旋锁

```c++
#ifndef SKYNET_SPINLOCK_H
#define SKYNET_SPINLOCK_H

#define SPIN_INIT(q) spinlock_init(&(q)->lock);
#define SPIN_LOCK(q) spinlock_lock(&(q)->lock);
#define SPIN_UNLOCK(q) spinlock_unlock(&(q)->lock);
#define SPIN_DESTROY(q) spinlock_destroy(&(q)->lock);

#ifndef USE_PTHREAD_LOCK

#ifdef __STDC_NO_ATOMICS__

#define atomic_flag_ int
#define ATOMIC_FLAG_INIT_ 0
#define atomic_flag_test_and_set_(ptr) __sync_lock_test_and_set(ptr, 1)
#define atomic_flag_clear_(ptr) __sync_lock_release(ptr)

struct spinlock
{
	atomic_flag_ lock;
};

static inline void spinlock_init(struct spinlock* lock)
{
	atomic_flag_ v = ATOMIC_FLAG_INIT_;
	lock->lock = v;
}

static inline void spinlock_lock(struct spinlock* lock)
{
	while (atomic_flag_test_and_set_(&lock->lock))
	{
	}
}

static inline int spinlock_trylock(struct spinlock* lock)
{
	return atomic_flag_test_and_set_(&lock->lock) == 0;
}

static inline void spinlock_unlock(struct spinlock* lock)
{
	atomic_flag_clear_(&lock->lock);
}

static inline void spinlock_destroy(struct spinlock* lock)
{
	(void)lock;
}

#else // __STDC_NO_ATOMICS__

#include "atomic.h"

#define atomic_test_and_set_(ptr) STD_ atomic_exchange_explicit(ptr, 1, STD_ memory_order_acquire)
#define atomic_clear_(ptr) STD_ atomic_store_explicit(ptr, 0, STD_ memory_order_release);
#define atomic_load_relaxed_(ptr) STD_ atomic_load_explicit(ptr, STD_ memory_order_relaxed)

#if defined(__x86_64__)
#include <immintrin.h> // For _mm_pause
#define atomic_pause_() _mm_pause()
#else
#define atomic_pause_() ((void)0)
#endif

struct spinlock
{
	STD_ atomic_int lock;
};

static inline void spinlock_init(struct spinlock* lock)
{
	STD_ atomic_init(&lock->lock, 0);
}

static inline void spinlock_lock(struct spinlock* lock)
{
	for (;;)
	{
		if (!atomic_test_and_set_(&lock->lock))
			return;
		while (atomic_load_relaxed_(&lock->lock))
			atomic_pause_();
	}
}

static inline int spinlock_trylock(struct spinlock* lock)
{
	return !atomic_load_relaxed_(&lock->lock) &&
		!atomic_test_and_set_(&lock->lock);
}

static inline void spinlock_unlock(struct spinlock* lock)
{
	atomic_clear_(&lock->lock);
}

static inline void spinlock_destroy(struct spinlock* lock)
{
	(void)lock;
}

#endif // __STDC_NO_ATOMICS__

#else

#include <pthread.h>

// we use mutex instead of spinlock for some reason
// you can also replace to pthread_spinlock

struct spinlock
{
	pthread_mutex_t lock;
};

static inline void spinlock_init(struct spinlock* lock)
{
	pthread_mutex_init(&lock->lock, NULL);
}

static inline void spinlock_lock(struct spinlock* lock)
{
	pthread_mutex_lock(&lock->lock);
}

static inline int spinlock_trylock(struct spinlock* lock)
{
	return pthread_mutex_trylock(&lock->lock) == 0;
}

static inline void spinlock_unlock(struct spinlock* lock)
{
	pthread_mutex_unlock(&lock->lock);
}

static inline void spinlock_destroy(struct spinlock* lock)
{
	pthread_mutex_destroy(&lock->lock);
}

#endif
#endif

```

