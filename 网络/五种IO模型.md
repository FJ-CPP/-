[TOC]

# 五种IO模型

## 一、什么是IO

> IO的本质就是等待数据+拷贝数据，而IO的高效体现在是等待的比重较小。

网卡、键盘等硬件通过**“中断”**来通知CPU数据已到达。

其中，对于网络数据：

1. 数据通过网线传输到网卡
2. 网卡将数据写入内存
3. 网卡通过**中断信号**告知CPU数据已到达

 

## 二、同步/异步、阻塞/非阻塞

### 同步与异步

**在探讨IO模型时，同步与异步用来形容方法的<font color=red>调用方式</font>。**

> 同步：在发出调用时，调用方必须等待方法执行完成并得到结果，才会继续向下执行；
>
> 异步：在发出调用时，方法就立即返回了，调用方无需等待方法执行完成，可以继续向下执行其它任务。当被调用方法执行完毕后，会主动通知调用方处理。

同步IO和异步IO的区别就在于：IO操作的完成需要方法调用方自己确定，还是由内核通知。

### 阻塞与非阻塞

**在探讨IO模型时，阻塞与非阻塞用来形容程序在等待调用结果时的<font color=red>状态</font>。**

> 阻塞是指：在调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会被唤醒；
>
> 非阻塞是指：如果不能立刻得到结果之前，则该调用立刻返回并告知线程本次调用无果。因此，非阻塞需要采用轮询的方式多次调用以获取结果。

阻塞IO和非阻塞IO的区别就在于：是否需要轮询检测IO的结果。

> 注：阻塞与同步、非阻塞与异步，在IO模型中是一一对应的关系。
>
> 事实上，异步只有非阻塞，且它的非阻塞比较特殊：不需要轮询，而是被动地等待通知。

 

## 三、同步阻塞IO模型

 <img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091522015.png" alt="image-20220809152235951" style="zoom:80%;" />

> 若内核没有将数据准备好，那么调用方就会一直等待。

套接字和文件描述符默认都是阻塞方式。

 

## 四、同步非阻塞IO模型

 <img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091523603.png" alt="image-20220809152312543" style="zoom:80%;" />

> 若内核数据没有就绪，那么系统调用会立即返回，这种情况需要通过轮询的方式不断地尝试读写。

由于文件描述符默认都是阻塞模式，因此我们需要`fcntl()`函数进行修改：

`int fcntl(int fd, int cmd, ... /* arg */ )`

- `fd`：要操作的文件描述符；
- `cmd`：要进行的操作，这里使用F_GETFL，即获取文件状态的标记flag，以及F_SETFL，即设置文件状态标记flag；
- `...`：可变参数，可以用来设置文件状态；
- RetVal：如果是F_GETFL操作，那么成功会返回文件的状态flag；如果是F_SETFL操作，那么成功会返回0；失败返回-1。

### 示例：将fd修改为非阻塞模式

```c
int flag = fcntl(fd, F_GETFL);  // 获取文件状态
flag |= O_NONBLOCK;        // 文件状态添加非阻塞模式
fcntl(fd, F_SETFL, flag);    // 设置新的文件状态
```

> 注：当文件描述符被设为非阻塞模式时，如果内核没有数据准备就绪，那么全局变量errno会被设置成`EAGAIN`或者`EWOULDBLOCK`；

 

## 五、信号驱动IO模型

> 若内核将数据准备好，会向进程发送信号SIGIO，从而触发对应的信号处理回调函数进行IO操作。

 <img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091541171.png" alt="image-20220809154108058" style="zoom:80%;" />

在信号驱动IO模型中，**等待数据到来的阶段是非阻塞的**，而进程需要**阻塞等待数据从内核拷贝至用户空间**。

**该模型较为复杂，实现起来有些困难。**



## 六、异步IO模型

<img src="https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091521498.jpg" alt="img"  /> 

在异步IO模型中，**等待数据到来的阶段和等待数据从内核拷贝至用户空间的阶段都是非阻塞的**。

**虽然效率极高，但是整体开发难度较大。**



## 七、多路复用IO模型

> 多路复用，又称多路转接，本质上就是在一个线程下同时阻塞或非阻塞地**等待多个文件**是否发生事件(可读、可写等)。

