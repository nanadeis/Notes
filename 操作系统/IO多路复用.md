# IO多路复用的使用场景    
1. 客户端要处理多个文件描述符，如交互式输入和网络套接字，如聊天程序  
2. 客户端或服务器端要处理多个套接字  

# select  
int select(int maxfdp1, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval tvptr)  
**maxfdp1** 待测试的描述符个数，值为待测试的最大描述符加1，描述符0,1,2,...,一直到maxfdp1均将被测试  
**readfds**、**writefds**、**exceptfds**是指向描述符集的指针，指定了我们要让内核测试的可读、可写、处于异常条件的描述符集合  
【如果这三个指针都为空，则select提供比sleep更精确的定时器，sleep等待整秒数，select可以精确到微秒】    
**tvptr** 告知内核等待所指定描述符中的任何一个就绪可花多长时间   
1. 空指针，表示永远等待，如果捕捉到一个信号则中止此无限期等待。当所指定的描述符集中的一个已准备好或捕捉到一个信号则返回，如果捕捉到一个信号，则select返回-1，errno设置为EINTR  
2. tvptr->tv_sec == 0 && tvptr->usec == 0 表示根本不等待，测试所有指定描述符并立即返回，这是轮询系统找到多个描述符状态而不阻塞select函数的方法  
3. tvptr->tv_sec!=0 || tvptr->tv_usec 1= 0 等待指定的秒数和微秒数。当指定的描述符之一准备好或指定时间超时立即返回。如果超时到期还没一个描述符准备好，返回0。同样可以被捕捉到的信号中断    
**返回值**  
1. 返回-1表示出错，如一个描述符都没准备好时，捕捉到一个信号  
2. 返回0表示没有描述符准备好就超时了  
3. 返回正整数，是三个描述符集中准备好的描述符数的和。注意如果一个描述符已准备好读和写，那么这个描述符会被计数两次。fd_set中仍旧打开的位对应于已准备好的描述符  

**fd_set**   
fd_set 仅包含一个整数数组，该数组的每个元素的每一bit标记一个文件描述符  
对fd_set数据类型，唯一可以进行的处理是，分配一个这种类型的变量，将这种类型的变量赋值给同类型的另一个变量，以及使用下面四个函数：
1. int FD_ISSET(int fd, fd_set *fdset) 测试描述符集中的一个指定位是否打开  
2. int FD_CLR(int fd, fd_set *fdset) 清除分到对应的一位
3. int FD_SET(int fd, fd_set *fdset) 开启fd对应的一位
4. int FD_ZERO(int fd, fd_set *fdset) 将所有的位置0  

**可读**  
1. 该**套接字接收缓冲区**的数据字节数>=套接字接收缓冲区低水位标记（可以使用SO_RCVLOWAT套接字选项设置该套接字的低水位标记，默认值是1）  
2. 该连接读关闭，也就是接收了FIN的TCP连接，对这样的套接字读不会阻塞并返回0  
3. 该套接字是一个监听套接字且已完成的连接数不为0，对这样的套接字accept不会阻塞  
4. 其上有一个套接字错误待处理，对这样的套接字读不会阻塞并返回0，同时把errno设置成确切的错误条件，这些待处理错误也可以通过指定SO_ERROR套接字选项，调用getsockopt获取并清除  

**可写**  
1. 该套接字发送缓冲区中的可用空间字节数大于等于套接字发送缓冲区低水位标记（可以使用SO_SNDLOWAT套接字选项设置该套接字的低水位标记，默认值是2048）  
2. 该连接的写关闭，对这样的套接字的写操作将产生SIGPIPE信号  
3. 使用非阻塞式connect的套接字已建立连接，或者connect已经失败告终  
【在非阻塞的tcp套接字上调用connect，connect将立即返回一个EINPROGRESS错误，不过已经发起的三次握手继续。随后通过select检测这个连接成功或失败。非阻塞connect有三个用途：
1. 可以把三次握手叠加在其他处理上
2. 可以同时建立多个连接  
3. 可以通过给select指定时间限制，缩短connect的超时时间  
】
4. 其上有套接字错误

**异常条件**  
一个套接字存在带外数据或者仍处于带外标记  
【采用带MSG_OOB标志的recv接收带外数据】  

**对于读、写和异常条件，普通文件的文件描述符总是返回准备好**  

## 缺点  
1. readfds、writefds和expectfds都是值-结果参数，每次调用select前都要重新设置描述符集，因为事件发生后描述符集将被内核修改  


# poll   
int poll(struct pollfd fdarray[], unsigned long nfds, int timeout)  
struct pollfd{
    int fd;   
    short event;  /* 注册的事件 */  
    short revent; /* 实际发生的事件，由内核填充 */     
}  
timeout == -1 永远等待  
timeout == 0 不等待  
timeout > 0 等待timeout毫秒  

