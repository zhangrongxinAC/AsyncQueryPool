## 异步请求池

### 设计原理

当服务器需要请求第三方服务器如 mysql，redis，dns 等，每次处理请求都需要 tcp 三次握手，发送请求，等待接收数据，tcp 四次挥手，可以看出这样的效率并不高，问题主要出在每次的 tcp 连接以及等待第三方服务器的数据。

### 主要问题
1. 每次处理请求都要进行 tcp 连接
2. 等待第三方服务器返回的数据

首先解决第一个问题，可以在服务器开启后，提前创建好固定的连接即可。

然后解决第二个问题：这种同步的处理方式能否改成异步处理？

首先需要明白同步和异步的差异：

同步：client 发送请求后阻塞等待 server 数据返回
异步：client 发送请求后继续去执行其他任务，server 数据返回后通知 client 执行回调函数处理业务

因此可以将其改成回调，使用 epoll 管理这些回调，在 commit 的时候将这些 fd 和 callback 封装加入到 epoll 中，再使用一个线程不断探测 epoll 中是否有就绪的事件，拿出来执行 callback 即可。

### 设计流程

**Init**
1. 初始化 epoll
2. 创建一个线程死循环管理 epoll 的事件回调

**Commit**
1. 建立连接
2. 准备对应的协议（如 mysql，redis，dns等）
3. 发送数据到第三方服务器
4. 加入到 epoll 中管理

**Callback**
1. dns 解析
2. 执行回调函数

**Close**
1. 关闭对应 fd
2. 释放内存
3. 主线程结束，子线程退出

### htons() 与 ntohs()

网络字节顺序NBO（Network Byte Order）：
    按从高到低的顺序存储，在网络上使用统一的网络字节顺序，可以避免兼容性问题。

主机字节顺序HBO（Host Byte Order）：
    不同的机器HBO不相同，与CPU设计有关，数据的顺序是由cpu决定的,而与操作系统无关。
    如Intel x86下，int类型 0x12345678 表示为 78 56 34 12
    如IBM power PC下，int类型 0x12345678 表示为 12 34 56 78

导致不同体系结构下的机器之间无法通信，所以要转换成一种约定的顺序，也就是网络字节顺序

htons 和 ntohs 其实看函数名就能看明白

htons：h(HBO) to n(NBO) s(short) // 主机字节顺序 转换为 网络字节顺序 的 short 类型

ntohs：n(NBO) to h(HBO) s(short)

同理还有两个函数：

htonl：h(HBO) to n(NBO) l(long) // 主机字节顺序 转换为 网络字节顺序 的 long 类型

ntohl：n(NBO) to h(HBO) l(long)
