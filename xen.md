#### The role of Domain 0

The purpose of a hypervisor is to allow guests to be run. Xen runs guests in
environments known as domains, which encapsulate a complete running virtual
environment. When Xen boots, one of the first things it does is load a Domain 0
(dom0 ) guest kernel. This is typically specified in the boot loader as a module,
and so can be loaded without any filesystem drivers being available. Domain 0 is
the first guest to run, and has elevated privileges. In contrast, other domains are referred to as domain U (domU )—the “U” stands for unprivileged. However, it
is now possible to delegate some of dom0’s responsibilities to domU guests, which
blurs this line slightly.

Domain 0 is very important to a Xen system. Xen does not include any device
drivers by itself, nor a user interface. These are all provided by the operating
system and userspace tools running in the dom0 guest. The Domain 0 guest is
typically Linux, although NetBSD and Solaris can also be used and other operating
systems such as FreeBSD are likely to add support in the future. Linux is used by
most of the Xen developers, and both are distributed under the same conditions—
the GNU General Public License.

The most obvious task performed by the dom0 guest is to handle devices.
This guest runs at a higher level of privilege than others, and so can access the
hardware. For this reason, it is vital that the privileged guest be properly secured.

Part of the responsibility for handling devices is the multiplexing of them for
virtual machines. Because most hardware doesn’t natively support being accessed
by multiple operating systems (yet), it is necessary for some part of the system
to provide each guest with its own virtual device.
Figure 1.5 shows what happens to a packet when it is sent by an application
running in a domU guest. First, it travels through the TCP/IP stack as it would
normally. The bottom of the stack, however, is not a normal network interface
driver. It is a simple piece of code that puts the packet into some shared memory.

The memory segment has been previously shared using Xen grant tables and
advertised via the XenStore.

The other half of the split device driver, running on the dom0 guest, reads
the packet from the buffer, and inserts it into the firewalling components of the
operating system—typically something like iptables or pf, which routes it as it
would a packet coming from a real interface. Once the packet has passed through
any relevant firewalling rules, it makes its way down to the real device driver.
This is able to write to certain areas of memory reserved for I/O, and may require access to IRQs via Xen. The physical network device then sends the packet.

Note that the split network device here is the same irrespective of the real
networking card. Xen provides a simplified interface to these devices, which is
easy to implement for people porting systems to Xen. There are three components
to any driver:

• The split driver
• The multiplexer
• The real driver

The split driver is typically as simple as it can be. It is designed to move data
from the domU guests to the dom0 guest, usually using ring buffers in shared
memory.

The real driver should already exist in the dom0 operating system, and so it
cannot really be considered part of Xen. The multiplexer may or may not. In the
example of networking, the firewalling component of the network stack already
provides this functionality. In others, there may be no existing operating system
component that can be pressed into use.

![Xen packet](1.png)

#### Xen Configurations

In this example, all of the hardware is being controlled by the host in Domain
0. This is not always ideal. If a driver for a particular device contains bugs,
it can crash the dom0 kernel, which in turn can bring down all guests on the
system. Therefore, it is often beneficial to isolate a driver in its own domain,
which does nothing other than export the split device driver.

A physical guest typically spends some of its boot time firing off queries to
the BIOS to determine what hardware is available, including CPU capabilities. In
a Xen guest, however, the BIOS is unavailable. Because the BIOS allows direct
access to the hardware, permitting guests to interact with it directly would break the principle of isolation.

The BIOS is replaced by a variety of different facilities in Xen. The first is the start info page, which contains basic information required by a guest to initialize the kernel. Next is the shared info page, which gives some more data and is updated while the guest is run. Finally, there is the XenStore. Among other things, this is used to determine which (virtual) devices are available.

### Replacing Privileged Instructions with Hypercalls

Because the kernel is now running in ring 1 where it is not allowed to do every-
thing it wants, there has to be some mechanism for bypassing this restriction in
a controlled manner.

This is not a new problem. Userspace code, running in ring 3, encounters it
all the time. Displaying things on the screen, reading input from the keyboard,
and sending data over the network all involve interaction with hardware, which
can’t be performed by unprivileged code.

