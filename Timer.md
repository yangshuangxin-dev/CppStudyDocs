# 定时器

在软件开发中，定时器也是最常用的组件之一。定时器是组织大量定时任务的模块，它可以用作心跳检测、游戏技能冷却、倒计时等其他需要延迟处理的功能。

> 内核中的链接`keeplive` 仅仅为内核态发送数据包，只能检测网络是否畅通；心跳包是应用层发送数据，不仅仅可以检测网络，还能检测客户端是否出错宕机，是否能正常处理数据。

## 定时器的组成

定时器通常是由**有序数据结构**和**定时驱动方式**组成。有序的数据结构是为了组织大量的任务，驱动方式为了保证任务定时执行。

### 有序数据结构

对定时任务的组织本质就是要对定时任务进行优先级处理，于是有两种数据结构。

按触发事件进行循序组织。定时器可以按照**先后过期**的任务进行排序，有序的数据可以选择有红黑树、跳表等，相对有序的数据结构有最小堆。数据结构需要快速的查找到最近需要触发的定时任务。

按照**执行顺序**进行组织，就需要使用时间轮，时间轮就是时间指针定时移动。

> 使用红黑树的定时器：nginx、workflow。
>
> 使用最小堆的定时器：最小二叉堆--libevent、最小四叉堆--libev、libuv、go。
>
> 使用时间轮的定时器：skynet、netty、kafka。

### 驱动方式

通常的驱动方式是当前时间和数据结构的最小值进行比较，然后使用不同的方式进行阻塞或休眠，直到预定的时间唤醒。对于服务端来说，驱动服务端的业务逻辑的事件通常有网络事件、定时事件、信号事件，网络事件常常也和定时事件进行协同处理，于是驱动方式通常有一下几种：

1. Reactor模型。在IO多路复用和非阻塞IO中，使用的多路复用函数select、poll、epoll的最后一个参数都是timeout超时时间检测，可以利用这三个函数进行触发检测定时器。
2. timefd。把定时任务转换为一个IO事件，当任务到期时，触发一个IO事件，由内核通知应用层，作为IO多路复用当中的事件进行处理。
3. 游戏中的帧更新。游戏开发通常由一个update函数进行每一帧数据的更新，在固定的帧数后触发定时任务。
4. usleep。使用定时休眠，进行模拟时间指针的移动，从而实现时间轮。

## 定时器的数据结构

定时器使用的数据结构通常需要满足O(logN)的查找复杂度，每次查找数据可以丢弃一半的数据。

- 红黑树。红黑树是常用的定时器数据结构，是一颗有序的二叉树，并且它解决了二叉排序树最坏退化到O（N）的复杂度问题。节点按照红黑树的规则进行染色，按照中序遍历就是一个有序的数组，红黑树的最左边的节点就是最小节点。红黑树是平衡的二叉搜索树，时间复杂度O(logN)，每一次比较都能排除一半的节点。红黑树的增删改查都需要进行左旋、右旋、染色等操作，用来维持黑节点的高度一致。

> 在C++的STL中，map、set、mutimap、mutiset都是使用红黑树实现的，所以使用时如果是自定义的结构体，就需要自己实现排序函数。

- 最小堆。最小堆通常是一个完全二叉树。满二叉树是所有的层节点都是盖层所能容纳节点的最大数量（2^n）,而**完全二叉树**是如果二叉树的深度为h，则去除h层的节点，就是一个满二叉树，并且h层都集中在最左侧排列。最小堆的父节点总是小于它的子节点，堆中的任意一个节点的子树也是最小堆。最小堆只是相对有序，保证根节点是最小节点，同时只约束父子节点的大小关系，所以比红黑树更简单，最小堆父节点为k，则左右子节点就是2k+1和2k+2，所以最小堆可以使用数组实现。

> 最小堆使用数组实现，增加节点增加到数组的末尾，然后和父节点进行比较，考虑是否能**上升操作**，进行父子节点交换。
>
> 删除最小堆节点先查找是否包含这个节点，确认存在后，先和最后一个节点进行交换，然后考虑是否需要**上升**和**下降**操作，最后删除最后一个节点。
>
> 在C++的STL中优先级队列priority_queue就是使用堆实现。

## 定时器的实现

定时器是由有序数据结构和驱动方式组成，在实现定时器时，可以使用C++里的STL实现的红黑树map、set、multimap、multiset，或者使用最小堆优先级队列priority_queue。插入任务的key可以使用添加任务时间+延迟时间。由于很可能多个任务计算出来的时间key重复，相同的key插入应该后应该的放到红黑树的右边，所以还需要一个任务id进行标识相同key的任务执行顺序。

