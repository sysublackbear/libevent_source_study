#libevent(5)
@(源码)


###1.28.信号事件机制分析

简单来说，就是将外部信号转换为内部io事件来处理。

由于信号捕捉函数是全局绑定的，所以没办法像IO事件一样，将IO事件和文件描述符绑定在一起，而libevent又需要将IO事件、信号事件、定时事件都采用事件触发机制来实现，那么对于信号事件来说，就需要一层中间中转来实现，将信号事件在通知`event_base`时转换成IO事件，具体来说，就是在信号绑定时没办法改变全局绑定的实现，只能在通知`event_base`上作出改变了，全过程如下：

1. libevent在内部创建信号通知管道，用于内部传递捕捉到的信号；
2. 管道的两端分别是`event_base`和信号捕捉函数；
3. 将管道的一端文件描述符和信号触发事件的回调函数绑定在一块，注册到`event_base`，一旦管道中有信号传来，回调函数就会处理管道中的信号，因此`event_base`只需要后台方法监听此内部管道即可得知是否有信号事件来临。
4. 在添加信号事件时，会将信号绑定到全局信号捕捉函数，信号捕捉函数中的处理逻辑是一旦捕捉到信号，就将信号写入内部通知管道，这样，信号捕捉函数只需要将信号写入管道，就可以通知`event_base`信号发生。

全过程如下：
信号     —>     信号捕捉函数     —>     写入内部管道     —>     内部信号回调函数     —>     激活外部信号回调函数

![Alt text](./1538497121699.png)

###1.29.初始化内部信号事件函数——evsig_init_函数

位于signal.c，`evsig_init_`函数就是初始化内部信号事件，`epoll`、`poll`、`select`等后台方法在初始化时都会执行内部信号事件的初始化，为下一步信号事件注册做准备。代码如下：

```cpp
// 主要是将信号事件base->sig.ev_signal_pair[0]绑定到base上，信号关联文件描述符；
// base->sig.ev_signal_pair是文件描述符对，一个用来读取数据的，一个用来写数据的。
// 其中base->sig.ev_signal_pair[0],即用来读取信号的；base->sig.ev_signal_pair[1]是用来写信号的。
// 此处只是绑定0，因为这是信号接收端；
// 只有真有外部信号事件注册时，才会绑定base->sig.ev_signal_pair[0]
int
evsig_init_(struct event_base *base)
{
	/*
	 * Our signal handler is going to write to one end of the socket
	 * pair to wake up our event loop.  The event loop then scans for
	 * signals that got delivered.
	 */
	// 信号句柄将会写入socket对的其中一个，用来唤醒event_loop
	// event_loop会检查这些传输的信号
	// 创建内部使用的通知管道，将创建好的文件描述符存在ev_signal_pair中
	// 下面会分析
	if (evutil_make_internal_pipe_(base->sig.ev_signal_pair) == -1) {
#ifdef _WIN32
		/* Make this nonfatal on win32, where sometimes people
		   have localhost firewalled. */
		event_sock_warn(-1, "%s: socketpair", __func__);
#else
		event_sock_err(1, -1, "%s: socketpair", __func__);
#endif
		return -1;
	}

	// 如果存在原有信号句柄则释放，并置空
	if (base->sig.sh_old) {
		// base->sig的类型：
		// typedef void (*ev_sighandler_t)(int);
		mm_free(base->sig.sh_old);
	}
	base->sig.sh_old = NULL;
	base->sig.sh_old_max = 0;

	// evsig_info的数据结构
	/*
	struct evsig_info {
		// Event watching ev_signal_pair[1]    
		// 监听ev_signal_pair[1]的事件，即内部信号管道监听事件
		struct event ev_signal;
		// Socketpair used to send notifications from the signal handler
		// 内部信号管道的两端，0的一端用来读信号，1的一端用来写信号
		evutil_socket_t ev_signal_pair[2];
		// True iff we've added the ev_signal event yet.
		// 如果我们已经增添了ev_signal则为真，即注册了外部信号事件的话
		int ev_signal_added;
		// Count of the number of signals we're currently watching.     
		// 用来统计当前event_base到底监听了多少信号
		int ev_n_signals_added;
		// Array of previous signal handler objects before Libevent started
		// messing with them.  Used to restore old signal handlers.
		// 在libevent开始信号处理之前的原有信号句柄     
#ifdef EVENT__HAVE_SIGACTION
		struct sigaction **sh_old;
#else
		ev_sighandler_t **sh_old;
#endif
		// Size of sh_old.
		// 原有的信号句柄的最大个数
		int sh_old_max;
	}
	*/


	// 将信号事件base->sig.ev_signal绑定在base上，关联文件描述符实base->sig.ev_signal_pair[0]，ev_signal_pair[0]是用来读的，事件类型是EV_READ|EV_PERSIST，包括可读和持久化，事件发生时的回调函数是evsig_cb函数，回调函数的参数是base。
	// 这里就说明了，base->sig.ev_signal是内部事件，是用来外部信号发生时，通知内部信号处理函数的；其实就是将外部信号转换为内部IO事件来处理
	// 将对信号的处理base->sig.ev_signal(struct event)绑定到base里面。
	// 回调函数为evsig_cb

	// EV_READ：可读事件
	// EV_PERSIST：持久化事件
	event_assign(&base->sig.ev_signal, base, base->sig.ev_signal_pair[0],
		EV_READ | EV_PERSIST, evsig_cb, base);

	// 设置信号事件的事件状态为EVLIST_INTERNAL，即内部事件，区分内外部事件
	base->sig.ev_signal.ev_flags |= EVLIST_INTERNAL;
	// 设置信号事件的优先级为0，即最高优先级，信号事件优先处理
	event_priority_set(&base->sig.ev_signal, 0);  // ev->ev_pri = pri;

	// 设置信号事件的后台处理方法，evsigops
	// 下文分析信号处理后台方法的具体执行
	// static const struct eventop evsigops = {
	//     "signal",
	//     NULL,
	//     evsig_add,
	//     evsig_del,
	//     NULL,
	//     NULL,
	//     0, 0, 0 
	// };
	// 下文分析evsig_add和evsig_del函数
	// 每当信号事件注册时，就会调用evsig_add将信号注册到event_base中
	base->evsigsel = &evsigops;  // 注意与evsel区分。

	return 0;
}
```


