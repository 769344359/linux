
# 进程  
```
// glibc-master\sysdeps\nptl\fork.c
static inline pid_t
arch_fork (void *ctid)
{
  const int flags = CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD;
  long int ret;
  ...
  ret = INLINE_CLONE_SYSCALL (flags, 0, NULL, 0, ctid);  // 封装了 __clone 函数
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
 if (__glibc_unlikely (ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS,
				    clone_flags, pd, &pd->tid, tp, &pd->tid)
			== -1))                   // 封装了__clone 
    return errno;
```

# __clone 函数的实现
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

# flags 区别
- fork: 
```
CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD  
```
- thread : 
```
(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);
```

- CLONE_VM
```
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
	 ...
	tsk->mm = NULL;
	tsk->active_mm = NULL;
	oldmm = current->mm; // 当前的mm  线程的话都会用同一个mm struct  所以都共享同一个地址空间
	if (clone_flags & CLONE_VM) {
		...
		mm = oldmm;
		goto good_mm;
	}

	retval = -ENOMEM;
	mm = dup_mm(tsk); // 复制mm结构  
	if (!mm)
		goto fail_nomem;

good_mm:
	tsk->mm = mm;
	tsk->active_mm = mm;
	return 0;
}
```
