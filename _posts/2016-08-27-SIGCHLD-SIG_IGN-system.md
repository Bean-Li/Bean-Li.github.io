---
layout: post
title: SIGCHLD SIG_IGN和system
date: 2016-08-27 14:43:40
categories: linux
tag: kernel
excerpt: SIGCHLD and SIG_IGN
---

# 前言

周五的高峰问我，SIGCHLD如果设置了SIG\_IGN，对于wait或waitpid，能否等到子进程退出。我当时心里一咯噔，我突然想起了SIG\_IGN对system函数的影响。
书中154页曾经提到过，如果SIGCHLD显式地设置了SIG\_IGN活着SA
\_NOCLDWAIT标志位，那么system函数总是返回－1，并且errno 为ECHLD。

对于此处，我最早从网上找到该资料，写入书的时候，我确实验证过了，返回－1并且errno为ECHLD，但是我并没有深究背后的原理。

后面高峰摘了manual中的一段话：

       POSIX.1-2001  specifies  that  if  the disposition of SIGCHLD is set to SIG_IGN or the SA_NOCLDWAIT flag is set for SIGCHLD (see sigaction(2)), then
       children that terminate do not become zombies and a call to wait() or waitpid() will block until all children have terminated, and  then  fail  with
       errno  set  to ECHILD.  (The original POSIX standard left the behavior of setting SIGCHLD to SIG_IGN unspecified.  Note that even though the default
       disposition of SIGCHLD is "ignore", explicitly setting the disposition to SIG_IGN results in different treatment of zombie process children.)  Linux
       2.6  conforms  to  this  specification.   However,  Linux  2.4  (and earlier) does not: if a wait() or waitpid() call is made while SIGCHLD is being
       ignored, the call behaves just as though SIGCHLD were not being ignored, that is, the call blocks until the next child terminates and  then  returns
       the process ID and status of that child.

当时看到手册中这段话，确实很出乎以外。因为长久以来，wait和waitpid等待某个子进程，而manual提到，如果SIGCHLD信号的信号处理函数如果显式指定为SIG_IGN，那么wait或者waitpid需要等待所有的子进程退出，并且设置errno为ECHLD。

重点是wait或者waitpid函数会等待所有子进程退出， 是所有。

# 应用层测试

对于SIGCHLD的信号处理函数设置成SIG_IGN，我确实没深入的探究。昨天高峰写了测试程序waitpid，怀疑manual手册有误。

```
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>
#include <errno.h>
#include <string.h>

int main(void)
{
	pid_t child1, child2;

	printf("Parent is running\n");

	signal(SIGCHLD, SIG_IGN);


	child1 = fork();
	if (child1 == -1) {
		printf("Fail to fork child1\n");
		exit(1);
	} else if (child1 == 0) {
		printf("Child1 enter sleep\n");
		sleep(30);
		printf("Child1 wakes up and exit\n");
		exit(0);
	}

	printf("Fork child1 successfully\n");
	child2 = fork();
	if (child2 == -1) {
		printf("Fail to fork child2\n");
		waitpid(child1, NULL, 0);
		exit(1);
	} else if (child2 == 0) {
		printf("Child2 exit now\n");
		exit(0);
	}

	printf("Fork child2 successfully, and wait child2\n");
	sleep(3);
	pid_t ret = waitpid(child2, NULL, 0);
	if (ret == -1) {
		printf("waitpid child2 return -1, errno(%d): %s\n",
			errno, strerror(errno));
	}
	printf("Wait child1 now\n");
	waitpid(child1, NULL, 0);

	printf("Parent exit\n");
	
	return 0;
}
```

运行上面的测试结果，waitpid并没有等待所有的子进程退出，并返回ECHLD错误。因此高峰怀疑手册的中话有问题。

我今天也写了测试程序：

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <errno.h>

void print_wait_exit(pid_t child, int status, int err)
{
    printf("child = %d status = %x errno = %d\n",child, status, err);
    if(child > 0)
    {

        if(WIFEXITED(status))
        {
            printf("normal termination,exit code = %d\n", WEXITSTATUS(status));
        }
        else if(WIFSIGNALED(status))
        {
            printf("abnormal termination,signal number = %d%s\n", WTERMSIG(status),
#ifdef WCOREDUMP
                    WCOREDUMP(status)?"core file generated": "");
#else
            "");
#endif
        }
    }
}

