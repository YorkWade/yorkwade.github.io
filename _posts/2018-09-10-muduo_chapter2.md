---
layout:     post
title:      muduo读书笔记之线程同步精要
subtitle:   第2章
date:       2018-09-10
author:     BY
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - muduo
---


第2章 线程同步精要

并发编程有两种基本模型:

	1、message passing
	2、shared memory
	
本章多次引用《 Real-World Concurrenc》【RWC】</br>
线程同步四项原则：

    1、尽量最低限度的共享对象，减少同步场合。对象能不暴露就不暴露，如果暴露优先immutalbe,最后用同步措施。
    2、使用高级的并发编程构建。TaskQueue、Producer-Consumer Queue、CountDownLatch(倒计时)
    3、最后不得已使用同步原语。只用非递归互斥器和条件变量，慎用读写锁，不用信号量。
    4、除了使用atomic原子整数之外，不自己编写lock-free代码（见【RWC】），不要用内核级同步原语，不凭空猜测”哪种性能好“如spin lock vs. mutex。

mutex
原则：

    1、用RAII手法封装mutex的创建、销毁、加锁、解锁四个操作，这是c++标准实践。
    2、只用非递归mutex。
    3、不手动调用lock和unlock。使用栈上的Guard对象构造和析构，即scoped locking。 
    4、用栈上的Guard对象，看函数调用栈就能分析用锁情况，方便查询死锁。
    
次要原则：

    1、不用夸进程mutex，进程通信只用tcp socket。
    2、使用RAII可保证，加锁解锁在同一线程、不会重复解锁、忘记解锁。
    3、必要时可考虑PTHREAD_MUTEX_ERRORCHECK排错。

非递归锁和递归锁性能差别不大，因为少用一个计数器，前者略快。使用非递归锁，可将死锁暴露出来，可通过查看各个线程的调用堆栈（gdb中使用thread apply all bt命令），即可debug。使用递归锁，可能偶尔崩溃，天知道。
windows 的CRITICAL_SECTION 是递归锁。
避免误用：
    如果一个函数既可能在加锁的情况下调用，又可能在未加锁的情况下调用，那就拆成两个：
    1、跟原来的函数同名，函数加锁，转而调用第二个函数。
    2、给函数名加上后缀WithLockedHold,不加锁，把原来的函数搬过来。
