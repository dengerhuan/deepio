https://blog.csdn.net/qq_37058442/article/details/78004507

select需要驱动程序的支持，驱动程序实现fops内的poll函数。select通过每个设备文件对应的poll函数提供的信息判断当前是否有资源可用(如可读或写)，如果有的话则返回可用资源的文件描述符个数，没有的话则睡眠，等待有资源变为可用时再被唤醒继续执行。


下面我们分两个过程来分析select：

1. select的睡眠过程

支持阻塞操作的设备驱动通常会实现一组自身的等待队列如读/写等待队列用于支持上层(用户层)所需的BLOCK或NONBLOCK操作。当应用程序通过设备驱动访问该设备时(默认为BLOCK操作)，若该设备当前没有数据可读或写，则将该用户进程插入到该设备驱动对应的读/写等待队列让其睡眠一段时间，等到有数据可读/写时再将该进程唤醒。

select就是巧妙的利用等待队列机制让用户进程适当在没有资源可读/写时睡眠，有资源可读/写时唤醒。下面我们看看select睡眠的详细过程。

select会循环遍历它所监测的fd_set内的所有文件描述符对应的驱动程序的poll函数。驱动程序提供的poll函数首先会将调用select的用户进程插入到该设备驱动对应资源的等待队列(如读/写等待队列)，然后返回一个bitmask告诉select当前资源哪些可用。当select循环遍历完所有fd_set内指定的文件描述符对应的poll函数后，如果没有一个资源可用(即没有一个文件可供操作)，则select让该进程睡眠，一直等到有资源可用为止，进程被唤醒(或者timeout)继续往下执行。

下面分析一下代码是如何实现的。
select的调用path如下：sys_select -> core_sys_select -> do_select
其中最重要的函数是do_select, 最主要的工作是在这里, 前面两个函数主要做一些准备工作。do_select定义如下：

linux-4.0.1\fs\select.c

[cpp]  view plain  copy
int do_select(int n, fd_set_bits *fds, struct timespec *end_time)  
{  
    ktime_t expire, *to = NULL;  
    struct poll_wqueues table;  
    poll_table *wait;  
    int retval, i, timed_out = 0;  
    unsigned long slack = 0;  
    unsigned int busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;  
    unsigned long busy_end = 0;  

    rcu_read_lock();  
    retval = max_select_fd(n, fds);  
    rcu_read_unlock();  
      
    if (retval < 0)  
        return retval;  
    n = retval;  
      
    poll_initwait(&table);  
    wait = &table.pt;  
    if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {  
        wait->_qproc = NULL;  
        timed_out = 1;  
    }  
      
    if (end_time && !timed_out)  
        slack = select_estimate_accuracy(end_time);  
      
    retval = 0;     //retval用于保存已经准备好的描述符数，初始为0  
    for (;;) {  
        unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;  
        bool can_busy_loop = false;  
      
        inp = fds->in; outp = fds->out; exp = fds->ex;  
        rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;  
      
        for (i = 0; i < n; ++rinp, ++routp, ++rexp) {        //遍历每个描述符  
            unsigned long in, out, ex, all_bits, bit = 1, mask, j;  
            unsigned long res_in = 0, res_out = 0, res_ex = 0;  
      
            in = *inp++; out = *outp++; ex = *exp++;  
            all_bits = in | out | ex;  
            if (all_bits == 0) {  
                i += BITS_PER_LONG;     //如果这个字没有待查找的描述符, 跳过这个长字(32位)  
                continue;  
            }  
      
            for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {      //遍历每个长字里的每个位  
                struct fd f;  
                if (i >= n)  
                    break;  
                if (!(bit & all_bits))  
                    continue;  
                f = fdget(i);  
                if (f.file) {  
                    const struct file_operations *f_op;  
                    f_op = f.file->f_op;  
                    mask = DEFAULT_POLLMASK;  
                    if (f_op->poll) {  
                        wait_key_set(wait, in, out,  
                                 bit, busy_flag);  
                        /* 在这里循环调用所监测的fd_set内的所有文件描述符对应的驱动程序的poll函数 */  
                        mask = (*f_op->poll)(f.file, wait);  
                    }  
                    fdput(f);  
                    if ((mask & POLLIN_SET) && (in & bit)) {  
                        res_in |= bit;      //如果是这个描述符可读, 将这个位置位  
                        retval++;           //返回描述符个数加1  
                        wait->_qproc = NULL;  
                    }  
                    if ((mask & POLLOUT_SET) && (out & bit)) {  
                        res_out |= bit;  
                        retval++;  
                        wait->_qproc = NULL;  
                    }  
                    if ((mask & POLLEX_SET) && (ex & bit)) {  
                        res_ex |= bit;  
                        retval++;  
                        wait->_qproc = NULL;  
                    }  
                    /* got something, stop busy polling */  
                    if (retval) {  
                        can_busy_loop = false;  
                        busy_flag = 0;  
      
                    /* 
                     * only remember a returned 
                     * POLL_BUSY_LOOP if we asked for it 
                     */  
                    } else if (busy_flag & mask)  
                        can_busy_loop = true;  
      
                }  
            }  
            //返回结果  
            if (res_in)  
                *rinp = res_in;  
            if (res_out)  
                *routp = res_out;  
            if (res_ex)  
                *rexp = res_ex;  
            cond_resched();  
        }  
        wait->_qproc = NULL;  
        /* 到这里遍历结束。retval保存了检测到的可操作的文件描述符的个数。如果有文件可操作，则跳出for(;;)循环，直接返回。若没有文件可操作且timeout时间未到同时没有收到signal,则执行schedule_timeout睡眠。睡眠时间长短由__timeout决定，一直等到该进程被唤醒。 
        那该进程是如何被唤醒的？被谁唤醒的呢？ 
        我们看下面的select唤醒过程*/  
      
        if (retval || timed_out || signal_pending(current))  
            break;  
        if (table.error) {  
            retval = table.error;  
            break;  
        }  
      
        /* only if found POLL_BUSY_LOOP sockets && not out of time */  
        if (can_busy_loop && !need_resched()) {  
            if (!busy_end) {  
                busy_end = busy_loop_end_time();  
                continue;  
            }  
            if (!busy_loop_timeout(busy_end))  
                continue;  
        }  
        busy_flag = 0;  
      
        /* 
         * If this is the first loop and we have a timeout 
         * given, then we convert to ktime_t and set the to 
         * pointer to the expiry value. 
         */  
        if (end_time && !to) {  
            expire = timespec_to_ktime(*end_time);  
            to = &expire;  
        }  
      
        if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,  
                       to, slack))  
            timed_out = 1;  
    }  
      
    poll_freewait(&table);  
      
    return retval;  
}  


