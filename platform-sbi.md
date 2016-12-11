# Stefan O'Rear's strawman platform SBI spec for RISC-V

## Scope and definitions

The **platform supervisor binary interface (SBI)** is the inteface between
RISC-V System Software and the underlying Platform.  A **Standard Platform** is
a platform which implements all mandatory requirements in this specification.
**Standard System Software** is system software which only assumes details of
the platform which correspond to the mandatory requirements and zero or more
optional requirements for Standard Platforms.  Standard System Software MAY
assume that the platform conforms to optional requirements, but MUST document
which optional requirements it requires. _Rationales and implementation guidance
are given in italics._

Standard Platforms MAY provide features and behavior which is not described in
this specification as long as no requirements are violated.  Standard System
Software MUST NOT assume such nonstandard features, but MAY use them if
discovered. **Feature discovery** is central to the RISC-V Standard Platform; it
MUST be done either through the config string (described later), or non-platform
means which have previously been discovered (a recursive process which bottoms
out in the config string).  _For instance, it is permissible to discover a PCI
root port using the config string, then use PCI enumeration to discover devices
with additional MMIO resources._

_The specification is agnostic about the nature and existence of privileged
modes above S-mode.  Platform resources can be implemented in software or
hardware; this may or may not correspond to whether they are exposed as ECALL
or MMIO.  Other privileged modes such as an omniscient debug mode can exist with
no effect whatsoever on the platform interface._

_The purpose of this specification is to maximize the probability of successful
interoperation of independently created plaforms and system software.  It places
a particular emphasis on virtualization guest use cases._

A Standard Platform provides a **platform memory map** and a **platform software
interface**.  The platform memory map contains one or more memory resources and
zero or more memory mapped I/O (MMIO) resources.  The platform software
interface describes zero or more services which can be accessed using the ECALL
instruction.  The memory map and software interface are collectively known as
**platform resources**.  _There are no mandatory MMIOs and no mandatory software
interfaces.  A Standard Platform may offer resources exclusively as MMIO and not
implement the ECALL instruction at all; it may offer resources exclusively
through ECALL and provide no MMIOs; or it may provide a combination._

A Standard Platform MUST implement platform resources in a way which respects
the RISC-V Memory Model for all memory resources and provides a total ordering
of operations on other resources.  _This constitutes the primary restriction on
the size and composition of Standard Platforms; you cannot declare a network to
be a Standard Platform unless there are additional mechanisms in place to
present a globally consistent memory map._

Several per-hart features are optional but **MUST be present for full S-mode
implementations**; a hart claiming the S extension MUST provide features so
labelled.  _Dubious, discuss: How big do we want the tent to be?_

A Standard Platform provides one or more **ordinary harts**.   Ordinary harts
implement the RISC-V User-level ISA and have access to the platform resources.
All ordinary harts MUST have the same view of the platform resources.  Ordinary
harts MAY implement one or more standard or nonstandard extension and MAY have
access to additional resources which are not part of the global resource map.
Ordinary harts MAY exclude support for unaligned memory access but this MUST be
present for full S-mode implementations.  _Dubious, discuss: the smallest
implementations (GRVI, picorv32) have no CSR file.  I don't think we want to
ever consider those Standard Platforms but I could be persuaded._

A Standard Platform MAY provide **non-ordinary harts**, which do not have access
to the full platform resource map and may not implement the RISC-V User-level
ISA.  For the purposes of this document non-ordinary harts are considered
devices and will not be discussed further.

System software runs in a system or supervisor operating mode (**S-mode**) and
can access the global memory map through memory instructions, the global
software interface (if non-empty) through ECALL, and local resources through
resource-specific means.  _Several additional features are required to claim
support for the S extension; however, even if they are missing the most
privileged mode in Standard Platform is still considered to be an S-mode._ A
standard platform MAY also provide **less-privileged modes** (U-mode, SN-mode,
and UN-mode), described later.

_Dubious, discuss: This proposal allows former M-only implementations (provided
they support CSRs and a config-string) to be rebranded as S-only (with no
virtual resources) after CSR renumbering.  MU-systems without N can likewise be
made into SU-only, although most Standard System Software will declare a
dependency on paging, reducing the utility.  MU-styles with N can be presented
as S-only hypervised platforms, or as S-only platforms with the interpretive
execution facility._

_Dubious, discuss: Relative to priv-1.9.1 this proposal places more weight on
the config string and strongly discourages probing.  This is friendlier to
nonstandard extensions and matches modern industry practice, but may be
controvertial and is open to discussion._

## The config string

**TODO**

* Semantic superset of device-tree to allow shared effort on describing
physical hardware; platform features might or might not be in the device-tree
subset

* Syntax is less important and FDT/DTS are rather meh, so we'll probably use
something close to the priv-1.9.1 config string **but rigidly defined**

* Specific bindings are described throughout this document and in appendix

* Prefer fine-grained description over runtime probing.  Dubious, discuss: WARL
considered harmful?

* Bindings for features known at manufacture (or update) time, vendor/arch/impl
for "features" discovered later; config-string only, no utility in CSRs

* Most config string keys, all ECALLs, versioned and stabilized independently
of the spec in which they are described.

## Instruction-level functionality

### Reference facility for initial program load

**TODO**

* Platform-definded format -or- Simple binary format, not ELF

* ROM-friendly

* Config string address in a0

* Multihart TBD

* No paging, the usual language about quiescent states

### Facility for horizontal and vertical traps

**TODO**

* sstatus, sie, sip, sepc, scause, sbadaddr, stvec

* Dubious, discuss: Adopt luto's STE proposal?

* Interrupts may be hardwired or through PLIC

### User-mode execution facility

