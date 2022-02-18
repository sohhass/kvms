******************************************************************************
KVM compatible ARM64 hypervisor
******************************************************************************

KVM hypervisor variant that can do TDX/SEV like security for existing armv8
systems. The hypervisor is implemented such that it can function in almost
any armv8 board out there with virtualization support, regardless of the fact
if the system shipped with existing EL2 elements or not.

Features added over regular KVM configurations are:
- Nearly full guest and host memory space separation
- Complete, linear guest memory integrity protection. Guest pages remain
  intact and unmovable but they can still be swapped in/out when needed.
- Guest memory AES encrypted swapping
- Kernel memory protection toolchain:
  - Page table locks (including elements inside the P?Ds)
  - Memory region permission adjustments
- Memory region / permission validation tools for all CPU modes.


Building and running on QEMU:
-----------------------------
- Run 'make tools'. This will build all the tools required with right versions
  and install them into the 'buildtools' directory. This will also build the
  patched kernel under oss/linux.
- Download or assemble QEMU compatible arm64 linux image you wish to boot up
- Set environment variable BOOTIMG to point to the image. Set PLATFORM=virt
  as well to tell the build system which target you are going for.
- Run 'make DEBUG=1' to build the hypervisor against the kernel under oss/linux
- 'make run' will run the host emulation
- 'make gdb' will run a target debugger session. You can hit breakpoints
  anywhere in the kernel and in the hypervisor. This implies that that 'run'
  target was invoked with 'make DEBUGGER=1 run' such that the QEMU was waiting
  for the debugger connection.
- 'make GRAPHICS=1 ... run' will enable a spice display for the host. The
  invocation will echo the correct connection endpoint to connect to.
- Install more kvm virtual machines inside your host system emulation to see
  some more work the hypervisor is doing.
- Work with the kernel under oss/linux, hyp


Host <-> Guest separation
-------------------------
- Once guest touches a page, it is removed from the host. This is true for
  all guest pages.
- ARmv8 guests are extended to do 'set memory {encrypted, decrypted}' calls
- Guest is always responsible for opening the shared communication channels


Secure swapping
-----------------
- The support is experimental. High level logic is as follows:
  - When the linux mm evicts a clean page, we measure it (sha256) to make sure
    it can't change while the vm doesn't own it. Besides the actual page data,
    we also measure the page permissions so that the page cannot change from RO
    to RW once being reloaded.
  - When the mm evicts a dirty page, we encrypt AND measure it on its way to the
    swap. We don't use the authenticated encryption as the measurement code has
    to be in place anyway to handle the clean / RO pages.

Now, be warned, there are some rough corners. When a page has migrated away
from the host, the host mm looses visibility to the page state and all the
software and even the hardware managed dirty state it is able to perform go out
of sync. Thus, the mm might not know what to sync to the media when the page
eventually finds its way back to the host. Moreover, we also make the pages
dirty behind the scenes every time we encrypt a writable guest page, but we do
try to mark our changes in the qemu dirty log when the logging is active. It is
yet to be sorted what the mm does when it scans an area where the pages may seem
clean, but yet they might not be. Similar virtualization solutions (Intel TDX
and AMD SEV) probably don't suffer from this potential pithole as the mm remains
up to date about the page state, encrypted or not.


VCPU protection
---------------
KVM stores the VCPU context (i.e. registers) in the architecture specific part
of the kvm_vcpu struct. The context is

- Used by the KVM itself in HW, MMIO and instruction emulation, specifically
  mrs, msr, hvc and smc instructions
- Accessible by userspace via KVM_SET_ONE_REG and KVM_GET_ONE_REG ioctls,
  primarily to initialize a guest

The project attempts to limit the exposure of the context outside the guest
once the guest initialization phase is done. To do this, the context is moved
to hypervisor. KVM MMIO and instruction emulation still work on the existing
VCPU context. This is selectively synchronized with the hypervisor VCPU
context. On guest exit, the hypervisor updates the KVM context, and on guest
entry, the hypervisor updates the hypervisor context as follows:

|              | Copy hyp -> KVM    | Copy KVM -> hyp      |
|--------------|--------------------|----------------------|
| hvc          | hvc args (x0...x3) | hvc return code (x0) |
| MMIO read    | -                  | load target reg      |
| MMIO write   | store source reg   | -                    |
| sysreg read  | -                  | mrs target reg       |
| sysreg write | msr source reg     | -                    |

TBC: Floating point registers, QEMU state sync breakage


Guest hardening
---------------
- Follow the TDX prior art at:
  https://github.com/intel/tdx/blob/guest/arch/x86/kernel/tdx-filter.c
- Some sample patches under 'patches' directory


Migration support TODO
-----------------------
- Dirty bitmask handling (partly done)
- Key export / import
- QEMU register get/set support
- ??


SHORT TERM TODO
----------------
1) Memory pressure testing
2) Make guest system registers immutable
3) Hardened / versatile guest config and patches. We need guest configs and
   patches for various use cases. XHCI USB emulation support currently is one
   big hack (works but is insecure).
4) Migration support
5) Finish android hardware support (32bit environment goes SIGILL)
6) Add locking mechanism for guest specific EL2 mappings that are not allowed
   to be changed after VM has been initialized. At the moment the protection
   is for all EL2 mappings which prevents from creating new VMs after the lock
   has been set
7) QEMU protected guest support to give the madvise() hint to KSM not to scan
   the guest memory: -cpu=host,protected-guest-support=kvms
8) Move SHA-256 operations to hardware via the arm crypto-extension (mbedtls)
9) IPI based debug stops, backtraces, tlb flushes when needed
10) Libhybris based GPU access for QEMU