###1.30.创建内部管道函数——evutil_make_internal_pipe_

位于evutil.c，主要是创建内部管道，用来作为外部信号捕捉函数与内部IO事件回调函数之间通知管道。代码如下：

```cpp
// evutil_make_internal_pipe_(base->sig.ev_signal_pair)

/* Internal function: Set fd[0] and fd[1] to a pair of fds such that writes on
 * fd[1] get read from fd[0].  Make both fds nonblocking and close-on-exec.
 * Return 0 on success, -1 on failure.
 */
// 内部函数：设置fd[0] 和fd[1]为文件描述符对，这样就可以从fd[0]读，从fd[1]写，这两个fds的模式都为nonblocking以及close-on-exec
// close-on-exec的介绍: 子进程在fork父进程时，使用写时复制方式获取父进程的数据空间、堆和栈副本，其中包括文件描述符，刚fork成功时，父子进程中相同的文件描述符指向文件表中的同一项（即共享同一文件偏移量），这其中当然也包括父进程创建的socket；接着，一般会调用exec执行另外一个程序，此时会用全新的程序替换子进程的正文、数据、堆和栈等，此时保存的文件描述符的变量当然也不存在了，我们就无法关闭无用的文件描述符了，通常会使用fork子进程后在子进程中直接执行close关掉无用的文件描述符，然后在执行exec。但是复杂系统中，有时fork子进程时不知道打开了多少个文件描述符，逐一清理的话比较难，期望是fork子进程前打开某个文件描述符时就指定好：此描述符在fork子进程执行exec时就关闭，即指定打开模式为close-on-exec。
int
evutil_make_internal_pipe_(evutil_socket_t fd[2])
{
	/*
	  Making the second socket nonblocking is a bit subtle, given that we
	  ignore any EAGAIN returns when writing to it, and you don't usally
	  do that for a nonblocking socket. But if the kernel gives us EAGAIN,
	  then there's no need to add any more data to the buffer, since
	  the main thread is already either about to wake up and drain it,
	  or woken up and in the process of draining it.
	*/

// 指定第二个socket非阻塞有一些微妙，这使得我们当写这个socket时需要忽略任何EAGAIN返回结果，而一般情况下不能对非阻塞socket这样处理。但是如果内核返回EAGAIN，那么不需要向缓存中添加任何数据，因为主线程已经或者打算唤醒然后读完它，或者已经被唤醒并正在读取它。

// 如果支持pipe2接口，则使用pipe2接口创建管道
// 注意：这里只对fd[0]进行操作
#if defined(EVENT__HAVE_PIPE2)
	if (pipe2(fd, O_NONBLOCK|O_CLOEXEC) == 0)
		return 0;
#endif
// 如果没有PIPE2，有PIPE，则使用pipe+设置fd属性接口创建
#if defined(EVENT__HAVE_PIPE)
	if (pipe(fd) == 0) {
		if (evutil_fast_socket_nonblocking(fd[0]) < 0 ||
		    evutil_fast_socket_nonblocking(fd[1]) < 0 ||
		    evutil_fast_socket_closeonexec(fd[0]) < 0 ||
		    evutil_fast_socket_closeonexec(fd[1]) < 0) {
			close(fd[0]);
			close(fd[1]);
			fd[0] = fd[1] = -1;
			return -1;
		}
		return 0;
	} else {
		event_warn("%s: pipe", __func__);
	}
#endif

// 如果既没有PIPE2，也没有PIPE，则使用套接字
#ifdef _WIN32
#define LOCAL_SOCKETPAIR_AF AF_INET
#else
#define LOCAL_SOCKETPAIR_AF AF_UNIX
#endif
	if (evutil_socketpair(LOCAL_SOCKETPAIR_AF, SOCK_STREAM, 0, fd) == 0) {
		if (evutil_fast_socket_nonblocking(fd[0]) < 0 ||
		    evutil_fast_socket_nonblocking(fd[1]) < 0 ||
		    evutil_fast_socket_closeonexec(fd[0]) < 0 ||
		    evutil_fast_socket_closeonexec(fd[1]) < 0) {
			evutil_closesocket(fd[0]);
			evutil_closesocket(fd[1]);
			fd[0] = fd[1] = -1;
			return -1;
		}
		return 0;
	}
	fd[0] = fd[1] = -1;
	return -1;
}
```


###1.31.信号绑定event_base函数——event_assign函数

详见1.5

位于event.c，代码如下：

