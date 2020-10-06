## IO

1.关于linux的api：

![image-20200910231511664](E:\Typora\imgs\image-20200910231511664.png)



c调用printf函数时，输出内容先进入文件结构体的缓冲区的（8192B,减少磁盘IO，提高效率），但是比如QQ聊天的时候是不需要缓冲区的，应该直接发送， 用write调用就没有缓冲区。

printf（"hello"） 调用 10次，会调用10次write将这5个字符先写入缓冲区，然后调用sys_write一次性写入显示设备中。

![image-20200910232213677](E:\Typora\imgs\image-20200910232213677.png)

![image-20200910233127803](E:\Typora\imgs\image-20200910233127803.png)

这样一个问题？A写文件到磁盘， B读文件， 能立马读到么？

不能。因为首先文件内容，先写到A进程自己的缓冲区中，B是看不到的，然后write调用将文件写入内核缓冲区中（注意，什么时候将内核缓冲区的内容刷到磁盘呢？ OS有一个守护进程， 会定时将内核缓冲区的内容flush，“缓输出”的概念）。B发起read调用，此时即使A写的内容没有flush到磁盘，但是由于已经在内核缓冲区了，B还是能读到的。



- 一个进程打开文件的个数   ulimit -a  可以查看默认1024， 可以修改（``ulimit  -n``），但是最大值跟内存有关

  ![image-20200911101928537](E:\Typora\imgs\image-20200911101928537.png)

![image-20200911102745595](E:\Typora\imgs\image-20200911102745595.png)







​	

##### IO多路复用

首先，select， poll, epoll都是同步IO，读写事件就绪后，自己负责读写，这个过程是阻塞的，而异步IO的实现内核自己负责吧数据从内核空间拷贝到用户空间。

三者的区别：

![image-20200910094542498](E:\Typora\imgs\image-20200910094542498.png)

![image-20200910094550248](E:\Typora\imgs\image-20200910094550248.png)

![image-20200910094602495](E:\Typora\imgs\image-20200910094602495.png)

避免了fdset在内核用户空间来回拷贝

```java
表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

select低效是因为每次它都需要轮询。但低效也是相对的，视情况而定，也可通过良好的设计改善。
```

##### select

![image-20200910105346166](E:\Typora\imgs\image-20200910105346166.png)



![image-20201006100016079](E:\Typora\imgs\image-20201006100016079.png)

4,5,6,7这几个描述符都会交给select管理，负责监听4,5,6,7是否有请求事件（数据）到达。如果没有，就在selelct上阻塞。如果listenfd上有数据到达，这时候select会从阻塞返回，返回就绪的文件描述符的个数（这里就会返回1）。然后，需要判断是哪个文件描述符有数据到达了，当判断是7有数据到达了，然后会调用accept返回一个新的已连接描述符8，接着讲8添加到select监控的文件描述符中，然后调用select阻塞等待，等到请求事件的到来。

如果4,5有数据到达了，此时TCP/IP协议栈接受后，数据还在内核空间，select函数返回2。然后遍历fdset判断是哪两个文件描述符激活了，然后找到对应的fd，调用recfrom将数据从内核复制到用户空间。当处理完这个俩个fd的数据后，再进入select，进行监听。

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds, struct timeval *timeout);
nfds: 监控的文件描述符集里最大文件描述符加1，因为此参数会告诉内核检测前多少个文件描述符的状态
readfds：监控有读数据到达文件描述符集合，传入传出参数
writefds：监控写数据到达文件描述符集合，传入传出参数
exceptfds：监控异常发生达文件描述符集合,如带外数据到达异常，传入传出参数
 //比如你又想监控4fd的读，也想监控4的写，就把4分别传入readfds， writefds	
