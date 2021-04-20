# Overview of OSv

OSv is a unikernel dedicated for cloud VM system. First of all, it is a unikernel. This means, the system only has one single address space, no concept of process.  You can think of the whole system as a single process directly running in a VM. Thus, benefit gains for this kind of system because there is no need to distinguish kernel mode and user mode. Calling to “system calls” will be compiled as direct function call, thus, reducing the overheads of system calls. Complicated context switch overheads can also be eliminated because the system does not need to switch page spaces and go through complicated PCB data structures. Threads still exist in OSv and it provides thread scheduler in the system.

OSv overcomes some disadvantages of some unikernels running directly on hardware. A big problem is that those unikernels need to support kinds of device drivers in order to run in a specific hardware, thus adding code complexity. It is also not easy for those system to provide strong isolation because the single address model. Thus, OSv chooses to implement the system under VM which is now a basic infrastructure in the cloud. VM usually not only provides stable and fixed amount of device interfaces, but also provides strong isolation in a hardware level. Naturally, you can think of each of OSv instance as a single "VM based application", with relative small image size (around 6.7MB)  and quick boot time (around 5ms). 

This concept is very helpful in a cloud environment. For example, you have a Web server application running in a cloud. At a moment, a huge number of requests come in, you have no choice but to deploy massive VM servers in order to consume those requests. OSv is helpful in this case compared with traditional Linux VM because it is very light and small, and can be booted quickly. Providers can quickly and massively deploy their new servers. Also, because there is only one application in one OSv, the performance usually gains in this case. This means that the provider can use less money to provide the same service quality.

# Domain of the system

The target domain of OSv is unikernel and virtual machine. It is valuable for those applications like microservices that only need a small & lightweight runtime environment, not only improving the performance for those applications but also providing the ability to massively deploy services in a short time. Also, as an unikernel, because OSv is within a VM, it doesn't need to bother with different hardware drivers, the code base could be relative small, having less potential bugs.
The security of the system could also be guaranteed because each OSv instance is running under a VM providing strong isolation.

Disadvantages also exist in OSv. First, it is not easy to deploy complex systems within OSv because it only has a single address space and does not support multiple process. Thus,  APIs like fork()/vfork()/exec() cannot be used. However, modern applications usually compose of different components, each of which is a single processes. Transplanting those kind of systems running in OSv is not easy. Second, because OSv is running in a VM, different components have to be put into different VMs/OSvs, thus communication between components is not flexible compared with Linux VM guest and probably causing poor performance for applications that heavily rely on such communications.

# Components of OSv

## Memory
The memory management in the OSv follows the POSIX-like APIs which can map and
unmap memory by ’map’ and ’unmap’ API [1]. Since OSv aims to support a single ap-
plication and only has one single memory space, it does not support memory eviction.
However, it supports large memory allocation and can break the large page into smaller pages
by using the mechanism similar to Linux’s Transparent Huge Pages[1]. In the Uniker-
nel, the user and kernel share the same kernel space which means the user and kernel use
the same `malloc()` function. Here is the architecture comparison of the VM, container,
and unikernel[2].
    ![Mem-Pic1](resources/Pic1.png?raw=true)
    <center></center>
    
The single memory space can help improve the efficiency of the scheduler since it means
the context switch does not need to switch the page table and flush TLB.

- Implementation
  
    ```C++
    struct page_allocator {
        virtual bool map(uintptr_t offset, hw_ptep<0> ptep, pt_element<0> pte, bool write) = 0;
        virtual bool map(uintptr_t offset, hw_ptep<1> ptep, pt_element<1> pte, bool write) = 0;
        virtual bool unmap(void *addr, uintptr_t offset, hw_ptep<0> ptep) = 0;
        virtual bool unmap(void *addr, uintptr_t offset, hw_ptep<1> ptep) = 0;
        virtual ~page_allocator() {}
    };
    ```
    ```C++
    void flush_tlb_all()
    {
        static std::vector<sched::cpu*> ipis(sched::max_cpus);

        if (sched::cpus.size() <= 1) {
            mmu::flush_tlb_local();
            return;
        }

        SCOPE_LOCK(migration_lock);
        mmu::flush_tlb_local();
        std::lock_guard<mutex> guard(tlb_flush_mutex);
        tlb_flush_waiter.reset(*sched::thread::current());
        ...
        tlb_flush_waiter.clear();
    }
    ```
    
