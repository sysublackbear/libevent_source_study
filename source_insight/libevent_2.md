# libevent(2)
@(源码)


### 1.4.evconnlistener_new_bind函数

关于TCP网络编程，server端和client端分别进行的流程：
![Alt text](./1538923728425.png)


位于listener.c，代码如下：
```cpp
// listener = evconnlistener_new_bind(base, listener_cb, (void *)base, LEV_OPT_REUSEABLE|LEV_OPT_CLOSE_ON_FREE, -1, (struct sockaddr*)&sin, sizeof(sin));

// 其中，参数介绍：
// base:事件基础句柄
// cb：事件回调函数
// ptr：额外参数
// flags：额外选项
// backlog：这个参数涉及到一些网络的细节。在进程正理一个一个连接请求的时候，可能还存在其它的连接请求。因为TCP连接是一个过程，所以可能存在一种半连接的状态，有时由于同时尝试连接的用户过多，使得服务器进程无法快速地完成连接请求。如果这个情况出现了，服务器进程希望内核如何处理呢？内核会在自己的进程空间里维护一个队列以跟踪这些完成的连接但服务器进程还没有接手处理或正在进行的连接，这样的一个队列内核不可能让其任意大，所以必须有一个大小的上限。这个backlog告诉内核使用这个数值作为上限。毫无疑问，服务器进程不能随便指定一个数值，内核有一个许可的范围。这个范围是实现相关的。很难有某种统一，一般这个值会小30以内。
// sa：连接套接字信息
// socklen：
struct evconnlistener *
evconnlistener_new_bind(struct event_base *base, evconnlistener_cb cb,
    void *ptr, unsigned flags, int backlog, const struct sockaddr *sa,
    int socklen)
{
	struct evconnlistener *listener;
	evutil_socket_t fd;
	int on = 1;
	int family = sa ? sa->sa_family : AF_UNSPEC;
	int socktype = SOCK_STREAM | EVUTIL_SOCK_NONBLOCK;  // tcp协议+非阻塞

	if (backlog == 0)
		return NULL;

	// 给fd设置close-on-exec的标记
	/*
		关于close-on-exec：
大部分这种问题都能够解决，在文章的最后，提到了一种特殊情况，就是父子进程中的端口占用情况。父进程监听一个端口后，fork出一个子进程，然后kill掉父进程，再重启父进程，这个时候提示端口占用，用netstat查看，子进程占用了父进程监听的端口。

原理其实很简单，子进程在fork出来的时候，使用了写时复制（COW，Copy-On-Write）方式获得父进程的数据空间、 堆和栈副本，这其中也包括文件描述符。刚刚fork成功时，父子进程中相同的文件描述符指向系统文件表中的同一项（这也意味着他们共享同一文件偏移量）。这其中当然也包含父进程创建的socket。

接着，一般我们会调用exec执行另一个程序，此时会用全新的程序替换子进程的正文，数据，堆和栈等。此时保存文件描述符的变量当然也不存在了，我们就无法关闭无用的文件描述符了。所以通常我们会fork子进程后在子进程中直接执行close关掉无用的文件描述符，然后再执行exec。

但是在复杂系统中，有时我们fork子进程时已经不知道打开了多少个文件描述符（包括socket句柄等），这此时进行逐一清理确实有很大难度。我们期望的是能在fork子进程前打开某个文件句柄时就指定好：“这个句柄我在fork子进程后执行exec时就关闭”。其实时有这样的方法的：即所谓 的 close-on-exec。

回到我们的应用场景中来，只要我们在创建socket的时候加上SOCK_CLOEXEC标志，就能够达到我们要求的效果，在fork子进程中执行exec的时候，会清理掉父进程创建的socket。
	*/
	if (flags & LEV_OPT_CLOSE_ON_EXEC)
		socktype |= EVUTIL_SOCK_CLOEXEC;

	// 详见evutil.c，对socket函数封装了一层，初始化socket的函数
	fd = evutil_socket_(family, socktype, 0);
	if (fd == -1)
		return NULL;

	// 设置keepalive属性，保持心跳
	/*
	SO_KEEPALIVE 保持连接检测对方主机是否崩溃，避免（服务器）永远阻塞于TCP连接的输入。设置该选项后，如果2小时内在此套接口的任一方向都没有数据交换，TCP就自动给对方 发一个保持存活探测分节(keepalive probe)。这是一个对方必须响应的TCP分节.它会导致以下三种情况：对方接收一切正常：以期望的ACK响应。2小时后，TCP将发出另一个探测分节。对方已崩溃且已重新启动：以RST响应。套接口的待处理错误被置为ECONNRESET，套接 口本身则被关闭。对方无任何响应：源自berkeley的TCP发送另外8个探测分节，相隔75秒一个，试图得到一个响应。在发出第一个探测分节11分钟15秒后若仍无响应就放弃。套接口的待处理错误被置为ETIMEOUT，套接口本身则被关闭。如ICMP错误是“host unreachable(主机不可达)”，说明对方主机并没有崩溃，但是不可达，这种情况下待处理错误被置为 EHOSTUNREACH
	*/
	if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, (void*)&on, sizeof(on))<0)
		goto err;

	// 给fd设置SO_REUSEADDR属性。对于监听套接字，比较特殊。如果你定义了SO_REUSEADDR，并且让两个套接字在同一个端口上进行接听，那么对于由谁来ACCEPT，就会出现歧义。如果你定义个SO_REUSEADDR，只定义一个套接字在一个端口上进行监听，如果服务器出现意外而导致没有将这个端口释放，那么服务器重新启动后，你还可以用这个端口，因为你已经规定可以重用了，如果你没定义的话，你就会得到提示，ADDR已在使用中。 
	// 我用在多播的时候，也经常使用SO_REUSEADDR，也是为了防止机器出现意外，导致端口没有释放，而使重启后的绑定失败。
	if (flags & LEV_OPT_REUSEABLE) {
		if (evutil_make_listen_socket_reuseable(fd) < 0)
			goto err;
	}

	// 关于SO_REUSEADDR和SO_REUSEPORT的区别
	// 详见：https://blog.csdn.net/yaokai_assultmaster/article/details/68951150
	if (flags & LEV_OPT_REUSEABLE_PORT) {
		if (evutil_make_listen_socket_reuseable_port(fd) < 0)
			goto err;
	}

	// 关于TCP_DEFER_ACCEPT的介绍：
	// 详见：https://blog.csdn.net/dolphin98629/article/details/18308299
	if (flags & LEV_OPT_DEFERRED_ACCEPT) {
		if (evutil_make_tcp_listen_socket_deferred(fd) < 0)
			goto err;
	}

	if (sa) {
		if (bind(fd, sa, socklen)<0)
			goto err;
	}

	// 分配一个监听对象
	listener = evconnlistener_new(base, cb, ptr, flags, backlog, fd);
	if (!listener)
		goto err;

	return listener;
err:
	evutil_closesocket(fd);
	return NULL;
}
```


