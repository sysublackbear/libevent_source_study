#libevent(1)
@(源码)


本来是想要学习memcached的源码的，发现看了一点就发现memcached与libevent紧密相连，memcached是基于libevent库进行开发的。所以还是先学习libevent的源码。

libevent的介绍如下：
> Libevent是一个用于开发可扩展性网络服务器的基于事件驱动(event-driven)模型的网络库。Libevent有几个显著的亮点： 
> (1)事件驱动（event-driven），高性能；
> (2)轻量级，专注于网络，不如 ACE 那么臃肿庞大； 
> (3)源代码相当精炼、易读； 
> (4)跨平台，支持 Windows、Linux、*BSD和 Mac Os； 
> (5)支持多种 I/O多路复用技术， epoll、poll、dev/poll、select 和kqueue 等； 
> (6)支持 I/O，定时器和信号等事件； 
> (7)注册事件优先级； 
> Libevent 已经被广泛的应用，作为底层的网络库；比如 memcached、 Vomi t、 Nylon、 Netchat等等。

我们还是想阅读libco的方法一样，从sample目录入手，进行源码分析。

## 1.hello-world.c

位于`sample/hello-world.c`，如下：
```cpp
/*
  This example program provides a trivial server program that listens for TCP
  connections on port 9995.  When they arrive, it writes a short message to
  each client connection, and closes each connection once it is flushed.

  Where possible, it exits cleanly in response to a SIGINT (ctrl-c).
*/


#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <signal.h>
#ifndef _WIN32
#include <netinet/in.h>
# ifdef _XOPEN_SOURCE_EXTENDED
#  include <arpa/inet.h>
# endif
#include <sys/socket.h>
#endif

#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/listener.h>
#include <event2/util.h>
#include <event2/event.h>

static const char MESSAGE[] = "Hello, World!\n";

static const int PORT = 9995;

static void listener_cb(struct evconnlistener *, evutil_socket_t,
    struct sockaddr *, int socklen, void *);
static void conn_writecb(struct bufferevent *, void *);
static void conn_eventcb(struct bufferevent *, short, void *);
static void signal_cb(evutil_socket_t, short, void *);

int
main(int argc, char **argv)
{
	struct event_base *base;
	struct evconnlistener *listener;
	struct event *signal_event;

	struct sockaddr_in sin;
#ifdef _WIN32
	WSADATA wsa_data;
	WSAStartup(0x0201, &wsa_data);
#endif

	// 构造libevent句柄
	base = event_base_new();
	if (!base) {
		fprintf(stderr, "Could not initialize libevent!\n");
		return 1;
	}

	memset(&sin, 0, sizeof(sin));
	sin.sin_family = AF_INET;
	sin.sin_port = htons(PORT);   // 转换网络字节序

	// listen,bind
	listener = evconnlistener_new_bind(base, listener_cb, (void *)base,
	    LEV_OPT_REUSEABLE|LEV_OPT_CLOSE_ON_FREE, -1,
	    (struct sockaddr*)&sin,
	    sizeof(sin));

	if (!listener) {
		// 分配失败
		fprintf(stderr, "Could not create a listener!\n");
		return 1;
	}

	// 新创建一个signal事件，详见1.5
	signal_event = evsignal_new(base, SIGINT, signal_cb, (void *)base);

	if (!signal_event || event_add(signal_event, NULL)<0) {
		fprintf(stderr, "Could not create/add a signal event!\n");
		return 1;
	}

	event_base_dispatch(base);

	evconnlistener_free(listener);
	event_free(signal_event);
	event_base_free(base);

	printf("done\n");
	return 0;
}

static void
listener_cb(struct evconnlistener *listener, evutil_socket_t fd,
    struct sockaddr *sa, int socklen, void *user_data)
{
	struct event_base *base = user_data;
	struct bufferevent *bev;

	bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
	if (!bev) {
		fprintf(stderr, "Error constructing bufferevent!");
		event_base_loopbreak(base);
		return;
	}
	bufferevent_setcb(bev, NULL, conn_writecb, conn_eventcb, NULL);
	bufferevent_enable(bev, EV_WRITE);
	bufferevent_disable(bev, EV_READ);

	bufferevent_write(bev, MESSAGE, strlen(MESSAGE));
}

static void
conn_writecb(struct bufferevent *bev, void *user_data)
{
	struct evbuffer *output = bufferevent_get_output(bev);
	if (evbuffer_get_length(output) == 0) {
		printf("flushed answer\n");
		bufferevent_free(bev);
	}
}

static void
conn_eventcb(struct bufferevent *bev, short events, void *user_data)
{
	if (events & BEV_EVENT_EOF) {
		printf("Connection closed.\n");
	} else if (events & BEV_EVENT_ERROR) {
		printf("Got an error on the connection: %s\n",
		    strerror(errno));/*XXX win32*/
	}
	/* None of the other events can happen here, since we haven't enabled
	 * timeouts */
	bufferevent_free(bev);
}

static void
signal_cb(evutil_socket_t sig, short events, void *user_data)
{
	struct event_base *base = user_data;
	struct timeval delay = { 2, 0 };

	printf("Caught an interrupt signal; exiting cleanly in two seconds.\n");

	event_base_loopexit(base, &delay);
}
```


###1.1.event_base_new函数

我们可以看到一开始执行：`base = event_base_new();`。`event_base_new`位于event.c。

代码如下：
```cpp
// 创建并返回新的event_base对象——libevent句柄
// 成功则返回新的event_base对象，失败则返回NULL
// 相关查看event_base_free()，event_base_new_with_config
struct event_base *
event_base_new(void)
{
	struct event_base *base = NULL;
	// 调用event_config_new函数创建默认的struct event_config对象
	// 详见1.3
	struct event_config *cfg = event_config_new();  // 初始化event_config
	if (cfg) {  // 配置非空
		// 根据输入参数创建libevent句柄，event_base对象
		base = event_base_new_with_config(cfg);
		// 创建对象完毕后，无论成功与否，都需要释放配置信息对象struct event_config
		event_config_free(cfg);
	}
	return base;
}
```

####1.1.1.event_config_new函数