```cpp
// event_assign(&base->sig.ev_signal, base, base->sig.ev_signal_pair[0], EV_READ | EV_PERSIST, evsig_cb, base);

// 准备新的已经分配的事件结构体以添加到event_base中；函数event_assign准备事件结构体ev，以备将来event_add和event_del调用。不像event_new，他没有自己分配内存：他需要你已经分配了一个事件结构体，应该是在堆上分配的；这样做一般会让你的代码依赖于事件结构体的尺寸，并且会造成与未来版本不兼容。避免自己分配出现问题的最简单的办法是使用event_new和event_free来申请和释放。让代码未来兼容的稍微困难的方式是使用event_get_struct_event_size来确定运行时事件的尺寸。注意，如果事件处于激活或者未决状态，调用这个函数是不安全的。这样会破坏libevent内部数据结构，导致一些奇怪的，难以调查的问题。你只能使用event_assign来改变一个存在的事件，但是事件状态不能是激活或者未决的。
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
		base = current_base;
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
		// 信号事件与IO事件不能同时存在
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
			ev->ev_closure = EV_CLOSURE_EVENT;
		}
	}

	// 事件超时控制最小堆初始化
	min_heap_elem_init_(ev);

	if (base != NULL) {
		/* by default, we put new events into the middle priority */
		// 默认情况下，事件优先级设置为中间优先级
		ev->ev_pri = base->nactivequeues / 2;
	}

	event_debug_note_setup_(ev);

	return 0;
}
```


###1.32.回调函数——evsig_cb函数

内部信号通知管道检测到有信号发生时的回调函数，位于signal.c，代码如下：

```cpp
// arg = base;
/* Callback for when the signal handler write a byte to our signaling socket */
// 当信号句柄往信号处理socket中写入字节时的回调函数
// arg一般是event_base句柄
// 每当注册信号发生时，就会将该信号绑定的回调函数插入到激活队列中
static void
evsig_cb(evutil_socket_t fd, short what, void *arg)
{
	// 用来存储信号通知管道的数据，每个字节代表一个信号
	static char signals[1024];
	ev_ssize_t n;
	int i;
	// NSIG是linux系统定义的常量
	// 用来存储捕捉的信号，每个数组变量代表一个信号
	int ncaught[NSIG];
	struct event_base *base;

	base = arg;

	// 清空存储捕捉信号的缓存
	memset(&ncaught, 0, sizeof(ncaught));

	// 循环读取信号管道中的信号，直到读完或者无法读取为止。
	// 并将读取的信号，并记录信号发生的次数
	while (1) {
#ifdef _WIN32
		n = recv(fd, signals, sizeof(signals), 0);
#else
		n = read(fd, signals, sizeof(signals));
#endif
		if (n == -1) {
			int err = evutil_socket_geterror(fd);
			if (! EVUTIL_ERR_RW_RETRIABLE(err))
				event_sock_err(1, fd, "%s: recv", __func__);
			break;
		} else if (n == 0) {
			/* XXX warn? */
			break;
		}
		// 遍历缓存中的每个字节，转换成信号值
		// 如果信号值合法，则表示该信号发生一次，增加该信号发生次数
		for (i = 0; i < n; ++i) {
			ev_uint8_t sig = signals[i];
			if (sig < NSIG)
				ncaught[sig]++;
		}
	}

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
	for (i = 0; i < NSIG; ++i) {
		if (ncaught[i])
			// 遍历当前信号发生次数，激活与信号相关的事件，实际上是将信号相关的回调函数插入到激活事件队列中。
			evmap_signal_active_(base, i, ncaught[i]);
	}
	EVBASE_RELEASE_LOCK(base, th_base_lock);
}
```

其中，`evmap_signal_active_`位于evmap.c，代码如下：

```cpp
void
evmap_signal_active_(struct event_base *base, evutil_socket_t sig, int ncalls)
{
	struct event_signal_map *map = &base->sigmap;
	struct evmap_signal *ctx;
	struct event *ev;

	if (sig < 0 || sig >= map->nentries)
		return;
	// 从base->sigmap找出信号对应的回调事件
	GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);

	if (!ctx)
		return;
	LIST_FOREACH(ev, &ctx->events, ev_signal_next)
		event_active_nolock_(ev, EV_SIGNAL, ncalls);
}
```

`event_active_nolock_`详见1.36.3.2

###1.33.evsig_add——信号事件注册函数

位于signal.c，当执行`evsigsel->add`的时候，其实是调用的这个函数。

当注册信号事件时，会调用`evsig_add`函数，主要是将信号事件注册到`event_base`上，同时将信号和捕捉信号的函数通过signal系统函数绑定到一起，一旦捕捉信号的函数捕捉到信号，则会将信号写入内部信号通知管道，然后会触发内部信号通知事件，内部信号通知事件会从内部信号通知管道读取信号，
然后根据信号值，触发相应外部信号事件回调函数。

代码如下：