timeout：定时阻塞监控时间，3种情况
1.NULL，永远等下去
2.设置timeval，等待固定时间
3.设置timeval里时间均为0，检查描述字后立即返回，轮询
struct timeval {
long tv_sec; /* seconds */
long tv_usec; /* microseconds */
};
```



描述符还有监听描述符和已连接描述符之分。监听描述符是作为客户端连接请求的一个端点。它通常被创建一次，并**存在于服务器的整个生命周期**。已连接描述符是客户端和服务器之间已经建立起来了的连接的一个端点。服务器每次接受连接请求时都会创建一次，它**只存在于服务器为一个客户端服务的过程中**。

也就是accept会去处理监听描述符（listenfd），然后返回一个已连接描述符，然后再把已连接的文件描述符添加到select监听的描述符中。

调用 select 时，会发生以下事情：

1. 从**用户空间拷贝 fd_set到内核空间**；

2. 注册回调函数__pollwait；

3. **遍历所有 fd**，对全部指定设备做一次 poll（这里的 poll 是一个文件操作，它有两个参数，一个是文件 fd 本身， 一个是当设备尚未就绪时调用的回调函数__pollwait，这个函数把设备自己特有的等待队列传给内核，让内核把当前的进程挂载到其中）；

4. 当设备就绪时，设备就会唤醒在自己特有等待队列中的【所有】节点，于是当前进程就获取到了完成的信号。 poll 文件操作返回的是一组标准的掩码，其中的各个位指示当前的不同的就绪状态（全 0 为没有任何事件触发）， 根据 mask 可对 fd_set 赋值；

5. 如果所有设备返回的掩码都没有显示任何的事件触发，就去掉回调函数的函数指针，进入有限时的睡眠状态， 再**恢复和不断做 poll，再作有限时的睡眠，直到其中一个设备有事件触发为止。**

6. 只要有事件触发，系统调用返回，将 **fd_set 从内核空间拷贝到用户空间**，回到用户态，用户就可以对相关的 fd作进一步的读或者写操作了(recvfrom之类的)。

###### 性能分析

假设我们的服务器需要支持100万的并发连接，**则在_FD_SETSIZE为1024的情况下**，则我们至少需要开辟1k个进程才能实现100万的并发连接。除了进程间上下文切换的时间消耗外，从内核/用户空间大量的无脑内存拷贝、数组轮询等，是系统难以承受的。因此，基于select模型的服务器程序，要达到10万级别的并发访问，是一个很难完成的任务。还有一点就是，如果连接的客户端过多，select采用的**轮询模型**，会大大降低服务器的响应效率

select是单线程处理了并发请求噢

##### poll

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
//相当于将fd包装在了一个pollfd结构体中
struct pollfd {
int fd; /* 文件描述符 */
short events; /* 监控的事件 */
short revents; /* 监控事件中满足条件返回的事件 */
};
POLLIN普通或带外优先数据可读,即POLLRDNORM | POLLRDBAND
POLLRDNORM-数据可读
POLLRDBAND-优先级带数据可读
POLLPRI 高优先级可读数据
POLLOUT普通或带外数据可写
POLLWRNORM-数据可写
POLLWRBAND-优先级带数据可写
POLLERR 发生错误
POLLHUP 发生挂起
POLLNVAL 描述字不是一个打开的文件
nfds 监控数组中有多少文件描述符需要被监控
timeout 毫秒级等待
-1：阻塞等，#define INFTIM -1 Linux中没有定义此宏
0：立即返回，不阻塞进程
>0：等待指定毫秒数，如当前系统时间精度不够毫秒，向上取值
```

相比于select，可以通过改变open files （``ulimit -n 65536``）将监控的fd调的很大（select只能是1024，改变的话很麻烦）

##### epoll

相比于select的主要提升：

- 它能显著提高程序在大量并发连接中只有**少量活跃**的情况下的系统CPU利用率，因为它会**复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合**  
- 获取事件的时候，无须遍历整个被监听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的文件描述符集

###### 相关的API

- epll_create(int size)

  创建一个epoll句柄，参数size用来告诉内核监听的文件描述符个数，跟内存大小有关   

  ```c
  int epoll_create(int size)
  //size：告诉内核监听的数目
  ```

**刚开始在内核空间中创建一颗红黑树，然后一次性把监听的描述符添加到这颗红黑树上。 然后剩下的就是等待，有响应的时候，红黑树就把 间让用户自己去处理，中间的都是一些红黑树的插入删除操作了**。

