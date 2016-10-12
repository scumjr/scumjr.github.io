---
layout: post
author: scumjr
title: From SMM to userland in a few bytes
---

In 2014, [@coreykal](https://twitter.com/@coreykal),
[@xenokovah](https://twitter.com/@xenokovah),
[@jwbutterworth3](https://twitter.com/@jwbutterworth3)
[@ssc0rnwell](https://twitter.com/@ssc0rnwell)  gave a talk entitled
[Extreme Privilege Escalation on Windows 8/UEFI Systems](https://www.mitre.org/sites/default/files/publications/14-2221-extreme-escalation-presentation.pdf)
at Black Hat USA. They
[introduced](https://www.youtube.com/watch?v=UJp_rMwdyyI&feature=youtu.be&t=2926)
the idea of a SMM rootkit called *The Watcher* slides (57 to 63). To sum it up:

> - The Watcher lives in SMM (where you can't look for him)
> - It has no build-in capability except to scan memory for a magic signature
> - If it finds the signature, it treats the data immediately after the
>   signature as code to be executed
> - In this way the Watcher performs arbitrary code execution on behalf of some
>   controller.

This idea is awesome, and I wanted to try to implement it on Linux. Actually, it
was far more easier than expected, thanks to QEMU and SeaBIOS.

<!-- more -->



## SMM rootkit

The BIOS flash is initially modified in order to have a malicious code executed
in SMM by the SMI handler, which scans memory for a magic signature on a regular
basis.


### Periodic SMI generation

In order to scan memory regularly, the SMI handler must be called
regularly. There's a mechanism that's exactly meant for this: the `Periodic SMI#
Rate Select`, through `GEN_PMCON_1` register
([slide 18](http://opensecuritytraining.info/IntroBIOS_files/Day1_07_Advanced%20x86%20-%20BIOS%20and%20SMM%20Internals%20-%20SMM.pdf)).
It guarantees the generation of a SMI at least at least once every 8, 16, 32, or
64 seconds. On a modern CPU, it's straightforward to use this register to
ensures that the SMM handler is called regularly. Nevertheless, it doesn't seem
to exist on the emulated CPU (Intel® 440FX) of the virtual machine.

<!-- CPU? -->

But this isn't the only manner to call the SMI handler regularly. The APIC can
be configured to redirect IRQ to SMM. In fact, the I/O APIC defines a
redirection table for this purpose. This technique was already used by
[chpie](http://ivanlef0u.fr/repo/todo/chpie_smm_keysniff_ENG.pdf) and
[SMM-Rootkits-Securecom08.pdf](http://www.eecs.ucf.edu/~czou/research/SMM-Rootkits-Securecom08.pdf)
to implement a keylogger in 2008. Actually, it can also be used to call a SMI
handler regularly.

For instance, IRQs on `ata_piix` occurs quite frequently (about every 2 seconds)
in my VM:

<!-- what's ata_piix? -->

~~~~~ bash
$ grep ata_piix /proc/interrupts
14:       4357   IO-APIC-edge      ata_piix
~~~~~

ATA PIIX seems to be the
[Intel PATA/SATA controller](http://lxr.free-electrons.com/source/drivers/ata/ata_piix.c).
This looks like a great opportunity to let our SMI handler be called
regularly by redirecting IRQ #14 to SMM.

The
[Intel® 82093AA I/O APIC Datasheet](http://download.intel.com/design/chipsets/datashts/29056601.pdf)
describes how to use the I/O redirection table registers. The registers are 64
bits wide. Basically, they're just set to the interrupt vector.

<p align="center">
  <img src="/public/images/bits0-7.png" alt="INTVEC" />
</p>

For example, `IOREDTBL 14 = 0x000000000000003e`. But one can change the
*Delivery Mode* to another value, for example `0b010` to deliver a SMI:

<p align="center">
  <img src="/public/images/bits8-10.png" alt="DELMOD" />
</p>

Once the SMI handler is called, the interrupt must be sent to the Local APIC
through an IPI. Simply write to the
[Local APIC registers](http://wiki.osdev.org/APIC#Local_APIC_registers):

~~~~~ C
#define LOCAL_APIC_BASE   (void *)0xfee00000
#define INTERRUPT_VECTOR  0x3e
static void forward_interrupt(void)
{
    uint32_t volatile *a, *b;
    unsigned char *p;

    p = LOCAL_APIC_BASE;
    b = (void *)(p + 0x310);
    a = (void *)(p + 0x300);

    *b = 0x00000000;
    *a = INTERRUPT_VECTOR << 0;
}
~~~~~

The IRQ redirection must be created once the OS finished to configure APIC,
otherwise it may be overwritten.

If you're too lazy to read MSRs to get IOAPIC and Local APIC memory addresses,
they're located here:

~~~~~ bash
$ grep APIC /proc/iomem
fec00000-fec003ff : IOAPIC 0
fee00000-fee00fff : Local APIC
~~~~~


### Memory scan from the SMM handler

Since the SMM handler is called regularly, it must be as fast as possible. The
memory scan is easy: just walk through all the physical pages. Even if not
bullet-proof, the techniques from
[OS Dev.org](http://wiki.osdev.org/Detecting_Memory_(x86)) are sufficient for a
proof of concept. If one page starts with the magic signature, the payload
located just after is executed.


## Payloads


### SMM payload

The SMM payload must be as simple as possible since SMM handlers execute in with
paging disabled, no interruptions, etc.

<!-- XXX: paging? -->

Fortunately, the VDSO library is mmaped in every userland processes. A few
syscalls (on [x64](http://man7.org/linux/man-pages/man7/vdso.7.html):
`clock_gettime`, `getcpu`, `gettimeofday`, `time`) use VDSO to be
faster. One can dump the VDSO of a random process to take a look at it:

~~~~~ bash
$ gdb /bin/ls
gdb$ b __libc_start_main
gdb$ r
Breakpoint 1, __libc_start_main
gdb$ dump_binfile vdso 0x7ffff7ffa000 0x7ffff7ffc000
gdb$ q
~~~~~

and notice that `clock_gettime` is right at the entrypoint address:

~~~~~ bash
$ readelf -a vdso | grep Entry
  Entry point address:               0xffffffffff700700

$ gdb vdso
gdb$ x/5i 0xffffffffff700700
   0xffffffffff700700 <clock_gettime>:         push   rbp
   0xffffffffff700701 <clock_gettime+1>:       mov    rbp,rsp
   0xffffffffff700704 <clock_gettime+4>:       push   r15
   0xffffffffff700706 <clock_gettime+6>:       push   r14
   0xffffffffff700708 <clock_gettime+8>:       push   r13
gdb$
~~~~~

Moreover, there is about 2800 bytes of free space to put some code at the end of
the mapping:

~~~~~ bash
$ hd vdso | tail -3
00001510  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00002000
~~~~~

All of this make VDSO looks like a promising target. VDSO is easy to fingerprint
in memory because the first bytes are the ELF header, and a few strings are
always present. Since `clock_gettime` is the entrypoint of the library,
there's no need to implement an ELF parser in ring-2. Finally, VDSO library is
mapped on 2 consecutive physical pages. One don't have mess with the page
tables of the current process to find the second page.

Here's the plan: once the magic is found by the SMI handler at the beginning of
a physical page, the code located after it is executed. The code executed in SMM
walk (again) through the physical pages to find the 2 VDSO's consecutive
physical pages. Finally, the prologue of `clock_gettime` is hijacked to a custom
userland payload written at the end of VDSO.

This SMM payload written in C and compiled with
[metasm](https://github.com/jjyg/metasm/) is about 750 bytes long.

Once again, this is sufficient for a proof of concept, but a fully weaponized
exploit should parse the VDSO ELF carefully. One could note that once this SMM
payload is executed, all present and future userland processes will be
backdoored and the IRQ redirection can be safely removed.


### Userland payload

The userland payload will be called every time a process makes a call to one of
the previously mentioned system call. Since `clock_gettime()` might be called
frequently by different processes, we don't want to overload the machine with a
lot of Python processes. Thus, the payload checks if the file `/tmp/.x` exists,
and in that case does nothing because it has already been executed. On the other
hand if `/tmp/.x` doesn't exist, the payload forks:

- the parent restore registers before returning to `clock_gettime()`,
- the child executes a python one-liner which creates `/tmp/.x` and does a TCP
  reverse shell to the attacker.

This userland payload, written in assembly, is 280 bytes long.



## Trigger

For a local attacker, it's trivial to execute code in SMM: just mmap a page and
copy `MAGIC+CODE`. That's it.

For a remote attacker, there are several ways to reach one's goal. For example,
send a lot of UDP packets containing `MAGIC+CODE` prefixed with random padding
(between `0` and `PAGE_SIZE-sizeof(CODE)` bytes) to any UDP port. With a bit
of luck, it triggers the SMM handler in a few seconds.



## Waiting for the shell

Once the SMM payload is executed, the attacker only have to wait for the reverse
shell. It can take a few seconds to a few minutes given the running processes on
the machine.

An impatient attacker may want to check if everything went well by triggering
one of the VDSO syscalls. For instance, one can try to login to a ssh server
with an invalid account. OpenSSH server calls `clock_gettime()` before writing
the log entry, which triggers the execution of the userland payload by any
process.

A non-root process may however call `clock_gettime()`, and the reverse shell
would not have root privileges. The uid of the process may be checked in the
payload, but to save a few bytes, the attacker may also remove `/tmp/.x` and
wait for a new reverse shell.

<!--
~~~~~ bash
$ ssh -o PubkeyAuthentication=no -o PasswordAuthentication=no invalid@192.168.122.247
~~~~~
-->

Eventually, here's a screenshot of the backdoor. On the left, the debug log
`/tmp/bios.log` of the virtual machine whose SMM is backdoored. On the right a
Python script wich send in a loop the payload to be executed by the SMI handler
in the hope of being mmaped at the very beginning of a page. And at the bottom,
the reverse shell from a root process of the VM whose VDSO has been hijacked:

<p align="center">
  <img src="/public/images/screenshot.png" alt="SMM backdoor screenshot" />
</p>



## Conclusion

The idea of *The Watcher* is practical on Linux: it's pretty straightforward to
execute code in userland from SMM reliably, thanks to VDSO. The code of SeaBIOS
is modified to include a malicious SMI handler, and the memory of the OS is
never altered until the attacker manages to put the payload in memory. The
payload size is no longer than 1084 bytes, and can be injected through the
network even if not port is open.

Nevertheless, SeaBIOS' SMM support is basic, and I didn't find a way to
automatically install *The Watcher* at the boot of the machine (there should be
a more elegant way than a bootkit). At present, a SMI must be issued
(`outb(0xXY, 0xb2)`) to start *The Watcher*.

The code of this proof-of-concept is available on github:
[the-sea-watcher](https://github.com/scumjr/the-sea-watcher).

~~On a more encouraging note,
[spender](https://twitter.com/grsecurity/status/578558748496568322) mentioned
a simple trick to potentially determine if a machine is infected by a SMM
rootkit: just count the number of SMIs since the last reset of the machine, with
`MSR_SMI_COUNT` (since Nehalem):~~

~~~~~ bash
$ sudo aptitude install linux-tools-common linux-tools-generic
$ wget http://kernel.ubuntu.com/git/cking/debug-code/.git/tree/smistat/smistat.c http://kernel.ubuntu.com/git/cking/debug-code/.git/tree/smistat/Makefile
$ make
$ sudo modprobe msr
$ sudo ./smistat
  Time          SMIs
  02:21:41       268
  02:21:42       268
  02:21:43       268
~~~~~

**Update**: as
[pointed out by @XenoKovah](https://twitter.com/XenoKovah/status/687469257634914304),
the `MSR_SMI_COUNT` isn't a meaningful detector.
