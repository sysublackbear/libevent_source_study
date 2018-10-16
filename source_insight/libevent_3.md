# libevent(3)
@(源码)


### 1.7.事件类型的各个标志位含义

位于event.h，我们重点记住几个枚举值的含义：

```cpp
/**
 * @name event flags
 *
 * Flags to pass to event_new(), event_assign(), event_pending(), and
 * anything else with an argument of the form "short events"
 */
/**@{*/
/** Indicates that a timeout has occurred.  It's not necessary to pass
 * this flag to event_for new()/event_assign() to get a timeout. */

// 定时事件
#define EV_TIMEOUT	0x01

/** Wait for a socket or FD to become readable */
// 读事件
#define EV_READ		0x02

/** Wait for a socket or FD to become writeable */
// 写事件
#define EV_WRITE	0x04

/** Wait for a POSIX signal to be raised*/
// 信号
#define EV_SIGNAL	0x08


/**
 * Persistent event: won't get removed automatically when activated.
 *
 * When a persistent event with a timeout becomes activated, its timeout
 * is reset to 0.
 */
// 永久事件，激活执行后会重新加到队列中等待下一次激活，否则激活执行后会自动移除
#define EV_PERSIST	0x10

/** Select edge-triggered behavior, if supported by the backend. */
// 边沿触发
#define EV_ET		0x20

/**
 * If this option is provided, then event_del() will not block in one thread
 * while waiting for the event callback to complete in another thread.
 *
 * To use this option safely, you may need to use event_finalize() or
 * event_free_finalize() in order to safely tear down an event in a
 * multithreaded application.  See those functions for more information.
 *
 * THIS IS AN EXPERIMENTAL API. IT MIGHT CHANGE BEFORE THE LIBEVENT 2.1 SERIES
 * BECOMES STABLE.
 **/
// 终止事件，如果设置了这个选项，则event_del不会阻塞，需要使用event_finalize或者event_free_finalize以保证多线程安全
#define EV_FINALIZE     0x40

/**
 * Detects connection close events.  You can use this to detect when a
 * connection has been closed, without having to read all the pending data
 * from a connection.
 *
 * Not all backends support EV_CLOSED.  To detect or require it, use the
 * feature flag EV_FEATURE_EARLY_CLOSE.
 **/
// 检查事件连接是否关闭；可以使用这个选项来检测链接是否关闭，而不需要读取此链接所有未决数据
#define EV_CLOSED	0x80
/**@}*/
```


### 1.8.回调事件的状态标识

位于event_struct.h，代码如下：

```cpp
// 事件在time_min_heap堆中
#define EVLIST_TIMEOUT	    0x01
// 事件在已注册事件链表中
#define EVLIST_INSERTED	    0x02
// 事件在signal链表中
#define EVLIST_SIGNAL	    0x04
// 事件在激活链表中
#define EVLIST_ACTIVE	    0x08
// 事件的内部使用标记
#define EVLIST_INTERNAL	    0x10
// 事件在下一次激活链表
#define EVLIST_ACTIVE_LATER 0x20
// 事件已经终止
#define EVLIST_FINALIZING   0x40
// 事件初始化完成
#define EVLIST_INIT	    0x80
// 包含所有事件状态，用于判断合法性
#define EVLIST_ALL          0xff
```


### 1.9.有关事件的操作函数

位于event.c，代码如下：
```cpp
/* Array of backends in order of preference. */
static const struct eventop *eventops[] = {
#ifdef EVENT__HAVE_EVENT_PORTS
	&evportops,
#endif
#ifdef EVENT__HAVE_WORKING_KQUEUE
	&kqops,
#endif
#ifdef EVENT__HAVE_EPOLL
	&epollops,
#endif
#ifdef EVENT__HAVE_DEVPOLL
	&devpollops,
#endif
#ifdef EVENT__HAVE_POLL
	&pollops,
#endif
#ifdef EVENT__HAVE_SELECT
	&selectops,
#endif
#ifdef _WIN32
	&win32ops,
#endif
	NULL
};
```

