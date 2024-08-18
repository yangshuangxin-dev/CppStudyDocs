# 设计模式之结构型（单例/工厂/抽象工厂）

> 设计模式主要的目标是只需要修改少量的代码就可以满足需求的变化。解决方法是寻找程序中的稳定点和变化点，对稳定点进行**抽象**，对于变化点进行**扩展（继承和组合）**。

## 单例模式

​	**定义**：保证一个类仅有一个实例，并提供一个该实例的全局访问点。

​	**稳定点**：类只有一个实例，并且提供一个全局的访问点。

​	**变化点：**有多个类都是单例，如何复用代码。

​	**要点：**需要私有化构造函数，并且禁用拷贝构造函数、拷贝赋值构造函数、移动构造函数、移动拷贝构造函数。

```c++
class T {
public:
    T() {} //构造
    ~T() {} // 析构
    T(const T &) {} // 拷贝构造
    T& operator= (const T&) {} // 拷贝赋值构造
    T(T &&) {} // 移动构造
    T& operator=(T &&) {} // 移动赋值构造
};

// 拷贝构造
T t1;
T t2 = t1;
T t2(t1);

// 拷贝赋值构造
T t1;
T t2;
t1 = t2;

// 移动构造
T CreateT() {
    T temp;
    return temp;
}

T t =  CreateT();
//---- or ---
T t1;
T t2(std::move(t1));

// 移动赋值构造
T t;
t = T();
//----  or -----
T t1, t2;
t1 = std::move(t2);
```

### 版本一

```c++
class Singleton {
public:
    static Singleton * GetInstance() {
        if (instance_ == nullptr) {
            instance_ = new Singleton();
        }
        return instance_;
    }
private:
    Singleton() {} // 构造
    ~Singleton() {} 
    Singleton(const Singleton &) = delete; // 拷贝构造
    Singleton& operator= (const Singleton &) = delete; // 拷贝赋值构造
     Singleton(Singleton &&) = delete; // 移动构造
     Singleton& operator= (Singleton &&) = delete; // 移动赋值构造
    static Singleton* instance_;
};

Singleton* Singleton::instance_ = nullptr;// 初始化静态成员
```

**版本一存在的问题：** 结束时无法调用析构函数，可能会造成资源的泄漏或数据的丢失。

### 版本二

> 解决可能存在的资源泄漏或数据丢失问题。

```c++
class Singleton {
public:
    static Singleton * GetInstance() {
        if (instance_ == nullptr) {
            instance_ = new Singleton();
            atexit(Destructor);
        }
        return instance_;
    }
private:
    static void Destructor() {
        if (instance_ == nullptr) {
            delete instance_;
            instance_ = nullptr;
        }
    }
    Singleton() {} // 构造
    ~Singleton() {} 
    Singleton(const Singleton &) = delete; // 拷贝构造
    Singleton& operator= (const Singleton &) = delete; // 拷贝赋值构造
     Singleton(Singleton &&) = delete; // 移动构造
     Singleton& operator= (Singleton &&) = delete; // 移动赋值构造
    static Singleton* instance_;
};

Singleton* Singleton::instance_ = nullptr;// 初始化静态成员
```

**版本二存在的问题：** 多线程不安全，多个线性访问可能造成内存泄漏。

### 版本三

> 多线程安全

```c++
class Singleton {
public:
    static Singleton * GetInstance() {
        // std::lock_guard<std::mutex> lock(mtx); // RAII 类申明周期进行加锁，这里加锁虽然可以解决问题，但是会影响效率。
        if (instance_ == nullptr) {
            // 双重检测，保证执行效率
            std::lock_guard<std::mutex> lock(mtx);
            if (instance_ == nullptr) {
                instance_ = new Singleton();
            	atexit(Destructor);
            }
        }
        return instance_;
    }
private:
    static void Destructor() {
        if (instance_ == nullptr) {
            delete instance_;
            instance_ = nullptr;
        }
    }
    Singleton() {} // 构造
    ~Singleton() {} 
    Singleton(const Singleton &) = delete; // 拷贝构造
    Singleton& operator= (const Singleton &) = delete; // 拷贝赋值构造
     Singleton(Singleton &&) = delete; // 移动构造
     Singleton& operator= (Singleton &&) = delete; // 移动赋值构造
    static Singleton* instance_;
    static std::mutex mtx;
};

Singleton* Singleton::instance_ = nullptr;// 初始化静态成员
```