位于event.c，代码如下：
```cpp
// 分配新的event_config对象。event_config对象用来改变event_base的行为
// 返回event_config对象，里面存放着配置信息，失败则返回NULL
// 相关查看event_base_new_with_config,event_config_free,event_config结构体等
struct event_config *
event_config_new(void)
{
	// 使用内部分配api mm_calloc分配event_config对象，并赋初始值1
	蹦迪 *cfg = mm_calloc(1, sizeof(*cfg));
	// 新建event_config结构
	if (cfg == NULL)
		return (NULL);

	// 初始化屏蔽的后台方法列表
	// TAILQ_INIT:初始化队列
	/*
	//Tail queue functions.
	#define	TAILQ_INIT(head) do {						\
		(head)->tqh_first = NULL;					\
		(head)->tqh_last = &(head)->tqh_first;				\
	} while (0)
	*/
	TAILQ_INIT(&cfg->entries);
	// 设置最大调度时间间隔，初始为非法值
	cfg->max_dispatch_interval.tv_sec = -1;
	// 设置最大调度回调函数个数，初始值为INT_MAX
	cfg->max_dispatch_callbacks = INT_MAX;
	// 设置优先级后的回调函数的限制，初始值为1
	cfg->limit_callbacks_after_prio = 1;

	// 由于初始分配时赋初始值为1，经过上述显式设置之后，还有几个字段的初始值为1
	// 查看event_config定义发现，还有三个字段使用初始赋值；
	// n_cpus_hint = 1;  // 使用的cpu个数
	// require_features = 1;  // 查看后台方法特征宏定义，发现是边沿触发模式
	// flags = 1;  // 查看event_base支持的模式，发现是非阻塞模式

	return (cfg);
}
```

`event_config`的数据结构，详见1.18。


###1.2.event_base_new_with_config函数

