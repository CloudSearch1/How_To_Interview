# **网络编程：**

转载--from：https://juejin.cn/post/6844904200141438984

## IO多路复用

### 1）什么是IO多路复用

定义：IO多路复用是一种同步IO模型，实现一个线程可以监视多个文件句柄；一旦某个文件句柄就绪，就能够通知应用程序进行相应的读写操作；没有文件句柄就绪时会阻塞应用程序，交出cpu。多路是指网络连接，复用指的是同一个线程

### 2）为什么有IO多路复用机制?

没有IO多路复用机制时，有BIO、NIO两种实现方式，但有一些问题

#### 同步阻塞（BIO）

- 服务端采用单线程，当accept一个请求后，在recv或send调用阻塞时，将无法accept其他请求（必须等上一个请求处recv或send完），`无法处理并发`

```scss
// 伪代码描述
while(1) {
  // accept阻塞
  client_fd = accept(listen_fd)
  fds.append(client_fd)
  for (fd in fds) {
    // recv阻塞（会影响上面的accept）
    if (recv(fd)) {
      // logic
    }
  }  
}
复制代码
```

- 服务器端采用多线程，当accept一个请求后，开启线程进行recv，可以完成并发处理，但随着请求数增加需要增加系统线程，`大量的线程占用很大的内存空间，并且线程切换会带来很大的开销，10000个线程真正发生读写事件的线程数不会超过20%，每次accept都开一个线程也是一种资源浪费`

```scss
// 伪代码描述
while(1) {
  // accept阻塞
  client_fd = accept(listen_fd)
  // 开启线程read数据（fd增多导致线程数增多）
  new Thread func() {
    // recv阻塞（多线程不影响上面的accept）
    if (recv(fd)) {
      // logic
    }
  }  
}

复制代码
```

#### 同步非阻塞（NIO）

- 服务器端当accept一个请求后，加入fds集合，每次轮询一遍fds集合recv(非阻塞)数据，没有数据则立即返回错误，`每次轮询所有fd（包括没有发生读写事件的fd）会很浪费cpu`

```scss
setNonblocking(listen_fd)
// 伪代码描述
while(1) {
  // accept非阻塞（cpu一直忙轮询）
  client_fd = accept(listen_fd)
  if (client_fd != null) {
    // 有人连接
    fds.append(client_fd)
  } else {
    // 无人连接
  }  
  for (fd in fds) {
    // recv非阻塞
    setNonblocking(client_fd)
    // recv 为非阻塞命令
    if (len = recv(fd) && len > 0) {
      // 有读写数据
      // logic
    } else {
       无读写数据
    }
  }  
}
复制代码
```

#### IO多路复用（现在的做法）

- 服务器端采用单线程通过select/epoll等系统调用获取fd列表，遍历有事件的fd进行accept/recv/send，使其能`支持更多的并发连接请求`

```scss
fds = [listen_fd]
// 伪代码描述
while(1) {
  // 通过内核获取有读写事件发生的fd，只要有一个则返回，无则阻塞
  // 整个过程只在调用select、poll、epoll这些调用的时候才会阻塞，accept/recv是不会阻塞
  for (fd in select(fds)) {
    if (fd == listen_fd) {
        client_fd = accept(listen_fd)
        fds.append(client_fd)
    } elseif (len = recv(fd) && len != -1) { 
      // logic
    }
  }  
}
```

#### 3、IO多路复用的三种实现方式

- select
- poll
- epoll



### select函数接口

```arduino
#include <sys/select.h>
#include <sys/time.h>

#define FD_SETSIZE 1024
#define NFDBITS (8 * sizeof(unsigned long))
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS)

// 数据结构 (bitmap)
typedef struct {
    unsigned long fds_bits[__FDSET_LONGS];
} fd_set;

// API
int select(
    int max_fd, 
    fd_set *readset, 
    fd_set *writeset, 
    fd_set *exceptset, 
    struct timeval *timeout
)                              // 返回值就绪描述符的数目

FD_ZERO(int fd, fd_set* fds)   // 清空集合
FD_SET(int fd, fd_set* fds)    // 将给定的描述符加入集合
FD_ISSET(int fd, fd_set* fds)  // 判断指定描述符是否在集合中 
FD_CLR(int fd, fd_set* fds)    // 将给定的描述符从文件中删除  
```