**TODO**

* Optional but MUST be present in full S-mode implementations

* Adds U-mode, U-mode cannot touch s* CSRs, sstatus.SPP

* Adds sstatus.VM (TM?) but does not mandate any non-bare values

* Forbids new U-mode exposed state which is not covered by sstatus.XS

* N-extension cannot exist in a Standard Platform because of the above;
workarounds exist

* Clarifies XS and FS language to indicate that off means disabled but valid; a
disabled unit cannot be touched by U-mode for any reason

* Dubious, discuss: RTOSes will want a way to disable the vector unit without
erasing it, but I don't have express Foundation buyin for this.

* Dubious, discuss: Initial/Clean/Dirty might need to be revised in this light,
for instance Off+Initial might be a state we want to explicitly support

### Address translation and protection facility

**TODO**

* Not useful without either U-mode or interpretive execution

* Adds sstatus.VM, sstatus.PUM, VM operation is defined by the VM mode; if PUM=0
then S-mode R, W is a superset of below-S-mode RX, W

* Important change: VM can be changed by S-mode

* Dubious, discuss: Alternatively we could break out TUS, TUL, SXR so that
S-mode loads and stores become _exactly_ U-mode accesses; probably not worth it

* Address translation takes effect immediately, trampoline pages are standard
practice here and minimize microarchitecture exposure

* Sbare mandatory

* Optional Sbb/Sbbid with new registers

* Paging modes with platform-implemented PTW MUST be present in full S-mode
implementations (optional in minimal Standard Platforms)

* Single-root page table for simplicity / familiarity / minimization of S-mode
virtualization state (Dubious, discuss: luto wants two roots)

* PTE format as in priv-1.9.1.  Dubious, discuss: Do we need 31 bits in a
not-present PTE?  We could orthogonalize it to RWXN, and an invalid page would
have all four zero.  (from luto)

* SFENCE.VM instruction is provided, semantics TBD.  (Dubious, discuss: luto
thinks the current specification is not suitable for SMP applications.  I
haven't fully understood the complaints, and will need to do so.)

* Dubious, discuss: do we need SFENCE.VM to see an A or D bit set?  Current
trends point to "no", but we should make sure this isn't a problem for classic
virtualization.

* Maybe spec (but do NOT stabilize) a software TLB refill mode for experimental
use only

### Interpretive execution facility

_This is closely modeled on the facility of the same name in S/370-XA, for which
the earliest public documentation is ca. 1984._

**TODO**

* New operating mode S-nested (SN); if the U-mode facility also exists, a
U-nested mode will exist

* S-nested and U-nested see as physical addresses what U-mode sees (or would
see, if U-mode is not implemented) as virtual addresses

* Most/all traps cause exit to S-mode

* S-nested is entered by a virtual resource ECALL; instructions and CSRs may be
defined much later

* Can be (ab)used for fast paravirtual syscalls when running under a classical
hypervisor, as running U-mode in UN-mode replaces multiple s* CSR accesses with
a single ECALL

* Delegation, quite a bit else TBD

### Binding instruction-set extensions to the config string

**TODO**

* 26 bit flags will not be enough even for standardized features, we need to
plan ahead for extensibility

* We need a way to indicate "this is not implemented, but all of the opcodes
trap and you can emulate it" to support RV64G U-mode on RV64IMASU hardware

* Need to think more about the new HPM facility and how it manifests here.  In
particular, U-mode 2.1 says that TIME, CYCLE, INSTRET are mandatory but
priv-1.9.1 allow them to trap, which is the same as not existing from the
U-mode vantage point.

## Platform control resources

### General considerations ECALL vs. MMIO

**TODO**

* ECALL: good for virtualization, requires privileged SW

* MMIO: terrible for virtualization, can avoid privileged SW, likely faster
when non-virtualized and slower when virtualized

* General rule: identify the product lifecycle of the Platform and the System
Software and push as much complexity as possible to the side that can fix bugs
faster

* MMIO emulation code is difficult, dangerous, and actively counterproductive to
the RISC-V extensibility story; we shoould encourage hypervisors to run
MMIO-free except for pass-through devices.

* SBILIB, by analogy with the Linux VDSO, will likely play an important role
in the future but is too risky to freeze at this time

### Platform level interrupt controller

**TODO**

* Optional; wiring interrupts to specific sip bits is still valid

* priv-1.9.1 PLIC is very close to what we want, but provides a limited number
of virtual interrupt sources

* We may need to provide a paravirtual interrupt controller, for SSIP if
nothing else.

* Do we need to distinguish paravirtual interrupts from IPIs at
the scause level?

### Channel I/O

**TODO**

* OASIS virtio but with an ECALL doorbell???

### Non-channel devices

**TODO**

* Absolute RTC, power management, sync random numbers (maybe)

* Minimal paravirtual power management needed for VM shutdown/reset

* There is a suggestion shutdown could be handled with WFI; unsure myself

* Non-initial hart bringup and CPU hotplug support???

* Absolute RTC seems redundant with TIME CSR.  Should latter be mandatory?

### IOMMU

**TODO**

* Inherently bus-specific but we may want to offer guidance, e.g. PTE format

### Channel hotplug bus

**TODO**

* OASIS virtio leans on PCIe for IOMMU and hotplug functionality; providing
non-MMIO access to the same functionality would be useful.

## Recommendations for MMIO devices

**TODO** Systems that prioritize interoperability will provide and support the
ECALL interfaces, but we might still be able to improve compatibility in
practice by setting Schelling points for questions like "what is the minimum
register layout of a UART".

## Appendix: Platform-defined config string bindings

## Appendix: Platform-defined ECALLs

## Appendix: Platform-defined CSRs
