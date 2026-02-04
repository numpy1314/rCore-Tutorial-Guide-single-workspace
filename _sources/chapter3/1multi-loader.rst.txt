多道程序放置与加载
=====================================

多道程序放置
----------------------------


在第二章中，内核将每个应用都加载到同一个固定的起始地址，后加载的程序会覆盖之前加载的程序。
这使得内存中同时最多只能驻留一个应用。

想一次运行多个程序，就需要内核将每个用户程序加载到不同的起始地址。
注意运行本章节代码时，下面这一段中的输出：
.. code-block::

    for (i, app) in tg_linker::AppMeta::locate().iter().enumerate() {
        let entry = app.as_ptr() as usize;
        log::info!("load app{i} to {entry:#x}");
        tcbs[i].init(entry);
        index_mod += 1;
    }

代码中的 ``log::info!`` 的输出结果表明，内核确实将每个用户程序加载到了各自不同的地址，每个程序间隔 ``0x200000``。
只要我们假设每个程序所占内存都不会超过这个间隔，那么它们就都可以互不干扰地同时运行。

.. note::

    qemu 预留的内存空间是有限的，如果加载的程序过多，程序地址超出内存空间，可能出现 ``core dumped``.

多道程序加载
----------------------------

第三章沿用了第二章的许多用户程序。
它们的内容并没有改变，但却被内核以不同的方式放入内存。
观察在内核启动之前的输出：

.. code-block::

    # LOG=INFO cargo qemu --ch 3
        Finished `release` profile [optimized] target(s) in 0.05s
        Running `target/release/xtask qemu --ch 3`
    build "00hello_world" at 0x80400000
        Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
    build "01store_fault" at 0x80600000
    Compiling user_lib v0.0.1 (/home/scpointer/2026/rCore-Tutorial-in-single-workspace/user)
        Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
    build "02power" at 0x80800000
    ......

可以看到，每个用户程序在编译时就已经指定了各自的起始地址。
事实上，这个起始地址是由链接文件 ``linker.ld`` 指定的。
我们需要通过某种方法，在编译每个用户程序时自动修改该文件，就能将每个程序指定在不同位置了。


1. 首先， ``user\cases.toml`` 中指定了每个章节对应的参数，其中 ``[ch3]`` 下的子项指定了本章的参数，包括起始地址 ``0x8040_0000`` 、每个用户程序之间的间隔 ``0x0020_0000`` 和一个程序列表。

.. note::
    相比 ``Cargo.toml`` ，这个 ``cases.toml`` 并不是 Rust 规范中的文件，它只是用来存储参数的文件，理论上完全可以用 json 或其他格式代替。

2. 这个参数文件被 ``xtask/src/user.rs:build_for`` 读取，经过 ``toml::from_str`` 的解析，上述信息被传递给 ``cases`` 变量。

