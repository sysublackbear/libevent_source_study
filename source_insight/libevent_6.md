# libevent(6)
@(源码)


##### 1.36.3.2.evmap_io_active_函数

位于evmap.c，代码如下：

根据fd和事件类型之间的映射表，找出所有该类型事件，全部激活，将注册到指定fd上的特定事件类型的事件插入激活队列中。

```cpp
// evmap_io_active_(base, events[i].data.fd, ev | EV_ET);
void
evmap_io_active_(struct event_base *base, evutil_socket_t fd, short events)
{
	struct event_io_map *io = &base->io;
	struct evmap_io *ctx;
	struct event *ev;

#ifndef EVMAP_USE_HT
	if (fd < 0 || fd >= io->nentries)
		return;
#endif
	// 根据文件描述符获取该fd上注册的所有事件列表
	// fd根据io类型在evmap_io上找到了对应的事件列表，挂在ctx下面。
	// 如何添加事件，详见1.32.3.1
	GET_IO_SLOT(ctx, io, fd, evmap_io);

	if (NULL == ctx)
		return;
	LIST_FOREACH(ev, &ctx->events, ev_io_next) {
		if (ev->ev_events & events)
			// 下面会详解
			event_active_nolock_(ev, ev->ev_events & events, 1);
	}
}
```

其中，`event_active_nolock_`函数的定义位于：event.c，代码如下：

该函数主要用于激活指定类型的事件

```cpp
// event_active_nolock_(ev, ev->ev_events & events, 1);
// 参数解析：
// ev：激活的事件
// res：事件的类型
// ncalls：该事件需要激活的次数，主要是应对信号事件，某些信号事件可能会注册多次，所以需要激活多次。
void
event_active_nolock_(struct event *ev, int res, short ncalls)
{
	struct event_base *base;

	event_debug(("event_active: %p (fd "EV_SOCK_FMT"), res %d, callback %p",
		ev, EV_SOCK_ARG(ev->ev_fd), (int)res, ev->ev_callback));

	base = ev->ev_base;
	EVENT_BASE_ASSERT_LOCKED(base);

	if (ev->ev_flags & EVLIST_FINALIZING) {
		/* XXXX debug */
		return;
	}

	// 根据激活的事件类型
	switch ((ev->ev_flags & (EVLIST_ACTIVE|EVLIST_ACTIVE_LATER))) {
	default:
	case EVLIST_ACTIVE|EVLIST_ACTIVE_LATER:
		EVUTIL_ASSERT(0);
		break;
	case EVLIST_ACTIVE:
		/* We get different kinds of events, add them together */
		ev->ev_res |= res;
		return;
	case EVLIST_ACTIVE_LATER:
		ev->ev_res |= res;
		break;
	case 0:
		ev->ev_res = res;
		break;
	}

	if (ev->ev_pri < base->event_running_priority)
		base->event_continue = 1;

	// 如果是信号事件，则需要设置调用次数
	if (ev->ev_events & EV_SIGNAL) {
#ifndef EVENT__DISABLE_THREAD_SUPPORT
		if (base->current_event == event_to_event_callback(ev) &&
		    !EVBASE_IN_THREAD(base)) {
			++base->current_event_waiters;
			EVTHREAD_COND_WAIT(base->current_event_cond, base->th_base_lock);
		}
#endif
		ev->ev_ncalls = ncalls;
		ev->ev_pncalls = NULL;
	}

	// 激活事件实际上就是将事件的回调函数放入激活队列
	event_callback_activate_nolock_(base, event_to_event_callback(ev));
}

static inline struct event_callback *
event_to_event_callback(struct event *ev)
{
	return &ev->ev_evcallback;
}
```

其中，`event_callback_activate_nolock_`函数位于event.c，代码如下：

激活事件的回调函数。

```cpp
// event_callback_activate_nolock_(base, event_to_event_callback(ev));
int
event_callback_activate_nolock_(struct event_base *base,
    struct event_callback *evcb)
{
	int r = 1;

	// 合法状态检查
	if (evcb->evcb_flags & EVLIST_FINALIZING)
		return 0;

	switch (evcb->evcb_flags & (EVLIST_ACTIVE|EVLIST_ACTIVE_LATER)) {
	default:
		EVUTIL_ASSERT(0);
		EVUTIL_FALLTHROUGH;
	case EVLIST_ACTIVE_LATER:
		// 原本的事件状态为EVLIST_ACTIVE_LATER，但是在本次就需要激活，因此需要先从下一次激活队列中移除
		// 然后再加入到激活队列里面
		event_queue_remove_active_later(base, evcb);
		r = 0;
		break;
	case EVLIST_ACTIVE:  // 已经在激活队列里面了，直接返回
		return 0;
	case 0:
		break;
	}

	// 将回调函数插入到激活队列
	// 下面会详细介绍
	event_queue_insert_active(base, evcb);
	
	// 通知主线程loop
	if (EVBASE_NEED_NOTIFY(base))
		evthread_notify_base(base);

	return r;
}
```