## Thread
The design principles of OSv thread scheduling include 6 points: lock-free，preemptive，tick-less，fair，scalable and efficient.
- Lock-free

    The spin-lock is widely used in modern OS; however, when the spin-lock is implemented in the VM, it will lead to the 'lock-holder preemption' problem: while physical CPUs are always running if the OS wants them to, virtual CPUs may “pause” at unknown times for unknown durations[1]. If a virtual CPU is suspended while holding a spin lock, other CPUs that want to acquire the lock need to perform spin operations (spin) unnecessarily, which wastes CPU time. Similarly, for mutexes that use the spin method, the thread without lock will spin continuously and waste CPU time. Experiments show that this will bring about 7% to 99% (extreme case) performance loss[3].
    
    To solve this, OSv gives up the spin-locks and the sleeping mutex. In OSv, each CPU keeps a run-queue which lists all the runnable threads and use lock-free algorithm to implement mutex lock. Besides, to balance work-load, each CPU will have a load balancer thread.
    
- Implementaion

     ```C++
    void cpu::init_idle_thread()
    {
        running_since = osv::clock::uptime::now();
        std::string name = osv::sprintf("idle%d", id);
        idle_thread = thread::make([this] { idle(); }, thread::attr().pin(this).name(name));
        idle_thread->set_priority(thread::priority_idle);
    }
    ```
    ```C++
    void cpu::load_balance()
    {
        notifier::fire();
        timer tmr(*thread::current());
        while (true) {
            tmr.set(osv::clock::uptime::now() + 100_ms);
            thread::wait_until([&] { return tmr.expired(); });
            if (runqueue.empty()) {
                continue;
            }
            auto min = *std::min_element(cpus.begin(), cpus.end(),
                    [](cpu* c1, cpu* c2) { return c1->load() < c2->load(); });
            if (min == this) {
                continue;
            }
            // This CPU is temporarily running one extra thread (this thread),
            // so don't migrate a thread away if the difference is only 1.
            if (min->load() >= (load() - 1)) {
                continue;
            }
            WITH_LOCK(irq_lock) {
                ...
            }
        }
    }
    ```
- Preemptive

    OSv fully supports preemptive thread scheduling: threads can automatically cause thread scheduling by waiting, yielding or waking up other threads, or they can be preempted by other interrupts at any time. The thread can also temporarily avoid being preempted by increasing the preempt-disable counter.
   
- Implementaion 

    ```C++
    void thread::yield(thread_runtime::duration preempt_after)
    {
        trace_sched_yield();
        auto t = current();
        std::lock_guard<irq_lock_type> guard(irq_lock);
        // FIXME: drive by IPI
        cpu::current()->handle_incoming_wakeups();
        // FIXME: what about other cpus?
        if (cpu::current()->runqueue.empty()) {
            return;
        }
        assert(t->_detached_state->st.load() == status::running);
        // Do not yield to a thread with idle priority
        thread &tnext = *(cpu::current()->runqueue.begin());
        if (tnext.priority() == thread::priority_idle) {
            return;
        }
        trace_sched_yield_switch();
        ...
    }
    ```
    ```C++
    void thread::wait()
    {
        trace_sched_wait();
        cpu::schedule();
        trace_sched_wait_ret();
    }
    ```
- Tick-less
 
    Most of the OS kernel use a periodic timer interrupt, also known as tick which periodically causes thread scheduling, such as 100 times per second, the kernel determines the next scheduling by counting the execution time of each thread in each cycle[1]. However, excessive Tick wastes CPU time, especially on VM whose interrupt response is significantly slower than that on physical machines, because the virtual machine interrupts need to exit the hypervisor for execution. 
    OSv is designed to implement a tickless method: the scheduler uses a high-precision clock to calculate the accurate execution time of each thread, instead of roughly estimating the execution time by tick. In addition, the thread scheduler uses hysteresis to avoid frequent switch scheduling between two busy threads.
    