Pthreads 提供isLockedByThisThread（）方法，可assert(mutex.isLockedByThisThread()；
《Lock Convoys Explained》潘爱民 使用锁可能对性能的影响。

条件变量
    只有一种正确使用方式，不易用错：
wait端：

    1、必须与mutex一起使用，该布尔表达式的读写需要受此mutex保护。
    2、在mutex上锁的时候才能调用wait。
    3、把判断条件和wait()放到while循环中。while不能用if代替，原因是spurious wakeup，面试多线程编程常见考点。
条件变量是非常底层的同步原语，很少直接使用，一般都是用它来实现高层的同步措施，BlockingQueue<T>或CountDownLatch。
	
```objc
BlockingQueue
mutable MutexLock mutex_;
  Condition         notEmpty_;
  std::deque<int>     queue_;
 int enqueue(const T& x)
  {
    MutexLockGuard lock(mutex_);
    queue_.push_back(x);
    notEmpty_.notify(); // wait morphing saves us
    // http://www.domaigne.com/blog/computing/condvars-signal-with-mutex-locked-or-not/
  }

 void dequeue()
  {
    MutexLockGuard lock(mutex_);
    // always use a while-loop, due to spurious wakeup
    while (queue_.empty())
    {
      notEmpty_.wait();
    }
    assert(!queue_.empty());
    int front(queue_.front());
    queue_.pop_front();
    return front;
  }

CountDownLatch
using namespace muduo;

CountDownLatch::CountDownLatch(int count)
  : mutex_(),
    condition_(mutex_),
    count_(count)
{
}

void CountDownLatch::wait()
{
  MutexLockGuard lock(mutex_);
  while (count_ > 0)
  {
    condition_.wait();
  }
}

void CountDownLatch::countDown()
{
  MutexLockGuard lock(mutex_);
  --count_;
  if (count_ == 0)
  {
    condition_.notifyAll();
  }
}

int CountDownLatch::getCount() const
{
  MutexLockGuard lock(mutex_);
  return count_;
}
```

signal/broadcast端：

    1、不一定要在mutex已上锁情况下调用signal。
    2、在signal之前一般修改布尔表达式。
    3、修改布尔表达式通常要mutex保护。
    4、区分signal（表示资源可用）和broadcast（表明状态变化）
    
条件变量典型应用BlockingQueue和CountDownLatch
mutex应该先于condition构造。

先精通这两个同步原语，学会编写正确的、安全的多线程程序，再在必要的时候考虑“高技术”-lock-free  Lock-Free Code: A False Sense of Security http://www.drdobbs.com/cpp/lock-free-code-a-false-sense-of-security/210600279       https://coolshell.cn/articles/8239.html
不要用读写锁和信号量
读写锁

    1、程序员如果在读锁中误用修改数据的函数，出现问题不易维护
    2、如果临界区很小，mutex可能比读锁更快，因为读锁要更新当前reader的数据。
    3、有的读锁可以提升为写锁（Pthread rwlock不允许提升），如果提升为写锁，就跟递归锁一样，可能造成迭代器失效。
    4、读锁可重入，写锁不可重入，读锁在可重入时可能会与写锁发生死锁。追求低延时读取的场合不适合读写锁。写锁优先，会阻塞后面的读锁。对一个同享的数据布局,读的频率远弘远于写,所以用了读写锁.但是发现写线程老是抢不到锁。http://blog.csdn.net/ysu108/article/details/39343295 慎用读写锁

信号量
    信号量有自己的计数值，而通常我们自己的数据结构也有长度之，这就造成了同样的信息保存了两份，需要时刻保持一致，增加了程序员负担和出错的可能。

多线程编程中，尽量缩短临界区。
google的CHECK()宏可实现non-degug 的 assert

线程安全的Singleton
double checked locking（DCL），有“神牛”指出由于乱序执行的影响，DCL可能靠不住。
linux中可以使用pthread_onces实现singeton的线程安全。

Sleep(3)不是同步原语
sleep/usleep/nanosleep只能出现在测试代码中，用于有意延长临界区，加速复现死锁的情况。不要用它切换其他线程，这是业余做法。
生产代码中线程的等待分两种：    

    1、等待资源可用（等在select/poll/epoll_wait上，或者等在条件变量上）
    2、等待进入临界区。（临界区通常极短，否则程序性能和伸缩性就会有问题）
    
如果程序真的要等待一段已知时间，应该在event loop中注册一个timer，然后在timer回调里接着干活，不要阻塞线程，良妃珍贵的资源。如果等待某个事件发生，应该采用条件变量或者IO事件回调，不要用sleep轮询。
    如果多线程的安全性和效率要考代码主动调用sleep来保证，这显然是设计出现问题。正确做法：用select（等价物）或condition。用户态轮询（polling）是低效的。

真正影响性能的不是锁，而是锁的争用（正确的加锁）。
在分布式系统中，多机的伸缩性（scale out）比单机的性能优化更值得投入精力。在程序的复杂度和性能中，考虑未来两三年的可能（CPU变快，核数变多，机器数量增加，网络升级）。

归纳总结：
尽量用高层同步设施（线程池、队列】倒计时）
使用底层的同步设施（互斥器、条件变量）完成剩余的同步任务，采用RAII和Scoped locking方式。

借shared_ptr实现copy-on-write
CopyOnWrite_test 只用非递归锁
```objc
using namespace muduo;

class Foo
{
 public:
  void doit() const;
};

typedef std::vector<Foo> FooList;
typedef boost::shared_ptr<FooList> FooListPtr;
FooListPtr g_foos;
MutexLock mutex;

void post(const Foo& f)
{
  printf("post\n");
  MutexLockGuard lock(mutex);
  if (!g_foos.unique())
  {
    g_foos.reset(new FooList(*g_foos));
    printf("copy the whole list\n");
  }
  assert(g_foos.unique());
  g_foos->push_back(f);
}

void traverse()
{
  FooListPtr foos;
  {
    MutexLockGuard lock(mutex);
    foos = g_foos;
    assert(!g_foos.unique());
  }

  // assert(!foos.unique()); this may not hold

  for (std::vector<Foo>::const_iterator it = foos->begin();
      it != foos->end(); ++it)
  {
    it->doit();
  }
}

void Foo::doit() const
{
  Foo f;
  post(f);
}

int main()
{
  g_foos.reset(new FooList);
  Foo f;
  post(f);
  traverse();
}
```

解决死锁
```objc
class Request;
typedef boost::shared_ptr<Request> RequestPtr;

class Inventory
{
 public:
  Inventory()
    : requests_(new RequestList)
  {
  }

  void add(const RequestPtr& req)
  {
    muduo::MutexLockGuard lock(mutex_);
    if (!requests_.unique())
    {
      requests_.reset(new RequestList(*requests_));
      printf("Inventory::add() copy the whole list\n");
    }
    assert(requests_.unique());
    requests_->insert(req);
  }

  void remove(const RequestPtr& req) // __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    if (!requests_.unique())
    {
      requests_.reset(new RequestList(*requests_));
      printf("Inventory::remove() copy the whole list\n");
    }
    assert(requests_.unique());
    requests_->erase(req);
  }

  void printAll() const;

 private:
  typedef std::set<RequestPtr> RequestList;
  typedef boost::shared_ptr<RequestList> RequestListPtr;

  RequestListPtr getData() const
  {
    muduo::MutexLockGuard lock(mutex_);
    return requests_;
  }

  mutable muduo::MutexLock mutex_;
  RequestListPtr requests_;
};

Inventory g_inventory;

class Request : public boost::enable_shared_from_this<Request>
{
 public:
  Request()
    : x_(0)
  {
  }

  ~Request()
  {
    x_ = -1;
  }

  void cancel() __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    x_ = 1;
    sleep(1);
    printf("cancel()\n");
    g_inventory.remove(shared_from_this());
  }

  void process() // __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    g_inventory.add(shared_from_this());
    // ...
  }

  void print() const __attribute__ ((noinline))
  {
    muduo::MutexLockGuard lock(mutex_);
    // ...
    printf("print Request %p x=%d\n", this, x_);
  }

 private:
  mutable muduo::MutexLock mutex_;
  int x_;
};

void Inventory::printAll() const
{
  RequestListPtr requests = getData();
  printf("printAll()\n");
  sleep(1);
  for (std::set<RequestPtr>::const_iterator it = requests->begin();
      it != requests->end();
      ++it)
  {
    (*it)->print();
  }
}

void threadFunc()
{
  RequestPtr req(new Request);
  req->process();
  req->cancel();
}

int main()
{
  muduo::Thread thread(threadFunc);
  thread.start();
  usleep(500*1000);
  g_inventory.printAll();
  thread.join();
}
```
替换读写锁例子
```objc
class CustomerData : boost::noncopyable
{
 public:
  CustomerData()
    : data_(new Map)
  { }

  int query(const string& customer, const string& stock) const;

 private:
  typedef std::pair<string, int> Entry;
  typedef std::vector<Entry> EntryList;
  typedef std::map<string, EntryList> Map;
  typedef boost::shared_ptr<Map> MapPtr;
  void update(const string& customer, const EntryList& entries);
  void update(const string& message);

  static int findEntry(const EntryList& entries, const string& stock);
  static MapPtr parseData(const string& message);

  MapPtr getData() const
  {
    muduo::MutexLockGuard lock(mutex_);
    return data_;
  }

  mutable muduo::MutexLock mutex_;
  MapPtr data_;
};

int CustomerData::query(const string& customer, const string& stock) const
{
  MapPtr data = getData();

  Map::const_iterator entries = data->find(customer);
  if (entries != data->end())
    return findEntry(entries->second, stock);
  else
    return -1;
}

void CustomerData::update(const string& customer, const EntryList& entries)
{
  muduo::MutexLockGuard lock(mutex_);
  if (!data_.unique())
  {
    MapPtr newData(new Map(*data_));
    data_.swap(newData);
  }
  assert(data_.unique());
  (*data_)[customer] = entries;
}

void CustomerData::update(const string& message)
{
  MapPtr newData = parseData(message);
  if (newData)
  {
    muduo::MutexLockGuard lock(mutex_);
    data_.swap(newData);
  }
}

int main()
{
  CustomerData data;
}
```

## 我的总结：
    1、先考虑尽可能不暴露共享数据给别的线程；再考虑使用高级的并发编程构建。TaskQueue、Producer-Consumer Queue、CountDownLatch(倒计时)；最后再考虑使用同步原语。只用非递归互斥器和条件变量。
    2、用RAII方法加锁解锁，不要手动调用lock和unlock。所有线程加锁顺序一致，可避免死锁。
    3、非递归所，可以让死锁暴露，容易排错（查看所有线程堆栈。gdb中使用thread apply all bt命令）。条件变量注意spurious wakeup。
    4、多线程编程中，尽量缩短临界区。当临界区短时，非递归锁没有技术开销，比递归锁性能高。
    5、不用sleep来切换线程或者阻塞线程这种业余做法。正确做法：event loop注册timer，或者等待select（等同类函数）或condition下，不用低效的轮询（poling）。
    6、分布式下，考虑多机伸缩性比单机的性能优化更值得。
    7、copy on write技巧可以解决死锁，锁争用。

###附1 
Real-World Concurrency阅读笔记
链接： http://queue.acm.org/detail.cfm?id=1454462
由于文章是领域内高人多年经验的总结，有很多地方理解不够深刻，只能先写下自己的理解。
文章首先介绍了并发行的历史:提高系统并发性的唯一目标就是提高性能。并发性提高性能的三种方式：减少、隐藏延迟；提高吞吐量。
接下来是一系列的建议：

	建议1： 辨别系统的热点和非热点路径，并区别对待。 对于非热点路径，不要花太多心思提高并发：效果不彰、容易出错。关于容易出错这一点，系统的非热点路径很多时候是很难实现并发和易出错的代码（如启动、初始化）
	建议2： 数据说话，直觉不一定可靠。获取数据并不容易：搭建软硬件环境。软件环境中需要有强大的系统动态性能分析工具。
	建议3： 仔细权衡对大锁的处理。将全局锁改造成per-cpu lock, lock哈希表，单结构体锁貌似很酷，但有时通过减短持有大锁的时间更为明智。
	建议4： 读写锁的陷阱。当出现读多写少的锁时，直观上倾向于改造成读写锁，当锁持有时间不是很长时间，这样做是否能改进并发性（扩展性）值得商榷，仔细评估。
	建议5： 采用per-cpu锁。两个建议：考虑到实现上会增加复杂度，采用per-cpu锁也需要热点分析数据的支持；保持以同样的顺序获取锁，避免死锁。
	建议6：权衡合适用广播、何时用signal
	建议7：学会事后分析。如利用coredump查找问题。
	建议8：将系统设计成组件化（模块化）的。模块化和锁/条件变量是可以共存的，实现的方式有两种：将锁/条件变量限制在子系统内部；无锁化。
	建议9：当mutex满足要求时，不要使用semaphore。
	建议10：采用memory retiring来实现per-chain哈希表锁。当需要并行化对hash表的访问时，可以采用per-chain的锁。
	建议12：避免false sharing。
	建议13：采用同步非阻塞函数监控竞态条件。
	建议14：如非必须，不用waitfree和lockfree技巧。
	建议15： 时时保持一颗红心两种准备。很难知道当前解决的并发性问题是否是最后一个。

### 附2 spurious wakeup 虚假唤醒
多线程编程中条件变量和虚假唤醒(spurious wakeup)的讨论
多线程编程中条件变量和虚假唤醒的讨论

1. 概述
条件变量(condition variable)是利用共享的变量进行线程之间同步的一种机制。典型的场景包括生产者-消费者模型，线程池实现等。
对条件变量的使用包括两个动作：

	1) 线程等待某个条件, 条件为真则继续执行，条件为假则将自己挂起(避免busy wait,节省CPU资源)；
	2) 线程执行某些处理之后，条件成立；则通知等待该条件的线程继续执行。
	3) 为了防止race-condition，条件变量总是和互斥锁变量mutex结合在一起使用。