#### 1.4.1.evconnlistener_new函数

位于listener.c，代码如下：
```cpp
struct evconnlistener *
evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd)
{
	struct evconnlistener_event *lev;

#ifdef _WIN32
	if (base && event_base_get_iocp_(base)) {
		const struct win32_extension_fns *ext =
			event_get_win32_extension_fns_();
		if (ext->AcceptEx && ext->GetAcceptExSockaddrs)
			return evconnlistener_new_async(base, cb, ptr, flags,
				backlog, fd);
	}
#endif

	if (backlog > 0) {
		if (listen(fd, backlog) < 0)
			return NULL;
	} else if (backlog < 0) {
		if (listen(fd, 128) < 0)
			return NULL;
	}

	lev = mm_calloc(1, sizeof(struct evconnlistener_event));
	if (!lev)
		return NULL;

	/*
	static const struct evconnlistener_ops evconnlistener_event_ops = {
		event_listener_enable,
		event_listener_disable,
		event_listener_destroy,
		NULL, // shutdown
		event_listener_getfd,
		event_listener_getbase
	};
	*/
	lev->base.ops = &evconnlistener_event_ops;
	lev->base.cb = cb;
	lev->base.user_data = ptr;
	lev->base.flags = flags;
	lev->base.refcnt = 1;

	lev->base.accept4_flags = 0;
	if (!(flags & LEV_OPT_LEAVE_SOCKETS_BLOCKING))
		lev->base.accept4_flags |= EVUTIL_SOCK_NONBLOCK;
	if (flags & LEV_OPT_CLOSE_ON_EXEC)
		lev->base.accept4_flags |= EVUTIL_SOCK_CLOEXEC;

	if (flags & LEV_OPT_THREADSAFE) {
		EVTHREAD_ALLOC_LOCK(lev->base.lock, EVTHREAD_LOCKTYPE_RECURSIVE);
	}

	// 其中，lev（evconnlistener_event）的结构是：
	/*
	struct evconnlistener_event {
		struct evconnlistener base;
		struct event listener;
	};
	*/

	// event_assign的定义详见1.5
	event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST,
	    listener_read_cb, lev);

	if (!(flags & LEV_OPT_DISABLED))
	    evconnlistener_enable(&lev->base);

	return &lev->base;
}
```
上面函数用于创建和释放evconnlistener。

