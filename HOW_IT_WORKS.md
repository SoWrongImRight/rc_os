# How rc_os Works

rc_os is a small x86_64 kernel written in Rust. It follows Philipp Oppermann's [Writing an OS in Rust](https://os.phil-opp.com/) series. That series explains how to **build** each piece. This document explains how the pieces **work** once they are built: what the hardware does, what each file is responsible for, and how control moves from power-on to text appearing as you type.

Current stage: **Paging Implementation** (through "Creating a New Mapping"). The kernel boots, prints text, handles CPU exceptions and page faults, survives stack overflows, handles timer and keyboard interrupts, can read and modify its own page tables, and maps a virtual page to the VGA buffer.

---

## Table of contents

1. [The big picture](#1-the-big-picture)
2. [Building a kernel with no OS underneath](#2-building-a-kernel-with-no-os-underneath)
3. [Boot: from power-on to `kernel_main`](#3-boot-from-power-on-to-kernel_main)
4. [Printing to the screen: the VGA text buffer](#4-printing-to-the-screen-the-vga-text-buffer)
5. [Printing to the host: the serial port](#5-printing-to-the-host-the-serial-port)
6. [Testing inside QEMU](#6-testing-inside-qemu)
7. [CPU exceptions and the IDT](#7-cpu-exceptions-and-the-idt)
8. [Double faults, the GDT, and the TSS](#8-double-faults-the-gdt-and-the-tss)
9. [Hardware interrupts: the PIC and the timer](#9-hardware-interrupts-the-pic-and-the-timer)
10. [Shared state: why everything is a `Mutex` in a `lazy_static`](#10-shared-state-why-everything-is-a-mutex-in-a-lazy_static)
11. [Deadlocks: locks vs. interrupts](#11-deadlocks-locks-vs-interrupts)
12. [Idling with `hlt`](#12-idling-with-hlt)
13. [Keyboard input](#13-keyboard-input)
14. [Paging: virtual memory](#14-paging-virtual-memory)
15. [Reaching the page tables: `BootInfo` and the physical memory offset](#15-reaching-the-page-tables-bootinfo-and-the-physical-memory-offset)
16. [Creating a mapping](#16-creating-a-mapping)
17. [File map](#17-file-map)
18. [Running it](#18-running-it)

---

## 1. The big picture

```
 ┌──────────────── QEMU (emulated x86_64 PC) ─────────────────┐
 │                                                            │
 │  BIOS ─► bootloader ─► kernel_main(boot_info) (main.rs)    │
 │                                  │                         │
 │                                  ▼                         │
 │                           rc_os::init()  (src/lib.rs)      │
 │                 ┌────────────┬───┴──────────┬───────────┐  │
 │                 ▼            ▼              ▼           ▼  │
 │             gdt::init   init_idt()   PICS.initialize  sti  │
 │             (GDT+TSS)   (handlers)   (remap IRQs)  (enable)│
 │                                                            │
 │  CPU exception ──► IDT ──► breakpoint/page/double fault    │
 │  Timer chip ─► PIC ─► IDT[32] ──► timer_interrupt_handler  │
 │  Keyboard ─► PIC ─► IDT[33] ──► keyboard_interrupt_handler │
 │                                                            │
 │  println! ─► VGA buffer @ 0xb8000 ─► QEMU window           │
 │  serial_println! ─► UART @ port 0x3F8 ─► your terminal     │
 └────────────────────────────────────────────────────────────┘
```

The kernel is one Rust crate with two faces:

- **`src/lib.rs`** is the library. All real functionality lives here so integration tests in `tests/` can reuse it.
- **`src/main.rs`** is the kernel binary. It is a thin entry point that calls into the library.

---

## 2. Building a kernel with no OS underneath

A normal Rust program assumes an operating system is present to provide threads, files, heap memory, and a way to start `main`. A kernel has none of that, so several settings work together to remove those assumptions.

### `#![no_std]` and `#![no_main]`

- `no_std` drops the standard library and links only `core`, the part of Rust that needs no OS: integers, slices, `Option`, formatting, and so on.
- `no_main` drops the usual `main` entry point. The C runtime that would normally call `main` does not exist here. Instead we export a function named `_start`, the symbol the linker treats as the entry point by convention:

  ```rust
  #[unsafe(no_mangle)]               // keep the symbol name exactly "_start"
  pub extern "C" fn _start() -> ! {  // C calling convention, never returns
  ```

  It returns `!` (never) because there is nothing to return to.

  The kernel binary no longer writes `_start` by hand. The `bootloader` crate's `entry_point!` macro generates it and calls a normal Rust function, `kernel_main`, with a type-checked signature. See section 15. The integration tests in `tests/` still write `_start` directly, as shown above.

### Panics

With no OS there is no stack unwinding, so `Cargo.toml` sets `panic = "abort"`. We must also supply our own `#[panic_handler]`. In `main.rs` it prints the message to the screen and spins forever.

### The custom target: `x86_64-rc_os.json`

This file tells `rustc` what machine we are compiling for. The important fields:

| Field | Value | Why |
|---|---|---|
| `llvm-target` | `x86_64-unknown-none` | 64-bit x86 with no OS. |
| `linker` | `rust-lld` | LLVM's cross-platform linker, so we don't depend on the host's linker. |
| `panic-strategy` | `abort` | Matches `Cargo.toml`. |
| `disable-redzone` | `true` | The "red zone" is 128 bytes below the stack pointer that functions may use without moving the stack pointer. An interrupt can arrive at any moment and push its frame onto the stack, overwriting that area. Kernels must disable it. |
| `features` | `-mmx,-sse,+soft-float` | SIMD registers are large. Using them in the kernel would force us to save and restore them on every interrupt. We turn them off and do floating-point math in software instead. |

### `.cargo/config.toml`

```toml
[build]
target = "x86_64-rc_os.json"         # always build for our custom target

[unstable]
build-std = ["core", "compiler_builtins"]   # compile core from source for our target
build-std-features = ["compiler-builtins-mem"]  # provide memcpy/memset/etc.

[target.'cfg(target_os = "none")']
runner = "bootimage runner"          # `cargo run` / `cargo test` launch QEMU
```

There is no precompiled `core` for a custom target, so `build-std` compiles it from source. That is why this project needs a **nightly** toolchain. `compiler-builtins-mem` supplies `memcpy` and related functions that the compiler emits calls to, which would normally come from the C library.

### `bootimage` and the `bootloader` crate

The compiled kernel is an ELF file. A PC cannot boot an ELF file directly. The `bootimage` tool:

1. compiles the `bootloader` crate (v0.9),
2. links our kernel ELF into it, and
3. produces a raw disk image (`bootimage-rc_os.bin`) that QEMU can boot.

The kernel also uses the `bootloader` crate directly. Its `entry_point!` macro and `BootInfo` type are covered in section 15.

---

## 3. Boot: from power-on to `kernel_main`

1. **BIOS** runs from firmware, finds the bootable disk, and loads its first 512 bytes.
2. **The bootloader** (from the `bootloader` crate) takes over. The CPU starts in 16-bit *real mode*, and the bootloader moves it through 32-bit *protected mode* into 64-bit *long mode*. It sets up page tables (including a mapping of all physical memory, see section 15), loads our kernel from the disk image, sets up a stack, and jumps to `_start` with a pointer to a `BootInfo` struct.
3. **`_start`** is generated by `entry_point!` and immediately calls **`kernel_main(boot_info)`** (`src/main.rs`):

   ```rust
   println!("Hello World{}", "!");
   rc_os::init();          // GDT, IDT, PIC, enable interrupts
   // paging demo: get the page tables, map page 0 to the VGA buffer, write "New!"
   let mut mapper = unsafe { memory::init(phys_mem_offset) };
   memory::create_example_mapping(page, &mut mapper, &mut frame_allocator);
   #[cfg(test)] test_main();
   println!("It did not crash!");
   rc_os::hlt_loop();      // the kernel never exits; sleep until each interrupt
   ```

From here on, the kernel does no work of its own. Everything that happens next is driven by interrupts (see sections 9 and 12).

---

## 4. Printing to the screen: the VGA text buffer

**File:** `src/vga_buffer.rs`

In VGA text mode, the screen is a grid of 80 columns by 25 rows. That grid is **memory-mapped** at physical address `0xb8000`, so writing bytes to that address puts characters on the screen. No driver is needed.

Each cell is 2 bytes:

```
 byte 0: ASCII code         byte 1: color
                            ┌─────────┬──────────┐
                            │bg (4bit)│fg (4bit) │
                            └─────────┴──────────┘
```

The code mirrors that layout exactly:

- `Color` is a `#[repr(u8)]` enum of the 16 VGA colors.
- `ColorCode(u8)` packs background and foreground: `(bg << 4) | fg`.
- `ScreenChar` is `#[repr(C)]` so Rust keeps the field order (character, then color).
- `Buffer` is a 25×80 array of `ScreenChar`, and is `#[repr(transparent)]` so it has the same layout as the raw memory.

### Why `Volatile`?

The compiler sees that we write to the buffer but never read it back, so it could decide those writes are useless and delete them. `Volatile<ScreenChar>` forces every read and write to actually happen, because they have side effects the compiler can't see (characters appearing on screen).

### The `Writer`

`Writer` always writes to the **bottom row**. On a newline, or when the row is full, `new_line()` copies every row up by one and clears the bottom row. That is how scrolling works. Bytes outside printable ASCII are shown as `■` (`0xfe`).

`Writer` implements `core::fmt::Write`, which gives us Rust's formatting machinery (`{}`, `{:?}`, `{:#?}`) for free.

### The global `WRITER` and `println!`

```rust
lazy_static! {
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer {
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
        ...
    });
}
```

`print!` and `println!` are macros that call `vga_buffer::_print`, which locks `WRITER` and writes the formatted text. Section 10 explains why this is a `Mutex` inside a `lazy_static`, and section 11 explains why the lock is taken with interrupts disabled.

---

## 5. Printing to the host: the serial port

**File:** `src/serial.rs`

The VGA buffer is only visible inside the QEMU window. For tests we want output in the terminal that ran `cargo test`. The old **UART 16550** serial chip makes this possible:

- The kernel writes bytes to I/O port `0x3F8` (COM1).
- QEMU is started with `-serial stdio` (see `[package.metadata.bootimage]` in `Cargo.toml`), so whatever the guest writes to COM1 appears on the host's stdout.

`serial_print!` and `serial_println!` work like `print!` and `println!`, but go through a `SERIAL1` mutex instead of `WRITER`.

**Port I/O vs. memory-mapped I/O:** x86 has a separate 16-bit "port" address space accessed with the special `in` and `out` instructions. The VGA buffer is memory-mapped (normal memory writes). The serial chip, the PIC, and QEMU's exit device all use ports.

---

## 6. Testing inside QEMU

Rust's built-in test framework depends on `std`, so this project uses the unstable **custom test frameworks** feature.

### How a unit test run works

```rust
#![feature(custom_test_frameworks)]
#![test_runner(crate::test_runner)]
#![reexport_test_harness_main = "test_main"]
```

1. The compiler collects every function marked `#[test_case]`.
2. It generates a function, which we rename to `test_main`, that passes them all as a slice to our `test_runner`.
3. When built with `cfg(test)`, the entry point calls `test_main()`. In `lib.rs` that entry point is `test_kernel_main`, set up with `entry_point!`.
4. `test_runner` calls each test through the `Testable` trait, which prints `name...\t[ok]` over serial.
5. When all tests pass, it calls `exit_qemu(Success)`.

### Shutting QEMU down from inside the guest

QEMU normally runs until you close it. The test config adds a special device:

```
-device isa-debug-exit,iobase=0xf4,iosize=0x04
```

When the guest writes a value `v` to port `0xf4`, QEMU exits with status `(v << 1) | 1`.

| Kernel writes | QEMU exit status | Meaning |
|---|---|---|
| `0x10` (`Success`) | `(0x10 << 1) \| 1 = 33` | `test-success-exit-code = 33` tells `bootimage` this means pass. |
| `0x11` (`Failed`) | `35` | Anything other than 33 counts as failure. |

`-display none` hides the window during tests, and `test-timeout = 300` kills a run that hangs, such as one stuck in an infinite loop or a deadlock.

### Integration tests (`tests/`)

Each file in `tests/` is compiled as its **own separate kernel** with its own `_start` and panic handler. It links against the `rc_os` library.

| Test | What it proves | How |
|---|---|---|
| `basic_boot.rs` | `println!` works right after boot, before `init()`. | Normal test runner. |
| `should_panic.rs` | Panics are detected correctly. | `harness = false`. The panic handler *reports success*, and returning normally reports failure. |
| `stack_overflow.rs` | A stack overflow becomes a handled double fault, not a reboot. | Sets up its own IDT whose double-fault handler exits QEMU with `Success`. See section 8. |

---

## 7. CPU exceptions and the IDT

**File:** `src/interrupts.rs`

An **exception** is the CPU telling the kernel that something went wrong, or needs attention, while running the current instruction. Examples: divide by zero, page fault, invalid opcode, or the `int3` breakpoint instruction.

When this happens the CPU:

1. Looks up the exception number in the **Interrupt Descriptor Table (IDT)**.
2. Pushes an **interrupt stack frame** onto the stack: the instruction pointer, code segment, flags, stack pointer, and stack segment at the moment of the exception. Some exceptions also push an error code.
3. Jumps to the handler address stored in the IDT entry.

The IDT has 256 entries. The first 32 (0–31) are reserved for CPU exceptions. The rest are free for hardware and software interrupts.

### The `x86-interrupt` calling convention

A normal function call has rules about which registers the caller saves and which the callee saves. An interrupt can arrive *between any two instructions*, so the interrupted code had no chance to save anything. A handler must therefore preserve **every** register it touches and return with the special `iretq` instruction instead of `ret`.

`extern "x86-interrupt" fn` (enabled by `#![feature(abi_x86_interrupt)]`) tells the compiler to generate exactly that code. We write ordinary Rust and the compiler handles the save, restore, and `iretq`.

### Our IDT

```rust
lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);              // vector 3
        idt.double_fault.set_handler_fn(double_fault_handler)           // vector 8
            .set_stack_index(gdt::DOUBLE_FAULT_IST_INDEX);
        idt[InterruptionIndex::Timer.as_u8()]                           // vector 32
            .set_handler_fn(timer_interrupt_handler);
        idt[InterruptionIndex::Keyboard.as_u8()]                        // vector 33
            .set_handler_fn(keyboard_interrupt_handler);
        idt.page_fault.set_handler_fn(page_fault_handler);              // vector 14
        idt
    };
}
```

`init_idt()` calls `IDT.load()`, which runs the `lidt` instruction to tell the CPU where the table lives. The table must stay valid for the whole life of the kernel. That is why it is a `static` and not a local variable.

- **Breakpoint (3):** prints the stack frame and **returns**, so execution continues after the `int3`. The `test_breakpoint_exception` test checks exactly this: if the handler were broken, the test kernel would crash.
- **Double fault (8):** returns `!` and panics. See the next section.
- **Page fault (14):** prints the faulting address and error code, then halts. See section 14.
- **Timer (32) and keyboard (33):** hardware interrupts. See sections 9 and 13.

---

## 8. Double faults, the GDT, and the TSS

**File:** `src/gdt.rs`

### What is a double fault?

If the CPU hits an exception *while trying to call the handler for a previous exception* (for example, the handler is missing, or pushing the stack frame itself faults), it raises a **double fault** (vector 8). If calling the double-fault handler also fails, that is a **triple fault**. The CPU resets, and QEMU reboots in a loop.

### The stack overflow problem

Consider a stack overflow:

1. Infinite recursion runs the stack pointer into the **guard page**, an unmapped page the bootloader places below the stack.
2. The CPU raises a **page fault** and tries to push the exception frame onto the stack...
3. ...but the stack pointer still points into the guard page, so the push faults again → **double fault**.
4. The CPU tries to push a frame for the double fault, onto the same broken stack → **triple fault** → reboot.

The fix is to make the CPU **switch to a known-good stack** before calling the double-fault handler.

### The Interrupt Stack Table (IST)

x86_64 provides the IST: a list of up to 7 alternate stack pointers. An IDT entry can say "always switch to IST stack *N* before calling me". That is what `.set_stack_index(gdt::DOUBLE_FAULT_IST_INDEX)` does.

The IST lives inside a structure called the **Task State Segment (TSS)**. In 64-bit mode the TSS no longer manages tasks (its 32-bit purpose). It is now mostly a holder for these stack pointers.

```rust
static ref TSS: TaskStateSegment = {
    let mut tss = TaskStateSegment::new();
    tss.interrupt_stack_table[DOUBLE_FAULT_IST_INDEX as usize] = {
        const STACK_SIZE: usize = 4096 * 5;           // 20 KiB
        static mut STACK: [u8; STACK_SIZE] = [0; STACK_SIZE];
        let stack_start = VirtAddr::from_ptr(&raw const STACK);
        stack_start + STACK_SIZE as u64               // stacks grow DOWN, so give the END address
    };
    tss
};
```

The stack is a plain `static` array. There is no memory allocator yet. It also has no guard page of its own, so a double-fault handler that used too much stack would silently corrupt memory. This is acceptable for now.

### The Global Descriptor Table (GDT)

The CPU can't be handed a TSS directly. It must be referenced through a **segment descriptor** in the **Global Descriptor Table (GDT)**. In 64-bit mode, segmentation is mostly disabled, but the GDT survives for two jobs: holding the kernel code segment and holding the TSS descriptor.

```rust
let code_selector = gdt.append(Descriptor::kernel_code_segment());
let tss_selector  = gdt.append(Descriptor::tss_segment(&TSS));
```

`gdt::init()` then:

1. loads the GDT (`lgdt`),
2. reloads the code segment register `CS` with our new code selector, because the old value pointed into the bootloader's GDT, and
3. loads the TSS (`ltr`) so the CPU knows where the IST is.

These are `unsafe` because a wrong selector breaks memory safety in ways the compiler can't check.

### Putting it together

```
stack overflow ─► page fault ─► can't push frame ─► DOUBLE FAULT
                                                        │
                         IDT[8] says "use IST[0]"  ◄────┘
                                  │
               TSS.ist[0] = top of 20 KiB static STACK
                                  │
                 CPU switches stacks, pushes frame safely
                                  │
                     double_fault_handler runs ✔
```

`tests/stack_overflow.rs` checks this whole chain. It recurses forever, and its custom double-fault handler exits QEMU with `Success`.

### Initialization order matters

`init()` calls `gdt::init()` **before** `init_idt()`. The IDT's double-fault entry refers to IST index 0, which only means something once the TSS is loaded.

`stack_overflow.rs` deliberately calls only `gdt::init()` and loads its own IDT, not `rc_os::init()`. So in that test interrupts are never enabled, and a timer tick can't interfere with the overflow being tested.

---

## 9. Hardware interrupts: the PIC and the timer

**Files:** `src/interrupts.rs`, `src/lib.rs`. This matches the [Hardware Interrupts](https://os.phil-opp.com/hardware-interrupts/) post.

Exceptions come from the CPU itself. **Hardware interrupts** come from external devices (timer, keyboard, disk) to say "something happened, come look". They let the CPU react to events instead of constantly checking (*polling*) every device.

### The 8259 PIC

On a classic PC, devices don't connect to the CPU directly. They connect to a **Programmable Interrupt Controller**. There are two 8259 chips, *chained*: the secondary PIC's output feeds into line 2 of the primary.

```
                     ┌──────────────┐        ┌──────────────┐
 Timer (PIT) ─ IRQ0 ─┤              │        │              ├─ IRQ8  RTC
 Keyboard    ─ IRQ1 ─┤   Primary    │◄─IRQ2──┤  Secondary   ├─ IRQ9..11
                     │    PIC       │        │    PIC       ├─ IRQ12 Mouse
 COM2/COM1   ─ IRQ3/4┤  (ports      │        │  (ports      ├─ IRQ13 FPU
 ...         ─ IRQ5-7┤  0x20/0x21)  │        │  0xA0/0xA1)  ├─ IRQ14/15 ATA
                     └──────┬───────┘        └──────────────┘
                            │ "interrupt vector N"
                            ▼
                           CPU ──► IDT[N]
```

Each PIC turns an IRQ line into an **interrupt vector number** for the CPU to look up in the IDT.

### Remapping: why the offsets are 32 and 40

By default the primary PIC maps IRQ 0–7 to vectors **8–15**. Those collide with CPU exceptions: IRQ0 (the timer) would arrive as vector 8, which looks exactly like a **double fault**. So we reprogram the PICs to use vectors after the 32 reserved exception slots:

```rust
pub const PIC_1_OFFSET: u8 = 32;                  // IRQ0–7  → vectors 32–39
pub const PIC_2_OFFSET: u8 = PIC_1_OFFSET + 8;    // IRQ8–15 → vectors 40–47

pub static PICS: spin::Mutex<ChainedPics> =
    spin::Mutex::new(unsafe { ChainedPics::new(PIC_1_OFFSET, PIC_2_OFFSET) });
```

`PICS.lock().initialize()` (called from `init()`) sends the PICs their initialization command words over their I/O ports, including these offsets. `ChainedPics::new` is `unsafe` because wrong offsets would make device interrupts look like CPU exceptions.

### Naming vectors: `InterruptionIndex`

```rust
#[repr(u8)]
pub enum InterruptionIndex {
    Timer = PIC_1_OFFSET,   // IRQ0 → vector 32
}
```

This gives vector numbers readable names. More entries are added as more devices are handled. `Keyboard` (`PIC_1_OFFSET + 1`) is covered in section 13.

### Turning interrupts on

```rust
pub fn init() {
    gdt::init();
    interrupts::init_idt();
    unsafe { interrupts::PICS.lock().initialize() };
    x86_64::instructions::interrupts::enable();   // the `sti` instruction
}
```

The CPU ignores hardware interrupts until the interrupt flag is set with `sti`. The order is deliberate: the IDT must contain the timer handler **before** we enable interrupts. Otherwise the first tick would hit an empty IDT entry, causing a fault that becomes a double fault.

### The timer handler

The **PIT** (Programmable Interval Timer) is wired to IRQ0. Without any configuration it fires about 18.2 times per second.

```rust
extern "x86-interrupt" fn timer_interrupt_handler(_stack_frame: InterruptStackFrame) {
    print!(".");
    unsafe {
        PICS.lock().notify_end_of_interrupt(InterruptionIndex::Timer.as_u8());
    }
}
```

### End of Interrupt (EOI)

The PIC won't deliver another interrupt from a line until the kernel confirms the current one has been handled. That confirmation is the **End of Interrupt** command. Without it you'd see exactly one `.` and then nothing. `ChainedPics` works out from the vector number whether only the primary PIC needs the EOI, or both (for IRQ 8–15).

The result: after boot, a stream of dots is printed across the screen, one per timer tick.

---

## 10. Shared state: why everything is a `Mutex` in a `lazy_static`

The globals `WRITER`, `SERIAL1`, `IDT`, `GDT`, `TSS`, and `PICS` all follow similar patterns, for two separate reasons.

### `lazy_static!`: initialization that can't happen at compile time

Rust `static`s must be computed at compile time. Building an IDT, calling `Uart16550Tty::new_port`, or turning a raw pointer into a reference can't be done in a `const` context. `lazy_static!` delays initialization until the first time the value is accessed. The `spin_no_std` feature makes it work without `std`.

`PICS` does not need `lazy_static` because `ChainedPics::new` and `spin::Mutex::new` are both `const fn`.

### `spin::Mutex`: mutation of a global

`WRITER`, `SERIAL1`, and `PICS` need to be changed after creation, and Rust rightly forbids `static mut` access without `unsafe`. A mutex provides safe interior mutability. There is no OS to put a waiting thread to sleep, so we use a **spinlock**: a thread that wants the lock loops until it's free.

`IDT`, `GDT`, and `TSS` are never changed after creation, so they need no mutex.

---

## 11. Deadlocks: locks vs. interrupts

**Files:** `src/vga_buffer.rs`, `src/serial.rs`

### The problem

Once interrupts are enabled, a handler can run *in the middle of* any other code, including code that is holding a lock:

```
 main code                         timer interrupt
 ─────────                         ───────────────
 println!("...")
   WRITER.lock()   ✔ lock held
   writing bytes...
        ◄──────── tick! CPU jumps into handler ────────
                                   print!(".")
                                     WRITER.lock()  ✖ already held
                                     spin... spin... forever
```

The handler waits for a lock that the interrupted code holds, and the interrupted code can't continue until the handler returns. Neither side can make progress: a **deadlock**.

This is different from a deadlock between two threads. There is only **one** CPU and **one** flow of control here. The interrupt handler is *borrowing* the CPU from the code it interrupted, so the lock holder literally cannot run until the handler finishes. A spinlock can never be released in this situation.

Before the fix, `cargo test` reliably hung in `test_println_many`. Printing 200 lines takes long enough that a timer tick is almost certain to land while `WRITER` is locked.

### The fix: no interrupts while holding the lock

Both `_print` functions now take the lock inside `without_interrupts`:

```rust
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    use x86_64::instructions::interrupts;

    interrupts::without_interrupts(|| {
        WRITER.lock().write_fmt(args).unwrap();
    });
}
```

`without_interrupts` does three things:

1. Saves whether interrupts were enabled, then runs `cli` (clear interrupt flag).
2. Runs the closure. The lock is taken *and released* inside it.
3. Re-enables interrupts (`sti`) only if they were enabled before.

Step 3 matters. Interrupt handlers run with interrupts already disabled, so when the timer handler calls `print!`, `without_interrupts` must **not** turn them back on at the end. Restoring the previous state, rather than always enabling, makes the function safe to call from anywhere.

While interrupts are disabled, a timer tick isn't lost. The PIC holds it as *pending* and delivers it the moment `sti` runs. So the `.` just arrives a few microseconds later.

`serial.rs` gets the same treatment. Nothing prints to serial from an interrupt handler yet, but the first time one does, it would deadlock in exactly the same way. Fixing both now means it can't happen later.

**Rule of thumb:** any lock that an interrupt handler might take must only be held with interrupts disabled. Keep those sections short, because interrupts that wait too long add latency to every device.

### A related race in `test_println_output`

This test prints a line and then reads it back from the screen buffer, expecting it on the second-to-last row (`BUFFER_HEIGHT - 2`). The `_print` fix stops the deadlock, but leaves a gap *between* the print and the check:

```
println!(s)          ← lock taken and released
      ◄── tick: "." is printed on the bottom row,
          or wraps a line, and shifts everything up
read row BUFFER_HEIGHT - 2   ← no longer holds `s`  ✖
```

The fix is to make print-and-check one uninterruptible step, holding a single lock for the whole test:

```rust
interrupts::without_interrupts(|| {
    let mut writer = WRITER.lock();
    writeln!(writer, "\n{}", s).expect("writeln failed");
    for (i, c) in s.chars().enumerate() {
        let screen_char = writer.buffer.chars[BUFFER_HEIGHT - 2][i].read();
        assert_eq!(char::from(screen_char.ascii_character), c);
    }
});
```

Two details:

- It writes through the `writer` guard with `writeln!` (which is why `core::fmt::Write` is imported), not with `println!`. `println!` would try to take `WRITER` again.
- The leading `\n` starts a fresh line, in case an earlier `.` left the cursor partway along the bottom row.

> **Spinlocks are not re-entrant.** Inside this closure, calling `WRITER.lock()` a second time, or calling `println!`, would deadlock on the guard that `writer` already holds. It would hang the same way as the interrupt case, just without any interrupt involved. Always read through the guard you already have.

---

## 12. Idling with `hlt`

**Files:** `src/lib.rs`, `src/main.rs`

Before this step, the kernel ended with `loop {}`: an empty loop that keeps the CPU busy at 100% doing nothing. You can see this on the host, where QEMU uses a full core even while the kernel is idle.

`hlt` is the x86 instruction for "stop executing until the next interrupt arrives". The CPU drops into a low-power state, wakes up to run the interrupt handler, and then continues with the instruction after `hlt`. Wrapping it in a loop gives an idle loop that sleeps between interrupts:

```rust
pub fn hlt_loop() -> ! {
    loop {
        x86_64::instructions::hlt();
    }
}
```

Each timer tick now wakes the CPU, the handler prints a `.`, `iretq` returns to just after `hlt`, the loop goes round, and the CPU sleeps again. Host CPU usage drops to almost nothing.

`hlt_loop` returns `!`, so it can be the last thing in any function that must never return. It replaces `loop {}` in:

- `kernel_main` in `main.rs` (after boot),
- the normal panic handler in `main.rs`,
- the test entry point (`test_kernel_main`) and `test_panic_handler` in `lib.rs`,
- the page fault handler (section 14).

Two things to know:

- **`hlt` only wakes on an interrupt.** If interrupts are disabled, for example after a panic inside an interrupt handler, `hlt_loop` sleeps forever. That's fine for a panic, since we want to stop anyway. If a panic happens in normal code, interrupts are still on, so timer dots keep appearing after the panic message.
- **The integration tests in `tests/` still use `loop {}`.** Those spots are only reached after `exit_qemu`, which ends QEMU straight away, so the loop body never actually runs.

---

## 13. Keyboard input

**Files:** `src/interrupts.rs`. This matches the "Keyboard Input" part of the [Hardware Interrupts](https://os.phil-opp.com/hardware-interrupts/) post.

### From key press to interrupt

The keyboard does not talk to the CPU directly. It sends bytes to the **PS/2 controller**, a chip on the motherboard. Even USB keyboards are usually emulated as PS/2 by the firmware, and QEMU emulates a PS/2 keyboard. When the controller has a byte ready, it raises **IRQ 1** on the primary PIC.

```
 key pressed ─► keyboard ─► PS/2 controller ─► PIC IRQ1 ─► vector 33 ─► keyboard_interrupt_handler
                                   │
                         byte waits in I/O port 0x60
```

Because the primary PIC was remapped to start at 32, IRQ 1 arrives as vector **33**. The enum gets this for free: `#[repr(u8)]` enum values count up from the previous one, so `Keyboard` comes right after `Timer = 32`.

```rust
pub enum InterruptionIndex {
    Timer = PIC_1_OFFSET,   // 32, IRQ0
    Keyboard,               // 33, IRQ1
}
```

The handler is registered in the IDT alongside the timer:

```rust
idt[InterruptionIndex::Keyboard.as_u8()].set_handler_fn(keyboard_interrupt_handler);
```

### Scancodes

The keyboard doesn't send characters. It sends **scancodes**: numbers that identify a *physical key* and whether it went **down** or **up**. Turning that into the character `A` is the kernel's job, and depends on layout, Shift, Caps Lock, and so on.

The handler reads one scancode byte from the controller's data port:

```rust
let mut port = Port::new(0x60);
let scancode: u8 = unsafe { port.read() };
```

**Reading the port is required, not optional.** The controller won't send the next byte, or raise another interrupt, until the current one has been read. A handler that only sends EOI without reading port `0x60` sees exactly one key press and then silence. That's the same symptom as forgetting EOI, but for a different reason.

There are three scancode *sets*. By default the PS/2 controller translates everything to **set 1**, the original IBM XT format, which is why the code uses `ScancodeSet1`. In set 1:

| Event | Bytes | Example: the `A` key |
|---|---|---|
| Key down ("make" code) | one byte below `0x80` | `0x1E` |
| Key up ("break" code) | the make code with the top bit set (`make \| 0x80`) | `0x9E` |
| Extended keys (arrows, right Ctrl...) | prefixed with `0xE0` | right arrow: `0xE0 0x4D` |

Each byte arrives as a **separate interrupt**. A single press of right arrow causes two interrupts, and releasing it causes two more. Decoding therefore needs memory that lasts *between* interrupts, which is what the `pc-keyboard` crate provides.

### Decoding with `pc-keyboard`

```rust
static KEYBOARD: Mutex<Keyboard<layouts::Us104Key, ScancodeSet1>> =
    Mutex::new(Keyboard::new(
        ScancodeSet1::new(),     // how to read the bytes
        layouts::Us104Key,       // how keys map to characters (US layout)
        HandleControl::Ignore,   // Ctrl+A gives 'a', not the control character U+0001
    ));
```

Decoding happens in two stages:

```
 scancode byte ──add_byte()──► Option<KeyEvent> ──process_keyevent()──► Option<DecodedKey>
                (set 1 parser:                    (layout + modifier state:
                 handles 0xE0 prefixes,            tracks Shift/Ctrl/Alt/Caps,
                 make vs. break)                   picks the character)
```

1. **`add_byte`** feeds one byte into the scancode parser. It returns `Ok(None)` when it needs more bytes (after an `0xE0` prefix) and `Ok(Some(KeyEvent))` once it has a complete event such as "key `A`, went down".
2. **`process_keyevent`** updates the modifier state. For example, Shift down sets a flag, and Shift up clears it. It then applies the layout and returns:
   - `DecodedKey::Unicode(char)` for keys that produce text: `a`, `A` (with Shift), `!`, space, `\n` for Enter.
   - `DecodedKey::RawKey(KeyCode)` for keys that don't: `F1`, arrows, and the modifier keys themselves.
   - `None` for events that produce nothing, such as most key *releases*.

The handler prints Unicode keys as characters and raw keys with `{:?}`. That's why pressing Shift+R shows `LShiftR` on screen: `RawKey(LShift)` is printed as `LShift` when Shift goes down, then `Unicode('R')` is printed when R is pressed.

### Why `KEYBOARD` is a `static` inside the handler

The decoder's state (modifiers held, half-received `0xE0` sequences) must survive from one interrupt to the next, so it can't be a local variable. Declaring the `static` *inside* the handler function keeps it private: no other code can even name it.

The post wraps this in `lazy_static!`. This project uses a plain `static` instead, which works because `Keyboard::new`, `ScancodeSet1::new` and `spin::Mutex::new` are all `const fn` in the versions used. The value is built at compile time, with no first-use check at runtime.

It still needs a `Mutex` because Rust doesn't allow mutating a `static` without one (section 10). The lock can never deadlock: only this handler takes it, and the handler runs with interrupts disabled, so it can't interrupt itself.

### Printing from the handler

`print!` here goes through the interrupt-safe `_print` from section 11. Interrupts are already off inside a handler, so `without_interrupts` leaves them off. If the main code happens to be printing when a key is pressed, the keyboard interrupt is simply delivered once that print finishes.

### End of Interrupt

As with the timer, the handler finishes by sending EOI for the keyboard vector. Otherwise the PIC would block every further IRQ 1, and all lower-priority IRQs as well:

```rust
unsafe {
    PICS.lock().notify_end_of_interrupt(InterruptionIndex::Keyboard.as_u8());
}
```

---

## 14. Paging: virtual memory

**Files:** `src/interrupts.rs` (page fault handler). This matches the [Introduction to Paging](https://os.phil-opp.com/paging-introduction/) post.

### Virtual vs. physical addresses

Every address the kernel uses, such as a pointer, the stack pointer, or the instruction pointer, is a **virtual address**. The CPU's **memory management unit (MMU)** translates each one to a **physical address**, an actual location in RAM or a device, on every single memory access.

Memory is managed in fixed-size blocks:

- a **page** is a 4 KiB block of *virtual* address space,
- a **frame** is a 4 KiB block of *physical* memory,
- a **page table** says which frame, if any, each page maps to.

This is what lets every program have its own address space, lets the kernel protect memory (read-only, no-execute, kernel-only), and lets an unmapped **guard page** catch stack overflows (section 8).

### Four-level page tables

On x86_64, a virtual address is split into five parts. Only the low 48 bits are used. Bits 48–63 must be copies of bit 47 (a *canonical* address), or the CPU faults.

```
 63      48 47     39 38     30 29     21 20     12 11          0
┌──────────┬─────────┬─────────┬─────────┬─────────┬─────────────┐
│ sign ext │ L4 idx  │ L3 idx  │ L2 idx  │ L1 idx  │   offset    │
│ (unused) │ 9 bits  │ 9 bits  │ 9 bits  │ 9 bits  │   12 bits   │
└──────────┴─────────┴─────────┴─────────┴─────────┴─────────────┘
```

Each page table is exactly one 4 KiB frame holding 512 entries of 8 bytes, so 9 bits pick an entry. The 12-bit offset picks a byte within the 4 KiB page. To translate, the MMU walks the tables:

```
 CR3 ──► Level 4 table ──[L4 idx]──► Level 3 table ──[L3 idx]──► Level 2 table
                                                                      │
                                                                  [L2 idx]
                                                                      ▼
 physical address ◄── + offset ◄── frame ◄──[L1 idx]── Level 1 table
```

- The **`CR3` register** holds the *physical* address of the level 4 table.
- Each entry holds the *physical* address of the next table (or, at level 1, of the final frame) plus **flags**: `PRESENT`, `WRITABLE`, `USER_ACCESSIBLE`, `NO_EXECUTE`, `HUGE_PAGE`, and so on.
- If an entry at level 3 or 2 has `HUGE_PAGE` set, it points straight at a 1 GiB or 2 MiB frame and the walk stops early.

Walking four tables on every access would be slow, so the CPU caches recent translations in the **TLB** (translation lookaside buffer). The TLB is *not* updated automatically when a page table changes. After changing a mapping, the kernel must flush that page's TLB entry with the `invlpg` instruction, or the CPU may keep using the old translation.

### Page faults

If the walk finds a missing entry, or the access isn't allowed (a write to a read-only page, for example), the CPU raises a **page fault** (exception 14). Before calling the handler it:

- stores the address that was accessed in the **`CR2` register**, and
- pushes an **error code** describing what went wrong.

```rust
extern "x86-interrupt" fn page_fault_handler(
    stack_frame: InterruptStackFrame,
    error_code: PageFaultErrorCode,
) {
    use x86_64::registers::control::Cr2;

    println!("EXCEPTION: PAGE FAULT");
    println!("Accessed Address: {:?}", Cr2::read());
    println!("Error code: {:?}", error_code);
    println!("{:#?}", stack_frame);
    hlt_loop();
}
```

`PageFaultErrorCode` decodes the error code bits:

| Flag | Meaning when set |
|---|---|
| `PROTECTION_VIOLATION` | The page *was* present, but the access wasn't allowed. If clear, the page wasn't mapped at all. |
| `CAUSED_BY_WRITE` | The access was a write (otherwise a read). |
| `USER_MODE` | The access came from user mode (ring 3). |
| `MALFORMED_TABLE` | A reserved bit was set in a page table entry. |
| `INSTRUCTION_FETCH` | The CPU was fetching an instruction (no-execute violation). |

The handler can't fix anything yet, so it reports and stops in `hlt_loop()`. It can't simply return: the CPU would retry the same instruction and fault again, forever. Unlike the timer and keyboard handlers, no EOI is sent, because a page fault is a CPU exception, not a PIC interrupt.

Before this handler existed, a page fault had no IDT entry, which escalated into a double fault. Now the actual cause and address are printed.

---

## 15. Reaching the page tables: `BootInfo` and the physical memory offset

**Files:** `Cargo.toml`, `src/main.rs`, `src/lib.rs`, `src/memory.rs`. This matches the [Paging Implementation](https://os.phil-opp.com/paging-implementation/) post.

### The chicken-and-egg problem

Page table entries contain **physical** addresses, but the kernel can only use **virtual** addresses. Reading `CR3` gives the physical address of the level 4 table, and dereferencing that number as a pointer would go through the MMU and land somewhere else entirely.

The kernel needs *some* virtual address that maps to each page table frame. This project uses the simplest approach: **map all of physical memory** into the virtual address space at a fixed offset.

```
 virtual address space                          physical memory
┌────────────────────────────┐
│ kernel, stack, VGA, ...    │
├────────────────────────────┤ ◄─ physical_memory_offset
│ physical frame 0           │ ──────────────► frame 0
│ physical frame 1           │ ──────────────► frame 1
│ ...                        │                  ...
│ physical frame N           │ ──────────────► frame N
└────────────────────────────┘
```

Then **virtual = physical + offset** for every frame, including every page table.

Turning on the bootloader's `map_physical_memory` feature makes it set up this mapping before jumping to the kernel:

```toml
bootloader = { version = "0.9", features = ["map_physical_memory"] }
```

The bootloader chooses the offset, so the kernel has to be told what it is.

### `entry_point!` and `BootInfo`

The bootloader passes a pointer to a **`BootInfo`** struct as the first argument to the entry point. It contains `physical_memory_offset` and also a `memory_map` of which physical regions are usable RAM, which the next step of the post uses.

We could just add a `&'static BootInfo` parameter to `_start`, but nothing would check that the signature matches what the bootloader really passes. The `entry_point!` macro solves this:

```rust
entry_point!(kernel_main);

fn kernel_main(boot_info: &'static BootInfo) -> ! { ... }
```

It generates the real `#[no_mangle] extern "C" fn _start(boot_info: &'static BootInfo)` for us, and it **type-checks** that `kernel_main` has exactly the right signature. `kernel_main` is now an ordinary Rust function, with no `no_mangle` or `extern "C"` needed.

The library's test entry point in `lib.rs` uses the same macro (`entry_point!(test_kernel_main)`, only under `cfg(test)`). The integration tests in `tests/` still define `_start` by hand. That's fine because they don't need `BootInfo`.

### Getting the level 4 table: `memory::init`

```rust
unsafe fn active_level_4_table(physical_memory_offset: VirtAddr) -> &'static mut PageTable {
    let (level_4_table_frame, _) = Cr3::read();              // physical address
    let phys = level_4_table_frame.start_address();
    let virt = physical_memory_offset + phys.as_u64();       // where we can reach it
    let page_table_ptr: *mut PageTable = virt.as_mut_ptr();
    unsafe { &mut *page_table_ptr }
}
```

`memory::init` wraps that table in an `OffsetPageTable`, a type from the `x86_64` crate that knows the "physical + offset" trick and can translate addresses and create mappings for us.

Both functions are `unsafe`, and the caller is responsible for two promises the compiler can't check:

1. **All physical memory really is mapped at `physical_memory_offset`.** If not, the pointer arithmetic reads garbage.
2. **`init` is called only once.** Each call creates a new `&'static mut` to the same level 4 table. Two mutable references to the same data break Rust's aliasing rules, which is undefined behaviour.

### Translating by hand: `translate_addr`

`translate_addr` does in software exactly what the MMU does in hardware, which makes it a good way to understand the walk:

1. Read `CR3` to get the level 4 frame.
2. For each level, use the matching 9-bit index from the address (`p4_index()`, `p3_index()`, ...).
3. Turn the current table's frame into a virtual address (`offset + physical`) and read the entry.
4. If the entry isn't present, return `None` (the address is unmapped). Otherwise continue with the frame it points to.
5. After level 1, add the 12-bit page offset to the final frame address.

The public `unsafe fn translate_addr` only wraps a safe inner function. This keeps the `unsafe` region small: the inner function is safe *given* the promise the caller made about the offset.

It has one limitation. It panics on huge pages, and the bootloader may use huge pages for the physical memory mapping itself. `OffsetPageTable` handles huge pages correctly, so the kernel uses that, and `translate_addr` remains as a teaching example.

---

## 16. Creating a mapping

**Files:** `src/memory.rs`, `src/main.rs`

### `map_to` and frame allocators

```rust
pub fn create_example_mapping(
    page: Page,
    mapper: &mut OffsetPageTable,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) {
    let frame = PhysFrame::containing_address(PhysAddr::new(0xb8000));   // the VGA buffer
    let flags = Flags::PRESENT | Flags::WRITABLE;

    let map_to_result = unsafe { mapper.map_to(page, frame, flags, frame_allocator) };
    map_to_result.expect("map_to failed").flush();
}
```

`map_to` writes a level 1 entry that points `page` at `frame` with the given flags. If a level 3, 2 or 1 table along the path doesn't exist yet, it needs a fresh physical frame to hold the new table. It asks the **frame allocator** for one.

`map_to` returns a `MapperFlush`, which is marked `#[must_use]`. Calling `.flush()` runs `invlpg` for that page to clear its stale TLB entry, so the compiler warns you if you forget.

`map_to` is `unsafe` because a mapping can break memory safety. For example, mapping a frame that is already in use somewhere else gives two writable views of the same memory.

### Why the empty allocator works (for now)

```rust
pub struct EmptyFrameAllocator;

unsafe impl FrameAllocator<Size4KiB> for EmptyFrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame> { None }
}
```

This allocator never hands out a frame, so `map_to` can only succeed when every table along the path already exists. Virtual page `0` satisfies that. The bootloader runs in the first megabyte of the address space, so the level 3, 2 and 1 tables covering that region are already there. Page 0 itself is unmapped, so `map_to` only has to fill in one level 1 entry.

Choose a page in an untouched region instead, such as `0xdeadbeaf000`, and `map_to` would need new tables, get `None` from the allocator, and the `expect` would panic with `map_to failed (FrameAllocationFailed)`. The next step of the post replaces this with a real allocator built from `boot_info.memory_map`.

The allocator trait is `unsafe impl` because the implementer promises to hand out only frames that are actually free. Handing out a frame that's already in use would corrupt memory.

### Seeing it work: "New!"

```rust
let page = Page::containing_address(VirtAddr::new(0));
memory::create_example_mapping(page, &mut mapper, &mut frame_allocator);

let vga_address = page.start_address() + 400 * size_of::<u64>() as u64;
let vga_ptr = vga_address.as_mut_ptr::<u64>();
unsafe { vga_ptr.write_volatile(0xf021_f877_f065_f04e) };
```

Virtual page 0 now maps to physical frame `0xb8000`, the VGA text buffer, so writing near virtual address 0 changes the screen:

- **Where:** 400 × 8 = 3200 bytes into the page. Each screen cell is 2 bytes, so that's cell 1600, the start of row 20 (1600 ÷ 80).
- **What:** x86 is little-endian, so the `u64` is stored as the bytes `4e f0 65 f0 77 f8 21 f0`. That's four cells of (character, color): `N`, `e`, `w`, `!`, with dark text on a light background.

"New!" ends up one row higher than row 20 on screen, because the next `println!` scrolls everything up by one line.

The VGA buffer can now be reached through two virtual addresses: `0xb8000`, which the `WRITER` uses, and page 0. Both lead to the same physical frame.

**Side effect to be aware of:** with page 0 mapped, dereferencing a null pointer, or anything in the first 4 KiB, no longer page faults. Instead it silently reads or writes the VGA buffer. This is fine for a demonstration, but page 0 should be left unmapped once the real frame allocator arrives.

---

## 17. File map

| File | Role |
|---|---|
| `src/main.rs` | Kernel entry point (`kernel_main` via `entry_point!`), paging demo, normal panic handler. |
| `src/lib.rs` | `init()`, `hlt_loop()`, test entry point (`test_kernel_main`), the test framework (`test_runner`, `Testable`), QEMU exit, test panic handler. |
| `src/vga_buffer.rs` | VGA text-mode writer, `print!` and `println!` (interrupt-safe). |
| `src/serial.rs` | UART serial output, `serial_print!` and `serial_println!` (interrupt-safe). |
| `src/interrupts.rs` | IDT, exception handlers (breakpoint, page fault, double fault), PIC setup, timer and keyboard handlers. |
| `src/memory.rs` | Page table access (`init`, `OffsetPageTable`), manual `translate_addr`, example mapping, `EmptyFrameAllocator`. |
| `src/gdt.rs` | GDT, TSS, double-fault IST stack. |
| `tests/basic_boot.rs` | Integration test: printing works at boot. |
| `tests/should_panic.rs` | Integration test: panics are reported. |
| `tests/stack_overflow.rs` | Integration test: stack overflow → handled double fault. |
| `x86_64-rc_os.json` | Custom compilation target (bare metal, no red zone, no SIMD). |
| `.cargo/config.toml` | Default target, `build-std`, `bootimage` runner. |
| `Cargo.toml` | Dependencies, `panic = "abort"`, QEMU test arguments. |

### Dependencies

| Crate | Used for |
|---|---|
| `bootloader` 0.9 | Bootloader that `bootimage` links with the kernel. Provides `entry_point!` and `BootInfo`. The `map_physical_memory` feature maps all RAM at `physical_memory_offset`. |
| `x86_64` | Safe wrappers for IDT, GDT, TSS, page tables (`OffsetPageTable`, `Mapper`, `FrameAllocator`), control registers (`CR2`, `CR3`), port I/O, and CPU instructions (`int3`, `sti`, `lidt`, `invlpg`...). |
| `pic8259` | `ChainedPics`: initializing and sending EOIs to the two 8259 PICs. |
| `pc-keyboard` | Decodes PS/2 scancodes (set 1, US layout) into key events and characters. |
| `uart_16550` | Serial port driver. |
| `spin` | Spinlock `Mutex` for `no_std`. |
| `lazy_static` | Statics that are initialized at runtime. |
| `volatile` | Prevents the compiler from optimizing away VGA buffer writes. |

---

## 18. Running it

Requirements: a nightly Rust toolchain with `rust-src`, `llvm-tools-preview`, `bootimage` (`cargo install bootimage`), and QEMU.

```sh
cargo run          # build the disk image and boot it in QEMU; click the window and type
cargo test         # run unit + integration tests headless; results print to your terminal
cargo test --test stack_overflow   # run a single integration test
```

To debug a triple-fault reboot loop, add `-no-reboot` to the QEMU arguments (bootimage already passes it during `cargo test`). Adding `-d int` shows every interrupt and exception QEMU delivers.
