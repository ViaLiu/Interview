# Linux网络编程

### 1、孤儿进程、僵尸进程和守护进程

当一个进程工作终止时，它的父进程需要调用wait()或waitpid()系统调用取得子进程的终止状态

孤儿进程：

​	一个父进程退出，它的子进程还在进行，那么这些子进程就变成孤儿进程，孤儿进程会被init进程收养。

僵尸进程：

​	一个进程使用fork创建子进程，如果子进程退出，而父进程并没有调用wait或waitpid获取子进程的状态信息，那么子进程的进程描述符仍然保存在系统中。这种进程称为僵尸进程。

守护进程：

​	守护进程是在后台运行且不与任何终端关联的进程，通常在系统引导装入时启动，仅在系统关闭时才终止。

### 2、进程间通信方式signal、file、pipe、shm、sem、msg、socket

**管道**：

​	半双工，父子进程之间通信。  

```
int pipe(int fd[2])  //fd[0]为读打开，fd[1]为写打开
```

**命名管道**：

​	半双工，允许不相关的进程间通信

**消息队列：**

​	消息队列是消息的链接表，存储在内核中，由消息队列标识符标识。

​	缺点：一个进程创建消息队列并放入几条消息，然后终止，消息队列不会被删除，直至某个进程读消息或删除消息。

```
int msgget(key_t key, int flag)   //创建新队列或打开现有队列
int msgsnd(int msgid,const void *ptr, size_t nbytes, int flags) //将新消息添加到队列中
```

**信号量：**

​	信号量是一个计数器，通过P、V操作来控制多个进程对共享资源的访问。

​	信号量大于0，进程可以使用共享资源，将值减1；若为0，进程进入休眠，直至大于0.

​	进程退出后将信号量增1。

```
int semget(key_t key, int nsems, int flags) //获得一个信号量
```

**共享内存：**

​	允许多个进程共享同一给定内存区域。通常使用信号量或者互斥锁控制访问。

**socket套接字**

​	它可用于不同机器间的进程通信。

### 3、线程间同步机制：互斥量、条件变量、信号量、读写锁、自旋锁

**互斥量**

​	本质上是一把锁，访问共享数据前对互斥量进行设置，确保同一时间只有一个线程访问数据。任何其他试图加锁的线程都会被阻塞。

```
int pthread_mutex_init(pthread_mutex_t *mutex, *attr)
int pthread_mutex_lock(pthread_mutex_t *mutex)
int pthread_mutex_trylock(pthread_mutex_t *mutex)
int pthread_mutex_unlock(pthread_mutex_t *mutex)
```

**读写锁**

​	三种状态：读模式下加锁状态、写模式下加锁状态、不加锁状态。适合读次数远大于写的情况

```
int pthread_rwlock_init(pthread_rwlock_t * rwlock, )  //初始化
int pthread_rwlock_rdlock(pthread_rwlock_t * rwlock)  //读加锁
int pthread_rwlock_wrlock(pthread_rwlock_t * rwlock)  //写加锁
int pthread_rwlock_unlock(pthread_rwlock_t * rwlock)  //解锁
```

**条件变量**

​	条件变量与互斥量一起使用时，允许线程以无竞争的方式等待特定的条件发生。

```
int pthread_cond_init(pthread_cond_t *cond, )       //初始化
int pthread_cond_destroy(pthread_cond_t *cond) 
```

​	 调用者需要把锁住的互斥量传给函数，函数然后自动把调用线程放到条件变量的线程列表上，对互斥量解锁

```
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex)  //函数返回时，互斥量再次被锁住
```

​	条件改变时，可以通过两个函数唤醒等待条件的线程

```
int pthread_cond_signal(pthread_cond_t *cond)  //至少唤醒一个等待的线程
int pthread_cond_broadcast(pthread_cond_t *cond) //唤醒所有等待条件的线程
```

**自旋锁**

​	自旋锁不是通过休眠使进程阻塞，而是在获取锁之前一直处于忙等阻塞状态。

​	相对于互斥锁，少两次线程上下文切换。但会一直占用CPU。

### 4、fork返回值

子进程返回0,子进程可通过getppid获得父进程的ID

而父进程返回新建子进程的进程ID。父进程可能有多个子进程，且无其他函数获得子进程ID。

### 5、五大IO模型：阻塞IO、非阻塞IO、I/O复用、信号驱动IO、异步IO

**阻塞IO**：系统调用直到获得结果或者出错才返回结果。

**非阻塞IO**：当请求IO操作时，数据包还没有准备好，它不会阻塞进程，而是立即返回错误。

```
fcntl(fd,F_SETFL,O_NONBLOCK)
```

**I/O复用**：阻塞在select、poll之上，等待数据包套接字可读，返回可读条件，再调用I/O系统调用

**信号驱动IO**：使用信号，让内核再描述符就绪时发送SIGIO信号通知。（开启信号驱动式IO功能，通过sigaction系统调用安装一个信号处理函数）

**异步IO**：内核自动执行某个操作，在操作完成后通知我们。

### 6、select、poll、epoll

#### select

```
fd_set rset;   FD_ZERO(&rset); FD_SET(1,&rset);
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, timeval *timeout);
FD_ISSET(fd,&rset);
```

​	select调用过程:（文件描述符限制1024）

- 使用copy_from_user从用户空间拷贝fd_set到内核空间
- 注册回调函数_pollwait
- 遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）
- __pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。

- poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。

- 把fd_set从内核空间拷贝到用户空间。

  

  

  **select机制的缺点**

  - 每次调用select，都需要把监听的文件描述符集合fd_set从用户态拷贝到内核态，从算法角度来说就是O(n)的时间开销。

  - 每次调用select调用返回之后都需要遍历所有文件描述符，判断哪些文件描述符有读写事件发生，这也是O(n)的时间开销。

  - 内核对被监控的文件描述符集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)。

    

##### poll

poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构。

- 它将用户传入的数组拷贝到内核空间

- 然后查询每个fd对应的设备状态：如果就绪，在设备等待队列中加入一项继续遍历；

   如果遍历完后，未发现就绪设备，挂起进程，知道设备就绪或者主动超时，被唤醒后，它会再次遍历。

- 结束后，又把进程从各个等待队列中删除。