```cpp
// 参数介绍：
// base：绑定到哪个event_base上
// evsignal：需要绑定的信号
// old:该信号上老的事件类型
// events：该信号上当前的事件类型
// p：类似于prepare函数，在转化为内部事件之前需要触发的函数
static int
evsig_add(struct event_base *base, evutil_socket_t evsignal, short old, short events, void *p)
{
	// 这里实际上是内部信号通知事件，用于将外部信号转换为内部事件的具体实现
	struct evsig_info *sig = &base->sig;
	(void)p;

	EVUTIL_ASSERT(evsignal >= 0 && evsignal < NSIG);

	/* catch signals if they happen quickly */
	EVSIGBASE_LOCK();
	if (evsig_base != base && evsig_base_n_signals_added) {
		event_warnx("Added a signal to event base %p with signals "
		    "already added to event_base %p.  Only one can have "
		    "signals at a time with the %s backend.  The base with "
		    "the most recently added signal or the most recent "
		    "event_base_loop() call gets preference; do "
		    "not rely on this behavior in future Libevent versions.",
		    base, evsig_base, base->evsel->name);
	}

	// 以下是三个全局变量，都是讲关于信号事件的。
	// evsig_base是event_base句柄
	// evsig_base_n_signals_added是当前event_base中增加的信号事件的总数
	// evsig_base_fd是信号事件的文件描述符，其实就是上文中提到的base->sig.ev_signal_pair[1]，即用来写入信号的内部管道
	evsig_base = base;
	evsig_base_n_signals_added = ++sig->ev_n_signals_added;
	evsig_base_fd = base->sig.ev_signal_pair[1];
	EVSIGBASE_UNLOCK();

	event_debug(("%s: %d: changing signal handler", __func__, (int)evsignal));

	// 将信号注册到信号捕捉函数evsig_handler上
	// evsig_handler会捕捉信号，然后将信号写入base->sig.ev_signal_pair[1]，用来通知内部管道，进而通知event_base处理
	if (evsig_set_handler_(base, (int)evsignal, evsig_handler) == -1) {
		goto err;
	}

	// 时间参数为NULL，表明这不是一个超时时间，只有当注册事件描述符有读写行为时，更准确的说，只有当内部信号通知管道发生写事件时，才会触发回调函数去读信号，进而去激活外部信号注册的回调函数。这是具体的事件添加，需要查看事件注册相关文档。这里实际上是在添加外部信号事件时，会添加内部信号通知事件，目的是为了将外部信号事件借助内部管道将捕捉到的外部信号转换为内部IO事件。如果event_base上还没有注册内部通知事件，则需要注册；如果已经注册过了，就不需要重复注册了。这就说明，只有当注册外部信号事件，才会触发注册内部信号通知事件，否则内部通知事件只申请而不会使用。
	if (!sig->ev_signal_added) {
		if (event_add_nolock_(&sig->ev_signal, NULL, 0))
			goto err;
		sig->ev_signal_added = 1;
	}

	return (0);

err:
	EVSIGBASE_LOCK();
	--evsig_base_n_signals_added;
	--sig->ev_n_signals_added;
	EVSIGBASE_UNLOCK();
	return (-1);
}
```


###1.34.evsig_set_handler_——设置外部信号的捕捉处理句柄

位于signal.c，代码如下：

```cpp
/* Helper: set the signal handler for evsignal to handler in base, so that
 * we can restore the original handler when we clear the current one. */
// 在event_base中为信号设置信号处理句柄，这样当清除当前信号时，就可以重新存储一般的信号句柄处理
int
evsig_set_handler_(struct event_base *base,
    int evsignal, void (__cdecl *handler)(int))
{
#ifdef EVENT__HAVE_SIGACTION
	struct sigaction sa;
#else
	ev_sighandler_t sh;
#endif
	struct evsig_info *sig = &base->sig;
	void *p;

	/*
	 * resize saved signal handler array up to the highest signal number.
	 * a dynamic array is used to keep footprint on the low side.
	 */
	// 重新调整信号句柄数组大小，以适应最大的信号值。
	// 动态数组用来适应当前最大的信号值
	// 一旦当前信号值大于原有最大信号值，则需要重新申请空间
	if (evsignal >= sig->sh_old_max) {
		int new_max = evsignal + 1;
		event_debug(("%s: evsignal (%d) >= sh_old_max (%d), resizing",
			    __func__, evsignal, sig->sh_old_max));
		p = mm_realloc(sig->sh_old, new_max * sizeof(*sig->sh_old));
		if (p == NULL) {
			event_warn("realloc");
			return (-1);
		}

		// 为新增的信号分配空间
		memset((char *)p + sig->sh_old_max * sizeof(*sig->sh_old),
		    0, (new_max - sig->sh_old_max) * sizeof(*sig->sh_old));

		sig->sh_old_max = new_max;
		sig->sh_old = p;
	}

	/* allocate space for previous handler out of dynamic array */
	// 保存以前的信号句柄，并配置新的信号句柄
	// 将新增信号和信号处理函数handler绑定到一起
	sig->sh_old[evsignal] = mm_malloc(sizeof *sig->sh_old[evsignal]);
	if (sig->sh_old[evsignal] == NULL) {
		event_warn("malloc");
		return (-1);
	}

	/* save previous handler and setup new handler */
#ifdef EVENT__HAVE_SIGACTION
	memset(&sa, 0, sizeof(sa));
	sa.sa_handler = handler;
	sa.sa_flags |= SA_RESTART;
	sigfillset(&sa.sa_mask);

	// signal()是标准C的信号接口(如果程序必须在非POSIX系统上运行，那么就应该使用这个接口)。给信号signum设置新的信号处理函数act， 同时保留该信号原有的信号处理函数oldact
	if (sigaction(evsignal, &sa, sig->sh_old[evsignal]) == -1) {
		event_warn("sigaction");
		mm_free(sig->sh_old[evsignal]);
		sig->sh_old[evsignal] = NULL;
		return (-1);
	}
#else
	if ((sh = signal(evsignal, handler)) == SIG_ERR) {
		event_warn("signal");
		mm_free(sig->sh_old[evsignal]);
		sig->sh_old[evsignal] = NULL;
		return (-1);
	}
	*sig->sh_old[evsignal] = sh;
#endif

	return (0);
}
```


###1.35.evsig_handler函数

