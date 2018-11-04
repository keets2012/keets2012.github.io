---
title: 设计模式之线程池模式
date: 2017-02-6
categories: 设计模式
tags:
  - 设计模式
abbrlink: 54736
---
## 背景
Thread-Per-Message Pattern，是一种对于每个命令或请求，都分配一个线程，由这个线程执行工作。它将`委托消息的一端`和`执行消息的一端`用两个不同的线程来实现。该线程模式主要包括三个部分：

- Request参与者(委托人)，也就是消息发送端或者命令请求端
- Host参与者，接受消息的请求，负责为每个消息分配一个工作线程。
- Worker参与者，具体执行Request参与者的任务的线程，由Host参与者来启动。

由于常规调用一个方法后，必须等待该方法完全执行完毕后才能继续执行下一步操作，而利用线程后，就不必等待具体任务执行完毕，就可以马上返回继续执行下一步操作。

由于在Thread-Per-Message Pattern中对于每一个请求都会生成启动一个线程，而线程的启动是很花费时间的工作，所以鉴于此，提出了Worker Thread，重复利用已经启动的线程。

## 线程池模式定义
Worker Thread，一般都称为线程池。该模式事先启动一定数目的工作线程。当没有请求工作的时候，所有的工人线程都会等待新的请求过来，一旦有工作到达，就马上从线程池中唤醒某个线程来执行任务，执行完毕后继续在线程池中等待任务池的工作请求的到达。

- 任务池：主要是存储接受请求的集合，利用它可以缓冲接受到的请求，可以设置大小来表示同时能够接受最大请求数目。这个任务池主要是供线程池来访问。
- 线程池：这个是工作线程所在的集合，可以通过设置它的大小来提供并发处理的工作量。对于线程池的大小，可以事先生成一定数目的线程，根据实际情况来动态增加或者减少线程数目。线程池的大小不是越大越好，线程的切换也会耗时的。

### 线程池模式中的角色

 存放池的数据结构，可以用数组也可以利用集合，在集合类中一般使用Vector，这个是线程安全的。

