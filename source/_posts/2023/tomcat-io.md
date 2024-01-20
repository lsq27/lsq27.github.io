---
title: Tomcat 的三种 IO 模型
date: 2023-08-22 17:13:45
tags:
---

在用 Java 手写服务器示例代码的时候有一些疑惑，加之日常开发中只对 Tomcat 有浅显的理解，知道其是用线程池处理请求，所以就翻了翻 Tomcat 的源代码。本文记录 Tomcat 中 IO 模型的相关代码，版本为 tomcat-embed-core:9.0.78。

<!-- more -->

Tomcat 支持的 IO 模型有 APR、NIO、NIO2 三种。

APR、NIO 两种实现方式本质上都是 reactor 模式，使用 IO 多路复用和线程池实现并发处理多个客户端请求。使用的是主从 reactor 多线程模型。其中 acceptor 为主 reactor，负责接受新连接。poller 为从 reactor，负责监控 IO 事件，worker 为执行 IO 操作和业务代码的线程池。

## endpoint

Tomcat 中封装 IO 相关方法的类叫做 endpoint，负责创建 acceptor, poller, worker 等对象，并负责这几个类之间的交互，作用是胶水代码。
抽象类为 AbstractEndpoint，具体的子类有 AprEndpoint、NioEndpoint 和 Nio2Endpoint，分别代表 APR、NIO 和 NIO2 三种 IO 模型。

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

该类将在 Tomcat 10 被废弃，APR 通过 JNI 进行 socket 操作，相比使用 Java IO API 效率更高。
使用该类会创建以下线程（池）

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
     * 把socket交给poller处理
     */
    protected class SocketWithOptionsProcessor implements Runnable {
        @Override
        public void run() {
            Lock lock = socket.getLock();
            lock.lock();
            try {
                getPoller().add(socket.getSocket().longValue(), getConnectionTimeout(), Poll.APR_POLLIN);
            } finally {
                lock.unlock();
            }
        }
    }
}
```

### NioEndpoint

NioEndpoint 通过 Java NIO 进行同步非阻塞 IO 操作。相比使用同步阻塞 IO，大大减少了高并发下所需线程的数量，因而效率更高。

使用该类会创建以下线程（池）

- acceptor 线程（阻塞接受新连接）
- poller 线程（配置 IO 多路复用，检测 IO 事件）
- worker 线程池（处理 IO 事件）

acceptor 循环接受连接并将注册事件放入 poller 的事件队列，poller 循环处理事件队列中的事件并检测 IO 事件，如果发生 IO 事件让 worker 进行 IO 操作并执行业务代码。

功能示意图如下

![NIO](nio.png)

向服务器发出一个请求，查看线程状态。编号 1-10 的线程为 worker 线程。

![NIO thread](nio线程.png)

收到请求后，一个 worker 线程由 Park 状态转为 Running 状态；接着服务器访问外部服务，等待 IO 期间线程转为 Wait 状态；访问外部服务完成后整个请求结束，线程又转为 Park 状态。

NIO 相关代码如下

```java
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
        // 阻塞接受连接
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
            // 因使用 IO 多路复用，设置为非阻塞 IO
            socket.configureBlocking(false);
            if (getUnixDomainSocketPath() == null) {
                socketProperties.setProperties(socket.socket());
            }

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            // 将 socket 注册到 poller
            poller.register(socketWrapper);
            return true;
        } catch (Throwable t) {
        }
        return false;
    }
}
```

### Nio2Endpoint

Nio2Endpoint 通过 Java NIO 进行异步 IO 操作，相比使用同步非阻塞 IO，应用不需将数据拷贝到进程中，而由内核代为完成后通知应用，执行回调。Java 异步 IO 在 Windows 中通过 IOCP 实现，在 Linux 中通过 epoll 模拟实现，所以理论上在 Linux 下同步非阻塞 IO 和异步 IO 的效率无明显差别。

使用该类会创建以下线程（池）

- acceptor 线程
- worker 线程池

相比同步非阻塞 IO，少了 poller，操作系统承担了 poller 的工作和。

功能示意图如下

![NIO2](nio2.png)

向服务器发出一个请求，查看线程状态。编号 1-10 的线程为 worker 线程。

![NIO2 thread](nio2线程.png)

收到请求后，一个 worker 线程由 Park 状态转为 Running 状态；接着服务器访问外部服务，等待 IO 期间线程转为 Wait 状态；访问外部服务完成后整个请求结束，线程又转为 Park 状态。

NIO2 相关代码如下

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
        // 使用 worker 线程池处理 IO 事件
        threadGroup = AsynchronousChannelGroup.withThreadPool((ExecutorService) getExecutor());
        serverSock = AsynchronousServerSocketChannel.open(threadGroup);
        socketProperties.setProperties(serverSock);
        InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
        // 绑定地址
        serverSock.bind(addr, getAcceptCount());
    }

    /**
     * 启动 NIO2 endpoint, 启动 acceptor
     */
    @Override
    public void startInternal() throws Exception {
        // 创建 worker 线程池
        if (getExecutor() == null) {
            createExecutor();
        }
        // 启动 acceptor 线程
        startAcceptorThread();
    }

    /**
     * 启动 NIO2 acceptor
     */
    @Override
    protected void startAcceptorThread() {
        if (acceptor == null) {
            acceptor = new Nio2Acceptor(this);
            acceptor.setThreadName(getName() + "-Acceptor");
        }
        // 使用 worker 线程池启动 accepor
        getExecutor().execute(acceptor);
    }

    /**
     * 阻塞接受连接，该方法未被调用
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
            // 调用 AbstractEndpoint.processSocket 方法处理 IO 事件。因为是异步 IO，所以不需要使用新线程处理 IO 事件
            return processSocket(socketWrapper, SocketEvent.OPEN_READ, false);
        } catch (Throwable t) {
        }
        return false;
    }
}
```

