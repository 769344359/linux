ip协议族的协议主要包括以下几个函数  
socket  
bind  
listen  
accept  


--------------
背景知识  
相关ops函数指针的赋值
```c
int sock_register(const struct net_proto_family *ops)
{
	int err;

	if (ops->family >= NPROTO) {
		pr_crit("protocol %d >= NPROTO(%d)\n", ops->family, NPROTO);
		return -ENOBUFS;
	}

	spin_lock(&net_family_lock);
	if (rcu_dereference_protected(net_families[ops->family],
				      lockdep_is_held(&net_family_lock)))
		err = -EEXIST;
	else {
		rcu_assign_pointer(net_families[ops->family], ops);
		err = 0;
	}
	spin_unlock(&net_family_lock);

	pr_info("NET: Registered protocol family %d\n", ops->family);
	return err;
}
EXPORT_SYMBOL(sock_register);
```
> socket 函数  
socket 函数主要有两部分  
- sock_create    // 分配socket 结构
- sock_map_fd    // socket 绑定fd
```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	int retval;
	struct socket *sock;
	int flags;
...
	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		return retval;

	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
```

首先看 sock_create 函数,他会委托给__sock_create 函数
```c
int sock_create(int family, int type, int protocol, struct socket **res)
{
	return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}
int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;

...
	sock = sock_alloc();
...

	sock->type = type;
	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);
	...
	err = pf->create(net, sock, protocol, kern);
	...
	*res = sock;

	return 0;

...
}
EXPORT_SYMBOL(__sock_create);
```


