# OsTrain2019 项目结题报告

## 1.  项目目标概述

​	操作系统的一个重要作用是隔离，使得用户态的程序不能进行一些违规的操作（执行特权指令，管理文件系统，设备驱动等等），但是用户态和内核态的隔离也使得用户程序即使进行输出字符这种简单也工作也要进行状态切换进入内核态，导致了速度的降低。那么对于某个被信任的用户程序，是否可以将它的系统调用全部替换成内核的函数调用来加速呢？本项目就是想尝试对这一问题进行研究。

​	具体而言，该项目希望通过对 rcore 内核进行相应调整，增加相应的接口，使得一般的用户程序可以直接在ring0运行，同时取消 syscall 时的 context switch (syscall => function call)，从而对用户程序进行加速。



## 2. 已有工作

​	主要基于 rcore [https://github.com/rcore-os/rCore]，无其他相关工作。

​	主要代码参考也来自于rcore其他部分，知识性内容参考了大量博客。

## 3. 实现方案

​	由于不能对相关硬件进行更改，所以一旦被执行程序的机器码中出现 syscall 就一定会出发 context switch，解决方案有两个，一者是对 ELF 程序进行扫描，并替换其中的全部 syscall，这一点难度很大，所以采取重新编译源码的方案，这是相对可以接受的。此外需要在内核进行一定的修改，使得程序可以直接进入并一直在内核态运行，还需要处理进一步产生的 fork，exec等系统调用语义的改变。

​	编译器方面，出于对工具链方便性的考虑，决定对 x86_64-linux-musl-gcc 的源码进行调整，在 syscall 的底层对其进行替换，具体调整在第四部分叙述。

​	简而言之：

## 4 主要代码修改描述

#### 4.1 musl 库的修改

* arch/x86_64/syscall_arch.h

  ```c++
  // 原始：
  static __inline long __syscall0(long n)
  {
  	unsigned long ret;
  	__asm__ volatile("syscall" : "=a"(ret) :"a"(n): "memory", "r11", "rcx");
  	return ret;
  }
  // 修改为：
  static __inline long __syscall0(long n)
  {
  	unsigned long ret;
  	__asm__ volatile("call __rcore_syscall" : "=a"(ret) :"a"(n): "memory");
  	return ret;
  }
  ```

  替换最底层 `syscall`的汇编实现。

  * musl syscall的实现是通过巧妙的宏将相应的 syscall 指令转化为只与参数数量相关的 inline 指令，最终通过内嵌汇编转化为 syscall，所以将不同参数数量的 inline 全部改写后就能够得到几乎不会生成 syscall 的编译器，当然在C库自身的初始化和退出阶段还是会生成 syscall，但是数量为固定值，可以不予考虑）

* arch/x86_64/rcore_syscall.h

  ```c++
  __asm__(
  ".text \n"
  ".global __rcore_syscall \n"
  "__rcore_syscall: \n"
  "    subq $48, %rsp \n"		// 跳过 cs，ss，rsp等仅仅用于跳转的存储值
  "    pushq $0x667788 \n"	// 对于 fork，exec，clone 特殊处理
  "    pushq %rax \n"
  "    pushq %rcx \n"
  "    pushq %rdx \n"
  "    pushq %rdi \n"
  "    pushq %rsi \n"
  "    pushq %r8 \n"
  "    pushq %r9 \n"
  "    pushq %r10 \n"
  "    pushq %r11 \n"
  "    pushq %rbx \n"
  "    pushq %rbp \n"
  "    pushq %r12 \n"
  "    pushq %r13 \n"
  "    pushq %r14 \n"
  "    pushq %r15 \n"
  //   push fs.base, save fp registers   		// 大部分syscall不需要，跳过
  "    subq $(512+16+8+8), %rsp \n"
  
  "    movq %rsp, %rdi \n"
  "    call rcore_syscall_entry(%rip) \n"		// 跳转到预先得到的syscall真正入口
  
  "    addq $(512+16+8+8), %rsp \n"
  
  "    popq %r15 \n"
  "    popq %r14 \n"
  "    popq %r13 \n"
  "    popq %r12 \n"
  "    popq %rbp \n"
  "    popq %rbx \n"
  "    popq %r11 \n"
  "    popq %r10 \n"
  "    popq %r9 \n"
  "    popq %r8 \n"
  "    popq %rsi \n"
  "    popq %rdi \n"
  "    popq %rdx \n"
  "    popq %rcx \n"
  "    popq %rax \n"
  //   pop trap_num, error_code
  "    addq $56, %rsp \n"
  "    ret \n"
  );
  ```

  用以代替 `syscall` 指令的跳转函数，由于 `rsp`，`cs` 以及 大量的浮点寄存器，`fsbase` 寄存器等绝大部分的`syscall`并不需要，这里就直接跳过，对于相应的 `syscall` 特殊处理。

  和直接使用 `syscall`指令相比，仅仅储存了必要的寄存器(由于是inline函数，不能保证那些通用寄存器被使用了，必须全部保存)，跳过了 trap 分发的部分过程，在qemu上的模拟显示，省略了大约一半的context switch的时间。

* crt/crt1.c, crt/rcrt1.c

  ```c++
  static inline void get_syscall_entry() {
  	if(rcore_syscall_entry == 0L) {
  		unsigned long ret;
  		long n = SYS_GET_SYSCALL;
  		__asm__ __volatile__ ("syscall" : "=a"(ret) : "a"(n) : "rcx", "r11", "memory");
  		rcore_syscall_entry = (void (*)(void*))ret;
  	}
  }
  ```

  在静态库和动态库的初始化阶段插入一个自定义的 `syscall`指令，用以获得rcore的 syscall 处理入口。这样`rcore_syscall.h` 中的函数跳转才能正确执行。

* Makefile

  ```makefile
  CFLAGS_ALL += -mno-red-zone
  ```

  由于 redzone 的问题（详见第5部分），增加一条编译选项。

#### 4.2  rcore的修改

* kernel/src/shell.rs

  为了使得程序可以直接进入并一直在内核态运行，需要修改程序加载部分，这里我们先禁用 busybox 的命令行，启用rcore的 shell，这个shell 虽然功能十分简陋，但是我们可以自己处理命令行输入 。

  ```rust
  pub fn add_user_shell() {
      processor().manager().add(Thread::new_kernel(shell, 0));
  }
  ```

  ```rust
  pub extern "C" fn shell(_arg: usize) -> ! {	
      loop {
          print!(">> ");
          let cmd = get_line(&mut history).trim();
          if cmd == "" {
              continue;
          }
          let mut cmd_iter = cmd.trim().split(' ');
          let exec_mod = cmd_iter.next().unwrap(); 
          if exec_mod == "kexec" {
              let name = cmd_iter.next().unwrap();
              let _cmd: &str = cmd.trim()[5..].trim();
              if let Ok(inode) = ROOT_INODE.lookup(name) {
                  let _tid = processor().manager().add(Thread::new_kernel_from_inode(
                      &inode,
                      &_cmd,
                      _cmd.split(' ').map(|s| s.into()).collect(),
                      Vec::new(),
                  ));
                  // TODO: wait until process exits, or use user land shell completely
              } else {
                  println!("Program not exist");
              }
          } 
      }
  }
  ```

  一段十分丑陋的字符串处理，由于我对与rust不是十分熟悉，只能写出这样暴力难看的代码。功能就是如果一个正常的语句以 `“kexec”`开头，就认为会在特殊模式下执行一个程序，去掉这个前缀，并使用 `Thread::new_kernel_from_inode()` 加载，而非普通的 `Thread::new_user()`。

* src/process/structs.rs 

  仅仅做了一些关于权限的很 naive 的修改。重定位了动态加载器。

  ```rust
  impl Thread {
      pub fn new_kernel_vm(
          inode: &Arc<dyn INode>,
          exec_path: &str,
          mut args: Vec<String>,
          envs: Vec<String>,
      ) -> Result<(MemorySet, usize, usize), &'static str> {
          // ... 省略部分与 new_user_vm 函数相同
          let (mut vm, bias) = elf.make_kernel_memory_set(inode);
          if let Ok(loader_path_) = elf.get_interpreter() {
              // 强行改变loader的位置，仅仅实现了musl的动态加载库，分辨没有意义
              let loader_path = "/rethink/ld-musl-x86_64.so.1";
              // ...
          }
  
          // User stack
          use crate::consts::{USER_STACK_OFFSET, USER_STACK_SIZE};
          let mut ustack_top = {
              // ...
              vm.push(
                  // ...
                  MemoryAttr::default().kernel(),  // 仅仅改变内存区域的权限
                  // ...
              );
              ustack_top
          };
          // ...
      }
  
      pub fn new_kernel_from_inode() -> Box<Thread> {
          // 调用 new_kernel_vm()
          let (vm, entry_addr, ustack_top) =
              Self::new_kernel_vm(inode, exec_path, args, envs).unwrap();
          // ...
  
          Box::new(Thread {
              context: unsafe {
                  // 设置 context 权限，具体设置可以查看相关函数
                  Context::new_kernel_thread_for_user(entry_addr, ustack_top, kstack_top, vm_token)
              },
              // ...
          })
      }
  ```

* memory 模块的修改

  // 由于代码丢失，后补

* syscall 的修改

  // 由于代码丢失，后补

* alltraps.asm 的修改

  // 由于代码丢失，后补

## 后续完善

* 更加全面的测试

  Nginx 报错退出。推测是文件IO的复杂系统调用仍不兼容。

* 舍弃fork

  放弃fork，提供全新的系统调用完成 fork+exec的功能。可以在纯内核stack的情况下运行程序。

* shell

  移植ucore shell。

* 支持更多的架构

  支持 arm，mips，riscv。该项目硬件相关性很强 ，由于涉及到了不少汇编，移植工作较多，但是修改思路基本可以照搬。