#### 5、select使用示例

```scss
int main() {
  /*
   * 这里进行一些初始化的设置，
   * 包括socket建立，地址的设置等,
   */

  fd_set read_fs, write_fs;
  struct timeval timeout;
  int max = 0;  // 用于记录最大的fd，在轮询中时刻更新即可

  // 初始化比特位
  FD_ZERO(&read_fs);
  FD_ZERO(&write_fs);

  int nfds = 0; // 记录就绪的事件，可以减少遍历的次数
  while (1) {
    // 阻塞获取
    // 每次需要把fd从用户态拷贝到内核态
    nfds = select(max + 1, &read_fd, &write_fd, NULL, &timeout);
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 0; i <= max && nfds; ++i) {
      if (i == listenfd) {
         --nfds;
         // 这里处理accept事件
         FD_SET(i, &read_fd);//将客户端socket加入到集合中
      }
      if (FD_ISSET(i, &read_fd)) {
        --nfds;
        // 这里处理read事件
      }
      if (FD_ISSET(i, &write_fd)) {
         --nfds;
        // 这里处理write事件
      }
    }
  }
复制代码
```

#### 6、select缺点

- 单个进程所打开的FD是有限制的，通过FD_SETSIZE设置，默认1024
- 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
- 对socket扫描时是线性扫描，采用轮询的方法，效率较低（高并发时）

#### 7、poll函数接口

> poll与select相比，只是没有fd的限制，其它基本一样

```arduino
#include <poll.h>
// 数据结构
struct pollfd {
    int fd;                         // 需要监视的文件描述符
    short events;                   // 需要内核监视的事件
    short revents;                  // 实际发生的事件
};

// API
int poll(struct pollfd fds[], nfds_t nfds, int timeout);
复制代码
```

#### 8、poll使用示例

```c++
// 先宏定义长度
#define MAX_POLLFD_LEN 4096  

int main() {
  /*
   * 在这里进行一些初始化的操作，
   * 比如初始化数据和socket等。
   */

  int nfds = 0;
  pollfd fds[MAX_POLLFD_LEN];
  memset(fds, 0, sizeof(fds));
  fds[0].fd = listenfd;
  fds[0].events = POLLRDNORM;
  int max  = 0;  // 队列的实际长度，是一个随时更新的，也可以自定义其他的
  int timeout = 0;

  int current_size = max;
  while (1) {
    // 阻塞获取
    // 每次需要把fd从用户态拷贝到内核态
    nfds = poll(fds, max+1, timeout);
    if (fds[0].revents & POLLRDNORM) {
        // 这里处理accept事件
        connfd = accept(listenfd);
        //将新的描述符添加到读描述符集合中
    }
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 1; i < max; ++i) {     
      if (fds[i].revents & POLLRDNORM) { 
         sockfd = fds[i].fd
         if ((n = read(sockfd, buf, MAXLINE)) <= 0) {
            // 这里处理read事件
            if (n == 0) {
                close(sockfd);
                fds[i].fd = -1;
            }
         } else {
             // 这里处理write事件     
         }
         if (--nfds <= 0) {
            break;       
         }   
      }
    }
  }
```

#### 9、poll缺点

- 每次调用poll，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
- 对socket扫描时是线性扫描，采用轮询的方法，效率较低（高并发时）

### 10、epoll函数接口

```arduino
#include <sys/epoll.h>

// 数据结构
// 每一个epoll对象都有一个独立的eventpoll结构体
// 用于存放通过epoll_ctl方法向epoll对象中添加进来的事件
// epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可
struct eventpoll {
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
};

// API

int epoll_create(int size); // 内核中间加一个 ep 对象，把所有需要监听的 socket 都放到 ep 对象中
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // epoll_ctl 负责把 socket 增加、删除到内核红黑树
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);// epoll_wait 负责检测可读队列，没有可读 socket 则阻塞进程
复制代码
```

#### 11、epoll使用示例

