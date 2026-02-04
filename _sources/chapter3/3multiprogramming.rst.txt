管理多道程序
=========================================


而内核为了管理任务，需要维护任务信息，相关内容包括：

- 任务运行状态：未初始化、准备执行、正在执行、已退出
- 任务控制块：维护任务状态和任务上下文
- 任务相关系统调用：程序主动暂停 ``sys_sched_yield`` 和主动退出 ``sys_exit``


yield 系统调用
-------------------------------------------------------------------------


.. image:: multiprogramming.png

上图描述了一种多道程序执行的典型情况。其中横轴为时间线，纵轴为正在执行的实体。
开始时，蓝色应用向外设提交了一个请求，外设随即开始工作，
但是它要一段时间后才能返回结果。蓝色应用于是调用 ``sys_sched_yield`` 交出 CPU 使用权，
内核让绿色应用继续执行。一段时间后 CPU 切换回蓝色应用，发现外设仍未返回结果，
于是再次 ``sys_sched_yield`` 。直到第二次切换回蓝色应用，外设才处理完请求，于是蓝色应用终于可以向下执行了。

我们还会遇到很多其他需要等待其完成才能继续向下执行的事件，调用 ``sys_sched_yield`` 可以避免等待过程造成的资源浪费。

.. code-block:: rust
    :caption: 第三章新增系统调用（一）

    /// 功能：应用主动交出 CPU 所有权并切换到其他应用。
    /// 返回值：总是返回 0。
    /// syscall ID：124
    fn sched_yield() -> isize;

用户库对应的实现和封装：

.. code-block:: rust
    
    // tg-syscall/src/user.rs

    pub fn sched_yield() -> isize {
        // SAFETY: 无参数系统调用
        unsafe { syscall0(SyscallId::SCHED_YIELD) }
    }


任务控制块与任务运行状态
---------------------------------------------------------

由于第三章需要同时运行多个程序，我们不能再像第二章那样，把程序内的信息全堆在 ``main.rs`` 中了。于是，本章节新增了一个 ``task.rs`` 来统一管理每个程序各自的信息。这样的数据结构名为 **任务控制块** (Task Control Block) 。

.. code-block:: rust

    // ch3/src/task.rs

    pub struct TaskControlBlock {
        ctx: LocalContext,
        pub finish: bool,
        stack: [usize; 256],
    }


它包含第二章提到的上下文、用户栈的信息，还新增了一个 ``pub finish: bool`` 用于标识当前任务是否以运行完成。此外，之前的上下文初始化移到了 ``TaskControlBlock::init(entry)`` 中，执行过程移到了 ``TaskControlBlock::execute()`` ，而处理 syscall 的流程则移动到 ``TaskControlBlock::handle_syscall()`` 中。

有了这些封装之后，我们就可以将所有用户程序塞进一个数组里，用于之后的执行；

.. code-block:: rust

    // ch3/src/main.rs

    // 任务控制块
    let mut tcbs = [TaskControlBlock::ZERO; APP_CAPACITY];
    let mut index_mod = 0;
    // 初始化
    for (i, app) in tg_linker::AppMeta::locate().iter().enumerate() {
        let entry = app.as_ptr() as usize;
        log::info!("load app{i} to {entry:#x}");
        tcbs[i].init(entry);
        index_mod += 1;
    }

任务控制块非常重要。在内核中，它就是应用的管理单位。后面的章节中它会化身为 ``struct Process`` ，并增加更多内容，不过这就是另一个故事了。

实现 sys_sched_yield
----------------------------------------------------------------------------

这个系统调用并不需要对单个程序内部做任何操作，因此它的实现和 ``sys_exit`` 一样，直接按规范返回 0。

.. code-block:: rust

    // ch3/src/main.rs

    impl Process for SyscallContext {
        #[inline]
        fn exit(&self, _caller: tg_syscall::Caller, _status: usize) -> isize {
            0
        }
    }

    impl Scheduling for SyscallContext {
        #[inline]
        fn sched_yield(&self, _caller: tg_syscall::Caller) -> isize {
            0
        }
    }

它和 ``sys_exit`` 的实际差别体现在 ``TaskControlBlock::handle_syscall()`` 中。任务控制块预先定义了几种“调度事件”，以此将 syscall 的执行结果返回给更上层的（接下来要介绍的）任务管理器。

调度事件分别是：
- ``None`` ， 表示处理了一个常规 Syscall；
- ``Yield``，表示处理了一个 ``sys_sched_yield``；
- ``Exit(usize)``，表示处理了一个 ``sys_exit``，并包含返回值；
- ``UnsupportedSyscall(SyscallId)``，表示遇到本章节不支持的 syscall。

