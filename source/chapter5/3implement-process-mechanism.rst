进程管理机制的设计实现
============================================

本节导读
--------------------------------------------

本节将介绍如何基于上一节设计的 ``Process``、``ProcManager`` 和 ``Processor`` 数据结构来实现完整的进程管理功能：

- **初始进程创建**：构建系统启动后的第一个用户进程 ``initproc``。
- **进程调度机制**：实现 ``sys_yield`` 和时间片轮转，利用 ``ForeignContext`` 进行特权级切换。
- **进程生成机制**：深入解析 ``sys_fork`` 和 ``sys_exec`` 的实现细节，特别是地址空间的复制与替换。
- **字符输入机制**：实现 ``sys_read`` 系统调用，处理标准输入。
- **进程资源回收机制**：当进程调用 ``sys_exit`` 退出后，父进程如何通过 ``sys_waitpid`` 回收僵尸进程资源。

初始进程的创建
--------------------------------------------

内核初始化完毕之后，即会调用 ``task`` 子模块提供的 ``add_initproc`` 函数来将初始进程 ``initproc``
加入任务管理器。在这之前，我们需要通过 ``lazy_static`` 初始化一个全局的初始进程实例。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/main.rs

    use spin::Lazy;
    use xmas_elf::ElfFile;

    /// 初始进程 initproc
    pub static INITPROC: Lazy<Process> = Lazy::new(|| {
        let elf_data = APPS.get("ch5b_initproc").unwrap();
        Process::from_elf(ElfFile::new(elf_data).unwrap()).unwrap()
    });

    /// 将初始进程加入调度队列
    pub fn add_initproc() {
        // 由于 Process 不支持 Clone，我们需要创建一个新副本或移动它。
        // 在实际实现中，我们通常会在 main 函数中直接获取并插入，
        // 或者通过 fork 的方式（如果支持内核线程）。
        // 这里的逻辑稍微调整为直接在 main 中初始化：
        
        /* 在 rust_main 中 */
        let initproc_data = APPS.get("ch5b_initproc").unwrap();
        let initproc = Process::from_elf(ElfFile::new(initproc_data).unwrap()).unwrap();
        let pid = initproc.pid;
        PROCESSOR.get_mut().insert(pid, initproc);
        PROCESSOR.get_mut().add(pid);
    }

我们调用 ``Process::from_elf`` 来创建一个进程控制块。这个函数封装了从 ELF 文件构建进程所需的复杂逻辑：

.. code-block:: rust
    :linenos:

    // tg-ch5/src/process.rs

    impl Process {
        pub fn from_elf(elf: ElfFile) -> Option<Self> {
            // 1. 解析 ELF Header 获取入口点
            let entry = match elf.header.pt2 { ... };

            // 2. 创建地址空间并映射段
            let mut address_space = AddressSpace::new();
            // ... 遍历 program headers 进行映射 ...
            
            // 3. 分配用户栈
            // 在地址空间的高地址分配一个用户栈，并设置栈指针
            let stack = unsafe { alloc_zeroed(...) };
            address_space.map_extern(...);

            // 4. 映射异界传送门 (Portal)
            // 这是关键步骤：用户进程必须映射跳板页才能进行系统调用
            crate::process::map_portal(&address_space);

            // 5. 初始化上下文
            let mut context = LocalContext::user(entry);
            // 设置 satp：模式为 SV39，根页表物理页号
            let satp = (8 << 60) | address_space.root_ppn().val();
            // 设置用户栈指针
            *context.sp_mut() = 1 << 38; // 虚地址

            Some(Self {
                pid: ProcId::new(), // 分配 PID
                context: ForeignContext { context, satp },
                address_space,
                heap_bottom: ...,
                program_brk: ...,
            })
        }
    }

关键点在于第 17 行的 ``map_portal``。在 SV39 分页模式下，当发生 Trap 时，硬件会跳转到 ``stvec`` 寄存器指向的地址。由于我们需要在不切换页表的情况下执行一段内核代码（保存用户寄存器），这段代码（Trampoline）必须同时存在于内核地址空间和所有用户地址空间的相同虚拟地址处（通常是最高页）。

进程调度机制
--------------------------------------------

