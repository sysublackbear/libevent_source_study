#libevent(10)
@(源码)


## 4.event-read-fifo.c

代码如下：

```cpp
/*
 * This sample code shows how to use Libevent to read from a named pipe.
 * XXX This code could make better use of the Libevent interfaces.
 *
 * XXX This does not work on Windows; ignore everything inside the _WIN32 block.
 *
 * On UNIX, compile with:
 * cc -I/usr/local/include -o event-read-fifo event-read-fifo.c \
 *     -L/usr/local/lib -levent
 */

#include <event2/event-config.h>

#include <sys/types.h>
#include <sys/stat.h>
#ifndef _WIN32
#include <sys/queue.h>
#include <unistd.h>
#include <sys/time.h>
#include <signal.h>
#else
#include <winsock2.h>
#include <windows.h>
#endif
#include <fcntl.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

#include <event2/event.h>

static void
fifo_read(evutil_socket_t fd, short event, void *arg)
{
	char buf[255];
	int len;
	struct event *ev = arg;
#ifdef _WIN32
	DWORD dwBytesRead;
#endif

	fprintf(stderr, "fifo_read called with fd: %d, event: %d, arg: %p\n",
	    (int)fd, event, arg);
#ifdef _WIN32
	len = ReadFile((HANDLE)fd, buf, sizeof(buf) - 1, &dwBytesRead, NULL);

	/* Check for end of file. */
	if (len && dwBytesRead == 0) {
		fprintf(stderr, "End Of File");
		event_del(ev);
		return;
	}

	buf[dwBytesRead] = '\0';
#else
	len = read(fd, buf, sizeof(buf) - 1);

	if (len <= 0) {
		if (len == -1)
			perror("read");
		else if (len == 0)
			fprintf(stderr, "Connection closed\n");
		event_del(ev);
		event_base_loopbreak(event_get_base(ev));
		return;
	}

	buf[len] = '\0';
#endif
	fprintf(stdout, "Read: %s\n", buf);
}

/* On Unix, cleanup event.fifo if SIGINT is received. */
#ifndef _WIN32
static void
signal_cb(evutil_socket_t fd, short event, void *arg)
{
	struct event_base *base = arg;
	event_base_loopbreak(base);
}
#endif

int
main(int argc, char **argv)
{
	struct event *evfifo;
	struct event_base* base;
#ifdef _WIN32
	HANDLE socket;
	/* Open a file. */
	socket = CreateFileA("test.txt",	/* open File */
			GENERIC_READ,		/* open for reading */
			0,			/* do not share */
			NULL,			/* no security */
			OPEN_EXISTING,		/* existing file only */
			FILE_ATTRIBUTE_NORMAL,	/* normal file */
			NULL);			/* no attr. template */

	if (socket == INVALID_HANDLE_VALUE)
		return 1;

#else
	struct event *signal_int;
	struct stat st;
	const char *fifo = "event.fifo";
	int socket;

	if (lstat(fifo, &st) == 0) {
		if ((st.st_mode & S_IFMT) == S_IFREG) {  // ls "event.fifo"返回的结果是普通文件的话
			errno = EEXIST;  // 文件已存在
			perror("lstat");
			exit(1);
		}
	}

	unlink(fifo);  // 删除文件，删除软链
	if (mkfifo(fifo, 0600) == -1) {  // 生成一个命名管道（注意：命名管道fifo，匿名管道pipe）
		perror("mkfifo");
		exit(1);
	}

	socket = open(fifo, O_RDONLY | O_NONBLOCK, 0);  // 当前打开操作是为读而打开FIFO时，非阻塞

	if (socket == -1) {
		perror("open");
		exit(1);
	}

	fprintf(stderr, "Write data to %s\n", fifo);
#endif
	/* Initalize the event library */
	base = event_base_new();  // 初始化event_base

	/* Initalize one event */
#ifdef _WIN32
	evfifo = event_new(base, (evutil_socket_t)socket, EV_READ|EV_PERSIST, fifo_read,
                           event_self_cbarg());
#else
	/* catch SIGINT so that event.fifo can be cleaned up */
	signal_int = evsignal_new(base, SIGINT, signal_cb, base);  // 建立信号事件，作为清理现场
	event_add(signal_int, NULL);

	evfifo = event_new(base, socket, EV_READ|EV_PERSIST, fifo_read,
                           event_self_cbarg());  // 建立evfifo
#endif

	/* Add it to the active events, without a timeout */
	event_add(evfifo, NULL);

	event_base_dispatch(base);
	event_base_free(base);
#ifdef _WIN32
	CloseHandle(socket);
#else
	close(socket);  // 关闭socket
	unlink(fifo);  // 删除管道
#endif
	libevent_global_shutdown();
	return (0);
}
```


###4.1.signal_cb函数

```cpp
static void
signal_cb(evutil_socket_t fd, short event, void *arg)
{
	struct event_base *base = arg;
	event_base_loopbreak(base);  // 退出event_base_loop
}
```

这个跟之前的例子一样，没什么好说的。

###4.2.fifo_read函数

位于event-read-fifo.c，代码如下：