```scss
int main(int argc, char* argv[])
{
   /*
   * 在这里进行一些初始化的操作，
   * 比如初始化数据和socket等。
   */

    // 内核中创建ep对象
    epfd=epoll_create(256);
    // 需要监听的socket放到ep中
    epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);
 
    while(1) {
      // 阻塞获取
      nfds = epoll_wait(epfd,events,20,0);
      for(i=0;i<nfds;++i) {
          if(events[i].data.fd==listenfd) {
              // 这里处理accept事件
              connfd = accept(listenfd);
              // 接收新连接写到内核对象中
              epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
          } else if (events[i].events&EPOLLIN) {
              // 这里处理read事件
              read(sockfd, BUF, MAXLINE);
              //读完后准备写
              epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
          } else if(events[i].events&EPOLLOUT) {
              // 这里处理write事件
              write(sockfd, BUF, n);
              //写完后准备读
              epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
          }
      }
    }
    return 0;
}
```

#### 12、epoll缺点

- epoll只能工作在linux下

#### 13、epoll LT 与 ET模式的区别

- epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。
- LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作
- ET模式下，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读完，或者遇到EAGAIN错误

#### 14、epoll应用

- redis
- nginx

### 15、select/poll/epoll之间的区别

|            | select             | poll             | epoll                                             |
| ---------- | ------------------ | ---------------- | ------------------------------------------------- |
| 数据结构   | bitmap             | 数组             | 红黑树                                            |
| 最大连接数 | 1024               | 无上限           | 无上限                                            |
| fd拷贝     | 每次调用select拷贝 | 每次调用poll拷贝 | fd首次调用epoll_ctl拷贝，每次调用epoll_wait不拷贝 |
| 工作效率   | 轮询：O(n)         | 轮询：O(n)       | 回调：O(1)                                        |

### 16、高频面试题

- 什么是IO多路复用?
- nginx/redis 所使用的IO模型是什么？
- select、poll、epoll之间的区别
- epoll 水平触发（LT）与 边缘触发（ET）的区别？

### 17、完整代码示例

https://github.com/CloudSearch1/How_To_Interview



## epoll边缘触发与水平触发

转载 --- 链接：https://juejin.cn/post/6844903878685622285

在网络编程中，会涉及到水平触发与边缘触发的概念，工程中以边缘触发较为常见，本文讲述了边缘触发与水平触发的概念，并给出代码示例，通过代码可以很清楚的看到它们之间的区别。

### 水平触发与边缘触发

水平触发(level-trggered)

- 只要文件描述符关联的读内核缓冲区非空，有数据可以读取，就一直发出可读信号进行通知，
- 当文件描述符关联的内核写缓冲区不满，有空间可以写入，就一直发出可写信号进行通知

边缘触发(edge-triggered)

- 当文件描述符关联的读内核缓冲区由空转化为非空的时候，则发出可读信号进行通知，
- 当文件描述符关联的内核写缓冲区由满转化为不满的时候，则发出可写信号进行通知

### 两者的区别？

水平触发是只要读缓冲区有数据，就会一直触发可读信号，而边缘触发仅仅在空变为非空的时候通知一次，举个例子：

1. 读缓冲区刚开始是空的
2. 读缓冲区写入2KB数据
3. 水平触发和边缘触发模式此时都会发出可读信号
4. 收到信号通知后，读取了1kb的数据，读缓冲区还剩余1KB数据
5. 水平触发会再次进行通知，而边缘触发不会再进行通知

所以边缘触发需要一次性的把缓冲区的数据读完为止，也就是一直读，直到读到`EGAIN`(`EGAIN`说明缓冲区已经空了)为止，因为这一点，边缘触发需要设置文件句柄为非阻塞。

> ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

这里只简单的给出了水平触发与边缘触发的处理方式的不同，边缘触发相对水平触发处理的细节更多一些，

```c++
//水平触发
ret = read(fd, buf, sizeof(buf));

//边缘触发（代码不完整，仅为简单区别与水平触发方式的代码）
while(true) {
    ret = read(fd, buf, sizeof(buf);
    if (ret == EAGAIN) break;
}
```

### 代码示例

通过下面的代码示例，能够看到水平触发与边缘触发代码的不同以及触发次数的不同。通过这个示例能够加深你对边缘触发与水平触发的理解。

##### echo  server代码：

