# libevent(7)
@(源码)


### 1.37.signal_cb函数

位于原demo的`signal_event = evsignal_new(base, SIGINT, signal_cb, (void *)base);`，位于hello-world.c，代码如下：

```cpp
static void
signal_cb(evutil_socket_t sig, short events, void *user_data)
{
	struct event_base *base = user_data;
	struct timeval delay = { 2, 0 };  // 延迟2秒

	printf("Caught an interrupt signal; exiting cleanly in two seconds.\n");

	// 遇到CTRL-C，中断event_base_loop
	event_base_loopexit(base, &delay);
}
```

#### 1.37.1.event_base_loopexit函数

位于event.c，代码如下：

```cpp
// event_base_loopexit(base, &delay);
int
event_base_loopexit(struct event_base *event_base, const struct timeval *tv)
{
	return (event_base_once(event_base, -1, EV_TIMEOUT, event_loopexit_cb,
		    event_base, tv));
}
```

本质调用的是`event_base_once`，位于event.c，代码如下：

```cpp
// event_base_once(event_base, -1, EV_TIMEOUT, event_loopexit_cb, event_base, tv)
/* Schedules an event once */
int
event_base_once(struct event_base *base, evutil_socket_t fd, short events,
    void (*callback)(evutil_socket_t, short, void *),
    void *arg, const struct timeval *tv)
{
	// event_once：只触发一次的事件
	/*
	// Sets up an event for processing once
	struct event_once {
		LIST_ENTRY(event_once) next_once;
		struct event ev;

		void (*cb)(evutil_socket_t, short, void *);
		void *arg;
	};
	*/
	struct event_once *eonce;
	int res = 0;
	int activate = 0;

	/* We cannot support signals that just fire once, or persistent
	 * events. */
	// 如果是信号事件(EV_SIGNAL)和持久化事件(EV_PERSIST)的话，属于异常，因为：
	// EV_SIGNAL：信号事件，有可能会触发多次，不可能保证能只触发一次；
	// EV_PERSIST：持久化事件，触发完事件还会存活，有可能重复激活。
	if (events & (EV_SIGNAL|EV_PERSIST))
		return (-1);

	if ((eonce = mm_calloc(1, sizeof(struct event_once))) == NULL)
		return (-1);

	eonce->cb = callback;
	eonce->arg = arg;

	if ((events & (EV_TIMEOUT|EV_SIGNAL|EV_READ|EV_WRITE|EV_CLOSED)) == EV_TIMEOUT) {
		evtimer_assign(&eonce->ev, base, event_once_cb, eonce);
		// #define evtimer_assign(ev, b, cb, arg) event_assign((ev), (b), -1, 0, (cb), (arg))
		// 这里上面fd为-1,events为0代表不监听任何事件，仅仅作为定时器作用。

		if (tv == NULL || ! evutil_timerisset(tv)) {
			/* If the event is going to become active immediately,
			 * don't put it on the timeout queue.  This is one
			 * idiom for scheduling a callback, so let's make
			 * it fast (and order-preserving). */
			// 已经超时了，早就应该触发了，不需要放入timeout队列
			activate = 1;
		}
	} else if (events & (EV_READ|EV_WRITE|EV_CLOSED)) {
		events &= EV_READ|EV_WRITE|EV_CLOSED;

		event_assign(&eonce->ev, base, fd, events, event_once_cb, eonce);
	} else {
		/* Bad event combination */
		mm_free(eonce);
		return (-1);
	}

	if (res == 0) {
		EVBASE_ACQUIRE_LOCK(base, th_base_lock);
		if (activate)
			// 加入到激活队列
			event_active_nolock_(&eonce->ev, EV_TIMEOUT, 1);  // 详见1.36.3.2
		else
			// 加入到timeout队列
			res = event_add_nolock_(&eonce->ev, tv, 0);

		if (res != 0) {
			mm_free(eonce);
			return (res);
		} else {
			LIST_INSERT_HEAD(&base->once_events, eonce, next_once);
		}
		EVBASE_RELEASE_LOCK(base, th_base_lock);
	}

	return (0);
}
```

##### 1.37.1.1.event_loopexit_cb函数

位于event.c，代码如下：