### 1.10.后台方法特征列表

位于event.h，代码如下：

```cpp
/**
   A flag used to describe which features an event_base (must) provide.

   Because of OS limitations, not every Libevent backend supports every
   possible feature.  You can use this type with
   event_config_require_features() to tell Libevent to only proceed if your
   event_base implements a given feature, and you can receive this type from
   event_base_get_features() to see which features are available.
*/
// 用来描述event_base必须提供的特征值，其实是后台方法提供的；
// 因为OS的限制，不是所有event_base后台方法都支持每个可能的特征；
// 必须使用event_config_require_features（）进行配置，同时必须使用
// event_base_get_features（）查看是否支持配置的特征
enum event_method_feature {
    /** Require an event method that allows edge-triggered events with EV_ET. */
    // 边缘触发，高效但是如果后台处置不当，容易丢消息，注意和水平触发（LT）区分
    EV_FEATURE_ET = 0x01,
    
    /** Require an event method where having one event triggered among
     * many is [approximately] an O(1) operation. This excludes (for
     * example) select and poll, which are approximately O(N) for N
     * equal to the total number of possible events. */
    // 要求具有很多事件的后台方法可以以近似O（1）处理事件；select和poll
    // 无法提供这种特征，它们只能提供近似O（N）的操作
    EV_FEATURE_O1 = 0x02,
    
    /** Require an event method that allows file descriptors as well as
     * sockets. */
    // 后台方法可以处理包括sockets在内的各种文件描述符
    EV_FEATURE_FDS = 0x04,
    /** Require an event method that allows you to use EV_CLOSED to detect
     * connection close without the necessity of reading all the pending data.
     *
     * Methods that do support EV_CLOSED may not be able to provide support on
     * all kernel versions.
     **/
    // 要求后台方法可以使用EV_CLOSED检测链接关闭，而不需要读完所有未决数据才能判断
    // 支持EV_CLOSED的后台方法不是所有OS内核都支持的
    EV_FEATURE_EARLY_CLOSE = 0x08
};
```


### 1.11.event_base的工作模式

位于event.h，代码如下：

```cpp
/**
   A flag passed to event_config_set_flag().

    These flags change the behavior of an allocated event_base.

    @see event_config_set_flag(), event_base_new_with_config(),
       event_method_feature
 */
enum event_base_config_flag {
	/** Do not allocate a lock for the event base, even if we have
	    locking set up.

	    Setting this option will make it unsafe and nonfunctional to call
	    functions on the base concurrently from multiple threads.
	*/
	// 非阻塞模式，非线程安全
	EVENT_BASE_FLAG_NOLOCK = 0x01,
	
	/** Do not check the EVENT_* environment variables when configuring
	    an event_base  */
	// 这种模式下不再检查EVENT_*环境变量
	EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
	
	/** Windows only: enable the IOCP dispatcher at startup

	    If this flag is set then bufferevent_socket_new() and
	    evconn_listener_new() will use IOCP-backed implementations
	    instead of the usual select-based one on Windows.
	 */
	// 只应用于windows环境
	EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
	
	/** Instead of checking the current time every time the event loop is
	    ready to run timeout callbacks, check after each timeout callback.
	 */
	// 每次event_loop准备运行timeout回调时，不再检查当前的时间，而是在每次timeout回调之后检查
	EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,

	/** If we are using the epoll backend, this flag says that it is
	    safe to use Libevent's internal change-list code to batch up
	    adds and deletes in order to try to do as few syscalls as
	    possible.  Setting this flag can make your code run faster, but
	    it may trigger a Linux bug: it is not safe to use this flag
	    if you have any fds cloned by dup() or its variants.  Doing so
	    will produce strange and hard-to-diagnose bugs.

	    This flag can also be activated by setting the
	    EVENT_EPOLL_USE_CHANGELIST environment variable.

	    This flag has no effect if you wind up using a backend other than
	    epoll.
	 */
	// 如果后台方法是epoll，则此模式是指可以安全的使用libevent内部changelist
	// 进行批量增删而尽可能减少系统调用。这种模式可以让代码性能更高，
	// 但是可能会引起Linux bug：如果有任何由dup（）或者他的变量克隆的fds，
	// 则是不安全的。这样做会引起奇怪并且难以检查的bugs。此模式可以通过
	// EVENT_EPOLL_USE_CHANGELIST环境变量激活。
	// 此模式只有在使用epoll时可用
	EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,

	/** Ordinarily, Libevent implements its time and timeout code using
	    the fastest monotonic timer that we have.  If this flag is set,
	    however, we use less efficient more precise timer, assuming one is
	    present.
	 */
	// 通常情况下，libevent使用最快的monotonic计时器实现自己的计时和超时控制；
	// 此模式下，会使用性能较低但是准确性更高的计时器。
	EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
```