一般的编程模式：
```objc
var mutex;
var cond;
var something;

Thread1: (等待线程)
lock(mutex);
while( something not true ){
	condition_wait( cond, mutex);
}
do(something);
unlock(mutex);
============================
Thread2: (解锁线程)

do(something);
....
something = true;

unlock(mutex);
condition_signal(cond);

```

函数说明：
(1) Condition_wait()：调用时当前线程立即进入睡眠状态,同时互斥变量mutex解锁(这两步操作是原子的，不可分割)，以便其它线程能进入临界区修改变量。
也就是说pthread_cond_wait实际上可以看作是以下几个动作的合体:
解锁线程锁
等待条件为true
加锁线程锁.
(2) Condition_signal(): 线程调用此函数后，除了当前线程继续往下执行以外； 操作系统同时做如下动作：从condition_wait()中进入睡眠的线程中选一个线程唤醒， 同时被唤醒的线程试图锁(lock)住互斥量mutex, 当成功锁住后，线程就从condition_wait()中成功返回了。
2. 函数接口
pthread: pthread_cond_wait/pthread_cond_signal/pthread_cond_broadcast()
Java: Condition.await()/Condition.signal()/Condition.signalAll()
3. 虚假唤醒(spurious wakeup)在采用条件等待时，我们使用的是
```objc
while(条件不满足){
   condition_wait(cond, mutex);
}
```
而不是:
```objc
If( 条件不满足 ){
   Condition_wait(cond,mutex);
} 
```
这是因为可能会存在虚假唤醒”spurious wakeup”的情况。
也就是说，即使没有线程调用condition_signal, 原先调用condition_wait的函数也可能会返回。此时线程被唤醒了，但是条件并不满足，这个时候如果不对条件进行检查而往下执行，就可能会导致后续的处理出现错误。
虚假唤醒在linux的多处理器系统中/在程序接收到信号时可能回发生。在Windows系统和JAVA虚拟机上也存在。在系统设计时应该可以避免虚假唤醒，但是这会影响条件变量的执行效率，而既然通过while循环就能避免虚假唤醒造成的错误，因此程序的逻辑就变成了while循环的情况。
注意：即使是虚假唤醒的情况，线程也是在成功锁住mutex后才能从condition_wait()中返回。即使存在多个线程被虚假唤醒，但是也只能是一个线程一个线程的顺序执行，也即：lock(mutex)   检查/处理  condition_wai()或者unlock(mutex)来解锁. 

