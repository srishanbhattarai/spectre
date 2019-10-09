# Spectre

Spectre describes a family of vulnerabilities that arise due to problems with [speculative execution](https://en.wikipedia.org/wiki/Speculative_execution) in modern processors.

This repository functions as distillation of the original [paper](https://spectreattack.com/spectre.pdf) that describes the attacks, as well a source of relevant surrounding information
about how modern processors are implemented, eventually leading up to the Spectre attacks.

# Table of contents
- [Speculative Execution](#speculative-execution)
- [Branch Prediction](#branch-prediction)
- [Multi-layered Memory Hierarchy](#multi-layered-memory-hierarchy)
- [Side Channel Attacks](#side-channel-attacks)
- [Return Oriented Programming](#return-oriented-programming)
- [Transient Instructions](#transient-instructions)
- [Anatomy of a Spectre attack](#side-channel-attacks)
- [Cache Lines](#cache-lines)

# Speculative Execution

tl;dr: Hardware optimization technique to allow instructions to execute *out-of-order* but guaranteeing that they are truly committed *in-order* to prevent any invalid states that would arise had the speculation guessed incorrectly. The invariant about reverting to clean states is not always guaranteed.

Modern microprocessors, especially [wide-issue](https://en.wikipedia.org/wiki/Wide-issue) ones, try to maintain maximal performance by keeping the CPU busy at all times, using one or more hardware level optimization techniques, collectively called speculative execution (or hardware based speculative execution). The CPU will prematurely start processing instructions that fall into the likely execution path. 

## Example
Consider a situation where the program flow depends on an uncached value in main memory. Wasting hundreds of clock cycles to fetch the value, whilst keeping the CPU idling is not wise. The difference between CPU speed and main memory speeds continue to grow apart, and is often called the "memory wall".

![Memory Wall](images/memory%20wall.png)

Instead, the CPU, or a branch predictor, guesses the direction of the control flow, and proceeds to speculatively execute the instructions on the guessed path. When the value eventually does arrive from memory, the results are either committed, or rolled back completely in case of a wrong guess. A wrong guess could provide performance comparable to idling, but a correct guess could provide a tremendous boost to clock cycle utilization.

However, the invariant that any misspeculation on the CPUs part will be reverted, is not held in every case, leading to a series of microarchitecture attacks, collectively coined as *Spectre*.

# Branch Prediction
Branch prediction is an optimization technique, where the CPU (or a separate chip altogether) has the dedicated task of *improving* guesses during speculative execution i.e. branch predictors concern themselves with increasing the percentage of
speculatively executed instructions that can be committed. 

| Branch type | Example                                 | Remarks |
|-------------|-----------------------------------------|--------------------|
| Direct      | `jmp eax`                               | Straightforward - monotonic/static predictions    |
| Indirect    | x86 - `jmp [eax]` ARM - `MOV pc, r14`   | Trickier - uses recent program behavior in addition to monotonic predictions        |
| Stack jump  | `RET`                                   | Could be thought of as an indirect branch, but in practice is implemented using a Return Stack Buffer (RSB) instead of a Branch Target Buffer (BTB)        |
|             |                                         |         |

BTB, and RSB are typically local to each core. Thus, branch predictor training can only happen locally on that core.

# Multi-layered Memory Hierarchy
Modern architectures have a hierarchical memory design - from slow but spacious to fast but small memory types as we move closer to the CPU. For example, many modern processors have a small L1 and a (larger) L2 cache on-chip. A larger L3 cache will be shared by all cores. 

As discussed above, the difference between CPU and memory speeds makes it such that a Load/Store instruction into main memory will stall the CPU for hundreds of cycles. Instead, processors try to achieve better temporal and spatial locality by caching data as close to the CPU as possible. Several policies pertaining to cache mapping, replacement and writing exist, but are beyond the scope of this discussion.

Modern caches almost ubiquitously cache data in chunks, often called a cache line, which maps to a block in main memory. Whenever CPU needs to fetch data from memory i.e. a cache miss has happened, it pulls an entire cache line worth of data corresponding to the block which the memory location belongs to, into the cache. Typical cache line sizes are 64 or 128 bytes. There is a tradeoff to having arbitrarily large cache lines because it starts affecting cache hit times.

## False Sharing
Consider the following situation: instructions are being executed in a *true* concurrent setting i.e. there are multiple instructions executing at the same instant across multiple cores. Each core has it's own L1 and L2 cache. Each instruction
of a program can execute in any of the cores. Whenever a data is cached in one core, it might not be available in the L1 of another core. Even more problematic is the fact that when data is updated in an L1 cache of one core, all other cores now have a stale copy of the data. This assumes some form of write-back policy in action i.e. the CPU does not flush each cache write to memory. This is a *cache coherency* problem and is a *hardware concern* - programmers do not need to deal with it. The hardware synchronizes multiple cores
to mark L1 caches as dirty, which means successive accesses to this memory in L1 will be cache misses.

However, this exhibits what is called *cache line bouncing*. In the worst case, each L1 cache write dirties every other core such that every read is a cache miss. Furthermore, this is compounded by the fact that cache works in *lines*, i.e. even if different cores are accessing completely different memory locations that belong to the same cache line, the bouncing effect is still visible. This behavior is called *false sharing*.

x86 provides a [clflush](https://www.felixcloutier.com/x86/clflush) instruction to flush cache lines. An attacker could initiate a flush to evict a memory location that he controls, but if performed correctly, could also flush the victim's data, since they exist on the same cache line(s).

# Side Channel Attacks
A [side-channel attack](https://en.wikipedia.org/wiki/Side-channel_attack) targets implementation of computer systems, rather than particular algorithms or code. When multiple programs execute on the CPU, either concurrently or through scheduling, there are microarchitectural changes that happen which can leave the system vulnerable. These include timing attacks, differential power analysis etc. and can affect a variety of components like the L1 instruction/data cache, BTB, Address Space Layout Randomization etc.

## Cache Timing Attacks
An attacker begins by evicting a cache line affiliated with the victim. After the victim executes for a while, the attacker evaluates the time it takes to perform a read at the corresponding address of the evicted cache line. If the victim accessed that cache line, the data will now be in the cache, and access will be fast, otherwise the CPU needs to go out to memory and the access will be slow. By measuring access time, the attacker can determine if the victim has read the cache line between eviction and a subsequent probes.

The process of eviction can be either *flush + reload*, where as discussed above, the attacker can use an instruction like *clflush* or *evict + reload* where the attacker can force contention on the same cache line by accessing other memory locations that map to the same cache line, which causes the cache line already present to be evicted.

# Return Oriented Programming
ROP is a technique to allow an attacker to hijack control flow to make the victim execute instructions in it's address space. It is accomplished by chaining *gadgets* in the victim's vulnerable code. Gadgets are snippets of machine code. The attacker starts by finding usable gadgets in the victim's binary. While this requires intricate knowledge of the victim's binary and instruction layout, it [may not be impossible](https://www.radare.org/r/) to reverse engineer. The gadgets perform some arbitrary computation before executing a `return` instruction. By controlling the stack, the attacker can chain a series of addresses to be returned to by the instruction, effectively causing the victim to execute a series of gadgets in succession. How the attacker controls the stack could mean a variety of options - such as a buffer overflow attack.

# Transient Instructions
The Spectre paper describes *transient instructions* as instruction sequences that Spectre tricks the processor into executing. The effects of the instructions on the CPU are eventually reverted - hence, *transient*.

# Anatomy of a Spectre attack
Recall that Spectre is a family of related attacks with several variants, but the general approach is to exploit a vulnerability in the processor where it can leak information across security domains (between native code and JavaScript, for example) by violating memory boundaries. At a high level, the following occurs:

1. Attacker identifies/introduces a sequences of instructions in victim's address space that can leak information via some form of *covert channel*.
2. Attacker tricks the CPU into speculatively executing these (transient) instructions which leaks information onto the *covert channel*
3. Even if the state of the system reverts when misspeculated, the covert channel has already leaked the information. For example, cache contents can still be retained after nominal state reversion.

Two questions arise: 
1. What constitues of a covert channel?
  - This can potentially be anything that might constitute a [side-channel attack](https://en.wikipedia.org/wiki/Side-channel_attack), but the paper limits the scope to cache-based covert channels.
  
2. How does an attacker induce the CPU into executing erroneous transient instructions, and leaking information onto the covert channel?
  - Enter Spectre variants

# Variant 1: Conditional Branch Misprediction
TODO

# Variant 2: Poisoning Indirect Branching
TODO

# Meltdown
TODO