![image-20200918103419270](E:\Typora\imgs\image-20200918103419270.png)

- epoll_ctl

  注册fd，修改fd相关的事件，删除fd

```c
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)

epfd：为epoll_creat的句柄
op：表示动作，用3个宏来表示：
EPOLL_CTL_ADD(注册新的fd到epfd)，
EPOLL_CTL_MOD(修改已经注册的fd的监听事件)，
EPOLL_CTL_DEL(从epfd删除一个fd)；
fd：后面的event那个参数所对应的fd
event：告诉内核需要监听的事件
struct epoll_event {
__uint32_t events; /* Epoll events */
epoll_data_t data; /* User data variable */
};
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）
EPOLLOUT：表示对应的文件描述符可以写
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）
EPOLLERR：表示对应的文件描述符发生错误
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来
说的
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需
要再次把这个socket加入到EPOLL队列里
```

- epoll_wait

  等待监控的文件描述符上事件的产生，类似于select（）调用

  ```c
  int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
  events：用来从内核得到事件的集合(传出参数)
  maxevents：告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，
  timeout：是超时时间
  -1：阻塞
  0：立即返回，非阻塞
  >0：指定微秒
  返回值：成功返回有多少文件描述符就绪，时间到时返回0，出错返回
  ```

  

epoll 原理概述

调用 epoll_create 时，做了以下事情：

1. 内核帮我们在 epoll 文件系统里建了个 file结点；

2. 在内核 cache里建了个红黑树用于存储以后 epoll_ctl传来的 socket；

3. 建立一个 **list 链表**，用于存储准备就绪的事件。 调用 epoll_ctl 时，做了以下事情：

   **当我们执行epoll_ctl时，除了把socket放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后就来把socket插入到准备就绪链表里了。** 

调用 epoll_wait时，做了以下事情：

观察 list 链表里有没有数据。有数据就返回，没有数据就 sleep，等到 timeout 时间到后即使链表没数 据也返回。而且，通常情况下即使我们要监控百万计的句柄，大多一次也只返回很少量的准备就绪句 柄而已，所以，**epoll_wait 仅需要从内核态 copy 少量的句柄到用户态而已**。

epoll 的提升：

1. 本身没有最大并发连接的限制，仅受系统中进程能打开的最大文件数目限制；

2. 效率提升：只有活跃的 socket 才会主动的去调用 callback 函数；

   

3. 省去不必要的内存拷贝：epoll 通过内核与用户空间 mmap 同一块内存实现。

   ​	每次注册新的事件到epoll句柄中时(在epoll ctI中指定EPOLL CTL ADD) ,会把所有的fd拷贝进内核,而**不是在epoll wait的时候重复拷贝**。epoll保证 了每个fd在整个过程中只会拷贝一次吗？ 调用epoll ctl的时候？存疑

mmap:

mmap将一个文件或者其它对象映射进内存。文件被映射到多个页上，如果文件的大小不是所有页的大小之和，最后一个页不被使用的空间将会清零。munmap执行相反的操作，删除特定地址区域的对象映射。

当使用mmap映射文件到进程后，就可以直接操作这段虚拟地址进行文件的读写等操作，**不必再调用read，write等系统调用**。但需注意，直接对该段内存写时不会写入超过当前文件大小的内容。

采用共享内存通信的一个显而易见的好处是效率高，因为**进程可以直接读写内存，而不需要任何数据的拷贝。对于像管道和消息队列等通信方式，则需要在内核和用户空间进行四次的数据拷贝，而共享内存则只拷贝两次数据：一次从输入文件到共享内存区，另一次从共享内存区到输出文件**。实际上，进程之间在共享内存时，并不总是读写少量数据后就解除映射，有新的通信时，再重新建立共享内存区域。而是保持共享区域，直到通信完毕为止，这样，数据内容一直保存在共享内存中，并没有写回文件。共享内存中的内容往往是在解除映射时才写回文件的。因此，采用共享内存的通信方式效率是非常高的。

通常使用mmap()的三种情况： **提高I/O效率、匿名内存映射、共享内存进程通信** 。

**eg**: 进程A, B同时读一个文件的同一页：

