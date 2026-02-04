引言
========================================

本章导读
--------------------------


本章的目标是实现分时多任务系统，它能并发地执行多个用户程序，并调度这些程序。为此需要实现

- 一次性加载所有用户程序，减少任务切换开销；
- 支持任务切换机制，保存切换前后程序上下文；
- 支持程序主动放弃处理器，实现 yield 系统调用；
- 以时间片轮转算法调度用户程序，实现资源的时分复用。


实践体验
-------------------------------------

在 ``rCore-Tutorial-in-single-workspace`` 项目中运行本章代码（ ``DEBUG`` 类型的调试输出过多，建议使用 ``LOG=INFO`` 过滤输出）：

.. code-block:: console
   
   $ LOG=INFO cargo qemu --ch 3

运行后，看到用户程序交替输出信息：

.. code-block::

   [rustsbi] RustSBI version 0.3.1, adapting to RISC-V SBI v1.0.0
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
   [rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
   [rustsbi] Platform Name      : riscv-virtio,qemu
   [rustsbi] Platform SMP       : 1
   [rustsbi] Platform Memory    : 0x80000000..0x84000000
   [rustsbi] Boot HART          : 0
   [rustsbi] Device Tree Region : 0x83e00000..0x83e010f6
   [rustsbi] Firmware Address   : 0x80000000
   [rustsbi] Supervisor Address : 0x80200000
   [rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
   [rustsbi] pmp02: 0x80000000..0x80200000 (---)
   [rustsbi] pmp03: 0x80200000..0x84000000 (xwr)
   [rustsbi] pmp04: 0x84000000..0x00000000 (-wr)

      ______                       __
   / ____/___  ____  _________  / /__
   / /   / __ \/ __ \/ ___/ __ \/ / _ \
   / /___/ /_/ / / / (__  ) /_/ / /  __/
   \____/\____/_/ /_/____/\____/_/\___/
   ====================================
   [ INFO] LOG TEST >> Hello, world!
   [ WARN] LOG TEST >> Hello, world!
   [ERROR] LOG TEST >> Hello, world!

   [ INFO] load app0 to 0x80400000
   [ INFO] load app1 to 0x80600000
   [ INFO] load app2 to 0x80800000
   [ INFO] load app3 to 0x80a00000
   [ INFO] load app4 to 0x80c00000
   [ INFO] load app5 to 0x80e00000
   [ INFO] load app6 to 0x81000000
   [ INFO] load app7 to 0x81200000
   [ INFO] load app8 to 0x81400000
   [ INFO] load app9 to 0x81600000
   [ INFO] load app10 to 0x81800000
   [ INFO] load app11 to 0x81a00000

   Hello, world from user mode program!
   [ INFO] app0 exit with code 0
   Into Test store_fault, we will insert an invalid store operation...
   Kernel should kill this application!
   [ERROR] app1 was killed by StoreFault
   Try to execute privileged instruction in U Mode
   Kernel should kill this application!
   [ERROR] app3 was killed by IllegalInstruction
   Try to access privileged CSR in U Mode
   Kernel should kill this application!
   [ERROR] app4 was killed by IllegalInstruction
   AAAAAAAAAA [1/5]
   BBBBBBBBBB [1/2]
   AAAAAAAAAA [2/5]
   BBBBBBBBBB [2/2]
   CCCCCCCCCC [1/3]
   3^10000=5079(MOD 10007)
   AAAAAAAAAA [3/5]
   Test write B OK!
   [ INFO] app6 exit with code 0
   CCCCCCCCCC [2/3]
   power_3 [10000/200000]
   power_5 [10000/140000]
   power_7 [10000/160000]
   3^20000=8202(MOD 10007)
   AAAAAAAAAA [4/5]
   CCCCCCCCCC [3/3]
   power_3 [20000/200000]
   power_5 [20000/140000]
   power_7 [20000/160000]
   3^30000=8824(MOD 10007)
   AAAAAAAAAA [5/5]
   Test write C OK!
   [ INFO] app7 exit with code 0
   power_3 [30000/200000]
   power_5 [30000/140000]
   power_7 [30000/160000]
   3^40000=5750(MOD 10007)
   Test write A OK!
   [ INFO] app5 exit with code 0
   power_3 [40000/200000]
   power_5 [40000/140000]
   power_7 [40000/160000]
   3^50000=3824(MOD 10007)
   power_3 [50000/200000]
   power_5 [50000/140000]
   power_7 [50000/160000]
   3^60000=8516(MOD 10007)
   power_3 [60000/200000]
   power_5 [60000/140000]
   power_7 [60000/160000]
   3^70000=2510(MOD 10007)
   power_3 [70000/200000]
   power_5 [70000/140000]
   power_7 [70000/160000]
   3^80000=9379(MOD 10007)
   power_3 [80000/200000]
   power_5 [80000/140000]
   power_7 [80000/160000]
   3^90000=2621(MOD 10007)
   power_3 [90000/200000]
   power_5 [90000/140000]
   power_7 [90000/160000]
   3^100000=2749(MOD 10007)
   Test power OK!
   [ INFO] app2 exit with code 0
   power_3 [100000/200000]
   power_5 [100000/140000]
   power_7 [100000/160000]
   power_3 [110000/200000]
   power_5 [110000/140000]
   power_7 [110000/160000]
   power_3 [120000/200000]
   power_5 [120000/140000]
   power_7 [120000/160000]
   power_3 [130000/200000]
   power_5 [130000/140000]
   power_7 [130000/160000]
   power_3 [140000/200000]
   power_5 [140000/140000]
   5^140000 = 386471875(MOD 998244353)
   Test power_5 OK!
   [ INFO] app9 exit with code 0
   power_7 [140000/160000]
   power_3 [150000/200000]
   power_7 [150000/160000]
   power_3 [160000/200000]
   power_7 [160000/160000]
   7^160000 = 667897727(MOD 998244353)
   Test power_7 OK!
   [ INFO] app10 exit with code 0
   power_3 [170000/200000]
   power_3 [180000/200000]
   power_3 [190000/200000]
   power_3 [200000/200000]
   3^200000 = 871008973(MOD 998244353)
   Test power_3 OK!
   [ INFO] app8 exit with code 0
   Test sleep OK!
   [ INFO] app11 exit with code 0


本章代码树
---------------------------------------------

.. code-block::

   ├── ch3（内核实现）
   │   ├── Cargo.toml（配置文件）
   │   └── src（内核源代码）
   │       ├── main.rs（内核主函数，包括系统调用接口实现）
   │       └── task.rs（任务控制块）
   ├── tg-syscall（系统调用模块）
   │   └── src
   │       ├── kernel/mod.rs（内核端系统调用接口定义，无需修改）
   │       ├── user.rs（用户端系统调用，无需修改）
   │       └── syscall.h.in（系统调用号，无需修改）
   ├── user（用户程序）
   │   └── src/bin（测试用例，无需修改）
   ├── ...


.. 本章代码导读
.. -----------------------------------------------------

.. 本章的重点是实现对应用之间的协作式和抢占式任务切换的操作系统支持。与上一章的操作系统实现相比，有如下一些不同的情况导致实现上也有差异：

.. - 多个应用同时放在内存中，所以他们的起始地址是不同的，且地址范围不能重叠
.. - 应用在整个执行过程中会暂停或被抢占，即会有主动或被动的任务切换

.. 这些实现上差异主要集中在对应用程序执行过程的管理、支持应用程序暂停的系统调用和主动切换应用程序所需的时钟中断机制的管理。

.. 对于第一个不同情况，需要对应用程序的地址空间布局进行调整，每个应用的地址空间都不相同，且不能重叠。这并不要修改应用程序本身，而是通过一个脚本 ``build.py`` 来针对每个应用程序修改链接脚本 ``linker.ld`` 中的 ``BASE_ADDRESS`` ，让编译器在编译不同应用时用到的 ``BASE_ADDRESS`` 都不同，且有足够大的地址间隔。这样就可以让每个应用所在的内存空间是不同的。

.. 对于第二个不同情况，需要实现任务切换，这就需要在上一章的 ``trap`` 上下文切换的基础上，再加上一个 ``task`` 上下文切换，才能完成完整的任务切换。这里面的关键数据结构是表示应用执行上下文的 ``TaskContext`` 数据结构和具体完成上下文切换的汇编语言编写的 ``__switch`` 函数。一个应用的执行需要被操作系统管理起来，这是通过 ``TaskControlBlock`` 数据结构来表示应用执行上下文的动态过程和动态状态（运行态、就绪态等）。而为了做好应用程序第一次执行的前期初始化准备， ``TaskManager`` 数据结构的全局变量实例 ``TASK_MANAGER`` 描述了应用程序初始化所需的数据， 而 ``TASK_MANAGER`` 的初始化赋值过程是实现这个准备的关键步骤。

.. 应用程序可以在用户态执行后，还需要有新的系统调用 ``sys_yield`` 的实现来支持应用自己的主动暂停；还要添加对时钟中断的处理，来支持抢占应用执行的抢占式切换。有了时钟中断，就可以在一定时间内打断应用的执行，并主动切换到另外一个应用，这部分主要是通过对 ``trap_handler`` 函数中进行扩展，来完成在时钟中断产生时可能进行的任务切换。  ``TaskManager`` 数据结构的成员函数 ``run_next_task`` 来实现基于任务控制块的切换，并会具体调用 ``__switch`` 函数完成硬件相关部分的任务上下文切换。

.. 如果理解了上面的数据结构和相关函数的关系和相互调用的情况，那么就比较容易理解本章改进后的操作系统了。


.. .. [#prionosuchus] 锯齿螈身长可达9米，是迄今出现过的最大的两栖动物，是二叠纪时期江河湖泊和沼泽中的顶级掠食者。
.. .. [#eoraptor] 始初龙（也称始盗龙）是后三叠纪时期的两足食肉动物，也是目前所知最早的恐龙，它们只有一米长，却代表着恐龙的黎明。
.. .. [#coelophysis] 腔骨龙（也称虚形龙）最早出现于三叠纪晚期，它体形纤细，善于奔跑，以小型动物为食。
