# Spectre

Spectre describes a family of vulnerabilities that arise due to problems with [speculative execution](https://en.wikipedia.org/wiki/Speculative_execution) in modern processors.

This repository functions as distillation of the original [paper](https://spectreattack.com/spectre.pdf) that describes the attacks, as well a source of information
about how modern processors are implemented, eventually leading up to the Spectre attacks.

# Table of contents
- [Speculative Execution](#speculative-execution)
- [Branch Prediction](#branch-prediction)
- [Side Channel Attacks](#side-channel-attacks)
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
TODO

# Side Channel Attacks
TODO

# Cache Lines
TODO
