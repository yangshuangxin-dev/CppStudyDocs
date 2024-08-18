# 设计模式只结构型（责任链/装饰器/组合模式）

> 设计模式的目标是只需要修改少量的代码，就可以满足不断变化的需求。做法是对需求的稳定点和变化点进行分离，对稳定点进行**抽象**，对变化点进行**扩展**（继承和组合）。

## 责任链模式

​	**定义**：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之前的耦合关系。将这些对象连成一条链，并且沿着这条链传递请求，直到有一个对象处理它为止。

​	**稳定点**：请求的处理流程，按照链条传递，某一个对象处理完成后就可以返回。

​	**变化点**：新增和删除处理对象、调整处理的顺序。

​	**要点**：

- 责任链的作用是解耦请求方和处理方，请求方不知道请求是如何被处理的，处理方的组成是由独立的子处理构成，子处理流程通过链表的方式连接，子处理请求可以按任意顺序组合。
- 责任链的请求是强调请求最终由一个子流程处理，每一个子处理都可以判断是否处理。
- 责任链的扩展是功能链，功能链是指一个请求依次经过由功能链中的子处理流程进行处理。
- 将职责以及责任顺序进行抽象，职责变化可以任意的扩展，职责的顺序也可以任意的扩展。

### 责任链编程例子

> 请假流程，1天内需要主程序批准，3天内需要项目经理批准，3天以上需要版本经理批准。

**没有使用设计模式的代码**：

```c++
class Context {
public:
    std::string name;
    int day;
};

class LeaveRequest {
public:
    bool HandleRequest(const Context &ctx) {
        if (ctx.day <= 1) {
            return HandleByPL(ctx);
        } else if (ctx.day <= 3) {
            return HandleByMainProgram(ctx);
        } else if (ctx.day < 10) {
            return HandleByMgr(ctx);
        } else {
            return HandleByBOss(ctx);
        }
    }
    
private:
    bool HandleRequest(const Context &ctx) {
        return true;
    }
    
    bool HandleByMainProgram(const Context &ctx) {
        return true;
    }
    
    bool HandleByMgr(const Context &ctx) {
        return true;
    }
    
    bool HandleByBOss(const Context &ctx) {
        return true;
    }
};
```

**使用责任链改造后的代码**：

```c++
class Context {
public:
    std::string name;
    int day;
};
// 对稳定点的链表关系进行抽象，我能处理就我处理，我不能就交给下一个人。
class IHandler {
public:
    IHandler() : next(nullptr) {}
    virtual ~IHandler() {}
    void SetNextHandler(IHandler *next_) {
        this.next = next_;
    }
    
    bool Handle(const Context &ctx) {
        if (CanHandle(ctx)) {
            return HandleRequest(ctx);
        } else if (GetNextHandler()) {
            return GetNextHandler()->Handle(ctx);
        } else {
            // error 无法处理
        }
    }
protected:
    virtual bool HandleRequest(const Context &ctx) = 0;
    virtual bool CanHandle(const Context &ctx) = 0;
    IHandler * GetNextHandler() {
        return next;
    }
private:
    IHandler *next;
};
// 构建子处理流程
class HandleByPL : public IHandler {
protected:
    bool HandleRequest(const Context &ctx) override {
        return true;
    }
    
    bool CanHandle(const Context &ctx) override {
        if (ctx.day < 1) {
            return true;
        }
        return false;
    }
};

class HandleByMainProgram : public IHandler {
protected:
    bool HandleRequest(const Context &ctx) override {
        return true;
    }
    
    bool CanHandle(const Context &ctx) override {
        if (ctx.day <= 3) {
            return true;
        }
        return false;
    }
};

class HandleByMgr : public IHandler {
protected:
    bool HandleRequest(const Context &ctx) override {
        return true;
    }
    
    bool CanHandle(const Context &ctx) override {
        if (ctx.day <= 10) {
            return true;
        }
        return false;
    }
};

class HandleByBoss : public IHandler {
protected:
    bool HandleRequest(const Context &ctx) override {
        return true;
    }
    
    bool CanHandle(const Context &ctx) override {
        return true;
    }
};
// 构建请求的责任链
class Request {
public:
    Request() {
        IHandler *h0 = new HandleByPL();
        IHandler *h1 = new HandleByMainProgram();
        IHandler *h2 = new HandleByMgr();
        IHandler *h3 = new HandleByBoss();
        
        h0->SetNextHandler(h1);
        h1->SetNextHandler(h2);
        h2->SetNextHandler(h3);
        handler = h0;
    }
    
    bool Hanle(const Context &ctx) {
        return handdler->hanle(ctx);
    }
private:
    IHandler *handler;
};

// 使用方法
Context ctx;
auto req = std::make_unique<Request>();
req->Handle(ctx);
```

​	责任链的**代码结构** 为从单个节点出发，实现每一个处理接口，实现把处理单元构成一个链表的接口。这样扩展代码只需要实现新的处理接口，然后增加入责任链中。本质是分离职责，动态组合。



## 装饰器