void test_wait()
{

    printf("parent=%d\n", getpid());

    if (fork() == 0)
    {
        printf("child1=%d\n", getpid());
        sleep(3);
        printf("exit child1\n");
        exit(1);
    }

    if (fork() == 0)
    {
        printf("child2=%d\n", getpid());
        sleep(5);
        printf("exit child2\n");
        exit(2);

    }

    while (1)
    {
        int stat;
        pid_t c = wait(&stat);
        int err = errno;

        if (c < 0)
            perror("wait");

        printf("child=%d, exit=%x, err=%d\n", c, stat, err);
        print_wait_exit(c, stat, err);
        if (c < 0)
            break;
    }

}



int main()
{

    printf("------- ROUND 1 SIG_DFL---------\n");
    test_wait();
    printf("-------------------------\n\n");

    printf("------- ROUND 2 SIG_IGN---------\n");
    signal(SIGCHLD, SIG_IGN);
    test_wait();
    printf("-------------------------\n\n");

    return 0;
}

```

运行结果如下：

```
manu in ~/GIT/linux_detail_codes/1.waitpid_sigchld_ignore on master ● ● λ ./wait_sigchld_ignore
------- ROUND 1 SIG_DFL---------
parent=2631
child2=2633
child1=2632
exit child1
child=2632, exit=100, err=0
child = 2632 status = 100 errno = 0
normal termination,exit code = 1
exit child2
child=2633, exit=200, err=0
child = 2633 status = 200 errno = 0
normal termination,exit code = 2
wait: No child processes
child=-1, exit=200, err=10
child = -1 status = 200 errno = 10
-------------------------

------- ROUND 2 SIG_IGN---------
parent=2631
child1=2634
child2=2635
exit child1
exit child2
wait: No child processes
child=-1, exit=0, err=10
child = -1 status = 0 errno = 10
-------------------------

```


通过上面运行结果可以看出，只要将SIGCHLD的信号处理函数显式地指定为SIG_IGN，那么wait函数就会一直等到所有的子进程退出，然后返回－1，并置errno为ECHLD。

那么昨天高峰的程序为什么得到了相反的结论呢？
事实上，高峰的程序也没错，waitpid确实没有等待所有子进程退出，但是得出manual有错是不对的。

当waitpid明确等待某一个特定的子进程时，waitpid表现为正常的语意，即不会等待所有的子进程退出。
但是如果waitpid等待任意子进程的时候,如果SIGCHLD的信号处理函数为SIG_IGN时，会等待所有子进程退出，返回-1,并置errno为ECHLD
waitpid(-1,&status,0)

将高峰程序，改写下，waitpid不等待特定子进程而是任意子进程，实验结果，也验证了上述结论：

```
manu in ~/GIT/linux_detail_codes/1.waitpid_sigchld_ignore on master ● ● λ ./waitpid_sigchld_ignore
Parent is running
Fork child1 successfully
Child1 enter sleep
Fork child2 successfully, and wait child2
Child2 exit now
Child1 wakes up and exit
Wait child1 now
Parent exit
```

再回到system系统调用，尽管不显式指定SIGCHLD的信号处理函数为SIG_IGN，该信号的默认行为也是ignore，但是ignore和显式指定为SIG_IGN还是有区别的。昨天高峰向我求证，system函数对SIGCHLD做了什么事情，我回答说，仅仅是阻塞。高峰担心阻塞时不足够的。

现在比较明确了，尽管SIGCHLD的默认行为是ignore，但是SIG_IGN会影响wait和waitpid的行为模式，因此，调用system函数之前，一定不要将SIGCHLD的信号处理函数设置成SIG_IGN，否则system函数返回值不能正确获取命令的返回值。


# 内核实现

SIGCHLD信号的默认行为ignore和SIG_IGN到底有啥不同。

笼统地说，ignore是父进程受到该信号之后的行为模式是ignore，丢弃之，并不影响子进程的行为，但是如果父进程的信号处理函数设置成了SIG_IGN或者指定了SA_NOCLDWAIT标志位，影响的是子进程的退出行为，进而影响到父进程的wait行为。

下面我结合代码，详细展开：

```
static long do_wait(struct wait_opts *wo)
{
	struct task_struct *tsk;
	int retval;

	trace_sched_process_wait(wo->wo_pid);

	init_waitqueue_func_entry(&wo->child_wait, child_wait_callback);
	wo->child_wait.private = current;
	add_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
repeat:
	/*
	 * If there is nothing that can match our criteria, just get out.
	 * We will clear ->notask_error to zero if we see any child that
	 * might later match our criteria, even if we are not able to reap
	 * it yet.
	 */
	wo->notask_error = -ECHILD;
	if ((wo->wo_type < PIDTYPE_MAX) &&
	   (!wo->wo_pid || hlist_empty(&wo->wo_pid->tasks[wo->wo_type])))
		goto notask;

	set_current_state(TASK_INTERRUPTIBLE);
	read_lock(&tasklist_lock);
	tsk = current;
	do {
		retval = do_wait_thread(wo, tsk);
		if (retval)
			goto end;

		retval = ptrace_do_wait(wo, tsk);
		if (retval)
			goto end;

		if (wo->wo_flags & __WNOTHREAD)
			break;
	} while_each_thread(current, tsk);
	read_unlock(&tasklist_lock);

notask:
	retval = wo->notask_error;
	if (!retval && !(wo->wo_flags & WNOHANG)) {
		retval = -ERESTARTSYS;
		if (!signal_pending(current)) {
			schedule();
			goto repeat;
		}
	}
end:
	__set_current_state(TASK_RUNNING);
	remove_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
	return retval;
}
```

上面的代码是wait包括waitpid函数的内核主干部分。我们把上面代码分解，其实就三个部分

1 初始化等待队列，就相当于部署了传感器，任何一个子进程退出，都会传递信号，父进程就收到了传感器信号，及时醒来。可以看下后面的goto repeat，即醒来之后再检查：

```
	init_waitqueue_func_entry(&wo->child_wait, child_wait_callback);
	wo->child_wait.private = current;
	add_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