位于event.c，代码如下：
```cpp
// 初始化libevent的API
// 使用本函数初始化新的event_base，可以使用配置对象来屏蔽某些特定事件通知机制
// cfg：事件配置对象
// 返回初始化后的event_base对象，可以用来注册事件；失败则返回NULL
struct event_base *
event_base_new_with_config(const struct event_config *cfg)
{
	int i;
	struct event_base *base;
	int should_check_environment;

#ifndef EVENT__DISABLE_DEBUG_MODE  // 测试模式
	event_debug_mode_too_late = 1;
#endif
	// mm_calloc:#define mm_calloc(n, sz) calloc((n), (sz))
	// 新建一个event_base结构
	// 使用内部分配api mm_calloc分配event_config对象，并赋初始值1
	if ((base = mm_calloc(1, sizeof(struct event_base))) == NULL) {
		event_warn("%s: calloc", __func__);
		return NULL;
	}

	// 使用配置对象的event_base工作模式，有下面：
	// 多线程调用是不安全的，单线程非阻塞模式
	// EVENT_BASE_FLAG_NOLOCK = 0x01,
	// 忽略检查EVENT_*等环境变量
	// EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
	// 只用于windows
	// EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
	// 不使用缓存的时间，每次回调都会获取系统时间
	// EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
	// 如果使用epoll方法，则使用epoll内部的changelist
	// EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
	// 使用更精确的时间，但是可能性能会降低
	// EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
	if (cfg)
		base->flags = cfg->flags;

	// 获取cft中event_base工作模式是否包含EVENT_BASE_FLAG_IGNORE_ENV
	// 即查看是否忽略检查EVENT_*等环境变量
	// 默认情况下，cfg->flags = EVENT_BASE_FLAG_NOBLOCK，所以是没有设置忽略环境变量
	// 因此should_check_environment = 1，是应该检查环境变量
	should_check_environment =
	    !(cfg && (cfg->flags & EVENT_BASE_FLAG_IGNORE_ENV));

	// 花括号里面的内容代表记录事件初始化的准确时间
	{
		struct timeval tmp;
		// 获取cfg中event_base工作模式中是否包含使用精确时间的设置
		// EVENT_BASE_FLAG_PRECISE_TIME含义看上文
		// 如果使用精确时间，则precise_time为1，否则为0
		// 默认配置下，cfg->flags = EVENT_BASE_FLAG_NOBLOCK，所以precise_time = 0
		int precise_time =
		    cfg && (cfg->flags & EVENT_BASE_FLAG_PRECISE_TIMER);
		int flags;
		
		// 如果检查EVENT_*等环境变量并且不使用精确时间，
		// 则需要检查编译时的环境变量中是否打开了使用精确时间的模式；
		// 这一段检查的目的是说，虽然配置结构体struct event_config中没有指定
		// event_base使用精确时间的模式，但是libevent提供编译时使用环境变量来控制
		// 使用精确时间的模式，所以如果开启检查环境变量的开关，则需要检查是否在编译时
		// 打开了使用精确时间的模式，例如在CMakeList中就有有关开启选项。
		// 默认情况下，检查环境变量并且precise_time = 0，所以需要执行这个if分支。
		// 执行结果之后，precise_time = 1, base->flags = cfg->flags | EVENT_BASE_FLAG_PRECISE_TIMER
		// 即base->flags = EVENT_BASE_FLAG_NOBLOCK | EVENT_BASE_FLAG_PRECISE_TIME = 0x21
		if (should_check_environment && !precise_time) {
			// evutil_getenv_实际上是getenv函数的封装，用来获取环境变量。
			// 查看上面CMakeList中信息，可以看出实际上默认是打开的。
			precise_time = evutil_getenv_("EVENT_PRECISE_TIMER") != NULL;
			if (precise_time) {
				base->flags |= EVENT_BASE_FLAG_PRECISE_TIMER;
			}
		}

		// 根据precise_time的标志信息，确认是否使用MONOT_PRECISE模式
		// linux下实际上一般是调用clock_gettime实现
		// 默认情况下，precise_time = 1,所以 flags = EV_MONOT_PRECISE
		// 所以默认情况下，使用EV_MONOT_PRECISE模式配置event_base的monotonic_time
		flags = precise_time ? EV_MONOT_PRECISE : 0;
		// 后面会专门分析这个函数，详见1.2.2
		evutil_configure_monotonic_time_(&base->monotonic_timer, flags);

		// 根据base获取当前的时间
		// 如果base中有缓存的时间，则将缓存的时间赋给tmp，然后返回即可；
		// 如果base中没有缓存的时间，则使用clock_gettime获取当前系统的monotonic时间；
		// 否则根据上次更新系统时间的时间点，更新间隔，以及当前使用clock_gettime等函数
		// 获取当前的系统时间查看是否需要更新base->tv_clock_diff以及base->last_updated_clock_diff
		// 详见1.2.3
		gettime(base, &tmp);
	}

	// 最小堆初始化
	// void min_heap_ctor_(min_heap_t* s) { s->p = 0; s->n = 0; s->a = 0; } (位于minheap-internal.h)
	
	// typedef struct min_heap
	// {
			//struct event** p;
			//unsigned n, a;
	//} min_heap_t;
	min_heap_ctor_(&base->timeheap);

	// 内部信号通知的管道，0读1写
	base->sig.ev_signal_pair[0] = -1;
	base->sig.ev_signal_pair[1] = -1;
	// 内部线程通知的文件描述符，0读1写
	base->th_notify_fd[0] = -1;
	base->th_notify_fd[1] = -1;

	// 初始化下一次激活的队列
	TAILQ_INIT(&base->active_later_queue);

	// struct event_io_map io;
	// void evmap_io_initmap_(struct event_io_map *ctx)
	// {
		// HT_INIT(event_io_map, ctx);
	// }
	// #define HT_INIT(name, head)          name##_HT_INIT(head)
	// event_io_map_HI_INIT(ctc)
	/*
	 static inline void name##_HT_INIT(struct name *head) 
	 {
		 head->hth_table_length = 0;
		 head->hth_table = NULL;
		 head->hth_n_entries = 0;
		 head->hth_load_limit = 0;
		 head->hth_prime_idx = -1;
	 }               
  */
	// 初始化IO事件和文件描述符的映射
	// 如果是在win32下，则使用hash table，一些宏定义的实现，具体可以看event-interval.h中的EVMAP_USE_HT这个编译选项
	// 如果是在其他环境下，#define event_io_map event_signal_map
	// 则base->io名为struct event_io_map，实际上是struct event_signal_map
	// 因此linux下实际调用的是evmap_signal_initmap_，后面会大体分析一下这个函数
	// 详见1.2.4
	evmap_io_initmap_(&base->io);
	
	// 初始化信号和文件描述符的映射
	// 后面会分析这个函数
	// 详见1.2.4
	evmap_signal_initmap_(&base->sigmap);
	
	/*
	void event_changelist_init_(struct event_changelist *changelist)
	{
		changelist->changes = NULL;
		changelist->changes_size = 0;
		changelist->n_changes = 0;
	}
	*/
	event_changelist_init_(&base->changelist);

	// 在没有初始化后台方法之前，后台方法必需的数据信息为空
	base->evbase = NULL;

	// 配置event loop的检查回调函数的间隔信息，默认情况下，是不检查的，即一旦event_base确认当前的
	// 优先级是最高优先级队列，则会一直把该优先队列事件中同优先级事件都执行完再检查是否有更高优先级事件发生
	// 在执行过程中，即使有高优先级事件触发，event_base也不会中断去执行；
	// 配置之后的优点是，如果配置可以支持优先级抢占，提高高优先级事件执行效率，
	// 配置之后的缺点是，但是一旦配置了，每次执行完回调函数都需要检查，则会轻微降低事件吞吐量
	// max_dispatch_callbacks：event_base在两次检查新事件之间的执行回调函数个数的最大个数
	// max_dispatch_interval：event_base在两次检查新事件之间消耗的最大时间。
	// limit_callbacks_after_prio：是只有优先级数字>=limit_callbacks_after_prio的事件触发时，
	// 才会强迫event_base 去检查是否有更高优先级事件发生，低于这个优先级的事件触发时，不会检查。

	// 如果配置信息对象struct event_config存在，则依据配置信息配置，
	// 否则，赋给默认初始值
	if (cfg) {
		// 配置非空，将cfg的max_dispatch_interval复制到base的max_dispatch_interval里面。
		memcpy(&base->max_dispatch_time,
		    &cfg->max_dispatch_interval, sizeof(struct timeval));
		base->limit_callbacks_after_prio =
		    cfg->limit_callbacks_after_prio;
	} else {
		// max_dispatch_time.tv_sec = -1即不进行此项检查
		// limit_callbacks_after_prior=1 是指>=1时都需要检查，最高优先级为0，
		// 即除了最高优先级事件执行时不需要检查之外，其他优先级都需要检查
		base->max_dispatch_time.tv_sec = -1;
		base->limit_callbacks_after_prio = 1;
	}
	if (cfg && cfg->max_dispatch_callbacks >= 0) {
		base->max_dispatch_callbacks = cfg->max_dispatch_callbacks;
	} else {
		base->max_dispatch_callbacks = INT_MAX;
	}
	if (base->max_dispatch_callbacks == INT_MAX &&
	    base->max_dispatch_time.tv_sec == -1)
		base->limit_callbacks_after_prio = INT_MAX;
	// 上面这三个变量将会在事件回调的时候使用到


	// 遍历静态全局变量eventops，对选择的后台方法进行初始化
	// eventops定义：
	// 关于event_base和eventop，详见1.2.1.和1.15
	// 这里根据操作系统(linux)的测试结果是，默认选择使用epoll方法，而测试环境支持的后台方法包括
	// epoll，poll，select
	// 下面的程序选出了epoll方法，跳过poll和select方法的
	for (i = 0; eventops[i] && !base->evbase; i++) {
		if (cfg != NULL) {
			/* determine if this backend should be avoided */
			// 如果方法以及屏蔽，则跳过去，继续遍历下一个方法
			// 默认情况下，是不屏蔽任何后台方法，除非通过编译选项控制或者使用API屏蔽

			// 遍历cfg->entries
			if (event_config_is_avoided_method(cfg,
				eventops[i]->name))
				continue;
				
			// 如果后台方法的工作模式特征和配置的工作模式不同，则跳过去
			// 查看libevent源码中epoll.c，poll.c,select.c文件相应静态全局变量的方法定义
			// 发现默认情况下，各个后台方法的特征如下：
			// epoll方法的特征是:EV_FEATURE_ET|EV_FEATURE_O1|EV_FEATURE_EARLY_CLOSE
			// poll方法的特征是：EV_FEATURE_FDS
			// select方法的特征是：EV_FEATURE_FDS
			// 默认情况下，cfg中require_features 是EV_FEATURE_ET，所以poll和select都会跳过去，
			// 只有epoll执行初始化
			if ((eventops[i]->features & cfg->require_features)
			    != cfg->require_features)
				continue;
		}

		/* also obey the environment variables */
		// 如果检查环境变量，并发现OS环境不支持的话，也会跳过去
		if (should_check_environment &&
		    event_is_method_disabled(eventops[i]->name))
			continue;

		// 下面两步正确情况下只会选择一个后台方法，也只会执行一次
		// 保存后台方法句柄，实际是静态全局变量数组成员，具体定义在每种方法文件中定义
		base->evsel = eventops[i];

		// 调用相应后台方法的初始化函数进行初始化，这个就用到结构体
		// struct eventops中的定义的init方法，具体init方法的实现需要查看每种方法自己的定义
		// 以epoll为例，epoll相关实现都在epoll.c中
		// 后面会分析epoll的初始化函数
		base->evbase = base->evsel->init(base);  // void * evbase;
	}

	// 如果遍历一遍没有发现合适的后台方法，就报错退出，退出前释放资源
	if (base->evbase == NULL) {
		event_warnx("%s: no event mechanism available",
		    __func__);
		base->evsel = NULL;
		event_base_free(base);
		return NULL;
	}

	// 获取环境变量EVENT_SHOW_METHOD，是否打印输出选择的后台方法名字
	if (evutil_getenv_("EVENT_SHOW_METHOD"))
		event_msgx("libevent using: %s", base->evsel->name);

	/* allocate a single active event queue */
	// 分配优先级队列成员个数，前面分配event_base时会填入初始值1
	// 此处分配的优先队列成员个数为1，否则直接返回了。

	// 最开始只建立一条优先级的队列
	if (event_base_priority_init(base, 1) < 0) {
		event_base_free(base);
		return NULL;
	}

	/* prepare for threading */

#if !defined(EVENT__DISABLE_THREAD_SUPPORT) && !defined(EVENT__DISABLE_DEBUG_MODE)
	event_debug_created_threadable_ctx_ = 1;
#endif

#ifndef EVENT__DISABLE_THREAD_SUPPORT
// 多线程模式
	if (EVTHREAD_LOCKING_ENABLED() &&
	    (!cfg || !(cfg->flags & EVENT_BASE_FLAG_NOLOCK))) {
		int r;
		EVTHREAD_ALLOC_LOCK(base->th_base_lock, 0);
		EVTHREAD_ALLOC_COND(base->current_event_cond);
		r = evthread_make_base_notifiable(base);
		if (r<0) {
			event_warnx("%s: Unable to make base notifiable.", __func__);
			event_base_free(base);
			return NULL;
		}
	}
#endif

#ifdef _WIN32
	if (cfg && (cfg->flags & EVENT_BASE_FLAG_STARTUP_IOCP))
		event_base_start_iocp_(base, cfg->n_cpus_hint);
#endif

	// 注意：其他没有显式初始化的域都被前面分配空间时默认初始化为1
	return (base);
}
```