```cpp
static void
event_loopexit_cb(evutil_socket_t fd, short what, void *arg)
{
	struct event_base *base = arg;
	base->event_gotterm = 1;  // 在event_base_loop通过event_gotterm来终止循环
}
```


### 1.38.listener_cb函数

位于原demo的`listener = evconnlistener_new_bind(base, listener_cb, (void *)base, LEV_OPT_REUSEABLE|LEV_OPT_CLOSE_ON_FREE, -1, (struct sockaddr*)&sin, sizeof(sin));`，hello-world.c，代码如下：

`event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST, listener_read_cb, lev);`

```cpp
// cb(lev, new_fd, (struct sockaddr*)&ss, (int)socklen, user_data);
static void
listener_cb(struct evconnlistener *listener, evutil_socket_t fd,
    struct sockaddr *sa, int socklen, void *user_data)
{
	struct event_base *base = user_data;
	struct bufferevent *bev;

	// bufferevent内置了两个event（读/写）和对应的缓冲区。当有数据被读入(input)的时候，readcb被调用，当output被输出完成的时候，writecb被调用，当网络I/O出现错误，如链接中断，超时或者其他错误时，errorcb被调用。
	bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
	if (!bev) {
		fprintf(stderr, "Error constructing bufferevent!");
		event_base_loopbreak(base);
		return;
	}

	// 设置读写对应的回调函数
	// readcb = NULL
	// writecb = conn_writecb
	// errorcb = conn_eventcb
	// 进入bufferevent_setcb回调函数：
	// 在readcb里面从input中读取数据，处理完毕后填充到output中；
	// writecb对于服务端程序，只需要readcb就可以了，可以置为NULL；
	// errorcb用于处理一些错误信息。
	bufferevent_setcb(bev, NULL, conn_writecb, conn_eventcb, NULL);
	
	// 启用读写事件，其实是调用了event_add将相应读写事件加入事件监听队列poll。正如文档所说，如果相应事件不置为true，bufferevent是不会读写数据的
	// 这里设置可写不可读
	bufferevent_enable(bev, EV_WRITE);
	bufferevent_disable(bev, EV_READ);

	// 一直往bufferevent写入MESSAGE即，"Hello World!\n"
	bufferevent_write(bev, MESSAGE, strlen(MESSAGE));
}
```

调用的细节位于1.4.1。在看这个函数之前，我们先看`listener_read_cb`这个函数，位于listener.c，代码如下：

```cpp
static void
listener_read_cb(evutil_socket_t fd, short what, void *p)
{
	struct evconnlistener *lev = p;
	int err;
	evconnlistener_cb cb;
	evconnlistener_errorcb errorcb;
	void *user_data;
	LOCK(lev);
	while (1) {
		struct sockaddr_storage ss;
		ev_socklen_t socklen = sizeof(ss);
		// accept连接
		evutil_socket_t new_fd = evutil_accept4_(fd, (struct sockaddr*)&ss, &socklen, lev->accept4_flags);
		if (new_fd < 0)
			break;
		if (socklen == 0) {
			/* This can happen with some older linux kernels in
			 * response to nmap. */
			evutil_closesocket(new_fd);
			continue;
		}

		if (lev->cb == NULL) {
			evutil_closesocket(new_fd);
			UNLOCK(lev);
			return;
		}
		++lev->refcnt;
		cb = lev->cb;
		user_data = lev->user_data;
		UNLOCK(lev);
		// 回调函数
		cb(lev, new_fd, (struct sockaddr*)&ss, (int)socklen,
		    user_data);
		LOCK(lev);
		if (lev->refcnt == 1) {
			int freed = listener_decref_and_unlock(lev);
			EVUTIL_ASSERT(freed);
			return;
		}
		--lev->refcnt;
		if (!lev->enabled) {
			/* the callback could have disabled the listener */
			UNLOCK(lev);
			return;
		}
	}
	err = evutil_socket_geterror(fd);
	if (EVUTIL_ERR_ACCEPT_RETRIABLE(err)) {
		UNLOCK(lev);
		return;
	}
	if (lev->errorcb != NULL) {
		++lev->refcnt;
		errorcb = lev->errorcb;
		user_data = lev->user_data;
		UNLOCK(lev);
		errorcb(lev, user_data);
		LOCK(lev);
		listener_decref_and_unlock(lev);
	} else {
		event_sock_warn(fd, "Error from accept() call");
		UNLOCK(lev);
	}
}
```

