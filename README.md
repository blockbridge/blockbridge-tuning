# Performance Tuning for Storage Systems

There are several optimizations that you can make to the underlying operating
environment to obtain the best performance from your Blockbridge storage node
and any iSCSI storage client.  The optimizations are listed below, along with
an estimate of how much impact each one will have on performance.

1. Interrupt Affinity [Impact = High]
2. Tune NIC Interrupt Coalescing [Impact = High]
3. Disable Device I/O Scheduling [Impact = High]
4. Enable Hugepages [Impact = Medium]
5. Disable I/O Merging [Impact = Medium]
6. Disable Block Device Entropy Gathering [Impact = Low]


## 1. Interrupt Affinity [Impact = High]


Your system's hardware issues interrupt requests (IRQs) to request service from
the Linux kernel.  When the kernel receives an IRQ on a CPU core, it handles
the request at the expense of whatever application processing is happening on
the core.  While timely processing of interrupts is an important part of
reaching your system's performance capabilities, taking time out of a
latency-sensitive task to handle an interrupt can negatively impact
performance.

First, remove the irqbalance service if it is installed on your system:

```
yum remove irqbalance
```

We've obtained the best performance by moving all of the the storage
controller- and network-related interrupt request handling to CPU 0.  The
easiest way to do this is to use the following script to move all of the
interrupts labelled "PCI-MSI" to CPU 0 (run as root):

```
for i in `grep PCI-MSI /proc/interrupts | awk -F : '{print $1}'`; do
    echo 1 > /proc/irq/$i/smp_affinity
done
```

For example, here's what matches `grep PCI-MSI /proc/interrupts` on one
system:

```
 33:     130874     601874     311785     224546  IR-PCI-MSI-edge      mpt3sas0-msix0
 34:     268426     250892     320991     187540  IR-PCI-MSI-edge      mpt3sas0-msix1
 35:     116486     286284      94243     108178  IR-PCI-MSI-edge      mpt3sas0-msix2
 36:      80633     212228     125968      42316  IR-PCI-MSI-edge      mpt3sas0-msix3
 37:        181         55         26         39  IR-PCI-MSI-edge      xhci_hcd
 38:     101020      97931      76978      86727  IR-PCI-MSI-edge      ahci
 39:      14959       5532       4080       2966  IR-PCI-MSI-edge      eth0
```

This list includes the mpt3sas SCSI HBA, the USB 3.0 controller (xhci_hcd), the
SATA controller (ahci), and eth0.  Though we don't strictly need the USB 3.0
controller over on CPU 0, it's fine to put it there.

For another example, the following output is from a VMware virtual machine:


```
 24:          0          0          0          0   PCI-MSI-edge      pciehp
 25:          0          0          0          0   PCI-MSI-edge      pciehp
 ... 
 55:          0          0          0          0   PCI-MSI-edge      pciehp
 56:      80674  100837474     100888     153583   PCI-MSI-edge      eth0-rxtx-0
 57:      58882     147023   27864637      63966   PCI-MSI-edge      eth0-rxtx-1
 58:      40638     253094     118793   34339318   PCI-MSI-edge      eth0-rxtx-2
 59:   34114324      70420      36347      33226   PCI-MSI-edge      eth0-rxtx-3
 60:          0          0          0          0   PCI-MSI-edge      eth0-event-4
 61:          0          0          0          0   PCI-MSI-edge      vmci
 62:          0          0          0          0   PCI-MSI-edge      vmci
```

The above list includes numerous PCI express hotplug controllers (pciehp),
eth0, and the virtual machine communication interface (vmci).

After running the script, to verify that the interrupts have been moved to CPU
0, do some I/O while you watch cat /proc/interrupts to make sure that only the
CPU0 column's counters are incrementing.

To make this change persistent, copy the script into `/etc/rc.local`.


## 2. Tune NIC Interrupt Coalescing [Impact = High]

Many high speed Network Interface Cards are able to coalesce or
throttle their interrupts so that the host doesn't spend as much time
processing them.  For some workloads, this can help achieve higher
throughputs, with a tradeoff for increased latency.  Most storage
workflows are more latency sensitive, so we recommend setting the NIC
to wait no more than 5 microseconds to issue an interrupt request
after receipt of an ethernet frame:

    ethtool -C eth0 rx-usecs 5
  
To set this parameter persistently, either add the ethtool line to
`/etc/rc.local`, or see the section at the end of this document on
persistent ethtool settings for RHEL and CentOS.


## 3. Disable Device I/O Scheduling [Impact = High]

By default, Linux schedules the I/O it sends to a block device, delaying some
requests in order to lump them together or to issue them sequentially.  The
Blockbridge storage node schedules its own I/O, so the effect of this
additional Linux scheduling layer only adds latency in the I/O path.

Disable I/O scheduling for each block device that will be incorporated into
Blockbridge datastores by echoing "noop" to the
`/sys/block/<DEV>/queue/scheduler` file.  For example, to change the scheduler
for `/dev/sdb`:

    # cat /sys/block/sdb/queue/scheduler
    noop deadline [cfq]
    # echo noop > /sys/block/sdb/queue/scheduler
    # cat /sys/block/sdb/queue/scheduler
    [noop] deadline cfq

To make this change persistent, add the `echo noop` lines to `/etc/rc.local`.


## 4. Enable Hugepages [Impact = Medium]

The Blockbridge storage controller allocates the bulk of your system's
RAM to use for metadata and data caching.  It attempts to allocate the
memory in large chunks, called "huge pages" if it can, falling back to
the conventional 4 KiB page size when the supply of hyge pages is
exhausted.  It's much more efficient for the system to track this
memory in 1 GiB or 2 MiB chunks instead of many small 4 KiB pages.  We
recommend that you reserve at least half of your system's RAM in huge
pages (either 1GB or 2MB as available).

