---
title: Tomcat 三种 IO 模型
date: 2023-08-22 17:13:45
tags:
---

很多人对 Java 的 IO 很熟悉，张口就可以说出 IO 的几种模型，各自的优缺点；但是同时又对 Java 的 IO 很陌生，毕竟作为一个 CRUD Boy，老项目 Spring + Tomcat 两把梭，新项目 Spring Boot 一把梭，除了日志里偶尔出现的 Tomcat 日志和代码里的输入输出流，根本感觉不到 IO 的存在。本文就聊一聊 Tomcat 中的网络编程部分。

<!-- more -->

Tomcat 支持的 io 模型有 NIO、NIO2、APR
Endpoint

NioEndpoint 在 Tomcat 中扮演的角色

负责接收请求的连接，并封装成 SocketProcessor，最后交给线程池去执行；NioEndpoint 组件是 I/O 多路复用模型的一种实现。其主体结构 Acceptor+Poller+线程池是典型的主从 Reactor 多线程模型，其中 Acceptor 为主 Reactor，Poller 为从 Reactor

Tomcat 连接器模块下的三大核心功能之一，主要负责网络通信；连接器的另外两个核心功能分别是 应用层协议解析-Processor、Tomcat Request/Response 与 ServletRequest/ServletResponse 的转化-Adapter

acceptor, poller, worker

## endpoint

Tomcat 中负责抽象类是 AbstractEndpoint，具体子类在 AprEndpoint, NioEndpoint 和 Nio2Endpoint，
Tomcat 支持的连接器有 NIO、NIO.2 和 APR。

```java
public abstract class AbstractEndpoint<S,U> {
    /**
     * 绑定并监听地址
     */
    public abstract void bind() throws Exception;

    /**
     * endpoint 的入口，用于启动 acceptor, poller, worker
     */
    public abstract void startInternal() throws Exception;

    /**
     * 创建 worker 线程池
     */
    public void createExecutor() {
        internalExecutor = true;
        if (getUseVirtualThreads()) {
            // 创建 Java 21 虚拟线程池
            executor = new VirtualThreadExecutor(getName() + "-virt-");
        } else {
            // 创建多线程线程池，默认核心线程数 10，最大线程数 200，优先级 5
            TaskQueue taskqueue = new TaskQueue();
            TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
            executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
            taskqueue.setParent( (ThreadPoolExecutor) executor);
        }
    }

    /**
     * 创建并启动 acceptor 线程，APR, NIO 使用该默认实现，NIO2 重写了该方法
     */
    protected void startAcceptorThread() {
        acceptor = new Acceptor<>(this);
        String threadName = getName() + "-Acceptor";
        acceptor.setThreadName(threadName);
        Thread t = new Thread(acceptor, threadName);
        t.setPriority(getAcceptorThreadPriority());
        t.setDaemon(getDaemon());
        t.start();
    }

    /**
     * 接受连接，被 acceptor 调用
     */
    protected abstract U serverSocketAccept() throws Exception;

    /**
     * 设置连接，被 acceptor 调用
     */
    protected abstract boolean setSocketOptions(U socket);

    /**
     * 处理 socket 事件，被 poller 调用
     */
    public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            // 获取 socket 处理器
            SocketProcessorBase<S> sc = createSocketProcessor(socketWrapper, event);
            // 处理 socket 事件，根据 dispatch 判断是否使用 worker 处理
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (Throwable t) {
            return false;
        }
        return true;
    }

    /**
     * 创建 socket 处理器
     */
    protected abstract SocketProcessorBase<S> createSocketProcessor(
            SocketWrapperBase<S> socketWrapper, SocketEvent event);
}
```

### AprEndpoint

APR 通过 JNI 进行 socket 操作，相比使用 Java API 效率更高 以前的 Tomcat 有 JioEndpoint
使用该类会创建以下线程

- acceptor 线程
- poller 线程
- sendfile 线程
- worker 线程池

