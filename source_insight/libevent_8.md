#libevent(8)
@(源码)


## 2.signal-test.c

可以看出来是信号事件相关的demo，看下代码：

```cpp
/*
 * Compile with:
 * cc -I/usr/local/include -o signal-test \
 *   signal-test.c -L/usr/local/lib -levent
 */

#include <sys/types.h>

#include <event2/event-config.h>

#include <sys/stat.h>
#ifndef _WIN32
#include <sys/queue.h>
#include <unistd.h>
#include <sys/time.h>
#else
#include <winsock2.h>
#include <windows.h>
#endif
#include <signal.h>
#include <fcntl.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

#include <event2/event.h>

int called = 0;

static void
signal_cb(evutil_socket_t fd, short event, void *arg)
{
	struct event *signal = arg;

	printf("signal_cb: got signal %d\n", event_get_signal(signal));

	if (called >= 2)
		event_del(signal);

	called++;
}

int
main(int argc, char **argv)
{
	struct event *signal_int;
	struct event_base* base;
#ifdef _WIN32
	WORD wVersionRequested;
	WSADATA wsaData;

	wVersionRequested = MAKEWORD(2, 2);

	(void) WSAStartup(wVersionRequested, &wsaData);
#endif

	/* Initalize the event library */
	base = event_base_new();

	/* Initalize one event */
	signal_int = evsignal_new(base, SIGINT, signal_cb, event_self_cbarg());

	event_add(signal_int, NULL);

	event_base_dispatch(base);
	event_free(signal_int);
	event_base_free(base);

	return (0);
}
```

如果看过例子1，hello-world.c的话，就会觉得这个例子并不复杂，就是单纯建立一个信号的事件`signal_int`，针对`SIGINT`进行`signal_cb`操作。`signal_cb`的细节如下：

```cpp
static void
signal_cb(evutil_socket_t fd, short event, void *arg)
{
	struct event *signal = arg;

	printf("signal_cb: got signal %d\n", event_get_signal(signal));

	if (called >= 2)
		event_del(signal);

	called++;
}
```

我们可以看到，这里对全局变量`called`进行计数，超过了2次之后，就会自动删除事件，删除事件的逻辑之前没讲过，可以讲一下。

### 2.1.event_del函数

位于event.c，代码如下：

注意：这里只是删除，并不是释放事件空间。