### 1.12.事件关闭时的回调函数模式类型

位于event-internal.h，代码如下：

```cpp
/** @name Event closure codes

    Possible values for evcb_closure in struct event_callback

    @{
 */
/** A regular event. Uses the evcb_callback callback */
// 常规事件，使用evcb_callback回调
#define EV_CLOSURE_EVENT 0

/** A signal event. Uses the evcb_callback callback */
// 信号事件，使用evcb_callback回调
#define EV_CLOSURE_EVENT_SIGNAL 1

/** A persistent non-signal event. Uses the evcb_callback callback */
// 永久性非信号事件：使用evcb_callback回调
#define EV_CLOSURE_EVENT_PERSIST 2

/** A simple callback. Uses the evcb_selfcb callback. */
// 简单回调，使用evcb_selfcb回调
#define EV_CLOSURE_CB_SELF 3

/** A finalizing callback. Uses the evcb_cbfinalize callback. */
// 结束的回调，使用evcb_cbfinalize回调
#define EV_CLOSURE_CB_FINALIZE 4

/** A finalizing event. Uses the evcb_evfinalize callback. */
// 结束事件的回调，使用evcb_evfinalize回调
#define EV_CLOSURE_EVENT_FINALIZE 5

/** A finalizing event that should get freed after. Uses the evcb_evfinalize
 * callback. */
// 结束事件之后应该释放，使用evcb_evfinalize回调
#define EV_CLOSURE_EVENT_FINALIZE_FREE 6
/** @} */
```


### 1.13.event_callback的数据结构

位于event_struct.h，代码如下：
```cpp
struct event_callback {
	// 下一个回调事件
	TAILQ_ENTRY(event_callback) evcb_active_next;
	// 回调事件的状态标识，详见1.8
	short evcb_flags;

	// 回调函数的优先级，越小优先级越高
	ev_uint8_t evcb_pri;	/* smaller numbers are higher priority */

	// 执行不同的回调函数
	ev_uint8_t evcb_closure;
	
	/* allows us to adopt for different types of events */
	// 允许我们自动适配不同类型的回调事件
        union {
		void (*evcb_callback)(evutil_socket_t, short, void *);
		void (*evcb_selfcb)(struct event_callback *, void *);
		void (*evcb_evfinalize)(struct event *, void *);
		void (*evcb_cbfinalize)(struct event_callback *, void *);
	} evcb_cb_union;

	// 回调参数
	void *evcb_arg;
};
```


### 1.14.event结构体

位于event_struct.h，代码如下：

