进程管理的核心数据结构
===================================

本节导读
-----------------------------------

为了在内核中支持多进程机制，我们需要设计并实现一系列核心数据结构。这些结构体构成了操作系统管理进程的基石。在当前的实现版本中，我们对原有的设计进行了重构，引入了更模块化的组件库。主要涉及的数据结构包括：

- **应用链接/加载器**：基于 ``tg-linker`` 的应用元数据管理。
- **进程标识符 (ProcId)**：用于唯一标识进程的句柄。
- **异界上下文 (ForeignContext)**：管理用户态与内核态的上下文切换。
- **进程控制块 (Process)**：描述进程资源的核心结构。
- **任务管理器 (ProcManager)**：负责进程的存储与调度队列维护。
- **处理器管理结构 (Processor)**：提供对任务管理器的并发安全访问。

基于应用名的应用链接/加载器
------------------------------------------------------------------------

在实现 ``exec`` 系统调用时，操作系统需要根据应用程序的文件名（字符串）来加载对应的 ELF 可执行文件，而不仅仅是通过索引编号。

在之前的章节中，我们可能通过汇编代码手动嵌入字符串。而在当前的工程架构中，我们使用了 ``tg-linker`` 库来自动处理应用程序的链接和元数据生成。这大大简化了内核代码。

在 ``tg-ch5/src/main.rs`` 中，我们定义了一个全局的只读映射表 ``APPS``：

.. code-block:: rust
    :linenos:

    // tg-ch5/src/main.rs

    use spin::Lazy;
    use alloc::collections::BTreeMap;
    use core::ffi::CStr;

    /// 加载用户进程。
    /// 使用 BTreeMap 将应用名称映射到对应的 ELF 二进制数据切片。
    static APPS: Lazy<BTreeMap<&'static str, &'static [u8]>> = Lazy::new(|| {
        extern "C" {
            // 由链接脚本提供的符号，指向应用名称列表的起始地址
            static app_names: u8;
        }
        unsafe {
            // tg_linker::AppMeta::locate() 获取所有应用的数据段位置信息
            tg_linker::AppMeta::locate()
                .iter()
                .scan(&app_names as *const _ as usize, |addr, data| {
                    // 从内存中解析以 \0 结尾的 C 风格字符串
                    let name = CStr::from_ptr(*addr as _).to_str().unwrap();
                    // 更新地址指针，指向下一个字符串
                    *addr += name.as_bytes().len() + 1;
                    // 返回 (名称, 数据切片) 的键值对
                    Some((name, data))
                })
        }
        .collect() // 收集为 BTreeMap
    });

**实现解析：**

1.  **Lazy Initialization**：使用 ``spin::Lazy`` 确保 ``APPS`` 在第一次被访问时才进行初始化。这是因为构建 map 需要在运行时执行代码（解析内存），而不能在编译期完成。
2.  **AppMeta**：``tg_linker`` 自动收集了所有链接进内核的用户程序，并生成了 ``app_names`` 符号表。
3.  **Scan 与 Collect**：通过迭代器扫描内存中的字符串表，将其与 ELF 数据段关联，最终构建出一个 ``BTreeMap<&str, &[u8]>``。

有了这个结构，在 ``exec`` 系统调用中，我们只需要调用 ``APPS.get(name)`` 即可获取目标程序的数据，无需再手动遍历。

进程标识符与上下文管理
------------------------------------------------------------------------

进程标识符 (ProcId)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在操作系统中，PID (Process Identifier) 是进程的身份证。同一时间系统中存在的每个进程必须拥有唯一的 PID。

我们使用 ``tg-task-manage`` 库提供的 ``ProcId`` 类型来管理 PID：

.. code-block:: rust

    // 伪代码描述 ProcId 的行为

    pub struct ProcId(usize);

    impl ProcId {
        /// 分配一个新的、唯一的 PID
        pub fn new() -> Self { ... }

        /// 获取 PID 的数值
        pub fn get_usize(&self) -> usize { self.0 }
    }

    impl Drop for ProcId {
        fn drop(&mut self) {
            // 当 ProcId 离开作用域被销毁时，自动回收该 PID 数值以便复用
            pid_allocator.dealloc(self.0);
        }
    }

这种设计极大地简化了资源管理，我们不需要手动调用 ``free_pid`` 之类的函数，只要 ``Process`` 结构体被销毁，其中包含的 ``ProcId`` 字段就会自动触发回收逻辑。

异界上下文 (ForeignContext)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在早期的设计中，我们为每个进程分配一个独立的 **内核栈**，Trap 发生时硬件将栈指针切换到该进程的内核栈。然而，在本章的实现中，我们采用了更轻量级的 **单内核栈模型**（或者说是共享启动栈模型）。

内核本质上是一个循环运行的程序，它运行在启动时分配的全局栈上。当内核决定运行某个用户进程时，它执行一个“切换”操作，暂停内核当前的执行流，切换到用户态。

这个切换过程由 ``ForeignContext`` 管理：