```cpp
static void
fifo_read(evutil_socket_t fd, short event, void *arg)
{
	char buf[255];
	int len;
	struct event *ev = arg;
#ifdef _WIN32
	DWORD dwBytesRead;
#endif

	fprintf(stderr, "fifo_read called with fd: %d, event: %d, arg: %p\n",
	    (int)fd, event, arg);
#ifdef _WIN32
	len = ReadFile((HANDLE)fd, buf, sizeof(buf) - 1, &dwBytesRead, NULL);  
	/* Check for end of file. */
	if (len && dwBytesRead == 0) {
		fprintf(stderr, "End Of File");
		event_del(ev);
		return;
	}

	buf[dwBytesRead] = '\0';
#else
	len = read(fd, buf, sizeof(buf) - 1);  // 读取命名管道

	if (len <= 0) {
		if (len == -1)
			perror("read");
		else if (len == 0)
			fprintf(stderr, "Connection closed\n");
		event_del(ev);
		event_base_loopbreak(event_get_base(ev));
		return;
	}

	buf[len] = '\0';
#endif
	fprintf(stdout, "Read: %s\n", buf);
}
```

这个例子也没什么好说的，就是读取管道的内容，并且打印出来。


###4.3.关于管道

一、进程间通信

每个进程各自有不同的用户地址空间，任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核，在内核中开辟一块缓冲区，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为**进程间通信（IPC，InterProcess Communication）**。如下图所示。

![Alt text](./1539528084336.png)

二、管道是一种最基本的IPC机制，由`pipe`函数创建，其中`pipe`属于匿名管道：

```
#include <unistd.h>

int pipe(int filedes[2]);
```

调用`pipe`函数时在内核中开辟一块缓冲区（称为管道）用于通信，它有一个读端一个写端，然后通过`filedes`参数传出给用户程序两个文件描述符，`filedes[0]`指向管道的读端，`filedes[1]`指向管道的写端（很好记，就像0是标准输入1是标准输出一样）。所以管道在用户程序看起来就像一个打开的文件，通过`read(filedes[0]);`或者`write(filedes[1]);`向这个文件读写数据其实是在读写内核缓冲区。`pipe`函数调用成功返回0，调用失败返回-1。

开辟了管道之后如何实现两个进程间的通信呢？比如可以按下面的步骤通信。

![Alt text](./1539528151414.png)



进程间通信必须通过内核提供的通道，而且必须有一种办法在进程中标识内核提供的某个通道，前面讲过的匿名管道是用打开的文件描述符来标识的。如果要互相通信的几个进程没有从公共祖先那里继承文件描述符，它们怎么通信呢？内核提供一条通道不成问题，问题是如何标识这条通道才能使各进程都可以访问它？文件系统中的路径名是全局的，各进程都可以访问，因此可以用文件系统中的路径名来标识一个IPC通道。



FIFO和UNIX Domain Socket这两种IPC机制都是利用文件系统中的特殊文件来标识的。

FIFO文件在磁盘上没有数据块，仅用来标识内核中的一条通道，如 `prw-rw-r-- 1 simba simba      0 May 21 10:13 p2`，文件类型标识为p表示FIFO，文件大小为0。各进程可以打开这个文件进行read/write，实际上是在读写内核通道（根本原因在于这个file结构体所指向的read、write函数和常规文件不一样），这样就实现了进程间通信。UNIX Domain Socket和FIFO的原理类似，也需要一个特殊的socket文件来标识内核中的通道，例如/run目录下有很多系统服务的socket文件：
```bash
srw-rw-rw- 1 root       root          0 May 21 09:59 acpid.socket
....................
```

文件类型`s`表示`socket`，这些文件在磁盘上也没有数据块。


三、命名管道（FIFO）

匿名管道应用的一个限制就是只能在具有共同祖先（具有亲缘关系）的进程间通信。
如果我们想在不相关的进程之间交换数据，可以使用FIFO文件来做这项工作，它经常被称为命名管道。

命名管道可以从命令行上创建，命令行方法是使用下面这个命令：
```bash
$ mkfifo filename
```
命名管道也可以从程序里创建，相关函数有：
```cpp
int mkfifo(const char *filename,mode_t mode);
```

四、命名管道和匿名管道

匿名管道由`pipe`函数创建并打开。
命名管道由`mkfifo`函数创建，打开用`open`。
`FIFO`（命名管道）与`pipe`（匿名管道）之间唯一的区别在它们创建与打开的方式不同，这些工作完成之后，它们具有相同的语义。
>The  only difference between pipes and FIFOs is the manner in which they are created and opened.  Once these tasks have been accomplished, I/O on pipes and FIFOs has exactly the same semantics.



五、命名管道的打开规则

如果当前打开操作是为读而打开FIFO时
`O_NONBLOCK disable`：阻塞直到有相应进程为写而打开该FIFO
`O_NONBLOCK enable`：立刻返回成功
如果当前打开操作是为写而打开FIFO时
`O_NONBLOCK disable`：阻塞直到有相应进程为读而打开该FIFO
`O_NONBLOCK enable`：立刻返回失败，错误码为ENXIO

需要注意的是打开的文件描述符默认是阻塞的，大家可以写两个很简单的小程序测试一下，主要也就一条语句

`int fd = open("p2", O_WRONLY);` 假设p2是命名管道文件，把打开标志换成 `O_RDONLY` 就是另一个程序了，可以先运行RD程序，此时会阻塞，再在另一个窗口运行WR程序，此时两个程序都会从`open`返回成功。非阻塞时也不难测试，`open`时增加标志位就可以了。