```cpp
struct event {
	// event的回调函数，被event_base调用
	struct event_callback ev_evcallback;

	/* for managing timeouts */
	// 用来管理超时事件
	union {
		// 公用超时队列
		TAILQ_ENTRY(event) ev_next_with_common_timeout;
		// 最小堆索引
		int min_heap_idx;
	} ev_timeout_pos;

	// 如果是I/O事件，ev_fd为文件描述符；如果是信号，ev_fd为信号
	evutil_socket_t ev_fd;

	// 事件类型，详见1.7
	// 事件类型，它可以是以下三种类型，其中io事件和信号无法同时成立
	// io事件：EV_READ，EV_WRITE
	// 定时事件：EV_TIMEOUT
	// 信号：EV_SIGNAL
	// 以下是辅助选项，可以和任何事件同时成立
	// EV_PERSIST：表明是一个永久事件，表示执行完毕不会移除，如不加，则执行完毕之后会自动移除
	// EV_ET：边沿触发，如果后台方法可用的话，就可以使用，注意区分水平触发
	// EV_FINALIZE：删除事件就不会阻塞了，不会等到回调函数执行完毕，为了在多线程中安全使用，需要使用event_finalize()或者event_free_finalize()
	// EV_CLOSED：可以自动监测关闭的连接，然后放弃读取未完的数据，但是不是所有后台方法都支持这个选项
	short ev_events;
	// 记录当前激活事件的类型
	short ev_res;		/* result passed to event callback */
	
	// libevent句柄，每个事件都会保存一份句柄
	struct event_base *ev_base;

	// 使用共用体union来表现IO事件和信号
	union {
		/* used for io events */
		struct {
			// 下一个io事件
			LIST_ENTRY (event) ev_io_next;
			// 事件超时时间（既可以是相对时间，也可以是绝对时间）
			struct timeval ev_timeout;
		} ev_io;

		/* used by signal events */
		struct {
			// 下一个信号
			LIST_ENTRY (event) ev_signal_next;
			// 信号准备激活时，调用ev_callback的次数
			short ev_ncalls;
			/* Allows deletes in callback */
			// 通常指向 ev_ncalls或者NULL
			short *ev_pncalls;
		} ev_signal;
	} ev_;

	// 保存事件的超时时间
	struct timeval ev_timeout;
};
```

### 1.15.eventop结构体

位于event_internal.h，代码如下：