.. code-block:: rust

    // 位于 tg-kernel-context 库中

    pub struct ForeignContext {
        /// 本地上下文，保存了通用的寄存器（ra, sp, s0-s11 等）
        pub context: LocalContext,
        /// 监管者地址翻译与保护寄存器，指向进程的页表根节点
        pub satp: usize,
    }

    impl ForeignContext {
        /// 执行该上下文。
        /// 这是一段汇编代码，它会：
        /// 1. 保存当前内核的寄存器到栈上。
        /// 2. 切换页表 (satp) 到用户地址空间。
        /// 3. 恢复用户寄存器。
        /// 4. 执行 sret 返回用户态。
        /// 
        /// 当用户态发生 Trap 时，会逆向执行上述过程，返回到 execute 调用的下一条指令。
        pub unsafe fn execute(&mut self);
    }

**为什么不再需要对每个进程分配一个独立的内核栈？**

因为在 ``ForeignContext::execute()`` 返回后，CPU 已经回到了内核地址空间，并且使用的是调用 ``execute`` 时的内核栈（即全局启动栈）。所有的 Trap 处理逻辑（系统调用处理、缺页异常处理）都直接在这个共享的内核栈上运行。

这种设计使得 ``Process`` 结构体不需要再拥有一个 ``kernel_stack`` 字段，减少了内存开销，也简化了进程创建的过程。

进程控制块 (Process)
------------------------------------------------------------------------

进程控制块 (PCB) 是操作系统感知进程存在的唯一凭证。在之前的章节中它被称为 ``TaskControlBlock``。现在，我们将它重命名为 ``Process``，以更准确地反映其语义。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/process.rs

    use tg_kernel_context::foreign::ForeignContext;
    use tg_kernel_vm::{AddressSpace, Sv39, Sv39Manager};
    use tg_task_manage::ProcId;

    /// 进程控制块
    pub struct Process {
        /// 进程标识符 (不可变)
        /// 负责 PID 的生命周期管理
        pub pid: ProcId,

        /// 异界上下文 (可变)
        /// 保存了进程暂停时的寄存器状态和页表指针
        pub context: ForeignContext,

        /// 地址空间 (可变)
        /// 管理进程的所有虚拟内存映射 (代码, 数据, 栈, 堆等)
        pub address_space: AddressSpace<Sv39, Sv39Manager>,

        /// 用户堆底 (可变)
        /// 用于 sbrk 系统调用，记录堆的起始地址
        pub heap_bottom: usize,

        /// 当前程序断点 (可变)
        /// 记录当前堆的结束位置 (Program Break)
        pub program_brk: usize,
    }

**结构体设计分析：**

1.  **无内部锁**：你会发现 ``Process`` 内部没有使用 ``Mutex`` 或 ``UPSafeCell``。这是因为我们将并发控制的粒度提升到了任务管理器层级。当内核获取一个进程进行执行时，是独占访问的。
2.  **AddressSpace**：使用了 ``tg-kernel-vm`` 提供的 ``AddressSpace``，它封装了 SV39 页表的操作。
3.  **资源聚合**：``Process`` 聚合了标识符 (PID)、计算资源 (Context) 和内存资源 (AddressSpace)。

``Process`` 提供了一些关键方法来支持进程的生命周期：

.. code-block:: rust

    impl Process {
        /// 从 ELF 文件创建新进程
        pub fn from_elf(elf: ElfFile) -> Option<Self> {
            // 1. 分配 PID
            // 2. 创建地址空间，解析 ELF 加载代码段和数据段
            // 3. 分配用户栈和异界传送门
            // 4. 初始化 ForeignContext，设置入口地址
            // ...
        }

        /// exec 系统调用实现：替换当前进程内容
        pub fn exec(&mut self, elf: ElfFile) {
            // 使用新程序创建临时进程实例
            let proc = Process::from_elf(elf).unwrap();
            // 用新实例的资源替换当前资源
            self.address_space = proc.address_space;
            self.context = proc.context;
            self.heap_bottom = proc.heap_bottom;
            self.program_brk = proc.program_brk;
            // 注意：pid 保持不变
        }

        /// fork 系统调用实现：复制当前进程
        pub fn fork(&mut self) -> Option<Process> {
            // 1. 分配新 PID
            let pid = ProcId::new();
            
            // 2. 深度拷贝地址空间 (Copy-on-Write 可在此处优化，目前为深拷贝)
            let mut address_space = AddressSpace::new();
            self.address_space.cloneself(&mut address_space);
            
            // 3. 必须重新映射异界传送门 (Trampoline)
            // 因为新页表需要访问内核的 Trap 处理入口
            crate::process::map_portal(&address_space);

            // 4. 复制上下文
            // 注意更新 satp 为新页表的物理页号
            let context = self.context.context.clone();
            let satp = (8 << 60) | address_space.root_ppn().val();
            
            Some(Self {
                pid,
                context: ForeignContext { context, satp },
                address_space,
                heap_bottom: self.heap_bottom,
                program_brk: self.program_brk,
            })
        }
    }

任务管理器 (ProcManager)
------------------------------------------------------------------------

