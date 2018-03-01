########################
内核中的task结构以及调度
########################


task结构参考: https://tampub.uta.fi/bitstream/handle/10024/96864/GRADU-1428493916.pdf

其中的schdule章节

user-thread/lwp
======================

内核中调度的单位是task, task对应一个process还是thread? 这首先是一个历史问题了.

POSIX, process, thread, light weight process(lwp), user-space-thread, green thread这些关系(历史), 参考http://blog.csdn.net/u010154760/article/details/45310513

而在https://stackoverflow.com/questions/8639150/is-pthread-library-actually-a-user-thread-solution中, 提到关于内核的nptl:

*NPTL is a so-called 1×1 threads library, in that threads created by the user (via the pthread_create() library function) are in 1-1 correspondence with schedulable entities in the kernel (tasks, in the Linux case). This is the simplest possible threading implementation.*

--- nptl的维基

看起来glibc中nptl实现的user-space-thread和lwp是1对1关系.


task结构
=========


https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L520

.. code-block:: c


struct task_struct {

// 先省略了代码

}


由于linux内核调度是preempty的, 也就是抢占式的, 所以必然有一个优先级的属性


优先级
==========


.. code-block:: c

    int				prio;
    int				static_prio;
    int				normal_prio;
    unsigned int		rt_priority;
    const struct sched_class	*sched_class;
    struct sched_entity		se;