其中，`event_queue_insert_active`函数，位于event.c，代码如下：

将事件插入到激活队列
```cpp
// event_queue_insert_active(base, evcb);
static void
event_queue_insert_active(struct event_base *base, struct event_callback *evcb)
{
	EVENT_BASE_ASSERT_LOCKED(base);

	if (evcb->evcb_flags & EVLIST_ACTIVE) {
		/* Double insertion is possible for active events */
		return;
	}

	// 增加base中的事件数量，并设置回调函数的执行状态为激活状态
	/*
	#define INCR_EVENT_COUNT(base,flags) do {					\
	((base)->event_count += !((flags) & EVLIST_INTERNAL));			\
	MAX_EVENT_COUNT((base)->event_count_max, (base)->event_count);		\
} while (0)
	*/
	INCR_EVENT_COUNT(base, evcb->evcb_flags);

	evcb->evcb_flags |= EVLIST_ACTIVE;

	// 激活事件数量增加，并将激活事件回调函数插入所在优先级队列中
	base->event_count_active++;
	MAX_EVENT_COUNT(base->event_count_active_max, base->event_count_active);
	EVUTIL_ASSERT(evcb->evcb_pri < base->nactivequeues);
	TAILQ_INSERT_TAIL(&base->activequeues[evcb->evcb_pri],
	    evcb, evcb_active_next);
}
```



#### 1.36.4.timeout_process函数

将超时事件放入激活队列，位于event.c，代码如下：

```cpp
/* Activate every event whose timeout has elapsed. */
// timeout_process(base);
static void
timeout_process(struct event_base *base)
{
	/* Caller must hold lock. */
	struct timeval now;
	struct event *ev;

	// 如果超时最小堆为空，则没有超时事件，直接返回即可。
	// int min_heap_empty_(min_heap_t* s) { return 0u == s->n; }
	if (min_heap_empty_(&base->timeheap)) {
		return;
	}

	// 获取当前base中的时间，用于下面比较超时事件是否超时
	gettime(base, &now);

	// 遍历最小堆，如果最小堆中超时事件没有超时的，则退出循环；
	// 如果有超时的事件，则先将超时事件移除，然后激活超时事件；
	// 先将超时事件移除的原因是：事件已超时，所以需要激活，就不需要继续监控了；
	// 还有就是对于永久性监控的事件，可以在执行事件时重新加入到监控队列中，同时可以重新配置超时时间
	while ((ev = min_heap_top_(&base->timeheap))) {
		// 连最接近的超时的事件目前还没超时，说明后面的事件更加不会超时，直接退出循环
		if (evutil_timercmp(&ev->ev_timeout, &now, >))
			break;

		/* delete this event from the I/O queues */
		// 将超时事件移除
		// TODO:具体实现后面要看
		event_del_nolock_(ev, EVENT_DEL_NOBLOCK);

		event_debug(("timeout_process: event: %p, call %p",
			 ev, ev->ev_callback));
		// 激活超时事件
		event_active_nolock_(ev, EV_TIMEOUT, 1);
	}
}
```


#### 1.36.5.event_process_active函数

处理激活的事件逻辑。