```scss
/* echo server*/

#include<sys/epoll.h>
#include<stdio.h>
#include<stdlib.h>
#include<arpa/inet.h>
#include<netinet/in.h>
#include<sys/socket.h>
#include<unistd.h>
#include<string.h>
#include<fcntl.h>
#include<errno.h>

#define MAX_EVENTS 1024
#define LISTEN_PORT 33333
#define MAX_BUF 1024

// #define LEVEL_TRIGGER

int setnonblocking(int sockfd);
int events_handle_level(int epfd, struct epoll_event ev);
int events_handle_edge(int epfd, struct epoll_event ev);
void run();

int main(int _argc, char* _argv[]) {
    run();

    return 0;
}

void run() {
    int epfd = epoll_create1(0);
    if (-1 == epfd) {
        perror("epoll_create1 failure.");
        exit(EXIT_FAILURE);
    }

    char str[INET_ADDRSTRLEN];
    struct sockaddr_in seraddr, cliaddr;
    socklen_t cliaddr_len = sizeof(cliaddr);
    int listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&seraddr, sizeof(seraddr));
    seraddr.sin_family = AF_INET;
    seraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    seraddr.sin_port = htons(LISTEN_PORT);

    int opt = 1;
    setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    if (-1 == bind(listen_sock, (struct sockaddr*)&seraddr, sizeof(seraddr))) {
        perror("bind server addr failure.");
        exit(EXIT_FAILURE);
    }
    listen(listen_sock, 5);

    struct epoll_event ev, events[MAX_EVENTS];
    ev.events = EPOLLIN;
    ev.data.fd = listen_sock;
    if (-1 == epoll_ctl(epfd, EPOLL_CTL_ADD, listen_sock, &ev)) {
        perror("epoll_ctl add listen_sock failure.");
        exit(EXIT_FAILURE);
    }

    int nfds = 0;
    while (1) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (-1 == nfds) {
            perror("epoll_wait failure.");
            exit(EXIT_FAILURE);
        }

        for ( int n = 0; n < nfds; ++n) {
            if (events[n].data.fd == listen_sock) {
                int conn_sock = accept(listen_sock, (struct sockaddr *)&cliaddr, &cliaddr_len);
                if (-1 == conn_sock) {
                    perror("accept failure.");
                    exit(EXIT_FAILURE);
                }
                printf("accept from %s:%d\n", inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)), ntohs(cliaddr.sin_port));

                setnonblocking(conn_sock);
#ifdef LEVEL_TRIGGER
                ev.events = EPOLLIN;
#else
                ev.events = EPOLLIN | EPOLLET;
#endif
                ev.data.fd = conn_sock;
                if (-1 == epoll_ctl(epfd, EPOLL_CTL_ADD, conn_sock, &ev)) {
                    perror("epoll_ctl add conn_sock failure.");
                    exit(EXIT_FAILURE);
                }
            } else {
#ifdef LEVEL_TRIGGER
                events_handle_level(epfd, events[n]);
#else
                events_handle_edge(epfd, events[n]);
#endif
            }
        }
    }

    close(listen_sock);
    close(epfd);
}

int setnonblocking(int sockfd){
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFD, 0)|O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}

int events_handle_level(int epfd, struct epoll_event ev) {
    printf("events_handle, ev.events = %d\n", ev.events);
    int fd = ev.data.fd;
    if (ev.events == EPOLLIN) {
        char buf[MAX_BUF];
        bzero(buf, MAX_BUF);
        int n = 0;
        n = read(fd, buf, 5);
        printf("step in level_trigger, read bytes:%d\n", n);

        if (n < 0) {
            perror("read fd failure.");
            if (-1 == epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &ev)) {
                perror("epoll_ctl del fd failure.");
                exit(EXIT_FAILURE);
            }

            return -1;
        }

        if (0 == n) {
            if (-1 == epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &ev)) {
                perror("epoll_ctl del fd failure.");
                exit(EXIT_FAILURE);
            }
            close(fd);

            return 0;
        }

        printf("recv from client: %s\n", buf);
        write(fd, buf, n);

        return 0;
    }

    return 0;
}


int events_handle_edge(int epfd, struct epoll_event ev) {
    printf("events_handle, ev.events = %d\n", ev.events);
    int fd = ev.data.fd;
    if (ev.events == EPOLLIN) {
        char* buf = (char*)malloc(MAX_BUF);
        bzero(buf, MAX_BUF);
        int count = 0;
        int n = 0;
        while (1) {
            n = read(fd, (buf + n), 5);
            printf("step in edge_trigger, read bytes:%d\n", n);
            if (n > 0) {
                count += n;
            } else if (0 == n) {
                break;
            } else if (n < 0 && EAGAIN == errno) {
                printf("errno == EAGAIN, break.\n");
                break;
            } else {
                perror("read failure.");
                break;
            }

        }

        if (0 == count) {
            if (-1 == epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &ev)) {
                perror("epoll_ctl del fd failure.");
                exit(EXIT_FAILURE);
            }
            close(fd);

            return 0;
        }

        printf("recv from client: %s\n", buf);
        write(fd, buf, count);

        free(buf);
        return 0;
    }

    return 0;
}

复制代码
```