这是外部信号发生时的处理函数，主要用于将信号值写入`base->sig.ev_signal_pair[1]`，以达到通知外部信号发生的目的。

位于signal.c，代码如下：

```cpp
// __cdecl 是C Declaration的缩写（declaration，声明），表示C语言默认的函数调用方法：所有参数从右到左依次入栈，这些参数由调用者清除，称为手动清栈。被调用函数不会要求调用者传递多少参数，调用者传递过多或者过少的参数，甚至完全不同的参数都不会产生编译阶段的错误。
static void __cdecl
evsig_handler(int sig)
{
	int save_errno = errno;
#ifdef _WIN32
	int socket_errno = EVUTIL_SOCKET_ERROR();
#endif
	ev_uint8_t msg;

	if (evsig_base == NULL) {
		event_warnx(
			"%s: received signal %d, but have no base configured",
			__func__, sig);
		return;
	}

#ifndef EVENT__HAVE_SIGACTION
	signal(sig, evsig_handler);
#endif

	/* Wake up our notification mechanism */
	// 唤醒通知机制
    // 使用写入内部管道的消息保存信号值
	msg = sig;
#ifdef _WIN32
	send(evsig_base_fd, (char*)&msg, 1, 0);
#else
	{
		// 将消息写入内部管道文件描述符，注意是一个字节，因为信号使用一个字节就可以表示了
		// evsig_base_fd在前面就已经设置了
		int r = write(evsig_base_fd, (char*)&msg, 1);
		(void)r; /* Suppress 'unused return value' and 'unused var' */
	}
#endif
	errno = save_errno;
#ifdef _WIN32
	EVUTIL_SET_SOCKET_ERROR(socket_errno);
#endif
}
```

###1.36.event_base_dispatch——事件调度函数

位于event.c，代码如下：

```cpp
int
event_base_dispatch(struct event_base *event_base)
{
	return (event_base_loop(event_base, 0));
}
```

我们可以看到，内部调用了`event_base_loop`函数。

`event_base_loop`此函数主要是运行激活事件，它会根据配置中的参数来确定是否需要在执行激活事件过程中中断执行并检查新事件以及检查频率；同时也会根据事件类型执行不同的回调函数，并且决定是否将事件重新添加到队列中。

`event_base_loop`位于event.c，代码如下：

```cpp
// event_base_loop(event_base, 0);
// 等待事件变得活跃，然后运行事件回调函数
// 相比event_base_dispatch函数，这个函数更为灵活。默认情况下，loop会一直运行到没有等待事件或者要激活的事件，或者运行到调用event_base_loopbreak或者event_base_loopexit函数。
// 可以使用flags参数来调整loop的行为。
// 函数参数：
// base：event_base_new或者event_base_new_with_config产生的event_base结构体
// flags:可以是EVLOOP_ONCE | EVLOOP_NONBLOCK
// 返回值：成功则为0，失败则为－1，如果因为没有等待的事件或者激活事件而退出则返回1
// 相关查看event_base_loopexit，event_base_dispatch
int
event_base_loop(struct event_base *base, int flags)
{
	const struct eventop *evsel = base->evsel;
	struct timeval tv;
	struct timeval *tv_p;
	int res, done, retval = 0;

	/* Grab the lock.  We will release it inside evsel.dispatch, and again
	 * as we invoke user callbacks. */
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

	// 一个event_base同一时间只能运行一次event_base_loop，保证event_base_loop是一个单例函数
	if (base->running_loop) {
		event_warnx("%s: reentrant invocation.  Only one event_base_loop"
		    " can run on each event_base at once.", __func__);
		EVBASE_RELEASE_LOCK(base, th_base_lock);
		return -1;
	}

	// 启动running_loop
	base->running_loop = 1;

	// 清空当前event_base中的时间，防止误用。
	/*
	// Make 'base' have no current cached time.
	static inline void
	clear_time_cache(struct event_base *base)
	{
		base->tv_cache.tv_sec = 0;
	}
	*/
	clear_time_cache(base);

	// 如果event_base中有信号事件，则需要设置
	if (base->sig.ev_signal_added && base->sig.ev_n_signals_added)
	{
		/*
		void evsig_set_base_(struct event_base *base)
		{
			EVSIGBASE_LOCK();
			// evsig_base，evsig_base_n_signals_added，evsig_base_fd是维护信号事件的三个全局变量。
			// evsig_base是event_base句柄
			// evsig_base_n_signals_added是当前event_base中增加的信号事件的总数
			// evsig_base_fd是信号事件的文件描述符，其实就是上文中提到的base->sig.ev_signal_pair[1]，即用来写入信号的内部管道
			evsig_base = base;
			evsig_base_n_signals_added = base->sig.ev_n_signals_added;
			evsig_base_fd = base->sig.ev_signal_pair[1];
			EVSIGBASE_UNLOCK();
		}
		*/
		evsig_set_base_(base);
	}

	// 初始化done
	done = 0;

#ifndef EVENT__DISABLE_THREAD_SUPPORT
	base->th_owner_id = EVTHREAD_GET_ID();
#endif
	// 初始化终止或者退出的标志位
	base->event_gotterm = base->event_break = 0;

	// 死循环
	while (!done) {
		base->event_continue = 0;
		base->n_deferreds_queued = 0;

		/* Terminate the loop if we have been asked to */
		// 每次loop时，需要判定是否别的地方已经设置了终止或者退出的标志位
		if (base->event_gotterm) {
			break;
		}

		if (base->event_break) {
			break;
		}

		tv_p = &tv;
		// 如果event_base的活跃事件数量为空并且是非阻塞模式，则获取下一个超时事件的距离超时的时间间隔
		// 用于后台方法调度超时事件，否则清空存储距离超时的时间间隔。

		// #define N_ACTIVE_CALLBACKS(base)	 ((base)->event_count_active)
		if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) {
	
			// 用于获取超时时间间隔，用以下一次dispatch的超时
			timeout_next(base, &tv_p);
		} else {
			/*
			 * if we have active events, we just poll new events
			 * without waiting.
			 */
			// 清空tv结构体
			evutil_timerclear(&tv);
		}

		/* If we have no events, we just exit */
		// 如果没有事件并且模式是EVLOOP_NO_EXIT_ON_EMPTY，则退出。
		if (0==(flags&EVLOOP_NO_EXIT_ON_EMPTY) &&
		    !event_haveevents(base) && !N_ACTIVE_CALLBACKS(base)) {
			event_debug(("%s: no events registered.", __func__));
			retval = 1;
			goto done;
		}

		// 将later事件激活放入到活跃队列
		// 下面有讲，详见1.36.2
		event_queue_make_later_events_active(base);

		// 清空event_base中的时间，防止误用
		clear_time_cache(base);

		// 调用后台方法的dispatch方法
		// 对于linux系统，这里调用的是epoll_dispatch方法
		// 详见1.36.3
		res = evsel->dispatch(base, tv_p);

		if (res == -1) {
			event_debug(("%s: dispatch returned unsuccessfully.",
				__func__));
			retval = -1;
			goto done;
		}

		// 更新当前event_base中缓存的时间，因为这是在执行后台
		update_time_cache(base);

		// 从超时事件最小堆中取出超时事件，并将超时事件放入激活队列。
		// 详见1.36.4
		timeout_process(base);

		// 如果激活队列不为空，则处理激活的事件
		// 否则，如果模式为非阻塞，则退出loop
		if (N_ACTIVE_CALLBACKS(base)) {
			int n = event_process_active(base);
			if ((flags & EVLOOP_ONCE)
			    && N_ACTIVE_CALLBACKS(base) == 0
			    && n != 0)
				done = 1;
		} else if (flags & EVLOOP_NONBLOCK)
			done = 1;
	}
	event_debug(("%s: asked to terminate loop.", __func__));

done:
	clear_time_cache(base);
	base->running_loop = 0;

	EVBASE_RELEASE_LOCK(base, th_base_lock);

	return (retval);
}
```