- Implementaion 

    ```C++
    void thread::cputime_estimator_set(
        osv::clock::uptime::time_point running_since,
        osv::clock::uptime::duration total_cpu_time)
    {
        u32 rs = running_since.time_since_epoch().count() >> cputime_shift;
        u32 tc = total_cpu_time.count() >> cputime_shift;
        _cputime_estimator.store(rs | ((u64)tc << 32), std::memory_order_relaxed);
    }
    ```
- Fair

    The OSv scheduler calculates the exponential decay moving average of the recent running time of each thread. The scheduler selects the thread with the lowest moving average as the thread to run next, and calculates the running time to ensure that the running time of the next thread is not exceeded.
    
- Implementaion

    ```C++
    void cpu::reschedule_from_interrupt(bool called_from_yield,
                                    thread_runtime::duration preempt_after)
    {
        #endif
            trace_sched_sched();
            assert(sched::exception_depth <= 1);
            need_reschedule = false;
            handle_incoming_wakeups();

        auto now = osv::clock::uptime::now();
        auto interval = now - running_since;
        running_since = now;
        if (interval <= 0) {
            // During startup, the clock may be stuck and we get zero intervals.
            // To avoid scheduler loops, let's make it non-zero.
            // Also ignore backward jumps in the clock.
            interval = context_switch_penalty;
        }
        thread* p = thread::current();

        const auto p_status = p->_detached_state->st.load();
        assert(p_status != thread::status::queued);

        p->_total_cpu_time += interval;
        p->_runtime.ran_for(interval);

        if (p_status == thread::status::running) {
            // The current thread is still runnable. Check if it still has the
            // lowest runtime, and update the timer until the next thread's turn.
        ...
        } else {
            // p is no longer running, so we'll switch to a different thread.
            // Return the runtime p borrowed for hysteresis.
            p->_runtime.hysteresis_run_stop();
        }

        auto ni = runqueue.begin();
        auto n = &*ni;
        runqueue.erase(ni);
        n->cputime_estimator_set(now, n->_total_cpu_time);
        assert(n->_detached_state->st.load() == thread::status::queued);
        trace_sched_switch(n, p->_runtime.get_local(), n->_runtime.get_local());

        ...
    }
    ```
- Scalable
    
    The time complexity of the OSv scheduler to maintain each CPU running thread queue is O(lgN): it is implemented by sorting the heap thread queue according to the thread moving average running time. Since the calculation of moving average time requires floating-point operations, the biggest obstacle is scalability. It is impractical for the scheduler to update the moving average running time of all threads each time, so the scheduler only updates the running time of one thread at a time[1].
    
- Efficient
    
    OSv single address space feature helps to improve the efficiency because the switch context does not need to switch page table and flush TLB which saves the cost. Besides, the x86 64 ABI guarantees that the FPU registers are caller-saved. So for voluntary context switches, OSv can skip saving the FPU state[1]. It reduces the cost of saving FPU register when doing swith context.
    
## File system

Quote on quote, in their paper it says, “For its filesystem support, OSv follows a traditional UNIX-like VFS (virtual filesystem) design and adopts ZFS as its major filesystem.”. However, as a group of students who barely knows what a virtual filesystem is before this semester, it still worth diving into the code and understanding all underlying code logic. 

### VFS
The virtual filesystem codes themselves are located at `fs/vfs`. To begin, let's dive into `vfs.h` first. Here is a brief look of the header file.

``` C++
int	 sys_open(char *path, int flags, mode_t mode, struct file **fp);
int	 sys_read(struct file *fp, const struct iovec *iov, size_t niov,
		off_t offset, size_t *count);
int	 sys_write(struct file *fp, const struct iovec *iov, size_t niov,
		off_t offset, size_t *count);
int	 sys_lseek(struct file *fp, off_t off, int type, off_t * cur_off);
```