当进程主动调用 ``sys_yield`` 交出 CPU，或时间片耗尽时，内核需要进行进程切换。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/main.rs (主循环逻辑)

    extern "C" fn rust_main() -> ! {
        // ... 初始化 ...
        
        loop {
            // 1. 从管理器中取出一个任务
            if let Some(pid) = PROCESSOR.get_mut().fetch() {
                let task = PROCESSOR.get_mut().get_mut(pid).unwrap();
                
                // 2. 执行任务 (切换到用户态)
                // ForeignContext::execute 会阻塞直到 Trap 发生
                unsafe { task.context.execute() };

                // 3. 处理 Trap
                use scause::{Exception, Trap};
                match scause::read().cause() {
                    // 系统调用处理
                    Trap::Exception(Exception::UserEnvCall) => {
                        use SyscallResult::*;
                        // handle_syscall 返回一个状态，指示调度行为
                        match handle_syscall(task) {
                            Done => { /* 继续运行当前进程 */ }
                            // 进程退出：从管理器中移除
                            Exit(code) => {
                                log::info!("proc {} exit with {}", pid, code);
                                PROCESSOR.get_mut().delete(pid);
                                // 注意：这里需要处理父子进程关系（见后文）
                            }
                            // 进程 Yield：重新加入调度队列
                            Yield => {
                                PROCESSOR.get_mut().add(pid);
                            }
                            // ...
                        }
                    }
                    // 时间片耗尽
                    Trap::Interrupt(Interrupt::SupervisorTimer) => {
                        tg_sbi::set_timer(u64::MAX); // 清除时钟
                        PROCESSOR.get_mut().add(pid); // 重新加入调度队列
                    }
                    // 异常处理：杀死进程
                    trap => {
                        log::error!("proc {} killed by {:?}", pid, trap);
                        PROCESSOR.get_mut().delete(pid);
                    }
                }
            } else {
                // 队列为空，等待中断或关机
                // ...
            }
        }
    }

与之前的版本不同，这里没有显式的 ``suspend_current_and_run_next`` 函数。调度逻辑被内嵌在内核的主循环中：
1.  **取任务**：``fetch()`` 从队列头部拿 PID。
2.  **执行**：``task.context.execute()`` 本质上就是一次上下文切换。
3.  **放回/销毁**：根据 Trap 原因，决定是将 PID 重新 ``add()`` 到队尾，还是 ``delete()`` 销毁。

这种基于主循环的调度结构更加扁平，避免了深层函数调用带来的栈开销，也更符合“单内核栈”的模型。

进程的生成机制
--------------------------------------------

fork 系统调用的实现
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``fork`` 的核心是创建一个现有进程的精确副本。这涉及到地址空间的深拷贝。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/process.rs

    impl Process {
        pub fn fork(&mut self) -> Option<Process> {
            // 1. 分配新 PID
            let pid = ProcId::new();

            // 2. 复制地址空间
            // AddressSpace::cloneself 是 tg-kernel-vm 提供的接口
            // 它会遍历源地址空间的所有映射区域，并在目标地址空间分配新的物理页，
            // 然后将数据从源物理页拷贝到新物理页。
            let parent_addr_space = &self.address_space;
            let mut address_space: AddressSpace<Sv39, Sv39Manager> = AddressSpace::new();
            parent_addr_space.cloneself(&mut address_space);

            // 3. 映射异界传送门
            // 新的地址空间必须包含 Portal，否则无法处理 Trap
            crate::process::map_portal(&address_space);

            // 4. 复制上下文
            // 此时 context 中保存的是父进程 Trap 时的状态
            let context = self.context.context.clone();
            
            // 5. 构造新 satp
            // 使用新页表的根物理页号
            let satp = (8 << 60) | address_space.root_ppn().val();
            let foreign_ctx = ForeignContext { context, satp };

            // 6. 返回新进程实例
            Some(Self {
                pid,
                context: foreign_ctx,
                address_space,
                heap_bottom: self.heap_bottom,
                program_brk: self.program_brk,
            })
        }
    }

在 ``sys_fork`` 系统调用接口层，我们需要处理返回值的差异：

