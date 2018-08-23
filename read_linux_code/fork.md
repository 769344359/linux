```
SYSCALL_DEFINE0(fork)
{
	return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);

}
```

线程
```
//  glibc-master\sysdeps\unix\sysv\linux\createthread.c
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);
```