2.  select的唤醒过程
前面介绍了select会循环遍历它所监测的fd_set内的所有文件描述符对应的驱动程序的poll函数。驱动程序提供的poll函数首先会将调用select的用户进程插入到该设备驱动对应资源的等待队列(如读/写等待队列)，然后返回一个bitmask告诉select当前资源哪些可用。
一个典型的驱动程序poll函数实现如下：
(摘自《Linux Device Drivers – ThirdEdition》Page 165)
[cpp]  view plain  copy
static unsigned int scull_p_poll(struct file *filp, poll_table *wait)  
{  
    struct scull_pipe *dev = filp->private_data;  
    unsigned int mask = 0;  
    /* 
    * The buffer is circular; it is considered full 
    * if "wp" is right behind "rp" and empty if the 
    * two are equal. 
      */  
      down(&dev->sem);  
      poll_wait(filp, &dev->inq,  wait);  
      poll_wait(filp, &dev->outq, wait);  
      if (dev->rp != dev->wp)  
        mask |= POLLIN | POLLRDNORM;    /* readable */  
      if (spacefree(dev))  
        mask |= POLLOUT | POLLWRNORM;   /* writable */  
      up(&dev->sem);  
      return mask;  
}  
将用户进程插入驱动的等待队列是通过poll_wait做的。
Poll_wait定义如下：
[cpp]  view plain  copy
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)  
{  
      if (p && wait_address)  
        p->qproc(filp, wait_address, p);  
}  

这里的p->qproc在do_select内poll_initwait(&table)被初始化为__pollwait，如下：
[cpp]  view plain  copy
void poll_initwait(struct poll_wqueues *pwq)  
{  
    init_poll_funcptr(&pwq->pt, __pollwait);  
    pwq->error = 0;  
    pwq->table = NULL;  
    pwq->inline_index = 0;  
}  

__pollwait定义如下：
[cpp]  view plain  copy
/* Add a new entry */  
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,  
                        poll_table *p)  
{  
    struct poll_table_entry *entry = poll_get_entry(p);  
    if (!entry)  
    return;  
    get_file(filp);  
    entry->filp = filp;  
    entry->wait_address = wait_address;  
    init_waitqueue_entry(&entry->wait, current);  
    add_wait_queue(wait_address,&entry->wait);  
}  

通过init_waitqueue_entry初始化一个等待队列项，这个等待队列项关联的进程即当前调用select的进程。然后将这个等待队列项插入等待队列wait_address。Wait_address即在驱动poll函数内调用poll_wait(filp, &dev->inq,  wait);时传入的该驱动的&dev->inq或者&dev->outq等待队列。

注: 关于等待队列的工作原理可以参考下面这篇文档:
http://blog.chinaunix.net/u2/60011/showart_1334657.html

到这里我们明白了select如何当前进程插入所有所监测的fd_set关联的驱动内的等待队列，那进程究竟是何时让出CPU进入睡眠状态的呢？
进入睡眠状态是在do_select内调用schedule_timeout(__timeout)实现的。当select遍历完fd_set内的所有设备文件，发现没有文件可操作时(即retval=0),则调用schedule_timeout(__timeout)进入睡眠状态。

唤醒该进程的过程通常是在所监测文件的设备驱动内实现的，驱动程序维护了针对自身资源读写的等待队列。当设备驱动发现自身资源变为可读写并且有进程睡眠在该资源的等待队列上时，就会唤醒这个资源等待队列上的进程。

举个例子，比如内核的8250 uart driver:
Uart是使用的Tty层维护的两个等待队列, 分别对应于读和写: (uart是tty设备的一种)
struct tty_struct {
         ……
         wait_queue_head_t write_wait;
         wait_queue_head_t read_wait;
         ……
}
当uart设备接收到数据，会调用tty_flip_buffer_push(tty);将收到的数据push到tty层的buffer。
然后查看是否有进程睡眠的读等待队列上，如果有则唤醒该等待会列。
过程如下：
serial8250_interrupt -> serial8250_handle_port -> receive_chars -> tty_flip_buffer_push ->
flush_to_ldisc -> disc->receive_buf
在disc->receive_buf函数内：
if (waitqueue_active(&tty->read_wait)) //若有进程阻塞在read_wait上则唤醒
wake_up_interruptible(&tty->read_wait);

到这里明白了select进程被唤醒的过程。由于该进程是阻塞在所有监测的文件对应的设备等待队列上的，因此在timeout时间内，只要任意个设备变为可操作，都会立即唤醒该进程，从而继续往下执行。这就实现了select的当有一个文件描述符可操作时就立即唤醒执行的基本原理。
————————————————
版权声明：本文为CSDN博主「qq_37058442」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_37058442/article/details/78004507