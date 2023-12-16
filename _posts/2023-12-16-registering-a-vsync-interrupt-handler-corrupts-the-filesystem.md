---
title: "Registering a VSync interrupt handler corrupts the filesystem"
date: 2023-12-15
---

Context: I'm working on an ARM Cortex-A9 embedded system running FreeRTOS.

# The bug

After registering a VSync interrupt handler, any attempt to write to the filesystem will instantly trigger a prefetch abort exception and most of the time the filesystem will get corrupted.

# The story

I spent days trying to figure out what was going on.

At first, I thought there was a problem with the VSync interrupt handler, so I commented all the code inside it. The problem persisted.
This told me that the problem was deeper than I thought and somehow registering this interrupt handler was influencing the flash memory even though it seemed impossible because they are completely unrelated.

I looked at the ARM documentation to see how I can debug this prefetch abort exception. The docs mention that the LR register is set to the address of the instruction that caused the exception. It can be a consequence of the CPU trying to jump to an illegal virtual memory address. What really confused me is that the LR register was set to the address of the prefetch abort handler itself. That is, the prefetch abort handler was called because the prefetch abort handler could not be prefetched. I really thought I was reading the wrong documentation or that I was too dumb to understand it, because it seemed like an error.

Boy, was I wrong! The answer was literally in my face and my brain refused to acknowledge it because it seemed too absurd. It's not the first time this is happening and it won't be the last time unfortunately. After losing a lot of time putting breakpoints all over the place without gaining new knowledge, I looked more closely at the address of the prefetch abort handler. That address was part of the first page of flash memory. This is the moment I started to realize that maybe the flash API is doing something with virtual memory and the LR register didn't lie.

I decided to spend some time reading the code of the flash API hoping that maybe it gives me some clues about how and why this is happening. One function caught my eye. The function would modify the MMU to mark all pages that span the flash memory as unaccessible and non-executable. After the write was done, it would restore the MMU to its previous state. I was pretty sure this was the source of the problem, and to test my hypothesis I modified the code to skip over the first page of flash memory, and only mark the rest of them as unaccessible.

I got a slightly different error, which means progress! I still got a prefetch abort exception, but this time the LR register pointed to the IRQ handler, and the interrupt that was handled was VSync. I was like "What? The flash API already wraps everything in a critical section, how is it possible that I still get interrupts here?". I did some research and found out that ARM CPUs have 2 types of interrupts. Those with priority smaller than configMAX_SYSCALL_INTERRUPT_PRIORITY will not be disabled by FreeRTOS even in critical sections, and also cannot call FreeRTOS API functions. This explains why FreeRTOS was complaining when I once tried to notify a task from the interrupt handler! I then changed the priority of the VSync handler and everything was working as expected.

# The lesson

Know your architecture, it will save you lots of time and headaches.

Also, copy your interrupt handlers to RAM to make sure they are always available. When flash memory is unavailable and the CPU throws an exception, you're screwed because the CPU will hide the problem from you by throwing this very confusing prefetch abort exception saying that the prefetch abort handler is inaccessible.
