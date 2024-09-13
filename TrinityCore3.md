# TrinityCore 的定时器

定时器是由两个部分组成--容器和驱动方式。

容器可以使用红黑树、最小堆、时间轮。在C++的STL里，map、set、mutimap、multiset都是红黑树实现，可以直接作为定时器的容器。

驱动方式有timefd(定时任务转换为IO事件)、带超时的阻塞系统调用(epoll_wait)、帧更新接口(mmo 游戏的update帧更新)。

在TrinityCore 使用的定时器容器使用红黑树，驱动方式使用帧更新，在updata中进行计算时间，触发定时器。

在TrinityCore中，定时器代码在`src/common/Utilities/EventProcessor.cpp`。

定时器的定义为：

```c++
class EventProcessor;

// Note. All times are in milliseconds here.
// 定时器任务基类
class TC_COMMON_API BasicEvent
{
        friend class EventProcessor;
        // 定时器的状态
        enum class AbortState : uint8
        {
            STATE_RUNNING,
            STATE_ABORT_SCHEDULED,
            STATE_ABORTED
        };

    public:
        BasicEvent()
          : m_abortState(AbortState::STATE_RUNNING), m_addTime(0), m_execTime(0) { }

        virtual ~BasicEvent() { }                           // override destructor to perform some actions on event removal

        // this method executes when the event is triggered
        // return false if event does not want to be deleted
        // e_time is execution time, p_time is update interval
        // 定时器触发，如果true，对象的生命周期由定时器管理
        virtual bool Execute(uint64 /*e_time*/, uint32 /*p_time*/) { return true; }

        virtual bool IsDeletable() const { return true; }   // this event can be safely deleted

        virtual void Abort(uint64 /*e_time*/) { }           // this method executes when the event is aborted

        // Aborts the event at the next update tick
        void ScheduleAbort();

    private:
        void SetAborted();
        bool IsRunning() const { return (m_abortState == AbortState::STATE_RUNNING); }
        bool IsAbortScheduled() const { return (m_abortState == AbortState::STATE_ABORT_SCHEDULED); }
        bool IsAborted() const { return (m_abortState == AbortState::STATE_ABORTED); }

        AbortState m_abortState;                            // set by externals when the event is aborted, aborted events don't execute

        // these can be used for time offset control
        uint64 m_addTime;         // 每次帧更新，增加的时间                          // time when the event was added to queue, filled by event handler
        uint64 m_execTime;        // 任务执行的时间                          // planned time of next execution, filled by event handler
};
// 定时器任务也可以是匿名函数（接受闭包）
template<typename T>
class LambdaBasicEvent : public BasicEvent
{
public:
    LambdaBasicEvent(T&& callback) : BasicEvent(), _callback(std::move(callback)) { }

    bool Execute(uint64, uint32) override
    {
        _callback();
        return true;
    }

private:

    T _callback;
};
// 判断是否是匿名函数，匿名函数才编译
template<typename T>
using is_lambda_event = std::enable_if_t<!std::is_base_of_v<BasicEvent, std::remove_pointer_t< std::remove_cvref_t<T> > > >;

// 定时器
class TC_COMMON_API EventProcessor
{
    public:
        EventProcessor() : m_time(0) { }
        ~EventProcessor();

        void Update(uint32 p_time);
        void KillAllEvents(bool force);
        // 向定时器增加任务，携带接受时间
        void AddEvent(BasicEvent* event, Milliseconds e_time, bool set_addtime = true);
        template<typename T>
        is_lambda_event<T> AddEvent(T&& event, Milliseconds e_time, bool set_addtime = true) { AddEvent(new LambdaBasicEvent<T>(std::move(event)), e_time, set_addtime); }
        
        // 向定时器增加任务，携带偏移时间
        void AddEventAtOffset(BasicEvent* event, Milliseconds offset) { AddEvent(event, CalculateTime(offset)); }
        void AddEventAtOffset(BasicEvent* event, Milliseconds offset, Milliseconds offset2) { AddEvent(event, CalculateTime(randtime(offset, offset2))); }
        template<typename T>
        is_lambda_event<T> AddEventAtOffset(T&& event, Milliseconds offset) { AddEventAtOffset(new LambdaBasicEvent<T>(std::move(event)), offset); }
        template<typename T>
        is_lambda_event<T> AddEventAtOffset(T&& event, Milliseconds offset, Milliseconds offset2) { AddEventAtOffset(new LambdaBasicEvent<T>(std::move(event)), offset, offset2); }


        void ModifyEventTime(BasicEvent* event, Milliseconds newTime);
        Milliseconds CalculateTime(Milliseconds t_offset) const { return Milliseconds(m_time) + t_offset; }

    protected:
        uint64 m_time; // 帧更新的参照时间
        std::multimap<uint64, BasicEvent*> m_events;    // 定时器的容器，红黑树的multimap
};
```