Optimally, some 1 GiB pages would be available to the storage controller.
These (really) huge pages are a relatively new feature which may not be
supported by your CPU.  To check if your CPU supports them, look for “pdpe1gb”
in the output of `cat /proc/cpuinfo`:

	# grep -o pdpe1gb /proc/cpuinfo | uniq
    pdpe1gb

If your CPU supports them, you can enable 1GB pages as a kernel boot option
(source:
[Huge pages part 3: Administration](https://lwn.net/Articles/376606/)). In your
bootloader, add the following parameters to the kernel to enable 1GiB hugepages
and to reserve (for example) eight of them (NOTE: once reserved, they cannot be
freed):

	hugepagesz=1GB hugepages=8

2MiB pages may be enabled directly from sysfs. For example, to reserve 512 2MiB
pages:

	# cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
	# 0
	# echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
	# cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
	# 512

If not enough memory was available due to system memory limits or memory
fragmentation, then `cat nr_hugepages` will show less than 512.

To make this setting persistent across reboot, edit `/etc/sysctl.conf` and add
the setting there:

    kernel.mm.hugepages.hugepages-2048kb/nr_hugepages = 512

To view the current reservation and usage of hugepages in the system, use the
legacy proc interface:

	# cat /proc/meminfo | grep Huge
	AnonHugePages:    655360 kB
    HugePages_Total:     512
    HugePages_Free:      512
    HugePages_Rsvd:        0
    HugePages_Surp:        0
    Hugepagesize:       2048 kB

Or with hugeadm:

	# hugeadm --pool-list
      Size  Minimum  Current  Maximum  Default
     2097152      512      512      512        *

Do not make this reservation while the storage node software is
running.  Shut it down first, reserve the pages, and start the storage
node up again. The storage processor will reserve its memory first
from hugepages if they are configured.


## 5. Disable I/O Merging [Impact = Medium]

While you're in `/sys/block/<DEV>/queue`, disable I/O merging.  This prevents
the kernel from combining neighboring reads and writes into larger operations.

    # echo 1 > /sys/block/sdb/queue/nomerges
    # cat /sys/block/sdb/queue/nomerges
    0


## 6. Disable Block Device Entropy Gathering [Impact = Low]

One more parameter to tweak in `/sys/block/<dev>/queue`.  This minor
performance boost prevents I/O to the devices from feeding into the kernel's
entropy pool:

    echo 0 > add_random (per device)


## Appendix 1: Persistent Ethtool Settings for RHEL and CentOS

> This content is reproduced from an answer at Stack Overflow
> [How to persist ethtool settings though reboot](http://serverfault.com/questions/463111/how-to-persist-ethtool-settings-through-reboot).

You can enter the ethtool commands in `/etc/rc.local` (or your distribution's
equivalent) where commands are run after the current runlevel completes, but
this isn't ideal. Network services may have started during the runlevel and
ethtool commands tend to interrupt network traffic. It would be more preferable
to have the commands applied as the interface is brought up.

The network service in CentOS has the ability to do this. The script
/etc/sysconfig/network-scripts/ifup-post checks for the existence of
/sbin/ifup-local, and if it exists, runs it with the interface name as a
parameter (eg: /sbin/ifup-local eth0)

We can create this file with touch /sbin/ifup-local make it executable with
chmod +x /sbin/ifup-local set its SELinux context with chcon --reference
/sbin/ifup /sbin/ifup-local and then open it in an editor.

A simple script to apply the same settings to all interfaces would be something
like:

    #!/bin/bash
    if [ -n "$1" ]; then
        /sbin/ethtool -G $1 rx 4096 tx 4096
        /sbin/ethtool -K $1 tso on gso on
    fi

Keep in mind this will attempt to apply settings to ALL interfaces, even the
loopback.

If we have different interfaces we want to apply different settings to, or want
to skip the loopback, we can make a case statement

    #!/bin/bash
    case "$1" in
    eth0)
        /sbin/ethtool -G $1 rx 16384 tx 16384
        /sbin/ethtool -K $1 gso on gro on
        ;;
    eth1)
        /sbin/ethtool -G $1 rx 64 tx 64
        /sbin/ethtool -K $1 tso on gso on
        /sbin/ip link set $1 txqueuelen 0
        ;;
    esac
    exit 0

Now ethtool settings are applied to interfaces as they start, all potential
interruptions to network communication are done as the interface is brought up,
and your server can continue to boot with full network capabilities.


## Appendix 2: Disable VMware Balloon Driver (VMware Guests Only)

If you're running the storage node as a VMware guest, disable VMware's balloon
driver for the storage node.  The balloon driver is normally a good idea; it
allows the VMware host to reclaim memory hoarded by virtual machines.  However,
it's not compatible with the large memory reservation that will be made by your
storage node.  If left enabled, it will trigger the kernel's out-of-memory
(OOM) killer, resulting in service outages.

Disable the balloon driver with the following procedure:

* Using the vSphere Client, connect to the vCenter Server or the ESXi/ESX host
  where the virtual machine resides.
* Log into the ESXi/ESX host as a user with administrative rights.
* Shut down the virtual machine.
* Right-click the virtual machine listed on the Inventory panel and click Edit
  Settings.
* Click the Options tab, then under Advanced, click General.
* Click Configuration Parameters.
* Click Add row and add the parameter sched.mem.maxmemctl in the text box.
* Click on the row next to it and add 0 in the text box.
* Click OK to save changes.

Source: [Disabling the balloon driver (1002586).](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1002586).
