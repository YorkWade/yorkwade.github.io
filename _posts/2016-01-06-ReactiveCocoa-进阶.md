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



 而Actor则是一种全新的抽象，使用Actor要面临整个应用架构机制和思维方式的变更。它试图要解决的问题要更广一些，比如容错，比如分布式。但Actor的问题在于以当前的调度效率，哪怕是用Goroutine这样的机制，也很难达到直接方法调用的效率。当前要像OO的『一切皆对象』一样实现一个『一切皆Actor』的语言，效率上肯定有问题。所以折中的方式是在OO的基础上，将系统的某个层面的组件抽象为Actor。


# 参考

- [关于并发模型 Actor 和 CSP](https://blog.csdn.net/hotdust/article/details/72475630)
- [并发之痛 Thread，Goroutine，Actor](http://jolestar.com/parallel-programming-model-thread-goroutine-actor/