##### 客户端代码

```ini
#include<unistd.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<netdb.h>

#include<stdlib.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>

#define SERVER_PORT 33333
#define MAXLEN 1024

void client_handle(int sock);

int main(int argc, char* argv[]) {
    for (int i = 1; i < argc; ++i) {
        printf("input args %d: %s\n", i, argv[i]);
    }
    struct sockaddr_in seraddr;
    int server_port = SERVER_PORT;
    if (2 == argc) {
        server_port = atoi(argv[1]);
    }

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&seraddr, sizeof(seraddr));
    seraddr.sin_family = AF_INET;
    inet_pton(AF_INET, "127.0.0.1", &seraddr.sin_addr);
    seraddr.sin_port = htons(server_port);

    connect(sock, (struct sockaddr *)&seraddr, sizeof(seraddr));
    client_handle(sock);

    return 0;
}

void client_handle(int sock) {
    char sendbuf[MAXLEN], recvbuf[MAXLEN];
    bzero(sendbuf, MAXLEN);
    bzero(recvbuf, MAXLEN);
    int n = 0;

    while (1) {
        if (NULL == fgets(sendbuf, MAXLEN, stdin)) {
            break;
        }
        // 按`#`号退出
        if ('#' == sendbuf[0]) {
            break;
        }

        write(sock, sendbuf, strlen(sendbuf));
        n = read(sock, recvbuf, MAXLEN);
        if (0 == n) {
            break;
        }
        write(STDOUT_FILENO, recvbuf, n);
    }

    close(sock);
}
复制代码
```

客户端连接服务端，分别向服务端发送`123`和`123456`，服务端运行结果如下：

##### 边缘触发运行结果：

可以看到边缘触发只触发一次

```arduino
sl@Li:~/Works/study/epoll$ ./server 
accept from 127.0.0.1:38170
events_handle, ev.events = 1
step in edge_trigger, read bytes:4
step in edge_trigger, read bytes:-1
errno == EAGAIN, break.
recv from client: 123

events_handle, ev.events = 1
step in edge_trigger, read bytes:5
step in edge_trigger, read bytes:2
step in edge_trigger, read bytes:-1
errno == EAGAIN, break.
recv from client: 123456

复制代码
```

重新编译服务端程序，再次发送`123`和`123456`，服务端运行结果如下：

##### 水平触发运行结果：

可以看到，在接收`123456`时，触发了两次，而边缘触发只触发一次。

```perl
sl@Li:~/Works/study/epoll$ ./server 
accept from 127.0.0.1:38364
events_handle, ev.events = 1
step in level_trigger, read bytes:4
recv from client: 123