同时，上面`event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST, listener_read_cb, lev);`也把`evconnlistener_event`的`listener`监听到


### 1.5.evsignal_new函数

位于event.h，代码如下：
```cpp
// signal_event = evsignal_new(base, SIGINT, signal_cb, (void *)base);
#define evsignal_new(b, x, cb, arg)				\
	event_new((b), (x), EV_SIGNAL|EV_PERSIST, (cb), (arg))
```

上面这里展开即是：
```cpp
event_new((base), (SIGINT), EV_SIGNAL|EV_PERSIST, (signal_cb), (arg))
```

`event_new`位于event.c，代码如下：
主要用来创建事件结构体，根据监听事件类型，文件描述符，以及回调函数，回调函数参数等创建，可以看成是事件的初始化过程，主要是设定事件的初始状态，此时事件结构体刚刚创建出来还没有添加到`event_base`的激活或者等待列表中，是孤立存在的，需要调用`event_add`函数将此事件添加到`event_base`中。
```cpp
// 分配新的event结构，并准备好添加。函数event_new返回新的事件，可以用在event_add和event_del函数中。fd和events参数决定了什么条件可以触发这个事件；
// 回调函数和回调函数参数告诉libevent当event激活时需要做什么。
// 如果events包括EV_READ,EV_WRITE,EV_READ|EV_WRITE之一，那么fd是文件描述符或者socket，这两个都是可以在读、写条件下被监控的。
// 如果events包含EV_SIGNAL，那么fd是等待的信号值。
// 如果events不包含上述任何一种，那么事件是定时事件或者通过event_active进行人工触发：如果是这种情况，fd必须是－1。
// EV_PERSIST可以通过events参数传递：它使得event_add永久生效，直到event_del调用；
// EV_ET与EV_READ以及EV_WRITE是兼容的，并且只能被部分后台方法支持。它告诉libevent使用边沿触发事件；
// EV_TIMEOUT此处没有影响；
// 在同一个fd上监听多个events是合法的；但是它们必需都是边沿触发，或者都不是边沿触发。
// 当事件激活时，event loop将运行提供的回调函数，就使用这三个参数：第一个就是提供的fd；第二个是触发的events，EV_READ，EV_WRITE，EV_SIGNAL，EV_TIMEOUT表示超时事件发生，EV_ET表示边沿触发事件发生；第三个参数是提供的callback_arg指针。
// 参数介绍：
// base：需要绑定的base
// fd：需要监听的文件描述符或者信号，或者为－1，如果为－1则是定时事件
// events：事件类型，信号事件和IO事件不能同时存在
// callback：回调函数，信号发生或者IO事件或者定时事件发生时
// callback_arg：传递给回调函数的参数，一般是base
// 返回值：是指事件结构体，必须由event_free释放
// 相关查看event_free，event_add，event_del，event_assign
struct event *
event_new(struct event_base *base, evutil_socket_t fd, short events, void (*cb)(evutil_socket_t, short, void *), void *arg)
{
	struct event *ev;
	ev = mm_malloc(sizeof(struct event));
	if (ev == NULL)
		return (NULL);
	if (event_assign(ev, base, fd, events, cb, arg) < 0) {
		mm_free(ev);
		return (NULL);
	}

	return (ev);
}
```

其中，`event_assign`位于event.c，代码如下：