.. code-block:: rust

    // xtask/src/user.rs

    pub fn build_for(ch: u8, release: bool, exercise: bool, ci: bool) {
        let cfg = std::fs::read_to_string(PROJECT.join("user/cases.toml")).unwrap();
        let key = if exercise {
            format!("ch{ch}_exercise")
        } else {
            format!("ch{ch}")
        };
        let mut cases = toml::from_str::<HashMap<String, Cases>>(&cfg)
            .unwrap()
            .remove(&key)
            .unwrap_or_default();
        let CasesInfo { base, step, bins } = cases.build(ch, release, ci);

3. 然后， ``Case::build`` 函数枚举每个用户程序，通过程序的编号算出它应在的地址（ ``base + i as u64 * step`` ），将程序名与这个地址一同传给 ``build_one`` 函数。

.. code-block:: rust

    // xtask/src/user.rs

    impl Cases {
        fn build(&mut self, ch: u8, release: bool, ci: bool) -> CasesInfo {
            if let Some(names) = &self.cases {
                let base = self.base.unwrap_or(0);
                let step = self.step.filter(|_| self.base.is_some()).unwrap_or(0);
                let chapter_env = if ci { Some(ch) } else { None };
                let cases = names
                    .iter()
                    .enumerate()
                    .map(|(i, name)| build_one(chapter_env, name, release, base + i as u64 * step))
                    .collect();

4. 这个 ``build_one`` 中包含一条超长的 cargo 命令。
它的核心是 ``Cargo::build().package("user_lib").invoke()`` ，而中间的语句都是在增加参数和环境变量。
例如，在 ``build_one(Some(3), "00hello_world", true, 0x80400000)`` 中，上述命令实际相当于 ``CHAPTER=3 BASE_ADDRESS=0x80400000 cargo build --package user_lib --bin 00hello_world --release`` 。

.. code-block:: rust

    // xtask/src/user.rs

    fn build_one(
        chapter_env: Option<u8>,
        name: impl AsRef<OsStr>,
        release: bool,
        base_address: u64,
    ) -> PathBuf {
        let name = name.as_ref();
        let binary = base_address != 0;
        if binary {
            println!("build {name:?} at {base_address:#x}");
        }
        Cargo::build()
            .package("user_lib")
            .target(TARGET_ARCH)
            .arg("--bin")
            .arg(name)
            .conditional(chapter_env.is_some(), |cargo| {
                cargo.env("CHAPTER", chapter_env.unwrap().to_string());
            })
            .conditional(release, |cargo| {
                cargo.release();
            })
            .conditional(binary, |cargo| {
                cargo.env("BASE_ADDRESS", base_address.to_string());
            })
            .invoke();
        let elf = TARGET
            .join(if release { "release" } else { "debug" })
            .join(name);
        if binary {
            objcopy(elf, binary)
        } else {
            elf
        }
    }

5. 这里的 ``BASE_ADDRESS`` 只是一个环境变量，它本身无法“魔法”地改变起始地址。这还需要 ``user/build.rs`` 的帮助。

注意， ``cargo:rerun-if-env-changed=BASE_ADDRESS`` 一行说明 ``BASE_ADDRESS`` 这一环境变量改变时，该模块需要重新构建。随后，我们构建了一个 ``text`` 字符串，将这一地址信息存入其中，然后将字符串写入 ``linker.ld`` 文件。

.. code-block:: rust

    // user/build.rs

    fn main() {
        use std::{env, fs, path::PathBuf};

        println!("cargo:rerun-if-changed=build.rs");
        println!("cargo:rerun-if-env-changed=LOG");
        println!("cargo:rerun-if-env-changed=BASE_ADDRESS");
        println!("cargo:rerun-if-env-changed=CHAPTER");

        if let Ok(chapter) = env::var("CHAPTER") {
            println!("cargo:rustc-env=CHAPTER={chapter}");
        }

        if let Some(base) = env::var("BASE_ADDRESS")
            .ok()
            .and_then(|s| s.parse::<u64>().ok())
        {
            let text = format!(
                "\
    OUTPUT_ARCH(riscv)
    ENTRY(_start)
    SECTIONS {{
        . = {base};
        .text : {{
            *(.text.entry)
            *(.text .text.*)
        }}
        .rodata : {{
            *(.rodata .rodata.*)
            *(.srodata .srodata.*)
        }}
        .data : {{
            *(.data .data.*)
            *(.sdata .sdata.*)
        }}
        .bss : {{
            *(.bss .bss.*)
            *(.sbss .sbss.*)
        }}
    }}"
            );
            let ld = PathBuf::from(env::var_os("OUT_DIR").unwrap()).join("linker.ld");
            fs::write(&ld, text).unwrap();
            println!("cargo:rustc-link-arg=-T{}", ld.display());
        }
    }

6. 至此，我们完成了本小节开头的任务——即在编译每个用户程序时，自动修改 ``linker.ld`` 中的起始地址。

但别忘了 ``build_for`` 函数的后半段，我们还得把起始地址与间隔的信息告诉内核。这些信息被打包在 ``app.asm`` 文件中，你可以直接在 ``target/riscv64gc-unknown-none-elf/debug/app.asm`` 找到它，对照代码阅读文件内容。

.. code-block:: rust

    // xtask/src/user.rs

    let CasesInfo { base, step, bins } = cases.build(ch, release, ci);
    if bins.is_empty() {
        return;
    }
    let asm = TARGET
        .join(if release { "release" } else { "debug" })
        .join("app.asm");
    let mut ld = File::create(asm).unwrap();
    writeln!(
        ld,
        "\
        .global apps
        .section .data
        .align 3
    apps:
        .quad {base:#x}
        .quad {step:#x}
        .quad {}",
        bins.len(),
    )
    .unwrap();

    (0..bins.len()).for_each(|i| writeln!(ld, "    .quad app_{i}_start").unwrap());

    writeln!(ld, "    .quad app_{}_end", bins.len() - 1).unwrap();

    bins.iter().enumerate().for_each(|(i, path)| {
        writeln!(
            ld,
                "
    app_{i}_start:
        .incbin {path:?}
    app_{i}_end:",
        )
        .unwrap();
    });

    // app.asm
        .global apps
        .section .data
        .align 3
    apps:
        .quad 0x80400000
        .quad 0x200000
        .quad 12
        .quad app_0_start
        .quad app_1_start
        .quad app_2_start
        .quad app_3_start
        .quad app_4_start
        .quad app_5_start
        .quad app_6_start
        .quad app_7_start
        .quad app_8_start
        .quad app_9_start
        .quad app_10_start
        .quad app_11_start
        .quad app_11_end

7. 最后，内核启动后，``tg-linker/src/app.rs`` 中读取上述 ``app.asm`` 中的 apps 变量存入 ``AppMeta`` 中，就有了我们在 ``ch3/src/main.rs`` 里看到的每个用户程序的不同地址。

.. code-block:: rust
    
    // tg-linker/src/app.rs

    /// 应用程序元数据。
    #[repr(C)]
    pub struct AppMeta {
        base: u64,
        step: u64,
        count: u64,
        first: u64,
    }

    impl AppMeta {
        /// 定位应用程序元数据。
        ///
        /// 返回由链接脚本定义的应用程序元数据的静态引用。
        #[inline]
        pub fn locate() -> &'static Self {
            extern "C" {
                static apps: AppMeta;
            }
            // SAFETY: `apps` 是由链接脚本定义的静态符号，在程序运行期间始终有效。
            // 它的内存布局与 AppMeta 结构体匹配（由 #[repr(C)] 保证）。
            unsafe { &apps }
        }

        /// 遍历链接进来的应用程序。
        #[inline]
        pub fn iter(&'static self) -> AppIterator {
            AppIterator { meta: self, i: 0 }
        }
    }

