# Notes about RDMA Core Userspace Libraries and Daemons

RDMA Core library has a lot stuff.
It provides the day-to-day-use IB commands such as `ibv_devinfo`.
It also provides the userspace IB verbs library.
Similar to the kernel Infiniband architecture,
rdma-core has a generic layer interface with applications
and a lower-level vendor-specific callback layer.

This is a very high quality repo, it's one of the best I've seen.
We will look at how those commands are implemented,
libibverbs internal, and vendor specific part (e.g., mlx5).

Happy hacking!

## Commands

We use these commands a lot during our day-to-day RDMA usage.
We can learn how they use the Linux Infiniband devices!
They are implemented in `infiniband-diags` folder.

- ibv_devinfo
- iblinkinfo
- ibping
- ibaddr
- more..

## libibverbs

### Examples

Some useful IB verbs examples.
These are the starting point for many people!

- asyncwatch.c
- device_list.c
- devinfo.c
- pingpong.c
- rc_pingpong.c
- srq_pingpong.c
- uc_pingpong.c
- ud_pingpong.c
- xsrq_pingpong.c

### Internal

The first thing is to understand how userspace libraries
communicate with device directly.

The kernel doc explained it well:
control path use some `/dev/infiniband/uverbs` ioctl methods,
fast path operations are typically performed by writing
directly to hardware registers mmap()ed into userspace,
with no system call or context switch into the kernel.
(https://www.kernel.org/doc/html/latest/infiniband/user_verbs.html)

QUESTION: since PCIe MMIO space is directly exposed to userspace, is VFIO used here?

Public Verbs API:

- They are spread across many files.. because of compatible issues
- `libibverbs/verbs.h`, `libibverbs/compat-1_0.c`

I'm reading one of the control path operations, the `ibv_create_qp`.
I followed its callpath, and confirmed that it is using ioctl at last.
It's fine to use ioctl, these are infrequent calls.
And I think this would apply to all the control path IB verbs.
```c
create_qp
 mlx5_create_qp
  ibv_cmd_create_qp_ex
   execute_cmd_write
     ioctl(fd)
```

Now, onto the data path.
I think the kernel exposed a limited range of the PCIe data MMIO region to the userspace.
And I think the kernel does not expose the PCIe configruation part, right? Need to confirm.

I'm wondering: a) When this shared mapping is created? b) how much PCIe MMIO is exposed?
c) will userspace be able to crash the device?


### Providers (Vendors)

#### Mellanox
TODO

#### VMware Para-virtualized RDMA

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