`event_assign`函数的含义是准备将新创建的`event`添加到`event_base`中，注意，这里的添加到`event_base`中只是将事件和`event_base`关联起来，并不是加入到`event_base`的激活队列或者等待队列中。
```cpp
// event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST, listener_read_cb, lev);
// 准备新的已经分配的事件结构体以添加到event_base中；
// 函数event_assign准备事件结构体ev，以备将来event_add和event_del调用。
// 不像event_new，他没有自己分配内存：他需要你已经分配了一个事件结构体，应该是在堆上分配的；
// 这样做一般会让你的代码依赖于事件结构体的尺寸，并且会造成与未来版本不兼容。
// 避免自己分配出现问题的最简单的办法是使用event_new和event_free来申请和释放。
// 让代码未来兼容的稍微困难的方式是使用event_get_struct_event_size来确定运行时事件的尺寸。
// 注意，如果事件处于激活或者未决状态，调用这个函数是不安全的。这样会破坏libevent内部数据结构，导致一些奇怪的，难以调查的问题。你只能使用event_assign来改变一个存在的事件，但是事件状态不能是激活或者未决的。
// ev：需要修正的事件结构体
// base： 将ev添加到的event_base
// fd： 需要监控的文件描述符
// events： 需要监控的事件类型；可以是EV_READ或者EV_WRITE
// callback：当事件发生时的回调函数
// callback_arg: 传递给回调函数的参数，一般是event_base句柄
// 如果成功则返回0，如果失败则返回－1
// 相关查看 event_new,event_add,event_del,event_base_once,event_get_struct_event_size	   
int
event_assign(struct event *ev, struct event_base *base, evutil_socket_t fd, short events, void (*callback)(evutil_socket_t, short, void *), void *arg)
{
	if (!base)
		base = current_base;  // 没有设置默认值，默认选择全局唯一的event_base：event_global_current_base_
	if (arg == &event_self_cbarg_ptr_)
		arg = ev;

	event_debug_assert_not_added_(ev);

	// 事件结构体初始设置
	ev->ev_base = base;

	ev->ev_callback = callback;
	ev->ev_arg = arg;
	ev->ev_fd = fd;
	ev->ev_events = events;
	ev->ev_res = 0;
	ev->ev_flags = EVLIST_INIT;
	ev->ev_ncalls = 0;
	ev->ev_pncalls = NULL;

	// 如果是信号事件
	if (events & EV_SIGNAL) {
		// 信号事件和IO事件不能同时存在
		if ((events & (EV_READ|EV_WRITE|EV_CLOSED)) != 0) {
			event_warnx("%s: EV_SIGNAL is not compatible with "
			    "EV_READ, EV_WRITE or EV_CLOSED", __func__);
			return -1;
		}
		// 设置事件关闭时执行回调函数的类型：evcb_callback
		ev->ev_closure = EV_CLOSURE_EVENT_SIGNAL;
	} else {
	// 如果是其它类型的事件：IO事件、定时事件
		// 如果事件是永久事件，即每次调用之后不会移除出事件列表
        // 清空IO超时控制，并设置事件关闭时回调函数类型：evcb_callback
		if (events & EV_PERSIST) {
			evutil_timerclear(&ev->ev_io_timeout);
			ev->ev_closure = EV_CLOSURE_EVENT_PERSIST;
		} else {
			// 设置事件回调函数类型：evcb_callback（注意区别上面的EV_CLOSURE_EVENT_SIGNAL）
			ev->ev_closure = EV_CLOSURE_EVENT;
		}
	}

	// void min_heap_elem_init_(struct event* e) { e->ev_timeout_pos.min_heap_idx = -1; }
	// 事件超时控制最小堆初始化
	min_heap_elem_init_(ev);

	if (base != NULL) {
		/* by default, we put new events into the middle priority */
		// 默认情况下，事件优先级设置为中间优先级
		/** The length of the activequeues array */
		ev->ev_pri = base->nactivequeues / 2;
	}

	event_debug_note_setup_(ev);

	return 0;
}
```

作用就是把给定的`event`类型对象的每一个成员赋予一个指定的值。

因此，原文`signal_event = evsignal_new(base, SIGINT, signal_cb, (void *)base);`的意思是：

新建一个事件，当接收到SIGINIT信号（*程序终止(interrupt)信号, 在用户键入INTR字符(通常是Ctrl-C)时发出，用于通知前台进程组终止进程。*）的时候，调用回调函数`signal_cb`，函数的参数为`base`。

### 1.6.event_add函数

