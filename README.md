# 日志系统Log
总体概况：利用单例模式与阻塞队列实现异步的日志系统，记录服务器运行状态
## 单例模式
保证整个系统中一个类只有一个对象的实例，实现这种功能的方式就叫单例模式。单例模式节省公共资源并且方便控制。

**实现思路：**

1. 构造私有：保证一个类不能多次被实例化，就要阻止对象被new出来，要私有化类的所有构造方法。
1. 以静态方式返回实例：因为外界不能通过new来获取对象， 所以我们要通过提供类的方法来让外界获取对象实例。
1. 确保对象实例只有一个：只对类进行一次实例化，以后都直接获取第一次实例化的对象。

## blockqueue.h
利用阻塞队列的模型进行信号量与互斥锁共同维护多线程日志异步操作，其中注意的点提一下：
首先是`std::lock_guard<std::mutex>`和`std::unique_lock<std::mutex>`的区别：
C++11中引入了std::unique_lock与std::lock_guard两种数据结构。通过对lock和unlock进行一次锁的封装，实现自动unlock的功能。
采用RAII手法管理mutex的std::lock_guard，其功能是在对象构造时将mutex加锁，析构时对mutex解锁，这样一个栈对象保证了在异常情形下mutex可以在lock_guard对象析构被解锁，lock_guard拥有mutex的所有权。

unique_lock 在使用上比lock_guard更具有弹性，和 lock_guard 相比，unique_lock 主要的特色在于：
unique_lock 不一定要拥有 mutex，所以可以透过 default constructor 建立出一个空的 unique_lock。
unique_lock 虽然一样不可复制（non-copyable），但是它是可以转移的（movable）。所以，unique_lock 不但可以被函数回传，也可以放到 STL 的 container 里。
另外，unique_lock 也有提供 lock()、unlock() 等函数，可以用来加锁解锁mutex，也算是功能比较完整的地方。
unique_lock本身还可以用于std::lock参数，因为其具备lock、unlock、try_lock成员函数,这些函数不仅完成针对mutex的操作还要更新mutex的状态。

值得一提的是，如果编译器支持C++17，`std::lock_guard<std::mutex>`可以被替换为`std::scoped_lock<...>`。后者具有类模板参数推导的特性，可以接受多个参数，与std::lock()一样可以在同时获取多个锁的时候防止死锁。

有关unique_lock资料可见C++并发编程实战第二版P60
有关scoped_lock资料可见C++并发编程实战第二版P52

# 线程池和数据库pool
**注意semaphore 、mutex 、condition_variable 的区别**
一：信号量 (semaphore)是一种轻量的同步原件，用于制约对共享资源的并发访问。在可以使用
两者时，信号量能比条件变量更有效率。

二：互斥(mutex)算法避免多个线程同时访问共享资源。这会避免数据竞争，并提供线程间的同步支持。

三：条件变量(condition_variable)是允许多个线程相互交流的同步原语。它允许一定量的线程等待（可以定时）另一线程的提醒，然后再继续。条件变量始终关联到一个互斥。

1. semaphore 对 acquire 和 release 操作没有限制，可以在不同线程操作；可以仅在线程 A 里面acquire,仅在线程 B 里面 release。mutex 的 lock 和 unlock 必须在同一个线程配对使用；也就是说线程 A 内 mutex 如果 lock了，必须在线程 A 内 unlock，线程 B 内 lock 了，也必须在线程 B 内 unlock。
1. semaphore 和 mutex 是可以独立使用的；condition_variable 必须和 mutex 配对使用。
1. semaphore 一般用于控制多个并发资源的访问或者控制并行数量;mutex 一般是起到同步访问一个资源的作用。同一时刻，mutex 保护的资源只能被一个线程访问；semaphore 的保护对象上面是可以有多个线程在访问的。mutex 是同步，semaphore 是并行。
1. 由于 condition_variable 和 mutex 结合使用，condition_variable 更多是为了通知、顺序之类的控制。
1. C++语言中的 mutex、semaphore、condition 和系统级的概念不同。都是线程级别的，也就是不能跨进程控制的。要区别于 windows api 的 mutex、semaphore、event。windows 系统上这几个 api 创建有名对象时，是进程级别的。

# 端口复用问题
如果两个套接字使用了同一个端口，那么将会导致 bind 失败。
端口复用允许一个应用程序把 n 个套接字绑定在一个端口上，**设置socket的SO_REUSEADDR选项，即可实现端口复用：**
```cpp
int opt = 1;
// sockfd为需要端口复用的套接字
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, (const void *)&opt, sizeof(opt));
```
需要注意的是，设置端口复用函数要在绑定之前调用，而且只要绑定到同一个端口的所有套接字都得设置复用。
要不然就换一个端口吗，大于1024小于65535都可以（用户端口，0~1023是系统端口）
# 缓冲区
利用标准容器库vector封装了char，实现了自动增长的缓冲区。
缓冲区标识出可写大小、可读大小、当前位置等信息，为文件读取工作提供便捷的操作入口，并提供重置、附加、取出、检查容量等操作。
缓冲区类私有成员readPos_和writePos_皆为原子变量， 防止并发操作时因冲突而产生脏数据。

# 边缘触发和水平触发
ET和LT分别是epoll句柄事件触发的两种模式：ET是一次事件只会触发一次，如一次客户端发来消息，fd可读，epoll_wait返回。等下次再调用epoll_wait则不会返回了；LT是一次事件会触发多次,如一次客户端发消息，fd可读，epoll_wait返回，不处理这个fd，再次调用epoll_wait立刻返回。
ET模式要搭配非阻塞fd，因为ET事件只触发一次， epoll_wait返回后一定要处理完毕 ；LT都可，但是LT模式下，可写状态的fd会一直触发事件。所以我们每次要写数据时，将fd绑定EPOLLOUT事件，写完后将fd同EPOLLOUT从epoll中移除。
> **EPOLLOUT事件：**
EPOLLOUT事件只有在连接时触发一次，表示可写，其他时候想要触发，那你要先准备好下面条件：
1.某次write，写满了发送缓冲区，返回错误码为EAGAIN。
2.对端读取了一些数据，又重新可写了，此时会触发EPOLLOUT。
简单地说：EPOLLOUT事件只有在不可写到可写的转变时刻，才会触发一次，所以叫边缘触发，这叫法没错的！  

ET效率更高，适合高并发的情况；LT模型简单，适合相对简洁的业务逻辑。因为ET在通知用户后，就会把fd从就绪队列里删除；而LT通知用户后fd还在就绪链表中，随着fd的增多，就绪链表越大。下次epoll要通知用户时还需要遍历整个就绪链表，遍历的性能是线性，如果fd的数量非常多，就会带来比较显著的效率下降。同样数量的fd下，LT模式维护的就绪链表比ET的大。

# 报文解析和上传
mmap用于将 一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。 举个例子理解一下，使用mmap方式获取磁盘上的文件信息，只需要将磁盘上的数据拷贝至那块共享内存中去。IO系统调用, 必须先把数据从磁盘拷贝至到**内核缓冲区中(页缓冲)**，然后再把数据拷贝用户进程中。两者相比，**mmap会少一次拷贝数据**，这样带来的性能提升是巨大的。  用户进程可以直接获取到信息，而相对于传统的write/read IO系统调用, 必须先把数据从磁盘拷贝至到**内核缓冲区中(页缓冲)**，然后再把数据拷贝至用户进程中。两者相比，**mmap会少一次拷贝数据**，这样带来的性能提升是巨大的。  

