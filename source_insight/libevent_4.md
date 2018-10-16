#libevent(4)
@(源码)


###1.19.设置event_base采用的工作模式——event_config_set_flag

位于event.c，代码如下：

+ 设置`event_base`的工作模式，需要在申请`event_config`之后运行，在配置`event_base`之前执行；
+ 可以设置多个工作模式同时存在，但是需要注意的是不是每种工作模式都是可以设置的；
+ 需要查看本地内核环境以及后台方法是否支持。

```cpp
int
event_config_set_flag(struct event_config *cfg, int flag)
{
	if (!cfg)
		return -1;
	cfg->flags |= flag;
	return 0;
}
```


###1.20.设置后台方法的工作方式——event_config_require_features

位于event.c，代码如下：

```cpp
// 设置后台方法特征，注意不是每个平台都会支持所有特征或者支持几个特征同时存在；
// 设置后台方法特征的代码应该在event_base_new_with_config之前进行；
// 注意，这里不是采用或的方式，而是直接替换为输入的方法特征。
int
event_config_require_features(struct event_config *cfg,
    int features)
{
	if (!cfg)
		return (-1);
	cfg->require_features = features;
	return (0);
}
```


###1.21.屏蔽某些后台的方法——event_config_avoid_method

位于event.c，代码如下：
```cpp
// 输入需要屏蔽的方法名字；可以用来避免某些不支持特定文件描述符类型的后台方法。
// 或者调试用来屏蔽某些特定事件的机制，应用可以使用多个event_base以适应不兼容的文件描述符类型。
int
event_config_avoid_method(struct event_config *cfg, const char *method)
{
	// 申请存储屏蔽后台方法名字的空间
	struct event_config_entry *entry = mm_malloc(sizeof(*entry));
	if (entry == NULL)
		return (-1);

	// 拷贝需要屏蔽的方法名字
	if ((entry->avoid_method = mm_strdup(method)) == NULL) {
		mm_free(entry);
		return (-1);
	}
	// 将屏蔽的方法插入到屏蔽的队列
	TAILQ_INSERT_TAIL(&cfg->entries, entry, next);

	return (0);
}
```

###1.22.设置cpu个数提示信息——event_config_set_num_cpus_hint

位于event.c，代码如下：

+ 记录有关系统cpu个数的提示；用来调整线程池，在2.0中，只能用于windows；
+ 而且只能当使用IOCP时才有效。

```cpp
int
event_config_set_num_cpus_hint(struct event_config *cfg, int cpus)
{
	if (!cfg)
		return (-1);
	cfg->n_cpus_hint = cpus;
	return (0);
}
```


###1.23.设置event_base调度的一些时间间隔信息——event_config_set_max_dispatch_interval

位于event.c，代码如下：

```cpp
// 记录event_base用于检查新事件的时间间隔或者回调函数的个数；
// 默认情况下，event_base在检查新事件之前，应当是有多少最高优先级的激活事件；
// 如果你通过max_interval设置了两次检查之间的时间间隔，它将在每次执行回调之后检查距离上一次检查新事件的时间间隔
// 是否超过了max_interval，在两次检查新事件之间不允许超过max_interval。
// 如果你通过配置max_callbacks>=0，则两次检查新事件之间不会执行超过max_callbacks个回调函数。

// min_priority这个选项可以降低高优先级事件的延迟，同时避免优先级颠倒执行，即多个低优先级事件
// 屏蔽了高优先级事件，即多个低优先级先发生，高优先级事件后发生，而event_base一直在执行低优先级事件，
// 而导致高优先级事件迟迟得不到执行，但是会轻微降低吞吐量，谨慎使用这个。

// cfg：event_config对象
// max_interval：event_base停止执行回调并检查新事件的时间间隔，如果不想设置这样的间隔，可以设置为NULL
// max_callbacks：event_base停止执行回调并检查新事件的已执行的最大回调函数个数，如果不需要，可以设置成-1
// min_priority：即低于这个值的优先级，就不应该强制执行max_interval和max_callbacks检查；
// 如果设置为0，则对每个优先级都执行这两个检查；如果设置为1，只有priority>=1时，才执行这样的检查。

int
event_config_set_max_dispatch_interval(struct event_config *cfg,
    const struct timeval *max_interval, int max_callbacks, int min_priority)
{
	// 如果max_interval不为空，则将输入的参数拷贝到cfg中，
	// 否则设置为非法值
	if (max_interval)
		memcpy(&cfg->max_dispatch_interval, max_interval,
		    sizeof(struct timeval));
	else
		cfg->max_dispatch_interval.tv_sec = -1;
	
	// 如果max_callbacks >= 0，则设置为max_callbacks，否则设置为INT_MAX
	cfg->max_dispatch_callbacks =
	    max_callbacks >= 0 ? max_callbacks : INT_MAX;
	// 如果<0，则所有优先级都执行检查，否则设置为传入参数
	if (min_priority < 0)
		min_priority = 0;
	cfg->limit_callbacks_after_prio = min_priority;
	return (0);
}
```


###1.24.配置获取（获取当前正在使用的后台方法）——event_base_get_method

位于event.c，代码如下：

```cpp
// 获取event使用的内核事件通知机制
// base是使用event_base_new()创建的event_base句柄
const char *
event_base_get_method(const struct event_base *base)
{
	EVUTIL_ASSERT(base);
	return (base->evsel->name);
}
```

###1.25.获取当前环境支持的后台方法——event_get_supported_methods

位于event.c，代码如下：

```cpp
// 获取event_base支持的所有事件通知机制，这个函数返回libevent选择的事件机制
// 注意，这个列表包含libevent编译时就支持的所有后台方法，
// 它不会做有关OS的必要性检查，以查看是否有必要的资源。
// 返回指针数组，每个指针指向支持方法的名字，数组的末尾指向NULL
const char **
event_get_supported_methods(void)
{
	static const char **methods = NULL;
	const struct eventop **method;
	const char **tmp;
	int i = 0, k;

	/* count all methods */
	// 遍历静态全局数组eventops，获取编译后的后台方法个数
	for (method = &eventops[0]; *method != NULL; ++method) {
		++i;
	}

	/* allocate one more than we need for the NULL pointer */
	// 分配临时空间，二级指针，用于存放名字指针
	tmp = mm_calloc((i + 1), sizeof(char *));
	if (tmp == NULL)
		return (NULL);

	/* populate the array with the supported methods */
	// 在tmp数组中保存名字指针
	for (k = 0, i = 0; eventops[k] != NULL; ++k) {
		tmp[i++] = eventops[k]->name;
	}
	tmp[i] = NULL;

	if (methods != NULL)
		mm_free((char**)methods);

	methods = tmp;

	return (methods);
}
```


###1.26.获取后台方法的工作模式——event_base_get_features

位于event.c，代码如下：

```cpp
// 获取event_base的后台方法特征，可以是多个特征通过OR方法求并的结果
int
event_base_get_features(const struct event_base *base)
{
	return base->evsel->features;
}
```

###1.27.获取event_base的优先级个数——event_base_get_npriorities函数

位于event.c，代码如下：

```cpp
// 获取不同事件的优先级个数
int
event_base_get_npriorities(struct event_base *base)
{

	int n;
	if (base == NULL)
		base = current_base;

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
	n = base->nactivequeues;
	EVBASE_RELEASE_LOCK(base, th_base_lock);
	return (n);
}
```