####1.36.1.timeout_next函数

位于event.c，主要是获取下一个超时事件超过超时时间的时间间隔；如果没有超时事件，则存储超时时间间隔则为空。代码如下：

```cpp
// timeout_next(base, &tv_p);
static int
timeout_next(struct event_base *base, struct timeval **tv_p)
{
	/* Caller must hold th_base_lock */
	struct timeval now;
	struct event *ev;
	struct timeval *tv = *tv_p;
	int res = 0;

	// 获取超时事件最小堆根部，即紧接着下来的下一个超时事件
	ev = min_heap_top_(&base->timeheap);

	if (ev == NULL) {
		/* if no time-based events are active wait for I/O */
		*tv_p = NULL;
		goto out;
	}

	// 获取base中现在的时间，如果base中时间为空，则获取现在系统中的时间
	// 关于gettime，详见1.2.3
	if (gettime(base, &now) == -1) {
		res = -1;
		goto out;
	}

	// 比较事件超时时间和当前base中的时间，如果超时时间<=当前时间，则表明已经超时，则返回即可；
	// 如果超时时间>=当前时间，表明超时时间还没到，将超时时间减去当前时间的结果存储在tv中，即超过超时时间的时间间隔
	if (evutil_timercmp(&ev->ev_timeout, &now, <=)) {
		evutil_timerclear(tv);
		goto out;
	}

	evutil_timersub(&ev->ev_timeout, &now, tv);

	EVUTIL_ASSERT(tv->tv_sec >= 0);
	EVUTIL_ASSERT(tv->tv_usec >= 0);
	event_debug(("timeout_next: event: %p, in %d seconds, %d useconds", ev, (int)tv->tv_sec, (int)tv->tv_usec));

out:
	return (res);
}
```


####1.36.2.event_queue_make_later_events_active函数

位于event.c，主要是将下一次激活事件队列中的事件都移动到激活队列中。代码如下：

```cpp
// event_queue_make_later_events_active(base);
static void
event_queue_make_later_events_active(struct event_base *base)
{
	struct event_callback *evcb;
	EVENT_BASE_ASSERT_LOCKED(base);

	// 判断下一次激活队列中是否存在回调事件，如果存在，则将回调事件状态增加激活状态
	// 然后将该回调事件插入对应优先级的激活队列中，并将推迟的事件总数加一。
	while ((evcb = TAILQ_FIRST(&base->active_later_queue))) {
		TAILQ_REMOVE(&base->active_later_queue, evcb, evcb_active_next);  // 把evcb从base->active_later_queue中去掉。地址记在evcb_active_next中
		evcb->evcb_flags = (evcb->evcb_flags & ~EVLIST_ACTIVE_LATER) | EVLIST_ACTIVE;  // 去掉EVLIST_ACTIVE_LATER属性，加上EVLIST_ACTIVE属性
		EVUTIL_ASSERT(evcb->evcb_pri < base->nactivequeues);
		// 在对应的优先级队列里面插入该evcb
		TAILQ_INSERT_TAIL(&base->activequeues[evcb->evcb_pri], evcb, evcb_active_next);
		base->n_deferreds_queued += (evcb->evcb_closure == EV_CLOSURE_CB_SELF);
	}
}
```