# epoll  
**epoll把用户关心的文件描述符上的事件放在内核的一个事件表（红黑树）中，从而无须像select和poll那样每次调用都要重复传入文件描述符集和事件集** epoll需要使用一个额外的文件描述符，来唯一标识内核中的这个事件表。这个文件描述符使用epoll_create函数来创建：
epoll_create(int size)  
size告诉内核它的事件表需要多大，返回文件描述符用作其他所有epoll系统调用的第一个参数，以指定要访问的内核事件表  

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)  
op参数指定操作类型，有如下三种：
1. EPOLL_CTL_ADD 往事件表中注册fd上的事件   
2. EPOLL_CTL_MOD 修改fd上的注册事件  
3. EPOLL_CTL_DEL 删除fd上的注册事件   

int epoll_wait(int epfd, struct epoll_event *event, int maxevents, int timeout)  
epoll_wait如果检测到就绪事件，就将所有就绪的事件从内核事件表（由epfd参数指定）中复制到event指向的数组。**提高了应用程序索引就绪文件描述符的效率**   


## LT模式  
Level Trigger是默认的工作模式
此模式下，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序可以不立即处理该事件。这样，当应用程序下次再次调用epoll_wait还会再次向应用程序通告此事件。直到事件被处理。

## ET模式  
Edge Trigger，当往epoll内核事件表中注册一个文件描述符上的EPOLLET事件时，epoll将以ET模式来操作该文件描述符。ET模式是epoll的高效工作模式  
此模式下，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理该事件，因为后续的epoll_wait调用将不再向应用程序通知这一事件。**这样可以降低用一个epoll事件被重复触发的次数，效率更高**  
### ET模式只能和非阻塞IO配合使用  


## EPOLLONESHOT  
即使使用ET模式，一个socket上的某个事件还是可能被触发多次。在读取完某个socket上的数据后开始处理这些数据，而在数据处理过程中，又有新的数据到达，此时另外一个线程被唤醒来读取这些新的数据。就会出现两个线程同时操作一个socket的局面，这是我们不希望看到的  
对于注册了EPOLLONESHOT事件的文件描述符，操作系统最多触发其上注册的一个可读、可写或异常事件，且只触发一次，除非我们使用epoll_ctl函数重置该文件描述符上注册的EPOLLONESHOT事件。  
注册了EPOLLONESHOT事件的socket一旦被某个线程处理完毕，应该立即重置这个socket上的EPOLLONESOT事件，以确保这个socket下一次可读时，其EPOLLIN事件能被触发。   


# 三组IO复用函数的比较  
## 事件集  
select的fd_set没有将文件描述符和事件绑定，因此需要readfds，writefds，exceptfds三个文件描述符集合，使得select不能处理更多类型的事件，内核对fd_set的修改使得每次调用select之前都要重置这三个fd_set  
poll的pollfd把文件描述符和事件都定义其中，内核每次修改的是pollfd结构体的revents成员，而events成员保存不变，因此下次调用poll时应用程序无需重置pollfd类型的事件集参数   
epoll在内核中维护一个事件表，并提供独立的系统调用epoll_ctl来添加，删除，修改事件。每次epoll_wait调用都之间从该内核事件表中取得用户注册的事件，而无需反复从用户控件读入这些事件   
每次select和poll调用都返回整个用户注册的事件的集合，应用程序索引就绪文件描述符的时间比epoll只返回就绪文件描述符集合长  
## 最大支持文件描述符数  


## 工作模式  
select和poll只能工作在LT模式，而epoll可以工作在ET模式。epoll还支持EPOLLONESHOT模式  

## 具体实现  
select和poll采用**轮询**的方式【循环调用所监测的fd_set内的所有文件描述符对应的驱动程序的poll函数】，每次调用都要扫描整个注册文件描述符集合，检测就绪事件的时间复杂度是O(n)  
epoll_wait采用的是**回调**的方式，内核检测到就绪的文件描述符时，将触发回调函数，回调函数就将该文件描述符上对应的事件插入内核就绪事件队列。内核最后在适当的时机将该就绪事件队列中的内容拷贝到用户空间，时间复杂度O(1)  
当活动连接较多的时候，epoll_wait的效率未必比select和poll高，因为此时回调函数被触发得过于频繁。epoll为实现线程安全要同步处理，加锁，带来额外开销。epoll_wait适用于连接数量多，但活动连接较少的情况  

# epoll原理  
## 等待队列  
内核中的一个socket包含了发送缓冲区，接收缓冲区，等待队列等成员  
对这个socket感兴趣的进程会提供回调函数，并将自己注册到这个socket的等待队列。当事件发生，会调用注册在等待队列上的回调函数。常见的回调函数是，将对这个事件感兴趣的进程的调度状态置为ready，唤醒进程  

## epoll的执行过程  
执行epoll_creat在内核创建红黑树和就绪list  
执行epoll_ctl将文件描述符插入红黑树，向内核中断处理程序注册回调函数，当事件到来时，会调用该回调函数将fd插入rdlist  
epoll_wait观察就绪list有没有数据，有数据就返回，没有数据就阻塞  


## epoll的LT模式和ET模式的实现  
当一个socket上有事件时，内核会把该sock插入到准备就绪list链表，这时调用epool_wait会把准备就绪的socket拷贝到用户态，然后清空准备就绪list。  
如果是LT模式。epoll_wait会检查这些socket，如果socket上有未处理的事件，又把socket放回到刚刚清空的准备就绪list，所以只要还有事件，epoll_wait每次都会返回该socket  
而ET模式，则不会检查  