**版本三存在的问题：**

​	执行`instance_ = new Singleton();` 时，是多条指令，主要是：

1. 分配内存
2. 调用析构函数
3. 返回指针

​	在多核、多线程的CPU的环境下，会产生编译器重排或CPU重排执行指令，会违反顺序一致性。从而产生可见性问题和执行序问题。解决方法是使用**原子变量**和**内存栅栏**。

​	原子变量可以解决原子执行问题、可见性问题、执行序问题。原子变量的`load` 可以看见其他线程最新操作的数据。原子变量的`store` 修改的数据其他线程可见。

​	内存栅栏可以解决可见性问题和执行序问题，通常使用的内存模型为`memoey_order_acquire` 和`memory_order_release`。

### 版本四

> 解决多线程环境下 CPU 的 reorder  操作

```c++
// 只使用原子变量
class Singleton {
public:
    static Singleton * GetInstance() {
		Singleton* tmp = instance_.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            // 双重检测，保证执行效率
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance_.load(std::memory_order_acquire);
            if (tmp == nullptr) {
                // 如果构造函数中也有原子操作，可能会影响执行效率。
                tmp = new Singleton();
                instance_.store(tmp, std::memory_order_release);
            	atexit(Destructor);
            }
        }
        return tmp;
    }
private:
    static void Destructor() {
        if (instance_ == nullptr) {
            delete instance_;
            instance_ = nullptr;
        }
    }
    Singleton() {} // 构造
    ~Singleton() {} 
    Singleton(const Singleton &) = delete; // 拷贝构造
    Singleton& operator= (const Singleton &) = delete; // 拷贝赋值构造
     Singleton(Singleton &&) = delete; // 移动构造
     Singleton& operator= (Singleton &&) = delete; // 移动赋值构造
    static std::atomic<Singleton*> instance_;
    static std::mutex mtx;
};
```

```c++
// 使用内存栅栏和原子变量
class Singleton {
public:
    static Singleton * GetInstance() {
		Singleton* tmp = instance_.load(std::memory_order_relaxed);
        std::atomic_thread_fence(std::memory_order_acquire); // 获取内存屏障
        if (tmp == nullptr) {
            // 双重检测，保证执行效率
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance_.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                // 如果构造函数中也有原子操作，可能会影响执行效率。
                tmp = new Singleton();
                std::atomic_thread_fence(std::memory_order_release);// 释放内存屏障
                instance_.store(tmp, std::memory_order_relaxed);
            	atexit(Destructor);
            }
        }
        return tmp;
    }
private:
    static void Destructor() {
        if (instance_ == nullptr) {
            delete instance_;
            instance_ = nullptr;
        }
    }
    Singleton() {} // 构造
    ~Singleton() {} 
    Singleton(const Singleton &) = delete; // 拷贝构造
    Singleton& operator= (const Singleton &) = delete; // 拷贝赋值构造
     Singleton(Singleton &&) = delete; // 移动构造
     Singleton& operator= (Singleton &&) = delete; // 移动赋值构造
    static std::atomic<Singleton*> instance_;
    static std::mutex mtx;
};
```

**版本四存在的问题：** 太复杂。

### 版本五

> C++11 中的static 新特性：如果当 变量在初始化的时候，并发同时进入声明语句时，并发线程将会阻塞等待初始化结束。

利用C++11 的static 静态局部变量新特性，从而可以实现：

1. 静态局部变量的延迟加载。
2. 静态局部变量的系统自动回收内存，自动调用析构函数。
3. 静态局部变量初始化时，没有new 操作带来的CPU指令reorder 操作。
4. 静态局部变量初始化时，具备线程安全。

