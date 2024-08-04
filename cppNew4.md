# 现代C++新特性值多线程和异步

## 线程thread

​	c++11及以上原生支持多线程，创建线程的函数为`std::thread`，只需要包含`#include<thread>` 就可以使用。

​	std::thread的初始化构造函数：

```c++
// 新线程调用 fn 函数，参数为 args
template<calss Fn, class... Args>
explicit thread(Fn&& fn, Args&&... args);
// 拷贝构造函数是禁用的，不能拷贝
thread(const thread&) = delete;

thread t1;
thread t2 = t1; // 错误!

// 移动构造函数
thread(thread&& x) noexcept;

void func(int a, int b)
{
    cout << a << b << endl;
}

thread t1(func, 1, 2);
thread t2(std::move(t1)); // 线程移动
```

`std::thread` 主要成员函数：

- `get_id();` ：获取线程id，返回的类型为`std::thread::id`对象。
- `joinable();` ：判断线程是否可以加入等待。
- `join();` ：等待该线程执行完成后才返回。
- `detach();`：目标线程成为守护线程，驻后台运行。与之关联的std::thread对象失去对目标线程的关联，无法取到该线程的控制权(`joinable() == false`)。

### 线程的创建

- 创建无参数的线程

  ```c++
  void func()
  {
      cout << "hello world!" << endl;
  }
  
  thread t1(func);
  t1.join();
  ```

- 创建带参数的线程

  ```c++
  void func(int a, int b)
  {
      cout << a << b << endl;
  }
  
  thread t1(func, 1, 2);
  t1.join();
  ```

- 创建传引用的线程

  ```c++
  void func(int &a)
  {
      a += 10;
      cout << "hello world!" << endl;
  }
  int a = 10;
  thread t1(func, std::ref(a));
  t1.join();
  cout << a <<endl; // a = 20
  ```

- 传入类函数

  ```c++
  class A {
  public:  
      void func(int a, int b) {
          cout << a << b << endl;
      }
  };
  
  A *aPtr = new A();
  std::thread t1(&A::func, aPtr, 10, 20);
  t1.join();
  delete aPtr;
  
  A a;
  std::thread t2(&A::func, &a, 10, 20);
  t2.detach(); //脱离后无法join()
  ```

- 类中创建线程

  ```c++
  class A {
      std::thread t;
  public:  
      void func(int a, int b) {
          cout << a << b << endl;
      }
      
      void stratThread() 
      {
          t = thread(&A::func, this, 10, 20);
      }
      
      ~A() {
          if (t.joinable()) {
              t.join();
          }
      }
  };
  ```

  ## 线程锁（互斥量）

  ​	c++11 及以上标准库中有线程锁，函数声明在`#include<mutex>` 中。C++11提供4种锁：

  - std::mutex , 独占的锁，不能递归使用。
  - std::time_mutex，带超时的独占锁，不能递归使用。
  - std::recursive_mutex，递归锁，不带超时功能。
  - std::recursive_time_mutex，带超时的递归锁。

  > std::mutex 不允许构造拷贝，也不能move移动拷贝。

  ​	锁的用法：

  - `lock(); ` ：调用线程将锁住该互斥量，如果锁被其他线程占用，该线程会**阻塞**住。 当前线程获取后不调用unlock 释放会产生死锁。
  - `unlock();` ：解锁，释放对互斥量的所有权。
  - `try_lock();` ：尝试锁住互斥量，如果互斥量被其他线程占用，**不会被阻塞**， 当前线程返回false。当前线程获取后不调用unlock 释放会产生死锁。

  

### RAII 实现避免死锁

​	RAII 也称为"资源获取就是初始化"，是C++常用的资源管理、避免内存泄漏的方法。它保证任何情况下，使用对象时先构造对象，最后析构对象。

- lock_guard   自动加锁和释放锁

  ```c++
  std::mutex mtx;
  
  void func1() {
      std::lock_guard<std::mutex> lck(mtx);
      cout << "func1 " << endl;
  }
  
  void func2() {
      std::lock_guard<std::mutex> lck(mtx);
      cout << "func2 " << endl;
  }
  ```

- unique_lock 自动加锁和释放锁，也支持手动释放

  ```c++
  std::mutex mtx;
  
  void func1() {
      std::unique_lock<std::mutex> lck(mtx);
      cout << "func1 " << endl;
  }
  
  void func2() {
      std::unique_lock<std::mutex> lck(mtx);
      cout << "func2 " << endl;
      lck.unlock(); // 支持手动释放锁
      cout << "hello world ~" << endl;
  }
  ```

  > std::unique_lock 可以手动解锁，所以常常配合条件变量std::condition_variable 使用。

## 条件变量

​	互斥量（锁）是多线程同时访问同一个共享变量时，保证变量可以安全的访问和修改。条件变量是实现多线程直接的同步，线程可以按照预定的先后顺序进行。

​	条件变量使用流程：