```java
public class AprEndpoint extends AbstractEndpoint<Long,Long> implements SNICallBack {
    /**
     * 绑定并监听地址
     */
    @Override
    public void bind() throws Exception {
        long serverSock = Socket.create(family, Socket.SOCK_STREAM, 0, rootPool);
        int ret = Socket.bind(serverSock, sockAddress);
        ret = Socket.listen(serverSock, getAcceptCount());
    }

    /**
     * 启动 APR endpoint，启动 acceptor, poller, sendfile
     */
    @Override
    public void startInternal() throws Exception {
        if (!running) {
            running = true;
            paused = false;

            // 创建 worker 线程池
            if (getExecutor() == null) {
                createExecutor();
            }

            // 启动 poller 线程
            poller = new Poller();
            poller.init();
            poller.start();

            // 启动 sendfile 线程
            if (getUseSendfile()) {
                sendfile = new Sendfile();
                sendfile.init();
                sendfile.start();
            }

            // 启动 acceptor 线程
            startAcceptorThread();
        }
    }

    /**
     * AprEndpoint 接受连接
     */
    @Override
    protected Long serverSocketAccept() throws Exception {
        long socket = Socket.accept(serverSock);
        return Long.valueOf(socket);
    }

    /**
     * AprEndpoint 设置连接
     */
    @Override
    protected boolean setSocketOptions(Long socket) {
        try {
            AprSocketWrapper wrapper = new AprSocketWrapper(socket, this);
            connections.put(socket, wrapper);
            wrapper.setKeepAliveLeft(getMaxKeepAliveRequests());
            wrapper.setReadTimeout(getConnectionTimeout());
            wrapper.setWriteTimeout(getConnectionTimeout());
            getExecutor().execute(new SocketWithOptionsProcessor(wrapper));
            return true;
        } catch (Throwable t) {
        }
        return false;
    }

    /**
     * This class is the equivalent of the Worker, but will simply use in an external Executor thread pool. This will also set the socket options and do the handshake. This is called after an accept().
     */
    protected class SocketWithOptionsProcessor implements Runnable {
        @Override
        public void run() {
            Lock lock = socket.getLock();
            lock.lock();
            try {
                setSocketOptions(socket);
                getPoller().add(socket.getSocket().longValue(), getConnectionTimeout(), Poll.APR_POLLIN);
            } finally {
                lock.unlock();
            }
        }
    }
}
```

### NioEndpoint

![Alt text](../images/NIO.png)

```java
/**
 * NIO tailored thread pool, providing the following services:
 * <ul>
 * <li>Socket acceptor thread</li>
 * <li>Socket poller thread</li>
 * <li>Worker threads pool</li>
 * </ul>
 *
 * TODO: Consider using the virtual machine's thread pool.
 *
 * @author Mladen Turk
 * @author Remy Maucherat
 */
public class NioEndpoint extends AbstractJsseEndpoint<NioChannel,SocketChannel> {
    /**
     * 绑定并监听地址
     */
    @Override
    public void bind() throws Exception {
        serverSock = ServerSocketChannel.open();
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
        serverSock.bind(addr, getAcceptCount());
        serverSock.configureBlocking(true);
    }

    /**
     * 启动 NIO endpoint，启动 acceptor, poller
     */
    @Override
    public void startInternal() throws Exception {
        if (!running) {
            running = true;
            paused = false;

            // 创建 worker 线程池
            if (getExecutor() == null) {
                createExecutor();
            }

            // 启动 poller 线程
            poller = new Poller();
            Thread pollerThread = new Thread(poller, getName() + "-Poller");
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();

            // 启动 acceptor 线程
            startAcceptorThread();
        }
    }

    /**
     * NioEndpoint 接受连接
     */
    @Override
    protected SocketChannel serverSocketAccept() throws Exception {
        SocketChannel result = serverSock.accept();
        return result;
    }

    /**
     * NioEndpoint 设置连接
     */
    @Override
    protected boolean setSocketOptions(SocketChannel socket) {
        NioSocketWrapper socketWrapper = null;
        try {
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            NioChannel channel = new NioChannel(bufhandler);
            NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            socket.configureBlocking(false);
            if (getUnixDomainSocketPath() == null) {
                socketProperties.setProperties(socket.socket());
            }

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            poller.register(socketWrapper);
            return true;
        } catch (Throwable t) {
        }
        return false;
    }
}
```

### Nio2Endpoint