####1.2.1.event_base数据结构

位于event-internal.h，代码如下：
```cpp
struct event_base {
	/** Function pointers and other data to describe this event_base's
	 * backend. */
	const struct eventop *evsel;
	/** Pointer to backend-specific data. */
	void *evbase;

	/** List of changes to tell backend about at next dispatch.  Only used
	 * by the O(1) backends. */
	struct event_changelist changelist;

	/** Function pointers used to describe the backend that this event_base
	 * uses for signals */
	const struct eventop *evsigsel;
	/** Data to implement the common signal handler code. */
	struct evsig_info sig;

	/** Number of virtual events */
	int virtual_event_count;
	/** Maximum number of virtual events active */
	int virtual_event_count_max;
	/** Number of total events added to this event_base */
	int event_count;
	/** Maximum number of total events added to this event_base */
	int event_count_max;
	/** Number of total events active in this event_base */
	int event_count_active;
	/** Maximum number of total events active in this event_base */
	int event_count_active_max;

	/** Set if we should terminate the loop once we're done processing
	 * events. */
	int event_gotterm;
	/** Set if we should terminate the loop immediately */
	int event_break;
	/** Set if we should start a new instance of the loop immediately. */
	int event_continue;

	/** The currently running priority of events */
	int event_running_priority;

	/** Set if we're running the event_base_loop function, to prevent
	 * reentrant invocation. */
	int running_loop;

	/** Set to the number of deferred_cbs we've made 'active' in the
	 * loop.  This is a hack to prevent starvation; it would be smarter
	 * to just use event_config_set_max_dispatch_interval's max_callbacks
	 * feature */
	int n_deferreds_queued;

	/* Active event management. */
	/** An array of nactivequeues queues for active event_callbacks (ones
	 * that have triggered, and whose callbacks need to be called).  Low
	 * priority numbers are more important, and stall higher ones.
	 */
	struct evcallback_list *activequeues;
	/** The length of the activequeues array */
	int nactivequeues;
	/** A list of event_callbacks that should become active the next time
	 * we process events, but not this time. */
	struct evcallback_list active_later_queue;

	/* common timeout logic */

	/** An array of common_timeout_list* for all of the common timeout
	 * values we know. */
	struct common_timeout_list **common_timeout_queues;
	/** The number of entries used in common_timeout_queues */
	int n_common_timeouts;
	/** The total size of common_timeout_queues. */
	int n_common_timeouts_allocated;

	/** Mapping from file descriptors to enabled (added) events */
	struct event_io_map io;

	/** Mapping from signal numbers to enabled (added) events. */
	struct event_signal_map sigmap;

	/** Priority queue of events with timeouts. */
	struct min_heap timeheap;

	/** Stored timeval: used to avoid calling gettimeofday/clock_gettime
	 * too often. */
	struct timeval tv_cache;

	struct evutil_monotonic_timer monotonic_timer;

	/** Difference between internal time (maybe from clock_gettime) and
	 * gettimeofday. */
	struct timeval tv_clock_diff;
	/** Second in which we last updated tv_clock_diff, in monotonic time. */
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
	struct event_callback *current_event;

#ifdef _WIN32
	/** IOCP support structure, if IOCP is enabled. */
	struct event_iocp_port *iocp;
#endif

	/** Flags that this base was configured with */
	enum event_base_config_flag flags;

	struct timeval max_dispatch_time;
	int max_dispatch_callbacks;
	int limit_callbacks_after_prio;

	/* Notify main thread to wake up break, etc. */
	/** True if the base already has a pending notify, and we don't need
	 * to add any more. */
	int is_notify_pending;
	/** A socketpair used by some th_notify functions to wake up the main
	 * thread. */
	evutil_socket_t th_notify_fd[2];
	/** An event used by some th_notify functions to wake up the main
	 * thread. */
	struct event th_notify;
	/** A function used to wake up the main thread from another thread. */
	int (*th_notify_fn)(struct event_base *base);

	/** Saved seed for weak random number generator. Some backends use
	 * this to produce fairness among sockets. Protected by th_base_lock. */
	struct evutil_weakrand_state weakrand_seed;

	/** List of event_onces that have not yet fired. */
	LIST_HEAD(once_event_list, event_once) once_events;

};
```