events_handle, ev.events = 1
step in level_trigger, read bytes:5
recv from client: 12345
events_handle, ev.events = 1
step in level_trigger, read bytes:2
recv from client: 6
```



# 传输层协议简介

常见的传输层协议主要有 TCP 协议和 UDP 协议。TCP 协议是面向有连接的协议，也就是说在使用 TCP 协议传输数据之前一定要在发送方和接收方之间建立连接。一般情况下建立连接需要三步，关闭连接需要四步。

建立 TCP 连接后，由于有数据重传、流量控制等功能，TCP 协议能够正确处理丢包问题，保证接收方能够收到数据，与此同时还能够有效利用网络带宽。然而 TCP 协议中定义了很多复杂的规范，因此效率不如 UDP 协议，不适合实时的视频和音频传输。

UDP 协议是面向无连接的协议，它只会把数据传递给接收端，但是不会关注接收端是否真的收到了数据。但是这种特性反而适合多播，实时的视频和音频传输。因为个别数据包的丢失并不会影响视频和音频的整体效果。

IP 协议中的两大关键要素是源 IP 地址和目标 IP 地址。而刚刚我们说过，传输层的主要作用是**实现应用程序之间的通信**。因此传输层的协议中新增了三个要素：源端口号，目标端口号和协议号。通过这五个信息，可以唯一识别一个通信。

不同的端口用于区分同一台主机上不同的应用程序。假设你打开了两个浏览器，浏览器 A 发出的请求不会被浏览器 B 接收，这就是因为 A 和 B 具有不同的端口。

协议号用于区分使用的是 TCP 还是 UDP。因此相同两台主机上，相同的两个进程之间的通信，在分别使用 TCP 协议和 UDP 协议时也可以被正确的区分开来。

用一句话来概括就是：“源 IP 地址，目标 IP 地址，源端口号，目标端口号和协议号”这五个信息只要有一个不同，都被认为是不同的通信。

## UDP 首部

UDP 协议最大的特点就是简单，它的首部如下图所示：



![UDP 首部](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4917a0c314f~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



包长度表示 UDP 首部的长度和 UDP 数据长度之和。

校验和用来判断数据在传输过程中是否损坏。计算这个校验和的时候，不仅考虑源端口号和目标端口号，还要考虑 IP 首部中的源 IP 地址，目标 IP 地址和协议号（这些又称为 UDP 伪首部）。这是因为以上五个要素用于识别通信时缺一不可，如果校验和只考虑端口号，那么另外三个要素收到破坏时，应用就无法得知。这有可能导致不该收到包的应用收到了包，该收到包的应用反而没有收到。

这个概念同样适用于即将介绍的 TCP 首部。

## TCP 首部

和 UDP 首部相比，TCP 首部要复杂得多。解析这个首部的时间也相应的会增加，这是导致 TCP 连接的效率低于 UDP 的原因之一。



![TCP 首部](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4917702553b~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



其中某些关键字段解释如下：

- 序列号：它表示发送数据的位置，假设当前的序列号为 s，发送数据长度为 l，则下次发送数据时的序列号为 s + l。在建立连接时通常由计算机生成一个随机数作为序列号的初始值。
- 确认应答号：它等于下一次应该接收到的数据的序列号。假设发送端的序列号为 s，发送数据的长度为 l，那么接收端返回的确认应答号也是 s + l。发送端接收到这个确认应答后，可以认为这个位置以前所有的数据都已被正常接收。
- 数据偏移：TCP 首部的长度，单位为 4 字节。如果没有可选字段，那么这里的值就是 5。表示 TCP 首部的长度为 20 字节。
- 控制位：改字段长度为 8 比特，分别有 8 个控制标志。依次是 CWR，ECE，URG，ACK，PSH，RST，SYN 和 FIN。在后续的文章中你会陆续接触到其中的某些控制位。
- 窗口大小：用于表示从应答号开始能够接受多少个 8 位字节。如果窗口大小为 0，可以发送窗口探测。
- 紧急指针：尽在 URG 控制位为 1 时有效。表示紧急数据的末尾在 TCP 数据部分中的位置。通常在暂时中断通信时使用（比如输入 Ctrl + C）。

## TCP 握手

TCP 是面向有连接的协议，连接在每次通信前被建立，通信结束后被关闭。了解连接建立和关闭的过程通常是考察的重点。连接的建立和关闭过程可以用一张图来表示：



![TCP 连接建立和关闭](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4917725c794~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



通常情况下，我们认为客户端首先发起连接。

### 三次握手建立连接

这个过程可以用以下三句形象的对话表示：

1. （客户端）：我要建立连接了。
2. （服务端）：我知道你要建立连接了，我这边没有问题。
3. （客户端）：我知道你知道我要建立连接了，接下来我们就正式开始通信。

### 为什么是三次握手

根据一般的思路，我们可能会觉得只要两次握手就可以了，第三步确认看似是多余的。那么 TCP 协议为什么还要费力不讨好的加上这一次握手呢？

这是因为在网络请求中，我们应该时刻记住：“网络是不可靠的，数据包是可能丢失的”。假设没有第三次确认，客户端向服务端发送了 SYN，请求建立连接。由于延迟，服务端没有及时收到这个包。于是客户端重新发送一个 SYN 包。回忆一下介绍 TCP 首部时提到的序列号，这两个包的序列号显然是相同的。

假设服务端接收到了第二个 SYN 包，建立了通信，一段时间后通信结束，连接被关闭。这时候最初被发送的 SYN 包刚刚抵达服务端，服务端又会发送一次 ACK 确认。由于两次握手就建立了连接，此时的服务端就会建立一个新的连接，然而客户端觉得自己并没有请求建立连接，所以就不会向服务端发送数据。从而导致服务端建立了一个空的连接，白白浪费资源。

在三次握手的情况下，服务端直到收到客户端的应答后才会建立连接。因此在上述情况下，客户端会接受到一个相同的 ACK 包，这时候它会抛弃这个数据包，不会和服务端进行第三次握手，因此避免了服务端建立空的连接。

### ACK 确认包丢失怎么办

三次握手其实解决了第二步的数据包丢失问题。那么第三步的 ACK 确认丢失后，TCP 协议是如何处理的呢？

按照 TCP 协议处理丢包的一般方法，服务端会重新向客户端发送数据包，直至收到 ACK 确认为止。但实际上这种做法有可能遭到 SYN 泛洪攻击。所谓的泛洪攻击，是指发送方伪造多个 IP 地址，模拟三次握手的过程。当服务器返回 ACK 后，攻击方故意不确认，从而使得服务器不断重发 ACK。由于服务器长时间处于半连接状态，最后消耗过多的 CPU 和内存资源导致死机。

正确处理方法是服务端发送 RST 报文，进入 CLOSE 状态。这个 RST 数据包的 TCP 首部中，控制位中的 RST 位被设置为 1。这表示连接信息全部被初始化，原有的 TCP 通信不能继续进行。客户端如果还想重新建立 TCP 连接，就必须重新开始第一次握手。

### 四次握手关闭连接

这个过程可以用以下四句形象的对话表示：

1. （客户端）：我要关闭连接了。
2. （服务端）：你那边的连接可以关闭了。
3. （服务端）：我这边也要关闭连接了。
4. （客户端）：你那边的连接可以关闭了。

由于连接是双向的，所以双方都要主动关闭自己这一侧的连接。

### 关闭连接的最后一个 ACK 丢失怎么办

实际上，在第三步中，客户端收到 FIN 包时，它会设置一个计时器，等待相当长的一段时间。如果客户端返回的 ACK 丢失，那么服务端还会重发 FIN 并重置计时器。假设在计时器失效前服务器重发的 FIN 包没有到达客户端，客户端就会进入 CLOSE 状态，从而导致服务端永远无法收到 ACK 确认，也就无法关闭连接。

示意图如下：



![TCP 关闭连接](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4917affaab0~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



## 数据包重发

### 数据发送

丢包重发的前提是发送方能够知道接收方是否成功的接收了消息。所以，在 TCP 协议中，接收端会给发送端返回一个通知，也叫作确认应答（ACK），这表示接收方已经收到了数据包。

根据上一节对 TCP 首部的分析得知，ACK 的值和下次发送数据包的序列号相等。因此 ACK 也可以理解为：“发送方，下次你从这个位置开始发送！”。下图表示了数据发送与确认应答的过程：



![ACK 确认](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4a42d1ce607~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



数据包和 ACK 应答都有可能丢失，在这种情况下，发送方如果在一段时间内没有收到 ACK，就会重发数据：



![未收到 ACK 时重发数据](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4a429f16556~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



即使网络连接正常，由于延迟的存在，接收方也有可能收到重复的数据包，因此接收方通过 TCP 首部中的 SYN 判断这个数据包是否曾经接收过。如果已经接收过，就会丢弃这个包。

### 重传超时时间(RTO)

如果发送方等待一段时间后，还是没有收到 ACK 确认，就会启动超时重传。这个等待的时间被称为重传超时时间(RTO，Retransmission TimeOut)。RTO 的值具体是多久呢？

首先，RTO 的值不是固定的，它是一个动态变化的时间。这个时间总是略大于连接往返时间（RTT，Round Trip Time）。这个设定可以这样理解：“数据发送给对方，再返回到我这里，假设需要 10 秒，那我就等待 12秒，如果超过 12 秒，那估计就是回不来了。”

RTT 是动态变化的，因为谁也不知道网络下一时刻是否拥堵。而当前的 RTO 需要根据未来的 RTT 估算得出。RTO 不能估算太大，否则会多等待太多时间；也不能太小，否则会因为网络突然变慢而将不该重传的数据进行重传。

RTO 有自己的估算公式，可以保证即使 RTT 波动较大，它的变化也不会太剧烈。感兴趣的读者可以自行查阅相关资料。

### TCP 窗口

按照之前的理论，在数据包发出后，直至 ACK 确认返回以前，发送端都无法发送数据，而且包的往返时间越长，网络利用效率和通信性能就越低。前两张图片形象的解释了这一点。

为了解决这个问题，TCP 使用了“窗口”这个概念。窗口具有大小，它表示无需等待确认应答就可以继续发送数据包的最大数量。比如窗口大小为 4 时，数据发送的示意图如下：



![窗口大小为 4](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4a42d43f0b4~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



不等确认就连续发送若干个数据包会不会有问题呢？我们首先来看数据包丢失问题。

我们知道 TCP 首部中的 ACK 字段表示接收方已经收到数据的最后位置。因此，接收方成功接收到了 1-1000 字节的数据后，它会发送一个 ACK = 1001 的确认包。假设 1001-2000 字节的数据包丢失了，由于窗口长度比较大，发送方会继续发送 2001-3000 字节的数据包。接收端并不会返回这个数据包的确认，因为它最后收到的数据还是 1-1000 字节的数据包。

因此，接收端返回的数据包的 ACK 依然是 1001。这表示：“喂，发数据的，别往后发了，你第 1001 字节开始的数据还没来呢”。可以想见，发送端以后每次发送数据包得到的确认中，ACK 的值都是 1001。当连续收到三次确认之后，发送方会意识到：“对方还没有接收到数据，这个包需要重传”。

因此，引入窗口的概念后，被发送的数据不能立刻丢弃，需要缓存起来以备将来需要重发。

利用窗口发送数据的过程可以用下图表示：



![快速重传](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4a429d0211f~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



如果是数据包没有丢失，但是确认包丢失了呢？这就是窗口最擅长处理的问题了。假设发送发收到的确认包中的 ACK 第一次是 1001，第二次是 4001。那么我们完全可以相信中间的两个包是成功被接收的。因为如果有没接收到的包，接收方是不会增加 ACK 的。

在这种情况下，如果不使用窗口，发送方就需要重传第二、三个数据包，但是有了窗口的概念后，发送方就省略了两次重传。因此使用窗口实际上可以理解为“空间换时间”。



![某些确认包丢失时不用重发](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4a42edeb3ce~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)



## 流量控制

### 窗口大小

如果窗口过大，会导致接收方的缓存区数据溢出。这时候本该被接收的数据反而丢弃了，就会导致无意义的重传。因此，窗口大小是一个可以改变的值，它由接收端主机控制，附加在 TCP 首部的“窗口大小”字段中。

### 慢启动

在连接建立的初期，如果窗口比较大，发送方可能会突然发送大量数据，导致网络瘫痪。因此，在通信一开始时，TCP 会通过慢启动算法得出窗口的大小，对发送数据量进行控制。

流量控制是由发送方和接收方共同控制的。刚刚我们介绍了接收方会把自己能够承受的最大窗口长度写在 TCP 首部中，实际上在发送方这里，也存在流量控制，它叫拥塞窗口。TCP 协议中的窗口是指发送方窗口和接收方窗口的较小值。

**慢启动过程如下：**

1. 通信开始时，发送方的拥塞窗口大小为 1。每收到一个 ACK 确认后，拥塞窗口翻倍。
2. 由于指数级增长非常快，很快地，就会出现确认包超时。
3. 此时设置一个“慢启动阈值”，它的值是当前拥塞窗口大小的一半。
4. 同时将拥塞窗口大小设置为 1，重新进入慢启动过程。
5. 由于现在“慢启动阈值”已经存在，当拥塞窗口大小达到阈值时，不再翻倍，而是线性增加。
6. 随着窗口大小不断增加，可能收到三次重复确认应答，进入“快速重发”阶段。
7. 这时候，TCP 将“慢启动阈值”设置为当前拥塞窗口大小的一半，再将拥塞窗口大小设置成阈值大小（也有说加 3）。
8. 拥塞窗口又会线性增加，直至下一次出现三次重复确认应答或超时。

以上过程可以用下图概括：



![窗口大小变化示意图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/12/12/1604b4a42ce0527d~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)


转载from

+链接：https://juejin.cn/post/6844903521268203528