.. code-block:: rust

    // ch3/src/task.rs
    /// 调度事件。
    pub enum SchedulingEvent {
        None,
        Yield,
        Exit(usize),
        UnsupportedSyscall(SyscallId),
    }

    impl TaskControlBlock {
    /// 处理系统调用，返回是否应该终止程序。
    pub fn handle_syscall(&mut self) -> SchedulingEvent {
        use tg_syscall::{SyscallId as Id, SyscallResult as Ret};
        use SchedulingEvent as Event;

        let id = self.ctx.a(7).into();
        let args = [ ... ];
        match tg_syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
            Ret::Done(ret) => match id {
                Id::EXIT => Event::Exit(self.ctx.a(0)),
                Id::SCHED_YIELD => {
                    *self.ctx.a_mut(0) = ret as _;
                    self.ctx.move_next();
                    Event::Yield
                }
                _ => {
                    *self.ctx.a_mut(0) = ret as _;
                    self.ctx.move_next();
                    Event::None
                }
            },
            Ret::Unsupported(_) => Event::UnsupportedSyscall(id),
        }
    }

任务管理器
--------------------------------------

内核需要一个全局的任务管理器来管理这些任务控制块，并根据它们执行时的结果进行调度。
本章节比较偷懒，直接写在了 ``main.rs`` 中。它的外层框架如下

.. code-block:: rust
    :linenos:

    // ch3/src/main.rs

    // 多道执行
    let mut remain = index_mod;
    let mut i = 0usize;
    while remain > 0 {
        let tcb = &mut tcbs[i];
        if !tcb.finish {
            loop {
                #[cfg(not(feature = "coop"))]
                tg_sbi::set_timer(time::read64() + 12500);
                unsafe { tcb.execute() };

                use scause::*;
                let finish = {
                    ... => continue,
                    ... => ...
                };
                if finish {
                    tcb.finish = true;
                    remain -= 1;
                }
                break;
            }
        }
        i = (i + 1) % index_mod;
    }

- 第 2 行：记录还未结束的程序个数。
- 第 4~6 行与 24 行：循环选择未结束的程序并运行
- 第 8~9 行：设置时间中断（下一节介绍），先略过。
- 第 13~21 行：检查当前程序运行状态，如走到 continue 分支则继续运行当前程序，否则暂停并运行下一个程序。并顺便标记当前程序是否结束。

而内层判断 ``finish`` 的部分如下：

.. code-block:: rust
    :linenos:

    // ch3/src/main.rs

    let finish = match scause::read().cause() {
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            tg_sbi::set_timer(u64::MAX);
            log::trace!("app{i} timeout");
            false
        }
        Trap::Exception(Exception::UserEnvCall) => {
            use task::SchedulingEvent as Event;
            match tcb.handle_syscall() {
                Event::None => continue,
                Event::Exit(code) => {
                    log::info!("app{i} exit with code {code}");
                    true
                }
                Event::Yield => {
                    log::debug!("app{i} yield");
                    false
                }
                Event::UnsupportedSyscall(id) => {
                    log::error!("app{i} call an unsupported syscall {}", id.0);
                    true
                }
            }
        }
        Trap::Exception(e) => {
            log::error!("app{i} was killed by {e:?}");
            true
        }
        Trap::Interrupt(ir) => {
            log::error!("app{i} was killed by an unexpected interrupt {ir:?}");
            true
        }
    };

- 第 4~8 行：触发时钟中断（下一节介绍），当前程序未结束，切换下一个程序。
- 第 9~26 行：是系统调用，此时需通过 ``TaskControlBlock::handle_syscall()`` 完成系统调用并处理完成情况。注意此后的返回值 **都是任务控制块自己定义的返回值，不是 syscall 规范中的返回值** 。
    - 第 12 行：系统调用正常完成，当前程序未结束，且继续运行。
    - 第 13~16 行：已触发 ``sys_exit``，标记当前程序结束。
    - 第 17~20 行：已触发 ``sys_sched_yield``，当前程序未结束，切换下一个程序。
    - 第 21~24 行：遇到不支持的 syscall，标记当前程序结束。
- 第 27~30 行：遇到异常，标记当前程序结束。
- 第 31~34 行，遇到不支持的中断，标记当前程序结束。

小结
--------------------------------------

我们首先通过任务控制块的包装，把加载进来的所有用户程序塞进一个数组里，然后通过实现了一个简易任务管理器，控制它们轮流运行。
在运行过程中，用户程序可以调用 ``sys_sched_yield`` 来提示内核将当前任务暂时挂起，切换到其他任务执行。而内核则通过调度事件，将这一信息传递给上层的任务管理器，实现调度。

但是，如果有程序完全不使用 ``sys_sched_yield`` ，就能长时间霸占 CPU 直到运行结束。
有没有什么办法无需依赖主动的系统调用，而是让程序被动让出 CPU 呢？
下一节我们将介绍上面几段代码中埋下的伏笔——时钟中断机制。