> 委托给 `ep_insert`
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

> 初始化epitem
在`ep_insert` 里面会初始化`epitem`

```c
static int ep_insert(struct eventpoll *ep, const struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{

	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))  // 分配epitem 不知道是分配页还是slab 分配 应该是slab
		return -ENOMEM;

	/* Item initialization follow here ... */
	INIT_LIST_HEAD(&epi->rdllink);   // 初始化 rdllink
	INIT_LIST_HEAD(&epi->fllink);    // 初始化 fllink
	INIT_LIST_HEAD(&epi->pwqlist);    // 初始化pwqlist
	epi->ep = ep;                      // epi 指向 相应的ep
	ep_set_ffd(&epi->ffd, tfile, fd);   // 设置自己的fd 和 file *
	epi->event = *event;                 // 设置自己关注的事件
	epi->nwait = 0;                       // 初始化
	epi->next = EP_UNACTIVE_PTR;           // 也是初始化
}
```