​	**定义**：动态的给一个对象增加一些额外的职责。就增加功能而言，装饰器模式比生产子类更加灵活。

​	**稳定点**：可以顺序无关的增加职责。

​	**变化点**：增加这个行为。

​	**要点**：

- 装饰器通过组合而非继承的方式。装饰器实现了在运行时，动态的扩展对象的功能的能力，并且可以根据需要扩展多个功能。可以避免使用继承而带来的"灵活性差"、"多子类衍生" 等问题。
- 解决父类在多个方向上扩展功能的问题。
- 装饰器模式把一系列复杂的功能分散到每一个装饰器中，一个装饰器实现一个功能。

### 装饰器的例子

> 普通员工由销售奖金、累计奖金，经理还有团队奖金，主管还有环比增长奖金。不同的职位有不同的奖金组合。

**不使用装饰器的代码**：

```c++
enum User{
    DEV,
    PL,
    PM
};
class Context {
public:
    User user;
    double groupsSale;
};

class Bonus {
public:
    double CalcBouns(const Context &ctx) {
        double bonus = 0.0;
        bouns += CalcMonth(ctx);
        bonus += CalcSumBonus(ctx);
        if (ctx.user == PL) {
            bonus += CalcGroupBonus(ctx);
        }
    }
private:
    double CalcMonth(const Context &cxt) {
        return 1.0* ctx.groupsSale;
    }
    double CalcSumBonus(const Context &cxt) {
        return 2.0* ctx.groupsSale;
    }
    double CalcGroupBonus(const CalcGroupBonus &cxt) {
        return 3.0 * ctx.groupsSale;
    }
};

// 使用
Context ctx;
Bonus * bonus = new Bonus();
bonus->CalcBouns(ctx);
```

**使用设计模式装饰器进行改造**：

```c++
enum User{
    DEV,
    PL,
    PM
};
class Context {
public:
    User user;
    double groupsSale;
};

class CalcBonus {
public:
    CalcBonus(CalcBonus *c = nullptr) : cc(c) {}
    virtual ~CalcBonus() {}
    virtual double Calc(Context &ctx) {
        return 0.0;
    }
protected:
    CalcBonus *cc;
};

class CalcMonthBonus : public CalcBonus {
public:
    CalcMonthBonus(CalcBonus *calc) : CalcBonus(c) {}
    double Calc(Context &ctx) {
        return 1.0* ctx.groupsSale + cc->Calc(ctx);
    }
};

class CalcSumBonus : public CalcBonus {
public:
    CalcMonthBonus(CalcBonus *calc) : CalcBonus(c) {}
    double Calc(Context &ctx) {
        return 2.0* ctx.groupsSale + cc->Calc(ctx);
    }
};

class CalcGroupBonus : public CalcBonus {
public:
    CalcMonthBonus(CalcBonus *calc) : CalcBonus(c) {}
    double Calc(Context &ctx) {
        return 3.0* ctx.groupsSale + cc->Calc(ctx);
    }
};

// 使用方法
// 普通开发
Context ctx1 {DEV, 1.0};
CalcBonus *base = new CalcBonus();
CalcBonus *cb1 = new CalcMonthBonus(base);
CalcBonus *cb2 = new CalcSumBonus(cb1);

cb2->Calc(ctx1);

// PL
Context ctx2 {PL, 1.0}
CalcBonus *cb3 = new CalcSumBonus(cb2);
cb3->Calc(ctx2);
```

​	装饰器模式在不影响其他对象的情况下，以动态、透明的方式给对象增加职责。每一个职责都是完全独立的功能，彼此直接没有依赖。本质是动态组合。

## 组合模式

​	**定义**：将对象组合成树型结构以表示"部分 -- 整体" 的层次结构。组合对象使得用户对单个对象和组合对象的使用具有一致性。

​	**稳定点**：对象和组合对象之间的层次关系是稳定的。

​	**变化点**：对象的职责变更、组合对象里的对象变更。

​	**要点**：

- 在表示对象的 "部分-- 整体" 层次机构时，选择组合模式。组合模式把字体和部分的操作统一起来，从而使层次结构实现更简单。
- 组合模式可以统一的使用组合结构中的所有对象。

### 组合模式使用例子

> 树中将叶子节点当成特殊的组合对象看待，从而统一叶子对象和组合对象，执行都放入叶子节点，中间的节点进行数据处理。

```c++
class IComponent {
public:
    IComponent() {}
    virtual ~IComponent() {}
    virtual void Execute() = 0;
    virtual void AddChild(IComponent *ele) {}
    virtual void RemoveChild (IComponent *ele) {}
};

class Leaf : public IComponent {
public:
    void Execute() override {
        cout << "leaf execute !!" << endl;
    }
};

class Composite : public IComponent {
public:
    void AddChild(IComponent * ele) override {
        leafs.emplace(ele);
    }
    
    void RemoveChild(IComponent * ele) override {
        leafs.erase(ele);
    }
    
    void Execute()  override {
        for (const auto iter : leafs) {
            iter->Execute();
        }
    }
private:
    std::list<IComponent*> leafs;
}
```