```cpp
//if (N_ACTIVE_CALLBACKS(base)) {
//	int n = event_process_active(base);
/*
 * Active events are stored in priority queues.  Lower priorities are always
 * process before higher priorities.  Low priority events can starve high
 * priority ones.
 */
// 激活存储在优先级队列中的事件；表示优先级的数字越小，优先级越高；
// 优先级高的事件可能会一直阻塞低优先级事件运行。
static int
event_process_active(struct event_base *base)
{
	/* Caller must hold th_base_lock */
	struct evcallback_list *activeq = NULL;
	int i, c = 0;
	const struct timeval *endtime;
	struct timeval tv;

	// 获取重新检查新事件产生之前，可以处理的回调函数的最大个数；
	// 优先级低于limit_callbacks_after_prio的事件执行时，才会检查新事件，否则不检查。
	const int maxcb = base->max_dispatch_callbacks;
	const int limit_after_prio = base->limit_callbacks_after_prio;

	// 如果检查新事件间隔时间大于或者等于0，为了检查间隔更为精确，则需要更新base中的时间，并获取base中当前的时间
	// 然后下一次检查的时间为base->max_dispatch_time + tv
	if (base->max_dispatch_time.tv_sec >= 0) {
		update_time_cache(base);
		gettime(base, &tv);
		evutil_timeradd(&base->max_dispatch_time, &tv, &tv);
		endtime = &tv;
	} else {
		endtime = NULL;
	}

	// 如果base中激活队列不为空，则根据优先级遍历激活队列；
	// 遍历每一个优先级子队列，处理子队列中回调函数；
	// 在执行时，需要根据limit_after_prio设置两次检查新事件之间的间隔；
	// 如果优先级<limit_after_prio，即当事件优先级高于limit_after_prio时，是不需要查看新事件的；
	// 如果优先级>=limit_after_prio，即当事件优先级不高于limit_after_prio时，是需要根据maxcb以及end_time检查新事件。
	for (i = 0; i < base->nactivequeues; ++i) {
		if (TAILQ_FIRST(&base->activequeues[i]) != NULL) {
			base->event_running_priority = i;
			activeq = &base->activequeues[i];
			if (i < limit_after_prio)
				// 下面有详解
				c = event_process_active_single_queue(base, activeq,
				    INT_MAX, NULL);
			else
				c = event_process_active_single_queue(base, activeq,
				    maxcb, endtime);
			if (c < 0) {
				goto done;  // 遇到event_break，退出循环
			} else if (c > 0)
				break; /* Processed a real event; do not
					* consider lower-priority events */
			/* If we get here, all of the events we processed
			 * were internal.  Continue. */
		}
	}

done:
	base->event_running_priority = -1;

	return c;
}
```

##### 1.36.5.1.event_process_active_single_queue函数

位于event.c，代码如下：