```
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

```
typedef struct pollfd {
        int fd;                        
        short events;                   
        short revents;                  
} pollfd_t;
```

##### epoll

​	epoll由三组函数组成：

```
int epoll_create(int size)  //创建一个文件描述符，来唯一标识内核的事件表
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)//注册、修改、添加事件。
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout)//一段超时时间内等待一组文件描述符上的事件。
```

调用epoll_create方法，内核调用sys_epoll_create()，创建新的inode，file和fd，再调用ep_file_init(file),创建一个struct eventpoll结构，并把它放入file->private_data;再通过eventpoll_init()注册并挂载新的一个文件系统

```
 static int __init eventpoll_init(void)
 {
 	epi_cache = kmem_cache_create("eventpoll_epi", sizeof(struct epitem),
 					0, SLAB_HWCACHE_ALIGN|EPI_SLAB_DEBUG|SLAB_PANIC, NULL, NULL);

 	pwq_cache = kmem_cache_create("eventpoll_pwq",
 					sizeof(struct eppoll_entry), 0, EPI_SLAB_DEBUG|SLAB_PANIC, NULL, NULL);
 					//注册了一个新的文件系统，叫"eventpollfs"

 	error = register_filesystem(&eventpoll_fs_type);  
    eventpoll_mnt = kern_mount(&eventpoll_fs_type);   //挂载文件系统
 }
 
 asmlinkage long sys_epoll_create(int size)
 {
     int error, fd;
     struct inode *inode;
     struct file *file;
     error = ep_getfd(&fd, &inode, &file);
     /* Setup the file internal data structure ( "struct eventpoll" ) */
     error = ep_file_init(file);

}
```

![img](.\28541347_1399302858cZNB.jpg)



**epoll_ctl**  ：就是在（struct eventpoll）里先ep_find，如果找到了struct epitem,而根据用户操作是ADD、DEL、MOD调用相应的函数，这些函数在epitem组成红黑树中增加、删除、修改相应节点（每一个监听fd对应一个节点）

​	ep_insert做的最重要的事创建一个eppoll_entry,设置其唤醒回调函数ep_poll_callback，然后加入等待队列中。这样，设备就绪唤醒等待进程时，回调函数被调用，把红黑树上收到的event的epitem（代表一个fd）插入rdllist。这样，调用epoll_wait时只用收集rdllist里的fd就可以了。



```
 asmlinkage long  sys_epoll_ctl(int epfd, int op, int fd, struct epoll_event __user *event)
 {
 	struct file *file, *tfile;
 	struct eventpoll *ep;
 	struct epitem *epi;
 	struct epoll_event epds;
		....
 	epi = ep_find(ep, tfile, fd);//tfile存放要监听的fd对应在rb-tree中的epitem
 	switch (op) {//省略了判空处理
 		case EPOLL_CTL_ADD: epds.events |= POLLERR | POLLHUP;
 			error = ep_insert(ep, &epds, tfile, fd); break;
 		case EPOLL_CTL_DEL: error = ep_remove(ep, epi); break;
 		case EPOLL_CTL_MOD: epds.events |= POLLERR | POLLHUP;
		    error = ep_modify(ep, epi, &epds); break;
 }
 static int ep_insert(struct eventpoll *ep, struct epoll_event *event, struct file *tfile, int fd)
{

	struct epitem *epi;
 	struct ep_pqueue epq;// 创建ep_pqueue对象
	epi = EPI_MEM_ALLOC();//分配一个epitem
	/* 初始化这个epitem ... */
 	epi->ep = ep;//将创建的epitem添加到传进来的struct eventpoll

	/*后几行是设置epitem的相应字段*/
 	EP_SET_FFD(&epi->ffd, tfile, fd);//将要监听的fd加入到刚创建的epitem
 	epi->event = *event;
 	epi->nwait = 0;

	/* Initialize the poll table using the queue callback */
	epq.epi = epi;  //将一个epq和新插入的epitem(epi)关联

	//下面一句等价于&(epq.pt)->qproc = ep_ptable_queue_proc;

	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

	revents = tfile->f_op->poll(tfile, &epq.pt);  //tfile代表target file，即被监听的文件,poll()返回就绪事												件的掩码，赋给revents.

	list_add_tail(&epi->fllink, &tfile->f_ep_links);// 每个文件会将所有监听自己的epitem链起来

	ep_rbtree_insert(ep, epi);// 都搞定后, 将epitem插入到对应的eventpoll中去
	……
}
```

**epoll_wait()**:从file->private_data中拿到eventpoll，再调用ep_poll

```
[fs/eventpoll.c-->sys_epoll_wait()->ep_poll()]
 static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout)
 {
     int res;
     wait_queue_t wait;                      //等待队列项
     if (list_empty(&ep->rdllist)) {
     	//ep->rdllist存放的是已就绪(read)的fd，为空时说明当前没有就绪的fd，所以需要将当前
     	init_waitqueue_entry(&wait, current);     //创建一个等待队列项，并使用当前进程（current）初始化
     	add_wait_queue(&ep->wq, &wait);           //将刚创建的等待队列项加入到ep中的等待队列
     	                                           （即将当前进程添加到等待队列）
     	for (;;) {
     		/*将进程状态设置为TASK_INTERRUPTIBLE，因为我们不希望这期间ep_poll_callback()发信号唤醒进程的时               候，进程还在sleep */
     		set_current_state(TASK_INTERRUPTIBLE);
     		if (!list_empty(&ep->rdllist) || !jtimeout)   break;
     		//如果ep->rdllist非空(即有就绪的fd)或时间到则跳出循环
   			if (signal_pending(current)) {
     			res = -EINTR;
     			break;
     		}
     	}
     remove_wait_queue(&ep->wq, &wait);     //将等待队列项移出等待队列(将当前进程移出)
     set_current_state(TASK_RUNNING);
 }
```

### 7、epoll的LT水平触发和ET边缘触发





### 8、Reactor和Proactor模式

###### Reactor：同步IO实现

主线程（IO处理单元）只负责监听socket文件描述符上是否有事件发生，有的话就立即将该事件通知工作线程(逻辑处理单元)。除此之外，主线程不做任何其他实质性的工作，读写数据、接受新的连接请求、处理客户端请求均在工作线程中完成。 
  以epoll_wait()为例实现的Reactor模式的工作流程为(所有IO复用技术都是基于同步IO模型)： 
  （1）主线程往epoll内核事件表中注册socket的可读事件(就绪读)，进而调用epoll_wait()监听socket上的可读事件。
  （2）当socket上有数据可读时(含有客户端来连接)，epoll_wait()通知主线程，主线程将socket的可读事件放入请求队列。 
  （3）唤醒在请求队列上的某个工作线程，工作线程从socket读取数据，并处理请求，然后注册写就绪事件

​		（4）主线程等待socket写就绪事件.可写时将其放入请求队列

​		（5）请求队列上的某个工作线程被唤醒，往该socket上写入请求的处理结果。

###### Proactor：异步IO实现

Proactor模式是将所有IO事件操作都交由主线程和内核处理，工作线程只负责业务逻辑。

​		（1）主线程调用aio_read()函数向内核注册sockst上的读完成事件，并告诉内核在用户空间的读缓冲区的位置，以及读操作完成时如何通知应用程序（比如信号机制）。 
  （2）主线程继续处理业务逻辑。 
  （3）当socket上的数据被读入用户缓冲区后（读操作交由内核完成），内核将向应用程序发送信号以通知应用程序数据可用。 
  （4）应用程序在预先定义的信号处理函数中选择一个工作线程来处理客户需求。工作线程处理完客户请求后调用aio_write()函数向内核注册socket上的写完成事件，并告诉内核用户写缓冲区的位置以及写操作完成时如何通知应用程序，即写操作交由内核执行。 
  （5）主线程继续处理其他业务逻辑。 
  （6）当用户缓冲区的数据被内核写入socket后内核向应用程序发送一个信号以通知应用程序发送完毕。 
  （7）应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，如决定是否关闭socket连接。 
内核负责监听读写事件且执行读写操作，读写完毕后再通过信号机制向应用程序报告，应用程序的信号处理函数完成后续操作。



### 9、反向代理和负载均衡



### 10、为什么要有timewait状态，怎么处理大量的Timewait状态

（1）可靠地实现TCP全双工连接的终止。

​			若最终的ACK丢失，服务器将重发它最终的FIN，维持TIMEWAIT状态才能可靠地终止TCP连接

（2）允许老的重复分节在网络中消逝。

​		  关闭一个TCP连接后，若新建立的连接有相同的IP地址和端口，维持2MSL的timewait状态可以保证老的重复分组报文已经在网络中消逝。

若出现大量的timewait状态。解决办法如下：

- 修改内核参数   vi /etc/sysctl.conf

```
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
```

- 调整短连接为长连接

### 11、Linux中如何实现Signal？

基于软中断，不同的Signal对应不同处理函数

### 12、kill -9发生了什么？

- kill pid是向进程号为pid的进程发送SIGTERM,该信号是一个结束进程的信号且可以被应用程序捕获。若应用程序没有捕获并响应该信号的逻辑代码，则该信号的默认动作是kill掉进程。这是终止指定进程的推荐做法。
- kill -9 pid 则是向进程号为pid的进程发送SIGKILL（该信号编号为9），SIGKILL不能被应用程序捕获，也不能被阻塞或忽略，其动作是立即结束进程。SIGKILL信号是直接发送给init进程的，它收到信号后，负责终止pid指定的进程。

### 13、服务器主机崩溃

​		客户端在给服务器发送数据时，由于收不到服务器端回传的ACK确认报文，正常情况下，客户端TCP均会进行超时重传，一般为重传12次大约9分钟后才放弃重传，并关闭客户端TCP链接。

### 14、服务器主机崩溃后重启

​		如果服务器主机在崩溃重启的这段时间里，客户端没有向服务器发送数据，即客户端没有因重传次数超过限制关闭TCP链接。则在服务器重启后，当客户端再向服务器发送TCP报文时，由于服务器中的TCP链接已经关闭，会直接向客户端回复RST报文，客户端在接收RST报文后关闭自己的TCP链接。

### 15、服务器主机断网或者中间路由器出现故障

​		与情况1类似，客户端会进行超时重传，直到重传次数超过后放弃重传，并关闭客户端TCP链接。(因为TCP中会忽略目的主机不可达和目的网络不可达的ICMP报文，并进行重传，直到重传总时间超过限制)

### 16、服务器主机断网或者中间路由器出现故障后又恢复

​		如果在服务器主机断网或者中间路由器出现故障这段时间内，客户端和服务器之间没有进行相互通信，即双方均没有察觉对方目的不可达，则在恢复网络链接后两端的TCP链接均有效，能够正常继续进行通信。

  如果在服务器主机断网或者中间路由器出现故障这段时间内，客户端因向服务器发送数据超时，并重传总时间超过限制关闭TCP链接。则再网络恢复后，服务器再向客户端发送TCP报文时，客户端也会直接回复RST报文，服务器再收到RST报文后关闭自己的TCP连接。

​	



# STL面试题汇总

### 1、vector和array有什么区别？

- array是静态空间，空间大小配置后不能改变
- vector是动态空间，

### 2、vector、list、deque都是什么，怎么实现的？有什么区别?

### 3、智能指针（unique_ptr、shared_ptr、weak_ptr）,unique_ptr如何传递给函数

### 4、map和unordered_map, set和unordered_set?

unordered_map是基于哈希实现的，查找和插入开销都是O(1)，而map是基于红黑树实现的，查找和插入的开销都是O(logn)。

set和unordered_set同理。

map和set区别在于：

（1）map中的元素是key-value（关键字—值）对：关键字起到索引的作用，值则表示与索引相关联的数据；Set与之相对就是关键字的简单集合，set中每个元素只包含一个关键字。

（2）set的迭代器是const的，不允许修改元素的值；map允许修改value，但不允许修改key。其原因是因为map和set是根据关键字排序来保证其有序性的，如果允许修改key的话，那么首先需要删除该键，然后调节平衡，再插入修改后的键值，调节平衡，如此一来，严重破坏了map和set的结构，导致iterator失效，不知道应该指向改变前的位置，还是指向改变后的位置。所以STL中将set的迭代器设置成const，不允许修改迭代器的值；而map的迭代器则不允许修改key值，允许修改value值。

（3）map支持下标操作，set不支持下标操作。map可以用key做下标，map的下标运算符[ ]将关键码作为下标去执行查找，如果关键码不存在，则插入一个具有该关键码和mapped_type类型默认值的元素至map中，因此下标运算符[ ]在map应用中需要慎用，const_map不能用，只希望确定某一个关键值是否存在而不希望插入元素时也不应该使用，mapped_type类型没有默认值也不应该使用。如果find能解决需要，尽可能用find。

5、哈希实现原理和哈希冲突解决办法

6、shared_ptr时线程安全的吗？可以指向数组吗？





# C++面试题汇总

#### 1、define和const区别

- define在编译预处理阶段起作用，而const是在编译、运行的时候起作用

- define不会做类型检查，const拥有类型，会执行相应的类型检查。
- define仅仅是宏替换，不占⽤内存，⽽const会占用内存
- const内存效率更高，编译器通常将const变量保存在符号表中，而不会分配存储空间，这使得它成 为一个编译期间的常量，没有存储和读取的操作

#### 2、如何定义一个只能在堆上（栈上）生成对象的类

- 只能建立在堆上：将析构函数设为私有，对象建立在栈上时，由编译器分配内存空间，它会检查析构函数访问性。

- 只能建立在栈上：将new和delete重载为私有。

#### 3、new和malloc区别

- new可以认为是malloc加构造函数的执行。new出来的指针是直接带类型信息的。而malloc返回的都是void指针
- new/delete是**C++运算符，需要**编译器**支持。malloc/free是**库函数**，需要头文件支持
- .malloc分配内存前需要手工计算分配多大空间，new能自动计算需要分配的内存空间
- malloc和free是c++/c语言的**标准库函数**；而new和delete是c++的**运算符**，对于非内部数据类的对象而言，光用malloc无法满足动态对象的要求，因为对象在创建的时候要执行构造函数，对象在消亡之前要执行析构函数。由于malloc是库函数而不是运算符，不在编译器控制权限之内，不能把执行构造函数和析构函数的任务强加malloc

#### 4、C++11新特性



#### 5、虚函数实现



#### 6、c++和C的区别

设计思想上：

​	C++是面向对象的语言，而C是面向过程的结构化编程语言

语法上：

​    C++具有封装、继承和多态三种特性

​	C++相比C，增加多许多类型安全的功能，比如强制类型转换、

​	C++支持范式编程，比如模板类、函数模板等

#### 7、指针和引用的区别

1、指针有自己的一块空间，而引用只是一个别名；

2、使用sizeof看一个指针的大小是4，而引用则是被引用对象的大小；

3、指针可以被初始化为NULL，而引用必须被初始化且必须是一个已有对象的引用；

4、作为参数传递时，指针需要被解引用才可以对对象进行操作，而直接对引用的修改都会改变引用所指向的对象；

5、可以有const指针，但是没有const引用；

6、指针在使用中可以指向其它对象，但是引用只能是一个对象的引用，不能 被改变；

7、指针可以有多级指针（**p），而引用至于一级；

8、指针和引用使用++运算符的意义不一样；

9、如果返回动态内存分配的对象或者内存，必须使用指针，引用可能引起内存泄露。

#### 8、虚函数表什么时候创建、在哪创建

#### 9、C++内存管理机制



![img](.\311436_1552467921124_13956548C4BB199139A2744C39350272)

32bitCPU可寻址4G线性空间，每个进程都有各自独立的4G逻辑地址，其中0~3G是用户态空间，3~4G是内核空间，不同进程相同的逻辑地址会映射到不同的物理地址中。其逻辑地址其划分如下：

各个段说明如下：

3G用户空间和1G内核空间

静态区域：

text segment(代码段):包括只读存储区和文本区，其中只读存储区存储字符串常量，文本区存储程序的机器代码。

data segment(数据段)：存储程序中已初始化的全局变量和静态变量

bss segment：存储未初始化的全局变量和静态变量（局部+全局），以及所有被初始化为0的全局变量和静态变量，对于未初始化的全局变量和静态变量，程序运行main之前时会统一清零。即未初始化的全局变量编译器会初始化为0

动态区域：

heap（堆）： 当进程未调用malloc时是没有堆段的，只有调用malloc时采用分配一个堆，并且在程序运行过程中可以动态增加堆大小(移动break指针)，从低地址向高地址增长。分配小内存时使用该区域。  堆的起始地址由mm_struct 结构体中的start_brk标识，结束地址由brk标识。

memory mapping segment(映射区):存储动态链接库等文件映射、申请大内存（malloc时调用mmap函数）

stack（栈）：使用栈空间存储函数的返回地址、参数、局部变量、返回值，从高地址向低地址增长。在创建进程时会有一个最大栈大小，Linux可以通过ulimit命令指定。

#### 10、野指针和空悬指针

野指针就是指向一个已删除的对象或者未申请访问受限内存区域的指针

#### 11、构造函数可以是私有的吗？析构函数呢？

​	构造函数声明为私有的。则类不能被外部实例化。只能在栈上创建对象（还可以屏蔽掉new和delete）

​	**析构函数**声明为私有的，则类只能在堆上分配内存。

​	如果在栈上分配空间，类在离开作用域时会调用析构函数释放空间，此时无法调用私有的析构函数。

​	如果在堆上分配空间，只有在delete时才会调用析构函数。

#### 12、const

（1）欲阻止一个变量被改变，可以使用const关键字。在定义该const变量时，通常需要对它进行初始化，因为以后就没有机会再去改变它了； 
（2）对指针来说，可以指定指针本身为const，也可以指定指针所指的数据为const，或二者同时指定为const； 
（3）在一个函数声明中，const可以修饰形参，表明它是一个输入参数，在函数内部不能改变其值； 
（4）对于类的成员函数，若指定其为const类型，则表明其是一个常函数，不能修改类的 成员变量； 
（5）对于类的成员函数，有时候必须指定其返回值为const类型，以使得其返回值不为“左值”。例如： 

#### 13、struct和class区别

C++中，可以用struct和class定义类，都可以继承。区别在于：struct的默认继承权限和默认访问权限是public，而class的默认继承权限和默认访问权限是private。

另外，class还可以定义模板类形参，比如template <class T, int i>。

#### 14、C++如何强行访问私有成员？

- ​	加一句#define private public 欺骗编译器
- ​    构造一个与A布局相同的类B，将private改成public，将A实例的指针转换成B来访问私有成员（直接访问地址）。

#### 15、C++多态



#### 16、静态链接和动态链接

##### 17、static关键字作用

（1）函数体内static变量的作用范围为该函数体，不同于auto变量，该变量的内存只被分配一次，因此其值在下次调用时仍维持上次的值； 
（2）在模块内的static全局变量可以被模块内所用函数访问，但不能被模块外其它函数访问； 
（3）在模块内的static函数只可被这一模块内的其它函数调用，这个函数的使用范围被限制在声明它的模块内； 
（4）在类中的static成员变量属于整个类所拥有，对类的所有对象只有一份拷贝； 
（5）在类中的static成员函数属于整个类所拥有，这个函数不接收this指针，因而只能访问类的static成员变量。 

##### 18、C++中成员函数能够同时用static和const进行修饰？

否，因为static表⽰示该函数为静态成员函数，为类所有；而const是用于修饰成员函数的，两者相矛盾

##### 19、智能指针



# 计算机网络



### 1、TCP可靠性是如何保证的



==TCP协议保证数据传输可靠性的方式主要有：==

- 校验和
- 序列号
- 确认应答
- 超时重传
- 连接管理
- 流量控制
- 拥塞控制

**校验和**

计算方式：在数据传输的过程中，将发送的数据段都当做一个16位的整数。将这些整数加起来。并且前面的进位不能丢弃，补在后面，最后取反，得到校验和。 
发送方：在发送数据之前计算检验和，并进行校验和的填充。 
接收方：收到数据后，对数据以同样的方式进行计算，求出校验和，与发送方的进行比对。

![img](G:/刘强/面试题汇总/面试&面经总结.assets/20180524102010286)

**确认应答与序列号**

序列号：TCP传输时将每个字节的数据都进行了编号，这就是序列号。 
确认应答：TCP传输的过程中，每次接收方收到数据后，都会对传输方进行确认应答。也就是发送ACK报文。这个ACK报文当中带有对应的确认序列号，告诉发送方，接收到了哪些数据，下一次的数据从哪里发。

![img](G:/刘强/面试题汇总/面试&面经总结.assets/20180524103121705)

序列号的作用不仅仅是应答的作用，有了序列号能够将接收到的数据根据序列号排序，并且去掉重复序列号的数据。这也是TCP传输可靠性的保证之一。

**超时重传**

在进行TCP传输时，由于确认应答与序列号机制，也就是说发送方发送一部分数据后，都会等待接收方发送的ACK报文，并解析ACK报文，判断数据是否传输成功。如果发送方发送完数据后，迟迟没有等到接收方的ACK报文，这该怎么办呢？而没有收到ACK报文的原因可能是什么呢？

首先，发送方没有接收到响应的ACK报文原因可能有两点：

1. 数据在传输过程中由于网络原因等直接全体丢包，接收方根本没有接收到。
2. 接收方接收到了响应的数据，但是发送的ACK报文响应却由于网络原因丢包了。

TCP在解决这个问题的时候引入了一个新的机制，叫做超时重传机制。简单理解就是发送方在发送完数据后等待一个时间，时间到达没有接收到ACK报文，那么对刚才发送的数据进行重新发送。如果是刚才第一个原因，接收方收到二次重发的数据后，便进行ACK应答。如果是第二个原因，接收方发现接收的数据已存在（判断存在的根据就是序列号，所以上面说序列号还有去除重复数据的作用），那么直接丢弃，仍旧发送ACK应答。

那么发送方发送完毕后等待的时间是多少呢？如果这个等待的时间过长，那么会影响TCP传输的整体效率，如果等待时间过短，又会导致频繁的发送重复的包。如何权衡？

由于TCP传输时保证能够在任何环境下都有一个高性能的通信，因此这个最大超时时间（也就是等待的时间）是动态计算的。

> 在Linux中（BSD Unix和Windows下也是这样）超时以500ms为一个单位进行控制，每次判定超时重发的超时时间都是500ms的整数倍。重发一次后，仍未响应，那么等待2*500ms的时间后，再次重传。等待4*500ms的时间继续重传。以一个指数的形式增长。累计到一定的重传次数，TCP就认为网络或者对端出现异常，强制关闭连接。

**连接管理**

连接管理就是三次握手与四次挥手的过程，在前面详细讲过这个过程，这里不再赘述。保证可靠的连接，是保证可靠性的前提。

### **流量控制**

#### 为什么要使用流量控制

双方在通信的时候，发送方的速率与接收方的速率是不一定相等，如果发送方的发送速率太快，会导致接收方处理不过来，这时候接收方只能把处理不过来的数据存在缓存区里（失序的数据包也会被存放在缓存区里）。

如果缓存区满了发送方还在疯狂着发送数据，接收方只能把收到的数据包丢掉，大量的丢包会极大着浪费网络资源，因此，我们需要控制发送方的发送速率，让接收方与发送方处于一种动态平衡才好。

对发送方发送速率的控制，我们称之为流量控制。

![img](G:/刘强/面试题汇总/面试&面经总结.assets/640)

#### 如何控制?

接收方每次收到数据包，可以在发送确定报文的时候，同时告诉发送方自己的缓存区还剩余多少是空闲的，我们也把缓存区的剩余大小称之为接收窗口大小，用变量win来表示接收窗口的大小。

发送方收到之后，便会调整自己的发送速率，也就是调整自己发送窗口的大小，当发送方收到接收窗口的大小为0时，发送方就会停止发送数据，防止出现大量丢包情况的发生。

![img](G:/刘强/面试题汇总/面试&面经总结.assets/640)

#### 发送方何时再继续发送数据?

我们可以采用这样的策略：当接收方处理好数据，接受窗口 win > 0 时，接收方发个通知报文去通知发送方，告诉他可以继续发送数据了。当发送方收到窗口大于0的报文时，就继续发送数据。

不过这时候可能会遇到一个问题，假如接收方发送的通知报文，由于某种网络原因，这个报文丢失了，这时候就会引发一个问题：接收方发了通知报文后，继续等待发送方发送数据，而发送方则在等待接收方的通知报文，此时双方会陷入一种僵局。

为了解决这种问题，我们采用了另外一种策略：当发送方收到接受窗口 win = 0 时，这时发送方停止发送报文，并且同时开启一个定时器，每隔一段时间就发个测试报文去询问接收方，打听是否可以继续发送数据了，如果可以，接收方就告诉他此时接受窗口的大小；如果接受窗口大小还是为0，则发送方再次刷新启动定时器。

![img](G:/刘强/面试题汇总/面试&面经总结.assets/640)

### **拥塞控制**

==拥塞控制的方法主要有四种：慢启动、拥塞避免、快重传、快恢复====  

#### **慢开始与拥塞避免**

发送方维持一个叫做**拥塞窗口cwnd（congestion window）**的状态变量。拥塞窗口的大小取决于网络的拥塞程度，并且动态地在变化。发送方让自己的发送窗口等于拥塞窗口，另外考虑到接受方的接收能力，发送窗口可能小于拥塞窗口。

慢开始算法的思路就是，不要一开始就发送大量的数据，先探测一下网络的拥塞程度，也就是说由小到大逐渐增加拥塞窗口的大小。

这里用报文段的个数的拥塞窗口大小举例说明慢开始算法，实时拥塞窗口大小是以字节为单位的。如下图：

![img](G:/刘强/面试题汇总/面试&面经总结.assets/20130801220358468)

当然收到单个确认但此确认多个数据报的时候就加相应的数值。所以一次传输轮次之后拥塞窗口就加倍。这就是乘法增长，和后面的拥塞避免算法的加法增长比较。

为了防止cwnd增长过大引起网络拥塞，还需设置一个慢开始门限ssthresh状态变量。ssthresh的用法如下：

**当cwnd<ssthresh时，使用慢开始算法。**

**当cwnd>ssthresh时，改用拥塞避免算法。**

**当cwnd=ssthresh时，慢开始与拥塞避免算法任意。**

拥塞避免算法让拥塞窗口缓慢增长，即每经过一个往返时间RTT就把发送方的拥塞窗口cwnd加1，而不是加倍。这样拥塞窗口按线性规律缓慢增长。

​    无论是在**慢开始阶段**还是在**拥塞避免阶段**，只要发送方判断网络出现拥塞（其根据就是没有收到确认，虽然没有收到确认可能是其他原因的分组丢失，但是因为无法判定，所以都当做拥塞来处理），就把慢开始门限设置为出现拥塞时的发送窗口大小的一半。然后把拥塞窗口设置为1，执行慢开始算法。如下图：

![img](G:/刘强/面试题汇总/面试&面经总结.assets/20180524125815394)

#### 快重传和快恢复

快重传要求接收方在收到一个失序的报文段后就立即发出重复确认（为的是使发送方及早知道有报文段没有到达对方）而不要等到自己发送数据时捎带确认。快重传算法规定，发送方只要一连收到三个重复确认就应当立即重传对方尚未收到的报文段，而不必继续等待设置的重传计时器时间到期。如下图：

![img](G:/刘强/面试题汇总/面试&面经总结.assets/20130801220556750)

快重传配合使用的还有快恢复算法，有以下两个要点:

①当发送方连续收到三个重复确认时，就执行“乘法减小”算法，把ssthresh门限减半。但是接下去并不执行慢开始算法。

②考虑到如果网络出现拥塞的话就不会收到好几个重复的确认，所以发送方现在认为网络可能没有出现拥塞。所以此时不执行慢开始算法，而是将cwnd设置为ssthresh的大小，然后执行拥塞避免算法。如下图：

![img](G:/刘强/面试题汇总/面试&面经总结.assets/20130801220615250)

拥塞控制是TCP在传输时尽可能快的将数据传输，并且避免拥塞造成的一系列问题。是可靠性的保证，同时也是维护了传输的高效性。

## TCP滑动窗口

**TCP头里有一个字段叫Window，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。**于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来。

下面我们来看一下发送方的滑动窗口示意图：

![img](G:/刘强/面试题汇总/面试&面经总结.assets/9c002a0f-5e43-318e-99f7-be4f913fe760.png)

上图中分成了四个部分，分别是：（其中那个**黑模型就是滑动窗口**）

#1已收到ack确认的数据。
#2已发出但还没收到ack的。
#3在窗口中还没有发出的（接收方还有空间）。
#4窗口以外的数据（接收方没空间）

==**滑动窗口里是已发出但未收到ACk和还未发出的数据**==

下面是个滑动后的示意图（收到36的ack，并发出了46-51的字节）：

![img](G:/刘强/面试题汇总/面试&面经总结.assets/8d39063f-73e0-39d3-b4a5-d1ef33e7c0ba.png)

## 三次握手

**三次握手过程：**

![image-20210302155735466](G:/刘强/面试题汇总/面试&面经总结.assets/image-20210302155735466.png)

另外需要注意的是，从图中可以看出，SYN 是需要消耗一个序列号的，下次发送对应的 ACK 序列号要加1，为什么呢？只需要记住一个规则:**凡是需要对端确认的，一定消耗TCP报文的序列号。SYN 需要对端的确认， 而 ACK 并不需要，因此 SYN 消耗一个序列号而 ACK 不需要。**

### 为什么进行第三次握手

3次握手完成两个重要的功能，既要双方做好发送数据的准备工作(双方都知道彼此已准备好)，也要允许双方就初始序列号进行协商，这个序列号在握手过程中被发送和确认。现在把三次握手改成仅需要两次握手，死锁是可能发生的。作为例子，考虑计算机S和C之间的通信，假定C给S发送一个连接请求分组，S收到了这个分组，并发送了确认应答分组。

### 三次握手时第三个ACK包丢了怎么办

==**Server 端**==：第三次的ACK在网络中丢失，那么Server 端该TCP连接的状态为SYN_RECV,并且会根据 TCP的超时重传机制，会等待3秒、6秒、12秒后重新发送SYN+ACK包，以便Client重新发送ACK包。而Server重发SYN+ACK包的次数，可以通过设置/proc/sys/net/ipv4/tcp_synack_retries修改，默认值为5.如果重发指定次数之后，仍然未收到 client 的ACK应答，那么一段时间后，Server自动关闭这个连接。

**==Client 端：==**在linux c 中，client 一般是通过 connect() 函数来连接服务器的，而connect()是在 TCP的三次握手的第二次握手完成后就成功返回值。也就是说 client 在接收到 SYN+ACK包，它的TCP连接状态就为 established （已连接），表示该连接已经建立。那么如果 第三次握手中的ACK包丢失的情况下，Client 向 server端发送数据，Server端将以 RST包响应，方能感知到Server的错误。

### 为什么不是两次握手？

当客户端向服务器端发送一个连接请求时，由于某种原因长时间驻留在网络节点中，无法达到服务器端，由于 TCP 的超时重传机制，当客户端在特定的时间内没有收到服务器端的确认应答信息，则会重新向服务器端发送连接请求，且该连接请求得到服务器端的响应并正常建立连接，进而传输数据，当数据传输完毕，并释放了此次 TCP 连接。==若此时第一次发送的连接请求报文段延迟了一段时间后，到达了服务器端，本来这是一个早已失效的报文段，但是服务器端收到该连接请求后误以为客户端又发出了一次新的连接请求，于是服务器端向客户端发出确认应答报文段，并同意建立连接==。如果没有采用三次握手建立连接，由于服务器端发送了确认应答信息，则表示新的连接已成功建立，但是客户端此时并没有向服务器端发出任何连接请求，因此客户端忽略服务器端的确认应答报文，更不会向服务器端传输数据。而服务器端却认为新的连接已经建立了，并在一直等待客户端发送数据，这样服务器端一直处于等待接收数据，直到超出计数器的设定值，则认为服务器端出现异常，并且关闭这个连接。在这个等待的过程中，浪费服务器的资源。**如果采用三次握手，客户端就不会向服务器发出确认应答消息，服务器端由于没有收到客户端的确认应答信息，从而判定客户端并没有请求建立连接，从而不建立该连接**。  

### **为什么不是四次握手？** 

三次握手的目的是确认双方`发送`和`接收`的能力，那四次握手可以嘛？

当然可以，100 次都可以。但为了解决问题，三次就足够了，再多用处就不大了。

### **三次握手过程中可以携带数据么？** 

**第三次握手的时候，可以携带。前两次握手不能携带数据。**

如果前两次握手能够携带数据，那么一旦有人想攻击服务器，那么他只需要在第一次握手中的 SYN 报文中放大量数据，那么服务器势必会消耗更多的**时间**和**内存空间**去处理这些数据，增大了服务器被攻击的风险。第三次握手的时候，客户端已经处于`ESTABLISHED`状态，并且已经能够确认服务器的接收、发送能力正常，这个时候相对安全了，可以携带数据。

### SYN Flood攻击原理及解决方法

==SYN洪水攻击==

客户端发送一个 **SYN包** 给服务端后就退出，而服务端接收到 **SYN包** 后，会回复一个 **SYN+ACK包** 给客户端，然后等待客户端回复一个 **ACK包**。

但此时客户端并不会回复 **ACK包**，所以服务端只能一直等待直到超时。服务端超时后，会重发 **SYN+ACK包** 给客户端，默认会重试 5 次，而且每次等待的时间都会增加（可以参考 TCP 协议超时重传的实现）。

另外，当服务端接收到 **SYN包** 后，会建立一个半连接状态的 Socket。所以，当客户端一直发送 **SYN包**，但不回复 **ACK包**，那么将会耗尽服务端的资源，这就是 **SYN Flood 攻击**。

==解决方法==

1、可缩短 SYN Timeout时间，可以通过缩短从接收到SYN报文到确定这个报文无效并丢弃该连接的时间，可以降低服务器负荷。

2、设置SYN Cookie，给每个请求连接的IP地址分配一个Cookie，如果短时间内收到同一个IP的重复SYN报文，则以后从这个IP地址来的包会被丢弃。

## 四次挥手

**四次挥手过程：**

​     ![image-20210302155750071](G:/刘强/面试题汇总/面试&面经总结.assets/image-20210302155750071.png)

### **为什么要四次挥手？**

试想一下，假如现在你是客户端你想断开跟Server的所有连接该怎么做？第一步，你自己先停止向Server端发送数据，并等待Server的回复。但事情还没有完，虽然你自身不往Server发送数据了，但是因为你们之前已经建立好平等的连接了，所以此时他也有主动权向你发送数据；故Server端还得终止主动向你发送数据，并等待你的确认。其实，说白了就是保证双方的一个合约的完整执行！

**为什么有时候是三次挥手？**
答：因为假设在服务器发送 ack 给客户端的时候，服务器端已经没有要发送的数据，则这时服务器会将 ACK 捎带在第二次挥手的 FIN 报文里面返回回去，也就是第二和第三合在一起发送了。  

### 如果是三次挥手会有什么问题？

等于说服务端将`ACK`和`FIN`的发送合并为一次挥手，这个时候长时间的延迟可能会导致客户端误以为`FIN`没有到达客户端，从而让客户端不断的重发`FIN`。

## 3、get和post

- GET参数通过URL传递，POST放在Request body中。
- GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息。
- GET请求在URL中传送的参数是有长度限制的，而POST没有。
- GET请求会被浏览器主动cache，而POST不会，除非手动设置。
- GET请求只能进行url编码，而POST支持多种编码方式。
- GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留。
- 对参数的数据类型，GET只接受ASCII字符，而POST没有限制。
- ==GET 产生一个 TCP 数据包；POST 产生两个 TCP 数据包。==对于 GET 方式的请求，浏览器会把 http header 和 data 一并发送出去，服务器响应 200（返回数据）；而对于 POST，浏览器先发送 header，服务器响应 100 continue，浏览器再发送data，服务器响应 200 ok（返回数据）。  

**除了get和post方法外，标准Http协议还支持另外几种请求方法，即：**

**HEAD**：HEAD和GET本质是一样的，区别在于HEAD不含有呈现数据，而仅仅是HTTP头信息）

**PUT**： PUT和POST极为相似，都是向服务器发送数据，但它们之间有一个重要区别，PUT通常指定了资源的存放位置，而POST则没有，POST的数据存放位置由服务器自己决定）

**DELETE**：删除某一个资源

**OPTIONS**：它用于获取当前URL所支持的方法。若请求成功，则它会在HTTP头中包含一个名为“Allow”的头，值是所支持的方法，如“GET, POST”。







# 操作系统

#### 腾讯

##### 1、物理内存和虚拟内存

2、逻辑地址和物理地址的关系

##### 2、进程和线程区别

​	进程：进程是程序的一次执行过程，是程序在执行过程中分配和管理资源的基本单位，每个进程都有自己的地址空间；它至少有五种基本状态，初始态、执行态、等待状态、就绪状态、终止状态。

​	线程：是CPU调度和分派的基本单位，它可与同属同一进程的其他线程共享进程的全部资源。

​	区别：

- 根本区别：进程是资源分配的基本单位，线程是CPU调度和执行的基本单位。
- 在资源开销方面：每个进程有自己独立代码和数据空间，进程切换开销较大。线程可以看作轻量级进程，它共享进程的代码和数据空间，每个线程有自己的程序计数器、寄存器、堆栈，线程切换开销较小。在linux内核中，进程和线程创建都是通过clone（）系统调用指定不同参数创建进程或者线程。
- 在通信方面：进程内的线程本身就共享资源，通信很容易，但是存在同步的问题。而进程间通信，需要进程通信机制。



​	协程

1) 是一种比线程更加轻量级的存在。正如一个进程可以拥有多个线程一样，一个线程可以拥有多个协程；协程不是被操作·	系统内核管理，而完全是由程序所控制。

2) 协程的开销远远小于线程；

3) 协程拥有自己寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切换回来的时候，恢复先前保存的寄存器上下文和栈。

4) 每个协程表示一个执行单元，有自己的本地数据，与其他协程共享全局数据和其他资源。

5) 跨平台、跨体系架构、无需线程上下文切换的开销、方便切换控制流，简化编程模型；

6) 协程又称为微线程，协程的完成主要靠yeild关键字，协程执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行；

7) 协程极高的执行效率，和多线程相比，线程数量越多，协程的性能优势就越明显；

8) 不需要多线程的锁机制；



##### 3、内存分区



##### 4、死锁发生的条件，解决办法

必要条件：

- 互斥条件：每个资源要么已经分配给了一个进程，要么就是可用的。
- 占有和等待条件：已经得到资源的进程可以再次申请新的资源。
- 不可抢占条件：已经分配的资源不能从相应的进程中强制剥夺。
- 环路等待条件：系统中若干进程组成环路，该环路中每个进程都在等待相邻进程正占用的资源。

​	死锁避免：

​	动态地检测资源分配状态，以确保循环等待条件不成立，从而确保系统处于安全状态。

​	银行家算法：对每一个请求进行检查，检查如果满足这一请求系统是否处于安全状态，若是，就满足，若不是，就推迟这一请求。

##### 5、进程间通信方式

6、信号量的具体操作（P、V）



7、多线程的同步

8、锁和条件变量的区别

10、如何判断主机字节序

11、结构体中的内存对齐？为什么要内存对齐

12、内核态和用户态

### **13、同样可以实现互斥，互斥锁和信号量有什么区别？**

“信号量用在多线程多任务同步的，一个线程完成了某一个动作就通过信号量告诉别的线程，别的线程再进行某些动作（大家都在semtake的时候，就阻塞在哪里）。而互斥锁是用在多线程多任务互斥的，一个线程占用了某一个资源，那么别的线程就无法访问，直到这个线程unlock，其他的线程才开始可以利用这  个资源。比如对全局变量的访问，有时要加锁，操作完了，在解锁。有的时候锁和信号量会同时使用的”

**作用域**  
信号量:  进程间或线程间(linux仅线程间的无名信号量pthread semaphore)

互斥锁: 线程间 

  **上锁时**   

信号量: 只要信号量的value大于0，其他线程就可以sem_wait成功，成功后信号量的value减一。若value值不大于0，则sem_wait使得线程阻塞，直到sem_post释放后value值加一,但是sem_wait返回之前还是会将此value值减一
互斥锁:  只要被锁住，其他任何线程都不可以访问被保护的资源

### 14、简述Linux内存分配--伙伴系统 原理

- 目的：最大限度的降低内存的碎片化

    1.将内存块分为了11个连续的页框块（1,2,4,8....512,1024），其中每一个页框块中用链表将内存块对应内存大小的块进行  	链接。 

    2.若需要一块256大小的内存块，则从对应的256链表中查找空余的内存块，若有则分配。否则查找512等等。 

    3.若在256中未找到空余内存块，在512中查找到空余的内存块。则将512分成两部分，一部分进行分配，另一部分则插入      	  256链表中。

    4.内存的释放过程与分配过程相反。在分配过程中由大块分解而成的小块中没有被分配的块将一直等着被分配的块被释   	  放，从而和其合并。最终相当于没有划分小块。

# 数据库

**1、数据库三大范式**

**2、索引是什么？索引的数据结构为什么不用红黑树，B+树每层设置了多少个节点？**

​	  索引是存储引擎用于快速找到记录的一种数据结构。

​	  与红黑树比较：

- **更少的查找次数**。红黑树出度为2，而B+树出度较大，树高更小，查找次数更少
- **利用磁盘预读特性**。为减少磁盘IO，磁盘一般不会严格按需读取，而是每次都会预读。预读过程中，磁盘会进行顺序读取，预读速度快。而操作系统一般将内存和磁盘分割成固定大小的页。内存与磁盘以页为单位交换数据。数据库系统将索引的一个节点大小设置为页的大小，使得一次IO就能完全载入一个节点，利用预读特性，相邻的节点也会被预先载入。

**3、B+树叶子节点存放的是数据还是主键？**

​	  聚集索引：叶子节点存放的是完整的数据记录

​	  辅助索引：叶子节点存放的是主键的值，再到主索引中查找

**4、数据库锁的种类和加锁sql语句**

​	  按照锁的级别可分为：行级锁、表级锁、页级锁。 

# 算法设计

1、设计一个微信红包算法？

二倍均值算法

### 2、给一个超过100G大小的log file, log中存着IP地址, 设计算法找到出现次数最多的IP地址？

​                                         如何找到top K的IP？如何直接用Linux系统命令实现？

Hash分桶法：
• 将100G文件分成1000份，将每个IP地址映射到相应文件中：file_id = hash(ip) % 1000
• 在每个文件中分别求出最高频的IP，再合并 Hash分桶法：
• 使用Hash分桶法把数据分发到不同文件
• 各个文件分别统计top K
• 最后Top K汇总
Linux命令，假设top 10：sort log_file | uniq -c | sort -nr k1,1 | head -10

### 3、有十个机器，九个生产的金币是5g，一个生产的金币是4g，给你一个称，问你一次怎么找出来那个生产4g金币的机器？

给机器编号为1 - 10，按照序号取生产的金币，1号1个，2号2个。。。，称总重G，用（1+10）* 5 - G，算出的值即为产4g的机器。 

### 4、超大规模无序数组，找到中位数



内存足够的情况： 可以使⽤用类似quick sort的思想进行，均摊复杂度为O(n)，算法思想如下：
• 随机选取一个元素，将比它小的元素放在它左边，比它大的元素放在右边
• 如果它恰好在中位数的位置，那么它就是中位数，可以直接返回
• 如果小于它的数超过一半，那么中位数一定在左半边，递归到左边处理
• 否则，中位数一定在右半边，根据左半边的元素个数计算出中位数是右半边的第几大，然后递归 到右半边处理
内存不⾜足的情况：
方法⼀：⼆分法
思路：一个重要的线索是，这些数都是整数。整数就有范围了，32位系统中就是[-2^32, 2^32- 1]， 有了范围我们就可以对这个范围进行二分，然后找有多少个数⼩于Mid,多少数大于mid，然后递归， 和基于quicksort思想的第k大⽅方法类似
方法二：分桶法 思路：化大为小，把所有数划分到各个小区间，把每个数映射到对应的区间⾥里，对每个区间中数的 个数进行计数，数一遍各个区间，看看中位数落在哪个区间，若够小，使⽤用基于内存的算法，否则 继续划分

### 5、给两个文件，分别由100亿个整数，我们只有1G内存，如何找到两个文件交集



使用hash函数将第一个文件的所有整数映射到1000个文件中，每个文件有1000万个整数，大约40M内存， 

内存可以放下，把1000个文件记为 a1,a2,a3.....a1000,用同样的hash函数映射第二个文件到1000个文件中，这1000个文件记为b1,b2,b3......b1000，由于使用的是相同的hash函数，所以两个文件中一样的数字会被分配到文件下标一致的文件中，分别对a1和b1求交集，a2和b2求交集，ai和bi求交集，最后将结果汇总，即为两个文件的交集

### 6、给上千个文件，每个文件大小为1K—100M。给n个词，设计算法对每个词找到所有包含它的文件，你只有100K内存

使用trie树即可。

# GDB调试

1、程序崩溃怎么用GDB调试？

# 项目相关

1、线程池怎么创建的？怎么销毁的，用的什么函数

2、socket有哪些，用的什么函数，传入参数是什么，还能使用那些函数

3、为什么用epoll？知道select和poll吗

4、