```cpp
/** Structure to define the backend of a given event_base. */
struct eventop {
	/** The name of this backend. */
	// 后台方法名字，即epoll,select,poll等
	const char *name;
	
	/** Function to set up an event_base to use this backend.  It should
	 * create a new structure holding whatever information is needed to
	 * run the backend, and return it.  The returned pointer will get
	 * stored by event_init into the event_base.evbase field.  On failure,
	 * this function should return NULL. */
	// 配置libevent句柄event_base使用当前后台方法；它应该创建新的数据结构
	// 隐藏了后台方法运行所需的信息，然后返回这些信息的结构体，为了支持多种结构体，因此返回void*；
	// 返回的指针将保存在event_base.evbase中；如果失败，将返回NULL
	void *(*init)(struct event_base *);
	
	/** Enable reading/writing on a given fd or signal.  'events' will be
	 * the events that we're trying to enable: one or more of EV_READ,
	 * EV_WRITE, EV_SIGNAL, and EV_ET.  'old' will be those events that
	 * were enabled on this fd previously.  'fdinfo' will be a structure
	 * associated with the fd by the evmap; its size is defined by the
	 * fdinfo field below.  It will be set to 0 the first time the fd is
	 * added.  The function should return 0 on success and -1 on error.
	 */
	// 使给定的文件描述符或者信号变得可读或者可写。'events'将是我们尝试添加的事件类型
	// 一个或者更多的EV_READ，EV_WRITE,EV_SIGNAL,EV_ET。'old'是这些事件先前的事件类型；
	// 'fdinfo'将是fd在evmap中的辅助结构体信息，它的大小由下面的fdinfo_len给出。
	// fd第一次添加时将设置为0，成功则返回0，失败则返回-1
	int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
	
	/** As "add", except 'events' contains the events we mean to disable. */
	// 删除事件
	int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
	
	/** Function to implement the core of an event loop.  It must see which
	    added events are ready, and cause event_active to be called for each
	    active event (usually via event_io_active or such).  It should
	    return 0 on success and -1 on error.
	 */
	// event_loop实现的核心代码。它必须察觉哪些添加的事件已经准备好，然后触发每个活跃事件都被调用
	// 通常是通过event_io_active或者类似这样。成功返回0，失败则返回-1。
	int (*dispatch)(struct event_base *, struct timeval *);
	
	/** Function to clean up and free our data from the event_base. */
	// 清除event_base并释放数据
	void (*dealloc)(struct event_base *);
	
	/** Flag: set if we need to reinitialize the event base after we fork.
	 */
	// 在执行fork之后是否需要重新初始化的标识位
	int need_reinit;
	
	/** Bit-array of supported event_method_features that this backend can
	 * provide. */
	// 后台方法可以提供的特征
	// 详见1.10
	// enum event_method_feature {
	// 边沿触发
	// EV_FEATURE_ET = 0x01,
	// 要求事后台方法在调度很多事件时大约为O(1)操作，select和poll无法提供这种特征，
	// 这两种方法具有N个事件时，可以提供O(N)操作
	// EV_FEATURE_O1 = 0x02,
	
	// 后台方法可以处理各种文件描述符，而不仅仅是sockets
	// EV_FEATURE_FDS = 0x04,
	/** Require an event method that allows you to use EV_CLOSED to detect
	* connection close without the necessity of reading all the pending data.
	*
	* Methods that do support EV_CLOSED may not be able to provide support on
	* all kernel versions.
	**/
	// 要求后台方法允许使用EV_CLOSED特征检测链接是否中断，而不需要读取
	// 所有未决数据；但是不是所有内核都能提供这种特征
	// EV_FEATURE_EARLY_CLOSE = 0x08
	enum event_method_feature features;


	/** Length of the extra information we should record for each fd that
	    has one or more active events.  This information is recorded
	    as part of the evmap entry for each fd, and passed as an argument
	    to the add and del functions above.
	 */
	// 应该为每个文件描述符保留的额外信息长度，额外信息可能包括一个或者多个活跃事件。
	// 这个信息是存储在每个文件描述符的evmap中，然后通过参数传递到上面的add和del函数中。
	size_t fdinfo_len;
};
```


### 1.16.event_base结构体

位于event-internal.h，代码如下：