​	进程A和进程B都将该页映射到自己的地址空间, 当进程A第一次访问该页中的数据时, 它生成一个缺页中断. 内核此时读入这一页到内存并更新页表使之指向它.以后, 当进程B访问同一页面而出现缺页中断时, 该页已经在内存, 内核只需要将进程B的页表登记项指向次页即可. 如下图所示:



## 文件系统

linux  ext2， ext3， ext4

![image-20200913090227173](E:\Typora\imgs\image-20200913090227173.png)

![image-20200913094202033](E:\Typora\imgs\image-20200913094202033.png)

一个inode是128字节， 一个inode对应一个文件

由于数据块占了整个块组的绝大部分，也可以近似认为数据块有多少个8KB就分配多少个inode，换句话说，如果平均每个文件的大小是8KB，当分区存满的时候inode表会得到比较充分的利用，数据块也不浪费。如果这个分区存的都是很大的文件（比如电影），则数据块用完的时候inode会有一些浪费，如果这个分区存的都是很小的文件（比如源代码），则有可能数据块还没用完inode就已经用完了，数据块可能有很大的浪费。如果用户在格式化时能够对这个分区以后要存储的文件大小做一个预测，也可以用mke2fs的-i参数手动指定每多少个字节分配一个inode。  

``stat``   获取文件的inode属性 

![image-20200913094738825](E:\Typora\imgs\image-20200913094738825.png)



## 进程

![image-20200913193919515](E:\Typora\imgs\image-20200913193919515.png)

注意PCB存在于内核，只有OS能看到，用户进程时看不到自己的PCB的

每个进程都分配4G的内存地址，0-3G为用户空间，3-4G为内核空间， 不同进程的3-4G地址映射到内存的同样的地址空间（内核）。

![image-20200913200530081](E:\Typora\imgs\image-20200913200530081.png)

0级 是内核态， 3级用户态（1,2级并没有使用）

![image-20200913202100924](E:\Typora\imgs\image-20200913202100924.png)	

进程切换的时候，要保存当前运行进程的处理器现场，保存到进程PCB中的内核栈中。

CPU  1Ghz = 1ns  也就是说一个时钟周期为1ns。绝大多数机器指令都占一个时钟周期，除法比较耗时间，需要4个时钟周期，也就是除法的一条机器指令占4个时钟周期


修改进程资源限制，软限制可改，最大值不能超过硬限制，硬限制只有root用户可以修改
```c
int setrlimit(int resource, const struct rlimit *rlim);
int getrlimit(int resource, struct rlimit *rlim);
查看进程资源限制
cat /proc/self/limits
ulimit -a  
```

#### 进程原语

##### fork

eg：![image-20200913204530757](E:\Typora\imgs\image-20200913204530757.png)

![image-20200913210224054](E:\Typora\imgs\image-20200913210224054.png)

fork底层是调用了 create， clone 创建子进程。在调用了clone之后，子进程才诞生， 然后接着运行下一条指令， 即``return``返回

父子进程的数据已经不共享了。

创建子进程的时候，父子进程的地址空间映射到了相同的内存空间。**写时复制Copy on write的方式来优化fork的性能**，frork刚创建的子进程采用了共享的方式，只用指针指向了父进程的物理资源。当子进程真正要对某些物理资源写操作时，才会真正的复制一块物理资源来供子进程使用。这样就极大的优化了fork的性能，并且从逻辑来说子进程的确是拥有了独立的虚拟内存空间。

