epoll  有一个很重要的逻辑是添加进ep 的红黑树和添加一堆回调和添加唤醒的列表
```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
 ...
 switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= EPOLLERR | EPOLLHUP;
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
 ...
}
```
添加的时候会委托给函数`ep_insert` 函数,下面我们来看看`ep_insert`函数