位于event.c，代码如下：
```cpp
// event_add(signal_event, NULL);
int
event_add(struct event *ev, const struct timeval *tv)
{
	int res;

	/*
	// Replacement for assert() that calls event_errx on failure. 
#ifdef NDEBUG
#define EVUTIL_ASSERT(cond) EVUTIL_NIL_CONDITION_(cond)
#define EVUTIL_FAILURE_CHECK(cond) 0
#else
#define EVUTIL_ASSERT(cond)						\
	do {								\
		if (EVUTIL_UNLIKELY(!(cond))) {				\
			event_errx(EVENT_ERR_ABORT_,			\
			    "%s:%d: Assertion %s failed in %s",		\
			    __FILE__,__LINE__,#cond,__func__);		\
			// In case a user-supplied handler tries to 	\
			// return control to us, log and abort here. 	\
			(void)fprintf(stderr,				\
			    "%s:%d: Assertion %s failed in %s",		\
			    __FILE__,__LINE__,#cond,__func__);		\
			abort();					\
		}							\
	} while (0)
#define EVUTIL_FAILURE_CHECK(cond) EVUTIL_UNLIKELY(cond)
#endif
	*/
	if (EVUTIL_FAILURE_CHECK(!ev->ev_base)) {
		event_warnx("%s: event has no event_base set.", __func__);
		return -1;
	}

	// #define EVBASE_ACQUIRE_LOCK(base, lock) EVUTIL_NIL_STMT_
	// #define EVUTIL_NIL_STMT_ ((void)0)
	EVBASE_ACQUIRE_LOCK(ev->ev_base, th_base_lock);

	res = event_add_nolock_(ev, tv, 0);

	// #define EVBASE_RELEASE_LOCK(base, lock) EVUTIL_NIL_STMT_
	EVBASE_RELEASE_LOCK(ev->ev_base, th_base_lock);

	return (res);
}
```
可见，这是一个先上锁（用户自定义的锁），再往`event_base`里面加入`event`的操作。

`event_add`做这样的一个事情：
>将事件增加到一系列等待的事件中；

>函数event_add规划以下事情：当event_assign或者event_new函数

>创建’ev’时指定的条件发生时，或者超时时间达到时，事件’ev’的执行情况。

>如果超时时间为NULL，那么没有超时事件发生，只能等匹配的事件发生时才会调用回调函数；

>参数ev中的事件必须已经通过event_assign或者event_new进行过初始化了，而且只能当事件不再处于

>等待状态时才能调用event_assign或者event_new函数。

>如果ev参数中的事件有特定的超时时间，调用event_add时如果指定tc不为NULL的话，就会替代老的超时时间；

>参数 ev：通过event_assign或者event_new初始化过的事件

>参数timeout：等待事件执行的最长时间，如果NULL则从来不会超时，即永久等待

>成功则返回0，失败则－1

>相关查看event_del，event_assign，event_new


#### 1.6.1.event_add_nolock_函数

