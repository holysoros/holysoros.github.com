---
layout: post
title: "如何选择架构实现 Ruby 高性能程序"
description: ""
category:
tags: []
---

最近工作上写了一个需要处理较大并发数据的 Ruby 程序，简单来说就是, 每秒钟收到一个大压缩包，该压缩包是数千条 C Struct 封装后的信息经过 gzip 压缩的结果，程序需要解包，然后逐条处理数千条信息，在我们这里是解包后将每条存到 Redis 中.

我使用 `BinData` 这个 gem 解包，这个 gem 是纯 ruby 实现的，用起来的确很灵活，但是效率实在是较差. 使用 `redis` gem 与 Redis 交互.

由于需要处理较大并发量，程序从开始架构我就使用了线程池，粗略的架构图如下：

![drawing](/assets/images/my_architecture.png)

但是发现当线程池只有一个线程时，处理一个包的时间假设为 1s ；而有两个线程时，每个线程处理一个包的时间变成了近 2s ！以此类推。导致程序中消息总是处理不过来，造成很大的延迟。如果这样，多个线程就毫无意义。理论上线程并行执行，不应该有如此大的差异。这时想起了 Ruby MRI 的实现有 GIL.

什么是 GIL
-------
> Ruby MRI has something called a global interpreter lock (GIL). It is a lock around the execution of Ruby code. This means that in a multi-threaded context, only one thread can execute Ruby code at any one time.

![Ruby GIL](/assets/images/ruby-gil.png)

如图，Ruby 1.8.7 中 Ruby 的 thread 并不对应 OS native thread，称为 green thread。Ruby 1.9 后 Ruby 中新建一个 thread 就对应创建一个 OS native thread，但是由于 GIL 的存在，在任何时候只有一个线程能执行 Ruby 代码。这就减少了 Ruby thread 适用的场景.

事实上不只 Ruby ，其他的脚本语言的很多实现，如 Python, node.js 也都有 GIL。GIL 一方面使 Ruby 代码执行时是 thread-safe ，一方面又限制了并行.

GIL 会使 Ruby 执行时是线程安全的吗
-------
简单来说, all Ruby methods implemented in C are atomic. 如例子：

```ruby
array = []

5.times.map do
  Thread.new do
    1000.times do
      array << nil
    end
  end
end.each(&:join)

puts array.size
```

在 MRI 下结果总是 5000 ，但是在 JRuby 或 Rubinius 下结果通常小于 5000 。这是因为 `array <<`操作在 MRI 下由于 GIL 的存在，是线程安全的，但是 Jruby 与 Rubinius 却没有 GIL ，因此 `array <<` 不是线程安全的.

尽管 `array <<` 是线程安全的，但是类似 `array << User.find(1)` 却不是线程安全的，这是因此 `array <<` 是 C 实现的一个 Ruby 方法。

因此，应该尽量避免依赖 MRI GIL 提供的线程安全保证，否则写出的代码既不能确认真的是线程安全的，当切换使用其他 Ruby 实现时又会遇到较大的移植问题。该使用 mutex 的还是要使用。

关于 GIL 更详细的解释可以看 [Nobody understands the GIL](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil) ，从 MRI 源码级别讨论了 GIL 的行为，内容很长。

什么情况下使用 Ruby 多线程
-------
  并不是 MRI 有 GIL ，Ruby 的多线程就没有意义了。MRI 会在一个线程执行 100ms 或该线程状态变为 sleep 后，切换另一个线程执行. 当所有线程要处理的任务都是 CPU-bound 时，GIL 的确会让多线程毫无意义；但是如果任务是 IO-bound 的，线程经常会为了等待 IO 回应而处于 sleep 状态，这是 MRI 就可以切换另一个线程继续执行，而不是串行地等待。

因此，在 MRI 下：

- 对于 CPU-bound 的任务就不要使用多线程了；
- 对于 IO-bound 的任务可以继续考虑使用多线程，不过或许有更好的方法，如 EventMachine ；

我最终如何解决
-------
由于在我程序中，需要处理的任务是 CPU-bound 与 IO-bound 混合的，意识到 Ruby GIL 的存在，多线程处理就没有意义了。最终我只使用一个线程处理，在无法降低解包耗时的情况下，通过以下几个措施大幅降低了操作 Redis 的耗时：

- 使用 `hiredis` driver，默认 `redis` 是使用纯 Ruby driver 的，效率较低；
- 使用 UNIX Socket ，而不是 INET Socket ；这样 Redis 命令在传输时就不用横跨整个 TCP/IP 协议栈；
- 使用 Redis pipeline ；
每个措施都贡献了约 2~3x 的速度提升，通过这一系列优化，终于能从容地及时处理完源源不断的消息了。

这其实又是一个过早优化的教训，如果刚开始不使用线程池，而是从一个线程做起，通过度量，发现最终的性能瓶颈，就可以更快的解决问题。最终我度量了每种情况下的耗时，才能得出前面的数据。

更好的架构
-------
使用多线程是为了并行执行(Parallelism)，并且由于是在同一个进程空间，多个线程可以方便地快速共享数据。而同样通过多进程也能实现并行，多进程之间共享数据就需要使用各种 IPC(Inner Process Communication) 了，传统的 IPC 方式有：

- PIPE 或 FIFO(又称 Named PIPE);
- UNIX Socket 或 INET Socket;

但是这些 IPC 都是 Stream 型的，传输数据的时候需要自己定义边界，更好的选择是 Message Queue ，如:

- 使用 Redis 配合 lpush/brpop 命令;
- RabbitMQ ;

通常就是生产者-消费者模型的架构，这样架构的优势有：

- 通过消息队列，完全解耦合了生产者与消费者，生产者与消费者可以使用不同的语言开发实现；
- 可以同时运行多个生产者或多个消费者进程，甚至可以在不同的机器上运行，有很强的水平扩展性；
- 完全避免了 Ruby GIL 的影响；
- 不用再考虑复杂的线程同步的问题，但是如果多个进程同时操作外部资源时，如写文件，同样有 Race condition...
- 通常这种解耦合之后，写出的代码会更清晰；

其实，在 Linux 下，线程也是进程实现的，线程被称为轻量级线程(LWP, Light Weight Process)。至于是选择多线程架构还是多进程架构，还是要综合多种因素考虑，如 Redis 一秒钟能执行 100k~300k 次 push 命令，如果对性能的要求高于这个的话... 还是只能选择多线程架构。

References
-------
- [Parallelism is a Myth in Ruby](https://www.igvita.com/2008/11/13/concurrency-is-a-myth-in-ruby/)