```java
/**
 * NIO2 endpoint.
 */
public class Nio2Endpoint extends AbstractJsseEndpoint<Nio2Channel,AsynchronousSocketChannel> {
    /**
     * 绑定并监听地址
     */
    @Override
    public void bind() throws Exception {
        threadGroup = AsynchronousChannelGroup.withThreadPool((ExecutorService) getExecutor());
        serverSock = AsynchronousServerSocketChannel.open(threadGroup);
        socketProperties.setProperties(serverSock);
        InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
        serverSock.bind(addr, getAcceptCount());
    }

    /**
     * 启动 NIO2 endpoint, 启动 acceptor
     */
    @Override
    public void startInternal() throws Exception {
        if (!running) {
            allClosed = false;
            running = true;
            paused = false;

            // 创建 worker 线程池
            if (getExecutor() == null) {
                createExecutor();
            }

            // 启动 acceptor 线程
            startAcceptorThread();
        }
    }

    /**
     * 启动 NIO2 acceptor
     */
    @Override
    protected void startAcceptorThread() {
        // Instead of starting a real acceptor thread, this will instead call
        // an asynchronous accept operation
        if (acceptor == null) {
            acceptor = new Nio2Acceptor(this);
            acceptor.setThreadName(getName() + "-Acceptor");
        }
        acceptor.state = AcceptorState.RUNNING;
        getExecutor().execute(acceptor);
    }

    /**
     * Nio2Endpoint 接受连接
     */
    @Override
    protected AsynchronousSocketChannel serverSocketAccept() throws Exception {
        AsynchronousSocketChannel result = serverSock.accept().get();
        return result;
    }

    /**
     * Nio2Endpoint 设置连接
     */
    @Override
    protected boolean setSocketOptions(AsynchronousSocketChannel socket) {
        Nio2SocketWrapper socketWrapper = null;
        try {
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            Nio2Channel channel = new Nio2Channel(bufhandler);
            Nio2SocketWrapper newWrapper = new Nio2SocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            socketProperties.setProperties(socket);
            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(Nio2Endpoint.this.getMaxKeepAliveRequests());
            // Continue processing on the same thread as the acceptor is async
            return processSocket(socketWrapper, SocketEvent.OPEN_READ, false);
        } catch (Throwable t) {
        }
        return false;
    }
}
```

```java
public abstract class SocketProcessorBase<S> implements Runnable {
    protected SocketWrapperBase<S> socketWrapper;

    /**
     *
     */
    @Override
    public final void run() {
        Lock lock = socketWrapper.getLock();
        lock.lock();
        try {
            // It is possible that processing may be triggered for read and
            // write at the same time. The sync above makes sure that processing
            // does not occur in parallel. The test below ensures that if the
            // first event to be processed results in the socket being closed,
            // the subsequent events are not processed.
            if (socketWrapper.isClosed()) {
                return;
            }
            doRun();
        } finally {
            lock.unlock();
        }
    }

    protected abstract void doRun();
}
```

## acceptor

acceptor 调用 endpoint.serverSocketAccept 产生 socket，调用 endpoint.setSocketOptions 设置 socket

acceptor 的功能如其名称，用于接受 socket 连接请求。产生一个与客户端的连接的 socket 后，再将其交给 endpoint 处理。
acceptor 的逻辑特别简单，NIO、APR 使用类 Acceptor，NIO2 使用继承类 Nio2Acceptor，重写 run 方法，实现异步操作。
Acceptor 主要代码如下

```java
public class Acceptor<U> implements Runnable {
    /**
     * 在线程中循环接受新连接直到收到关闭命令
     */
    public void run() {
        int errorDelay = 0;
        while (!stopCalled) {
            try {
                // 接受连接
                U socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // 连接失败进行延时操作
                errorDelay = handleExceptionWithDelay(errorDelay);
            }
            // endpoint 设置连接，处理失败则关闭连接
            if (!endpoint.setSocketOptions(socket)) {
                endpoint.closeSocket(socket);
            }
        }
    }

    /**
     * 延时时长指数增加，防止短时间内产生大量错误
     */
    private static final int INITIAL_ERROR_DELAY = 50;
    private static final int MAX_ERROR_DELAY = 1600;
    protected int handleExceptionWithDelay(int currentErrorDelay) {
        // 第一次不延时
        if (currentErrorDelay > 0) {
            Thread.sleep(currentErrorDelay);
        }
        if (currentErrorDelay == 0) {
            return INITIAL_ERROR_DELAY;
        } else if (currentErrorDelay < MAX_ERROR_DELAY) {
            return currentErrorDelay * 2;
        } else {
            return MAX_ERROR_DELAY;
        }
    }
}
```