Those functions cannot be more familiar to any Computer Science student. They are not interesting and no strangers, programmers use them day and night. Nearly identical functions exist in Unix system. Now, one step further, let us jump over to `vfs_syscalls.cc` and see `sys_mknod` as another example. 

``` C++
int
sys_mknod(char *path, mode_t mode)
{
	char *name;
	struct dentry *dp, *ddp;
	int error;

	... // Skip some unintersting code seg

	vn_lock(ddp->d_vnode);
	if ((error = vn_access(ddp->d_vnode, VWRITE)) != 0)
		goto out;
	if (S_ISDIR(mode))
		error = VOP_MKDIR(ddp->d_vnode, name, mode);
	else
		error = VOP_CREATE(ddp->d_vnode, name, mode);
 out:
	vn_unlock(ddp->d_vnode);
	drele(ddp);
	return error;
}
```

Now, we can see it seems like actual operation of sys_mknod is done through calling the macro `VOP_MKDIR` and ` VOP_CREATE` which located in `include/osv/vnode.h` depends on the mode. 

``` C++
#define VOP_CREATE(DVP, N, M) ((DVP)->v_op->vop_create)(DVP, N, M)
#define VOP_MKDIR(DVP, N, M) ((DVP)->v_op->vop_mkdir)(DVP, N, M)
```

By the numbers of arrows in this macro and the placement of parentheses, any programmer can tell this defines a function pointer as macro. Function pointer, as its name, is a pointer to the function and it enables polymorphism in programming. In other words, OSv can have different underlying filesystem support which the same VFS API layer on top. The system can be configurated to support different filesystem without any major changes in code and having a consistent API design.
However, this isn’t new, as the quote at the beginning of this paragraphs says, UNIX has this design, it is a traditional design. 

Let’s try harder and see is other any other interesting things can be found under `fs/`

```
fs/sysfs
fs/pseudofs 
fs/virtiofs
fs/ramfs
fs/rofs
```
`sysfs` and `pseudofs` aren’t so interesting, if you are a daily Linux user, you should be familiar with them, so does the `devfs`. Those pseudofs exposing either physical devices or system components as files so users can better interact with them. 

`ramfs` is another boring part, using memory devices to simulate a blockdev is very simple. It can be done by just using an array. Providing a full feature filesystem on top is not much harder if there is a proper data structure design. Here are some parts of their code. It does not worth spending time talking about. 

 ``` C++
    np = (ramfs_node *) malloc(sizeof(struct ramfs_node));

    uint64_t file_offset = uio->uio_offset;
    for (; segment_to_read_or_write != file_segments_by_offset.end() && bytes_to_read_or_write > 0; segment_to_read_or_write++) {
        // First calculate where we need to start in this segment ...
        auto offset_in_segment = file_offset - segment_to_read_or_write->first;
        // .. then calculate how many bytes to read or write in this segment
        auto maximum_bytes_to_read_or_write_in_segment = segment_to_read_or_write->second.size - offset_in_segment;
        auto bytes_to_read_or_write_in_segment = std::min<uint64_t>(bytes_to_read_or_write,maximum_bytes_to_read_or_write_in_segment);
        assert(offset_in_segment >= 0 && offset_in_segment < segment_to_read_or_write->second.size);
        auto ret = uiomove(segment_to_read_or_write->second.data + offset_in_segment, bytes_to_read_or_write_in_segment, uio);
        if (ret) {
            return ret;
        }
        bytes_to_read_or_write -= bytes_to_read_or_write_in_segment;
        file_offset += bytes_to_read_or_write_in_segment;
    }

```

`virtiofs` does capture our eyeballs. For those who have never heard of it, `virtio` is a virtualization standard for network and disk device drivers where just the guest's device driver "knows" it is running in a virtual environment, and cooperates with the hypervisor [x]. In short, `virtio` enabling faster data exchange and interaction between host and guest. This is very useful when you need to transfer huge amount of data or many invocations between host and guest. In OSv, they implement `virtiofs` by using FUSE (Filesystem in Userspace) to enabling data exchange between host and guest (ref)[https://github.com/cloudius-systems/osv/commit/bae4381d1d0558b7a684294e9203864f9652395c#diff-8d4a85b0cc195b35a41f49935fab84f00bc38e5e6b62ded8e1340dd6611fc801].