```c++
class Singleton {
public:
    static Singleton * GetInstance() {
		static Singleton instance;
        return instance;
    }
private:
    Singleton() {} // 构造
    ~Singleton() {} 
    Singleton(const Singleton &) = delete; // 拷贝构造
    Singleton& operator= (const Singleton &) = delete; // 拷贝赋值构造
     Singleton(Singleton &&) = delete; // 移动构造
     Singleton& operator= (Singleton &&) = delete; // 移动赋值构造
};
```

​	**版本五的缺点：** 不通用。

### 版本六

> 使用模板进行抽象和封装。

```c++
template<typename T>
class Singleton {
public:
    static T& GetInstance() {
        static T instance;
        return instance;
    }
protected:
    virtual Singleton() {}
    virtual ~Singleton() {}
private:
    Singleton(const Singleton&) = delete;
    Singleton& operator= (const Singleton&) = delete;
    Singleton(Singleton &&) = delete;
    Singleton& operator=(Singleton &&) = delete;
};

// 使用方法
class TestPattern : public Singleton<TestPattern> {
    //为了让父类访问到TestPattern的构造函数，需要设置友元。
 	friend class Singleton<TestPattern>;
private:
    TestPattern() {}
    ~TestPattern() {}
};
```

## 工厂模式

​	**定义：** 定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂方法使得一个类的实例化延迟到子类。

​	**稳定点：**1. 创建同类对象的接口   2.  同类对象都有一个相同的职责。

​	**变化点：** 创建对象进行的扩展。

​	**要点：**解决对象创建过程中复杂的细节，希望对外隐藏这些细节。本质是延迟到子类来选择实现。使用的场景一般有：

- 连接池、线程池。
- 隐藏对象的真实类型。
- 对象创建会有很多参数来决定如何场景。
- 创建对象有复杂的依赖关系。

> 实现一个导出数据的接口，让客户选择导出数据的方式。

​	**使用设计模式之前：**

```c++
// 实现导出数据的接口, 导出数据的格式包含 xml，json，文本格式txt 后面可能扩展excel格式csv
class IExport {
public:
    virtual bool Export(const std::string &data) = 0;
    virtual ~IExport(){}
};

class ExportXml : public IExport {
public: 
    bool Export(const std::string &data) override {
        return true;
    }
};

class ExportJson : public IExport {
public: 
    bool Export(const std::string &data) override {
        return true;
    }
};

class ExportCsv : public IExport {
public: 
    bool Export(const std::string &data) override {
        return true;
    }
};

// 使用方法：
std::string choose;
std::cin >> choose;
if (choose == "txt") {
    IExport* e = new ExportCsv();
    // 进行大量初始化操作
    e->Export("Hello World!");
} else if (choose == "json") {
     // 进行大量初始化操作
    IExport* e = new ExportJson();
     // 进行大量初始化操作
    e->Export("Hello World!");
} else if (choose == "xml") {
    IExport* e = new ExportXml();
     // 进行大量初始化操作
    e->Export("Hello World!");
}
```

​	**使用设计模式进行封装：**

```c++
// 前面相同，略过
// 工厂类
class IExportFactory {
public:
    virtual IExportFactory() : export(nullptr) {}
    virtual ~IExportFactory() {
        if (export == nullptr) {
            delete export;
            export = nullptr;
        }
    }
    
    bool Export(const std::string &data) {
        if (export == nullptr) {
            export = NewExport();
        }
        return export->Export();
    }
protected:
    virtual IExport* NewExport(/* 构造参数 */) = 0;
private:
    IExport* export;
};


// 具体的工厂
class ExportXmlFactory : IExportFactory {
protected:
    IExport* NewExport(/* 构造参数 */) override {
        IExport* temp = new ExportXml();
        // 其他的初始化操作
        return temp;
    }
};

class ExportJsonFactory : IExportFactory {
protected:
    IExport* NewExport(/* 构造参数 */) override {
        IExport* temp = new ExportJson();
        // 其他的初始化操作
        return temp;
    }
};

class ExportCsvFactory : IExportFactory {
protected:
    IExport* NewExport(/* 构造参数 */) override {
        IExport* temp = new ExportCsv();
        // 其他的初始化操作
        return temp;
    }
};

// 使用方法
IExportFactory *factory = new ExportXmlFactory();
factory->Export("Hello World!");
```