Nio2Acceptor 没有调用 endpoint.serverSocketAccept 产生 socket，而是调用 serverSock.accept 产生 socket，另一个显著的特点是 Nio2Acceptor 的 run 方法中没有循环，这是在用异步。

```java
protected class Nio2Acceptor extends Acceptor<AsynchronousSocketChannel>
    implements CompletionHandler<AsynchronousSocketChannel, Void> {
    @Override
    public void run() {
        // 如果达到最大连接数，等待
        try {
            countUpOrAwaitConnection();
        } catch (InterruptedException e) {
        }
        // 连接完成后由 completed 方法处理，失败由 failed 方法处理
        serverSock.accept(null, this);
    }

    @Override
    public void completed(AsynchronousSocketChannel socket,
            Void attachment) {
        if (getMaxConnections() == -1) {
            serverSock.accept(null, this);
        } else if (getConnectionCount() < getMaxConnections()) {
            serverSock.accept(null, this);
        } else {
            // 达到最大连接数，使用新线程接受请求
            getExecutor().execute(this);
        }
        // 交给 endpoint 设置 socket
        if (!setSocketOptions(socket)) {
            closeSocket(socket);
        }
    }

    @Override
    public void failed(Throwable t, Void attachment) {
        if (getMaxConnections() == -1) {
            serverSock.accept(null, this);
        } else {
            // 使用新线程接受请求
            getExecutor().execute(this);
        }
        // 失败后等待一段事件
        errorDelay = handleExceptionWithDelay(errorDelay);
    }
}
```

`run` 方法在一个线程中被执行，根据不同的条件改变 Acceptor 的状态，Acceptor 有四种状态

- `NEW` 新建
- `RUNNING` 运行，已达最大连接数或正在接受连接
- `PAUSED` 暂停，Endpoint 处于暂停状态
- `ENDED` acceptor 停止运行

## poller

### APR poller

```java
public class Poller implements Runnable {
    protected void start() {
        pollerThread = new Thread(poller, getName() + "-Poller");
        pollerThread.setPriority(threadPriority);
        pollerThread.setDaemon(true);
        pollerThread.start();
    }

    @Override
    public void run() {
        SocketList localAddList = new SocketList(getMaxConnections());
        while (pollerRunning) {
            SocketInfo info = localAddList.get();
            while (info != null) {
                int rv = Poll.poll(aprPoller, pollTime, desc, true);
            }
        }
    }
}
```

### NIO poller

poller 负责两件事，处理 poller 事件，处理 IO 事件

```java
public class Poller implements Runnable {
    private AtomicLong wakeupCounter = new AtomicLong(0);
    // IO 多路复用选择器
    private Selector selector;
    // poller 事件队列
    private final SynchronizedQueue<PollerEvent> events =
                new SynchronizedQueue<>();

    /**
     * 将 socket 的读事件注册到事件队列中，被 acceptor 调用 setSocketOptions 时间接调用
     */
    public void register(final NioSocketWrapper socketWrapper) {
        socketWrapper.interestOps(SelectionKey.OP_READ);
        PollerEvent pollerEvent = createPollerEvent(socketWrapper, OP_REGISTER);
        addEvent(pollerEvent);
    }

    private void addEvent(PollerEvent event) {
        events.offer(event);
        // wakeupCounter 为 -1 时唤醒 selector，处理新事件
        if (wakeupCounter.incrementAndGet() == 0) {
            selector.wakeup();
        }
    }

    /**
     * 循环处理 poller 事件和 IO 事件
     */
    @Override
    public void run() {
        while (true) {
            boolean hasEvents = false;
            try {
                hasEvents = events();
                // 事件队列有新事件，wakeupCounter 为 -1 时，selector 正在进行 select 操作
                if (wakeupCounter.getAndSet(-1) > 0) {
                    // 检查有没有 IO，有 IO 先处理 IO
                    keyCount = selector.selectNow();
                } else {
                    // 没有新事件就等待已注册 IO 的，默认超时 1000ms
                    keyCount = selector.select(selectorTimeout);
                }
                wakeupCounter.set(0);
                // 没有 socket 可读写
                if (keyCount == 0) {
                    hasEvents = (hasEvents | events());
                }
            } catch (Throwable x) {
                continue;
            }
            // 遍历 selector key，处理 IO 事件
            Iterator<SelectionKey> iterator =
                keyCount > 0 ? selector.selectedKeys().iterator() : null;
            while (iterator != null && iterator.hasNext()) {
                SelectionKey sk = iterator.next();
                iterator.remove();
                NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                processKey(sk, socketWrapper);
            }
            // 处理超时的连接
            timeout(keyCount,hasEvents);
        }
        getStopLatch().countDown();
    }

    /**
     * 处理 poller 事件队列中的事件，如果处理了事件返回 true
     */
    public boolean events() {
        boolean result = false;
        PollerEvent pe = null;
        for (int i = 0, size = events.size(); i < size && (pe = events.poll()) != null; i++ ) {
            result = true;
            NioSocketWrapper socketWrapper = pe.getSocketWrapper();
            SocketChannel sc = socketWrapper.getSocket().getIOChannel();
            // 注册读事件到 selector
            sc.register(getSelector(), SelectionKey.OP_READ, socketWrapper);
        }
        return result;
    }

    /**
     * 处理 IO 事件
     */
    protected void processKey(SelectionKey sk, NioSocketWrapper socketWrapper) {
        if (sk.isReadable() || sk.isWritable()) {
            if (socketWrapper.getSendfileData() != null) {
                processSendfile(sk, socketWrapper, false);
            } else {
                // 先读再写
                if (sk.isReadable()) {
                    processSocket(socketWrapper, SocketEvent.OPEN_READ, true);
                }
                if (sk.isWritable()) {
                    processSocket(socketWrapper, SocketEvent.OPEN_WRITE, true);
                }
            }
        }
    }
}
```

