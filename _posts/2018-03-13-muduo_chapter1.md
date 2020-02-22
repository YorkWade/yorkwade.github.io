---
layout:     post
title:      muduo读书笔记第1章
subtitle:   第1章 线程安全的对象生命周期管理
date:       2018-03-13
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - muduo
---



用同步原语保护内部状态，但对象的生死不能由对象自身拥有的mutex保护。如何避免对象析构是可能存在的race condition是C++多线程面临的基本问题，可以借助boost的share_ptr和weak_ptr完美解决。Observer模式的必备技术。

析构遇到多线程时，面临的问题：

    一个线程执行对象某个成员函数，另一个线程析构该对象。
    对象销毁时，析构函数是否会执行一半，切换线程。（对象可能存在继承关系）

c++标准库里的大多数class都不是线程安全的,包括std::string、std::vetor、std::map等。需要外部加锁。

**构造的线程安全**
对象创建时做到线程安全，唯一的要求：构造期间不要泄漏this指针。

    1、构造函数中不要注册任何回调。(回调就意味着别的地方拥有该对象的指针)
    2、不要把this传给跨线程对象。（可能存在半成品对象）
    3、即便在构造函数最后一行也不行。（可能该类为基类，子类还在构造中）
正确方法：二段式构造。即，构造函数+initialize()。返回值判断是否构建成功。

**销毁**
锁被定义为类对象的成员，不能保护析构过程。</br>
一个函数如果要锁住相同类型的多个对象，防止发生死锁，可以保证按相同顺序加锁，如比较mutex对象地址。</br>
动态创建的对象是否活着，光看指针看不出来，这是c/c++指针的根源问题。</br>
需要一个东西，来分辨出指针当前时活着还是死了，即只能指针。shareds_ptr/weak_ptr</br>

内存问题在c++中很容易解决：

    1、缓冲区溢出（buffer overrun）。
    用std::vector<char>/std::string通过成员函数修改缓冲区，而不是指针。
    2、空指针\野指针。
    shared_ptr/weak_ptr
    3、重复释放（double delete）
    scoped_ptr，只在对象析构的时候释放一次。
    4、内存泄漏
    scoped_ptr。对象析构的时候自动释放。
    5、不配对的new[]/delete
    new[]，统统替换城std::vector/scoped_array。
    6、内存碎片（memory fragmentation）。

现代的c++程序中，一般不会出现delete语句，资源都是通过对象（智能指针或容器）管理的。

shared_ptr计数本身是线程安全且无锁的，但是shared_ptr对象的读写操作不是线程安全的，需要加锁。如shared_ptr<T> globalPtr;多线程读写他需要加锁。
shared_ptr注意：
    
    1、可能会意外延长对象生命期，如用boost::bind会把实参拷贝一份，不是形参。
    2、shared_ptr拷贝开销比原始指针高。可以最外层的函数用shared_ptr引用，函数最内层用原始指针。通常以常引用传参。
    3、shared_ptr是管理共享资源的利器，需要注意避免循环引用，通常的做法是，owner持有child的shared_ptr，child持有只想owner的weak_ptr。
    4、定制析构。shared_ptr的构造函数可以有一个额外的模版参数，可传入函数指针fun，在析构对象时，执行fun(ptr),ptr为shared_ptr保护的对象指针。可以用来解决，map中保存shared_ptr，而使得对象一直存在，延长对象生命期，当对象析构的时候，回调fun函数，有机会调用map.erase。还需要使用enable_shared_from_this可使成员函数fun函数能够保证对象的生命周期。在对象池的应用场景中。
    5、弱回调（如果活着，就调用他的成员函数，否则就忽略之）。weak_ptr转为shared_ptr，如果转成功就调用成员函数，否则不调用。