<font color=red>相比于创建多线程等待多个文件，多路复用的方式能够节约线程调度消耗的资源</font>。

![image-20220809152416898](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091525956.png) 

### 实现方式一、select

`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)`

- `nfds`：<font color=red>所有文件描述符中的最大值 + 1</font>。
- `readfds`：**读事件**的输入输出型参数。==作输入参数==时，表示调用方关注对应文件是否可读；==作输出参数==时，用来通知用户有哪些关注的文件现在是可读的；为NULL表示不关注任何文件是否可读。
- `writefds`：**写事件**的输入输出型参数，用法同`readfds`。
- `exceptfds`：**异常事件**的输入输出型参数，用法同`readfds`。
- `timeout`：为==NULL==表示以**阻塞**的方式调用select；为==0==表示**非阻塞**方式调用select；==大于0==表示在timeout这段时间内阻塞，超时就会返回。
- RetVal：返回三个fd_set中描述符的总数量；如果出错则返回-1。

#### I. fd_set结构体

> fd_set是一个位图结构，用来表示一个文件描述符集合。

Linux下用**宏**`FD_SETSIZE`来定义该位图能标识多少个文件描述符。

操作fd_set的相关接口：

1. `void FD_CLR(int fd, fd_set *set);  // 用来清除set中相关fd的位` 
2. `int FD_ISSET(int fd, fd_set *set); // 用来查看set中相关fd的位是否为真` 
3. `void FD_SET(int fd, fd_set *set);  // 用来设置set中相关fd的位` 
4. `void FD_ZERO(fd_set *set);         // 用来清除set的全部位`

#### II. timeval结构体	

```C
struct timeval
{
  __time_t tv_sec;        /* Seconds. */
  __suseconds_t tv_usec;  /* Microseconds. */
};
```

精确到微秒的时间结构体。

#### III. select的使用案例

```c++
const int SIZE = FD_SETSIZE;
// 一个基于select的服务器
class SelectServer
{
public:
    SelectServer(int port = 8081)
        : _port(port), _listenSock(-1)
    {
    }

    void InitSelectServer()
    {
        // 设置监听套接字
        _listenSock = Sock::Socket();
        // 设置无需TIME_WAIT
        int opt = 1;
        setsockopt(_listenSock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)); // 设置地址复用
        Sock::Bind(_listenSock, _port);
        Sock::Listen(_listenSock);  

        // 初始化套接字数组
        _fdArray[0] = _listenSock;
        for (int i = 1; i < SIZE; ++i)
        {
            _fdArray[i] = -1;
        }
    }

    void Loop()
    {
        while (1)
        {
            // 设置select需要关注读事件的文件描述符集
            fd_set readfds;
            FD_ZERO(&readfds);
            int maxFd = 0; // 记录最大的fd
            for (int i = 0; i < SIZE; ++i)
            {
                if (_fdArray[i] != -1)
                {
                    FD_SET(_fdArray[i], &readfds);
                    maxFd = std::max(maxFd, _fdArray[i]);
                }
            }
            struct timeval time = {0, 0}; // 非阻塞等待
            int n = select(maxFd + 1, &readfds, nullptr, nullptr, &time);
            if (n > 0) // 有读事件发生
            {
                for (int i = 0; i < SIZE; ++i)
                {
                    if (FD_ISSET(_fdArray[i], &readfds))
                    {
                        // 监听套接字可读，说明有新的连接请求
                        if (_fdArray[i] == _listenSock)
                        {
                            int sock = Sock::Accept(_listenSock);

                            // 将新套接字加入套接字数组
                            int j = 0;
                            for (; j < SIZE; ++j)
                            {
                                if (_fdArray[j] == -1)
                                {
                                    break;
                                }
                            }
                            if (j < SIZE)
                            {
                                _fdArray[j] = sock;
                                std::cout << "新的连接已建立: " << sock << std::endl;
                            }
                            else
                            {
                                std::cout << "连接已满!" << std::endl;
                                close(sock);
                            }
                        }
                        else
                        {
                            // 其它套接字可读，即用户发送数据，接收即可
                            char buffer[1024];
                            ssize_t s = recv(_fdArray[i], buffer, sizeof(buffer) - 1, 0);
                            if (s > 0)
                            {
                                buffer[s - 1] = 0;
                                std::cout << "已接收" << _fdArray[i] << "的数据# " << buffer << std::endl;
                                std::string msg = "服务器已收到数据!\n";
                                send(_fdArray[i], msg.c_str(), msg.size(), 0);
                            }
                            else if (s == 0)
                            {
                                std::cout << _fdArray[i] << "已断开连接" << std::endl;
                                close(_fdArray[i]);
                                _fdArray[i] = -1; // 从套接字数组中将其删除
                            }
                            else
                            {
                                std::cerr << "recv error!" << std::endl;
                                exit(4);
                            }
                        }
                    }
                }
            }
            else if (n == -1)
            {
                std::cerr << "select error!" << std::endl;
                exit(5);
            }
        }
    }

    ~SelectServer()
    {
        if (_listenSock != -1)
        {
            close(_listenSock);
        }
    }
private:
    uint16_t _port;
    int _listenSock;
    int _fdArray[SIZE];
};
```

