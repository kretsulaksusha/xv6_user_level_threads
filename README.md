# Additional lab work: xv6 User-level threads
Authors (team): [Ksenia Kretsula](https://github.com/kretsulaksusha)

**Task**: complete a simple user-level thread package by implementing the code to perform context switching between threads.

**Hardware**: CPU: Apple M1 Max; CPU cores (P+E): 10 (8+2).

## Installation on Mac OSX

1. Agree to the Xcode license:

   ```shell
   sudo xcodebuild -license accept
   ```
2. Installing QEMU for macOS:

   ```shell
   brew install qemu x86_64-elf-gcc
   ```
3. Building xv6:

   ```shell
   export TOOLPREFIX=x86_64-elf-
   export QEMU=qemu-system-x86_64
   make
   ```
4. Using QEMU emulator run xv6 (`-nox` is added to run with only the serial console),

   ```shell
   make qemu-nox
   ```
5. Exit xv6

   `Ctrl + a x`

### Adding `uthread` to xv6

Described in [Homework: User-level threads](https://pdos.csail.mit.edu/6.828/2018/homework/xv6-uthread.html).

**Steps:**

1. Add `uthread.c` and `uthread_switch.S` into xv6 directory.
2. Add the following rule to the xv6 Makefile after the `_forktest` rule:

   ```makefile
   _uthread: uthread.o uthread_switch.o
      $(LD) $(LDFLAGS) -N -e main -Ttext 0 -o _uthread uthread.o uthread_switch.o $(ULIB)
      $(OBJDUMP) -S _uthread > uthread.asm
   ```

3. Add `_uthread` in the Makefile to the list of user programs defined by `UPROGS`.
4. Run `uthread` from the xv6 shell. The xv6 kernel will print an error message about `uthread` encountering a page fault.

   ```text
   pid 4 uthread: trap 14 err 5 on cpu 0 eip 0xffffffff addr 0xffffffff--kill proc
   ```

5. The next step is to modify `uthread_switch.S`, so that the output is similar to this (run with `CPUS=1`):

   ```shell
   ~/classes/6828/xv6$ make CPUS=1 qemu-nox
   ...
   $ uthread
   my thread running
   my thread 0x2D68
   my thread running
   my thread 0x4D70
   my thread 0x2D68
   my thread 0x4D70
   my thread 0x2D68
   my thread 0x4D70
   my thread 0x2D68
   ...
   ```

6. Modifying `uthread_switch.S`

   At first understanding how `uthread.c` uses `thread_switch`:

   `uthread.c` has 2 global variables `current_thread` and `next_thread`. Each is a pointer to a thread structure.

   The thread structure has a stack for a thread and a saved stack pointer (`sp`, which points into the thread's stack). The job of `uthread_switch` is to save the current thread state into the structure pointed to by `current_thread`, restore `next_thread`'s state, and make `current_thread` point to where `next_thread` was pointing to, so that when `uthread_switch` returns `next_thread` is running and is the `current_thread`.

   `thread_create` provides hints about what `thread_switch` should do. The intent is that `thread_switch` use the assembly instructions `popal` and `pushal` to restore and save all eight x86 registers. Note that `thread_create` simulates eight pushed registers (32 bytes) on a new thread's stack.

   How the C compiler lays out struct thread in memory:

   ```text
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- current_thread
         ......

         ......
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- next_thread
    ```

   The variables `next_thread` and `current_thread` each contain the address of a struct thread.

   To write the `sp` field of the struct that `current_thread` points to:

   ```asm
   movl current_thread, %eax
   movl %esp, (%eax)
   ```

   This saves `%esp` in `current_thread->sp`. This works because `sp` is at offset 0 in the struct.

#### Implementation of the `thread_switch`

```asm
.text

.globl thread_switch
thread_switch:
  # 1. Save context
  pushal

  # 2. Save stack pointer of the current_thread
  movl current_thread, %eax
  movl %esp, (%eax)

  # 3. Change stack pointer to stack pointer of the next_thread
  movl next_thread, %eax
  movl (%eax), %esp

  # 4. Update the current_thread pointer to point to the next_thread
  movl %eax, current_thread

  # 5. Restore context
  popal

  # 6. Set next_thread pointer to null
  movl $0, next_thread

  ret
```

**Steps:**

1. **Save Context:**
    - Push all general-purpose registers onto the stack to preserve the current thread's execution state.

2. **Save Current Thread's Stack Pointer:**
    - Save the current stack pointer (`%esp`) into the `current_thread->sp`.

3. **Switch to Next Thread's Stack Pointer:**
    - Load the `next_thread->sp` into `%esp`.

4. **Update Thread Pointers:**
    - Set `current_thread` to point to `next_thread`, making it the new active thread.

5. **Restore Context:**
    - Pop all general-purpose registers.

## Testing & Results

All the commands executed in the video:
```bash
make clean
make
make qemu-nox CPUS=1

ls
uthread
```

Link to the video on Google Drive: [xv6 uthread usage](https://drive.google.com/file/d/1QUUsYp7KUENse1eH90NgM9aqazZtep-l/view?usp=share_link).

## Resources

- [Homework: User-level threads](https://pdos.csail.mit.edu/6.828/2018/homework/xv6-uthread.html)
- [x64 Cheat Sheet](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