其中，`event_callback`的结构为：

```cpp
struct event_callback {
	TAILQ_ENTRY(event_callback) evcb_active_next;
	short evcb_flags;
	ev_uint8_t evcb_pri;	/* smaller numbers are higher priority */
	ev_uint8_t evcb_closure;
	/* allows us to adopt for different types of events */
        union {
		void (*evcb_callback)(evutil_socket_t, short, void *);
		void (*evcb_selfcb)(struct event_callback *, void *);
		void (*evcb_evfinalize)(struct event *, void *);
		void (*evcb_cbfinalize)(struct event_callback *, void *);
	} evcb_cb_union;
	void *evcb_arg;
};
```

####1.36.3.epoll_dispatch函数

位于epoll.c，epoll的调度方法，代码如下：

```cpp
// res = evsel->dispatch(base, tv_p);
static int
epoll_dispatch(struct event_base *base, struct timeval *tv)
{
	struct epollop *epollop = base->evbase;
	struct epoll_event *events = epollop->events;
	int i, res;
	long timeout = -1;

#ifdef USING_TIMERFD
	if (epollop->timerfd >= 0) {
		struct itimerspec is;
		is.it_interval.tv_sec = 0;
		is.it_interval.tv_nsec = 0;
		if (tv == NULL) {
			// 当前没有超时事件
			/* No timeout; disarm the timer. */
			is.it_value.tv_sec = 0;
			is.it_value.tv_nsec = 0;
		} else {
			if (tv->tv_sec == 0 && tv->tv_usec == 0) {
				/* we need to exit immediately; timerfd can't
				 * do that. */
				timeout = 0;
			}
			is.it_value.tv_sec = tv->tv_sec;
			is.it_value.tv_nsec = tv->tv_usec * 1000;
		}
		/* TODO: we could avoid unnecessary syscalls here by only
		   calling timerfd_settime when the top timeout changes, or
		   when we're called with a different timeval.
		*/
		if (timerfd_settime(epollop->timerfd, 0, &is, NULL) < 0) {
			event_warn("timerfd_settime");
		}
	} else
#endif
	// 如果存在超时事件，则将超时时间转换为毫秒时间
	// 如果超时时间在合法范围之外，则设置超时时间为永久等待（代表异常了）
	if (tv != NULL) {
		timeout = evutil_tv_to_msec_(tv);
		if (timeout < 0 || timeout > MAX_EPOLL_TIMEOUT_MSEC) {
			/* Linux kernels can wait forever if the timeout is
			 * too big; see comment on MAX_EPOLL_TIMEOUT_MSEC. */
			timeout = MAX_EPOLL_TIMEOUT_MSEC;
		}
	}

	// 将event_base中有改变的事件列表都根据事件类型应用到epoll的监听上；重新设置之后，则删除改变
	epoll_apply_changes(base);
	event_changelist_remove_all_(&base->changelist, base);

	EVBASE_RELEASE_LOCK(base, th_base_lock);

	res = epoll_wait(epollop->epfd, events, epollop->nevents, timeout);

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

	if (res == -1) {
		if (errno != EINTR) {
			event_warn("epoll_wait");
			return (-1);
		}

		return (0);
	}

	event_debug(("%s: epoll_wait reports %d", __func__, res));
	EVUTIL_ASSERT(res <= epollop->nevents);

	for (i = 0; i < res; i++) {
		int what = events[i].events;
		short ev = 0;
#ifdef USING_TIMERFD
		if (events[i].data.fd == epollop->timerfd)
			continue;
#endif

		if (what & (EPOLLHUP|EPOLLERR)) {
			ev = EV_READ | EV_WRITE;
		} else {
			if (what & EPOLLIN)
				ev |= EV_READ;
			if (what & EPOLLOUT)
				ev |= EV_WRITE;
			if (what & EPOLLRDHUP)
				ev |= EV_CLOSED;
		}

		if (!ev)
			continue;

		evmap_io_active_(base, events[i].data.fd, ev | EV_ET);
	}

	if (res == epollop->nevents && epollop->nevents < MAX_NEVENT) {
		/* We used all of the event space this time.  We should
		   be ready for more events next time. */
		int new_nevents = epollop->nevents * 2;
		struct epoll_event *new_events;

		new_events = mm_realloc(epollop->events,
		    new_nevents * sizeof(struct epoll_event));
		if (new_events) {
			epollop->events = new_events;
			epollop->nevents = new_nevents;
		}
	}

	return (0);
}
```


#####1.36.3.1.epoll_apply_changes函数

位于epoll.c，代码如下：

```cpp
// 将event_base中有改变的事件列表都根据事件类型应用到epoll的监听上
static int
epoll_apply_changes(struct event_base *base)
{
	struct event_changelist *changelist = &base->changelist;
	struct epollop *epollop = base->evbase;  // epoll句柄
	struct event_change *ch;

	int r = 0;
	int i;

	for (i = 0; i < changelist->n_changes; ++i) {
		ch = &changelist->changes[i];
		if (epoll_apply_one_change(base, epollop, ch) < 0)
			r = -1;
	}

	return (r);
}
```

`struct event_changelist`的结构位于event-internal.h，如下：

```cpp
/* List of 'changes' since the last call to eventop.dispatch.  Only maintained
 * if the backend is using changesets. */
struct event_changelist {
	struct event_change *changes;
	int n_changes;
	int changes_size;
};
```

