# Notes about RDMA Core Userspace Libraries and Daemons

RDMA Core library has a lot stuff.
It provides the day-to-day-use IB commands such as `ibv_devinfo`.
It also provides the userspace IB verbs library.
Similar to the kernel Infiniband architecture,
rdma-core has a generic layer interface with applications
and a lower-level vendor-specific callback layer.

We look at those commands, libibverbs, and providers.

I've hacked this rdma-core, in-kernel Infiniband layer,
and `MLNX_OFED` before. `MLNX_OFED` is dirty,
it modifies the rdma-core and in-kernel mlx driver
and replace the open-source ones!
I happen to come across this part again
and thought I should keep some notes. Happy hacking!

## Commands

## libibverbs and more

The first thing is to understand how these userspace libraries
communicate directly with the device!
The kernel documentation explained it well:
control path use some `/dev/infiniband/uverbs` ioctl methods,
fast path operations are typically performed by writing
directly to hardware registers mmap()ed into userspace,
with no system call or context switch into the kernel.
(https://www.kernel.org/doc/html/latest/infiniband/user_verbs.html)

QUESTION: since PCIe MMIO space is directly exposed to userspace, is VFIO used here?

## Providers (Vendors)

### VMware Paravirtualized RDMA

I was surprised to discover VMware drivers in this repo.
I saw `providers/vmw_pvrdma`, the userspace driver to VMware paravirtualized RDMA device.

I further discover the Linux `drivers/infiniband/hw/vmx_pvrdma` driver.
Obviously. These two drivers should be doing the same thing, the
userspace is not built upon the kernel one.

So what exactly is paravirtualized RDMA?
As for as I could tell, it's no different from other virtio device/drivers.
The model should be as follows:
1) VMware ESXi hypervisor presents a virtualized RDMA PCIe NIC
to the guest OS. The guest OS, e.g., Linux, will discover
this PCIe device, just like how it discovers all the other
virtio devices (e.g., virtnet).
2) Then the kernel `vmx_pvrdma` detects the PCIe device,
register it as the low-level `ib_device_ops` providers.

Mellanox has a good reference: https://docs.mellanox.com/pages/releaseview.action?pageId=15055422
