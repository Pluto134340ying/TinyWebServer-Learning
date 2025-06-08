# webserver拆解

# 并发框架

最核⼼的部分就是 EventLoop 、 Channel 以及 Poller 三个类，其中 EventLoop 可以看作是对业务线程的封装，⽽ Channel 可以看作是对每 个已经建⽴连接的封装（即 accept(3) 返回的⽂件描述符），三者的关系⻅下图（来源⻅⽔印）。  

![image.webp](image.webp)

# main.cc

```cpp
#include <string>

#include <TcpServer.h>
#include <Logger.h>
#include <sys/stat.h>
#include <sstream>
#include "AsyncLogging.h"
#include "LFU.h"
#include "memoryPool.h"
// 日志文件滚动大小为1MB (1*1024*1024字节)
static const off_t kRollSize = 1*1024*1024;
class EchoServer
{
public:
    EchoServer(EventLoop *loop, const InetAddress &addr, const std::string &name)
        : server_(loop, addr, name)
        , loop_(loop)
    {
        // 注册回调函数
        server_.setConnectionCallback(
            std::bind(&EchoServer::onConnection, this, std::placeholders::_1));
        
        server_.setMessageCallback(
            std::bind(&EchoServer::onMessage, this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));

        // 设置合适的subloop线程数量
        server_.setThreadNum(3);
    }
    void start()
    {
        server_.start();
    }

private:
    // 连接建立或断开的回调函数
    void onConnection(const TcpConnectionPtr &conn)   
    {
        if (conn->connected())
        {
            LOG_INFO<<"Connection UP :"<<conn->peerAddress().toIpPort().c_str();
        }
        else
        {
            LOG_INFO<<"Connection DOWN :"<<conn->peerAddress().toIpPort().c_str();
        }
    }

    // 可读写事件回调
    void onMessage(const TcpConnectionPtr &conn, Buffer *buf, Timestamp time)
    {
        std::string msg = buf->retrieveAllAsString();
        conn->send(msg);
        // conn->shutdown();   // 关闭写端 底层响应EPOLLHUP => 执行closeCallback_
    }
    TcpServer server_;
    EventLoop *loop_;

};
AsyncLogging* g_asyncLog = NULL;
AsyncLogging * getAsyncLog(){
    return g_asyncLog;
}
 void asyncLog(const char* msg, int len)
{
    AsyncLogging* logging = getAsyncLog();
    if (logging)
    {
        logging->append(msg, len);
    }
}
int main(int argc,char *argv[]) {
    //第一步启动日志，双缓冲异步写入磁盘.
    //创建一个文件夹
    const std::string LogDir="logs";
    mkdir(LogDir.c_str(),0755);
    //使用std::stringstream 构建日志文件夹
    std::ostringstream LogfilePath;
    LogfilePath << LogDir << "/" << ::basename(argv[0]); // 完整的日志文件路径
    AsyncLogging log(LogfilePath.str(), kRollSize);
    g_asyncLog = &log;
    Logger::setOutput(asyncLog); // 为Logger设置输出回调, 重新配接输出位置
    log.start(); // 开启日志后端线程
    //第二步启动内存池和LFU缓存
     // 初始化内存池
    memoryPool::HashBucket::initMemoryPool();

    // 初始化缓存
    const int CAPACITY = 5;  
    KamaCache::KLfuCache<int, std::string> lfu(CAPACITY);
    //第三步启动底层网络模块
    EventLoop loop;
    InetAddress addr(8080);
    EchoServer server(&loop, addr, "EchoServer");
    server.start();
 // 主loop开始事件循环  epoll_wait阻塞 等待就绪事件(主loop只注册了监听套接字的fd，所以只会处理新连接事件)
    std::cout << "================================================Start Web Server================================================" << std::endl;
    loop.loop();
    std::cout << "================================================Stop Web Server=================================================" << std::endl;
    //结束日志打印
    log.stop();
}
```

这是 kama-webserver 项目的入口文件，展示了一个基于 muduo 网络库的 Web 服务器实现，同时集成了异步日志、LFU 缓存和内存池等高级功能。

## 主要组件

### 1. EchoServer 类

这是一个简单的回显服务器实现，它接收客户端消息并将其原样返回：

```cpp
class EchoServer
{
public:
    EchoServer(EventLoop *loop, const InetAddress &addr, const std::string &name);
    void start();
private:
    void onConnection(const TcpConnectionPtr &conn);
    void onMessage(const TcpConnectionPtr &conn, Buffer *buf, Timestamp time);
    TcpServer server_;
    EventLoop *loop_;
};

```

- `onConnection`: 处理连接建立/断开事件，输出日志
- `onMessage`: 处理消息到达事件，将接收到的消息回显给客户端
- 设置了 3 个 IO 线程处理连接（`server_.setThreadNum(3)`）

### 2. 异步日志系统

使用双缓冲技术实现的高性能异步日志系统：

```cpp
// 日志文件滚动大小为1MB
static const off_t kRollSize = 1*1024*1024;
AsyncLogging* g_asyncLog = NULL;

AsyncLogging * getAsyncLog() {
    return g_asyncLog;
}

void asyncLog(const char* msg, int len) {
    AsyncLogging* logging = getAsyncLog();
    if (logging) {
        logging->append(msg, len);
    }
}

```

日志系统特点：

- 日志文件每达到 1MB 会自动滚动
- 使用全局指针访问日志实例
- 通过 `asyncLog` 回调将日志消息追加到后端缓冲区

### 3. 性能优化组件

```cpp
// 初始化内存池
memoryPool::HashBucket::initMemoryPool();

// 初始化缓存
const int CAPACITY = 5;
KamaCache::KLfuCache<int, std::string> lfu(CAPACITY);
```

- **内存池**：自定义内存分配器，减少内存碎片和系统调用开销
- **LFU 缓存**：基于访问频率的缓存策略，提高热点数据访问性能

### 4. 主函数流程

主函数按以下顺序初始化各组件：

1. **设置日志系统**：
    - 创建日志目录
    - 初始化异步日志对象
    - 重定向 Logger 输出到异步日志系统
    - 启动日志后端线程
2. **初始化性能组件**：
    - 初始化内存池
    - 创建 LFU 缓存实例
3. **启动网络服务**：
    - 创建事件循环（EventLoop）
    - 绑定服务器地址（端口 8080）
    - 创建并启动 Echo 服务器
    - 开始事件循环
4. **优雅退出**：
    - 退出事件循环后停止日志系统

## 设计特点

1. **模块化设计**：网络、日志、缓存、内存管理各自独立
2. **高性能考虑**：使用异步日志、内存池和缓存优化性能
3. **事件驱动模型**：基于 muduo 的 Reactor 模式处理并发连接
4. **资源管理**：程序启动和退出时正确初始化和清理资源

这个程序展示了一个生产级 Web 服务器的基本架构，整合了多种高性能服务器常用的技术和组件。

# Eventloop.cc