**对象池最佳实现：**
```objc
class StockFactory : public boost::enable_shared_from_this<StockFactory>,
                     boost::noncopyable
{
 public:
  boost::shared_ptr<Stock> get(const string& key)
  {
    boost::shared_ptr<Stock> pStock;
    muduo::MutexLockGuard lock(mutex_);
    boost::weak_ptr<Stock>& wkStock = stocks_[key];
    pStock = wkStock.lock();
    if (!pStock)
    {
      pStock.reset(new Stock(key),
                   boost::bind(&StockFactory::weakDeleteCallback,
                               boost::weak_ptr<StockFactory>(shared_from_this()),
                               _1));
      wkStock = pStock;
    }
    return pStock;
  }

 private:
  static void weakDeleteCallback(const boost::weak_ptr<StockFactory>& wkFactory,
                                 Stock* stock)
  {
    printf("weakDeleteStock[%p]\n", stock);
    boost::shared_ptr<StockFactory> factory(wkFactory.lock());
    if (factory)
    {
      factory->removeStock(stock);
    }
    else
    {
      printf("factory died.\n");
    }
    delete stock;  // sorry, I lied
  }

  void removeStock(Stock* stock)
  {
    if (stock)
    {
      muduo::MutexLockGuard lock(mutex_);
      auto it = stocks_.find(stock->key());
      assert(it != stocks_.end());
      if (it->second.expired())
      {
        stocks_.erase(stock->key());
      }
    }
  }

 private:
  mutable muduo::MutexLock mutex_;
  std::map<string, boost::weak_ptr<Stock> > stocks_;
};

void testLongLifeFactory()
{
  boost::shared_ptr<StockFactory> factory(new StockFactory);
  {
    boost::shared_ptr<Stock> stock = factory->get("NYSE:IBM");
    boost::shared_ptr<Stock> stock2 = factory->get("NYSE:IBM");
    assert(stock == stock2);
    // stock destructs here
  }
  // factory destructs here
}

void testShortLifeFactory()
{
  boost::shared_ptr<Stock> stock;
  {
    boost::shared_ptr<StockFactory> factory(new StockFactory);
    stock = factory->get("NYSE:IBM");
    boost::shared_ptr<Stock> stock2 = factory->get("NYSE:IBM");
    assert(stock == stock2);
    // factory destructs here
  }
  // stock destructs here
}
```
**Observer模式最佳实现：**
```objc
#include <algorithm>
#include <vector>
#include <stdio.h>
#include "../Mutex.h"
#include <boost/enable_shared_from_this.hpp>
#include <boost/shared_ptr.hpp>
#include <boost/weak_ptr.hpp>

class Observable;

class Observer : public boost::enable_shared_from_this<Observer>
{
 public:
  virtual ~Observer();
  virtual void update() = 0;

  void observe(Observable* s);

 protected:
  Observable* subject_;
};

class Observable
{
 public:
  void register_(boost::weak_ptr<Observer> x);
  // void unregister(boost::weak_ptr<Observer> x);

  void notifyObservers()
  {
    muduo::MutexLockGuard lock(mutex_);
    Iterator it = observers_.begin();
    while (it != observers_.end())
    {
      boost::shared_ptr<Observer> obj(it->lock());
      if (obj)
      {
        obj->update();
        ++it;
      }
      else
      {
        printf("notifyObservers() erase\n");
        it = observers_.erase(it);
      }
    }
  }

 private:
  mutable muduo::MutexLock mutex_;
  std::vector<boost::weak_ptr<Observer> > observers_;
  typedef std::vector<boost::weak_ptr<Observer> >::iterator Iterator;
};

Observer::~Observer()
{
  // subject_->unregister(this);
}

void Observer::observe(Observable* s)
{
  s->register_(shared_from_this());
  subject_ = s;
}

void Observable::register_(boost::weak_ptr<Observer> x)
{
  observers_.push_back(x);
}

//void Observable::unregister(boost::weak_ptr<Observer> x)
//{
//  Iterator it = std::find(observers_.begin(), observers_.end(), x);
//  observers_.erase(it);
//}

// ---------------------

class Foo : public Observer
{
  virtual void update()
  {
    printf("Foo::update() %p\n", this);
  }
};

int main()
{
  Observable subject;
  {
    boost::shared_ptr<Foo> p(new Foo);
    p->observe(&subject);
    subject.notifyObservers();
  }
  subject.notifyObservers();
}
```


shared_ptr可以安全的跨越模块边界。不会出现DLLA分配的内存在DLLB里释放的情况。

RAII(资源获取即初始化)时C++语言区别其他语言的重要特性。不懂RAII的c++程序员不是一个合格的程序员。程序一般不出现delete。

无需自己编写引用计数的智能指针，重新发明轮子。shared_ptr提供了完美的解决方案。

尽量减少使用跨线程对象，使用生产者消费者，任务队列这种有规律的机制，最低限度的共享数据。</br>
原始指针暴露给多线程会造成race condition或额外的簿记负担。</br>
统一使用shared_ptr/scoped_ptr来管理对象的生命期，在多线程中尤为重要。</br>
shared_ptr当心延长对象生命周期。</br>
weak_ptr/shared_ptr的好搭档，可用作弱回调、对象池。</br>
Observer的设计模式，有本质问题：Observer是基类，强耦合（仅次于友元），如果需要观察多个类型的事件（时钟和温度），需要多继承，如果重复观察同一类型的事件（1秒一次心跳和3秒一次自检），需要额外技术。可用boost::function/boost:bind绕开。类似于Signal/Slots的模式。</br>
《C++沉思录》详细介绍了handle/body idiom，这是编写大型c++程序的必备技术，也是实现物理隔离的法宝。</br>
《java concurrency in practice》 多线程领域专著。</br>

