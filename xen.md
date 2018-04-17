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

#### The Split Driver Model

The XenStore itself is implemented as a split device. The location of the page
used to communicate is given as a machine frame number in the start info page.
This is slightly different to other devices, in that the page is made available to the guest before the system starts, rather than being exported via the grant table mechanism and advertised in the XenStore.


#### Handling Notifications from Events

Xen provides an analog of this in the form of event channels. These are de-
livered asynchronously to guests and, unlike interrupts, are enqueued when the
guest is not running. Interrupts, conventionally, are not aware of the existence of virtual machines and so are delivered immediately.

When a driver needs to notify a device of some waiting data, it typically
writes to a control register. Within Xen, the event mechanism also replaces this
for notifying the back end of a split device driver, because both directions are an example of interdomain communication.

#### Configuring via the XenStore

Among other things, the XenStore is the Xen equivalent of an OpenFirmware
device tree, or the results of querying an external bus. It provides a central
location for retrieving information about devices that are available to the domain. The XenStore is, itself, a device, and so must be bootstrapped using information from the start info page.

The XenStore provides an abstract way of discovering information about de-
vices, and about other aspects of the system. It is one of the first devices that must be supported, because it allows the information required for other devices to be found.

The XenStore is a simple hierarchical namespace containing strings.

#### The Console Device
After putting data in the buffer, we need to signal the back end to remove
it and display it. This is done via the event channel mechanism. Events will be
discussed in detail in the next chapter, including how to handle incoming ones. For now, we will just signal the event channel, and look at what is actually happening in the next chapter.

Signaling the event channel is fairly simple, and can be thought of in the same
way as sending a UNIX signal. The console evtchn variable holds the number of
the event channel being used for the console. This is similar to the signal number in UNIX, but is decided at runtime, rather than compile time. Note that sending an event after each character has been placed into the ring is highly inefficient. It is more efficient to only send an event at the end of sending, or when the buffer is full. This is left as an exercise for the reader.

To issue the signal, we use the HYPERVISOR event channel op hypercall. The
command we give to this tells it to send the event, and the control structure takes a single argument, indicating the event channel to be signaled.

#### Looking through the XenStore

The XenStore is a storage system shared between Xen guests. It is a simple hierarchical storage system, maintained by Domain 0 and accessed via a shared memory page and an event channel. Although the XenStore is fairly central to the operation of a Xen system, there are no hypercalls associated with it. The start info page contains the address of the shared memory page used to communicate with the store. A guest maps this page and then all further communication happens via the ring buffers in this page.

Unlike most filesystems, the XenStore supports a transactional model for I/O.
Groups of requests can be bundled into a transaction, assuring that they will
complete atomically. This allows consistent views of the store (or some subtree)
to be easily created.

##### From the kernel
A guest kernel can interact with the XenStore with a similar degree of control. The XenStore, like most other devices, is interacted with via a shared memory page and an event channel. In implementation, it is similar to the console device. Unlike other devices, which retrieve their configuration information from the XenStore, the store has its page mapped into the guest’s address space and the event channel connected on system boot.

The two pieces of information required to begin using the XenStore are found in
the start info page, in the store_mfn and store_evtchn fields. The first of these gives the machine frame number of the shared memory page containing the XenStore ring buffer. This must be converted to a virtual address before it can be used. The other is the event channel. A handler should be configured for this, as discussed in the last chapter.

The XenStore is very similar, in terms of interface, to the console device.
Both are mapped into the new domain’s address space by the domain builder,
and have their event channels assigned at boot time. Both have two rings, one for requests and the other for responses, and producer/consumer counters for each in the shared ring. Both mainly deal with text.

Setting up the XenStore device, as with the console, is simply a matter of
getting the pseudo-physical address of the shared page and keeping this as a
pointer. After that, an event handler should be set up for retrieving asynchronous responses, both from requests and from watches.

Most of our interactions with the store require writing a message into the
buffer and signaling the back end.

This example first required writing a key, so we’ll implement that first. The
basic process for doing this can be viewed as follows:
1. Prepare the message header.
2. Send the header.
3. Send the (key, value) pair.
4. Signal the event channel.
5. Read the response.