```cpp
#include <sys/eventfd.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <memory>

#include <EventLoop.h>
#include <Logger.h>
#include <Channel.h>
#include <Poller.h>

// 防止一个线程创建多个EventLoop
thread_local EventLoop *t_loopInThisThread = nullptr;

// 定义默认的Poller IO复用接口的超时时间
const int kPollTimeMs = 10000; // 10000毫秒 = 10秒钟

/* 创建线程之后主线程和子线程谁先运行是不确定的。
 * 通过一个eventfd在线程之间传递数据的好处是多个线程无需上锁就可以实现同步。
 * eventfd支持的最低内核版本为Linux 2.6.27,在2.6.26及之前的版本也可以使用eventfd，但是flags必须设置为0。
 * 函数原型：
 *     #include <sys/eventfd.h>
 *     int eventfd(unsigned int initval, int flags);
 * 参数说明：
 *      initval,初始化计数器的值。
 *      flags, EFD_NONBLOCK,设置socket为非阻塞。
 *             EFD_CLOEXEC，执行fork的时候，在父进程中的描述符会自动关闭，子进程中的描述符保留。
 * 场景：
 *     eventfd可以用于同一个进程之中的线程之间的通信。
 *     eventfd还可以用于同亲缘关系的进程之间的通信。
 *     eventfd用于不同亲缘关系的进程之间通信的话需要把eventfd放在几个进程共享的共享内存中（没有测试过）。
 */
// 创建wakeupfd 用来notify唤醒subReactor处理新来的channel
int createEventfd()
{
    int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evtfd < 0)
    {
        LOG_FATAL<<"eventfd error:%d"<<errno;
    }
    return evtfd;
}

EventLoop::EventLoop()
    : looping_(false)
    , quit_(false)
    , callingPendingFunctors_(false)
    , threadId_(CurrentThread::tid())
    , poller_(Poller::newDefaultPoller(this))
    , wakeupFd_(createEventfd())
    , wakeupChannel_(new Channel(this, wakeupFd_))
{
    LOG_DEBUG<<"EventLoop created"<<this<<"in thread"<<threadId_;
    if (t_loopInThisThread)
    {
        LOG_FATAL<<"Another EventLoop"<<t_loopInThisThread<<"exists in this thread "<<threadId_;
    }
    else
    {
        t_loopInThisThread = this;
    }
    
    wakeupChannel_->setReadCallback(
        std::bind(&EventLoop::handleRead, this)); // 设置wakeupfd的事件类型以及发生事件后的回调操作
    
    wakeupChannel_->enableReading(); // 每一个EventLoop都将监听wakeupChannel_的EPOLL读事件了
}
EventLoop::~EventLoop()
{
    wakeupChannel_->disableAll(); // 给Channel移除所有感兴趣的事件
    wakeupChannel_->remove();     // 把Channel从EventLoop上删除掉
    ::close(wakeupFd_);
    t_loopInThisThread = nullptr;
}

// 开启事件循环
void EventLoop::loop()
{
    looping_ = true;
    quit_ = false;

    LOG_INFO<<"EventLoop start looping";

    while (!quit_)
    {
        activeChannels_.clear();
        pollRetureTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
        for (Channel *channel : activeChannels_)
        {
            // Poller监听哪些channel发生了事件 然后上报给EventLoop 通知channel处理相应的事件
            channel->handleEvent(pollRetureTime_);
        }
        /**
         * 执行当前EventLoop事件循环需要处理的回调操作 对于线程数 >=2 的情况 IO线程 mainloop(mainReactor) 主要工作：
         * accept接收连接 => 将accept返回的connfd打包为Channel => TcpServer::newConnection通过轮询将TcpConnection对象分配给subloop处理
         *
         * mainloop调用queueInLoop将回调加入subloop（该回调需要subloop执行 但subloop还在poller_->poll处阻塞） queueInLoop通过wakeup将subloop唤醒
         **/
        doPendingFunctors();
    }
    LOG_INFO<<"EventLoopstop looping";
    looping_ = false;
}

/**
 * 退出事件循环
 * 1. 如果loop在自己的线程中调用quit成功了 说明当前线程已经执行完毕了loop()函数的poller_->poll并退出
 * 2. 如果不是当前EventLoop所属线程中调用quit退出EventLoop 需要唤醒EventLoop所属线程的epoll_wait
 *
 * 比如在一个subloop(worker)中调用mainloop(IO)的quit时 需要唤醒mainloop(IO)的poller_->poll 让其执行完loop()函数
 *
 * ！！！ 注意： 正常情况下 mainloop负责请求连接 将回调写入subloop中 通过生产者消费者模型即可实现线程安全的队列
 * ！！！       但是muduo通过wakeup()机制 使用eventfd创建的wakeupFd_ notify 使得mainloop和subloop之间能够进行通信
 **/
void EventLoop::quit()
{
    quit_ = true;

    if (!isInLoopThread())
    {
        wakeup();
    }
}

// 在当前loop中执行cb
void EventLoop::runInLoop(Functor cb)
{
    if (isInLoopThread()) // 当前EventLoop中执行回调
    {
        cb();
    }
    else // 在非当前EventLoop线程中执行cb，就需要唤醒EventLoop所在线程执行cb
    {
        queueInLoop(cb);
    }
}

// 把cb放入队列中 唤醒loop所在的线程执行cb
void EventLoop::queueInLoop(Functor cb)
{
    {
        std::unique_lock<std::mutex> lock(mutex_);
        pendingFunctors_.emplace_back(cb);
    }

    /**
     * || callingPendingFunctors的意思是 当前loop正在执行回调中 但是loop的pendingFunctors_中又加入了新的回调 需要通过wakeup写事件
     * 唤醒相应的需要执行上面回调操作的loop的线程 让loop()下一次poller_->poll()不再阻塞（阻塞的话会延迟前一次新加入的回调的执行），然后
     * 继续执行pendingFunctors_中的回调函数
     **/
    if (!isInLoopThread() || callingPendingFunctors_)
    {
        wakeup(); // 唤醒loop所在线程
    }
}

void EventLoop::handleRead()
{
    uint64_t one = 1;
    ssize_t n = read(wakeupFd_, &one, sizeof(one));
    if (n != sizeof(one))
    {
        LOG_ERROR<<"EventLoop::handleRead() reads"<<n<<"bytes instead of 8";
    }
}

// 用来唤醒loop所在线程 向wakeupFd_写一个数据 wakeupChannel就发生读事件 当前loop线程就会被唤醒
void EventLoop::wakeup()
{
    uint64_t one = 1;
    ssize_t n = write(wakeupFd_, &one, sizeof(one));
    if (n != sizeof(one))
    {
        LOG_ERROR<<"EventLoop::wakeup() writes"<<n<<"bytes instead of 8";
    }
}

// EventLoop的方法 => Poller的方法
void EventLoop::updateChannel(Channel *channel)
{
    poller_->updateChannel(channel);
}

void EventLoop::removeChannel(Channel *channel)
{
    poller_->removeChannel(channel);
}

bool EventLoop::hasChannel(Channel *channel)
{
    return poller_->hasChannel(channel);
}

void EventLoop::doPendingFunctors()
{
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;

    {
        std::unique_lock<std::mutex> lock(mutex_);
        functors.swap(pendingFunctors_); // 交换的方式减少了锁的临界区范围 提升效率 同时避免了死锁 如果执行functor()在临界区内 且functor()中调用queueInLoop()就会产生死锁
    }

    for (const Functor &functor : functors)
    {
        functor(); // 执行当前loop需要执行的回调操作
    }

    callingPendingFunctors_ = false;
}

```

## Eventloop.h

```cpp
#pragma once

#include <functional>
#include <vector>
#include <atomic>
#include <memory>
#include <mutex>

#include "noncopyable.h"
#include "Timestamp.h"
#include "CurrentThread.h"
#include "TimerQueue.h"
class Channel;
class Poller;

// 事件循环类 主要包含了两个大模块 Channel Poller(epoll的抽象)
class EventLoop : noncopyable
{
public:
    using Functor = std::function<void()>;

    EventLoop();
    ~EventLoop();

    // 开启事件循环
    void loop();
    // 退出事件循环
    void quit();

    Timestamp pollReturnTime() const { return pollRetureTime_; }

    // 在当前loop中执行
    void runInLoop(Functor cb);
    // 把上层注册的回调函数cb放入队列中 唤醒loop所在的线程执行cb
    void queueInLoop(Functor cb);

    // 通过eventfd唤醒loop所在的线程
    void wakeup();

    // EventLoop的方法 => Poller的方法
    void updateChannel(Channel *channel);
    void removeChannel(Channel *channel);
    bool hasChannel(Channel *channel);

    // 判断EventLoop对象是否在自己的线程里
    bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); } // threadId_为EventLoop创建时的线程id CurrentThread::tid()为当前线程id
    /**
     * 定时任务相关函数
     */
    void runAt(Timestamp timestamp, Functor &&cb)
    {
        timerQueue_->addTimer(std::move(cb), timestamp, 0.0);
    }

    void runAfter(double waitTime, Functor &&cb)
    {
        Timestamp time(addTime(Timestamp::now(), waitTime));
        runAt(time, std::move(cb));
    }

    void runEvery(double interval, Functor &&cb)
    {
        Timestamp timestamp(addTime(Timestamp::now(), interval));
        timerQueue_->addTimer(std::move(cb), timestamp, interval);
    }

private:
    void handleRead();        // 给eventfd返回的文件描述符wakeupFd_绑定的事件回调 当wakeup()时 即有事件发生时 调用handleRead()读wakeupFd_的8字节 同时唤醒阻塞的epoll_wait
    void doPendingFunctors(); // 执行上层回调

    using ChannelList = std::vector<Channel *>;

    std::atomic_bool looping_; // 原子操作 底层通过CAS实现
    std::atomic_bool quit_;    // 标识退出loop循环

    const pid_t threadId_; // 记录当前EventLoop是被哪个线程id创建的 即标识了当前EventLoop的所属线程id

    Timestamp pollRetureTime_; // Poller返回发生事件的Channels的时间点
    std::unique_ptr<Poller> poller_;
    std::unique_ptr<TimerQueue> timerQueue_;
    int wakeupFd_; // 作用：当mainLoop获取一个新用户的Channel 需通过轮询算法选择一个subLoop 通过该成员唤醒subLoop处理Channel
    std::unique_ptr<Channel> wakeupChannel_;

    ChannelList activeChannels_; // 返回Poller检测到当前有事件发生的所有Channel列表

    std::atomic_bool callingPendingFunctors_; // 标识当前loop是否有需要执行的回调操作
    std::vector<Functor> pendingFunctors_;    // 存储loop需要执行的所有回调操作
    std::mutex mutex_;                        // 互斥锁 用来保护上面vector容器的线程安全操作
};
```

