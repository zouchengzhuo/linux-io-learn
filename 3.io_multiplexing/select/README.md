# io阻塞导致单进程只能处理单fd上的io
io过程中，阻塞socket的 accept、recv、connect、send等都可能阻塞当前进程，导致进程只能处理一个fd上的io任务。  
# 通过非阻塞socket+循环判断实现单进程多fd上io
用非阻塞socket加上循环判断的方式，可以实现单进程处理多个fd上io的目的，不过因为大量CPU性能被耗费在循环判断上了，有一点得不偿失。 
# 通过select实现单进程多fd上的io
select提供一种机制，将这个循环判断的过程交由内核的中断程序来完成，socket没必要再设置为非阻塞，可以把所有需要处理的fd一起交给select，然后由select来产生阻塞。当有fd可读/可写时，内核中断程序会唤醒select，让其退出阻塞状态回到CPU任务队列中，并通过fd_set告诉应用层哪些fd可读了，哪些fd可写了。应用层遍历fd，根据需求执行 accept/recv/connect/send 操作。  
## select模式下socket需要设置为非阻塞吗
理论上是不需要，个人理解也可也设置一下，（需确认）可能有一些意外情况导致某一个socket上的io操作阻塞当前进程，从而影响整个应用的性能
## select能同时处理多少个fd
数量是 sizeof(fd_set) 由操作系统/硬件环境等决定，一般不会太多，man文档描述的是不大于1024，也是FD_SETSIZE值。  
可能因为连接数量多了之后性能实在太差吧。  
## 与epoll比的劣势
其一，每次调用select都需要将进程加入到所有监视socket的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定select的最大监视数量，默认只能监视1024个socket。  
其二，进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。  