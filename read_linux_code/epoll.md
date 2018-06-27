epoll  主要有三个函数  
epoll_create  
epoll_ctl  
epoll_wait  

下面主要看epoll_wait  函数有四个参数  


epfd 其实就是一个fd 和其他fd 没有区别,所以要校验f_op    

epoll_wait 的工作都委托给ep_poll 了

```c
/*
 * Implement the event wait interface for the eventpoll file. It is the kernel
 * part of the user space epoll_wait(2).
 */
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{

	...
	/* Get the "struct file *" for the eventpoll file */
	f = fdget(epfd);
	if (!f.file)                        // fd 获得文件
		return -EBADF;

	/*
	 * We have to check that the file structure underneath the fd
	 * the user passed to us _is_ an eventpoll file.
	 */
	error = -EINVAL;
	if (!is_file_epoll(f.file))                          // 判断是否是epoll fd 一直到这里都是校验接口
		goto error_fput;

	/*
	 * At this point it is safe to assume that the "private_data" contains
	 * our own data structure.
	 */
	ep = f.file->private_data;

	/* Time to fish for events ... */
	error = ep_poll(ep, events, maxevents, timeout);         // 委托给 ep_poll

error_fput:
	fdput(f);
	return error;
}
```


下面我们来看看 ep_poll 函数  

ep_poll 的参数就是epoll_wait 的参数

timeout 是 0 或者不是 0 会走不同的分支,先看0   
0 的话会直接跳转到查询rdlist
```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
	int res = 0, eavail, timed_out = 0;
	unsigned long flags;
	u64 slack = 0;
	wait_queue_entry_t wait;
	ktime_t expires, *to = NULL;
	 ...
	if (timeout == 0) { 
		/*
		 * Avoid the unnecessary trip to the wait queue loop, if the
		 * caller specified a non blocking operation.
		 */
		timed_out = 1;
		spin_lock_irqsave(&ep->lock, flags);
		goto check_events;
	}

	...
check_events:
	/* Is it worth to try to dig for events ? */
	eavail = ep_events_available(ep);

	spin_unlock_irqrestore(&ep->lock, flags);

	/*
	 * Try to transfer events to user space. In case we get 0 events and
	 * there's still timeout left over, we go trying again in search of
	 * more luck.
	 */
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;

	return res;
}

```

当timeout  = 0 时候会直接查询是否有准备好的,如果有就传到用户态去

我们先看ep_events_available 函数