1. 拥有条件变量的线程获取互斥量。
2. 检测某个条件，如果条件不满足则阻塞，满足则继续执行。
3. 某个线程满足条件后执行notify_one或notify_all唤醒一个或所有的线程。

条件变量`std::condition_variable`成员函数:

- wait(); ：线程阻塞等待直到收到通知后唤醒。
- wait_for(); ：线程阻塞等待直到收到通知或者时间。收到通知或时间超时，都会返回。
- wait_until()：可以指定时间点， 线程阻塞等待直到收到通知或者时间点超时。收到通知或时间超时，都会返回。
- notify_one(); ： 唤醒正在等待当前条件变量的线程中的一个，唤醒的线程是不确定的。
- notify_all();  ： 唤醒所有的线程。

```c++
std::deque<int> q;
std::mutex mtx;
std::condition_variable cond;
int count = 0;
void fun1() {
    while(true) {
      	std::unique_lock<std::mutex> locker(mtx);
      	q.push_front(count++);
        locker.unlock();
        cond.notify_one(); //唤醒其他线程
        sleep(1);
    }
}

void func2() {
    while(true) {
        std::unique_lock<std::mutex> locker(mtx);
        cond.wait(locker, []() { return !q.empty(); });
        auto data = q.back(); // 获取数据
        locker.unlock();
        std::cout << data << endl;
    }
}

std::thread t1(func1);
std::thread t2(func2);
t1.join();
t2.join();
```



## 原子变量

​	不用加锁，线程安全的共享变量。atomic 支持原生的类型变量。

```c++
std::atomic<int> count(0);
void setCount(int x) 
{
    count.store(x, std::memory_order_relaxed);
}

void print()
{
    int x = count.load(std::memory_order_relaxed);
    std::cout << x << endl;
}

std::thread t1(print);
std::thread t2(setCount, 10);
t1.join();
t2.join();
```

## 异步操作

- std::future ：异步指向某个任务，然后通过future特性去获取任务函数的返回结果。
- std::async ：异步执行某个任务。
- std::packaged_task：将任务和feature绑定在一起，是对任务的一种封装。
- std::promise：线程传递参数。

### std::async 异步执行

```c++
int func1(int a, int b) 
{
    sleep(10);
    return a + b;
}
do_some_thing()
{
    std::cout << "do other things" << std::endl;
}
// 异步执行，不会阻塞
std::future<int> result = std::async(func1, 10, 20); 

do_some_thing();
std::cout << result.get() << endl; // 获取结果
```

`std::async` 是否启动一个新线程，等待future时是否同步允许，可以配置参数类型`std::launch `。

- std::launch::defered ，函数延迟调用，直到future上调用get()或者wait()为止。
- std::launch::async，函数会在自己创建的线程上运行。
- std::launch::any = std::launch::defered  | std::launch::async ，不传入参数时默认参数选项为该参数。

### std::packaged_task 封装任务

```c++
int add(int a, int b, int c)
{
    sleep(10);
    return a + b;
}
do_some_thing()
{
    std::cout << "do other things" << std::endl;
}

std::packaged_task<int(int, int, int)> task(add); //封装任务
do_some_thing();
std::future<int> result = task.get_future();
task(1, 2, 3);
std::cout << "result:" << result.get() << std::endl;
```

### std::promise 异步手动设置值

```c++
int setString(std::promise<std::string>& str)
{
    str.set_value("hello world!");
}
do_some_thing()
{
    std::cout << "do other things" << std::endl;
}

std::promise<std::string> promise;
std::furture<std::string> result = promise.get_future();
std::thread t1(setString, std::ref(promise));

do_some_thing();
std::cout << result.get() << std::endl;
t1.join();
```

## std::function保存函数

- 保存普通函数

  ```c++
  void func1(int a)
  {
      std::cout << a << endl;
  }
  
  std::function<void(int)>  func;
  //保存
  func = func1;
  // 执行
  func(10);
  ```

- 保存lambdab表达式

  ```c++
  std::function<void()> func = []() { cout << "hello world !" };
  
  //执行
  func();
  ```

- 保存类成员函数

  ```c++
  class A {
  public:
      void func (int a) {
          cout << a << endl;
      }
  };
  
  std::function<void(const A&, int)> func = &A::func;
  
  A a;
  func(a, 10);
  ```

  

## std::bind 函数绑定

​	std::bind是通用的函数适配器，接受一个可调用的对象，生成一个新的可调用对象来使用原对象的列表参数。生成的可调用对象std::function。

- 绑定普通函数

  ```c++
  void func1 (int x, int y, int z)
  {
      cout << x + y + z << endl;
  }
  // 使用auto推导代替std::function
  auto f1 = std::bind(func1, 1, 2, 3);
  f1(); // 输出6
  
  auto f2 = std::bind(func1, std::placeholders::_1, std::placeholders::_2, 3);
  f2(4, 5); // 输出 12
  
  
  ```