#### IV. select的优缺点分析

优点：

1. **对超时值精确到了微秒**。
2. **移植性更好**，有些平台不支持poll。

缺点：

1. 每次调用select都需要手动设置fd集合, 从接口使用角度来说也非常不便。
2. 每次调用select都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大。
3. 每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大 。
4. select支持的文件描述符数量太小，其值取决于`FD_SERSIZE`，其值通常为**1024**。

### 实现方式二、poll

`int poll(struct pollfd *fds, nfds_t nfds, int timeout)`

- `fds`：输入输出型参数，`pollfd`数组指针，表示调用者关注哪些fd，作输出时用来查看哪些事件发生。
- `nfds`：fds数组的大小。
- `timeout`：以毫秒为单位；设为负数表示阻塞等待，0表示非阻塞，正数表示在一段时间内阻塞等待。
- RetVal：返回有多少个fd有事件发生；出错返回-1。

#### I. pollfd结构体

```c
struct pollfd
{
  int fd;            // 文件描述符
  short int events;  // 调用者关注fd的哪些事件，常用的有：POLLIN 读事件、 POLLOUT 写事件
  short int revents; // 实际fd发生了哪些事件
};
```

#### II. poll的使用案例

```C++
const int SIZE = 1024;
class PollServer
{
public:
	PollServer(int port = 8081)
		: _port(port), _listenSock(-1)
	{
	}

	void InitPollServer()
	{
		// 设置监听套接字
		_listenSock = Sock::Socket();
		// 设置无需TIME_WAIT
		int opt = 1;
		setsockopt(_listenSock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
		Sock::Bind(_listenSock, _port);
		Sock::Listen(_listenSock);  
		
		// 初始化pollfd数组
		_pollfds[0].fd = _listenSock;
		_pollfds[0].events = POLLIN; // 套接字设置关注读事件
		for (int i = 1; i < SIZE; ++i)
		{
			_pollfds[i].fd = -1;
			_pollfds[i].events = 0;
			_pollfds[i].revents = 0;
		}
	}

	void Loop()
	{
		while (1)
		{
			int n = poll(_pollfds, SIZE, 0); // 非阻塞等待
			if (n > 0) // 有读事件发生
			{
				for (int i = 0; i < SIZE; ++i)
				{
					if (_pollfds[i].revents & POLLIN)
					{
						// 监听套接字可读，说明有新的连接请求
						if (_pollfds[i].fd == _listenSock)
						{
							int sock = Sock::Accept(_listenSock);

							// 将新套接字加入套接字数组
							int j = 0;
							for (; j < SIZE; ++j)
							{
								if (_pollfds[j].fd == -1)
								{
									break;
								}
							}
							if (j < SIZE)
							{
								_pollfds[j].fd = sock;
								_pollfds[j].events = POLLIN; // 默认关注读事件
								std::cout << "新的连接已建立: " << sock << std::endl;
							}
							else
							{
								// 如果pollfds使用动态开辟的数组，那么这里可以选择将数组扩容！
								std::cout << "连接已满!" << std::endl;
								close(sock);
							}
						}
						else
						{
							// 其它套接字可读，即用户发送数据，接收即可
							char buffer[1024];
							ssize_t s = recv(_pollfds[i].fd, buffer, sizeof(buffer) - 1, 0);
							if (s > 0)
							{
								buffer[s - 1] = 0;
								std::cout << "已接收" << _pollfds[i].fd << "的数据# " << buffer << std::endl;
							}
							else if (s == 0)
							{
								std::cout << _pollfds[i].fd << "已断开连接" << std::endl;
								close(_pollfds[i].fd);
								_pollfds[i].fd = -1; // 从套接字数组中将其删除
							}
							else
							{
								std::cerr << "recv error!" << std::endl;
								exit(4);
							}
						}
					}
				}
			}
			else if (n == 0)
			{
				sleep(1);
				std::cout << "等待超时!" << std::endl;
			}
			else
			{
				std::cerr << "select error!" << std::endl;
				exit(5);
			}
		}
	}

	~PollServer()
	{
		if (_listenSock != -1)
		{
			close(_listenSock);
		}
	}

private:
	uint16_t _port;
	int _listenSock;
	struct pollfd _pollfds[SIZE];
};
```

