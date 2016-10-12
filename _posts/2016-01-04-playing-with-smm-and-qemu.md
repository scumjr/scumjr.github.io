---
layout: post
title: Playing with SMM and QEMU
---

10 years ago, playing with
[SMM](https://en.wikipedia.org/wiki/System_Management_Mode) seemed to be quite
risky. Audacious people were flashing their BIOS, running the risk of bricking
their machine. Since a few years, the raise of UEFI and Secure Boot is an
incentive for virtualisation solutions to implement SMM. For instance, KVM
developers work hard on the [subject](https://lwn.net/Articles/642596/).

<!-- more -->

The excellent training material
[Advanced x86: Introduction to BIOS & SMM](http://opensecuritytraining.info/IntroBIOS.html)
covers a lot topics in details, including SMM. SMM is a recurrent topic which
has been talked over and over again since a long time. The idea of playing with
the SMM of an hypervisor isn't new either. For instance, .aLS and Alfredo
modified the BIOS of VMWare in
[Persistent BIOS Infection](http://phrack.org/issues/66/7.html) (2009), and
chpie implemented a keylogger for VMWare in
[Implementing SMM PS/2 Keyboard sniffer](http://ivanlef0u.fr/repo/todo/chpie_smm_keysniff_ENG.pdf)
(also 2009).

Even if there's nothing new in this blogpost, I hope these quick notes may help
people willing to have  testing platform to play with SMM on a legacy BIOS.



# Requirements

- Linux 4.4. No need no compile a custom kernel with Ubuntu's kernel PPA:

{% highlight bash %}
$ wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.4-rc3-wily/linux-headers-4.4.0-040400rc3_4.4.0-040400rc3.201511300321_all.deb
$ wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.4-rc3-wily/linux-headers-4.4.0-040400rc3-generic_4.4.0-040400rc3.201511300321_amd64.deb
$ wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.4-rc3-wily/linux-image-4.4.0-040400rc3-generic_4.4.0-040400rc3.201511300321_amd64.deb
$ sudo dpkg -i linux-*-4.4*.deb
{% endhighlight %}

- QEMU 2.5:

{% highlight bash %}
$ wget http://wiki.qemu-project.org/download/qemu-2.5.0.tar.bz2
$ tar xjvf qemu-2.5.0.tar.bz2
$ cd qemu-2.5.0/
$ ./configure --target-list=x86_64-softmmu --enable-debug --enable-spice --disable-vnc --enable-sdl
$ make
$ sudo make install
{% endhighlight %}



# VM configuration

``libvirt`` is used to manage the VMs. After the installation of Ubuntu 14.10 in
a new VM, the following modifications (with ``virsh edit ubuntu-utopic``) must
be applied to load a custom BIOS (``/home/user/bios.bin``) and output
[SeaBIOS debug messages](http://www.seabios.org/Debugging#Timing_debug_messages)
in a file:

~~~~~ xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
    ...
    <qemu:commandline>
     <qemu:arg value='-bios'/>
      <qemu:arg value='/home/user/bios.bin'/>
      <qemu:arg value='-chardev'/>
      <qemu:arg value='file,id=seabios,path=/tmp/bios.log'/>
      <qemu:arg value='-device'/>
      <qemu:arg value='isa-debugcon,iobase=0x402,chardev=seabios'/>
    </qemu:commandline>
</domain>
~~~~~

On Ubuntu, files under ``/home/user/`` and ``/tmp/`` can't be accessed if the
AppArmor profile of libvirtd is enabled. The following command disables QEMU's
AppArmor profile:

~~~~~ bash
$ sudo ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
$ sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
~~~~~



# SeaBIOS

Since [SeaBIOS](http://www.seabios.org/SeaBIOS) is the default BIOS for QEMU and
KVM, it's a good target to play with SMM. As said in
[Advanced x86 - BIOS and SMM Internals - SMM](http://opensecuritytraining.info/IntroBIOS_files/Day1_07_Advanced%20x86%20-%20BIOS%20and%20SMM%20Internals%20-%20SMM.pdf),
the code that executes in SMM (called the SMI handler) is instantiated from the
BIOS flash. Our goal is to modify the default SMI handler by modifying the
BIOS, in order to get our code running in ring-2. The SeaBOS SMI handler prints
a debug message each time it is called:

~~~~~ C
$ grep -A 7 'handle_smi(' src/fw/smm.c
void VISIBLE32FLAT
handle_smi(u16 cs)
{
    if (!CONFIG_USE_SMM)
            return;
    u8 cmd = inb(PORT_SMI_CMD);
    struct smm_layout *smm = MAKE_FLATPTR(cs, 0);
    u32 rev = smm->cpu.i32.smm_rev & SMM_REV_MASK;
    dprintf(DEBUG_HDL_smi, "handle_smi cmd=%x smbase=%p\n", cmd, smm);
~~~~~

Activate debug log, increase debug log level and decrease SMI log level in order
to get some feedback:

~~~~~ bash
$ git clone git://git.seabios.org/seabios.git seabios
$ cd seabios/
$ sed -i 's/CONFIG_DEBUG_LEVEL=1/CONFIG_DEBUG_LEVEL=3/' .config
$ sed -i 's/DEBUG_HDL_smi 9/DEBUG_HDL_smi 1/' src/config.h
$ make
~~~~~

Once the BIOS is compiled and copied into ``/home/user/bios.bin``, the VM will
boot on this one.



# Tests

How can we ensure that SMM is working? One can simply write a byte to port
`0xb2` with the following program:

~~~~~ C
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/io.h>

#define PORT_SMI_CMD      0x00b2

int main(void)
{
    if (ioperm(PORT_SMI_CMD, 1, 1) != 0)
	        err(EXIT_FAILURE, "ioperm");

    outb(0x61, PORT_SMI_CMD);
    printf("done\n");

    return EXIT_SUCCESS;
}
~~~~~

and the SMI handler should print a debug message in `/tmp/bios.log`:

~~~~~ bash
$ virsh start --console ubuntu-utopic
user@ubuntu-utopic:~$ sudo ./trigger_smi

$ tail -1 /tmp/bios.log
handle_smi cmd=61 smbase=0x000a0000
~~~~~

If you have a doubt about the generation of SMIs by KVM, check that
[SMM](http://xiaogr.com/wp-content/uploads/2015/08/kvm-smm.pdf) is supported
and `KVM_SMI` correctly issued. Here are the ioctl's constants:

~~~~~ C
#define KVM_CAP_X86_SMM     0x17
#define KVM_SMI             _IO(KVMIO,   0xb7)
~~~~~

`KVM_CHECK_EXTENSION` ioctl is called at the start of the VM, and `KVM_SMI` each
time a SMI is triggered:

~~~~~ bash
$ sudo strace -e ioctl -ff -p $(pidof qemu-system-x86_64) \
  |& grep 'KVM_CHECK_EXTENSION\|KVM_SMI\|b7'
[pid 24432] ioctl(10, KVM_CHECK_EXTENSION, 0x75) = 1
[pid 24432] ioctl(10, 0xaeb7, 0x56353ed70500) = 0
~~~~~



# Conclusion

That's it, just modify the `handle_smi()` function of SeaBIOS to give SMM a try.