```

这部分代码和后面的如下代码相呼应，决定了父进程的不断沉睡等待子进程退出。

	retval = wo->notask_error;
	if (!retval && !(wo->wo_flags & WNOHANG)) {
		retval = -ERESTARTSYS;
		if (!signal_pending(current)) {
			schedule();
			goto repeat;
		}
	}

第二部分是scan，扫描所有的子进程，检查是否有符合条件的子进程退出。

```
	set_current_state(TASK_INTERRUPTIBLE);
	read_lock(&tasklist_lock);
	tsk = current;
	do {
		retval = do_wait_thread(wo, tsk);
		if (retval)
			goto end;

		retval = ptrace_do_wait(wo, tsk);
		if (retval)
			goto end;

		if (wo->wo_flags & __WNOTHREAD)
			break;
	} while_each_thread(current, tsk);
	read_unlock(&tasklist_lock);
	
```

这一部分代码的关键在do while循环，循环的关键在于

```
		retval = do_wait_thread(wo, tsk);
		if (retval)
			goto end;
```

do\_wait\_thread的返回值retval决定了是goto end 返回到用户空间，还是继续下一轮等待。

长久以来，我们总是认为，只要等到了一个子进程退出，就会goto end，返回到用户空间，这种想法大部分情况是对的，但是当SIGCHLD的信号处理函数为SIG_IGN的时候，这种想法并不对。

为什么不对？因为只要retval ＝ do\_wait\_thread的返回值是0，并不会goto end进而返回到用户空间，而是进入下一个轮回。

很明显，当SIGCHLD的信号处理函数为SIG_IGN的时候，retval = do\_wait\_thread的值就会是0。

```
static int do_wait_thread(struct wait_opts *wo, struct task_struct *tsk)
{
	struct task_struct *p;

	list_for_each_entry(p, &tsk->children, sibling) {
		int ret = wait_consider_task(wo, 0, p);

		if (ret)
			return ret;
	}

	return 0;
}

```

下面进入最关键的 wait\_consider\_task函数：

```
static int wait_consider_task(struct wait_opts *wo, int ptrace,
				struct task_struct *p)
{
	/*
	 * We can race with wait_task_zombie() from another thread.
	 * Ensure that EXIT_ZOMBIE -> EXIT_DEAD/EXIT_TRACE transition
	 * can't confuse the checks below.
	 */
	int exit_state = ACCESS_ONCE(p->exit_state);
	int ret;

	if (unlikely(exit_state == EXIT_DEAD))
		return 0;