```cpp
/*
  Helper for event_process_active to process all the events in a single queue,
  releasing the lock as we go.  This function requires that the lock be held
  when it's invoked.  Returns -1 if we get a signal or an event_break that
  means we should stop processing any active events now.  Otherwise returns
  the number of non-internal event_callbacks that we processed.
*/
// event_process_active函数的帮助实现函数：用于处理单个优先级队列中的所有事件；
// 当执行完毕时，需要释放锁：这个函数要求加锁；如果获得一个信号或者event_break时，
// 意味着需要停止执行任何活跃事件，则返回-1；否则返回已经处理的非内部event_callbacks的个数
// 函数参数介绍：
// base:event_base句柄；
// activeq：(activeq = &base->activequeues[i];)激活回调函数队列
// max_to_process：检查新事件之间执行的回调函数的最大个数
// endtime：检查新事件的时间间隔
// c = event_process_active_single_queue(base, activeq, INT_MAX, NULL);
// c = event_process_active_single_queue(base, activeq, maxcb, endtime);
static int
event_process_active_single_queue(struct event_base *base,
    struct evcallback_list *activeq,
    int max_to_process, const struct timeval *endtime)
{
	struct event_callback *evcb;
	int count = 0;

	EVUTIL_ASSERT(activeq != NULL);
	// 遍历子队列
	for (evcb = TAILQ_FIRST(activeq); evcb; evcb = TAILQ_FIRST(activeq)) {
		struct event *ev=NULL;
		// 如果回调函数状态为初始化状态
		if (evcb->evcb_flags & EVLIST_INIT) {
			ev = event_callback_to_event(evcb);

			if (ev->ev_events & EV_PERSIST || ev->ev_flags & EVLIST_FINALIZING)
				// 如果回调函数对应的事件为永久事件，或者事件状态为结束，则将事件回调函数从激活队列中移除。
				event_queue_remove_active(base, evcb);
			else
				// 否则，需要删除事件。
				event_del_nolock_(ev, EVENT_DEL_NOBLOCK);
			event_debug((
			    "event_process_active: event: %p, %s%s%scall %p",
			    ev,
			    ev->ev_res & EV_READ ? "EV_READ " : " ",
			    ev->ev_res & EV_WRITE ? "EV_WRITE " : " ",
			    ev->ev_res & EV_CLOSED ? "EV_CLOSED " : " ",
			    ev->ev_callback));
		} else {
			// 如果是其他状态，则将回调函数从激活队列中移除
			event_queue_remove_active(base, evcb);
			event_debug(("event_process_active: event_callback %p, "
				"closure %d, call %p",
				evcb, evcb->evcb_closure, evcb->evcb_cb_union.evcb_callback));
		}

		// 如果回调函数状态为非内部状态，则执行的事件个数+1
		if (!(evcb->evcb_flags & EVLIST_INTERNAL))
			++count;

		// 设置当前的事件
		base->current_event = evcb;
#ifndef EVENT__DISABLE_THREAD_SUPPORT
		base->current_event_waiters = 0;
#endif

		// 根据事件回调函数模式，执行不同的回调函数
		switch (evcb->evcb_closure) {
		case EV_CLOSURE_EVENT_SIGNAL:  // 信号事件
			EVUTIL_ASSERT(ev != NULL);
			event_signal_closure(base, ev);
			break;
		case EV_CLOSURE_EVENT_PERSIST:  // 永久性的非信号事件
			EVUTIL_ASSERT(ev != NULL);
			event_persist_closure(base, ev);
			break;
		case EV_CLOSURE_EVENT: {  // 常规的读写事件
			void (*evcb_callback)(evutil_socket_t, short, void *);
			short res;
			EVUTIL_ASSERT(ev != NULL);
			// 
			evcb_callback = *ev->ev_callback;
			res = ev->ev_res;
			EVBASE_RELEASE_LOCK(base, th_base_lock);
			evcb_callback(ev->ev_fd, res, ev->ev_arg);
		}
		break;
		case EV_CLOSURE_CB_SELF: {  // 简单的回调，使用evcb_selfcb回调
			void (*evcb_selfcb)(struct event_callback *, void *) = evcb->evcb_cb_union.evcb_selfcb;
			EVBASE_RELEASE_LOCK(base, th_base_lock);
			evcb_selfcb(evcb, evcb->evcb_arg);
		}
		break;
		case EV_CLOSURE_EVENT_FINALIZE:  // 结束事件
		case EV_CLOSURE_EVENT_FINALIZE_FREE: {  // 正在结束的事件后面要释放
			void (*evcb_evfinalize)(struct event *, void *);
			int evcb_closure = evcb->evcb_closure;
			EVUTIL_ASSERT(ev != NULL);
			base->current_event = NULL;
			evcb_evfinalize = ev->ev_evcallback.evcb_cb_union.evcb_evfinalize;
			EVUTIL_ASSERT((evcb->evcb_flags & EVLIST_FINALIZING));
			EVBASE_RELEASE_LOCK(base, th_base_lock);
			evcb_evfinalize(ev, ev->ev_arg);
			event_debug_note_teardown_(ev);
			if (evcb_closure == EV_CLOSURE_EVENT_FINALIZE_FREE)
				mm_free(ev);
		}
		break;
		case EV_CLOSURE_CB_FINALIZE: {  // 结束型回调
			void (*evcb_cbfinalize)(struct event_callback *, void *) = evcb->evcb_cb_union.evcb_cbfinalize;
			base->current_event = NULL;
			EVUTIL_ASSERT((evcb->evcb_flags & EVLIST_FINALIZING));
			EVBASE_RELEASE_LOCK(base, th_base_lock);
			evcb_cbfinalize(evcb, evcb->evcb_arg);
		}
		break;
		default:
			EVUTIL_ASSERT(0);
		}

		EVBASE_ACQUIRE_LOCK(base, th_base_lock);
		base->current_event = NULL;
#ifndef EVENT__DISABLE_THREAD_SUPPORT
		if (base->current_event_waiters) {
			base->current_event_waiters = 0;
			EVTHREAD_COND_BROADCAST(base->current_event_cond);
		}
#endif

		// 如果中止标志位设置，则中断执行
		if (base->event_break)
			return -1;
		// 如果当前执行的事件个数大于检查新事件之间执行回调函数个数，则需要返回函数，继续检查新事件。
		if (count >= max_to_process)
			return count;
		// 如果执行的事件个数不为0，且截至时间不为空，则需要判定当前时间是否已经超过了截至时间，如果超过了，则退出执行；
		if (count && endtime) {
			struct timeval now;
			update_time_cache(base);
			gettime(base, &now);
			if (evutil_timercmp(&now, endtime, >=))
				return count;
		}
		if (base->event_continue)
			break;
	}
	return count;
}
```

其中，`event_signal_closure`函数用于处理激活的信号事件时的调用函数。位于event.c，代码如下：