[`EventLoop.cc`][EventLoop.cc](http://eventloop.cc/) ) 实现了 Reactor 模式中的事件循环核心，是 kama-webserver 网络库的核心组件。每个线程最多只能有一个 EventLoop 实例，负责管理该线程内的所有 I/O 事件。

## 核心概念

### 1. 主要成员变量

```cpp
thread_local EventLoop *t_loopInThisThread = nullptr;  // 线程局部存储
looping_;        // 是否处于事件循环中
quit_;           // 是否退出事件循环
callingPendingFunctors_;  // 是否正在执行回调队列
threadId_;       // EventLoop所属线程ID
poller_;         // IO复用对象（epoll的封装）
wakeupFd_;       // 用于跨线程唤醒EventLoop的文件描述符
wakeupChannel_;  // 封装wakeupFd_的Channel对象
activeChannels_; // poller返回的活跃Channel列表
pendingFunctors_; // 待执行的回调函数队列

```

### 2. eventfd 机制

```cpp
int createEventfd()
{
    int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evtfd < 0)
    {
        LOG_FATAL<<"eventfd error:"<<errno;
    }
    return evtfd;
}

```

EventLoop 使用 eventfd 实现了线程间通信和唤醒机制，避免了多线程环境下的锁竞争问题。

## 核心功能

### 1. 构造与析构

```cpp
EventLoop::EventLoop()
    : looping_(false)
    , quit_(false)
    , callingPendingFunctors_(false)
    , threadId_(CurrentThread::tid())
    , poller_(Poller::newDefaultPoller(this))
    , wakeupFd_(createEventfd())
    , wakeupChannel_(new Channel(this, wakeupFd_))
{
    // 确保一个线程只有一个EventLoop
    if (t_loopInThisThread) {
        LOG_FATAL<<"Another EventLoop"<<t_loopInThisThread<<"exists in this thread "<<threadId_;
    } else {
        t_loopInThisThread = this;
    }

    // 设置wakeupChannel的回调和事件监听
    wakeupChannel_->setReadCallback(std::bind(&EventLoop::handleRead, this));
    wakeupChannel_->enableReading();
}

```

构造函数确保了 One Loop Per Thread 模型的约束，并初始化了 wakeupChannel。

### 2. 事件循环主体

```cpp
void EventLoop::loop()
{
    looping_ = true;
    quit_ = false;

    while (!quit_)
    {
        activeChannels_.clear();
        pollRetureTime_ = poller_->poll(kPollTimeMs, &activeChannels_);

        // 处理活跃事件
        for (Channel *channel : activeChannels_)
        {
            channel->handleEvent(pollRetureTime_);
        }

        // 处理待执行的回调函数
        doPendingFunctors();
    }

    looping_ = false;
}

```

事件循环的核心逻辑：

1. 通过 Poller 等待事件发生
2. 处理所有活跃的事件
3. 执行跨线程调度的回调函数

### 3. 线程间任务调度

```cpp
void EventLoop::runInLoop(Functor cb)
{
    if (isInLoopThread()) // 当前线程执行回调
    {
        cb();
    }
    else // 跨线程执行，需要唤醒目标EventLoop线程
    {
        queueInLoop(cb);
    }
}

void EventLoop::queueInLoop(Functor cb)
{
    {
        std::unique_lock<std::mutex> lock(mutex_);
        pendingFunctors_.emplace_back(cb);
    }

    // 唤醒条件：1.跨线程调用 2.当前正在执行回调（可能会产生新回调）
    if (!isInLoopThread() || callingPendingFunctors_)
    {
        wakeup();
    }
}

```

这两个方法实现了线程安全的任务调度机制，是 muduo 库中跨线程通信的关键。

### 4. 唤醒机制

```cpp
void EventLoop::wakeup()
{
    uint64_t one = 1;
    ssize_t n = write(wakeupFd_, &one, sizeof(one));
    // 错误处理...
}

void EventLoop::handleRead()
{
    uint64_t one = 1;
    ssize_t n = read(wakeupFd_, &one, sizeof(one));
    // 错误处理...
}

```

通过向 wakeupFd_ 写入数据，触发读事件，从而唤醒阻塞在 `poll` 调用上的 EventLoop 线程。

### 5. 回调执行

```cpp
void EventLoop::doPendingFunctors()
{
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;

    {
        std::unique_lock<std::mutex> lock(mutex_);
        functors.swap(pendingFunctors_); // 使用swap减小临界区
    }

    for (const Functor &functor : functors)
    {
        functor();
    }

    callingPendingFunctors_ = false;
}

```

通过 swap 技巧减小了互斥锁的临界区范围，提高了并发效率，同时避免了潜在的死锁问题。

## 设计亮点

1. **One Loop Per Thread 模式**：通过线程局部存储保证每个线程只有一个 EventLoop
2. **事件驱动**：基于 epoll 的事件驱动模式，高效处理大量连接
3. **跨线程唤醒**：使用 eventfd 实现无锁的线程间通信
4. **高效的回调处理**：使用 swap 技巧降低锁竞争，提高并发性能
5. **面向对象封装**：将底层的 I/O 操作封装为高级抽象，易于使用

这个 EventLoop 实现是整个网络库的核心，体现了高性能 C++ 服务器的关键设计原则。

# Channel.cc

```cpp
#include <sys/epoll.h>

#include <Channel.h>
#include <EventLoop.h>
#include <Logger.h>

const int Channel::kNoneEvent = 0; //空事件
const int Channel::kReadEvent = EPOLLIN | EPOLLPRI; //读事件
const int Channel::kWriteEvent = EPOLLOUT; //写事件

// EventLoop: ChannelList Poller
Channel::Channel(EventLoop *loop, int fd)
    : loop_(loop)
    , fd_(fd)
    , events_(0)
    , revents_(0)
    , index_(-1)
    , tied_(false)
{
}

Channel::~Channel()
{
}

// channel的tie方法什么时候调用过?  TcpConnection => channel
/**
 * TcpConnection中注册了Channel对应的回调函数，传入的回调函数均为TcpConnection
 * 对象的成员方法，因此可以说明一点就是：Channel的结束一定晚于TcpConnection对象！
 * 此处用tie去解决TcpConnection和Channel的生命周期时长问题，从而保证了Channel对象能够在
 * TcpConnection销毁前销毁。
 **/
void Channel::tie(const std::shared_ptr<void> &obj)
{
    tie_ = obj;
    tied_ = true;
}
//update 和remove => EpollPoller 更新channel在poller中的状态
/**
 * 当改变channel所表示的fd的events事件后，update负责再poller里面更改fd相应的事件epoll_ctl
 **/
void Channel::update()
{
    // 通过channel所属的eventloop，调用poller的相应方法，注册fd的events事件
    loop_->updateChannel(this);
}

// 在channel所属的EventLoop中把当前的channel删除掉
void Channel::remove()
{
    loop_->removeChannel(this);
}

void Channel::handleEvent(Timestamp receiveTime)
{
    if (tied_)
    {
        std::shared_ptr<void> guard = tie_.lock();
        if (guard)
        {
            handleEventWithGuard(receiveTime);
        }
        // 如果提升失败了 就不做任何处理 说明Channel的TcpConnection对象已经不存在了
    }
    else
    {
        handleEventWithGuard(receiveTime);
    }
}

void Channel::handleEventWithGuard(Timestamp receiveTime)
{
    LOG_INFO<<"channel handleEvent revents:"<<revents_;
    // 关闭
    if ((revents_ & EPOLLHUP) && !(revents_ & EPOLLIN)) // 当TcpConnection对应Channel 通过shutdown 关闭写端 epoll触发EPOLLHUP
    {
        if (closeCallback_)
        {
            closeCallback_();
        }
    }
    // 错误
    if (revents_ & EPOLLERR)
    {
        if (errorCallback_)
        {
            errorCallback_();
        }
    }
    // 读
    if (revents_ & (EPOLLIN | EPOLLPRI))
    {
        if (readCallback_)
        {
            readCallback_(receiveTime);
        }
    }
    // 写
    if (revents_ & EPOLLOUT)
    {
        if (writeCallback_)
        {
            writeCallback_();
        }
    }
}
```

[`Channel.cc`][Channel.cc](http://channel.cc/) ) 实现了 kama-webserver 的 Channel 类，是网络库的核心组件之一。Channel 封装了文件描述符及其感兴趣的事件，负责事件分发和回调处理。

## 核心概念

在网络库中，Channel 扮演的角色是：

- **事件源**：每个 Channel 对象负责一个文件描述符(fd)的事件
- **事件分发器**：根据发生的事件类型调用相应的回调函数
- **Reactor模式的处理器**：负责具体事件的回调处理

## 主要常量定义

```cpp
const int Channel::kNoneEvent = 0;                  // 空事件
const int Channel::kReadEvent = EPOLLIN | EPOLLPRI; // 读事件(普通数据+高优先级数据)
const int Channel::kWriteEvent = EPOLLOUT;          // 写事件

```

这些常量直接映射到 Linux epoll 的事件类型。

## 生命周期管理

```cpp
void Channel::tie(const std::shared_ptr<void> &obj)
{
    tie_ = obj;
    tied_ = true;
}

```

`tie` 方法解决了 Channel 与其所有者对象(如 TcpConnection)之间的生命周期依赖问题：

- 所有者通过智能指针传入自己
- Channel 保存为弱引用(`tie_`)
- 处理事件前，尝试提升弱引用，确保所有者仍然存活

## 事件注册与管理

```cpp
void Channel::update()
{
    loop_->updateChannel(this);
}

void Channel::remove()
{
    loop_->removeChannel(this);
}

```

这两个方法通过所属的 EventLoop 更新 Channel 在 Poller 中的状态，形成了 Channel → EventLoop → Poller 的调用链。

## 事件处理流程

事件处理分为两层：

### 1. 外层处理函数

```cpp
void Channel::handleEvent(Timestamp receiveTime)
{
    if (tied_)
    {
        std::shared_ptr<void> guard = tie_.lock();
        if (guard)
        {
            handleEventWithGuard(receiveTime);
        }
        // 如果提升失败，说明所有者已销毁，不做处理
    }
    else
    {
        handleEventWithGuard(receiveTime);
    }
}

```

这一层解决了对象生命周期问题，保证了回调的安全执行。

### 2. 内层处理函数

```cpp
void Channel::handleEventWithGuard(Timestamp receiveTime)
{
    // 按照优先级处理不同类型的事件
    // 1. 处理关闭事件
    if ((revents_ & EPOLLHUP) && !(revents_ & EPOLLIN))
    {
        if (closeCallback_) closeCallback_();
    }
    // 2. 处理错误事件
    if (revents_ & EPOLLERR)
    {
        if (errorCallback_) errorCallback_();
    }
    // 3. 处理读事件
    if (revents_ & (EPOLLIN | EPOLLPRI))
    {
        if (readCallback_) readCallback_(receiveTime);
    }
    // 4. 处理写事件
    if (revents_ & EPOLLOUT)
    {
        if (writeCallback_) writeCallback_();
    }
}

```

这一层根据实际发生的事件类型，按照特定顺序调用相应的回调函数。

## 设计亮点

1. **事件分离原则**：不同类型事件对应独立回调，符合单一职责原则
2. **生命周期保障**：通过 weak_ptr/shared_ptr 机制确保回调安全
3. **松耦合设计**：Channel 不直接操作 Poller，而是通过 EventLoop 间接交互
4. **优先级处理**：按照明确的顺序处理不同类型的事件
5. **精确日志**：记录处理的事件类型，便于调试

Channel 类是 Reactor 模式的核心组件，它将底层的文件描述符和事件机制封装成面向对象的接口，使得网络库的其他部分能够更加方便地处理 I/O 事件。

# Poller.cc

```cpp
#include <Poller.h>
#include <Channel.h>

Poller::Poller(EventLoop *loop)
    : ownerLoop_(loop)
{
}

bool Poller::hasChannel(Channel *channel) const
{
    auto it = channels_.find(channel->fd());
    return it != channels_.end() && it->second == channel;
}
```

[`Poller.cc`][Poller.cc](http://poller.cc/) ) 实现了 kama-webserver 网络库中 Poller 基类的部分功能。Poller 是整个网络库的 I/O 复用层抽象，是 Reactor 模式中的"事件分离器"。

## 核心功能

Poller 类作为一个抽象基类，定义了 I/O 复用器的通用接口。这个文件中实现了：

1. **构造函数**：
    
    ```cpp
    Poller::Poller(EventLoop *loop)
        : ownerLoop_(loop)
    {
    }
    
    ```
    
    - 每个 Poller 实例都从属于一个特定的 EventLoop
    - 构造时将 EventLoop 指针保存在 `ownerLoop_` 成员变量中
2. **hasChannel 方法**：
    
    ```cpp
    bool Poller::hasChannel(Channel *channel) const
    {
        auto it = channels_.find(channel->fd());
        return it != channels_.end() && it->second == channel;
    }
    
    ```
    
    - 检查指定的 Channel 是否已经注册到该 Poller 中
    - 首先通过 fd() 获取 Channel 关联的文件描述符
    - 然后在 channels_ 映射表中查找
    - 验证找到的 Channel 指针是否与参数一致，防止通过同一文件描述符注册了不同的 Channel

## 设计特点

1. **继承体系**：
    - Poller 是一个抽象基类，派生类（如 EPollPoller、PollPoller）提供具体实现
    - 这种设计使得 EventLoop 可以在不修改代码的情况下使用不同的 I/O 复用机制
2. **数据结构**：
    - `channels_` 是一个从文件描述符到 Channel 指针的映射表
    - 这种设计允许快速查找与特定文件描述符相关联的 Channel
3. **所有权管理**：
    - Poller 不拥有 Channel 对象，只是持有 Channel 的指针
    - Poller 从属于一个 EventLoop，形成了清晰的组件层次结构

虽然这个文件只实现了基本功能，但它是整个 I/O 复用层的基础，真正的 I/O 复用逻辑（如 epoll_wait、poll 调用）在具体的派生类（如 EPollPoller）中实现。

# Epollpoller.cc

```cpp
#include <errno.h>
#include <unistd.h>
#include <string.h>

#include <EPollPoller.h>
#include <Logger.h>
#include <Channel.h>

const int kNew = -1;    // 某个channel还没添加至Poller          // channel的成员index_初始化为-1
const int kAdded = 1;   // 某个channel已经添加至Poller
const int kDeleted = 2; // 某个channel已经从Poller删除

EPollPoller::EPollPoller(EventLoop *loop)
    : Poller(loop)
    , epollfd_(::epoll_create1(EPOLL_CLOEXEC)) 
    , events_(kInitEventListSize) // vector<epoll_event>(16)
{
    if (epollfd_ < 0)
    {
        LOG_FATAL<<"epoll_create error:%d \n"<<errno;
    }
}

EPollPoller::~EPollPoller()
{
    ::close(epollfd_);
}

Timestamp EPollPoller::poll(int timeoutMs, ChannelList *activeChannels)
{
    // 由于频繁调用poll 实际上应该用LOG_DEBUG输出日志更为合理 当遇到并发场景 关闭DEBUG日志提升效率
    LOG_INFO<<"fd total count:"<<channels_.size();

    int numEvents = ::epoll_wait(epollfd_, &*events_.begin(), static_cast<int>(events_.size()), timeoutMs);
    int saveErrno = errno;
    Timestamp now(Timestamp::now());

    if (numEvents > 0)
    {
        LOG_INFO<<"events happend"<<numEvents; // LOG_DEBUG最合理
        fillActiveChannels(numEvents, activeChannels);
        if (numEvents == events_.size()) // 扩容操作
        {
            events_.resize(events_.size() * 2);
        }
    }
    else if (numEvents == 0)
    {
        LOG_DEBUG<<"timeout!";
    }
    else
    {
        if (saveErrno != EINTR)
        {
            errno = saveErrno;
            LOG_ERROR<<"EPollPoller::poll() error!";
        }
    }
    return now;
}

// channel update remove => EventLoop updateChannel removeChannel => Poller updateChannel removeChannel
void EPollPoller::updateChannel(Channel *channel)
{
    const int index = channel->index();
    LOG_INFO<<"func =>"<<"fd"<<channel->fd()<<"events="<<channel->events()<<"index="<<index;

    if (index == kNew || index == kDeleted)
    {
        if (index == kNew)
        {
            int fd = channel->fd();
            channels_[fd] = channel;
        }
        else // index == kDeleted
        {
        }
        channel->set_index(kAdded);
        update(EPOLL_CTL_ADD, channel);
    }
    else // channel已经在Poller中注册过了
    {
        int fd = channel->fd();
        if (channel->isNoneEvent())
        {
            update(EPOLL_CTL_DEL, channel);
            channel->set_index(kDeleted);
        }
        else
        {
            update(EPOLL_CTL_MOD, channel);
        }
    }
}

// 从Poller中删除channel
void EPollPoller::removeChannel(Channel *channel)
{
    int fd = channel->fd();
    channels_.erase(fd);

    LOG_INFO<<"removeChannel fd="<<fd;

    int index = channel->index();
    if (index == kAdded)
    {
        update(EPOLL_CTL_DEL, channel);
    }
    channel->set_index(kNew);
}

// 填写活跃的连接
void EPollPoller::fillActiveChannels(int numEvents, ChannelList *activeChannels) const
{
    for (int i = 0; i < numEvents; ++i)
    {
        Channel *channel = static_cast<Channel *>(events_[i].data.ptr);
        channel->set_revents(events_[i].events);
        activeChannels->push_back(channel); // EventLoop就拿到了它的Poller给它返回的所有发生事件的channel列表了
    }
}

// 更新channel通道 其实就是调用epoll_ctl add/mod/del
void EPollPoller::update(int operation, Channel *channel)
{
    epoll_event event;
    ::memset(&event, 0, sizeof(event));

    int fd = channel->fd();

    event.events = channel->events();
    event.data.fd = fd;
    event.data.ptr = channel;

    if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
    {
        if (operation == EPOLL_CTL_DEL)
        {
            LOG_ERROR<<"epoll_ctl del error:"<<errno;
        }
        else
        {
            LOG_FATAL<<"epoll_ctl add/mod error:"<<errno;
        }
    }
}
```

[`EPollPoller.cc`][EPollPoller.cc](http://epollpoller.cc/) ) 实现了 kama-webserver 中的 EPollPoller 类，这是一个基于 Linux epoll 机制的 I/O 多路复用器，继承自 Poller 基类，是整个网络库中处理事件监听和分发的核心组件。

## Channel 状态常量

```cpp
const int kNew = -1;    // 某个channel还没添加至Poller
const int kAdded = 1;   // 某个channel已经添加至Poller
const int kDeleted = 2; // 某个channel已经从Poller删除

```

这些常量用于跟踪 Channel 在 Poller 中的状态，与 Channel 的 index_ 成员变量对应。

## 构造与析构

```cpp
EPollPoller::EPollPoller(EventLoop *loop)
    : Poller(loop)
    , epollfd_(::epoll_create1(EPOLL_CLOEXEC))
    , events_(kInitEventListSize) // vector<epoll_event>(16)
{
    if (epollfd_ < 0)
    {
        LOG_FATAL<<"epoll_create error:%d \\n"<<errno;
    }
}

EPollPoller::~EPollPoller()
{
    ::close(epollfd_);
}

```

构造函数创建 epoll 实例并初始化事件数组，析构函数负责关闭 epoll 文件描述符。通过 EPOLL_CLOEXEC 标志确保子进程 fork 后不会继承这个文件描述符。

## 核心方法：poll

```cpp
Timestamp EPollPoller::poll(int timeoutMs, ChannelList *activeChannels)
{
    // ... 日志记录 ...
    int numEvents = ::epoll_wait(epollfd_, &*events_.begin(),
                                static_cast<int>(events_.size()), timeoutMs);
    int saveErrno = errno;
    Timestamp now(Timestamp::now());

    if (numEvents > 0)
    {
        // ... 日志记录 ...
        fillActiveChannels(numEvents, activeChannels);
        if (numEvents == events_.size()) // 扩容操作
        {
            events_.resize(events_.size() * 2);
        }
    }
    // ... 错误处理 ...
    return now;
}

```

poll 方法是 EPollPoller 的核心，它：

1. 调用 epoll_wait 等待事件发生
2. 记录当前时间戳
3. 填充活跃通道列表
4. 需要时动态扩容事件数组
5. 处理超时和错误情况

## Channel 管理

### updateChannel

```cpp
void EPollPoller::updateChannel(Channel *channel)
{
    const int index = channel->index();
    // ... 日志记录 ...

    if (index == kNew || index == kDeleted)
    {
        if (index == kNew)
        {
            int fd = channel->fd();
            channels_[fd] = channel;
        }
        channel->set_index(kAdded);
        update(EPOLL_CTL_ADD, channel);
    }
    else // channel已经在Poller中注册过了
    {
        int fd = channel->fd();
        if (channel->isNoneEvent())
        {
            update(EPOLL_CTL_DEL, channel);
            channel->set_index(kDeleted);
        }
        else
        {
            update(EPOLL_CTL_MOD, channel);
        }
    }
}

```

这个方法处理 Channel 状态变更，根据 Channel 当前的状态（新建、已删除、已添加）和关注的事件，决定对 epoll 实例执行添加、修改或删除操作。

### removeChannel

```cpp
void EPollPoller::removeChannel(Channel *channel)
{
    int fd = channel->fd();
    channels_.erase(fd);
    // ... 日志记录 ...
    int index = channel->index();
    if (index == kAdded)
    {
        update(EPOLL_CTL_DEL, channel);
    }
    channel->set_index(kNew);
}

```

从 Poller 中完全移除一个 Channel，包括从内部的 channels_ 映射表中删除，并从 epoll 实例中注销对应的文件描述符。

## 辅助方法

### fillActiveChannels

```cpp
void EPollPoller::fillActiveChannels(int numEvents, ChannelList *activeChannels) const
{
    for (int i = 0; i < numEvents; ++i)
    {
        Channel *channel = static_cast<Channel *>(events_[i].data.ptr);
        channel->set_revents(events_[i].events);
        activeChannels->push_back(channel);
    }
}

```

这个方法收集发生事件的 Channel，并设置它们的 revents 成员，指示具体发生了什么事件。

### update

```cpp
void EPollPoller::update(int operation, Channel *channel)
{
    epoll_event event;
    ::memset(&event, 0, sizeof(event));
    // ... 设置事件属性 ...
    if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
    {
        // ... 错误处理 ...
    }
}

```

封装了对 epoll_ctl 的调用，用于向 epoll 实例添加、修改或删除监听事件。

## 设计亮点

1. **状态追踪**：通过 Channel 的 index_ 字段跟踪其在 Poller 中的状态
2. **自动扩容**：events_ 数组在需要时自动翻倍，适应高负载场景
3. **错误分级**：针对不同操作类型采用不同级别的错误日志
4. **回调机制**：使用 epoll_event 的 data.ptr 字段存储 Channel 指针，便于事件触发时快速找到对应的处理器
5. **资源管理**：使用 RAII 管理 epoll 文件描述符

EPollPoller 是整个网络库中的核心组件，它高效地利用了 Linux 的 epoll 机制，为服务器提供了高性能的 I/O 多路复用能力，是 Reactor 模式的典型实现。

# 日志系统

![image.webp](image%201.webp)

服务器的⽇志系统是⼀个多⽣产者，单消费者的任务场景：多⽣产者负责把⽇志写⼊缓冲区，单消费者负责把缓冲 区中数据写⼊⽂件。如果只⽤⼀个缓冲区，不光要同步各个⽣产者,还要同步⽣产者和消费者。⽽且最重要的是需要 保证⽣产者与消费者的并发，也就是前端不断写⽇志到缓冲区的同时，后端可以把缓冲区写⼊⽂件。

LOG 的实现参照了 muduo，但是⽐ muduo 要简化⼀点，⼤致的实现如上图所示，看图还是⽐较好懂的。

● ⾸先是 Logger 类， Logger 类⾥⾯有 Impl 类，其实具体实现是 Impl 类，我也不懂muduo为何要再封装⼀ 层，那么我们来说说 Impl ⼲了什么，在初始化的时候 Impl 会把时间信息存到 LogStream 的缓冲区⾥，在 我们实际⽤ Log 的时候，实际写⼊的缓冲区也是 LogStream ，在析构的时候 Impl 会把当前⽂件和⾏数等信 息写⼊到 LogStream ，再把 LogStream ⾥的内容写到 AsyncLogging 的缓冲区中，当然这时候我们要先 开启⼀个后端线程⽤于把缓冲区的信息写到⽂件⾥。

● LogStream 类，⾥⾯其实就⼀个 Buffer 缓冲区，是⽤来暂时存放我们写⼊的信息的。还有就是重载运算 符，因为我们采⽤的是 C++ 的流式⻛格。

● AsyncLogging 类，最核⼼的部分，在多线程程序中写 Log ⽆⾮就是前端往后端写，后端往硬盘写，⾸先将 LogStream 的内容写到了 AsyncLogging 缓冲区⾥，也就是前端往后端写，这个过程通过 append 函数实 现，后端实现通过 threadfunc 函数，两个线程的同步和等待通过互斥锁和条件变量来实现，具体实现使⽤了 双缓冲技术。

● 双缓冲技术的基本思路：准备两块 buffer，A 和 B,前端往 A 写数据，后端从 B ⾥⾯往硬盘写数据，当 A 写满 后，交换 A 和 B，如此反复。使⽤两个 buffer 的好处是在新建⽇志消息的时候不必等待磁盘⽂件操作，也避 免每条新⽇志消息都触发后端⽇志线程。换句话说，前端不是将⼀条条⽇志消息分别送给后端，⽽是将多条⽇ 志消息拼接成⼀个⼤的 buffer 传送给后端，相当于批处理，减少了线程唤醒的开销。不过实际的实现的话和 这个还是有点区别，具体看代码吧。

我们⾸先看下 logStream 类的实现， logStream 类的主要作⽤是将各个类型的数据转换为 char 的形式放⼊字符 数组中（也就是前端⽇志写⼊ Buffer A 的这个过程），⽅便后端线程写⼊硬盘。

# **Logstream.cc**

```cpp
#include "LogStream.h"
#include <algorithm>

static const char digits[] = "9876543210123456789";

template <typename T>
void LogStream::formatInteger(T num)
{
    if (buffer_.avail() >= kMaxNumberSize)
    {
        char *start = buffer_.current();
        char *cur = start;
        static const char *zero = digits + 9;
        bool negative = (num < 0); // 判断num是否为负数
        do
        {
            int remainder = static_cast<int>(num % 10);
            (*cur++) = zero[remainder];
            num /= 10;
        } while (num != 0);
        if (negative)
        {
            *cur++ = '-';
        }
        *cur = '\0';
        std::reverse(start, cur);
        int length = static_cast<int>(cur - start);
        buffer_.add(length);
    }
}
// 重载输出流运算符<<，用于将布尔值写入缓冲区
LogStream &LogStream::operator<<(bool express) {
    buffer_.append(express ? "true" : "false", express ? 4 : 5);
    return *this;
}

// 重载输出流运算符<<，用于将短整型写入缓冲区
LogStream &LogStream::operator<<(short number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将无符号短整型写入缓冲区
LogStream &LogStream::operator<<(unsigned short number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将整型写入缓冲区
LogStream &LogStream::operator<<(int number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将无符号整型写入缓冲区
LogStream &LogStream::operator<<(unsigned int number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将长整型写入缓冲区
LogStream &LogStream::operator<<(long number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将无符号长整型写入缓冲区
LogStream &LogStream::operator<<(unsigned long number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将长长整型写入缓冲区
LogStream &LogStream::operator<<(long long number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将无符号长长整型写入缓冲区
LogStream &LogStream::operator<<(unsigned long long number) {
    formatInteger(number);
    return *this;
}

// 重载输出流运算符<<，用于将浮点数写入缓冲区
LogStream &LogStream::operator<<(float number) {
    *this<<static_cast<double>(number);
    return *this;
}

// 重载输出流运算符<<，用于将双精度浮点数写入缓冲区
LogStream &LogStream::operator<<(double number) {
    char buffer[32];
    snprintf(buffer, sizeof(buffer), "%.12g", number);
    buffer_.append(buffer, strlen(buffer));
    return *this;
}

// 重载输出流运算符<<，用于将字符写入缓冲区
LogStream &LogStream::operator<<(char str) {
    buffer_.append(&str, 1);
    return *this;
}

// 重载输出流运算符<<，用于将C风格字符串写入缓冲区
LogStream &LogStream::operator<<(const char *str) {
    buffer_.append(str, strlen(str));
    return *this;
}

// 重载输出流运算符<<，用于将无符号字符指针写入缓冲区
LogStream &LogStream::operator<<(const unsigned char *str) {
    buffer_.append(reinterpret_cast<const char*>(str), strlen(reinterpret_cast<const char*>(str)));
    return *this;
}

// 重载输出流运算符<<，用于将std::string对象写入缓冲区
LogStream &LogStream::operator<<(const std::string &str) {
    buffer_.append(str.c_str(), str.size());
    return *this;
}

LogStream& LogStream::operator<<(const GeneralTemplate& g)
{
    buffer_.append(g.data_, g.len_);
    return *this;
}
```

[`LogStream.cc`][LogStream.cc](http://logstream.cc/) ) 实现了 kama-webserver 日志系统的流式接口，提供类似于 C++ 标准库 `iostream` 的使用方式，但针对日志记录进行了性能优化。

## 核心功能

### 1. 整数格式化函数

```cpp
template <typename T>
void LogStream::formatInteger(T num)
{
    // 检查缓冲区空间是否足够
    if (buffer_.avail() >= kMaxNumberSize)
    {
        char *start = buffer_.current();
        char *cur = start;
        static const char *zero = digits + 9; // 巧妙地定位到数字 '0'
        bool negative = (num < 0); // 处理负数

        // 从低位到高位处理数字
        do {
            int remainder = static_cast<int>(num % 10);
            (*cur++) = zero[remainder]; // 利用偏移获取字符
            num /= 10;
        } while (num != 0);

        // 处理负号
        if (negative) {
            *cur++ = '-';
        }

        *cur = '\\0';
        std::reverse(start, cur); // 反转得到正确顺序
        buffer_.add(static_cast<int>(cur - start)); // 更新缓冲区位置
    }
}

```

这个模板函数实现了高效的整数到字符串的转换：

- 不依赖 `sprintf` 等函数，减少了系统调用开销
- 使用预定义的字符表 `digits` 巧妙处理数字字符转换
- 通过反转字符串得到最终结果

### 2. 运算符重载

文件中重载了大量的 `<<` 运算符，支持不同数据类型的输出：

```cpp
LogStream &LogStream::operator<<(bool express) {
    buffer_.append(express ? "true" : "false", express ? 4 : 5);
    return *this;
}

LogStream &LogStream::operator<<(int number) {
    formatInteger(number);
    return *this;
}

LogStream &LogStream::operator<<(double number) {
    char buffer[32];
    snprintf(buffer, sizeof(buffer), "%.12g", number);
    buffer_.append(buffer, strlen(buffer));
    return *this;
}

LogStream &LogStream::operator<<(const std::string &str) {
    buffer_.append(str.c_str(), str.size());
    return *this;
}
// ... 其他类型的重载

```

这些重载使得 LogStream 能够像 std::ostream 一样使用，支持各种类型的输出。

## 设计特点

### 1. 高效的缓冲区管理

- 使用定制的 buffer_ 对象而不是 std::stringstream
- 直接操作字符缓冲区，减少内存分配和复制

### 2. 自定义数值转换

- 整数转换不依赖标准库函数，减少系统调用开销
- 浮点数使用 snprintf 限定精度为 12 位有效数字

### 3. 链式调用支持

- 所有运算符重载都返回对象自身的引用，支持链式调用
- 例如：`log << "Value: " << 42 << " is " << true;`

### 4. 通用模板支持

```cpp
LogStream& LogStream::operator<<(const GeneralTemplate& g)
{
    buffer_.append(g.data_, g.len_);
    return *this;
}

```

这允许通过 GeneralTemplate 类型包装特定格式的数据进行输出。

## 应用场景

这个 LogStream 类是高性能日志系统的基础组件，它使得：

1. 日志记录具有类似 C++ 标准流的便捷语法
2. 避免了标准库 iostream 的性能开销
3. 优化了数值到字符串的转换过程
4. 支持各种数据类型的高效输出

通过这些设计，kama-webserver 的日志系统能在保持易用性的同时提供高性能的日志记录功能。

# AsyncLogging.cc

```cpp
#include "AsyncLogging.h"
#include <stdio.h>
AsyncLogging::AsyncLogging(const std::string &basename, off_t rollSize, int flushInterval)
    :
      flushInterval_(flushInterval),
      running_(false),
      basename_(basename),
      rollSize_(rollSize),
      thread_(std::bind(&AsyncLogging::threadFunc, this), "Logging"),
      mutex_(),
      cond_(),
      currentBuffer_(new LargeBuffer),
      nextBuffer_(new LargeBuffer),
      buffers_()
{
    currentBuffer_->bzero();
    nextBuffer_->bzero();
    buffers_.reserve(16); // 只维持队列长度2~16.
}
// 调用此函数解决前端把LOG_XXX<<"..."传递给后端，后端再将日志消息写入日志文件
void AsyncLogging::append(const char *logline, int len)
{
    std::lock_guard<std::mutex> lg(mutex_);
    // 缓冲区剩余的空间足够写入
    if (currentBuffer_->avail() > static_cast<size_t>(len))
    {
        currentBuffer_->append(logline, len);
    }
    else
    {
        buffers_.push_back(std::move(currentBuffer_));

        if (nextBuffer_)
        {
            currentBuffer_ = std::move(nextBuffer_);
        }
        else
        {
            currentBuffer_.reset(new LargeBuffer);
        }
        currentBuffer_->append(logline, len);
        // 唤醒后端线程写入磁盘
        cond_.notify_one();
    }
}

void AsyncLogging::threadFunc()
{
    // output写入磁盘接口
    LogFile output(basename_, rollSize_);
    BufferPtr newbuffer1(new LargeBuffer); // 生成新buffer替换currentbuffer_
    BufferPtr newbuffer2(new LargeBuffer); // 生成新buffer2替换newBuffer_，其目的是为了防止后端缓冲区全满前端无法写入
    newbuffer1->bzero();
    newbuffer2->bzero();
    // 缓冲区数组置为16个，用于和前端缓冲区数组进行交换
    BufferVector buffersToWrite;
    buffersToWrite.reserve(16);
    while (running_)
    {
        {
            // 互斥锁保护这样就保证了其他前端线程无法向前端buffer写入数据
            std::unique_lock<std::mutex> lg(mutex_);
            if (buffers_.empty())
            {
                cond_.wait_for(lg, std::chrono::seconds(3));
            }
            buffers_.push_back(std::move(currentBuffer_));
            currentBuffer_ = std::move(newbuffer1);
            if (!nextBuffer_)
            {
                nextBuffer_ = std::move(newbuffer2);
            }
            buffersToWrite.swap(buffers_);
        }
        // 从待写缓冲区取出数据通过LogFile提供的接口写入到磁盘中
        for (auto &buffer : buffersToWrite)
        {
            output.append(buffer->data(), buffer->length());
        }

        if (buffersToWrite.size() > 2)
        {
            buffersToWrite.resize(2);
        }

        if (!newbuffer1)
        {
            newbuffer1 = std::move(buffersToWrite.back());
            buffersToWrite.pop_back();
            newbuffer1->reset();
        }
        if (!newbuffer2)
        {
            newbuffer2 = std::move(buffersToWrite.back());
            buffersToWrite.pop_back();
            newbuffer2->reset();
        }
        buffersToWrite.clear(); // 清空后端缓冲队列
        output.flush();         // 清空文件夹缓冲区
    }
    output.flush(); // 确保一定清空。
}
```

[`AsyncLogging.cc`][AsyncLogging.cc](http://asynclogging.cc/) ) 实现了 kama-webserver 的异步日志系统，采用**双缓冲技术**设计，通过将日志写入操作从主线程分离到专用的日志线程，大幅提升了服务器的性能和响应能力。

## 核心设计理念

异步日志系统基于**生产者-消费者模型**：

- **前端（生产者）**：应用线程生成日志
- **后端（消费者）**：专用日志线程将日志写入磁盘
- **双缓冲区**：减少线程间的竞争和等待

## 关键数据结构

```cpp
BufferPtr currentBuffer_;  // 当前正在写入的缓冲区
BufferPtr nextBuffer_;     // 备用缓冲区
BufferVector buffers_;     // 待写入磁盘的已满缓冲区

```

## 主要功能实现

### 1. 构造函数

```cpp
AsyncLogging::AsyncLogging(const std::string &basename, off_t rollSize, int flushInterval)
    : flushInterval_(flushInterval),
      running_(false),
      basename_(basename),
      rollSize_(rollSize),
      thread_(std::bind(&AsyncLogging::threadFunc, this), "Logging"),
      // ...其他初始化
{
    currentBuffer_->bzero();
    nextBuffer_->bzero();
    buffers_.reserve(16); // 预留16个缓冲区的容量
}

```

构造函数初始化了：

- 日志文件信息（文件名、滚动大小）
- 前端缓冲区（currentBuffer_和nextBuffer_）
- 后端日志线程

### 2. append 方法（前端接口）

```cpp
void AsyncLogging::append(const char *logline, int len)
{
    std::lock_guard<std::mutex> lg(mutex_);

    if (currentBuffer_->avail() > static_cast<size_t>(len)) {
        // 缓冲区空间充足，直接写入
        currentBuffer_->append(logline, len);
    } else {
        // 当前缓冲区已满，将其移入待写入队列
        buffers_.push_back(std::move(currentBuffer_));

        // 切换到备用缓冲区或创建新缓冲区
        if (nextBuffer_) {
            currentBuffer_ = std::move(nextBuffer_);
        } else {
            currentBuffer_.reset(new LargeBuffer);
        }

        // 写入新缓冲区并唤醒后端线程
        currentBuffer_->append(logline, len);
        cond_.notify_one();
    }
}

```

这个方法是应用线程调用的接口，它：

1. 加锁保护共享数据
2. 尝试将日志写入当前缓冲区
3. 如果缓冲区已满，将其加入待写入队列，并切换到新缓冲区
4. 唤醒后端线程处理已满的缓冲区

### 3. threadFunc 方法（后端处理）

```cpp
void AsyncLogging::threadFunc()
{
    LogFile output(basename_, rollSize_);
    BufferPtr newbuffer1(new LargeBuffer);
    BufferPtr newbuffer2(new LargeBuffer);
    // ...初始化

    while (running_) {
        {
            std::unique_lock<std::mutex> lg(mutex_);

            // 等待条件变量或超时（最多3秒）
            if (buffers_.empty()) {
                cond_.wait_for(lg, std::chrono::seconds(3));
            }

            // 交换缓冲区，最小化锁的持有时间
            buffers_.push_back(std::move(currentBuffer_));
            currentBuffer_ = std::move(newbuffer1);
            // ...替换缓冲区
            buffersToWrite.swap(buffers_);
        }

        // 锁外执行实际的文件写入操作
        for (auto &buffer : buffersToWrite) {
            output.append(buffer->data(), buffer->length());
        }

        // 缓冲区复用逻辑
        if (buffersToWrite.size() > 2) {
            buffersToWrite.resize(2);
        }
        // ...复用缓冲区

        output.flush();  // 清空文件缓冲区
    }
}

```

后端线程的主要任务：

1. 等待前端通知或定时超时（确保日志及时写入）
2. 与前端交换缓冲区，获取待写入数据
3. 在锁外将数据批量写入磁盘
4. 复用缓冲区以减少内存分配

## 性能优化设计

1. **最小临界区**：只在交换缓冲区时加锁，写入文件操作在锁外进行
2. **批量写入**：一次性写入多个缓冲区内容，减少I/O操作次数
3. **缓冲区复用**：保留旧缓冲区并重用，减少内存分配和释放的开销
4. **定时刷新**：即使缓冲区未满，也会定期将日志写入磁盘（最多3秒）
5. **双缓冲技术**：前端能够在后端写入时继续生成日志，不会相互阻塞

这种异步日志设计是高性能服务器的关键组件，确保了日志系统不会成为服务器性能的瓶颈。

# Logger.cc

```cpp
#include "Logger.h"
#include "CurrentThread.h"

namespace ThreadInfo
{
    thread_local char t_errnobuf[512]; // 每个线程独立的错误信息缓冲
    thread_local char t_timer[64];     // 每个线程独立的时间格式化缓冲区
    thread_local time_t t_lastSecond;  // 每个线程记录上次格式化的时间

}
const char *getErrnoMsg(int savedErrno)
{
    return strerror_r(savedErrno, ThreadInfo::t_errnobuf, sizeof(ThreadInfo::t_errnobuf));
}
// 根据Level 返回level_名字
const char *getLevelName[Logger::LogLevel::LEVEL_COUNT]{
    "TRACE ",
    "DEBUG ",
    "INFO  ",
    "WARN  ",
    "ERROR ",
    "FATAL ",
};
/**
 * 默认的日志输出函数
 * 将日志内容写入标准输出流(stdout)
 * @param data 要输出的日志数据
 * @param len 日志数据的长度W
 */
static void defaultOutput(const char *data, int len)
{
    fwrite(data, len, sizeof(char), stdout);
}

/**
 * 默认的刷新函数
 * 刷新标准输出流的缓冲区,确保日志及时输出
 * 在发生错误或需要立即看到日志时会被调用
 */
static void defaultFlush()
{
    fflush(stdout);
}
Logger::OutputFunc g_output = defaultOutput;
Logger::FlushFunc g_flush = defaultFlush;

Logger::Impl::Impl(Logger::LogLevel level, int savedErrno, const char *filename, int line)
    : time_(Timestamp::now()),
      stream_(),
      level_(level),
      line_(line),
      basename_(filename)
{
    // 根据时区格式化当前时间字符串, 也是一条log消息的开头
    formatTime();
    // 写入日志等级
    stream_ << GeneralTemplate(getLevelName[level], 6);
    if (savedErrno != 0)
    {
        stream_ << getErrnoMsg(savedErrno) << " (errno=" << savedErrno << ") ";
    }
}
// 根据时区格式化当前时间字符串, 也是一条log消息的开头
void Logger::Impl::formatTime()
{
    Timestamp now = Timestamp::now();
    //计算秒数
    time_t seconds = static_cast<time_t>(now.microSecondsSinceEpoch() / Timestamp::kMicroSecondsPerSecond);
    int microseconds = static_cast<int>(now.microSecondsSinceEpoch() % Timestamp::kMicroSecondsPerSecond);
    //计算剩余微秒数
    struct tm *tm_timer = localtime(&seconds);
    // 写入此线程存储的时间buf中
    snprintf(ThreadInfo::t_timer, sizeof(ThreadInfo::t_timer), "%4d/%02d/%02d %02d:%02d:%02d",
             tm_timer->tm_year + 1900,
             tm_timer->tm_mon + 1,
             tm_timer->tm_mday,
             tm_timer->tm_hour,
             tm_timer->tm_min,
             tm_timer->tm_sec);
    // 更新最后一次时间调用
    ThreadInfo::t_lastSecond = seconds;

    // muduo使用Fmt格式化整数，这里我们直接写入buf
    char buf[32] = {0};
    snprintf(buf, sizeof(buf), "%06d ", microseconds);

    
    stream_ << GeneralTemplate(ThreadInfo::t_timer, 17) << GeneralTemplate(buf, 7);
}
void Logger::Impl::finish()
{
    stream_ << " - " << GeneralTemplate(basename_.data_, basename_.size_)
            << ':' << line_ << '\n';
}
Logger::Logger(const char *filename, int line, LogLevel level) : impl_(level, 0, filename, line)
{
}
Logger::~Logger()
{
    impl_.finish();
    const LogStream::Buffer &buffer = stream().buffer();
    // 输出(默认项终端输出)
    g_output(buffer.data(), buffer.length());
    // FATAL情况终止程序
    if (impl_.level_ == FATAL)
    {
        g_flush();
        abort();
    }
}

void Logger::setOutput(OutputFunc out)
{
    g_output = out;
}

void Logger::setFlush(FlushFunc flush)
{
    g_flush = flush;
}
```

[`Logger.cc`][Logger.cc](http://logger.cc/) ) 实现了 kama-webserver 的核心日志系统，提供了一个功能完备且易于使用的日志记录机制，是整个系统可靠运行和调试的重要支撑。

## 线程安全设计

```cpp
namespace ThreadInfo
{
    thread_local char t_errnobuf[512]; // 每个线程独立的错误信息缓冲
    thread_local char t_timer[64];     // 每个线程独立的时间格式化缓冲区
    thread_local time_t t_lastSecond;  // 每个线程记录上次格式化的时间
}

```

使用 `thread_local` 关键字为每个线程创建独立的缓冲区，避免多线程竞争问题，同时提高了性能。

## 日志级别

```cpp
const char *getLevelName[Logger::LogLevel::LEVEL_COUNT]{
    "TRACE ", "DEBUG ", "INFO  ", "WARN  ", "ERROR ", "FATAL ",
};

```

定义了六个日志级别，从低到高依次为：TRACE、DEBUG、INFO、WARN、ERROR、FATAL。其中 FATAL 级别会导致程序终止。

## 可定制的输出机制

```cpp
static void defaultOutput(const char *data, int len)
{
    fwrite(data, len, sizeof(char), stdout);
}

static void defaultFlush()
{
    fflush(stdout);
}

Logger::OutputFunc g_output = defaultOutput;
Logger::FlushFunc g_flush = defaultFlush;

```

默认情况下日志输出到标准输出，但系统提供了设置自定义输出和刷新函数的接口，使得日志输出目标可以灵活配置：

```cpp
void Logger::setOutput(OutputFunc out)
{
    g_output = out;
}

void Logger::setFlush(FlushFunc flush)
{
    g_flush = flush;
}

```

## 日志格式与构建

### 日志消息构造

```cpp
Logger::Impl::Impl(Logger::LogLevel level, int savedErrno, const char *filename, int line)
    : time_(Timestamp::now()),
      stream_(),
      level_(level),
      line_(line),
      basename_(filename)
{
    formatTime();
    stream_ << GeneralTemplate(getLevelName[level], 6);
    if (savedErrno != 0)
    {
        stream_ << getErrnoMsg(savedErrno) << " (errno=" << savedErrno << ") ";
    }
}

```

每条日志消息包含：

- 时间戳（年/月/日 时:分:秒.微秒）
- 日志级别（TRACE/DEBUG/INFO/WARN/ERROR/FATAL）
- 错误信息（如果有）
- 源文件名和行号
- 实际日志内容

### 高效的时间格式化

```cpp
void Logger::Impl::formatTime()
{
    Timestamp now = Timestamp::now();
    // ... 计算时间和格式化 ...
    stream_ << GeneralTemplate(ThreadInfo::t_timer, 17) << GeneralTemplate(buf, 7);
}

```

通过自定义的 GeneralTemplate 类和线程局部缓冲区，优化了时间格式化性能。

## RAII 日志输出

```cpp
Logger::~Logger()
{
    impl_.finish();
    const LogStream::Buffer &buffer = stream().buffer();
    g_output(buffer.data(), buffer.length());
    if (impl_.level_ == FATAL)
    {
        g_flush();
        abort();
    }
}

```

Logger 利用 C++ 的 RAII 特性，在析构时自动输出日志内容，使日志使用更加简洁：

```cpp
// 使用示例
LOG_INFO << "Connection established with " << client_addr;
// 无需显式调用输出函数，变量生命周期结束时自动输出

```

当日志级别为 FATAL 时，会自动刷新缓冲区并调用 `abort()` 终止程序。

## 错误信息处理

```cpp
const char *getErrnoMsg(int savedErrno)
{
    return strerror_r(savedErrno, ThreadInfo::t_errnobuf, sizeof(ThreadInfo::t_errnobuf));
}

```

使用线程安全的 `strerror_r` 函数获取错误码对应的文本描述，便于记录系统错误。

这个日志系统结合了高效性、线程安全性和易用性，是 kama-webserver 稳定运行的重要基础设施。

# 内存池

内存池中哈希桶的思想借鉴了STL allocator。

![image.webp](image%202.webp)

主要框架如上图所示，主要就是维护⼀个哈希桶 MemoryPools ，⾥⾯每项对应⼀个内存池 MemoryPool ，哈希桶 中每个内存池的块⼤⼩ BlockSize 是相同的（4096字节，当然也可以设置为不同的），但是每个内存池⾥每个块 分割的⼤⼩（槽⼤⼩） SlotSize 是不同的，依次为8,16,32,...,512字节（需要的内存超过512字节就 ⽤ new/malloc ），这样设置的好处是可以保证内存碎⽚在可控范围内。

![image.webp](image%203.webp)

内存池的内部结构如上图所示，主要的对象有：指向第⼀个可⽤内存块的指针 Slot currentBlock （也是图中的 ptr to firstBlock ，图⽚已经上传懒得改了），被释放对象的slot链表 Slot freeSlot ，未使⽤的slot链 表 Slot* currentSlot ，下⾯讲下具体的作⽤：

●Slot currentBlock ：内存池实际上是⼀个⼀个的 Block 以链表的形式连接起来，每⼀个 Block 是⼀块 ⼤的内存，当内存池的内存不⾜的时候，就会向操作系统申请新的 block 加⼊链表。

●Slot freeSlot ：链表⾥⾯的每⼀项都是对象被释放后归还给内存池的空间，内存池刚创建时 freeSlot 是空的。⽤户创建对象，将对象释放时，把内存归还给内存池，此时内存池不会将内存归还给系统 （ delete/free ），⽽是把指向这个对象的内存的指针加到 freeSlot 链表的前⾯（前插），之后⽤户每次 申请内存时， memoryPool 就先在这个 freeSlot 链表⾥⾯找。

●Slot curretSlot ：⽤户在创建对象的时候，先检查 freeSlot 是否为空，不为空的时候直接取出⼀项作 为分配出的空间。否则就在当前 Block 将 currentSlot 所指的内存分配出去，如果 Block ⾥⾯的内存已 经使⽤完，就向操作系统申请⼀个新的 Block 。

# 线程池

![image.webp](image%204.webp)

借鉴 muduo 中 one loop per thread 的思想，每个线程与⼀个 EventLoop 对应，设计了 EventLoopThread 类 封装了 thread 和 EventLoop，然后在 EventLoopThreadPool 类中根据需要的线程数来创建 EventLoopThread ，最后根据 Round Robin 来选择下⼀个 EventLoop，实现负载均衡。

由此我们先对 EventLoop 进⾏封装，让每个 Thread 中运⾏ EventLoop 的 Loop。

# **EventLoopThread.cc**

```cpp
#include <EventLoopThread.h>
#include <EventLoop.h>

EventLoopThread::EventLoopThread(const ThreadInitCallback &cb,
                                 const std::string &name)
    : loop_(nullptr)
    , exiting_(false)
    , thread_(std::bind(&EventLoopThread::threadFunc, this), name)
    , mutex_()
    , cond_()
    , callback_(cb)
{
}

EventLoopThread::~EventLoopThread()
{
    exiting_ = true;
    if (loop_ != nullptr)
    {
        loop_->quit();
        thread_.join();
    }
}

EventLoop *EventLoopThread::startLoop()
{
    thread_.start(); // 启用底层线程Thread类对象thread_中通过start()创建的线程

    EventLoop *loop = nullptr;
    {
        std::unique_lock<std::mutex> lock(mutex_);
        cond_.wait(lock, [this](){return loop_ != nullptr;});
        loop = loop_;
    }
    return loop;
}

// 下面这个方法 是在单独的新线程里运行的
void EventLoopThread::threadFunc()
{
    EventLoop loop; // 创建一个独立的EventLoop对象 和上面的线程是一一对应的 级one loop per thread

    if (callback_)
    {
        callback_(&loop);
    }

    {
        std::unique_lock<std::mutex> lock(mutex_);
        loop_ = &loop;
        cond_.notify_one();
    }
    loop.loop();    // 执行EventLoop的loop() 开启了底层的Poller的poll()
    std::unique_lock<std::mutex> lock(mutex_);
    loop_ = nullptr;
}
```

[`EventLoopThread.cc`][EventLoopThread.cc](http://eventloopthread.cc/) ) 实现了 kama-webserver 网络库中的 EventLoopThread 类，该类负责创建一个独立线程并在其中运行事件循环(EventLoop)，是实现"one loop per thread"模型的关键组件。

## 核心功能

EventLoopThread 类封装了创建和管理事件循环线程的所有细节，确保：

1. 线程安全地创建 EventLoop 对象
2. 线程与 EventLoop 一一对应
3. 优雅地启动和停止事件循环

## 主要成员

```cpp
EventLoop *loop_;          // 指向线程创建的EventLoop对象
bool exiting_;             // 线程退出标志
Thread thread_;            // 底层线程对象
std::mutex mutex_;         // 互斥锁，保护共享数据
std::condition_variable cond_; // 条件变量，用于线程同步
ThreadInitCallback callback_; // 初始化回调，在EventLoop创建后调用

```

## 关键方法

### 1. 构造函数与析构函数

```cpp
EventLoopThread::EventLoopThread(const ThreadInitCallback &cb, const std::string &name)
    : loop_(nullptr)
    , exiting_(false)
    , thread_(std::bind(&EventLoopThread::threadFunc, this), name)
    , mutex_()
    , cond_()
    , callback_(cb)
{
}

EventLoopThread::~EventLoopThread()
{
    exiting_ = true;
    if (loop_ != nullptr)
    {
        loop_->quit();
        thread_.join();
    }
}

```

构造函数绑定线程函数，析构函数确保线程安全退出。

### 2. startLoop 方法

```cpp
EventLoop *EventLoopThread::startLoop()
{
    thread_.start(); // 启动新线程

    EventLoop *loop = nullptr;
    {
        std::unique_lock<std::mutex> lock(mutex_);
        cond_.wait(lock, [this](){return loop_ != nullptr;}); // 等待EventLoop创建完成
        loop = loop_;
    }
    return loop;
}

```

这个方法启动线程并等待 EventLoop 创建完成后返回其指针，这种设计保证了:

- 调用者可以立即获得可用的 EventLoop 对象
- 线程创建和 EventLoop 初始化的过程对调用者透明
- 线程间同步使用条件变量，避免忙等待

### 3. threadFunc 方法

```cpp
void EventLoopThread::threadFunc()
{
    EventLoop loop; // 在线程内创建EventLoop对象

    if (callback_)
    {
        callback_(&loop); // 执行初始化回调
    }

    {
        std::unique_lock<std::mutex> lock(mutex_);
        loop_ = &loop;
        cond_.notify_one(); // 通知startLoop方法EventLoop已创建
    }

    loop.loop(); // 启动事件循环，不会返回，直到quit被调用

    // 退出循环后清理
    std::unique_lock<std::mutex> lock(mutex_);
    loop_ = nullptr;
}

```

这是新线程的入口函数，它负责:

1. 创建 EventLoop 对象（体现"one loop per thread"原则）
2. 执行初始化回调（可选）
3. 通知主线程 EventLoop 已创建完成
4. 运行事件循环，直到退出
5. 清理资源

## 设计特点

1. **线程安全设计**：使用互斥锁和条件变量保证跨线程操作的安全性
2. **资源生命周期管理**：EventLoop 对象在线程内创建和销毁，遵循 RAII 原则
3. **灵活的初始化机制**：支持自定义初始化回调，便于配置 EventLoop
4. **清晰的所有权模型**：EventLoop 归属于创建它的线程，外部通过指针访问

EventLoopThread 是实现多线程网络服务器的基础组件，在 kama-webserver 中主要用于创建 IO 线程池中的工作线程，每个线程独立处理一部分网络连接，从而提高并发处理能力。

# Thread.cc

```cpp
#include <Thread.h>
#include <CurrentThread.h>

#include <semaphore.h>

std::atomic_int Thread::numCreated_(0);

Thread::Thread(ThreadFunc func, const std::string &name)
    : started_(false)
    , joined_(false)
    , tid_(0)
    , func_(std::move(func))
    , name_(name)
{
    setDefaultName();
}

Thread::~Thread()
{
    if (started_ && !joined_)
    {
        thread_->detach();                                                  // thread类提供了设置分离线程的方法 线程运行后自动销毁（非阻塞）
    }
}

void Thread::start()                                                        // 一个Thread对象 记录的就是一个新线程的详细信息
{
    started_ = true;
    sem_t sem;
    sem_init(&sem, false, 0);                                               // false指的是 不设置进程间共享
    // 开启线程
    thread_ = std::shared_ptr<std::thread>(new std::thread([&]() {
        tid_ = CurrentThread::tid();                                        // 获取线程的tid值
        sem_post(&sem);
        func_();                                                            // 开启一个新线程 专门执行该线程函数
    }));

    // 这里必须等待获取上面新创建的线程的tid值
    sem_wait(&sem);
}

// C++ std::thread 中join()和detach()的区别：https://blog.nowcoder.net/n/8fcd9bb6e2e94d9596cf0a45c8e5858a
void Thread::join()
{
    joined_ = true;
    thread_->join();
}

void Thread::setDefaultName()
{
    int num = ++numCreated_;
    if (name_.empty())
    {
        char buf[32] = {0};
        snprintf(buf, sizeof buf, "Thread%d", num);
        name_ = buf;
    }
}
```

[`Thread.cc`][Thread.cc](http://thread.cc/) ) 实现了 kama-webserver 的线程管理类，对 C++ 标准库的 std::thread 进行了封装，提供了更加安全和易用的线程管理接口。

## 核心成员变量

```cpp
bool started_;                          // 线程是否已启动
bool joined_;                           // 线程是否已经被join
pid_t tid_;                             // 线程ID
ThreadFunc func_;                       // 线程函数
std::string name_;                      // 线程名称
std::shared_ptr<std::thread> thread_;   // 底层std::thread对象的智能指针
static std::atomic_int numCreated_;     // 已创建线程的数量(原子类型)

```

## 核心功能实现

### 1. 构造与析构

```cpp
Thread::Thread(ThreadFunc func, const std::string &name)
    : started_(false)
    , joined_(false)
    , tid_(0)
    , func_(std::move(func))
    , name_(name)
{
    setDefaultName();
}

Thread::~Thread()
{
    if (started_ && !joined_)
    {
        thread_->detach(); // 防止资源泄漏
    }
}

```

- 构造函数使用移动语义获取线程函数，提高效率
- 析构函数通过检查线程状态，确保未join的线程会被detach，防止资源泄漏

### 2. 线程启动机制

```cpp
void Thread::start()
{
    started_ = true;
    sem_t sem;
    sem_init(&sem, false, 0);  // false表示不进程间共享

    // 创建并启动线程
    thread_ = std::shared_ptr<std::thread>(new std::thread([&]() {
        tid_ = CurrentThread::tid();  // 获取线程ID
        sem_post(&sem);               // 通知主线程
        func_();                      // 执行线程函数
    }));

    // 等待新线程设置好tid_
    sem_wait(&sem);
}

```

这个方法展示了几个设计亮点：

- 使用信号量实现线程同步，确保线程ID正确设置后再返回
- 使用lambda表达式简化线程创建
- 使用智能指针管理线程对象生命周期

### 3. 线程等待与命名

```cpp
void Thread::join()
{
    joined_ = true;
    thread_->join();
}

void Thread::setDefaultName()
{
    int num = ++numCreated_;
    if (name_.empty())
    {
        char buf[32] = {0};
        snprintf(buf, sizeof buf, "Thread%d", num);
        name_ = buf;
    }
}

```

- `join()` 方法包装了std::thread的join，并记录状态
- `setDefaultName()` 为未命名线程生成默认名称，格式为"ThreadN"

## 设计特点

1. **安全性**：
    - 使用信号量实现线程同步
    - 智能指针防止资源泄漏
    - 自动处理未join的线程
2. **可管理性**：
    - 统一的命名机制
    - 显式的状态跟踪
    - 简洁的线程创建和等待接口
3. **封装性**：
    - 隔离底层线程API细节
    - 提供一致的接口
    - 支持C++11线程特性

Thread类是kama-webserver的基础组件，为EventLoopThread等更高级别的类提供了线程管理能力，是整个网络库并发模型的底层支撑。

# **LFU**

问题：为什么要使用LFU而不是LRU？

● LRU：最近最少使⽤，发⽣淘汰的时候，淘汰访问时间最旧的⻚⾯。

●LFU：最近不经常使⽤，发⽣淘汰的时候，淘汰频次低的⻚⾯。

● 对于时间相关度较低（⻚⾯访问完全是随机，与时间⽆关） WebServer 来说，经常被访问的⻚⾯在下⼀次有 更⼤的可能被访问，此时使⽤LFU更好，并且LFU能够避免周期性或者偶发性的操作导致缓存命中率下降的问 题；⽽对于时间相关度较⾼（某些⻚⾯在特定时间段访问量较⼤，⽽在整体来看频率较低）的 WebServer 来 说，在特定的时间段内，最近访问的⻚⾯在下⼀次有更⼤的可能被访问，此时使⽤LRU更好。

● ⽬前设计的 WebServer 时间相关度较低，所以选择LFU。

LFU的原理⽐较简单，实现⽅法也⽐较多样（leetcode上⾯就有设计LFU和LRU的题），我⽬前采⽤的是双重链表 +hash表的经典做法。

![image.webp](image%205.webp)

LFU的主要框架如上图所示，主要对象有四个：⼀个⼤链表 FreqList ，⼀个⼩链表 KeyList ，⼀个 key- >FreqList 的哈希表 FreqHash 以及⼀个 key->KeyNode 的哈希表 KeyNodeHash ，下⾯简单说下作⽤：

● FreqList ：链表的每个节点都是⼀个⼩链表附带⼀个值表示频度，频度⽤来记录每个key的访问次数

● KeyList ：同⼀频度下的key value节点 KeyNode ，只要找到 key 对应的 KeyNode ，就可以找到对应的 value

●FreqHash ：key到⼤链表节点的映射，这个主要是为了更新频度，因为更新频度需要先将 KeyNode 节点从 当前频度的⼩链表中删除，然后加⼊下⼀频度的⼩链表（通过遍历⼤链表即可找到）

● KeyNodeHash ：key到⼩链表节点的映射，这个Hash主要是为了根据key来找到对应的 KeyNode ，从⽽获得 对应的value

● LRU中主要就两个操作： bool LFUCache::get(string& key, string& val) 和 void LFUCache::set(string& key, string& val) 。 get(2) 就是先在 FreqHash 中找有没有 key 对应的 节点，如果没有，就是缓存未命中，由⽤户调⽤ set(2) 向缓存中添加节点，如果缓存已满的话就在频度最 低的⼩链表中删除最后⼀个节点；如果有，就是缓存命中，此时需要更新当前节点的频度（先将 KeyNode 节 点从当前频度的⼩链表中删除，然后加⼊下⼀频度的⼩链表），加⼊下⼀频度的⼩链表的头部，这部分与LRU 类似。

● ⽽对于 WebServer 来说，key就是对应的⽂件名，value就是对应的⽂件内容（通过 mmap() 映射），这样 来建⽴⻚⾯缓存还是⽐较简单的。