	ret = eligible_child(wo, ptrace, p);
	if (!ret)
		return ret;

```
注意，当exit\_status == EXIT\_DEAD的时候，就会返回0，进而导致do\_wait\_thread返回0，导致do\_wait函数中的do while循环无法跳出，goto end。

这个地方就比较明显了，即当子进程没有进入Zombie状态，直接进入EXIT\_DEAD状态，do\_wait函数就不能及时的goto end，进而返回用户空间。

现在接力棒就交到了子进程的一边。子进程什么时候会进入僵尸状态，什么时候会跳过僵尸状态，直接进入EXIT\_DEAD状态？

当子进程退出的时候，自我了断灰飞烟灭还是进出僵尸状态等父进程收尸，这是一个to be or not to be的问题：

```
static void exit_notify(struct task_struct *tsk, int group_dead)
{
	bool autoreap;
	struct task_struct *p, *n;
	LIST_HEAD(dead);

	write_lock_irq(&tasklist_lock);
	forget_original_parent(tsk, &dead);

	if (group_dead)
		kill_orphaned_pgrp(tsk->group_leader, NULL);

	if (unlikely(tsk->ptrace)) {
		int sig = thread_group_leader(tsk) &&
				thread_group_empty(tsk) &&
				!ptrace_reparented(tsk) ?
			tsk->exit_signal : SIGCHLD;
		autoreap = do_notify_parent(tsk, sig);
	} else if (thread_group_leader(tsk)) {
		autoreap = thread_group_empty(tsk) &&
			do_notify_parent(tsk, tsk->exit_signal);
	} else {
		autoreap = true;
	}

```

上述代码中autoreap就是自我了断的意思，如果autoreap 是true，子进程就跳过了僵尸台，直接自行了断了。我们看一下do\_notify\_parent函数：

```
	psig = tsk->parent->sighand;
	spin_lock_irqsave(&psig->siglock, flags);
	if (!tsk->ptrace && sig == SIGCHLD &&
	    (psig->action[SIGCHLD-1].sa.sa_handler == SIG_IGN ||
	     (psig->action[SIGCHLD-1].sa.sa_flags & SA_NOCLDWAIT))) {
		/*
		 * We are exiting and our parent doesn't care.  POSIX.1
		 * defines special semantics for setting SIGCHLD to SIG_IGN
		 * or setting the SA_NOCLDWAIT flag: we should be reaped
		 * automatically and not left for our parent's wait4 call.
		 * Rather than having the parent do it as a magic kind of
		 * signal handler, we just set this to tell do_exit that we
		 * can be cleaned up without becoming a zombie.  Note that
		 * we still call __wake_up_parent in this case, because a
		 * blocked sys_wait4 might now return -ECHILD.
		 *
		 * Whether we send SIGCHLD or not for SA_NOCLDWAIT
		 * is implementation-defined: we do (if you don't want
		 * it, just use SIG_IGN instead).
		 */
		autoreap = true;
		if (psig->action[SIGCHLD-1].sa.sa_handler == SIG_IGN)
			sig = 0;
	}
	if (valid_signal(sig) && sig)
		__group_send_sig_info(sig, &info, tsk->parent);
	__wake_up_parent(tsk, tsk->parent);
	spin_unlock_irqrestore(&psig->siglock, flags);

	return autoreap;
```

psig，即parent sighand的意思，如果父进程很绝情，设置了SIG\_IGN或者指定了SA\_NOCLDWAIT标志位，那么子进程会设置autoreap ＝ true。无论是SIG\_IGN还是SA\_NOCLDWAIT，子进程都会通过等待队列唤醒父进程，但是两者之间有一点点差异，SIG\_IGN不会发送信号SIGCHLD给父进程，而 SA\_NOCLDWAIT标志位设置了，子进程退出时也会想父进程发送SIGCHLD信号，尽管父进程并不理会该信号。

OK，无论SIG\_IGN还是SA\_NOCLDWAIT，都会导致autoreap 为true，进而导致子进程直接进入EXIT\_DEAD的状态，在平行宇宙的另一端，父进程的do\_wait会醒来，会发现有一个子进程的状态时EXIT\_DEAD，默默地瞟了一眼，当什么也没发生过，继续沉睡。

这就是默认的ignore和设置了SIG\_IGN的区别。默认的ignore，不影响子进程进入僵尸态，而SIG_IGN，子进程根本就不会进入僵尸态，直接进入EXIT\_DEAD，导致在do\_wait中，父进程无法跳出循环，回到用户空间。