###1.3.event_config_free函数

位于event.c，代码如下：
```cpp
void
event_config_free(struct event_config *cfg)
{
	struct event_config_entry *entry;

	// 遍历屏蔽的后台方法列表，释放屏蔽的后台方法项目
	while ((entry = TAILQ_FIRST(&cfg->entries)) != NULL) {
		TAILQ_REMOVE(&cfg->entries, entry, next);
		event_config_entry_free(entry);
	}

	// mm_calloc的释放程序
	mm_free(cfg);
}
```
做一些很简单的回收工作。


####1.2.1.event_base_free函数

位于1.1.`hellow-world.c`的末尾，用于回收`event_base`资源的函数。位于event.c，代码如下：

```cpp
// 释放event_base所属的所有空间，并释放event_base；
// 注意：这个函数不会关闭任何fds或者释放在执行event_new时任何传递给callback的参数
// 如果未决的关闭类型的回调，本函数会唤醒这些回调
void
event_base_free(struct event_base *base)
{
	// 实际上是调用内部释放函数释放的。
	event_base_free_(base, 1);
}

static void
event_base_free_(struct event_base *base, int run_finalizers)
{
	int i, n_deleted=0;
	struct event *ev;
	/* XXXX grab the lock? If there is contention when one thread frees
	 * the base, then the contending thread will be very sad soon. */

	/* event_base_free(NULL) is how to free the current_base if we
	 * made it with event_init and forgot to hold a reference to it. */
	if (base == NULL && current_base)
		base = current_base;
	/* Don't actually free NULL. */
	if (base == NULL) {
		event_warnx("%s: no base to free", __func__);
		return;
	}
	/* XXX(niels) - check for internal events first */

#ifdef _WIN32
	event_base_stop_iocp_(base);
#endif

	/* threading fds if we have them */
	if (base->th_notify_fd[0] != -1) {
		event_del(&base->th_notify);
		EVUTIL_CLOSESOCKET(base->th_notify_fd[0]);
		if (base->th_notify_fd[1] != -1)
			EVUTIL_CLOSESOCKET(base->th_notify_fd[1]);
		base->th_notify_fd[0] = -1;
		base->th_notify_fd[1] = -1;
		event_debug_unassign(&base->th_notify);
	}

	/* Delete all non-internal events. */
	evmap_delete_all_(base);

	while ((ev = min_heap_top_(&base->timeheap)) != NULL) {
		event_del(ev);
		++n_deleted;
	}
	for (i = 0; i < base->n_common_timeouts; ++i) {
		struct common_timeout_list *ctl =
		    base->common_timeout_queues[i];
		event_del(&ctl->timeout_event); /* Internal; doesn't count */
		event_debug_unassign(&ctl->timeout_event);
		for (ev = TAILQ_FIRST(&ctl->events); ev; ) {
			struct event *next = TAILQ_NEXT(ev,
			    ev_timeout_pos.ev_next_with_common_timeout);
			if (!(ev->ev_flags & EVLIST_INTERNAL)) {
				event_del(ev);
				++n_deleted;
			}
			ev = next;
		}
		mm_free(ctl);
	}
	if (base->common_timeout_queues)
		mm_free(base->common_timeout_queues);

	for (;;) {
		/* For finalizers we can register yet another finalizer out from
		 * finalizer, and iff finalizer will be in active_later_queue we can
		 * add finalizer to activequeues, and we will have events in
		 * activequeues after this function returns, which is not what we want
		 * (we even have an assertion for this).
		 *
		 * A simple case is bufferevent with underlying (i.e. filters).
		 */
		int i = event_base_free_queues_(base, run_finalizers);
		if (!i) {
			break;
		}
		n_deleted += i;
	}

	if (n_deleted)
		event_debug(("%s: %d events were still set in base",
			__func__, n_deleted));

	while (LIST_FIRST(&base->once_events)) {
		struct event_once *eonce = LIST_FIRST(&base->once_events);
		LIST_REMOVE(eonce, next_once);
		mm_free(eonce);
	}

	if (base->evsel != NULL && base->evsel->dealloc != NULL)
		base->evsel->dealloc(base);

	for (i = 0; i < base->nactivequeues; ++i)
		EVUTIL_ASSERT(TAILQ_EMPTY(&base->activequeues[i]));

	EVUTIL_ASSERT(min_heap_empty_(&base->timeheap));
	min_heap_dtor_(&base->timeheap);

	mm_free(base->activequeues);

	evmap_io_clear_(&base->io);
	evmap_signal_clear_(&base->sigmap);
	event_changelist_freemem_(&base->changelist);

	EVTHREAD_FREE_LOCK(base->th_base_lock, 0);
	EVTHREAD_FREE_COND(base->current_event_cond);

	/* If we're freeing current_base, there won't be a current_base. */
	if (base == current_base)
		current_base = NULL;
	mm_free(base);
}
```


####1.2.2.evutil_configure_monotonic_time_函数

位于evutil_time.c，代码如下：

+ 配置单调递增的时间，默认情况下，会使用`EV_MONOTONIC`模式获取系统时间，并将`event_base`的`monotonic_clock`模式设置为`EV_MONOTONIC`；
+ `monotonic`时间是单调递增的时间，不受系统修改时间影响，对于计算两个时间点之间的时间比较精确；
+ `real`时间是系统时间，受系统时间影响

注意：libevent支持多种操作系统模式，以下只看Linux下的。