.. code-block:: rust
    :linenos:

    // tg-ch5/src/main.rs (handle_syscall)

    fn handle_syscall(task: &mut Process) -> SyscallResult {
        // ... 获取参数 ...
        match id {
            Id::FORK => {
                let new_proc = task.fork().unwrap();
                let new_pid = new_proc.pid.get_usize();
                
                // 子进程返回值设置为 0
                // 通过修改子进程保存在 context 中的 a0 寄存器实现
                *new_proc.context.context.a_mut(0) = 0;

                // 将子进程加入调度队列
                let new_proc_pid = new_proc.pid; // Copy pid handle if implementing Copy, or just use usize for lookup
                PROCESSOR.get_mut().insert(new_proc_pid, new_proc);
                PROCESSOR.get_mut().add(new_proc_pid);

                // 父进程返回子进程 PID
                *task.context.context.a_mut(0) = new_pid;
                task.context.move_next(); // PC + 4
                SyscallResult::Done
            }
            // ...
        }
    }

这里展示了如何直接操作上下文中的寄存器：``*context.a_mut(0) = val``。这比旧版本中通过 TrapContext 指针操作更加直观和安全。

exec 系统调用的实现
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``exec`` 用于替换当前进程的映像。得益于 Rust 的所有权机制，我们可以直接用新创建的资源“覆盖”旧资源，旧资源会自动 Drop 并回收。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/process.rs

    impl Process {
        pub fn exec(&mut self, elf: ElfFile) {
            // 1. 使用新 ELF 创建一个临时进程实例
            // 这会分配新的页表、物理页等资源
            let proc = Process::from_elf(elf).unwrap();

            // 2. 资源替换
            // 将当前进程的组件替换为新进程的组件
            // 原有的 address_space 被覆盖后，会自动调用 Drop 回收旧页表和物理页
            self.address_space = proc.address_space;
            self.context = proc.context;
            self.heap_bottom = proc.heap_bottom;
            self.program_brk = proc.program_brk;

            // 注意：self.pid 保持不变，因此进程 ID 不变
        }
    }

在系统调用层：

.. code-block:: rust

    // tg-ch5/src/main.rs

    Id::EXEC => {
        // 1. 从用户空间读取应用名称字符串
        let ptr = args[0] as *const u8;
        let name = unsafe { task.address_space.translated_str(ptr) };

        // 2. 查找应用数据
        if let Some(data) = APPS.get(name.as_str()) {
            // 3. 执行替换
            task.exec(ElfFile::new(data).unwrap());
            
            // exec 成功后，不需要手动设置返回值或移动 PC
            // 因为 context 已经被重置为新程序的入口状态
            SyscallResult::Done
        } else {
            // 找不到应用，返回错误
            *task.context.context.a_mut(0) = -1 as isize as usize;
            task.context.move_next();
            SyscallResult::Done
        }
    }

sys_read 获取输入
--------------------------------------------

在本章中，我们初步引入了文件描述符的概念，尽管完整的文件系统尚未实现。

.. code-block:: rust
    :linenos:

    // tg-ch5/src/main.rs

    Id::READ => {
        let fd = args[0];
        let buf_ptr = args[1] as *mut u8;
        let len = args[2];

        match fd {
            0 => { // 标准输入 (Stdin)
                // 暂时只支持读取 1 个字符，这是为了配合 user_shell 的 getchar
                // 实际生产系统中需要缓冲区管理
                let mut c: u8;
                loop {
                    let ch = tg_sbi::console_getchar();
                    if ch == 0 {
                        // 串口没有输入，Yield 让出 CPU
                        // 调度器会在后续轮次再次尝试
                        task.context.move_next(); // 注意：这里是否应该移动 PC？
                        // 实际上，如果 Yield 后回来重新执行 read，应该保持 PC 不变
                        // 但这里的实现采用了简化的“非阻塞”风格：
                        // 如果没读到，我们在用户库层面循环调用，
                        // 或者在这里返回 Yield 状态但不移动 PC。
                        
                        // 更合理的非阻塞实现：
                        return SyscallResult::Yield; 
                    } else {
                        c = ch as u8;
                        break;
                    }
                }
                
                // 将读取的字符写入用户缓冲区
                // 需要进行地址翻译
                let buffers = task.address_space.translate_bytes(buf_ptr, len);
                if !buffers.is_empty() {
                    buffers[0][0] = c;
                    *task.context.context.a_mut(0) = 1; // 返回读取字节数 1
                } else {
                    *task.context.context.a_mut(0) = 0;
                }
                task.context.move_next();
                SyscallResult::Done
            }
            _ => {
                // 不支持其他文件描述符
                *task.context.context.a_mut(0) = -1 as isize as usize;
                task.context.move_next();
                SyscallResult::Done
            }
        }
    }