```cpp
// 从一系列监听的事件中移除事件，函数event_del将取消参数ev中的event
// 如果event已经执行或者还没有添加成功，则此调用无效。
int
event_del(struct event *ev)
{
	return event_del_(ev, EVENT_DEL_AUTOBLOCK);
}

static int
event_del_(struct event *ev, int blocking)
{
	int res;
	struct event_base *base = ev->ev_base;

	if (EVUTIL_FAILURE_CHECK(!base)) {
		event_warnx("%s: event has no event_base set.", __func__);
		return -1;
	}

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
	res = event_del_nolock_(ev, blocking);
	EVBASE_RELEASE_LOCK(base, th_base_lock);

	return (res);
}

// 事件删除的内部实现，不加锁；
// 需要根据事件状态判断是否能够执行删除行为；
// 需要根据事件类型决定删除的具体操作。
int
event_del_nolock_(struct event *ev, int blocking)
{
	struct event_base *base;
	int res = 0, notify = 0;

	event_debug(("event_del: %p (fd "EV_SOCK_FMT"), callback %p",
		ev, EV_SOCK_ARG(ev->ev_fd), ev->ev_callback));

	/* An event without a base has not been added */
	if (ev->ev_base == NULL)
		return (-1);

	EVENT_BASE_ASSERT_LOCKED(ev->ev_base);

	// 如果事件已经处于结束中的状态，则不需要重复删除
	if (blocking != EVENT_DEL_EVEN_IF_FINALIZING) {
		if (ev->ev_flags & EVLIST_FINALIZING) {
			/* XXXX Debug */
			return 0;
		}
	}

	base = ev->ev_base;

	EVUTIL_ASSERT(!(ev->ev_flags & ~EVLIST_ALL));

	/* See if we are just active executing this event in a loop */
	// 如果是信号事件，同时信号触发事件不为0，则放弃执行这些回调函数
	if (ev->ev_events & EV_SIGNAL) {
		if (ev->ev_ncalls && ev->ev_pncalls) {
			/* Abort loop */
			*ev->ev_pncalls = 0;
		}
	}


	if (ev->ev_flags & EVLIST_TIMEOUT) {
		/* NOTE: We never need to notify the main thread because of a
		 * deleted timeout event: all that could happen if we don't is
		 * that the dispatch loop might wake up too early.  But the
		 * point of notifying the main thread _is_ to wake up the
		 * dispatch loop early anyway, so we wouldn't gain anything by
		 * doing it.
		 */
		// 如果事件状态处于已超时状态,
		// 从来不需要因为一个已删除的超时事件而通知主线程：即使我们不通知主线程，可能会发生的就是调度loop会过早的唤醒；
		// 但是通知主线程的点是尽可能早地唤醒调度loop，因此即使通知也不会获得更好的性能。

		// 从超时队列中移除事件
		event_queue_remove_timeout(base, ev);
	}

	// 如果事件正处于激活状态，则需要将事件的回调函数从激活队列中删除
	if (ev->ev_flags & EVLIST_ACTIVE)
		event_queue_remove_active(base, event_to_event_callback(ev));
	// 如果事件正处于下一次激活状态，则需要将事件的回调函数从下一次激活队列中删除
	else if (ev->ev_flags & EVLIST_ACTIVE_LATER)
		event_queue_remove_active_later(base, event_to_event_callback(ev));

	// 如果事件正处于已插入状态，则需要将事件从已插入状态队列中删除
	if (ev->ev_flags & EVLIST_INSERTED) {
		event_queue_remove_inserted(base, ev);
		// 如果事件是IO事件，则将事件从IO映射中删除
		if (ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED))
			res = evmap_io_del_(base, ev->ev_fd, ev);
		// 如果是信号事件，则将信号从信号映射删除
		else
			res = evmap_signal_del_(base, (int)ev->ev_fd, ev);
		// 如果删除正确，则需要通知主线程
		if (res == 1) {
			/* evmap says we need to notify the main thread. */
			notify = 1;
			res = 0;
		}
		/* If we do not have events, let's notify event base so it can
		 * exit without waiting */
		if (!event_haveevents(base) && !N_ACTIVE_CALLBACKS(base))
			notify = 1;
	}

	/* if we are not in the right thread, we need to wake up the loop */
	if (res != -1 && notify && EVBASE_NEED_NOTIFY(base))
		evthread_notify_base(base);

	event_debug_note_del_(ev);

	/* If the main thread is currently executing this event's callback,
	 * and we are not the main thread, then we want to wait until the
	 * callback is done before returning. That way, when this function
	 * returns, it will be safe to free the user-supplied argument.
	 */
#ifndef EVENT__DISABLE_THREAD_SUPPORT
	if (blocking != EVENT_DEL_NOBLOCK &&
	    base->current_event == event_to_event_callback(ev) &&
	    !EVBASE_IN_THREAD(base) &&
	    (blocking == EVENT_DEL_BLOCK || !(ev->ev_events & EV_FINALIZE))) {
		++base->current_event_waiters;
		EVTHREAD_COND_WAIT(base->current_event_cond, base->th_base_lock);
	}
#endif

	return (res);
}
```

#### 2.1.1.event_queue_remove_timeout函数

位于event.c，代码如下：

```cpp
static void
event_queue_remove_timeout(struct event_base *base, struct event *ev)
{
	EVENT_BASE_ASSERT_LOCKED(base);
	if (EVUTIL_FAILURE_CHECK(!(ev->ev_flags & EVLIST_TIMEOUT))) {
		event_errx(1, "%s: %p(fd "EV_SOCK_FMT") not on queue %x", __func__,
		    ev, EV_SOCK_ARG(ev->ev_fd), EVLIST_TIMEOUT);
		return;
	}
	
	// base中事件数量为-1，同时设置事件状态为非超时状态。	
	DECR_EVENT_COUNT(base, ev->ev_flags);
	ev->ev_flags &= ~EVLIST_TIMEOUT;

	// 如果使用了公用超时队列，则找到待删除事件所在公用超时队列，
	// 然后删除公用超时队列
	// 如果使用最小堆，则将事件从最小堆删除即可。
	if (is_common_timeout(&ev->ev_timeout, base)) {
		struct common_timeout_list *ctl =
		    get_common_timeout_list(base, &ev->ev_timeout);
		TAILQ_REMOVE(&ctl->events, ev,
		    ev_timeout_pos.ev_next_with_common_timeout);
	} else {
		min_heap_erase_(&base->timeheap, ev);
	}
}
```


#### 2.1.2.event_queue_remove_active函数

位于event.c，代码如下：

从队列中删除激活事件，实际上是从base维护的激活事件列表中删除事件的回调函数。

```cpp
static void
event_queue_remove_active(struct event_base *base, struct event_callback *evcb)
{
	EVENT_BASE_ASSERT_LOCKED(base);
	if (EVUTIL_FAILURE_CHECK(!(evcb->evcb_flags & EVLIST_ACTIVE))) {
		event_errx(1, "%s: %p not on queue %x", __func__,
			   evcb, EVLIST_ACTIVE);
		return;
	}

	// base中事件个数减一，同时设置事件回调函数状态为非激活状态
	// 同时base中激活事件个数减一
	DECR_EVENT_COUNT(base, evcb->evcb_flags);
	evcb->evcb_flags &= ~EVLIST_ACTIVE;
	base->event_count_active--;

	// 将base中激活队列中该事件的回调函数删除即可。
	TAILQ_REMOVE(&base->activequeues[evcb->evcb_pri],
	    evcb, evcb_active_next);
}
```

