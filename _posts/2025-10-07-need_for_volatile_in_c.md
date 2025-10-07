---
title: Why we need "volatile" in C
# date: YYYY-MM-DD HH:MM:SS +/-TTTT
date: 2025-10-07 22:02:49 +0530
categories: [technical, embedded]
tags: [volatile, compiler, optimization, debugging, c]
---

## Intro to Compilers

Compilers — this critical piece of software acts as the very bridge between imagination and reality, enabling developers to run code on all sorts of hardware — everything from tiny embedded systems to massive data centers. 

A compiler not only translates code, but also optimizes it — reordering, simplifying, and even removing instructions to improve performance. 

While this is generally beneficial, it can sometimes clash with the programmer’s intent, especially in systems where precise control over hardware and memory is crucial (e.g., embedded systems). 

This is where the `volatile` keyword comes into play, guiding the compiler to respect certain constraints to ensure correct behavior in concurrent and low-level code.

## The Setup

Here we have a simple piece of code where a flag, `system_ready`, is used to busy wait before starting the actual application code. This flag is accessed by other code files using `extern` to signal to the main program that it is ready to proceed.

``` c
#include "main.h"

uint8_t system_ready = 0;

int main() {
    /* System Init */

    // Busy wait until the flag is set and indicate by LED when ready
    while (system_ready == 0);
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);

    while (1) {
        /* Looping user code */
    }
}
```

On compilation, this is what we expect to see in the disassembly:

```
...

  while (system_ready == 0);
 800023a:	4b09      	ldr	r3, [pc, #36]	; (8000260 <main+0x38>)     <--- Load address of 'system_ready' to R3 <-----+
 800023c:	781b      	ldrb	r3, [r3, #0]                                <--- Load a byte from the address to R3         |
 800023e:	2b00      	cmp	r3, #0                                      <--- Compare R3 to 0                            |
 8000240:	d0fb      	beq.n	800023a <main+0x12>                         <--- If equal, loop back to address 800023a ----+
  HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);

...

 8000260:	20000028 	.word	0x20000028                                  <--- RAM address of 'system_ready' variable

...
```

Every time, we load the variable `system_ready` from memory, compare it to 0 and then either loop or break out based on the outcome of the comparison (as intended by the programmer).

But, as we increase the optimization level to `-O1` (or even `-Os`, which is the default for embedded systems), we see this instead

``` 
...

  while (system_ready == 0);
 8000236:	4b0b      	ldr	r3, [pc, #44]	; (8000264 <main+0x40>)     <--- Load address of 'system_ready' to R3
 8000238:	781b      	ldrb	r3, [r3, #0]                                <--- Load a byte from the address to R3
 800023a:	2b00      	cmp	r3, #0                                      <--- Compare R3 to 0 <--------------------------+
 800023c:	d0fd      	beq.n	800023a <main+0x16>                         <--- If equal, loop back to address 800023a ----+
  HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);

...

 8000264:	20000028 	.word	0x20000028                                  <--- RAM address of 'system_ready' variable

...
```

Here, the variable is loaded **only once** from memory, kind of like caching the variable, and the same cached value is repeatedly compared to 0. The compiler assumes the variable won’t change unexpectedly and optimizes the code by removing repeated memory reads. As a result, if the variable hasn't become non-zero *before* that initial load, the code gets stuck in an infinite loop, with no way to break out.

> Compilation results and outputs may vary depending on the compiler version, target architecture, optimization settings, and maybe even black magic.
{: .prompt-info }

This behavior would be entirely unexpected to the programmer, potentially resulting in hours — or even days — of unnecessary and most likely unproductive debugging.

## "volatile" to the rescue

`volatile` is the magic word that instructs the compiler to be mindful of the variable during optimization. It informs the compiler that its value **can change at any time**, outside the normal program flow, and hence **not to optimize** anything related to that variable.

Let’s make a small modification to the example above — we’ll add the `volatile` type qualifier to the `system_ready` variable, as shown below:

``` c
volatile uint8_t system_ready = 0;
```
{: .nolineno }

This tells the compiler **not** to optimize access to the variable and to **check the actual memory each time**, rather than relying on a cached value. The resulting disassembly (compiled with `-O3`) is shown below:

```
...

  while (system_ready == 0);
 800023a:	4a0a      	ldr	r2, [pc, #40]	; (8000264 <main+0x3c>)     <--- Load address of 'system_ready' to R2
 800023c:	7813      	ldrb	r3, [r2, #0]                                <--- Load a byte from the address to R3 <-------+
 800023e:	2b00      	cmp	r3, #0                                      <--- Compare R3 to 0                            |
 8000240:	d0fc      	beq.n	800023c <main+0x14>                         <--- If equal, loop back to address 800023c ----+
  HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);

  ...

 8000264:	20000028 	.word	0x20000028                                  <--- RAM address of 'system_ready' variable

  ...
```

And just like that, a single keyword saves us from hours of head-scratching. `volatile` to the rescue!

## TL;DR

### What it does?

The `volatile` keyword tells the compiler not to optimize reads / writes to a variable because its value might change unexpectedly (e.g., due to hardware or another thread).

### Why do we need it?

When using higher optimization levels, compilers may overlook the importance of a seemingly unchanged variable, resulting in incorrect logic in the generated binary.

### When to use it?

- Busy wait loops / Polling hardware flags
- Variable accessed / updated by ISRs or signal handlers
- Variable shared between threads / tasks / processes
- Variable mapped to hardware (e.g., memory-mapped I/O, DMA-controlled buffers)

---

Until next time, may your pointers never be null.