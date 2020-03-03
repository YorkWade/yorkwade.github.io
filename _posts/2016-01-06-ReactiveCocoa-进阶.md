---
layout:     post
title:      ReactiveCocoa 进阶
subtitle:   函数式编程框架 ReactiveCocoa 进阶
date:       2017-01-06
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Actor
---


# 前言



CSP的模式比较适合Boss-Worker模式的任务分发机制，它的侵入性没那么强，可以在现有的系统中通过CSP解决某个具体的问题。它并不试图解决通信的超时容错问题，这个还是需要发起方进行处理。同时由于Channel是显式的，虽然可以通过netchan（原来Go提供的netchan机制由于过于复杂，被废弃，在讨论新的netchan）实现远程Channel，但很难做到对使用方透明。
而Actor则是一种全新的抽象，使用Actor要面临整个应用架构机制和思维方式的变更。它试图要解决的问题要更广一些，比如容错，比如分布式。但Actor的问题在于以当前的调度效率，哪怕是用Goroutine这样的机制，也很难达到直接方法调用的效率。当前要像OO的『一切皆对象』一样实现一个『一切皆Actor』的语言，效率上肯定有问题。所以折中的方式是在OO的基础上，将系统的某个层面的组件抽象为Actor。


   对于某个 Actor 我们可以为某个消息类型注册多个消息处理函数，那么此消息类型对应的多个消息处理函数会按照注册的顺序被串行执行下去
   
   线程按顺序处理 Actor 收到的消息，一个消息未处理完成不会处理消息队列中的下一个消息 我们可以想象，如果存在三个 Actor，其中两个 Actor 的消息处理函数中存在死循环（例如上例中的 while(true)），那么它们一旦执行就会霸占两条线程，若线程池中没有多余线程，那么另一个 Actor 将被“饿死”（永远得不到执行）。我们可以在设计上避免这种 Actor 的出现，当然也可以适当的调整线程池的大小来解决此问题。Theron 中，线程池中线程的数量是可以动态控制的，线程利用率也可以测量。但是务必注意的是，过多的线程必然导致过大的线程上下文切换开销。
   
   和传统的多线程程序相比 Theron 有不少优势，通过使用 Actor，程序能够自动的并行执行，而无需开发者费心。Actor 总是利用消息进行通信，消息必须拷贝，这也意味着我们必须注意到，在利用 Actor 进行并行运算的同时需避免大量消息拷贝带来的额外开销。
   
   Actor 模型强调了一切皆为 Actor，这自然可以作为我们使用 Theron 的一个准则。但过多的 Actor 存在必然导致 Actor 间频繁的通信。适当的使用 Actor 并且结合 Object 模型也许会是一个不错的选择，例如，我们可以对系统进行适当划分，得到一些功能相对独立的模块，每个模块为一个 Actor，模块内部依然使用 Object 模型，模块间通过 Actor 的消息机制进行通信。

# 参考

- [关于并发模型 Actor 和 CSP](https://blog.csdn.net/hotdust/article/details/72475630)
- [并发之痛 Thread，Goroutine，Actor](http://jolestar.com/parallel-programming-model-thread-goroutine-actor/)
- [通过Actor模型解决C++ 并发编程的一种思维 — Theron 库简述](https://blog.csdn.net/sigh667/article/details/76438785)