```cpp
#if defined(HAVE_POSIX_MONOTONIC)
/* =====
   The POSIX clock_gettime() interface provides a few ways to get at a
   monotonic clock.  CLOCK_MONOTONIC is most widely supported.  Linux also
   provides a CLOCK_MONOTONIC_COARSE with accuracy of about 1-4 msec.

   On all platforms I'm aware of, CLOCK_MONOTONIC really is monotonic.
   Platforms don't agree about whether it should jump on a sleep/resume.
 */

// POSIX clock_gettime接口提供获得monotonic时间的方式。CLOCK_MONOTONIC基本上都是支持的；
// linux也提供CLOCK_MONOTONIC_COARSE模式，大约1-4毫秒的准确性。
// 所有平台上，CLOCK_MONOTONIC实际上是单调递增的。

// 调用的地方：evutil_configure_monotonic_time_(&base->monotonic_timer, flags);
// 关于struct evutil_monotonic_timer，位于util.h
int
evutil_configure_monotonic_time_(struct evutil_monotonic_timer *base,
    int flags)
{
	/* CLOCK_MONOTONIC exists on FreeBSD, Linux, and Solaris.  You need to
	 * check for it at runtime, because some older kernel versions won't
	 * have it working. */

// 如果定义了CLOCK_MONOTONIC_COARSE编译选项，则检查event_base工作模式是否选择了EV_MONOT_PRECISE；
// 如果选择了EV_MONOT_PRECISE，则设置标志位precise，表明选择使用准确的时间模式；
// 默认情况下，flags采用的是EV_MONOT_PRECISE，所以precise＝EV_MONOT_PRECISE＝1
#ifdef CLOCK_MONOTONIC_COARSE
	const int precise = flags & EV_MONOT_PRECISE;  // = 1 (default)
#endif
	// 设置fallback标志位，查看是否为EV_MONOT_FALLBACK=2
	// 默认情况下，flags采用的是EV_MONOT_PRECISE，所以fallback为0
	const int fallback = flags & EV_MONOT_FALLBACK; // = 0 ()
	struct timespec	ts;

#ifdef CLOCK_MONOTONIC_COARSE
	if (CLOCK_MONOTONIC_COARSE < 0) {
		/* Technically speaking, nothing keeps CLOCK_* from being
		 * negative (as far as I know). This check and the one below
		 * make sure that it's safe for us to use -1 as an "unset"
		 * value. */
		event_errx(1,"I didn't expect CLOCK_MONOTONIC_COARSE to be < 0");
	}

	// 如果既没有选择EV_MONOT_PRECISE模式，也没有选择EV_MONOT_FALLBACK模式，
	// 则使用CLOCK_MONOTONIC_COARSE获取当前系统时间
	// 默认情况下选择的是EV_MONOT_PRECISE，所以不走此分支
	if (! precise && ! fallback) {
		if (clock_gettime(CLOCK_MONOTONIC_COARSE, &ts) == 0) {
			base->monotonic_clock = CLOCK_MONOTONIC_COARSE;
			return 0;
		}
	}
#endif

	// 如果没有选择EV_MONOT_FALLBACK，则以CLOCK_MONOTONIC模式获取系统时间，并将event_base的monotonic_clock模式设置为CLOCK_MONOTONIC；     
	// 默认情况下，选择的是EV_MONOT_PRECISE，所以此分支执行，此函数最终的结果是获取CLOCK_MONOTONIC模式的时间，并将event_base的时钟模式设置为CLOCK_MONOTONIC模式
	if (!fallback && clock_gettime(CLOCK_MONOTONIC, &ts) == 0) {
		base->monotonic_clock = CLOCK_MONOTONIC;
		return 0;
	}

	if (CLOCK_MONOTONIC < 0) {
		event_errx(1,"I didn't expect CLOCK_MONOTONIC to be < 0");
	}

	base->monotonic_clock = -1;
	return 0;
}

int
evutil_gettime_monotonic_(struct evutil_monotonic_timer *base,
    struct timeval *tp)
{
	struct timespec ts;

	if (base->monotonic_clock < 0) {
		if (evutil_gettimeofday(tp, NULL) < 0)
			return -1;
		adjust_monotonic_time(base, tp);
		return 0;
	}

	if (clock_gettime(base->monotonic_clock, &ts) == -1)
		return -1;
	tp->tv_sec = ts.tv_sec;
	tp->tv_usec = ts.tv_nsec / 1000;

	return 0;
}
#endif
```

其中，`CLOCK_MONOTONIC`的说明如下：

> 在一些系统调用中需要指定时间是用CLOCK_MONOTONIC还是CLOCK_REALTIME，以前总是搞不太清楚它们之间的差别，现在终于有所理解了。
> CLOCK_MONOTONIC是monotonic time，而CLOCK_REALTIME是wall time。monotonic time字面意思是单调时间，实际上它指的是系统启动以后流逝的时间，这是由变量jiffies来记录的。系统每次启动时jiffies初始化为0，每来一个timer interrupt，jiffies加1，也就是说它代表系统启动后流逝的tick数。jiffies一定是单调递增的，因为时间不够逆嘛！wall time字面意思是挂钟时间，实际上就是指的是现实的时间，这是由变量xtime来记录的。系统每次启动时将CMOS上的RTC时间读入xtime，这个值是"自1970-01-01起经历的秒数、本秒中经历的纳秒数"，每来一个timer interrupt，也需要去更新xtime。
> 以前我一直想不明白，既然每个timer interrupt，jiffies和xtime都要更新，那么不都是单调递增的吗？那它们之间使用时有什么区别呢？昨天看到一篇文章，终于明白了，wall time不一定是单调递增的。因为wall time是指现实中的实际时间，如果系统要与网络中某个节点时间同步、或者由系统管理员觉得这个wall time与现实时间不一致，有可能任意的改变这个wall time。最简单的例子是，我们用户可以去任意修改系统时间，这个被修改的时间应该就是wall time，即xtime，它甚至可以被写入RTC而永久保存。一些应用软件可能就是用到了这个wall time，比如以前用vmware workstation，一启动提示试用期已过，但是只要把系统时间调整一下提前一年，再启动就不会有提示了，这很可能就是因为它启动时用gettimeofday去读wall time，然后判断是否过期，只要将wall time改一下，就可以欺骗过去了。


####1.2.3.gettime函数

位于event.c，代码如下：

此函数主要用来获取当前`event_base`中缓存的时间，并设置使用系统时间更新`event_base`缓存时间的时间间隔，并获取当前系统的monotonic时间和当前系统的real时间之间差值并存储在`event_base`中。