#### 2.1.3.event_queue_remove_active_later函数

实际上是从base维护的下一次激活列表中删除事件的回调函数。位于event.c，代码如下：

```cpp
static void
event_queue_remove_active_later(struct event_base *base, struct event_callback *evcb)
{
	EVENT_BASE_ASSERT_LOCKED(base);
	if (EVUTIL_FAILURE_CHECK(!(evcb->evcb_flags & EVLIST_ACTIVE_LATER))) {
		event_errx(1, "%s: %p not on queue %x", __func__,
			   evcb, EVLIST_ACTIVE_LATER);
		return;
	}

	// base中事件个数－1，同时事件回调函数状态设置为非下一次激活状态
	// 同时base中激活事件个数－1
	DECR_EVENT_COUNT(base, evcb->evcb_flags);
	evcb->evcb_flags &= ~EVLIST_ACTIVE_LATER;
	base->event_count_active--;

	// 将base中下一次激活队列中删除该事件的回调函数即可
	TAILQ_REMOVE(&base->active_later_queue, evcb, evcb_active_next);
}
```

#### 2.1.4.evmap_io_del_函数

从IO事件与文件描述符的映射中删除事件，位于evmap.c，代码如下：

```cpp
/* return -1 on error, 0 on success if nothing changed in the event backend,
 * and 1 on success if something did. */
// 删除IO事件：并告诉潜在的eventops，输入的fd状态已经改变了；
// 函数参数介绍：
// base：libevent句柄
// fd：和事件相关的文件描述符
// ev：需要移除的事件
int
evmap_io_del_(struct event_base *base, evutil_socket_t fd, struct event *ev)
{
	const struct eventop *evsel = base->evsel;
	struct event_io_map *io = &base->io;
	struct evmap_io *ctx;
	int nread, nwrite, nclose, retval = 0;
	short res = 0, old = 0;

	if (fd < 0)
		return 0;

	EVUTIL_ASSERT(fd == ev->ev_fd);

#ifndef EVMAP_USE_HT
	if (fd >= io->nentries)
		return (-1);
#endif

	GET_IO_SLOT(ctx, io, fd, evmap_io);

	// 获取当前fd上注册的读、写、关闭事件个数
	nread = ctx->nread;
	nwrite = ctx->nwrite;
	nclose = ctx->nclose;

	// 使用old变量保存原有的事件类型
	if (nread)
		old |= EV_READ;
	if (nwrite)
		old |= EV_WRITE;
	if (nclose)
		old |= EV_CLOSED;

	// 使用res保存删除的事件类型
	if (ev->ev_events & EV_READ) {
		if (--nread == 0)
			res |= EV_READ;
		EVUTIL_ASSERT(nread >= 0);
	}
	if (ev->ev_events & EV_WRITE) {
		if (--nwrite == 0)
			res |= EV_WRITE;
		EVUTIL_ASSERT(nwrite >= 0);
	}
	if (ev->ev_events & EV_CLOSED) {
		if (--nclose == 0)
			res |= EV_CLOSED;
		EVUTIL_ASSERT(nclose >= 0);
	}

	// 如果删除的事件不为空，则调用evsel->del删除事件（默认情况下调用epoll_del）
	if (res) {
		void *extra = ((char*)ctx) + sizeof(struct evmap_io);
		if (evsel->del(base, ev->ev_fd, old, res, extra) == -1) {
			retval = -1;
		} else {
			retval = 1;
		}
	}

	ctx->nread = nread;
	ctx->nwrite = nwrite;
	ctx->nclose = nclose;
	LIST_REMOVE(ev, ev_io_next);

	return (retval);
}
```


#### 2.1.5.evmap_signal_del_函数

从信号事件与信号的映射表中删除事件函数，位于evmap.c。代码如下：

```cpp
int
evmap_signal_del_(struct event_base *base, int sig, struct event *ev)
{
	const struct eventop *evsel = base->evsigsel;
	struct event_signal_map *map = &base->sigmap;
	struct evmap_signal *ctx;

	if (sig >= map->nentries)
		return (-1);

	// 根据sig获取相关ctx
	GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);

	// 删除事件
	LIST_REMOVE(ev, ev_signal_next);

	// 如果信号上注册的事件为空，则需要从信号的后台处理方法上删除
    //  实际上调用的是evsig_del函数，这是在evsig_init函数中指定的
	if (LIST_FIRST(&ctx->events) == NULL) {
		if (evsel->del(base, ev->ev_fd, 0, EV_SIGNAL, NULL) == -1)
			return (-1);
	}

	return (1);
}
```