注意：在真实的实现中，如果 ``console_getchar`` 返回 0，我们应当让进程进入等待状态，而不是忙轮询。但为了简化 ch5 的实现，通常策略是：
1. 用户库 ``wait`` 封装了循环。
2. 内核发现无输入时返回特定错误码或 Yield，由用户库决定重试。

进程资源回收机制
--------------------------------------------

进程的退出 (Exit)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当进程调用 ``sys_exit`` 时，它将变成“僵尸 (Zombie)”状态，等待父进程回收。在本章的架构中，我们将这个逻辑放在 ``ProcManager`` 的 ``delete`` 之后的处理逻辑中。

但更准确地说，我们需要维护父子关系树。``tg-task-manage`` 提供了基本的进程树支持。

.. code-block:: rust
    :linenos:

    // 伪代码逻辑：处理进程退出

    // 当 handle_syscall 返回 SyscallResult::Exit(code) 时
    
    let exit_code = code;
    
    // 1. 标记当前进程状态为 Zombie，记录 exit_code
    // 在本章简化实现中，我们可能直接将 exit_code 写入某个共享位置，
    // 或者利用 TaskControlBlock 的字段。
    
    // 2. 进程树维护：孤儿进程托管
    // 将当前进程的所有子进程挂载到 initproc 下
    // 这一步需要访问全局的进程树结构
    
    // 3. 资源回收
    // 物理页、页表等资源会随着 Process 结构的 Drop 自动回收。
    // 但为了让父进程能获取 exit_code，我们需要保留最小限度的信息（通常是 PID 和 code）。
    // 在本章简化版中，可以直接在 delete 时销毁 Process，
    // 但必须确保父进程能通过某种方式查到该子进程已退出。

    // 简化的处理逻辑：
    log::info!("proc {} exit", pid);
    PROCESSOR.get_mut().delete(pid);
    // 这里的 delete 会导致 Process Drop，回收所有资源。
    // 如果父进程此时调用 waitpid，它怎么知道子进程结束了？
    
    // 实际上，我们需要一个“僵尸队列”或者在 Process 中保留 Zombie 状态而不立即 delete。
    // 但 tg-task-manage 的设计可能倾向于立即回收。
    // 一种常见的做法是：sys_exit 不立即 delete，而是标记状态。
    // 真正的 delete 发生在父进程 waitpid 时。

父进程回收 (Waitpid)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``sys_waitpid`` 的核心逻辑是查询子进程状态。

.. code-block:: rust
    :linenos:

    Id::WAITPID => {
        let pid = args[0] as isize;
        let exit_code_ptr = args[1] as *mut i32;
        
        // 我们需要遍历当前进程的子进程列表
        // 这需要 PROCESSOR 提供相关接口，或者我们在 Process 中维护了 children 列表
        
        let mut found_child = false;
        let mut found_zombie = None;
        
        // 假设 manager 提供了迭代所有进程的能力，或者我们维护了父子关系
        // 这里演示逻辑：
        
        // 1. 查找符合条件的子进程
        // 条件：子进程的 parent_id == current_pid
        // 且 (pid == -1 || 子进程.pid == pid)
        
        // 2. 检查状态
        // 如果子进程是 Zombie -> 回收
        // 如果子进程是 Running -> 返回 -2 (忙等待)或 Yield
        // 如果找不到子进程 -> 返回 -1
        
        if let Some((child_pid, exit_code)) = found_zombie {
            // 回收僵尸进程
            PROCESSOR.get_mut().delete(child_pid);
            
            // 写入 exit_code 到用户空间
            let translated_ptr = task.address_space.translate_bytes(exit_code_ptr as *mut u8, 4);
            // 写入数据...
            
            *task.context.context.a_mut(0) = child_pid as usize;
        } else if found_child {
            // 有子进程但没退出
            *task.context.context.a_mut(0) = -2 as isize as usize;
        } else {
            // 无子进程
            *task.context.context.a_mut(0) = -1 as isize as usize;
        }
        
        task.context.move_next();
        SyscallResult::Done
    }