#### III. poll的优缺点分析

poll的优点：

1. **没有文件描述符数量的限制**。

2. 接口使用相比select更加方便。

poll的缺点：

1. 与select一样，poll返回后，需要轮询`pollfd`来获取就绪的描述符。

2. 将`pollfd`结构体数组从用户拷贝到内核有较大的消耗。

 

### 实现方式三、epoll

epoll的使用基于三个接口：

1、`int epoll_create(int size)`

- `size`：Linux 2.6.8版本之后，size就被忽略了，只要设置成**大于0的值即可**。在此之前，size用来表示调用者期望关注的文件描述符数量；
- RetVal：成功则返回一个文件描述符，用于后续epoll的相关操作；失败则返回-1。


2、`int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)`

- `epfd`：epoll_create返回的文件描述符
- `op`：操作选项，包括`EPOLL_CTL_ADD`(添加关注的fd)、`EPOLL_CTL_MOD`(修改关注的fd的事件)、`EPOLL_CTL_DEL`(删除关注的fd)；

> 注：在Linux_2.6.9之前，如果选项为EPOLL_CTL_DEL，event参数也不得为空，在此版本之后，event可以设为NULL；
>

- `fd`：op操作的文件描述符

- `event`：包括关注的文件描述符、关注的事件

- RetVal：成功返回0，失败返回-1

相关结构体：

```C
struct epoll_event
{
  uint32_t events;   // 调用者关注的事件，包括：EPOLLIN(读)、EPOLLOUT(写)，EPOLLERR(错误)，可按位或
  epoll_data_t data; // 用来存储数据的联合体，通常是我们关注的文件的相关信息，比如epoll_data.fd；
};

typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

3、`int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)`

- `epfd`：`epoll_create()`返回的文件描述符
- `events`：一个输出型数组(需要我们自己分配内存)，用来接收有哪些fd、发生了什么事件；
- `maxevents`：表示本次等待最多处理的事件个数，该值不能超过`events`数组的大小；
- `timeout`：超时时间，以毫秒为单位。其中-1表示阻塞式等待，0表示非阻塞；
- RetVal：成功则返回有多少个文件描述符有事件发生；失败则返回-1。


####  I. epoll的使用案例

```C++
const int NUM = 10;

class EpollServer
{
public:
	EpollServer(uint16_t port = 8081)
		: _port(port)
		, _listenSock(-1)
		, _epfd(-1)
	{
		// 初始化events数组，用于接收待处理的事件
		_events = new epoll_event[NUM];
	}

