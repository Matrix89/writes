Adventures in RISC-V
=

Recently I got bored of wrinting frontend at work every day. I've decided to try some lower level stuff.
I've heard a lot of people talking about this new great architecture called RISC-V and I've always liked low level stuff, so I've decided to play around with it.

As the name suggests RISC-V is a risc architecture, It only has a few instructions and is designed to be extensible. You can do everything with the base implementation but since it's so limited, most implementation use extensions to add additional functionality(like integer multiplication). The list of extensions can be found [here](https://en.wikipedia.org/wiki/RISC-V#Design). This makes the ISA really simple to implement and to implement software for it.

In this post I will explain what is needed to get from assembly to c and print a `hello world` message.

Hello RISC-V
==

When a RISC-V cpu starts all its harts(hardware threads) start executing code from the address `0x80000000`. In theory we could just write to the UART address to get things on the screen but since I don't like writing lots of assembly lets first jump to C and then display our message. To jump to C we need to setup the stack and park all the harts but one. The whole assembly code that setups stacks for each hart and parks not needed harts looks like this:

```assembly
.equ STACK_SIZE, 1024

.global _start

_start:
    # setup stacks per hart
    csrr t0, mhartid                # read current hart id
    slli t0, t0, 10                 # shift left the hart id by 1024
    la   sp, stacks + STACK_SIZE    # set the initial stack pointer 
                                    # to the end of the stack space
    add  sp, sp, t0                 # move the current hart stack pointer
                                    # to its place in the stack space

    # park harts with id != 0
    csrr a0, mhartid                # read current hart id
    bnez a0, park                   # if we're not on the hart 0
                                    # we park the hart

    j    enter                      # jump to c

park:
    wfi
    j park

stacks:
    .skip STACK_SIZE * 4            # allocate 1024 * 4 space for the harts stacks
```

In the liker script we tell the linker to put the `_start` symbol at `0x80000000`, the `enter` symbol is our entry point in C.

```c
void enter() {
}
```

*Awesome!* we're in C, because we're going to use qemu to run the code the easiest way to get IO is to use UART, It's quite a big standart and I could write a whole post about it but for now we only need to know that according to qemu [source](https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c#L64) it's memory mapped at address `0x10000000` and writing to it will make things appear on the screen. 

UART has some registers for controlling it for now we only care about the line status register(or LSR) located at offset `0x05` from the base uart address(the full list of register can be found [here](https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming#UART_Registers)).
The LSR can tell us whether we can write data the the UART buffer, the possible values of this register are [here](https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming#Line_Status_Register)

The code for writing to UART looks like this:
```c
static int putchar(uint8_t ch) {
    static uint8_t THR    = 0x00;
    static uint8_t LSR    = 0x05;
    static uint8_t LSR_RI = 0x40;

    while ((uart[LSR] & LSR_RI) == 0);
    return uart[THR] = ch;
}
```
The uart variable is globally defined as `volatile` so that the compiler doesn't try to optimize it out.
```c
static volatile uint8_t *uart = (void *)0x10000000;
```

Because this code only writes a single character we need to create a helper function to write whole lines.
```c
void puts(char *s) {
    while (*s) putchar(*s++);
    putchar('\n');
}
```

Now we can call `puts` and hopefully we should see stuff appear on the screen:
```c
puts("Hello RISC-V");
```

Putting it all together the code looks like this:
```c
typedef unsigned char uint8_t;

static volatile uint8_t *uart = (void *)0x10000000;

static int putchar(char ch) {
    static uint8_t THR    = 0x00;
    static uint8_t LSR    = 0x05;
    static uint8_t LSR_RI = 0x40;

    while ((uart[LSR] & LSR_RI) == 0);
    return uart[THR] = ch;
}

void puts(char *s) {
    while (*s) putchar(*s++);
    putchar('\n');
}

void enter() {
    puts("Hello RISC-V");
}
```

We can compile it using gcc `riscv64-unknown-elf-gcc -march=rv32imac -mabi=ilp32 -T default.lds -o out -nostdlib -fno-builtin start.s main.c`
The `start.s` is our assembly source and the `main.c` is our c code

This should produce a file called `out` which can be run using qemu: `qemu-system-riscv32 -nographic -smp 4 -machine virt -bios none -kernel out`

We should see `Hello RISC-V` being printed in the console \o/.

This might not be much but It's a good start, for writing a whole kernel. In the future I'd love to cover things like: interrupts memory management, unprivilaged ISA.

Interesting  reads aboud RISC-V
==

[The Adventures of OS: Making a RISC-V Operating System using Rust](https://osblog.stephenmarz.com/index.html)

[Wikipedia article about RISC-V](https://en.wikipedia.org/wiki/RISC-V)

The official RISC-V ISA specification: [Volume I](https://riscv.org/wp-content/plugins/pdf-viewer/stable/web/viewer.html?file=https://content.riscv.org/wp-content/uploads/2019/12/riscv-spec-20191213.pdf) and [Volume II](https://riscv.org/wp-content/plugins/pdf-viewer/stable/web/viewer.html?file=https://content.riscv.org/wp-content/uploads/2019/08/riscv-privileged-20190608-1.pdf)