通常在单线程的环境中，定时器使用红黑树、最小堆；如果是多线程，则使用时间轮，或者每一个定时器线程都维护一个自己的红黑树或最小堆。比如在skynet中，多线程提交定时任务到时间轮的线程中，时间轮线程仅仅监测是否到达定时的时间，如果任务需要执行，会把任务抛给任务队列进行执行。在workflow中有4个由Reactor驱动的定时器线程，每一个定时器的线程中数据结构使用红黑树管理任务。

### Reactor模型的定时器

```c++
#include <sys/epoll.h>
#include <functional>
#include <chrono>
#include <set>
#include <memory>
#include <iostream>

using namespace std;

// 定时器节点基类
struct TimerNodeBase {
    int64_t expire; // 超时时间
    int64_t id; // 任务ID
};

// 基类实现比较，基类的指针、引用都支持多态性
bool operator < (const TimerNodeBase& lhd, const TimerNodeBase& rhd) {
    // 先比较超时时间
    if (lhd.expire < rhd.expire) {
        return true;
    } else if (lhd.expire > rhd.expire) {
        return false;
    }
    // 相同时间比较任务ID
    return lhd.id < rhd.id;
}
// 定时器节点，增加了用户传入的闭包回调函数
struct TimerNode : public TimerNodeBase {
    using Callback = std::function<void(const TimerNode& node)>;
    Callback func;
    TimerNode(int64_t id, int64_t expire, Callback func) : func(func) {
        this->expire = expire;
        this->id = id;
    }
};



class Timer {
public:
    // 获取当前毫秒时间
    static int64_t GetTick() {
        return chrono::duration_cast<chrono::milliseconds>(chrono::steady_clock::now().time_since_epoch()).count();
    }
    // 向定时器增加任务, 返回任务节点，标识一个任务
    TimerNodeBase AddTimer(int64_t msec, TimerNode::Callback func) {
        int64_t expire = GetTick() + msec; // 获取超时时间
        if (timeouts.empty() || expire <= timeouts.crbegin()->expire) { // 插入红黑树的中间节点O（logN）
            auto pairs = timeouts.emplace(GenID(), expire, std::move(func)); // 性能优化，红黑树的末尾构造，避免拷贝
            return static_cast<TimerNodeBase>(*pairs.first);
        }
        // 比红黑树最大值还大，插入到最末尾的节点性能优化从O(logN) 优化到O(1)
        auto ele = timeouts.emplace_hint(timeouts.crbegin().base(), GenID(), expire, std::move(func));
        return static_cast<TimerNodeBase>(*ele);
    }
    // 使用基类节点删除子类，只有C++14及以上才有定价key的概念，使用父类进行比较、删除
    bool DelTimer(TimerNodeBase& node) {
        auto iter = timeouts.find(node);
        if (iter != timeouts.end()) {
            timeouts.erase(iter);
            return true;
        }
        return false;
    }
    // 定时器执行任务，已经超时的任务全部执行
    void HandleTimer(int64_t now) {
        auto iter = timeouts.begin();
        while (iter != timeouts.end() && iter->expire <= now) {
            iter->func(*iter);
            iter = timeouts.erase(iter);
        }
    }
    // 是否休眠，无定时器任务时-1，有任务时计算最小休眠时间
    int64_t TimeToSleep() {
        // 无任务
        auto iter = timeouts.begin();
        if (iter == timeouts.end()) {
            return -1;
        }
        // 有任务，最小任务超时时间减去当前时间
        int64_t diss = iter->expire - GetTick();
        return diss > 0 ? diss : 0;
    }
private:
    // 不断增加的任务ID
    static int64_t GenID() {
        return gid++;
    }
    static int64_t gid;                   // 计算任务ID
    set<TimerNode, std::less<>> timeouts; // 使用红黑树组织定时任务
};
int64_t Timer::gid = 0;


int main() {
    // 创建epoll和定时器
    int epfd = epoll_create(1);
    unique_ptr<Timer> timer = make_unique<Timer>();
    // 定时器增加任务
    int i = 0;
    timer->AddTimer(1000, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });

    timer->AddTimer(1000, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });

    timer->AddTimer(3000, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });
    // 定时器删除任务
    auto node = timer->AddTimer(2100, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });

    timer->DelTimer(node);

    cout << "now time:" << Timer::GetTick() << endl;
    epoll_event ev[64] = { 0 };

    while (true) {
        // 使用epoll超时触发定时器
        int n = epoll_wait(epfd, ev, 64, timer->TimeToSleep());
        int64_t now = Timer::GetTick();
        for (int i = 0; i < n; i++) {
            /*可以执行网络IO等操作*/
        }
        /* 处理定时事件*/
        timer->HandleTimer(now);
    }
    return 0;
}
```