```java
protected class SocketProcessor extends SocketProcessorBase<Nio2Channel> {
    /**
     * 被 AbstractEndpoint.processSocket 调用，处理 IO 事件
     */
    @Override
    protected void doRun() {
        boolean launch = false;
        try {
            SocketState state = SocketState.OPEN;
            // 处理 IO 事件
            if (event == null) {
                state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
            } else {
                state = getHandler().process(socketWrapper, event);
            }
            if (state == SocketState.UPGRADING) {
                launch = true;
            }
        } finally {
            if (launch) {
                try {
                    // 使用新线程处理 IO 事件
                    getExecutor().execute(new SocketProcessor(socketWrapper, SocketEvent.OPEN_READ));
                } catch (NullPointerException npe) {
                }
            }
        }
    }
}
```

## acceptor

acceptor 的功能如其名称，用于接受 socket 连接请求。产生一个与客户端的连接的 socket 后，再将其交给 poller 处理。
acceptor 的代码逻辑特别简单，不断地调用 `endpoint.serverSocketAccept` 产生 socket 连接，调用 `endpoint.setSocketOptions` 设置 socket。

### Acceptor

NIO、APR 的 acceptor 为类 Acceptor，Acceptor 主要代码如下

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

### Nio2Acceptor

NIO2 中的 acceptor 为类 Nio2Acceptor，Nio2Acceptor 重写了 Acceptor 的 `run` 方法，用异步 IO 替代同步 IO。且实现了 CompletionHandler 接口，用于处理异步 IO 完成事件。

Nio2Acceptor 的主要代码如下，其中没有调用 `endpoint.serverSocketAccept` 产生 socket，而是直接调用 `serverSock.accept` 产生 socket，另一个异同是 Nio2Acceptor 的 run 方法中没有循环，而是通过异步 IO 的回调函数重新接受新请求。

```java
protected class Nio2Acceptor extends Acceptor<AsynchronousSocketChannel>
    implements CompletionHandler<AsynchronousSocketChannel, Void> {
    @Override
    public void run() {
        // 如果达到最大连接数，等待
        countUpOrAwaitConnection();
        // 异步 IO，连接完成后由 completed 方法处理，失败由 failed 方法处理
        serverSock.accept(null, this);
    }

    /**
     * 处理连接请求成功
     */
    @Override
    public void completed(AsynchronousSocketChannel socket,
            Void attachment) {
        // 异步接受新请求
        if (getMaxConnections() == -1) {
            serverSock.accept(null, this);
        } else if (getConnectionCount() < getMaxConnections()) {
            countUpOrAwaitConnection();
            serverSock.accept(null, this);
        } else {
            // 达到最大连接数，使用新线程接受请求
            getExecutor().execute(this);
        }
        // 设置 socket
        if (!setSocketOptions(socket)) {
            closeSocket(socket);
        }
    }

    /**
     * 处理连接请求失败
     */
    @Override
    public void failed(Throwable t, Void attachment) {
        if (getMaxConnections() == -1) {
            serverSock.accept(null, this);
        } else {
            // 使用新线程接受请求
            getExecutor().execute(this);
        }
        countDownConnection();
        // 失败后等待一段事件
        errorDelay = handleExceptionWithDelay(errorDelay);
    }
}
```

## poller

poller 的作用是将新 socket 注册到 selector 并轮询是否有 IO 事件发生。

### APR poller

```java
public class Poller implements Runnable {
    /**
     * 启动线程
     */
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
                // poll 操作
                int rv = Poll.poll(aprPoller, pollTime, desc, true);
                for (int n = 0; n < rv; n++) {
                    processSocket();
                }
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
