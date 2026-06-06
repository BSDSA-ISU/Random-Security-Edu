# Security and Vulnerabilities

> here you can find some Vulnerabilities known, feel free to add some by making a pr

![cpu](https://media1.tenor.com/m/N80hQiCKCAUAAAAC/cpu-i7.gif)

- [Security and Vulnerabilities](#security-and-vulnerabilities)
  - [CPU Vuln](#cpu-vuln)
    - [Rogue Data Cache Load - CVE-2017-5754 (MELTDOWN)](#rogue-data-cache-load---cve-2017-5754-meltdown)
    - [Bounds Check Bypass - CVE-2017-5753 (Spectre V1)](#bounds-check-bypass---cve-2017-5753-spectre-v1)
    - [Branch Target Injection - CVE-2017-5715 (Spectre V2)](#branch-target-injection---cve-2017-5715-spectre-v2)
    - [Speculative Store Bypass - CVE-2018-3639 (Spectre V4)](#speculative-store-bypass---cve-2018-3639-spectre-v4)

## CPU Vuln

### Rogue Data Cache Load - CVE-2017-5754 (MELTDOWN)

Meltdown is a hardware vulnerability that affects how processors (CPUs) handle "speculative execution." It allows a rogue process to read memory that it shouldn't have access to—specifically, data stored in the kernel memory (the core of the Linux operating system).

- **affected:**
  - Almost all Intel processors manufactured since 1995 are affected.
  - Some ARM processors are also impacted, though it is primarily an Intel-focused issue.
- **Patch**
  - Kernel Page Table Isolation(KPTI)

### Bounds Check Bypass - CVE-2017-5753 (Spectre V1)

Triggers speculative execution to access out-of-bounds memory by bypassing bounds-checking code tricking the CPU into reading memory it isn't supposed to, specifically within the same application's memory space.

- **affected:**
  - Almost all modern high-performance CPU (Intel, AMD, ARM) that predict branches in code.
- **Patch:**
  - **Retpolines (Return Trampolines)**: This is a clever Linux/GCC technique that prevents the CPU from being "trained" to jump to malicious locations. It basically confuses the CPU’s branch predictor so it doesn't know where to guess, forcing it to wait for the actual, verified address.

### Branch Target Injection - CVE-2017-5715 (Spectre V2)

Manipulates and poisons the CPU's branch predictor to force software to speculatively execute code fragments known as "gadgets".

- **How the attack works:**
  - 1. Speculation the CPU makes a guess, Modern CPUs don’t wait for branches like:

    ```c
       if (x < array_size)
       y = array[x];
      ```

    Instead, they use a Branch Target Buffer (BTB) to predict:

    “This branch usually goes here”
    “Jump to this function address next”

    So the CPU:

    guesses the target
    executes instructions speculatively
    before it even knows if the guess was correct

    If it guessed wrong → results are discarded but microarchitectural traces remain (cache).

- **affected:**
  - Almost all modern high-performance CPU (Intel, AMD, ARM) that predict branches in code.
- **Patch:**
  - **Retpolines (Return Trampolines)**: This is a clever Linux/GCC technique that prevents the CPU from being "trained" to jump to malicious locations. It basically confuses the CPU’s branch predictor so it doesn't know where to guess, forcing it to wait for the actual, verified address.
  - **IBRS/IBPB (Indirect Branch Restricted Speculation/Prediction Barrier):** hardware-level CPU security mechanisms designed to mitigate Branch Target Injection (also known as Spectre v2) attacks. They prevent malicious code from hijacking a processor's speculative execution.

### Speculative Store Bypass - CVE-2018-3639 (Spectre V4)

SSB is a hardware security vulnerability affecting modern Intel, AMD, and ARM processors. It exploits performance-enhancing "speculative execution" out-of-order processing to trick a CPU into reading sensitive, privileged memory before a pending write (store) is completed.

- **How the Attack Works:**
  1. **Speculation**: Modern CPUs guess what code will execute next to speed up performance.
  2. **The Bypass**: If a read (load) instruction happens before the processor knows the value of an overlapping write (store), the CPU skips waiting and speculatively reads the old data from the memory.
  3. **Data Leakage**: The processor eventually realizes its mistake, discards the incorrect speculation, and performs the correct calculation. However, the data read during the brief speculation leaves traces in the system cache, which an attacker can infer through side-channel attacks to steal private information.
- **affected:**
  - modern Intel, AMD, and ARM processors
- **mitigations:**
  - Speculative Store Bypass Disable (SSBD)