### `Timerfd`的定时器

```c++
#include <sys/epoll.h>
#include <sys/timerfd.h>
#include <time.h> // for timespec itimerspec
#include <unistd.h> // for close

#include <functional>
#include <chrono>
#include <set>
#include <memory>
#include <iostream>

using namespace std;

struct TimerNodeBase {
    int64_t expire;
    uint64_t id;
};

struct TimerNode : public TimerNodeBase {
    using Callback = std::function<void(const TimerNode& node)>;
    Callback func;
    TimerNode(int64_t id, int64_t expire, Callback func) : func(func) {
        this->expire = expire;
        this->id = id;
    }
};

bool operator < (const TimerNodeBase& lhd, const TimerNodeBase& rhd) {
    if (lhd.expire < rhd.expire) {
        return true;
    } else if (lhd.expire > rhd.expire) {
        return false;
    }
    return lhd.id < rhd.id;
}

class Timer {
public:
    static inline int64_t GetTick() {
        return chrono::duration_cast<chrono::milliseconds>(chrono::steady_clock::now().time_since_epoch()).count();
    }

    TimerNodeBase AddTimer(int msec, TimerNode::Callback func) {
        time_t expire = GetTick() + msec;
        if (timeouts.empty() || expire <= timeouts.crbegin()->expire) {
            auto pairs = timeouts.emplace(GenID(), expire, std::move(func));
            return static_cast<TimerNodeBase>(*pairs.first);
        }
        auto ele = timeouts.emplace_hint(timeouts.crbegin().base(), GenID(), expire, std::move(func));
        return static_cast<TimerNodeBase>(*ele);
    }

    void DelTimer(TimerNodeBase& node) {
        auto iter = timeouts.find(node);
        if (iter != timeouts.end())
            timeouts.erase(iter);
    }

    void HandleTimer(int64_t now) {
        auto iter = timeouts.begin();
        while (iter != timeouts.end() && iter->expire <= now) {
            iter->func(*iter);
            iter = timeouts.erase(iter);
        }
    }

public:
    // 设置timefd的超时时间，内核自动触发读事件
    virtual void UpdateTimerfd(const int fd) {
        struct timespec abstime;
        auto iter = timeouts.begin();
        if (iter != timeouts.end()) {
            abstime.tv_sec = iter->expire / 1000;
            abstime.tv_nsec = (iter->expire % 1000) * 1000000;
        } else {
            abstime.tv_sec = 0;
            abstime.tv_nsec = 0;
        }
        struct itimerspec its = {
            .it_interval = {},
            .it_value = abstime
        };
        timerfd_settime(fd, TFD_TIMER_ABSTIME, &its, nullptr);
    }

private:
    static inline uint64_t GenID() {
        return gid++;
    }
    static uint64_t gid;

    set<TimerNode, std::less<>> timeouts;
};
uint64_t Timer::gid = 0;

int main() {
    int epfd = epoll_create(1);
    // 创建timefd,增加到epoll的监控读写事件
    int timerfd = timerfd_create(CLOCK_MONOTONIC, 0);
    struct epoll_event ev = { .events = EPOLLIN | EPOLLET };
    epoll_ctl(epfd, EPOLL_CTL_ADD, timerfd, &ev);
    // 创建定时器
    unique_ptr<Timer> timer = make_unique<Timer>();
    int i = 0;
    timer->AddTimer(1000, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });

    timer->AddTimer(1000, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });

    timer->AddTimer(3000, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });

    auto node = timer->AddTimer(2100, [&](const TimerNode& node) {
        cout << Timer::GetTick() << " node id:" << node.id << " revoked times:" << ++i << endl;
        });
    timer->DelTimer(node);

    cout << "now time:" << Timer::GetTick() << endl;

    struct epoll_event evs[64] = { 0 };
    while (true) {
        // 更新timefd的触发时间
        timer->UpdateTimerfd(timerfd);
        int n = epoll_wait(epfd, evs, 64, -1);
        int64_t now = Timer::GetTick();
        for (int i = 0; i < n; i++) {
            // 处理网络读写IO事件
        }
        // 触发定时器
        timer->HandleTimer(now);
    }
    epoll_ctl(epfd, EPOLL_CTL_DEL, timerfd, &ev);
    close(timerfd);
    close(epfd);
    return 0;
}
```