```cpp
/* "closure" function called when processing active signal events */
// event_signal_closure(base, ev);
static inline void
event_signal_closure(struct event_base *base, struct event *ev)
{
	short ncalls;
	int should_break;

	/* Allows deletes to work */
	ncalls = ev->ev_ncalls;  // 调用回调函数的次数
	if (ncalls != 0)
		ev->ev_pncalls = &ncalls;
	EVBASE_RELEASE_LOCK(base, th_base_lock);
	// 根据事件调用次数循环调用信号事件回调函数
	while (ncalls) {
		ncalls--;
		ev->ev_ncalls = ncalls;  // 指向ev_ncalls
		if (ncalls == 0)
			ev->ev_pncalls = NULL;
		// 实际调用注册在event上面的回调函数
		(*ev->ev_callback)(ev->ev_fd, ev->ev_res, ev->ev_arg);

		EVBASE_ACQUIRE_LOCK(base, th_base_lock);
		should_break = base->event_break;  
		EVBASE_RELEASE_LOCK(base, th_base_lock);
		// 遇到event_break，直接退出
		if (should_break) {
			if (ncalls != 0)
				ev->ev_pncalls = NULL;
			return;
		}
	}
}
```

而，`event_persist_closure`函数位于event.c，用于处理永久事件的回调函数，代码如下：

```cpp
/* Closure function invoked when we're activating a persistent event. */
static inline void
event_persist_closure(struct event_base *base, struct event *ev)
{
	void (*evcb_callback)(evutil_socket_t, short, void *);

        // Other fields of *ev that must be stored before executing
        evutil_socket_t evcb_fd;
        short evcb_res;
        void *evcb_arg;

	/* reschedule the persistent event if we have a timeout. */
	// 如果设置了超时时间
	if (ev->ev_io_timeout.tv_sec || ev->ev_io_timeout.tv_usec) {
		/* If there was a timeout, we want it to run at an interval of
		 * ev_io_timeout after the last time it was _scheduled_ for,
		 * not ev_io_timeout after _now_.  If it fired for another
		 * reason, though, the timeout ought to start ticking _now_. */
		// 如果有超时事件，并且想要它在上一次超时时间点ev_io_timeout间隔之后运行，而不是从现在的时间点之后的ev_io_timeout。如果不是超时时间，则需要从现在的时间点开始；
		struct timeval run_at, relative_to, delay, now;
		ev_uint32_t usec_mask = 0;
		/*
		// True iff tv1 and tv2 have the same common-timeout index, or if neither
  one is a common timeout.
static inline int
is_same_common_timeout(const struct timeval *tv1, const struct timeval *tv2)
{
	return (tv1->tv_usec & ~MICROSECONDS_MASK) ==
	    (tv2->tv_usec & ~MICROSECONDS_MASK);
}
		*/
		// ev_timeout：定时事件的超时时间
		// ev_io_timeout：永久事件的定时器，每次超时结束后继续通过该时间设置定时器
		// 如果有超时事件，并且想要它在上一次超时时间点ev_io_timeout间隔之后运行，
		// 而不是从现在的时间点之后的ev_io_timeout。
		EVUTIL_ASSERT(is_same_common_timeout(&ev->ev_timeout,
			&ev->ev_io_timeout));
		gettime(base, &now);

		// 如果使用公用超时队列，则需要重新调整掩码；
		// 如果不使用公用超时队列，则需要根据是否为超时事件来决定下一次的超时时间从哪个时间点开始算起；
		if (is_common_timeout(&ev->ev_timeout, base)) {
			delay = ev->ev_io_timeout;
			usec_mask = delay.tv_usec & ~MICROSECONDS_MASK;
			delay.tv_usec &= MICROSECONDS_MASK;
			if (ev->ev_res & EV_TIMEOUT) {
				relative_to = ev->ev_timeout;
				relative_to.tv_usec &= MICROSECONDS_MASK;
			} else {
				relative_to = now;
			}
		} else {
			delay = ev->ev_io_timeout;
			if (ev->ev_res & EV_TIMEOUT) {
				relative_to = ev->ev_timeout;
			} else {
				relative_to = now;
			}
		}
		// 算出准确的超时时间
		// run_at = relative_to + delay
		evutil_timeradd(&relative_to, &delay, &run_at);

		// 并比较超时时间和当前时间；如果超时，则将事件重新添加到队列中
		if (evutil_timercmp(&run_at, &now, <)) {
			/* Looks like we missed at least one invocation due to
			 * a clock jump, not running the event loop for a
			 * while, really slow callbacks, or
			 * something. Reschedule relative to now.
			 */
			evutil_timeradd(&now, &delay, &run_at);
		}
		run_at.tv_usec |= usec_mask;
		// 添加事件到队列
		event_add_nolock_(ev, &run_at, 1);
	}

	// Save our callback before we release the lock
	evcb_callback = ev->ev_callback;
        evcb_fd = ev->ev_fd;
        evcb_res = ev->ev_res;
        evcb_arg = ev->ev_arg;

	// Release the lock
 	EVBASE_RELEASE_LOCK(base, th_base_lock);

	// Execute the callback
        (evcb_callback)(evcb_fd, evcb_res, evcb_arg);
}
```