```cpp
struct event_base {
	/** Function pointers and other data to describe this event_base's
	 * backend. */
	// 实际使用后台方法的句柄，实际上指向的是静态全局数据变量，从静态全局变量eventops 中选择
	const struct eventop *evsel;

	/** Pointer to backend-specific data. */
	// 指向后台特定的数据，是由evsel->init返回的句柄
	// 实际上是对实际后台方法所需数据的封装，void出于兼容性考虑
	void *evbase;

	/** List of changes to tell backend about at next dispatch.  Only used
	 * by the O(1) backends. */
	// 告诉后台方法下一次调度的变化列表
	struct event_changelist changelist;

	/** Function pointers used to describe the backend that this event_base
	 * uses for signals */
	// 用于描述当前event_base用于信号的后台方法
	const struct eventop *evsigsel;
	
	/** Data to implement the common signal handler code. */
	// 用于实现公用信号句柄的代码
	struct evsig_info sig;

	/** Number of virtual events */
	// 虚拟事件的数量
	int virtual_event_count;
	
	/** Maximum number of virtual events active */
	// 虚拟事件的最大数量
	int virtual_event_count_max;
	
	/** Number of total events added to this event_base */
	// 添加到event_base上的事件总数
	int event_count;
	
	/** Maximum number of total events added to this event_base */
	int event_count_max;
	
	/** Number of total events active in this event_base */
	// 当前event_base中活跃事件的个数
	int event_count_active;
	
	/** Maximum number of total events active in this event_base */
	// 当前event_base中活跃事件的最大个数
	int event_count_active_max;

	/** Set if we should terminate the loop once we're done processing
	 * events. */
	// 一旦我们完成处理事件了，如果我们应该终止loop，可以设置这个
	int event_gotterm;
	
	/** Set if we should terminate the loop immediately */
	// 如果需要中止loop，可以设置这个变量
	int event_break;
	
	/** Set if we should start a new instance of the loop immediately. */
	// 如果启动新实例的loop，可以设置这个
	int event_continue;

	/** The currently running priority of events */
	// 当前运行事件的优先级
	int event_running_priority;

	/** Set if we're running the event_base_loop function, to prevent
	 * reentrant invocation. */
	// 防止event_base_loop重入的
	int running_loop;

	/** Set to the number of deferred_cbs we've made 'active' in the
	 * loop.  This is a hack to prevent starvation; it would be smarter
	 * to just use event_config_set_max_dispatch_interval's max_callbacks
	 * feature */
	// 设置已经在loop中设置为'active'的deferred_cbs的个数，这是为了避免饥饿的hack方法；
	// 只需要使用event_config_set_max_dispatch_intervals的max_callbacks特性就可以变得更智能。
	int n_deferreds_queued;

	/* Active event management. */
	/** An array of nactivequeues queues for active event_callbacks (ones
	 * that have triggered, and whose callbacks need to be called).  Low
	 * priority numbers are more important, and stall higher ones.
	 */
	// 存储激活事件的event_callbacks的队列，这些event_callbacks都需要调用
	// 数字越小优先级越高
	struct evcallback_list *activequeues;
	
	/** The length of the activequeues array */
	// 活跃队列的长度
	int nactivequeues;
	
	/** A list of event_callbacks that should become active the next time
	 * we process events, but not this time. */
	// 下一次会变成激活状态的回调函数的列表，但是当前这次不会调用
	struct evcallback_list active_later_queue;

	/* common timeout logic */

	/** An array of common_timeout_list* for all of the common timeout
	 * values we know. */
	// 公用超时事件列表，这是二级指针，每个元素都是具有同样超时时间的事件列表
	struct common_timeout_list **common_timeout_queues;
	
	/** The number of entries used in common_timeout_queues */
	// 公用超时队列中的事件个数
	int n_common_timeouts;
	
	/** The total size of common_timeout_queues. */
	// 公用超时队列的总个数
	int n_common_timeouts_allocated;

	/** Mapping from file descriptors to enabled (added) events */
	// 文件描述符和事件之间的映射表
	struct event_io_map io;

	/** Mapping from signal numbers to enabled (added) events. */
	// 信号数字和事件之间的映射表
	struct event_signal_map sigmap;

	/** Priority queue of events with timeouts. */
	// 事件超时的优先级队列，使用最小堆实现
	struct min_heap timeheap;

	/** Stored timeval: used to avoid calling gettimeofday/clock_gettime
	 * too often. */
	// 存储时间：用来避免频繁调用gettimeofday/clock_gettime
	struct timeval tv_cache;

	// monotonic格式的时间
	struct evutil_monotonic_timer monotonic_timer;

	/** Difference between internal time (maybe from clock_gettime) and
	 * gettimeofday. */
	// 内部时间（可以从clock_gettime获取）和gettimeofday之间的差异
	struct timeval tv_clock_diff;
	
	/** Second in which we last updated tv_clock_diff, in monotonic time. */
	// 更新内部时间的间隔秒数
	time_t last_updated_clock_diff;

#ifndef EVENT__DISABLE_THREAD_SUPPORT
	/* threading support */
	/** The thread currently running the event_loop for this base */
	unsigned long th_owner_id;
	/** A lock to prevent conflicting accesses to this event_base */
	void *th_base_lock;
	/** A condition that gets signalled when we're done processing an
	 * event with waiters on it. */
	void *current_event_cond;
	/** Number of threads blocking on current_event_cond. */
	int current_event_waiters;
#endif
	/** The event whose callback is executing right now */
	// 当前执行的回调函数
	struct event_callback *current_event;

#ifdef _WIN32
	/** IOCP support structure, if IOCP is enabled. */
	struct event_iocp_port *iocp;
#endif

	/** Flags that this base was configured with */
	// event_base配置的特征值
	enum event_base_config_flag flags;

	// 最大调度时间间隔
	struct timeval max_dispatch_time;
	// 最大调度的回调函数个数
	int max_dispatch_callbacks;
	// 优先级设置之后，对于活跃队列中子队列个数的限制
	// 但是当子队列个数超过这个限制之后，会以实际的回调函数个数为准
	int limit_callbacks_after_prio;

	/* Notify main thread to wake up break, etc. */
	/** True if the base already has a pending notify, and we don't need
	 * to add any more. */
	// 如果event_base已经有关于未决事件的通知，那么我们就不需要再次添加了。
	int is_notify_pending;

	/** A socketpair used by some th_notify functions to wake up the main
	 * thread. */
	// 某些th_notify函数用户唤醒主线程的socket pair，0读1写
	evutil_socket_t th_notify_fd[2];
	/** An event used by some th_notify functions to wake up the main
	 * thread. */
	// 用于th_notify函数唤醒主线程的事件
	struct event th_notify;
	
	/** A function used to wake up the main thread from another thread. */
	// 用于从其他线程唤醒主线程的事件
	int (*th_notify_fn)(struct event_base *base);

	/** Saved seed for weak random number generator. Some backends use
	 * this to produce fairness among sockets. Protected by th_base_lock. */
	// 保存弱随机数产生器的种子。某些后台方法会使用这个种子来公平的选择
	struct evutil_weakrand_state weakrand_seed;

	/** List of event_onces that have not yet fired. */
	LIST_HEAD(once_event_list, event_once) once_events;

};
```