我们可以看到，`listener_read_cb`本质上就是一直accept连接，然后进行回调。所谓的外部回调函数。而这里的cb函数，就是`listener_cb`函数。


#### 1.38.1.bufferevent_socket_new函数

位于bufferevent_sock.c，代码如下：

```cpp
// bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
struct bufferevent *
bufferevent_socket_new(struct event_base *base, evutil_socket_t fd,
    int options)
{
	struct bufferevent_private *bufev_p;
	struct bufferevent *bufev;

#ifdef _WIN32
	if (base && event_base_get_iocp_(base))
		return bufferevent_async_new_(base, fd, options);
#endif

	if ((bufev_p = mm_calloc(1, sizeof(struct bufferevent_private)))== NULL)
		return NULL;

	if (bufferevent_init_common_(bufev_p, base, &bufferevent_ops_socket,
				    options) < 0) {
		mm_free(bufev_p);
		return NULL;
	}
	bufev = &bufev_p->bev;
	evbuffer_set_flags(bufev->output, EVBUFFER_FLAG_DRAINS_TO_FD);

	event_assign(&bufev->ev_read, bufev->ev_base, fd,
	    EV_READ|EV_PERSIST|EV_FINALIZE, bufferevent_readcb, bufev);
	event_assign(&bufev->ev_write, bufev->ev_base, fd,
	    EV_WRITE|EV_PERSIST|EV_FINALIZE, bufferevent_writecb, bufev);

	evbuffer_add_cb(bufev->output, bufferevent_socket_outbuf_cb, bufev);

	evbuffer_freeze(bufev->input, 0);
	evbuffer_freeze(bufev->output, 1);

	return bufev;
}
```

本来想介绍的，发现有篇文章对`bufferevent`的介绍非常好，详见：https://blog.csdn.net/feitianxuxue/article/details/9386843


### 1.39.conn_writecb函数

位于hello-world.c，是`bufferevent`的`writecb`。代码如下：

```cpp
static void
conn_writecb(struct bufferevent *bev, void *user_data)
{
	// 其实就是取出bufferevent中的output
	struct evbuffer *output = bufferevent_get_output(bev);
	if (evbuffer_get_length(output) == 0) {
		printf("flushed answer\n");
		bufferevent_free(bev);  // output为0，代表已经没有可以写的空间了，回收bufferevent
	}
}
```

### 1.40.conn_eventcb函数

位于hello-world.c，是`bufferevent`的`errorcb`。代码如下：

```cpp
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
	// 不会再有其他事件发生，直接回收
	bufferevent_free(bev);
}
```

### 1.41.综述

hello-world.c这个例子，本质就是：新建一个`listerner`对象，连接9995端口，建立`socket`，然后一直往fd写入`"Hello World!\n"`，同时建立监听信号事件，对于SIGINT（比如用户键入CTRL-C），结束事件循环。这一切过程都是通过`event_base`进行调度，调度函数`event_base_dispatch`，里面本质执行`event_base_loop`。

下面图详解了`event_base`,以及一些关键操作的图解：


![7-1.png](https://github.com/sysublackbear/libevent_source_study/blob/master/libevent_pic/7-1.png)

![7-2.png](https://github.com/sysublackbear/libevent_source_study/blob/master/libevent_pic/7-2.png)

整个hello-world.c的执行顺序：

1. 初始化`event_config`；
2. 根据`event_config`来初始化`event_base`;
	2.1. 初始化信号通知管道`evsig_init_ `；
3. 建立`evconnlistener`对象，进行事件注册`event_assign`，其中利用到`bufferevent`结构。
4. 建立信号处理器，方便遇到`SIGINT`信号能够正常退出；
5. 陷入`event_base_dispatch`，进行事件调度。
	5.1. 获取超时时间间隔`timeout_next`；
	5.2. 将就绪队列的事件移动到激活队列；`event_queue_make_later_events_active`
	5.3. 调用`epoll_dispatch`接收可读可写的事件，加入到活跃队列里面；
	5.4. 从超时事件堆里面找出超时的事件，加入到活跃队列中。
	5.5. 统一处理活跃队列里面的事件，统一根据事件类型的不同，执行不同的回调函数。`event_process_active`