位于event.c，代码如下：
```cpp
/* Implementation function to add an event.  Works just like event_add,
 * except: 1) it requires that we have the lock.  2) if tv_is_absolute is set,
 * we treat tv as an absolute time, not as an interval to add to the current
 * time */
int
event_add_nolock_(struct event *ev, const struct timeval *tv,
    int tv_is_absolute)
{
	struct event_base *base = ev->ev_base;
	int res = 0;
	int notify = 0;

	EVENT_BASE_ASSERT_LOCKED(base);
	event_debug_assert_is_setup_(ev);

	event_debug((
		 "event_add: event: %p (fd "EV_SOCK_FMT"), %s%s%s%scall %p",
		 ev,
		 EV_SOCK_ARG(ev->ev_fd),
		 ev->ev_events & EV_READ ? "EV_READ " : " ",
		 ev->ev_events & EV_WRITE ? "EV_WRITE " : " ",
		 ev->ev_events & EV_CLOSED ? "EV_CLOSED " : " ",
		 tv ? "EV_TIMEOUT " : " ",
		 ev->ev_callback));

	EVUTIL_ASSERT(!(ev->ev_flags & ~EVLIST_ALL));

	if (ev->ev_flags & EVLIST_FINALIZING) {
		/* XXXX debug */
		return (-1);
	}

	/*
	 * prepare for timeout insertion further below, if we get a
	 * failure on any step, we should not change any state.
	 */
	if (tv != NULL && !(ev->ev_flags & EVLIST_TIMEOUT)) {
		if (min_heap_reserve_(&base->timeheap,
			1 + min_heap_size_(&base->timeheap)) == -1)  // 往堆里面添加元素
			// Priority queue of events with timeouts.
			// 带有超时时间的时间排成的优先队列
			struct min_heap timeheap;
			return (-1);  /* ENOMEM == errno */
	}

	/* If the main thread is currently executing a signal event's
	 * callback, and we are not the main thread, then we want to wait
	 * until the callback is done before we mess with the event, or else
	 * we can race on ev_ncalls and ev_pncalls below. */
#ifndef EVENT__DISABLE_THREAD_SUPPORT
// 多线程模式
	if (base->current_event == event_to_event_callback(ev) &&
	    (ev->ev_events & EV_SIGNAL)
	    && !EVBASE_IN_THREAD(base)) {
		// 如果当前线程正在执行某个事件的回调函数，则把该事件添加到等待队里里面(`current_event_waiters`)
		++base->current_event_waiters;
		// Number of threads blocking on current_event_cond.
		// 当前事件的条件变量阻塞的线程个数
		// int current_event_waiters; 
		EVTHREAD_COND_WAIT(base->current_event_cond, base->th_base_lock);
	}
#endif

	// 如果事件类型是IO事件／信号事件，同时事件状态不是已经插入／激活／下一次激活状态，
    // 则根据事件类型将事件添加到不同的映射表或者队列中
	if ((ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED|EV_SIGNAL)) &&
	    !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE|EVLIST_ACTIVE_LATER))) {
		if (ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED))
			// 如果事件是IO事件，则将事件插入到IO事件与文件描述符的映射表中
			res = evmap_io_add_(base, ev->ev_fd, ev);
		else if (ev->ev_events & EV_SIGNAL)
			// 如果事件是信号事件，则将事件插入信号与文件描述符的映射表中
			res = evmap_signal_add_(base, (int)ev->ev_fd, ev);
		if (res != -1)
			// 如果上述添加行为正确，则将事件插入到event_base的事件列表中
			event_queue_insert_inserted(base, ev);
		if (res == 1) {
			// 插入成功本身就会返回1
			// 如果上述添加行为正确，则设置通知主线程的标志，因为已经添加了新事件
			// 防止1）优先级高的事件被优先级低的事件倒挂，2）防止主线程忙等，会通知主线程有新事件
			notify = 1;
			res = 0;
		}
	}

	/*
	 * we should change the timeout state only if the previous event
	 * addition succeeded.
	 */
	// 只有当前面事件条件成功执行之后，才能改变超时状态
	if (res != -1 && tv != NULL) {
		struct timeval now;
		int common_timeout;
#ifdef USE_REINSERT_TIMEOUT
		int was_common;
		int old_timeout_idx;
#endif

		/*
		 * for persistent timeout events, we remember the
		 * timeout value and re-add the event.
		 *
		 * If tv_is_absolute, this was already set.
		 */
		// 对于持久化的定时事件，需要记住超时时间，并重新注册事件
		// 如果tv_is_absolute设置，则事件超时时间就等于输入时间参数
		// 相当于，持久化事件，直接更新其超时时间就可以。
		if (ev->ev_closure == EV_CLOSURE_EVENT_PERSIST && !tv_is_absolute)
			ev->ev_io_timeout = *tv;

#ifndef USE_REINSERT_TIMEOUT
		// 如果没有使用USE_REINSERT_TIMEOUT，则当事件处于超时状态时，需要从队列中移除事件
		// 因为同样的事件不能重新插入，所以当一个事件已经处于超时状态时，为防止执行，需要先移除后插入 
		if (ev->ev_flags & EVLIST_TIMEOUT) {
			event_queue_remove_timeout(base, ev);
		}
#endif

		/* Check if it is active due to a timeout.  Rescheduling
		 * this timeout before the callback can be executed
		 * removes it from the active list. */
		// 检查事件当前状态是否已经激活，而且是超时事件的激活状态，
		// 则在回调函数执行之前，需要重新调度这个超时事件，因此需要把它移出激活队列
		if ((ev->ev_flags & EVLIST_ACTIVE) &&
		    (ev->ev_res & EV_TIMEOUT)) {
			if (ev->ev_events & EV_SIGNAL) {
				/* See if we are just active executing
				 * this event in a loop
				 */
				if (ev->ev_ncalls && ev->ev_pncalls) {
					// TODO:目前不知道什么意思
					*ev->ev_pncalls = 0;
				}
			}
			// 将此事件的回调函数从激活队列中移除
			event_queue_remove_active(base, event_to_event_callback(ev));
		}
		// 获取base中的缓存时间
		gettime(base, &now);

		// 检查base是否使用了公用超时队列机制
		common_timeout = is_common_timeout(tv, base);
#ifdef USE_REINSERT_TIMEOUT
		was_common = is_common_timeout(&ev->ev_timeout, base);
		old_timeout_idx = COMMON_TIMEOUT_IDX(&ev->ev_timeout);
#endif

		// 下面有三种计算事件超时时间的机制
		// 1.如果设置绝对超时时间，则设置时间超时时间为输入时间参数（函数入参）
		// 2.如果使用的公用超时队列机制，则根据当前base中时间和输入超时时间间隔计算出时间超时时间，并对超时时间进行公用超时掩码计算
		// 3.如果是其他情况，则直接根据base中时间和输入超时时间间隔计算事件的超时时间
		if (tv_is_absolute) {
			ev->ev_timeout = *tv;
		} else if (common_timeout) {
			struct timeval tmp = *tv;
			tmp.tv_usec &= MICROSECONDS_MASK;
			evutil_timeradd(&now, &tmp, &ev->ev_timeout);
			ev->ev_timeout.tv_usec |=
			    (tv->tv_usec & ~MICROSECONDS_MASK);
		} else {
			evutil_timeradd(&now, tv, &ev->ev_timeout);
		}

		event_debug((
			 "event_add: event %p, timeout in %d seconds %d useconds, call %p",
			 ev, (int)tv->tv_sec, (int)tv->tv_usec, ev->ev_callback));

// 将事件插入超时队列
#ifdef USE_REINSERT_TIMEOUT
		// event_queue_reinsert_timeout会插入两个队列：一个是公用超时队列，一个超时队列
		event_queue_reinsert_timeout(base, ev, was_common, common_timeout, old_timeout_idx);
#else
		// 只会插入超时队列
		event_queue_insert_timeout(base, ev);
#endif

		if (common_timeout) {
			// 如果使用了公用超时队列机制，则需要根据当前事件的超时时间将当前事件插入具有相同超时时间的时间列表
			struct common_timeout_list *ctl =
			    get_common_timeout_list(base, &ev->ev_timeout);
			if (ev == TAILQ_FIRST(&ctl->events)) {
				// 如果当前事件是公用超时队列的第一个事件，则因此需要将此超时事件插入最小堆
				// 解释：公用超时队列机制：处于同一个公用超时队列中的所有事件具有相同的超时控制，因此只需要将公用超时队列
				// 的第一个事件插入最小堆，当超时触发时，可以通过遍历公用超时队列获取同样的超时事件。
				common_timeout_schedule(ctl, &now, ev);
			}
		} else {
			struct event* top = NULL;
			/* See if the earliest timeout is now earlier than it
			 * was before: if so, we will need to tell the main
			 * thread to wake up earlier than it would otherwise.
			 * We double check the timeout of the top element to
			 * handle time distortions due to system suspension.
			 */
			// 查看当前事件是否位于最小堆根部，如果是，则需要通知主线程
			// 否则，需要查看最小堆根部超时时间是否已经小于当前时间，即已经超时了，如果是，则需要通知主线程
			if (min_heap_elt_is_top_(ev))
				notify = 1;
			else if ((top = min_heap_top_(&base->timeheap)) != NULL &&
					 evutil_timercmp(&top->ev_timeout, &now, <))
				notify = 1;
		}
	}

	/* if we are not in the right thread, we need to wake up the loop */
	// 上述操作成功，通知主线程
	if (res != -1 && notify && EVBASE_NEED_NOTIFY(base))
		evthread_notify_base(base);

	event_debug_note_add_(ev);

	return (res);
}
```


看到这里，我已经觉得这里面的数据结构种类很多，故我们先打住demo的代码走读，重新去梳理有关事件的特性。