定时器的实现为:

```c++
void BasicEvent::ScheduleAbort()
{
    ASSERT(IsRunning()
           && "Tried to scheduled the abortion of an event twice!");
    m_abortState = AbortState::STATE_ABORT_SCHEDULED;
}

void BasicEvent::SetAborted()
{
    ASSERT(!IsAborted()
           && "Tried to abort an already aborted event!");
    m_abortState = AbortState::STATE_ABORTED;
}

EventProcessor::~EventProcessor()
{
    KillAllEvents(true);
}

void EventProcessor::Update(uint32 p_time)
{
    // update time
    m_time += p_time;

    // main event loop
    std::multimap<uint64, BasicEvent*>::iterator i;
    while (((i = m_events.begin()) != m_events.end()) && i->first <= m_time)
    {
        // get and remove event from queue
        BasicEvent* event = i->second;
        m_events.erase(i);

        if (event->IsRunning())
        {
            if (event->Execute(m_time, p_time))
            {
                // completely destroy event if it is not re-added
                // 定时器管理对象的生命周期
                delete event;
            }
            continue;
        }

        if (event->IsAbortScheduled())
        {
            event->Abort(m_time);
            // Mark the event as aborted
            event->SetAborted();
        }

        if (event->IsDeletable())
        {
            delete event;
            continue;
        }

        // Reschedule non deletable events to be checked at
        // the next update tick
        AddEvent(event, CalculateTime(1ms), false);
    }
}

void EventProcessor::KillAllEvents(bool force)
{
    for (auto itr = m_events.begin(); itr != m_events.end();)
    {
        // Abort events which weren't aborted already
        if (!itr->second->IsAborted())
        {
            itr->second->SetAborted();
            itr->second->Abort(m_time);
        }

        // Skip non-deletable events when we are
        // not forcing the event cancellation.
        if (!force && !itr->second->IsDeletable())
        {
            ++itr;
            continue;
        }

        delete itr->second;

        if (force)
            ++itr; // Clear the whole container when forcing
        else
            itr = m_events.erase(itr);
    }

    if (force)
        m_events.clear();
}

void EventProcessor::AddEvent(BasicEvent* event, Milliseconds e_time, bool set_addtime)
{
    if (set_addtime)
        event->m_addTime = m_time;
    event->m_execTime = e_time.count();
    m_events.insert(std::pair<uint64, BasicEvent*>(e_time.count(), event));
}

void EventProcessor::ModifyEventTime(BasicEvent* event, Milliseconds newTime)
{
    for (auto itr = m_events.begin(); itr != m_events.end(); ++itr)
    {
        if (itr->second != event)
            continue;

        event->m_execTime = newTime.count();
        m_events.erase(itr);
        m_events.insert(std::pair<uint64, BasicEvent*>(newTime.count(), event));
        break;
    }
}

```

在程序的单线程定时器下，全局只使用一个定时器，随着定时任务的大量增加，红黑树的效率会快速降低。在多线程大量定时任务时常常使用时间轮，由时间指针进行移动，也存在sleep过多系统调用的问题。

TrinityCore对定时器进行了优化，对定时任务进行了拆分，世界中的每一个对象(WorldObject)都有一个定时器。每一个技能作用在这个世界对象上时，创建一个技能对象定时任务，由世界对象对技能定时任务的生命周期进行管理。当开始技能时，创建一个定时任务，设置了该技能的执行时间，到定时的时间时由世界对象中的定时器直接删除任务时，也释放删除技能对象。

```c++
// src/server/game/Entities/Object/Object.h
// Event handler
EventProcessor m_Events;
```

技能的定时任务定义