```cpp
/** Set 'tp' to the current time according to 'base'.  We must hold the lock
 * on 'base'.  If there is a cached time, return it.  Otherwise, use
 * clock_gettime or gettimeofday as appropriate to find out the right time.
 * Return 0 on success, -1 on failure.
 */
// 将tp设置为base的当前时间。
// 必须在base上加锁；如果有缓存的时间，则返回缓存的时间
// 否则，需要调用clock_gettime或者gettimeofday获取合适的时间
static int
gettime(struct event_base *base, struct timeval *tp)
{
	EVENT_BASE_ASSERT_LOCKED(base);

	// 首先查看base中是否有缓存的时间，如果有，直接使用缓存时间，然后返回即可。
	if (base->tv_cache.tv_sec) {
		*tp = base->tv_cache;
		return (0);
	}

	// 使用monotonic_timer获取当前时间，二选一
	// CLOCK_MONOTONIC_COARSE 和 CLOCK_MONOTONIC
	// 默认情况下是 CLOCK_MONOTONIC模式
	if (evutil_gettime_monotonic_(&base->monotonic_timer, tp) == -1) {
		return -1;
	}

	// 查看是否需要更新缓存的时间
	// 如果上次更新时间的时间点距离当前时间点的间隔超过CLOCK_SYNC_INTERVAL，则需要更新
	if (base->last_updated_clock_diff + CLOCK_SYNC_INTERVAL
	    < tp->tv_sec) {
		struct timeval tv;
		// 使用gettimeofday获取当前系统real时间
		evutil_gettimeofday(&tv,NULL);
		// 将当前系统real时间和monotionic时间做差，存到base中
		evutil_timersub(&tv, tp, &base->tv_clock_diff);
		// 保存当前更新的时间点
		base->last_updated_clock_diff = tp->tv_sec;
	}

	return 0;
}
```

####1.2.4.evmap_io_initmap_和evmap_signal_initmap_函数

当系统不是win32的时候，`event_signal_map`和`event_io_map`是相同的。

位于evmap.c，代码如下：

```cpp
void
evmap_io_initmap_(struct event_io_map* ctx)
{
	evmap_signal_initmap_(ctx);
}


void
evmap_signal_initmap_(struct event_signal_map *ctx)
{
	ctx->nentries = 0;
	ctx->entries = NULL;
}


// event_io_map：用来存储fds与一系列事件之间的映射关系。
// event_signal_map：用来存储信号与一系列事件之间的映射关系。
#define event_io_map event_signal_map
#endif

/* Used to map signal numbers to a list of events.  If EVMAP_USE_HT is not
   defined, this structure is also used as event_io_map, which maps fds to a
   list of events.
*/
struct event_signal_map {
	/* An array of evmap_io * or of evmap_signal *; empty entries are
	 * set to NULL. */
	// 存放的是evmap_io和evmap_signal对象的指针。空entries设置为NULL
	void **entries;
	/* The number of entries available in entries */
	// 可用项目的个数
	int nentries;
};
```


####1.2.5.初始化变化列表函数——event_changelist_init_

位于evmap.c，代码如下：

```cpp
void
event_changelist_init_(struct event_changelist *changelist)
{
	changelist->changes = NULL;
	changelist->changes_size = 0;
	changelist->n_changes = 0;
}

// 列举自从上一次eventop.dispatch调用之后的改变列表。只有在后台使用改变集合时才会维护这个列表
/* List of 'changes' since the last call to eventop.dispatch.  Only maintained
 * if the backend is using changesets. */
struct event_changelist {
	struct event_change *changes;
	int n_changes;
	int changes_size;
};
```

####1.2.5.epoll的初始化

位于`epoll.c`。

epoll.c中，epoll后台方法定义：

参考前文中在event.c中定义的静态全局变量数组：`eventops`，
`eventops`中存储就是下面的`epollops`。
此处需要注意的是内部信号通知机制的实现，是通过管道传递外部信号的，为何不能像IO事件绑定文件描述符？
因为信号是全局的，无法真对某个事件进行绑定，只能通过函数捕捉信号，然后通过管道通知event_base来实现。

```cpp
const struct eventop epollops = {
	"epoll",
	epoll_init,
	epoll_nochangelist_add,
	epoll_nochangelist_del,
	epoll_dispatch,
	epoll_dealloc,
	1, /* need reinit */
	EV_FEATURE_ET|EV_FEATURE_O1|EV_FEATURE_EARLY_CLOSE,
	0
};

// 下面是eventop结构体的定义
/** Structure to define the backend of a given event_base. */
struct eventop {
	/** The name of this backend. */
	const char *name;
	/** Function to set up an event_base to use this backend.  It should
	 * create a new structure holding whatever information is needed to
	 * run the backend, and return it.  The returned pointer will get
	 * stored by event_init into the event_base.evbase field.  On failure,
	 * this function should return NULL. */
	void *(*init)(struct event_base *);
	/** Enable reading/writing on a given fd or signal.  'events' will be
	 * the events that we're trying to enable: one or more of EV_READ,
	 * EV_WRITE, EV_SIGNAL, and EV_ET.  'old' will be those events that
	 * were enabled on this fd previously.  'fdinfo' will be a structure
	 * associated with the fd by the evmap; its size is defined by the
	 * fdinfo field below.  It will be set to 0 the first time the fd is
	 * added.  The function should return 0 on success and -1 on error.
	 */
	int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
	/** As "add", except 'events' contains the events we mean to disable. */
	int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
	/** Function to implement the core of an event loop.  It must see which
	    added events are ready, and cause event_active to be called for each
	    active event (usually via event_io_active or such).  It should
	    return 0 on success and -1 on error.
	 */
	int (*dispatch)(struct event_base *, struct timeval *);
	/** Function to clean up and free our data from the event_base. */
	void (*dealloc)(struct event_base *);
	/** Flag: set if we need to reinitialize the event base after we fork.
	 */
	int need_reinit;
	/** Bit-array of supported event_method_features that this backend can
	 * provide. */
	enum event_method_feature features;
	/** Length of the extra information we should record for each fd that
	    has one or more active events.  This information is recorded
	    as part of the evmap entry for each fd, and passed as an argument
	    to the add and del functions above.
	 */
	size_t fdinfo_len;
};
```