4. 解锁和等待转移(wait morphing)

解锁互斥量mutex和发出唤醒信号condition_signal是两个单独的操作，那么就存在一个顺序的问题。谁先随后可能会产生不同的结果。如下：

	(1) 按照 unlock(mutex); condition_signal()顺序， 当等待的线程被唤醒时，因为mutex已经解锁，因此被唤醒的线程很容易就锁住了mutex然后从conditon_wait()中返回了。
	(2) 按照 condition_signal(); unlock(mutext)顺序，当等待线程被唤醒时，它试图锁住mutex,但是如果此时mutex还未解锁，则线程又进入睡眠，mutex成功解锁后，此线程在再次被唤醒并锁住mutex，从而从condition_wait()中返回。
可以看到，按照(2)的顺序，对等待线程可能会发生2次的上下文切换，严重影响性能。因此在后来的实现中，对(2)的情况，如果线程被唤醒但是不能锁住mutex,则线程被转移(morphing)到互斥量mutex的等待队列中，避免了上下文的切换造成的开销。 -- wait morphing
编程时，推荐采用(1)的顺序解锁和发唤醒信号。而Java编程只能按照(2)的顺序，否则发生异常!!。
在SUSv3http://en.wikipedia.org/wiki/Single_UNIX_Specification的规范中(pthread)，指明了这两种顺序不管采用哪种，其实现效果都是一样的。