​	代码结构为：

	1.  对象创建的接口，创建具体的对象，然后调用功能结构。
	1.  功能接口，对象实现功能接口。

​	设计原则：

- 最小知道原则
- 面向对象接口原则

​	扩展代码的步骤：

1. 实现对象创建接口。

2. 实现功能接口。

3. 多态调用。

## 抽象工厂

​	**定义：** 提供一个接口，让该接口负责创建一系列 "相关或者相互依赖的对象 "， 无需指定它们具体的类。

​	**稳定点：** 场景同类对象的接口、同类对象有**许多**相同的职责。

​	**变化点：** 创建对象的扩展。

> 实现一个拥有导入、导出数据的接口，让客户选择导入、导出的方式。

```c++
// 导出接口代码，与上面一致，略过。
// 导入接口
class IImport {
public:
    virtual bool Import(const std::string &data) = 0;
    virtual ~IImport(){}
};

class ImportXml : public IImport {
public:
    virtual bool Import(const std::string &data) {
        return true;
    }
};

class ImportJson : public IImport {
public:
    virtual bool Import(const std::string &data) {
        return true;
    }
};

class ImportCSV : public IImport {
public:
    virtual bool Import(const std::string &data) {
        return true;
    }
};

// 工厂类 
class IDataApiFactory {
public:
    IDataApiFactory(): export(nullptr), import(nullptr) {}
    virtual ~IDataApiFactory() {
        if (export) {
            delete export;
            export = nullptr;
        }
        if (import) {
            delete _import;
            import = nullptr;
        }
    }
    bool Export(const std::string &data) {
        if (export == nullptr) {
            export = NewExport();
        }
        return export->Export(data);
    }
    bool Import(const std::string &data) {
        if (import == nullptr) {
            import = NewImport();
        }
        return import->Import(data);
    }
protected:
    virtual IExport * NewExport(/* ... */) = 0;
    virtual IImport * NewImport(/* ... */) = 0;
private:
    IExport *_export;
    IImport *_import;
};

// 具体的工厂：
class XmlApiFactory : public IDataApiFactory {
protected:
    virtual IExport * NewExport(/* ... */) {
        // 可能有其它操作，或者许多参数
        IExport * temp = new ExportXml;
        // 可能之后有什么初始化操作
        return temp;
    }
    virtual IImport * NewImport(/* ... */) {
        // 可能有其它操作，或者许多参数
        IImport * temp = new ImportXml;
        // 可能之后有什么初始化操作
        return temp;
    }
};

class JsonApiFactory : public IDataApiFactory {
protected:
    virtual IExport * NewExport(/* ... */) {
        // 可能有其它操作，或者许多参数
        IExport * temp = new ExportJson;
        // 之后有什么初始化操作
        return temp;
    }
    virtual IImport * NewImport(/* ... */) {
        // 可能有其它操作，或者许多参数
        IImport * temp = new ImportJson;
        // 可能之后有什么初始化操作
        return temp;
    }
};

class CSVApiFactory : public IDataApiFactory {
protected:
    virtual IExport * NewExport(/* ... */) {
        // 可能有其它操作，或者许多参数
        IExport * temp = new ExportCSV;
        //可能之后有什么初始化操作
        return temp;
    }
    virtual IImport * NewImport(/* ... */) {
        // 可能有其它操作，或者许多参数
        IImport * temp = new ImportCSV;
        // 可能之后有什么初始化操作
        return temp;
    }
};

// 使用方法
IDataApiFactory *factory = new XmlApiFactory();
factory->Import("Hello World！");
factory->Export("Hello World！")
```