`struct event_change`的结构位于changelist-internal.h，如下

```cpp
/** Represents a */
struct event_change {
	/** The fd or signal whose events are to be changed */
	evutil_socket_t fd;
	/* The events that were enabled on the fd before any of these changes
	   were made.  May include EV_READ or EV_WRITE. */
	short old_events;

	/* The changes that we want to make in reading and writing on this fd.
	 * If this is a signal, then read_change has EV_CHANGE_SIGNAL set,
	 * and write_change is unused. */
	ev_uint8_t read_change;
	ev_uint8_t write_change;
	ev_uint8_t close_change;
};
```

`epoll_apply_one_change`位于epoll.c，代码如下：

```cpp
// 将event_base中有改变的事件列表都根据事件类型应用到epoll的监听上
// epoll_apply_one_change(base, epollop, ch)
static int
epoll_apply_one_change(struct event_base *base,
    struct epollop *epollop,
    const struct event_change *ch)
{
	struct epoll_event epev;
	int op, events = 0;
	int idx;

	// 根据事件的特点找出对应的epolltable下标
	// epoll_op_table的详细定义位于epolltable-internal.h
	idx = EPOLL_OP_TABLE_INDEX(ch);
	op = epoll_op_table[idx].op;
	events = epoll_op_table[idx].events;

	if (!events) {
		EVUTIL_ASSERT(op == 0);
		return 0;
	}

	// 读写事件均为边缘触发
	if ((ch->read_change|ch->write_change) & EV_CHANGE_ET)
		events |= EPOLLET;  // epoll设置为边缘触模式

	memset(&epev, 0, sizeof(epev));
	epev.data.fd = ch->fd;
	epev.events = events;
	// 第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：EPOLL_CTL_ADD：注册新的fd到epfd中；EPOLL_CTL_MOD：修改已经注册的fd的监听事件；EPOLL_CTL_DEL：从epfd中删除一个fd；第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，
	// struct epoll_event结构如下：
	// typedef union epoll_data {    
	//	void *ptr;    
	//	int fd;    
	//	__uint32_t u32;    
	//	__uint64_t u64;
	//	} epoll_data_t;
	// struct epoll_event {    
	//	__uint32_t events; // Epoll events    
	//	epoll_data_t data; // User data variable 
	//	};
	// events可以是以下几个宏的集合：
	// EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
	// EPOLLOUT：表示对应的文件描述符可以写；
	// EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
	// EPOLLERR：表示对应的文件描述符发生错误；
	// EPOLLHUP：表示对应的文件描述符被挂断；
	// EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
	// EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
	if (epoll_ctl(epollop->epfd, op, ch->fd, &epev) == 0) {
		event_debug((PRINT_CHANGES(op, epev.events, ch, "okay")));
		return 0;
	}

	// op是根据变化的事件ch在里面得出需要进行的epoll动作op
	switch (op) {
	case EPOLL_CTL_MOD:  // 对fd描述符的event事件进行修改
		if (errno == ENOENT) {
			/* If a MOD operation fails with ENOENT, the
			 * fd was probably closed and re-opened.  We
			 * should retry the operation as an ADD.
			 */
			// 如果遇到ENOENT这个错误，说明fd有可能已经关闭了和重新打开了，这个时候应该产生使用ADD操作
			if (epoll_ctl(epollop->epfd, EPOLL_CTL_ADD, ch->fd, &epev) == -1) {
				event_warn("Epoll MOD(%d) on %d retried as ADD; that failed too",
				    (int)epev.events, ch->fd);
				return -1;
			} else {
				event_debug(("Epoll MOD(%d) on %d retried as ADD; succeeded.",
					(int)epev.events,
					ch->fd));
				return 0;
			}
		}
		break;
	case EPOLL_CTL_ADD:  // 对fd描述符进行注册event事件操作
		if (errno == EEXIST) {
			/* If an ADD operation fails with EEXIST,
			 * either the operation was redundant (as with a
			 * precautionary add), or we ran into a fun
			 * kernel bug where using dup*() to duplicate the
			 * same file into the same fd gives you the same epitem
			 * rather than a fresh one.  For the second case,
			 * we must retry with MOD. */
			// EEXIST，说明已经注册了event操作，这个时候需要进行修改MOD操作
			if (epoll_ctl(epollop->epfd, EPOLL_CTL_MOD, ch->fd, &epev) == -1) {
				event_warn("Epoll ADD(%d) on %d retried as MOD; that failed too",
				    (int)epev.events, ch->fd);
				return -1;
			} else {
				event_debug(("Epoll ADD(%d) on %d retried as MOD; succeeded.",
					(int)epev.events,
					ch->fd));
				return 0;
			}
		}
		break;
	case EPOLL_CTL_DEL:  // 删除已经注册的event事件
		if (errno == ENOENT || errno == EBADF || errno == EPERM) {
			/* If a delete fails with one of these errors,
			 * that's fine too: we closed the fd before we
			 * got around to calling epoll_dispatch. */
			// 遇到这些错误码，有可能是我们之前已经提早关闭fd了，这个时候不报错返回
			event_debug(("Epoll DEL(%d) on fd %d gave %s: DEL was unnecessary.",
				(int)epev.events,
				ch->fd,
				strerror(errno)));
			return 0;
		}
		break;
	default:
		break;
	}

	event_warn(PRINT_CHANGES(op, epev.events, ch, "failed"));
	return -1;
}
```