## 我的总结：
    多线程下
    1、构造函数不应该暴露this指针给别人，可能构造一半，正确做法：构造函数+initialize()
    2、内存问题，用shared_ptr/weak_ptr都能解决，避免直接操作指针。
    3、代码中应避免出现delete。正确做法：使用RAII机制自动销毁，shared_ptr销毁new的资源。
    4、减少使用跨线程对象，使用生产者消费者，任务队列这种有规律的机制，最低限度的共享数据。
    5、handle/body物理隔离的法宝（桥接模式）
    6、对象池和Observer模式的实现

## 附：
std::shared_ptr 和 std::weak_ptr的用法以及引用计数的循环引用问题

    在std::shared_ptr被引入之前，C++标准库中实现的用于管理资源的智能指针只有std::auto_ptr一个而已。std::auto_ptr的作用非常有限，因为它存在被管理资源的所有权转移问题。这导致多个std::auto_ptr类型的局部变量不能共享同一个资源，这个问题是非常严重的哦。因为，我个人觉得，智能指针内存管理要解决的根本问题是：一个堆对象（或则资源，比如文件句柄）在被多个对象引用的情况下，何时释放资源的问题。何时释放很简单，就是在最后一个引用它的对象被释放的时候释放它。关键的问题在于无法确定哪个引用它的对象是被最后释放的。std::shared_ptr确定最后一个引用它的对象何时被释放的基本想法是：对被管理的资源进行引用计数，当一个shared_ptr对象要共享这个资源的时候，该资源的引用计数加1，当这个对象生命期结束的时候，再把该引用技术减少1。这样当最后一个引用它的对象被释放的时候，资源的引用计数减少到0，此时释放该资源。下边是一个shared_ptr的用法例子：
    
```objc
#include <iostream>  
#include <memory>  
  
class Woman;  
class Man{  
private:  
    std::weak_ptr<Woman> _wife;  
    //std::shared_ptr<Woman> _wife;  
public:  
    void setWife(std::shared_ptr<Woman> woman){  
        _wife = woman;  
    }  
  
    void doSomthing(){  
        if(_wife.lock()){  
        }  
    }  
  
    ~Man(){  
        std::cout << "kill man\n";  
    }  
};  
  
class Woman{  
private:  
    //std::weak_ptr<Man> _husband;  
    std::shared_ptr<Man> _husband;  
public:  
    void setHusband(std::shared_ptr<Man> man){  
        _husband = man;  
    }  
    ~Woman(){  
        std::cout <<"kill woman\n";  
    }  
};  
  
  
int main(int argc, char** argv){  
    std::shared_ptr<Man> m(new Man());  
    std::shared_ptr<Woman> w(new Woman());  
    if(m && w) {  
        m->setWife(w);  
        w->setHusband(m);  
    }  
    return 0;  
}  
```

    在Man类内部会引用一个Woman，Woman类内部也引用一个Man。当一个man和一个woman是夫妻的时候，他们直接就存在了相互引用问题。man内部有个用于管理wife生命期的shared_ptr变量，也就是说wife必定是在husband去世之后才能去世。同样的，woman内部也有一个管理husband生命期的shared_ptr变量，也就是说husband必须在wife去世之后才能去世。这就是循环引用存在的问题：husband的生命期由wife的生命期决定，wife的生命期由husband的生命期决定，最后两人都死不掉，违反了自然规律，导致了内存泄漏。
     解决std::shared_ptr循环引用问题的钥匙在weak_ptr手上。weak_ptr对象引用资源时不会增加引用计数，但是它能够通过lock()方法来判断它所管理的资源是否被释放。另外很自然地一个问题是：既然weak_ptr不增加资源的引用计数，那么在使用weak_ptr对象的时候，资源被突然释放了怎么办呢？呵呵，答案是你根本不能直接通过weak_ptr来访问资源。那么如何通过weak_ptr来间接访问资源呢？答案是：在需要访问资源的时候weak_ptr为你生成一个shared_ptr，shared_ptr能够保证在shared_ptr没有被释放之前，其所管理的资源是不会被释放的。创建shared_ptr的方法就是lock()方法。
    细节：shared_ptr实现了operator bool() const方法来判断一个管理的资源是否被释放
从对象的原始指针，生成需要的shared_ptr。
使用std::tr1::enable_shared_from_this 作为基类。这样可以从 this 生成具有统一计数的 shared_ptr了。
```objc
class C : public std::tr1::enable_shared_from_this<C>  
{  
public:  
    std::tr1::shared_ptr<C> getSharedPtr()  
    {  
        return shared_from_this();  
    }  
};  
```
shared_from_this() 不能在构造函数中使用，因为在 enable_shared_from_this 这个基类内部，是通过一个对自己的 weak_ptr 的引用来返回 this 的shared_ptr 的，而在对象的构造函数中，第一个 shared_ptr 尚未获得指针，所以 weak_ptr 是空的，shared_from_this() 返回会失败。