```c++
// 每一个技能都是定时任务
class TC_GAME_API SpellEvent : public BasicEvent
{
public:
    explicit SpellEvent(Spell* spell);
    ~SpellEvent();

    bool Execute(uint64 e_time, uint32 p_time) override;
    void Abort(uint64 e_time) override;
    bool IsDeletable() const override;
    Spell const* GetSpell() const { return m_Spell.get(); }
    Trinity::unique_weak_ptr<Spell> GetSpellWeakPtr() const { return m_Spell; }

    std::string GetDebugInfo() const { return m_Spell->GetDebugInfo(); }

protected:
    Trinity::unique_trackable_ptr<Spell> m_Spell;
};
```

在删除定时器时，删除技能对象（m_Spell是智能指针）

```c++
SpellEvent::~SpellEvent()
{
    if (m_Spell->getState() != SPELL_STATE_FINISHED)
        m_Spell->cancel();

    if (!m_Spell->IsDeletable())
    {
        TC_LOG_ERROR("spells", "~SpellEvent: {} {} tried to delete non-deletable spell {}. Was not deleted, causes memory leak.",
            (m_Spell->GetCaster()->GetTypeId() == TYPEID_PLAYER ? "Player" : "Creature"), m_Spell->GetCaster()->GetGUID().ToString(), m_Spell->m_spellInfo->Id);
        ABORT();
    }
}
```

在技能的Spell::prepare函数中，技能任务对象增加到世界对象的定时器中，开启定时器，技能的生命周期由定时器完成。

```c++
// create and add update event for this spell
_spellEvent = new SpellEvent(this);
m_caster->m_Events.AddEvent(_spellEvent, m_caster->m_Events.CalculateTime(1ms));
```

在main函数的帧更新中实现定时器更新

```c++
void WorldUpdateLoop()
{
    uint32 minUpdateDiff = uint32(sConfigMgr->GetIntDefault("MinWorldUpdateTime", 1));
    uint32 realCurrTime = 0;
    uint32 realPrevTime = getMSTime();

    uint32 maxCoreStuckTime = uint32(sConfigMgr->GetIntDefault("MaxCoreStuckTime", 60)) * 1000;
    uint32 halfMaxCoreStuckTime = maxCoreStuckTime / 2;
    if (!halfMaxCoreStuckTime)
        halfMaxCoreStuckTime = std::numeric_limits<uint32>::max();

    LoginDatabase.WarnAboutSyncQueries(true);
    CharacterDatabase.WarnAboutSyncQueries(true);
    WorldDatabase.WarnAboutSyncQueries(true);

    ///- While we have not World::m_stopEvent, update the world
    while (!World::IsStopped())
    {
        ++World::m_worldLoopCounter;
        realCurrTime = getMSTime();

        uint32 diff = getMSTimeDiff(realPrevTime, realCurrTime);
        if (diff < minUpdateDiff)
        {
            uint32 sleepTime = minUpdateDiff - diff;
            if (sleepTime >= halfMaxCoreStuckTime)
                TC_LOG_ERROR("server.worldserver", "WorldUpdateLoop() waiting for {} ms with MaxCoreStuckTime set to {} ms", sleepTime, maxCoreStuckTime);
            // sleep until enough time passes that we can update all timers
            std::this_thread::sleep_for(Milliseconds(sleepTime));
            continue;
        }
		// 定时器更新
        sWorld->Update(diff);
        realPrevTime = realCurrTime;

#ifdef _WIN32
        if (m_ServiceStatus == 0)
            World::StopNow(SHUTDOWN_EXIT_CODE);

        while (m_ServiceStatus == 2)
            Sleep(1000);
#endif
    }

    LoginDatabase.WarnAboutSyncQueries(false);
    CharacterDatabase.WarnAboutSyncQueries(false);
    WorldDatabase.WarnAboutSyncQueries(false);
}
```



## TrinityCore 的定时器总结

1. 引入事件生命周期管理，Execute返回true时，事件处理结束后删除事件，删除事件中可以删除关联的对象。
2. 增加终止Abort操作，延迟删除事件对象。
3. 不仅仅全局一个定时器，而是每一个对象（WorldObject）一个定时器。
4. 定时器驱动方式为游戏帧更新中进行，定时事件的参照时间为对象创建时开始计算。