	void InitEpollServer()
	{
		// 初始化epoll套接字
		_epfd = epoll_create(1);

		// 初始化监听套接字
		_listenSock = Sock::Socket();

		// 取消监听套接字的TimeWait
		int opt = 1;
		setsockopt(_listenSock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
		
		Sock::Bind(_listenSock, _port);
		Sock::Listen(_listenSock);

		// 将监听套接字添加至epoll
		struct epoll_event event;
		event.events = EPOLLIN;
		event.data.fd = _listenSock;
		epoll_ctl(_epfd, EPOLL_CTL_ADD, _listenSock, &event);
	}

	void Loop()
	{
		while (1)
		{
			int n = epoll_wait(_epfd, _events, NUM, 1000); // 最长等待1000ms
			switch (n)
			{
			case 0:
				std::cout << "timeout!" << std::endl;
				break;
			case -1:
				std::cerr << "epoll_wait error!" << std::endl;
				exit(1);
				break;
			default:
				Handler(n);
				break;
			}
		}
	}

	~EpollServer()
	{
		if (_listenSock != -1)
		{
			close(_listenSock);
		}
		if (_epfd != -1)
		{
			close(_epfd);
		}
	}
private:
	void Handler(int n)
	{
		for (int i = 0; i < n; ++i)
		{
			int sock = _events[i].data.fd;
			// 有读事件发生
			if (_events[i].events & EPOLLIN)
			{
				// 监听套接字发生读事件，说明有新的连接请求
				if (sock == _listenSock)
				{
					int newSock = Sock::Accept(_listenSock);
					std::cout << "接收到新的连接请求# " << newSock << std::endl;

					// 将新套接字添加至epoll
					struct epoll_event event;
					event.data.fd = newSock;
					event.events = EPOLLIN;
					epoll_ctl(_epfd, EPOLL_CTL_ADD, newSock, &event);
				}
				else
				{
					// 普通套接字可读
					char buffer[1024];
					ssize_t s = recv(sock, buffer, sizeof(buffer) - 1, 0);
					if (s > 0)
					{
						buffer[s - 1] = 0;
						std::cout << "接收到" << sock << "发送的数据# " << buffer << std::endl;

						// 对收到的数据进行反馈，需要先等待写事件
						struct epoll_event event;
						event.data.fd = sock;
						event.events = EPOLLIN | EPOLLOUT;
						epoll_ctl(_epfd, EPOLL_CTL_MOD, sock, &event);
					}
					else if (s == 0)
					{
						// 对端已关闭连接，将其从epoll中删除，并关闭文件描述符
						struct epoll_event event;
						epoll_ctl(_epfd, EPOLL_CTL_DEL, sock, &event);
						close(sock);
						std::cout << sock << "已断开连接" << std::endl;
					}
					else
					{
						std::cerr << "recv error!" << std::endl;
						exit(2);
					}
				}
			}

			// 有写事件发生
			if (_events[i].events & EPOLLOUT)
			{
				int sock = _events[i].data.fd;
				std::string msg = "服务器# 已收到你的信息！\n";
				send(sock, msg.c_str(), msg.size(), 0);
				
				// 由于这里只是对客户端信息进行一次反馈，之后不再关注写事件
				struct epoll_event event;
				event.data.fd = sock;
				event.events = EPOLLIN;
				epoll_ctl(_epfd, EPOLL_CTL_MOD, sock, &event);
			}
		}
	}

private:
	uint16_t _port; // 服务端口号
	int _listenSock; // 监听套接字
	int _epfd; // epoll文件描述符
	struct epoll_event* _events; // 用于接收待处理的事件
};
```

#### II. epoll的底层原理

1. 当用户调用`epoll_create()`时，内核会创建一个`eventpoll`对象，该对象是文件系统中的一员，因此拥有自己的fd，也就是`epoll_create()`返回的`epfd`；

2. `eventpoll`维护了一棵**<font color=red>红黑树</font>**作为epoll的**“监视列表”**，红黑树的每一个节点包括了fd(socket)、用户关注该fd的哪些事件(event)等数据。当用户调用`epoll_ctl()`时，实际上就是对这棵红黑树进行**增删查改**的操作；
3. 当有新的fd被添加到epoll的监视列表中时，<font color=red>**内核为每一个fd对应的设备驱动(如网卡)注册一个回调函数**</font>；

4. `eventpoll`还维护了一个<font color=red>**双向链表**</font>作为epoll的**“就绪队列”**，其中每个节点包括了fd、该fd发生了哪些用户关注的事件等数据。当epoll监视的fd有关注事件发生时，对应的设备驱动就会通过**回调函数**将这些数据写入节点并添加至双向链表中。
5. 当用户调用`epoll_wait()`时，**内核直接查看双向链表中是否有节点**。如果有，就将相关数据写入到events数组中并拷贝至用户空间，同时在链表中删除对应节点。

#### III. epoll工作方式

- **水平触发(LT, Level Trigger)**

在该模式下：

> 对于读操作：只要缓冲区还有数据没有读完，就返回读就绪；
>
> 对于写操作：只要缓冲区还没满，就返回写就绪。
>

总之，如果没有一次读写完数据，那么就会一直通知有事件发生。

- **边缘触发(ET, Edge Trigger)**

在该模式下：

> 对于读操作：
>
> 1. 缓冲区有新数据到达时，返回读就绪
> 2. 缓冲区有数据可读，且应用进程为相应的描述符添加`EPOLLIN`事件时，返回读就绪。
>
> 对于写操作：
>
> 1. 缓冲区旧数据被取走，有空间可写时，返回写就绪
>
> 2. 缓冲区有空间可写，且应用进程为相应的描述符添加`EPOLLOUT`事件时，返回写就绪。

即：只有在发生变化时(缓冲区数据变化、事件变化)，才会通知有事件发生。

- **LT与ET对比**

1. ET模式相比于LT模式减少了通知次数，因此**效率更高**；

2. ET模式只会通知一次，因此迫使用户必须在一次通知到来时将数据读取完毕，而LT模式下可以一次读取一部分，因为下一次依然会有通知到来；
3. LT模式**支持阻塞和非阻塞读写**方式，而**<font color=red>ET只能是非阻塞读写</font>**。因为：ET模式必须一次性读完数据，而每一次读取的buffer是有限大小，因此需要循环读取！如果是阻塞方式，那么在循环读取的最后一次，一旦没有数据了，这一次读取就会被挂起，也就是整个服务进程被挂起！
4. `select()`和`poll()`只能是LT模式，而epoll可以通过令`epoll_ctl()`的`op=EPOLLET`设置某个fd的事件通知是ET模式。

### epoll与select/poll的对比

1. `select/poll`都是由系统以遍历的方式去检测特定的fd是否有事件发生；而`epoll`是通过注册回调函数的方式，将fd和发生的事件添加至就绪队列，每一次`epoll`只需要检测队列是否有节点即可，效率更高。
2. `select/poll`每次都要用户将关注的事件结构体拷贝至内核，检测完成后再由内核拷贝结果给用户，且用户需要再完成一次遍历检测哪些fd发生了事件；而epoll在内核维护了一个红黑树管理用户关注的fd：一方面，不需要用户维护这些fd，另一方面，红黑树的增删查改效率相比数组更高；此外，`epoll`只需要将就绪队列中的节点返回给用户即可，无需内核遍历所有关注的fd，同时也节省了用户遍历检测的时间，效率更高。
3. 相比于`select`，epoll不再有fd的数量上限问题。

 

## 八、Reactor(反应堆)设计模式

> 反应堆设计模式(Reactor pattern)是一种高效处理并发服务请求的设计模式。
>
> 它是基于epoll多路复用的事件驱动模式，只需要添加用户关心的事件，当事件发生后，Reactor事件派发方法就会自动调用该事件注册的回调函数进行处理。该处理可以由事件派发方法完成，也可以由线程池的线程去完成。

![img](https://typora-1307604235.cos.ap-nanjing.myqcloud.com/typora_img/202208091521503.jpg) 

因此，Reactor模式的优势在于：

1. 使用多路复用，IO效率高。

2. 仅负责监听并派发事件，而事件的处理回调函数由用户提供，因此做到了将“事件处理”和“事件监听、派发”**解耦**，具有很高的**框架复用性**。

3. 可以增加Reactor实例，进一步提高CPU利用率。

### 多Reactor实例的实现思路

1、多线程/进程创建多个Reactor实例，通过管道建立它们之间的通信信道。

2、主进程/线程的Reactor只负责监听套接字的`accept`工作。当有新连接时，将fd通过管道**<font color=red>负载均衡</font>**地派发给其余的Reactor。

3、其余进程/线程获取到fd后，将其添加至Reactor中，并等待事件派发。

### Reactor是同步IO还是异步IO?

**同步**。因为Reactor中关注的读写等事件的处理任务需要Reactor进行派发。

与Reactor对应的**Proactor**则关注的是**读写完成的事件**，即读写任务的派发不需要Proactor去完成，而是由内核自动调用处理回调函数，完成时通知Proactor，因此**Proactor是异步IO**。



 

 

 

 

 

 