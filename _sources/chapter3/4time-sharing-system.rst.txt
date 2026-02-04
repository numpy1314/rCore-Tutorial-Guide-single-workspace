分时多任务系统
===========================================================


现代的任务调度算法基本都是抢占式的，它要求每个应用只能连续执行一段时间，然后内核就会将它强制性切换出去。
一般将 **时间片** (Time Slice) 作为应用连续执行时长的度量单位，每个时间片可能在毫秒量级。
简单起见，我们使用 **时间片轮转算法** (RR, Round-Robin) 来调度应用。
这个算法轮流切换每一个应用，然后运行一个固定长度的时间片。


计时器
------------------------------------------------------------------

实现调度算法需要计时。在 RISC-V 架构下，处理器维护两个特权寄存器：  ``mtime`` 和 ``mtimecmp`` 。
计数器 ``mtime`` 的值会由处理器自行递增，一旦它超过了 ``mtimecmp``，就会触发一次时钟中断。

运行在机器态(M模式)的 SEE 已经预留了相应的接口。我们可以使用 ``crate riscv`` 提供的方法直接获取 ``mtime`` 寄存器的值。
在 ``main.rs`` 的 ``impl Clock for SyscallContext`` 中就有这样一个例子（第 173 行）；

.. code-block:: rust

    // ch3/src/main.rs

    ClockId::CLOCK_MONOTONIC => {
        let time = riscv::register::time::read() * 10000 / 125;
        *unsafe { &mut *(tp as *mut TimeSpec) } = TimeSpec {
            tv_sec: time / 1_000_000_000,
            tv_nsec: time % 1_000_000_000,
        };
        0
    }

其中，由于 qemu 的默认频率是 1.25MHz，所以从寄存器中读出的主频需要运算后才能得到以秒/纳秒为单位的时间。

类似地，例如我们想在 10 ms 后设置时钟中断，就可以参考在 ``main.rs`` 的 ``fn rust_main()`` 中的例子（第 57 行）；

.. code-block:: rust

    // ch3/src/main.rs

    tg_sbi::set_timer(time::read64() + 12500);

其中 ``time::read64()`` 读取当前时间， 12500 tick 正好对应 qemu 频率算出的 10 ms。

这个方法的底层实现要复杂一些，它实际上是内核态(S模式)向机器态(M模式)发送了一个 ``sbi_call``，陷入机器态处理（就像用户向内核发送 ``syscall`` 然后陷入内核处理）。

.. code-block:: rust

    // tg-sbi/src/lib.rs

    /// 设置下一次定时器中断的时间。
    pub fn set_timer(timer: u64) {
        sbi_call(SBI_EXT_TIMER, 0, timer as usize, 0, 0);
    }

随后， ``tg-sbi`` 在机器态通过 ``handle_timer(u64)`` 来处理这个请求：

.. code-block:: rust
    
    // tg-sbi/src/msbi.rs

    /// 处理 Timer 扩展（EID 0x54494D45）。
    fn handle_timer(time: u64) -> SbiRet {
        const CLINT_MTIMECMP: usize = 0x200_4000;
        // SAFETY: 向 QEMU virt 机器的已知 MMIO 地址写入 CLINT mtimecmp 寄存器。
        // 这将设置下一次定时器中断的触发时间。
        unsafe {
            (CLINT_MTIMECMP as *mut u64).write_volatile(time);
        }
        // 清除挂起的 S-mode 定时器中断
        // SAFETY: 修改 mip CSR 以清除 STIP 位是有效的 M-mode 操作。
        // 这是确认定时器中断所必需的。
        unsafe {
            asm!(
                "csrc mip, {}",
                in(reg) (1 << 5), // Clear STIP
            );
        }
        SbiRet::success(0)
    }

它主要完成两件事：

- 首先通过 MMIO 写入 ``mtimecmp`` 寄存器。 
- 然后写入 ``mip`` 的 ``STIP`` 标志位，相当于关掉“闹钟”，才能等待它下一次响起。

.. note::

    ``m_entry.asm`` 和 ``msbi.rs`` 中的代码处于机器态，并不引入 ``crate riscv`` ，因此只能通过 MMIO 写入定时器。MMIO 会在第六章中详细介绍，但简单来说， ``qemu`` 虚拟机启动时，会将一些寄存器映射到固定的内存地址上，所以代码中会有直接对着常量地址写入的操作。

.. hint::

    感兴趣的同学可以尝试阅读 ``tg-sbi/src/m_entry.asm`` 和 ``tg-sbi/src/msbi.rs``，了解机器态实现的细节。这些代码其实不属于内核。通常而言，内核不会自己手写一个 ``sbi`` ，而是使用 ``qemu`` 自带的 ``OpenSBI`` 或 `RustSBI <https://github.com/rustsbi/rustsbi-qemu/>`_ 等成熟方案。