- 绑定类成员函数

  ```c++
  class A()
  {
  public:
      void func(int a, int b) {
          std::cout << a + b << endl;
      }
  };
  
  A a;
  auto f1 = std::bind(&A::func, &a, placeholders::_1, 10);
  
  f1(20); // 输出30.
  ```

  

  ## 总结，使用c++新特性实现线程池！

  ```c++
  class ThreadPool {
      struct TaskFunc {
        	std::function<void()> func;
      };
      using TaskFuncPtr = shared_ptr<TaskFunc> ;
      size_t threadNum; // 线程数量
      vector<thread*> threads;
      std::mutex mtx;
      queue<TaskFuncPtr> tasks; // 任务队列
      std::condition_varible cond;
      bool bTerminate;// 是否结束
      std::atomic<int> taskRunAtomic {0};
   public:
  	bool init(size_t num);
      bool start();
      void run();
      bool get(TaskFuncPtr &task);
      bool waitForAllDone(int millsecond);
      void stop();
      
      template<class F, class... Args>
      auto exec(F&& f, Args&&... args) -> std::future<decltype(f(args...))>  {
          //获取返回值类型
          using RetType = decltype(f(args...);
          // 封装任务
          auto task = std::make_shared<std::packaged_task<RetType()>>(std::bind(std::forward<F>(f), std::forward<Args>(args)...));
          // 封装任务指针
          TaskFuncPtr fptr = std::make_shared<TaskFunc>();
          fptr->func = [task]() {
              (*task)();
          };
          std::unique_lock<std::mutex> lock(mtx);
          tasks.push(fptr);// 插入任务
          cond.notify_one();// 唤醒线程执行
          return task->get_furture();
      }
  };
  
  bool ThreadPool::init（size_t num）{
      std::unique_lock<std::mutex> lock(mtx);
      if (!threads.empty()) {
          return false;
      }
      threadNum = num;
      return true;
  }
  
  bool ThreadPool::start() {
      std::unique_lock<std::mutex> lock(mtx);
      if (!threads.empty()) {
          return false;
      }
      for (size_t i = 0; i < threadNum; i++) {
          threads.push_back(new std::thread(&ThreadPool::run, this));
      }
      bTerminate = false;
      return true;
  }
  
  bool ThreadPool::get(TaskFuncPtr &task) {
      std::unique_lock<std::mutex> lock(mtx);
      if (tasks.empty()) {
          cond.wait(lock, [this] { return bTerminate || !tasks.empty()});
      }
      
      if (bTerminate) {
          return false;
      }
      if (!tasks.empty()) {
          task = std::move(tasks.front());
          tasks.pop();
          return true;
      }
  }
  
  void ThreadPool::run() {
      while (!bTerminate) {
          TaskFuncPtr task;
          bool ok get(task);
          if (ok) {
              ++taskRunAtomic;
              task->func();
              --taskRunAtomic;
          }
          
          {
              std::unique_lock<std::mutex> lock(mtx);
              if (atoic == 0 && tasks.empty()) {
                  cond.notify_all(); // 通知waitForAllDone
              }
          }
      }
  }
  bool ThreadPool::waitForAllDone(int millsecond) {
      std::unique_lock<std::mutex> lock(mtx);
      if (task.empty()) {
          return true;
      }
      if (millsecond < 0) {
          cond.wait(lock, [this] { return task.empty(); });
          return true;
      } else {
       	return cond.wait_for(lock, std::chrono::milliseconds(millsecond), [this] { return task.empty(); });
      }
  }
  void ThreadPool::stop() {
      {
           std::unique<std::mutex> lock(mtx);
      	 bTerminate = true;
          cond.notify_all();
      }
     for (size_t i = 0; i < threads.size(); i++) {
         if (threads[i]->joinable()) {
             threads[i]->join();
         }
         delete threads[i];
         threads[i] = nullptr;
     }
      
      std::unique_lock<std::mutex> lock(mtx);
      threads.clear();
      
  }
  
  ```

  ```c++
  // 使用线程池
  class A() {
  public:
      int add(int a, int b) {
          return a + b;
      }
  };
  ThreadPool threedPool;
  threedPool.init(10);
  threedPool.start();
  A a;
  auto f1 = threedPool.exec(std::bind(&A::add, &a, std::placeholders::_1, std::placeholders::_2), 10, 20);
  
  // 获取执行结果
  cout << f1.get() << endl; // 得到30
  ```

  

## 附加: 多态处理

```c++
// 类中函数
class A {
public:
    void func(int a) {
        std::cout << a << endl;
    } 
    
    void func(string str) {
        std::cout<< str << endl;
    }
};

A *a = new A();
//多态线程
std::thread t1((void(A::*)(int))A::func, a, 100);

std::thread t2((void(A::*)(string))A::func, a, "hello world!");

```

```c++
// 普通函数
void func(int a) {
    std::cout << a << endl;
}

void func(string str) {
    std::cout << str << endl;
}

std::thread t1((void(*)(int))func, a, 100);

std::thread t2((void(*)(string))func, a, "hello world!");
```



