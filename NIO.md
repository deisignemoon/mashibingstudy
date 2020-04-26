## OS
### 保护模式
1. OS启动会加载kernal，在CPU中加载GDT（全局描述符表），指定内核空间和用户空间内存。
2. 用户程序无法直接访问硬件，只有kernal才能操作硬件。用户专内核会中断（int 0x80软中断）,中断指令0~255。内核有中断表
3. 用户程序中断指令->cpu保存用户内存数据->kernal指令执行->内核调用硬件
4. IO设备也会发出中断指令
5. strace -ff -o <输出文件名> <启动指令>  
6. netstat -natp
7. cd /proc/<进程号>    task线程     fd文件描述符
### IO
1. jvm -> kernel
2. socket(返回fd1)->bind->listen->accept（如果获得连接，返回连接的fd2，否则阻塞）->recv->阻塞等待发来信息
3. BIO CONN-THREAD
4. NIO
5. socket 3->bind(3,8090)->lisen(3)->fcntl(3,nonblock)->accept循环，返回-1|5->fcntl(5,nonblock)->recv(5)
6. select->poll->epoll![select](select.jpg)![](poll.png)![](epoll.png)