### AprEndpoint

跟 NioEndpoint 一样，AprEndpoint 也实现了非阻塞 I/O，它们的区别是：NioEndpoint 通过调用 Java 的 NIO API 来实现非阻塞 I/O，而 AprEndpoint 是通过 JNI 调用 APR 本地库而实现非阻塞 I/O 的。在某些场景下，比如需要频繁与操作系统进行交互，Socket 网络通信就是这样一个场景， 特别是如果你的 Web 应用使用了 TLS 来加密传输，我们知道 TLS 协议在握手过程中有多 次网络交互，在这种情况下 Java 跟 C 语言程序相比还是有一定的差距，而这正是 APR 的 强项。Tomcat 本身是 Java 编写的，为了调用 C 语言编写的 APR，需要通过 JNI 方式来调用。 JNI（Java Native Interface） 是 JDK 提供的一个编程接口，它允许 Java 程序调用其他语 言编写的程序或者代码库，其实 JDK 本身的实现也大量用到 JNI 技术来调用本地 C 程序 库。

它跟 NioEndpoint 的工作原理很像，有 LimitLatch、Acceptor、Poller、 SocketProcessor 和 Http11Processor，只是 Acceptor 和 Poller 的实现和 NioEndpoint 不同。

Acceptor 接收到一个新的 Socket 连接后，按照 NioEndpoint 的实现，它会把这个 Socket 交给 Poller 去查询 I/O 事件。AprEndpoint 也是这样做的，不过 AprEndpoint 的 Poller 并不是调用 Java NIO 里的 Selector 来查询 Socket 的状态，而是通过 JNI 调用 APR 中的 poll 方法，而 APR 又是调用了操作系统的 epoll API 来实现的。在 AprEndpoint 中，我们可以配置一个叫 deferAccept 的参数，它对应的是 TCP 协议中的 TCP_DEFER_ACCEPT，设置这个参数后，当 TCP 客户端有新的连接请求到达时，TCP 服务端先不建立连接，而是再等等，直到客户端有请求数据发过来时再建立连接。这样的好处是服务端不需要用 Selector 去反复查询请求数据是否就绪。这是一种 TCP 协议层的优化，不是每个操作系统内核都支持，因为 Java 作为一种跨平台语言，需要屏蔽各种操作系统的差异，因此并没有把这个参数提供给用户；但是对于 APR 来说，它的目的就是尽可能提升性能，因此它向用户暴露了这个参数。

### Nio2Endpoint

接口，

Accpetor 的功能就是监听连接，接收并建立连接。它的本质就是调用了四个操作系统 API：socket、bind、listen 和 accept。那 Java 语言如何直接调用 C 语言 API 呢？答案 就是通过 JNI。

其中两个重要组件：Acceptor 和 SocketProcessor。

Acceptor 用于监听 Socket 连接请求，SocketProcessor 用于处理收到的 Socket 请求，提交到线程池 Executor 处理。
