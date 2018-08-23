
# 进程  
```
// glibc-master\sysdeps\nptl\fork.c
static inline pid_t
arch_fork (void *ctid)
{
  const int flags = CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD;
  long int ret;
  ...
  ret = INLINE_CLONE_SYSCALL (flags, 0, NULL, 0, ctid);  // 封装了 clone 系统调用
}
```
- strace 查看系统调用

```
strace ./a.out
execve("./a.out", ["./a.out"], [/* 21 vars */]) = 0
...
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7ff637155a10) = 1978
exit_group(0)                           = ?
+++ exited with 0 +++
```


# 线程
```
//  glibc-master\sysdeps\unix\sysv\linux\createthread.c
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);
```

__clone 函数的实现
```
// glibc-master\sysdeps\unix\sysv\linux\x86_64\sysdep.h
#define SYS_ify(syscall_name)	__NR_##syscall_name
```
```
// glibc-master\sysdeps\unix\sysv\linux\x86_64\clone.S
ENTRY (__clone)
	/* Sanity check arguments.  */
	movq	$-EINVAL,%rax
	testq	%rdi,%rdi		/* no NULL function pointers */
	jz	SYSCALL_ERROR_LABEL
	testq	%rsi,%rsi		/* no NULL stack pointers */
	jz	SYSCALL_ERROR_LABEL

	/* Insert the argument onto the new stack.  */
	subq	$16,%rsi
	movq	%rcx,8(%rsi)

	/* Save the function pointer.  It will be popped off in the
	   child in the ebx frobbing below.  */
	movq	%rdi,0(%rsi)

	/* Do the system call.  */
	movq	%rdx, %rdi
	movq	%r8, %rdx
	movq	%r9, %r8
	mov	8(%rsp), %R10_LP
	movl	$SYS_ify(clone),%eax

	/* End FDE now, because in the child the unwind info will be
	   wrong.  */
	cfi_endproc;
	syscall

	testq	%rax,%rax
	jl	SYSCALL_ERROR_LABEL
	jz	L(thread_start)

	ret

...
```