Thus, the filesystem part of OSv is boring because there is nearly no novel techniques being applied. They did implement some handy features based on modern techniques such as virtio but only in a small portion. 



## Network

### Problems of traditional network stack

- Too many layers in sending and receiving directions and extra data copies

     When a packet arrives, NIC will send a signal to CPU to interrupt it, this is generally a quick interrupt to send that packet into kernel. Then, there is usually a software interrupt to really begin to handle that packet, going through the incoming direction network stack. At one stage, there is a copy operation, moving the packet’s data from a kernel buffer into user space buffer. We need to notice this is a big overhead considering that there are tons of packets going into kernel and each of them needs to copy a chunk of data from kernel space into user buffer. Visiting/coping big chunk of memory frequently costs a lot. Finally, there will also be a context switch from kernel to user level in order to process that packet. Also, this context switch is a heavy operation because it will touch lots of kernel data structures, thus probably visiting memory rather than caches. What’s worse, probably the kernel processing part runs in one CPU core, the user thread happens to run in another core, which is bad for CPU locality and causes cache penalty.

- Lock contention problem

    Think of a situation. A web server is listening a socket, it uses this socket to accept requests and then do response for each request. The sever not only needs to receives packets into the socket, but also sends data back to clients. Both directions need to operate the socket, thus it is necessary to use locks to make sure data consistency. Locks are also expensive because it will also result a bad cache usage. And there are kinds of locks in the kernel network stack. 

    ![picture-1](resources/1.png?raw=true)
    <center>Figure 1</center>

- Networking data structure problem

    There are lots of linked lists (queues, like receiver queue and sender queue) in network stacks. As we know, linked list is bad for caching. 


These three reasons make it inefficient process network packets. In order to provide a high performance  networking, OSv implements the network channels first proposed by Van Jacobson in 2006. 

   ![picture-2](resources/2.png?raw=true)
    <center>Figure 2</center>


- Context switch problem is solved by the single address model, where all system calls are converted into normal calls. Also, OSv removes layers in receiving direction, instead replacing it with a simple classifier.

    ```C++
    class classifier {
    public
        ...
        void add(ipv4_tcp_conn_id id, net_channel* channel);
        void remove(ipv4_tcp_conn_id id)
        ...
        bool post_packet(mbuf* m);
        ...
        ipv4_tcp_channels _ipv4_tcp_channels;
    };


    struct ipv4_tcp_conn_id {
        ...
        in_addr src_addr;
        in_addr dst_addr;
        in_port_t src_port;
        in_port_t dst_port;

        size_t hash() const {
            // FIXME: protection against hash attacks?
            return src_addr.s_addr ^ dst_addr.s_addr ^ src_port ^ dst_port;
        }
        bool operator==(const ipv4_tcp_conn_id& x) const {
            return src_addr == x.src_addr
                && dst_addr == x.dst_addr
                && src_port == x.src_port
                && dst_port == x.dst_port;
        }
    };
    ``` 
    The classifier’s job is very simple. From the source code we can see, it just hashes each networking flow into different channels, thus sending different packet into different application threads. As a result, the kernel network stack is much more simplified. 

    Data copy overhead is also eliminated in this model because, first,  both kernel and user share the same address, there is no need to specially distinguish kernel and user space, we can view the application as part of kernel code actually; secondly, packets are directly forwarded into user thread, kernel does not process packets anymore. Since kernel does not process packets, the kernel code could be very efficient and quickly switches into user thread. This means, packets will be processed nearly just in user thread, thus in one CPU core.  CPU locality can be fully utilized. 

