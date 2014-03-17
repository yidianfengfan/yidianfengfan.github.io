---
layout: post
title: "bio nio aio"
date: 2014-03-17 18:15:06 +0800
comments: true
categories: 
---

### 按照《Unix网络编程》的划分，IO模型可以分为：阻塞IO、非阻塞IO、IO复用、信号驱动IO和异步IO，按照POSIX标准来划分只分为两类：同步IO和异步IO。如何区分呢？首先一个IO操作其实分成了两个步骤：发起IO请求和实际的IO操作，同步IO和异步IO的区别就在于第二个步骤是否阻塞，如果实际的IO读写阻塞请求进程，那么就是同步IO，因此阻塞IO、非阻塞IO、IO复用、信号驱动IO都是同步IO，如果不阻塞，而是操作系统帮你做完IO操作再将结果返回给你，那么就是异步IO。阻塞IO和非阻塞IO的区别在于第一步，发起IO请求是否会被阻塞，如果阻塞直到完成那么就是传统的阻塞IO，如果不阻塞，那么就是非阻塞IO.


bio
------------------------------
同步阻塞式IO,


nio
-----------------------------------
同步非阻塞IO, [Reactor](http://en.wikipedia.org/wiki/Reactor_pattern),从程序角度而言，当发起IO的读或写操作时，是非阻塞的；当socket有流可读或可写入socket时，操作系统会相应的通知引用程序进行处理，应用再将流读取到缓冲区或写入操作系统。对于网络IO而言，主要有连接建立、流读取及流写入三种事件.

#### select
```c
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select的一 个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但 是这样也会造成效率的降低。

####  poll
```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);



struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};

```
pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

从上面看，select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。

#### epool
```c
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
            typedef union epoll_data {
                void *ptr;
                int fd;
                __uint32_t u32;
                __uint64_t u64;
            } epoll_data_t;

            struct epoll_event {
                __uint32_t events;      /* Epoll events */
                epoll_data_t data;      /* User data variable */
            };

int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

 主要是epoll_create,epoll_ctl和epoll_wait三个函数。epoll_create函数创建epoll文件描述符，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。返回是epoll描述符。-1表示创建失败。epoll_ctl 控制对指定描述符fd执行op操作，event是与fd关联的监听事件。op操作有三种：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。epoll_wait 等待epfd上的io事件，最多返回maxevents个事件。

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知


aio
------------------
异步IO方式, [Proactor模式](http://en.wikipedia.org/wiki/Proactor_pattern),从程序的角度而言，与NIO不同，当进行读写操作时，只须直接调用API的read或write方法即可。这两种方法均为异步的，对于读操作而言，当有流可读取时，操作系统会将可读的流传入read方法的缓冲区，并通知应用程序；对于写操作而言，当操作系统将write方法传递的流写入完毕时，操作系统主动通知应用程序。较之NIO而言，AIO一方面简化了程序出的编写，流的读取和写入都由操作系统来代替完成；另一方面省去了NIO中程序要遍历事件通知队列（selector）的代价。Windows基于[IOCP](http://en.wikipedia.org/wiki/Input/output_completion_port）实现了AIO，Linux目前只有基于epoll实现的AIO



补充
-----------------------
#### 同步异步、阻塞非阻塞不是同一种概念，同步和异步是消息的通知机制的概念，阻塞非阻塞是指进程的状态。[see](http://www.cppblog.com/converse/archive/2009/05/13/82879.html)

1 阻塞非阻塞 [see](http://agio.iteye.com/blog/195548)

阻塞I/O, 当read数据时，如果这时没有数据可读，会一直等待有数据读，数据从kernel copy 到socket的buffer后返回；

非阻塞I/O, 当read数据时会立即返回. kernel copy数据到socket buffer后，会以事件等通知程序read操作完成。

关键就在于将数据从kernel copy到socket buffer这段时期，read操作是否阻塞，阻塞I/O是阻塞的，而异步I/O是不阻塞的。

- 异步与同步的区别：异步不会引起request进程阻塞。而同步会。
- 阻塞IO一定是同步的。
- 非阻塞IO分为两种：同步和异步的。
- 异步的非阻塞IO在read时，不管有无数据都直接返回。有数据了，会收到kernel copy的事件，来完成读取。
- 同步的非阻塞IO在read时，也是等数据copy完才返回。
- 由于非阻塞request不会一直与response保持连接，这样会降低response端压力，也就提高了并发性。


2 **同步和异步仅仅是关于所关注的消息如何通知的机制,而不是处理消息的机制.也就是说,同步的情况下,是由处理消息者自己去等待消息是否被触发,而异步的情况下是由触发机制来通知处理消息者** 所以在异步机制中,处理消息者和触发机制之间就需要一个连接的桥梁,在我们举的例子中这个桥梁就是小纸条上面的号码,而在select/poll等IO多路复用机制中就是fd,当消息被触发时,触发机制通过fd找到处理该fd的处理函数.

请注意理解**消息通知**和**处理消息**这两个概念,这是理解这个问题的关键所在.还是回到上面的例子,*轮到你办理业务*这个就是你关注的消息,而*去办理业务*就是对这个消息的处理,两者是有区别的.而在真实的IO操作时,所关注的消息就是该fd是否可读写,而对消息的处理就是对这个fd进行读写.同步/异步仅仅关注的是如何通知消息,它们对如何处理消息并不关心,好比说,银行的人仅仅通知你轮到你办理业务了,而如何办理业务他们是不知道的.

而很多人之所以把同步和阻塞混淆,我想也是因为没有区分这两个概念,比如阻塞的read/write操作中,其实是把消息通知和处理消息结合在了一起,在这里所关注的消息就是fd是否可读/写,而处理消息则是对fd读/写.当我们将这个fd设置为非阻塞的时候,read/write操作就不会在等待消息通知这里阻塞,如果fd不可读/写则操作立即返回.

很多人又会问了,异步操作不会是阻塞的吧?已经通知了有消息可以处理了就一定不是阻塞的了吧?
其实异步操作是可以被阻塞住的,只不过通常不是在处理消息时阻塞,而是在等待消息被触发时被阻塞.比如select函数,假如传入的最后一个timeout参数为NULL,那么如果所关注的事件没有一个被触发,程序就会一直阻塞在这个select调用处.而如果使用异步非阻塞的情况,比如aio_*组的操作,当我发起一个aio_read操作时,函数会马上返回不会被阻塞,当所关注的事件被触发时会调用之前注册的回调函数进行处理,具体可以参见我上面的连接给出的那篇文章.回到上面的例子中,如果在银行等待办理业务的人采用的是异步的方式去等待消息被触发,也就是领了一张小纸条,假如在这段时间里他不能离开银行做其它的事情,那么很显然,这个人被阻塞在了这个等待的操作上面;但是呢,这个人突然发觉自己烟瘾犯了,需要出去抽根烟,于是他告诉大堂经理说,排到我这个号码的时候麻烦到外面通知我一下(注册一个回调函数),那么他就没有被阻塞在这个等待的操作上面,自然这个就是异步+非阻塞的方式了.


[1](http://www.ibm.com/developerworks/cn/linux/l-async/)