### 1.17.event_config_entry结构体

位于event-internal.h，代码如下：

```cpp
struct event_config_entry {
	// 下一个屏蔽的后台方法名
	TAILQ_ENTRY(event_config_entry) next;
	// 屏蔽的后台方法名
	const char *avoid_method;
};
```

### 1.18.event_config结构体

位于event-internal.h，代码如下：

```cpp
/** Internal structure: describes the configuration we want for an event_base
 * that we're about to allocate. */
// 内部结构体，用于描述对event_base进行配置的配置信息
struct event_config {
	// 屏蔽的后台方法列表
	TAILQ_HEAD(event_configq, event_config_entry) entries;

	// cpu个数，这个仅仅是对event_base的建议，不是强制的
	int n_cpus_hint;
	
	// 如果不执行以下检查，默认情况下，event_base会按照当前优先级队列的顺序，一直将本优先级
	// 队列的事件执行完毕之后才会检查新事件，这样的好处是吞吐量大，但是在低优先级队列比较长时，
	// 会导致某些高优先级一直在等待执行，无法抢占cpu
	// event_base在event_loop中两次检查新事件之间执行回调函数的时间间隔
	// 需要每次执行完回调函数之后都进行检查
	struct timeval max_dispatch_interval;

	// 需要每次执行完回调函数之后都进行检查
	int max_dispatch_callbacks;  // 执行的回调函数的个数
	
	// 用于启动上面两个检查的开关，如果＝0，则每次执行完毕回调函数之后都强制进行检查；
	// 如果＝n，则只有在执行完毕>=n的优先级事件之后才会强制执行上述检查
	int limit_callbacks_after_prio;

	// event_base后台方法需要的特征
	// 详见1.10
	enum event_method_feature require_features;

	// event_base配置的特征值
	// 详见1.11
	enum event_base_config_flag flags;
};
```