时钟中断与抢占式调度
------------------------------------------------------------------

上文提到，一旦 ``mtime`` 的值超过 ``mtimecmp``，就会触发一次时钟中断。

在内核启动时，时钟中断是默认被屏蔽的。需要修改 ``sie`` 寄存器手动打开中断，如 ``main.rs`` 第 48 行：

.. code-block:: rust

    // ch3/src/main.rs

    // 打开中断
    unsafe { sie::set_stimer() };

类似 ``syscall`` ，时钟中断触发后也会陷入内核。不过此时异常中断的原因会被标记为 ``Trap::Interrupt(Interrupt::SupervisorTimer)``。 ``main.rs`` 第 54 行开始的代码演示了如何处理时钟中断：

.. code-block:: rust

    // ch3/src/main.rs

    if !tcb.finish {
        loop {
            #[cfg(not(feature = "coop"))]
            tg_sbi::set_timer(time::read64() + 12500);
            unsafe { tcb.execute() };

            use scause::*;
            let finish = match scause::read().cause() {
                Trap::Interrupt(Interrupt::SupervisorTimer) => {
                    tg_sbi::set_timer(u64::MAX);
                    log::trace!("app{i} timeout");
                    false
                }
                Trap::Exception(Exception::UserEnvCall) => {
                    ......
                }
            }
        }

如上述代码所示，除非命令专门指定了 ``coop`` 属性（合作调度），否则在应用正式通过 ``unsafe { tcb.execute() };`` 运行前就会设置一个 12500 tick(=10 ms) 的计时器。
在处理异常中断时，如果发现触发的是 ``Trap::Interrupt(Interrupt::SupervisorTimer)`` ，就说明当前应用已用完了它的时间片。于是我们标记当前应用未结束运行，然后让 ``loop {`` 去运行下一个应用。

结合上一小节的介绍，我们最终实现了时间片轮转任务调度算法。 ``power`` 系列用户程序可以验证我们取得的成果：这些应用并没有主动 yield，内核仍能公平地把时间片分配给它们。


clock_gettime 系统调用
-------------------------------------------------------------------------

至此，内核已经可以自由地获取时间、设置定时器。
但用户态(U模式)的应用并不能直接访问 ``mtime`` 等寄存器。
我们还需要新增一个系统调用，使应用获取当前的时间：

.. code-block:: rust
    :caption: 第三章新增系统调用（二）

    /// 功能：获取 clockid 对应计时器的时间，例如全局计时器 CLOCK_MONOTONIC 的编号为 1。然后，保存在 TimeSpec 结构体 tp 中。
    /// 返回值：返回是否执行成功，成功则返回 0
    /// syscall ID：113
    fn clock_gettime(clockid: ClockId, tp: *mut TimeSpec) -> isize;

    // tg-syscall/src/time.rs
    #[derive(Clone, Copy, PartialEq, PartialOrd, Eq, Ord, Debug)]
    #[repr(C)]
    pub struct TimeSpec {
        // seconds
        pub tv_sec: usize,
        // nanoseconds
        pub tv_nsec: usize,
    }

用户库对应的实现和封装：

.. code-block:: rust
    
    // user/src/lib.rs

    pub fn get_time() -> isize {
        let mut time: TimeSpec = TimeSpec::ZERO;
        clock_gettime(ClockId::CLOCK_MONOTONIC, &mut time as *mut _ as _);
        (time.tv_sec * 1000 + time.tv_nsec / 1_000_000) as isize
    }

上述 ``get_time`` 函数返回一个整数，表示系统启动后运行的毫秒数。


RISC-V 架构中的嵌套中断问题
-----------------------------------

默认情况下，当 Trap 进入某个特权级之后，在 Trap 处理的过程中同特权级的中断都会被屏蔽。

- 当 Trap 发生时，``sstatus.sie`` 会被保存在 ``sstatus.spie`` 字段中，同时 ``sstatus.sie`` 置零，
  这也就在 Trap 处理的过程中屏蔽了所有 S 特权级的中断；
- 当 Trap 处理完毕 ``sret`` 的时候， ``sstatus.sie`` 会恢复到 ``sstatus.spie`` 内的值。

也就是说，如果不去手动设置 ``sstatus`` CSR ，在只考虑 S 特权级中断的情况下，是不会出现 **嵌套中断** (Nested Interrupt) 的。

.. note::

    **嵌套中断与嵌套 Trap**

    嵌套中断可以分为两部分：在处理一个中断的过程中又被同特权级/高特权级中断所打断。默认情况下硬件会避免前一部分，
    也可以通过手动设置来允许前一部分的存在；而从上面介绍的规则可以知道，后一部分则是无论如何设置都不可避免的。

    嵌套 Trap 则是指处理一个 Trap 过程中又再次发生 Trap ，嵌套中断算是嵌套 Trap 的一种。
