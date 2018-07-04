ip协议族的协议主要包括以下几个函数  
socket  
bind  
listen  
accept  

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
```