![](http://ovcjgn2x0.bkt.clouddn.com/thread-design.jpg)

 Worker Thread的角色：

- Client参与者，发送Request的参与者
- Channel参与者，负责缓存Request的请求，初始化启动线程，分配工作线程
- Worker参与者，具体执行Request的工作线程
- Request参与者

在Worker线程内部等待任务池非空的方式称为正向等待。
在Channel线程提供Worker线程来判断任务池非空的方式称为反向等待。

## 实例应用
利用同步块来处理，Vector容器来存储客户端请求。利用Vector来存储，依旧是每次集合的最后一个位置添加请求，从开始位置移除请求来处理。在Channel有缓存请求方法和处理请求方法，利用生成者与消费者模式来处理存储请求，利用正向等待来判断任务池的非空状态。

这种实例，可以借鉴到网络ServerSocket处理用户请求的模式中，有很好的扩展性与实用性。
### Channel参与者

```java
public class Channel {
    public final static int THREAD_COUNT=4;
    public static void main(String[] args) {

      //注意：Map不是Collection的子类，都是java.util.*下的同级包
      Vector pool=new Vector();
      //工作线程，初始分配一定限额的数目
      WorkerThread[] workers=new WorkerThread[THREAD_COUNT];
                          
      //初始化启动工作线程
      for(int i = 0; i < workers.length; i++)
      {
          workers[i] = new WorkerThread(pool);
          workers[i].start();
      }
                           
      //接受新的任务，并且将其存储在Vector中
      Object task = new Object();//模拟的任务实体类
      //此处省略具体工作
      //在网络编程中，这里就是利用ServerSocket来利用ServerSocket.accept接受一个Socket从而唤醒线程
                           
      //当有具体的请求达到
      synchronized(pool)
      {
          pool.add(pool.size(), task);
          pool.notifyAll();//通知所有在pool wait set中等待的线程，唤醒一个线程进行处理
      }
      //注意上面这步骤添加任务池请求，以及通知线程，都可以放在工作线程内部实现
      //只需要定义该方法为static，在方法体用同步块，且共享的线程池也是static即可
                           
      //下面这步，可以有可以没有根据实际情况
      //取消等待的线程
      for(int i = 0; i < workers.length; i++)
      {
          workers[i].interrupt();
      }
    }
}
```
这个主要的作用如下

* 缓冲客户请求线程(利用生产者与消费者模式)
* 存储客户端请求的线程
* 初始化启动一定数量的线程
* 主动来唤醒处于任务池中wait set的一些线程来执行任务

定义两个集合，一个是存放客户端请求的，利用Vector；一个是存储线程的，就是线程池中的线程数目。
Vector是线程安全的，它实现了Collection和List，可以实现可增长的对象数组。
与数组一样，它包含可以使用整数索引进行访问的组件。但Vector 的大小可以根据需要增大或缩小，以适应创建 Vector 后进行添加或移除项的操作。

Collection中主要包括了list相关的集合以及set相关的集合，Queue相关的集合。

### 工作线程

```java
public class WorkerThread extends Thread {
    private List pool;//任务请求池
    private static int fileCompressed = 0;//所有实例共享的
                     
    public WorkerThread(List pool)
    {
          this.pool = pool; 
    }
                     
    //利用静态synchronized来作为整个synchronized类方法，仅能同时一个操作该类的这个方法
    private static synchronized void incrementFilesCompressed()
    {
        fileCompressed++;
    }
                     
    public void run()
    {
        while(true)
        {
            synchronized(pool)
            {
                //利用多线程设计模式中的
                //Guarded Suspension Pattern,警戒条件为pool不为空，否则无限的等待中
                while(pool.isEmpty())
                {
                    try{
                        pool.wait();//进入pool的wait set中等待着,释放了pool的锁
                    }catch(InterruptedException e)
                    {
                    }
                }                                 
                pool.remove(0);//获取任务池中的任务，并且要进行转换
            }
            //下面是线程所要处理的具体工作
        }
    }
}
```
在保证无限循环等待时，通过共享互斥来访问pool变量。当线程被唤醒，需要重新获取pool的锁，再次执行`synchronized`代码块中其余的工作；当不为空的时候，继续再判断是否为空，如果不为空，则跳出循环。必须先从任务池中移除一个任务来执行，统一用从末尾添加，从开始处移除


## 总结
线程池模式在实际编程中应用的很多，如数据库连接池，socket长连接等等。通过一定数量的工作者线程去执行不断被提交的任务，节约了线程这种有限且昂贵的资源。该模式有以下几个好处：

- 抵消线程创建的开销，提高响应性
- 封装了工作者线程生命周期管理
- 减少销毁线程的开销

但是要用好线程池模式，还需要考虑如下的问题：

- 工作队列的选择：通常有三种队列方式，有界队列（BoundedQueue）工作队列本身并不限制线程池中等待运行的任务的数量，但工作队列中实际可容纳的任务取决于任务本身对资源的使用情况；无界队列(UnboundQueue)工作队列限定线程池中等待大人物的数量，在一定成都上可以限制资源的消耗；直接交接队列(SymchrinousQueue)不适用缓冲空间内部提交任务的时候调用的是工作队列的非阻塞式入队列方法，所以没有等待队列，会有新的线程对入队列失败的任务进行处理。
- 线程池大小调参：太大了浪费资源，太大无法充分利用资源，所以线程池大小取决于该线程池所要处理任务的特性，系统资源以及任务锁使用的稀缺资源状况。
- 线程池监控：线程池的大小，工作队列的容量，线程空闲时间限制这些熟悉的调试过程需要有程序去监控来方便调试ThreadPoolExecutor类提供了监控的方法。
- 线程泄露：线程池中的工作者线程会意外终止，使得线程池中实际可用的工作者线程减少。出现的原因是线程对象的run方法的异常处理没有捕获RuntimeException和Error导致run方法意外返回，使得相应线程意外终止。所以要注入捕获相应异常。但是还有一种可能情况需要注意，如果线程需要请求外部资源而且对外部资源的请求没有时间限制的话，线程实际上可能已经泄露了。
- 可靠性和线程池饱和处理策略：工作队列的选择对于线程大小需求变化没有处理方式，所以需要线程饱和处理策略。
- 死锁：线程请求类似的资源可能形成死锁。
- 线程池空闲线程清理：过长时间没有进行任务处理的线程是对系统资源的浪费，所以需要相应的处理代码。

### 参考
本文主要参考，可以点击阅读原文：

1. [多线程设计模式——Thread Pool（线程池）模式](https://blog.csdn.net/buyoufa/article/details/51869942)
2. [Java多线程设计模式(4)线程池模式](http://blog.51cto.com/computerdragon/1205324)