![img](https://img-blog.csdn.net/20150116121324948?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSVRlcl9aQw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### exec族

一般都用来配合fork使用

![image-20200913213125961](E:\Typora\imgs\image-20200913213125961.png)

写时复制的好处之一：一般fork都是跟一个exec函数， 如果真的复制代码段数据段的话，又被exec覆盖掉，相当于做了无用功 

```c
 #include <unistd.h>
int execl(const char *path, const char *arg, ...);
//不用加路径， 直接在环境变量PATH中找int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execve(const char *path, char *const argv[], char *const envp[]);
```

##### wait && waitpid

**僵尸进程**  进程状态Z+

如果一个进程已经终止，但是它的父进程尚未调用wait或waitpid对它进行清理（父进程没有回收子进程资源（PCB）），这时的进程状态称为僵尸（Zombie）进程。  

一个进程在终止时会关闭所有文件描述符，**释放在用户空间分配的内存，但它的PCB还保留着**，内核在其中保存了一些信息：如果是正常终止则保存着退出状态，如果是异常终止则保存着导致该进程终止的信号是哪个。这个进程的父进程可以调用wait或waitpid获取这些信息，然后彻底清除掉这个进程。我们知道一个进程的退出状态可以在Shell中用特殊变量$?查看，因为Shell是它的父进程，当它终止时Shell调用``wait``或``waitpid``得到它的退出状
态同时彻底清除掉这个进程。  

- 既然僵尸进程可能会导致内存泄漏（父进程调用``wait``），那为什么OS还有设计僵尸这一状态呢?

  会因为父进程可以知道子进程时如何退出的？正常退出还是异常退出（协商好不同的返回值对应不同的异常类型）。监控这些信息后 ，可以将这些异常信息通过日志进程记录下来。（比如服务端连多个客户端连接，可以监控客户端连接如何断开的）

区别：

- 如果父进程的所有子进程都还在运行，调用wait将使父进程阻塞，而调用`waitpid`时如果在options参数中指定WNOHANG可以使父进程不阻塞而立即返回0(轮询 )。
- wait等待第一个终止的子进程，而``waitpid``可以通过pid参数指定等待哪一个子进程。  

##### pipe管道

每个进程各自有不同的**用户地址空间**，任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核，在内核中开辟一块缓冲区，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为进程间通信（IPC，InterProcess Communication）。  

![image-20200913235330461](E:\Typora\imgs\image-20200913235330461.png)

![image-20200914001142161](E:\Typora\imgs\image-20200914001142161.png)

pipe是单工通信：因为父子进程的文件描述符（比如3,4）指向的是同一个file结构体，比如两者的4都是指向同一个写描述符，父子都往这里写，回头读的时候不知道是父的数据还是子的数据，造成歧义

如果想实现双工，需要开辟两个管道

```c++
#include <stdlib.h>
#include <unistd.h>
#define MAXLINE 80
int main(void)
{
int n;
int fd[2];
pid_t pid;
char line[MAXLINE];
if (pipe(fd) < 0) {
perror("pipe");
exit(1);
}
if ((pid = fork()) < 0) {perror("fork");
exit(1);
}
if (pid > 0) { /* parent */
//父进程关闭读
close(fd[0]);
write(fd[1], "hello world\n", 12);
wait(NULL);
} else { /* child */
//子进程关闭写
close(fd[1]);
n = read(fd[0], line, MAXLINE);
write(STDOUT_FILENO, line, n);
}
return 0;
}
```

![image-20200914082822591](E:\Typora\imgs\image-20200914082822591.png)



##### fifo有名管道

解决无血缘关系的进程通信

创建一个named pipe

``mkfifo  xcs``

![image-20200914085106551](E:\Typora\imgs\image-20200914085106551.png)

相当于在磁盘上创建了一个文件 fifo，不过大小是0， 相当于是索引，指向内核缓冲区，读写都是在内核缓冲区进行的

本地聊天室：

![image-20200914105422401](E:\Typora\imgs\image-20200914105422401.png)



##### mmap/munmap

```c
#include <sys/mman.h>
//返回值是内存中的映射首地址
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);

int munmap(void *addr, size_t length);
```

**MAP_SHARED**    多个进程对同一个文件的映射是共享的，一个进程对映射的内存做了修改，另一个进程也会看到这种变化。
 **MAP_PRIVATE**    多个进程对同一个文件的映射不是共享的，一个进程对映射的内存做了修改，另一个进程并不会看到这种变化，也不会真的写到文件中去。  

```c
#include <stdlib.h>
#include <sys/mman.h>
#include <fcntl.h>
int main(void)
{
int *p;
int fd = open("hello", O_RDWR);
if (fd < 0) {
perror("open hello");
exit(1);
}
p = mmap(NULL, 6, PROT_WRITE, MAP_SHARED, fd, 0);
if (p == MAP_FAILED) {
perror("mmap");
exit(1);
}
close(fd);
p[0] = 0x30313233;
munmap(p, 6);
return 0;
}
```

通过mmap通信：

![image-20200914093825636](E:\Typora\imgs\image-20200914093825636.png)

​	缓输出机制：可以保证，进程对内存的数据读写都是基于内存的：内核缓冲区只要有相应的内容，会直接读内核缓冲区。守护进程，可能每隔一秒？ 将缓冲区数据刷到磁盘。

## 线程

![image-20200921093526294](E:\Typora\imgs\image-20200921093526294.png)

每个线程维护自己的栈，图上是用户空间栈，局部变量申请的时候，开辟栈空间保存局部变量

还有PCB中的内核栈，用来保存处理器现场

- 线程间共享资源

  1.文件描述符表
  2.每种信号的处理方式
  3.当前工作目录
  4.用户ID和组ID
  5.内存地址空间  

- 非共享资源

  1.线程id
  ![image-20200921093917576](E:\Typora\imgs\image-20200921093917576.png)

  **这个线程id不是LWP，CPU在调度是时候通过LWP调度对应的线程，LWP主要给调度器使用的；而线程id只在本进程内部有效，主要是为了进程识别不同线程的。**

  2.处理器现场和栈指针(内核栈)
  3.独立的栈空间(用户空间栈)
  4.errno变量
  5.信号屏蔽字
  6.调度优先级  

- 优点

  提高程序的并发性；开销小，不用重新分配内存；通信和共享数据方便  

- 缺点

  线程不稳定（库函数实现）；线程调试比较困难（gdb支持不好）;线程无法使用unix经典事件，例如信号  

#### 线程原语

##### pthread_create

创建线程

在一个线程中调用pthread_create()创建新的线程后，当前线程从pthread_create()返回继续往下执行，而新的线程所执行的代码由我们传给pthread_create的函数指针start_routine决定。  

##### pthread_exit

调用线程退出函数，注意和exit函数的区别，任何线程里exit导致进程退出，其他线程未工作结束，主控线程退出时不能return或exit。    

```c
void pthread_exit(void *retval);
//void *retval:线程退出时传递出的参数，可以是退出值或地址，如是地址时，不能是线程内部申请的局部地址。
```

##### pthread_join

```c
int pthread_join(pthread_t thread, void **retval);
//pthread_t thread:回收线程的tid
//void **retval:接收退出线程传递出的返回值
//返回值：成功返回0，失败返回错误号
//如果这个线程没有结束，调用者一直阻塞
```

指定一个线程，回收线程的资源（PCB，每个线程都在内核中对应一个PCB）。join函数可以释放线程的PCB。如果一个线程结束，没有被释放掉PCB,就是僵尸线程

##### pthread_cancel

在进程内某个线程可以取消另外一个线程

注意：**同一进程的线程间，pthread_cancel向另一线程发终止信号。系统并不会马上关闭被取消线程，只有在被取消线程下次系统调用时，才会真正结束线程。或调用pthread_testcancel，让内核去检测是否需要取消当前线程**  

##### pthread_detach

不需要主线程调用 pthread_join回收其资源，线程如果如果设置成分离态后，运行结束后OS自动回收资源。

一般情况下，线程终止后，其终止状态一直保留到其它线程调用pthread_join获取它的状态为止。但是线程也可以被置为detach状态，这样的线程一旦终止就立刻回收它占用的所有资源，而不保留终止状态。  



##### 多线程拷贝

![image-20200925104238381](E:\Typora\imgs\image-20200925104238381.png)

将源文件 和需要拷贝到哪的目标文件 通过mmap映射到内存上， 如果有N个线程，则每个线程拷贝1/N，设置好每个线程拷贝的起始地址。如果源文件太大，比如3G，无法一次性映射到内存，则需要分片映射。

如果想要实现已拷贝的进度条（比如显示拷贝进度），通过 ``已拷贝的字节数/总的字节数``

这里``copiedBytes``不能设置成全局变量哦，而是让每个线程维护自己的拷贝的字节数，然后所有线程的字节数相加。

![image-20200925104902812](E:\Typora\imgs\image-20200925104902812.png)

## 信号

比如：

```c
ctl+c   发送了一个SIGINT信号
//当用户按下了<Ctrl+C>组合键时，用户终端(shell)向正在运行中的由该终端启动的程序发出此信号。默认动作为终止里程
ctl+z 	SIGTSTP
ctl+\ 	SIGQUIT
```

都是通过传递了相应的信号（OS发送给当前shell的前台进程）：

硬件驱动程序负责读写实际的硬件设备，比如从键盘读入字符和把字符输出到显示器，线路规程像一个过滤器，对于某些特殊字符并不是让它直接通过，而是做特殊处理，比如在键盘上按下Ctrl-Z，对应的字符并不会被用户程序的read读到，而是被线路规程截获，解释  成SIGINT信号发给前台进程，通常会使该进程停止。线路规程应该过哪些字符和做哪些特殊处理是可以配置的  

![image-20200916154929969](E:\Typora\imgs\image-20200916154929969.png)



- 网络终端



![image-20200914142842837](E:\Typora\imgs\image-20200914142842837.png)

- PCB的信号集

  ![image-20200914144603110](E:\Typora\imgs\image-20200914144603110.png)
  
  如果在进程解除对某信号的阻塞之前这种信号产生过多次，将如何处理？POSIX.1允许系统递送该信号一次或  多次。Linux是这样实现的：常规信号在递达之前产生多次只计一次，而实时信号在递达之前产生多次可以依次放在一个队列里。本章不讨论实时信号。从上图来看，每个信号只有一个bit的未决标志，非0即1，不记录该信号产生了多少次，阻塞标志也是这样表示的。因此，未决和阻塞标志可以用相同的数据类型sigset_t来存储，sigset_t称为信号集，这个类型可以表示每个信号的“有效”或“无效”状态，在阻塞信号集中“有效”和“无效”的含义是该信号是否被阻塞，而在未决信号集中“有效”和“无效”的含义是该信号是否处于未决状态。阻塞信号集也叫做当前进程的信号屏蔽字（Signal Mask），这里的“屏蔽”应该理解为阻塞而不是忽略。  



## 网络编程

关于tcp和udp

tcp和udp都可以通过某些手段实现数据的可靠传输。

但是：tcp的可靠性通过内核的tcp/ip协议栈帮你做这件事，内核负担增加了；

而udp的可靠性是通过应用层，也就是用户空间的进程来完成的，用户进程负担增加了，但是解放了内核。



![image-20200914214902715](E:\Typora\imgs\image-20200914214902715.png)



###### ARP

以太网帧格式：

![image-20200926094255963](E:\Typora\imgs\image-20200926094255963.png)



arp数据报：

找到ip对应的mac地址。源主机发出ARP请求，询问“IP地址是192.168.0.1的主机的硬件地址是多少”，并将这个请求广播到本地网段（以太网帧首部的硬件地址填FF:FF:FF:FF:FF:FF表示广播），目的主机接收到广播的ARP请求，发现其中的IP地址与本机相符，则发送一个ARP应答数据包给源主机，将自己的硬件地址填写在应答包中  。

![image-20200926094402433](E:\Typora\imgs\image-20200926094402433.png)

注意到源MAC地址、目的MAC地址在以太网首部和ARP请求中各出现一次，对于链路层为以太网的情况是多余的，但如果链路层是其它类型的网络则有可能是必要的。  

![image-20200926094734851](E:\Typora\imgs\image-20200926094734851.png)

先发广播包，然后收端判断目标ip是不是自己的IP地址，如果不是则丢弃，否则将自己的mac地址填入arp应答包中，返回一个应答包，是一个单播。



###### 穿透

两个不同局域网的主机如何实现在以太网通信？穿透

![image-20200914224152518](E:\Typora\imgs\image-20200914224152518.png)

tcp状态图

![image-20200911153633776](E:\Typora\imgs\image-20200911153633776.png)



###### 线程池模型

有这样几个模型：

![image-20201006090752279](E:\Typora\imgs\image-20201006090752279.png)