内核需要一个容器来持有所有活跃的进程，并决定下一个运行哪个进程。本章引入了 ``ProcManager``，它利用 ``tg-task-manage`` 库提供的标准接口实现。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/processor.rs

    use alloc::collections::{BTreeMap, VecDeque};
    use tg_task_manage::{Manage, Schedule};
    use crate::process::Process;

    /// 任务管理器
    pub struct ProcManager {
        /// 任务存储：所有权持有者
        /// 使用 BTreeMap 根据 PID 索引进程实体
        tasks: BTreeMap<ProcId, Process>,

        /// 调度队列
        /// 仅存储 PID，表示处于就绪态 (Ready) 的进程
        ready_queue: VecDeque<ProcId>,
    }

    impl ProcManager {
        pub fn new() -> Self {
            Self {
                tasks: BTreeMap::new(),
                ready_queue: VecDeque::new(),
            }
        }
    }

``ProcManager`` 的设计遵循了 **存储与调度分离** 的原则：
- ``tasks`` 负责数据的生命周期管理。只要进程在 map 中，它就是存活的。
- ``ready_queue`` 负责调度逻辑。这里使用了简单的 FIFO (First-In-First-Out) 队列，实现了轮转调度 (Round-Robin)。

为了对接通用框架，``ProcManager`` 实现了两个 Traits：

.. code-block:: rust

    // 管理接口：负责增删改查
    impl Manage<Process, ProcId> for ProcManager {
        fn insert(&mut self, id: ProcId, task: Process) {
            self.tasks.insert(id, task);
        }
        fn get_mut(&mut self, id: ProcId) -> Option<&mut Process> {
            self.tasks.get_mut(&id)
        }
        fn delete(&mut self, id: ProcId) {
            self.tasks.remove(&id);
        }
    }

    // 调度接口：负责就绪队列操作
    impl Schedule<ProcId> for ProcManager {
        fn add(&mut self, id: ProcId) {
            self.ready_queue.push_back(id);
        }
        fn fetch(&mut self) -> Option<ProcId> {
            self.ready_queue.pop_front()
        }
    }

处理器管理结构 (Processor)
------------------------------------------------------------------------

最后，我们需要一个全局结构来统筹上述组件。``Processor`` 结构体本身变得非常简单，它主要作为一个并发安全的包装器（Wrapper），保护内部的 ``PManager``。

``PManager`` 是 ``tg-task-manage`` 库提供的一个组合结构，它将我们的 ``ProcManager``（策略与存储）和 ``Process``（数据）泛型组合在一起。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/processor.rs

    use core::cell::UnsafeCell;
    use tg_task_manage::PManager;

    pub struct Processor {
        // UnsafeCell 提供内部可变性
        inner: UnsafeCell<PManager<Process, ProcManager>>,
    }

    // 声明 Sync，表示该结构体可以在多线程（或中断上下文）间安全共享
    // 注意：实际的安全性需要通过 exclusive_access 或 get_mut 的正确使用来保证
    unsafe impl Sync for Processor {}

    impl Processor {
        pub const fn new() -> Self {
            Self {
                inner: UnsafeCell::new(PManager::new()),
            }
        }

        // 获取内部管理器的可变引用
        // 这一步通常需要配合关中断或锁机制，在本章单核非抢占/协作式环境下
        // 我们通过 Rust 的借用检查规则在逻辑上保证安全
        #[inline]
        pub fn get_mut(&self) -> &mut PManager<Process, ProcManager> {
            unsafe { &mut (*self.inner.get()) }
        }
    }

    // 全局唯一的处理器实例
    pub static PROCESSOR: Processor = Processor::new();

**内核主循环的工作流**

有了上述结构，内核的主循环（``rust_main``）变得非常清晰：

.. code-block:: rust

    // 伪代码演示主循环逻辑
    
    // 初始化...
    
    loop {
        // 1. 获取处理器管理器的可变引用
        let manager = PROCESSOR.get_mut();

        // 2. 尝试从调度队列获取下一个任务的 PID
        if let Some(pid) = manager.fetch() {
            // 3. 根据 PID 获取进程实体的引用
            let task = manager.get_mut(pid).unwrap();
            
            // 4. 执行进程！
            // 这一步会切换到用户态。
            // 当 execute 返回时，意味着发生了 Trap（系统调用、时钟中断等）
            unsafe { task.context.execute() };

            // 5. 处理 Trap
            // 此时我们回到了内核态，根据 scause 寄存器判断原因
            match scause::read().cause() {
                Trap::Exception(Exception::UserEnvCall) => {
                    // 处理系统调用
                    // 如果是 exit，则调用 manager.delete(pid)
                    // 如果是 yield，则调用 manager.add(pid) 重新放入队尾
                }
                Trap::Interrupt(Interrupt::SupervisorTimer) => {
                    // 时间片耗尽，放入队尾
                    manager.add(pid);
                }
                // ... 其他情况
            }
        } else {
            // 队列为空，无任务可运行
            // 对于批处理系统可能是结束，对于交互式系统可能是等待中断
            break; 
        }
    }

总结来说，``Processor`` 作为一个全局入口，通过 ``ProcManager`` 调度 ``Process``，利用 ``ForeignContext`` 实现特权级切换。这种模块化的设计使得各部分职责分明，易于扩展和维护。