The solution is to use a system call, a formal mechanism for telling the kernel to do something for you. System calls all work in roughly the same way, irrespective of your operating system 1 or platform:
1. Marshal the arguments in registers or on the stack.
2. Issue a well-known interrupt, or invoke a special system call instruction.
3. Jump to the kernel’s (privileged) interrupt handler as a result of the interrupt.
4. Process the system call at the kernel’s privilege.
5. Drop to a lower privilege level and return.

On x86 it is common to use interrupt 80h for system calls, although newer
CPUs have SYSENTER/SYSEXIT or SYSCALL/SYSRET instructions for fast
system calls, depending on the manufacturer.

The same mechanism was previously used by Xen. Hypercalls were generated
by a guest kernel in almost the same way as system calls are generated by userspace applications, the difference being that interrupt 82h, instead of 80h, is used.

This still works as of Xen 3, but is now deprecated. Instead, hypercalls are
issued indirectly via the hypercall page. This is a memory page mapped in to the
guest’s address space when the system is started.

Hypercalls are issued by CALLing an address within this page.

#### Split Device Driver Model

When the bottom half supports the necessary features, there needs to be some
mechanism for exporting it to other guests. This is typically done using ring
buffers in shared memory segments. The front-end driver initializes a memory
page with a ring data structure and exports it via the grant table mechanism.
It then advertises the grant reference via the XenStore where the back end can
retrieve it. The back end then maps it into its own address space, giving a shared communication channel into which the front end inserts requests and the back end places responses. Typically, an event channel is also configured for the device and used by both ends to signal when data is waiting in the ring.

The ring buffer is located in a shared memory segment. For it to be usable, the
Domain 0 guest stores the associated grant table reference in the XenStore. Other guests are then able to enumerate the available devices by reading the XenStore.

### Retrieving Boot Time Info

A kernel starting up in Xen begins with a page mapped into its address space
containing information about the system, known as the start info page. The way
the address of this page is transferred to the guest is architecture-dependent; on x86, the address is stored in the ESI register.

After you have set the pointer correctly, you can interact with the start info
page in exactly the same way you would with any other structure. The fields in
it provide all of the information required to bootstrap a kernel.

When you are happy that you are running in a supported Xen environment,
booting can proceed. A kernel typically needs to know early on how much RAM it has available to it, and how many CPUs. The number of pages is provided by
the start info structure in the nr pages element.

Several of the other fields in the structure refer to other pages in memory.
These are given in machine addresses; the guest must issue hypercalls to map them in to its own address space. Because they are machine addresses, they are subject to change if the virtual machine is suspended and resumed later, or migrated. For this reason, they should be remapped every time the virtual machine is resumed.

The shared info field is the first to specify a machine page. In this case, it is the one containing the shared info structure discussed in the next section. Mapping this is typically one of the first things a guest should do, because it contains a significant amount of information that can be useful near to system start.

The next two fields relate to the XenStore, described in Chapter 8. These form
a pair of a form that is used in many places in Xen. The first, store mfn, gives
the machine address of the shared memory page used for communication with the
XenStore. The second, store evtchn , gives an event channel used for notifications.

### The Shared Info Page
The shared info page is used throughout the runtime of a guest kernel to retrieve information about the global state. Unlike the start info page, the shared info page contains information that is dynamically updated as the system runs.

This contains information related to the running virtual CPU s, available event channels, wall clock time, and some architecture-specific information.


#### Device I/O Rings
One of the main uses for shared memory pages is to implement I/O rings for
communicating between parts of paravirtualized device drivers. These provide a
simple message-passing abstraction built on top of the shared memory mechanism
provided by Xen.

The I/O rings provide a method for asynchronous communication between
domains. One domain places a request in the ring, whereas the other removes it
and inserts a response. Because requests and responses are produced at (roughly)
the same rate, a single ring can accommodate them both.

I/O rings that are used to transfer a lot of data might be polled on both ends.
Those that are used more infrequently use the event mechanism to signal that
data is available.