- Lock contention problem is avoided because the kernel receiver side doesn’t process heavy logic. And within the user thread, because only one thread is processing socket receive buffer and socket send buffer, those two locks can be merged. Also, TCP layer lock can be combined with socket layer lock because TCP processing now is combined into socket processing context (in one thread).

- Linked list now is replaced with channels. A channel is just a fixed-size ring buffer queue, from the source code we can see the size of 256. Like an array, it is very friendly for cacheline. Mbufs can be quickly produced and consumed with little cache penalty.

    ```C++
    class net_channel {
    private:
        std::function<void (mbuf*)> _process_packet;
        ring_spsc<mbuf*, 256> _queue;
        sched::thread_handle _waiting_thread CACHELINE_ALIGNED;
        // extra list of threads to wake
        osv::rcu_ptr<std::vector<pollreq*>> _pollers;
        osv::rcu_hashtable<epoll_ptr> _epollers;
        mutex _pollers_mutex;
        ...
        // producer: try to push a packet
        bool push(mbuf* m) { return _queue.push(m); }
        // consumer: wake the consumer (best used after multiple push()s)
        void wake() {
            _waiting_thread.wake();
            if (_pollers || !_epollers.empty()) {
                wake_pollers();
            }
        }
        // consumer: consume all available packets using process_packet()
        void process_queue();
        ...
    }
    ```
## References

[1] Avi Kivity, Dor Laor, Glauber Costa, Pekka Enberg, Nadav Har’El, Don Marti,
and Vlad Zolotarov. Osv—optimizing the operating system for virtual machines.
In 2014 USENIX Annual Technical Conference (USENIX ATC 14), pages 61–72,
Philadelphia, PA, June 2014. USENIX Association.

[2] T. Goethals, M. Sebrechts, A. Atrey, B. Volckaert, and F. De Turck. Unikernels vs
containers: An in-depth benchmarking study in the context of microservice applica-
tions. In2018 IEEE 8th International Symposium on Cloud and Service Computing
(SC2), pages 1–8, 2018.
[3] FRIEBEL, T., AND BIEMUELLER, S. How to deal with
lock holder preemption. Xen Summit North America
(2008).

# Answer to questions
- What are the "modules" of the system (see early lectures), and how do they relate? Where are isolation boundaries present? How do the modules communicate with each other? What performance implications does this structure have?

    The system consists of different components (modules), including Virtual hardware drivers, Filesystem,   Network stack, Thread scheduler, etc. The boundaries is clear among these components since each component has its own specific functionality and generally does not rely on other components’ design and implementation. For example, the thread module provides thread abstraction, and the virtual hardware driver focuses on abstractions of different hardware, like NICs and disks, they have little in common. However, hierarchy relations also exist in the system. The virtual hardware drivers needs to provide interfaces for some upper components. For example, disk drivers provides interfaces for the filesystem to read/write to virtual disks and the communication is direct calls to interfaces. Direct calls means the operations will be very fast, thus good performance.
- What are the core abstractions that the system aims to provide. How and why do they depart from other systems?

    We think the core abstraction of OSv is to provide a VM based application with a strong isolation, as talked in the overview part. OSv is different from Linux as it does not support process, thus it can nearly be viewed as a single application. It is also different from other hardware based unikernels because those kernels cannot provide strong isolation and have problems of dealing with kinds of device drivers. OSv only needs to support limited number of drivers and because it is within a VM, strong isolation is naturally guaranteed.

- In what conditions is the performance of the system "good" and in which is it "bad"? How does its performance compare to a Linux baseline (this discussion can be quantitative or qualitative)?

    The system should be in good performance when dealing with tasks that has frequent Linux system calls and heavy networking, because, first, it removes the heavy system calls in Linux and replaces them with direct function calls, thus applications that rely on this gain a lot; second, network stack in OSv is much more simplified as we have discussed above, so applications like microservices should gain performance. The benchmark using Memcached support this idea, as it shows that "OSv was able to handle about 20% more requests per second than the same memcached version on Linux". The system performs not good in applications with heavy disk I/O operations compared with Linux  because the coarse-grained locking in VFS could lock the vnode for a long time.