在上面`event_base_new_with_config`函数里面，有这样初始化事件的逻辑：
```cpp
		// 下面两步正确情况下只会选择一个后台方法，也只会执行一次
		// 保存后台方法句柄，实际是静态全局变量数组成员，具体定义在每种方法文件中定义
		base->evsel = eventops[i];

		// 调用相应后台方法的初始化函数进行初始化，这个就用到结构体
		// struct eventops中的定义的init方法，具体init方法的实现需要查看每种方法自己的定义
		// 以epoll为例，epoll相关实现都在epoll.c中
		// 后面会分析epoll的初始化函数
		base->evbase = base->evsel->init(base);
```

上面`init`方法对应epoll.c里面的`epoll_init`函数。代码如下：

```cpp
/*
struct epollop {
	struct epoll_event *events;
	int nevents;
	int epfd;
#ifdef USING_TIMERFD
	int timerfd;
#endif
};
*/
static void *
epoll_init(struct event_base *base)
{
	int epfd = -1;
	// epollop结构体详见1.2.5.1
	struct epollop *epollop;

// epoll已经提供新的创建API，epoll_create1方法可以设置创建模式，
// 可以通过编译选项进行配置
#ifdef EVENT__HAVE_EPOLL_CREATE1
	/* First, try the shiny new epoll_create1 interface, if we have it. */
	// EPOLL_CLOEXEC——表示生成的epoll fd具有“执行后关闭”特性。
	epfd = epoll_create1(EPOLL_CLOEXEC);
#endif
	if (epfd == -1) {
		/* Initialize the kernel queue using the old interface.  (The
		size field is ignored   since 2.6.8.) */
		// 如果内核不支持epoll_create1方法，只能使用epoll_create创建
		if ((epfd = epoll_create(32000)) == -1) {
			if (errno != ENOSYS)
				event_warn("epoll_create");
			return (NULL);
		}
		evutil_make_socket_closeonexec(epfd);
	}

	// 分配struct epollop对象，并赋初值1
	if (!(epollop = mm_calloc(1, sizeof(struct epollop)))) {
		close(epfd);
		return (NULL);
	}

	// 保存epoll句柄描述符
	epollop->epfd = epfd;

	/* Initialize fields */
	// 初始化epoll句柄可以处理的事件最大个数以及当前注册事件数的初始值
	epollop->events = mm_calloc(INITIAL_NEVENT, sizeof(struct epoll_event));
	if (epollop->events == NULL) {
		mm_free(epollop);
		close(epfd);
		return (NULL);
	}
	epollop->nevents = INITIAL_NEVENT;

	// 如果libevent工作模式是启用epoll的changelist方式，则后台方法变为epollops_changelist
	// 默认情况下是不选择的。
	if ((base->flags & EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST) != 0 ||
	    ((base->flags & EVENT_BASE_FLAG_IGNORE_ENV) == 0 &&
		evutil_getenv_("EVENT_EPOLL_USE_CHANGELIST") != NULL)) {

		base->evsel = &epollops_changelist;
	}

#ifdef USING_TIMERFD
	/*
	  The epoll interface ordinarily gives us one-millisecond precision,
	  so on Linux it makes perfect sense to use the CLOCK_MONOTONIC_COARSE
	  timer.  But when the user has set the new PRECISE_TIMER flag for an
	  event_base, we can try to use timerfd to give them finer granularity.
	*/
	// epoll接口通常的精确度是1微秒，因此在Linux环境上，通过使用CLOCK_MONOTONIC_COARSE计时器,可以获得完美的体验。
	// 但是当用户使用新的PRECISE_TIMER模式时，可以使用timerfd获取更细的时间粒度。timerfd是Linux为用户提供的定时器接口，基于文件描述符，通过文件描述符的可读事件进行超时通知，timerfd、eventfd、signalfd配合epoll使用，可以构造出零轮询的程序，但是程序没有处理的事件时，程序是被阻塞的；timerfd的精度要比uspleep高

	// timerfd是Linux为用户程序提供的一个定时器接口。这个接口基于文件描述符，通过文件描述符的可读事件进行超时通知，所以能够被用于select/poll的应用场景。timerfd是linux内核2.6.25版本中加入的借口。timerfd、eventfd、signalfd配合epoll使用，可以构造出一个零轮询的程序，但程序没有处理的事件时，程序是被阻塞的。这样的话在某些移动设备上程序更省电。
	if ((base->flags & EVENT_BASE_FLAG_PRECISE_TIMER) &&
	    base->monotonic_timer.monotonic_clock == CLOCK_MONOTONIC) {
		int fd;
		// 构造定时器文件，能够做到零轮询直接等待到事件触发
		fd = epollop->timerfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK|TFD_CLOEXEC);
		if (epollop->timerfd >= 0) {
			struct epoll_event epev;
			memset(&epev, 0, sizeof(epev));
			epev.data.fd = epollop->timerfd;
			epev.events = EPOLLIN;
			if (epoll_ctl(epollop->epfd, EPOLL_CTL_ADD, fd, &epev) < 0) {
				event_warn("epoll_ctl(timerfd)");
				close(fd);
				epollop->timerfd = -1;
			}
		} else {
			if (errno != EINVAL && errno != ENOSYS) {
				/* These errors probably mean that we were
				 * compiled with timerfd/TFD_* support, but
				 * we're running on a kernel that lacks those.
				 */
				event_warn("timerfd_create");
			}
			epollop->timerfd = -1;
		}
	} else {
		epollop->timerfd = -1;
	}
#endif

	// 初始化信号通知的管道
	// 当设置信号事件时，由于信号捕捉是针对全局来说，所以此处信号捕捉函数是如何通知event_base的呢？
	// 答案是通过内部管道来实现的，一旦信号捕捉函数捕捉到信号，则将相应信号通过管道传递给event_base，然后event_base根据信号值将相应的回调事件加入激活事件队列，等待event_loop的回调。
	// 下文有分析此函数
	// 详见
	evsig_init_(base);

	return (epollop);
}
```
**注意：关于信号事件处理的机制，详见1.28~1.35。**

#####1.2.5.1.epollop结构体

位于epoll.c，代码如下：

```cpp
struct epollop {
	// epoll事件定义的结构体，详见epoll的实现
	struct epoll_event *events;
	// epoll当前注册的事件总数
	int nevents;
	// epoll返回的句柄
	int epfd;
#ifdef USING_TIMERFD
	int timerfd;
#endif
};
```
