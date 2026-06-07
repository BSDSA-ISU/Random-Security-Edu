# Security and Vulnerabilities

> here you can find some Vulnerabilities known, feel free to add some by making a pr

![cpu](https://media1.tenor.com/m/N80hQiCKCAUAAAAC/cpu-i7.gif)

**Here are the list of known vuln:**

- [Security and Vulnerabilities](#security-and-vulnerabilities)
  - [CPU Vuln](#cpu-vuln)
    - [Rogue Data Cache Load - CVE-2017-5754 (MELTDOWN)](#rogue-data-cache-load---cve-2017-5754-meltdown)
    - [Bounds Check Bypass - CVE-2017-5753 (Spectre V1)](#bounds-check-bypass---cve-2017-5753-spectre-v1)
    - [Branch Target Injection - CVE-2017-5715 (Spectre V2)](#branch-target-injection---cve-2017-5715-spectre-v2)
    - [Speculative Store Bypass - CVE-2018-3639 (Spectre V4)](#speculative-store-bypass---cve-2018-3639-spectre-v4)
  - [Software](#software)
    - [Log4shell (CVE-2021-44228)](#log4shell-cve-2021-44228)

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

---

**How the attack works:**

**1. Speculation (CPU ignores the “wait, is this safe?” check):**

Modern CPUs hate waiting for memory, so they do this:

```c
if (index < array_size)
    value = array[index];
```

CPU behavior:

- It predicts the condition is true
- It starts executing array[index] before - confirming bounds
- This is called speculative execution

Why it works:

- Branch results (true/false) are slow to compute
- CPU guesses based on history (branch predictor)

So even if:

```c
index >= array_size
```

the CPU may still temporarily run the “safe-looking” path.

**2. Attack Injection (training the CPU to behave “wrongly” on purpose):**

Now the attacker doesn’t poison branch targets (that’s v2).

Instead they:

- train the branch predictor + memory state

Attacker repeatedly runs:

```c
for (i = 0; i < safe_size; i++)
    access(array[i]);
```

CPU learns:

- “bounds check is usually true → safe to proceed”

Then attacker feeds a “bad index”:

```c
index = attacker_controlled_value; // huge number
if (index < array_size)
    value = array[index];
```

Even though condition is false:

- CPU speculatively still executes array access
- because it was “recently trained” to trust the branch

**3. The data leakage (turning prediction into a side channel):**

Even though the CPU later realizes:

- “Oops, that was out-of-bounds, discard results”

it already did something irreversible at micro-level:

- it touched cache based on secret data

**Typical Exploit Gadget:**

```c
secret = memory[array[index]];
probe_array[secret * 4096] += 1;
```

What happens:

- Out-of-bounds read leaks secret byte
- Secret is used as an index into probe_array
- That access loads a specific cache line

So now cache contains a pattern like:

| probe_array index | cached? |
| ----------------- | ------- |
| secret * 4096     | YES     |

**Attacker measures it:**

```c
for (i = 0; i < 256; i++) {
    time access(probe_array[i * 4096]);
}
```

- Fast access = cached = secret value
- Slow access = not used

Boom:

- secret reconstructed one byte at a time

---

- **affected:**
  - Almost all modern high-performance CPU (Intel, AMD, ARM) that predict branches in code.
- **Patch:**
  - **Retpolines (Return Trampolines)**: This is a clever Linux/GCC technique that prevents the CPU from being "trained" to jump to malicious locations. It basically confuses the CPU’s branch predictor so it doesn't know where to guess, forcing it to wait for the actual, verified address.

### Branch Target Injection - CVE-2017-5715 (Spectre V2)

Manipulates and poisons the CPU's branch predictor to force software to speculatively execute code fragments known as "gadgets".

---

**How the attack works:**

**1. Speculation (the CPU makes a guess):**

Modern CPUs don’t wait for branches like:

```c
if (x < array_size)
    y = array[x];
```

Instead, they use a Branch Target Buffer (BTB) to predict:

- “This branch usually goes here”
- “Jump to this function address next”

So the CPU:

- guesses the target
- executes instructions speculatively
- before it even knows if the guess was correct

If it guessed wrong → results are discarded but microarchitectural traces remain (cache).

**2. Attack Injection (poisoning the predictor):**

Attacker repeatedly executes a harmless branch:

```c
if (index < safe_limit)
    call victim_function();
```

CPU learns:

`“Oh, this branch usually goes to victim_function”`

Now attacker messes with prediction state by:

- using same virtual addresses / aliasing patterns
- or cross-process / cross-VM branch prediction collisions
- or indirect branch gadgets

So when real victim code runs, CPU predicts:

`“Jump to attacker-controlled gadget instead”`

Even though actual control flow says NO.

**3. Data Leakage (the real damage):**

Now the victim runs:

```c
indirect_call(secret_index);
```

CPU:

- mispredicts jump target
- executes attacker-chosen gadget speculatively

That gadget often does something like:

```c
value = secret_data[secret_index];
cache_probe_array[value * 4096] = 1;
```

Important:

- secret is NEVER architecturally returned
- but it touches cache lines based on secret value

Then attacker measures timing:

```c
for i in probe_array:
    measure access time

```

Fast access → cached → means:

- `“that index was used → secret value leaked”`

---

- **affected:**
  - Almost all modern high-performance CPU (Intel, AMD, ARM) that predict branches in code.
- **Patch:**
  - **Retpolines (Return Trampolines)**: This is a clever Linux/GCC technique that prevents the CPU from being "trained" to jump to malicious locations. It basically confuses the CPU’s branch predictor so it doesn't know where to guess, forcing it to wait for the actual, verified address.
  - **IBRS/IBPB (Indirect Branch Restricted Speculation/Prediction Barrier):** hardware-level CPU security mechanisms designed to mitigate Branch Target Injection (also known as Spectre v2) attacks. They prevent malicious code from hijacking a processor's speculative execution.
- **poc/exploitation concept:**
  - [Anton-Cao/spectrev2-poc](https://github.com/Anton-Cao/spectrev2-poc)

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
- **poc/exploitation:**
  - [coolcatlee/Speculative-Code-Store-Bypass-POC](https://github.com/coolcatlee/Speculative-Code-Store-Bypass-POC)

---

## Software

### Log4shell (CVE-2021-44228)

Log4Shell is a vulnerability reported in November 2021 in Log4j, a popular Java logging framework, involving arbitrary code execution and exploited as a zero-day vulnerability.

