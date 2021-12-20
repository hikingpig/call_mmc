# EXPORT_SYMBOL and EXPORT_SYMBOL_GPL 
Some special kernel symbol tables are used by the kernel to store the symbols that
can be accessed by modules together with their corresponding addresses. They are
contained in three sections of the kernel code segment: the _ _kstrtab section
includes the names of the symbols, the _ _ksymtab section includes the addresses of
the symbols that can be used by all kind of modules, and the _ _ksymtab_gpl section includes the addresses of the symbols that can be used by the modules
released under a GPL-compatible license. The EXPORT_SYMBOL macro and the
EXPORT_SYMBOL_GPL macro, when used inside the statically linked kernel code, force
the C compiler to add a specified symbol to the _ _ksymtab and _ _ksymtab_gpl sections, respectively

Only the kernel symbols actually used by some existing module are included in the
table. Should a system programmer need, within some module, to access a kernel
symbol that is not already exported, he can simply add the corresponding EXPORT_
SYMBOL_GPL macro into the Linux source code. Of course, he cannot legally export a
new symbol for a module whose license is not GPL-compatible.

Linked modules can also export their own symbols so that other modules can access
them. The module symbol tables are contained in the _ _ksymtab, _ _ksymtab_gpl, and
_ _kstrtab sections of the module code segment. To export a subset of symbols from
the module, the programmer can use the EXPORT_SYMBOL and EXPORT_SYMBOL_GPL macros described above. The exported symbols of the module are copied into two memory arrays when the module is linked, and their addresses are stored in the syms and
gpl_syms fields of the module object.

-----------------
(Another book...)

The Linux kernel header files provide a convenient way to manage the visibility of
your symbols, thus reducing namespace pollution (filling the namespace with names
that may conflict with those defined elsewhere in the kernel) and promoting proper
information hiding. If your module needs to export symbols for other modules to
use, the following macros should be used.
  - EXPORT_SYMBOL(name);
  - EXPORT_SYMBOL_GPL(name);
Either of the above macros makes the given symbol available outside the module.
The _GPL version makes the symbol available to GPL-licensed modules only. Symbols must be exported in the global part of the module’s file, outside of any function, because the macros expand to the declaration of a special-purpose variable that
is expected to be accessible globally. This variable is stored in a special part of the module executible (an “ELF section”) that is used by the kernel at load time to find the variables exported by the module.


# PCI
PCI core and bus access routines live in drivers/pci/. To obtain a list of helper routines offered by the PCI subsystem, search for EXPORT_SYMBOL inside this directory. For definitions and prototypes related to the PCI layer, look at include/linux/pci*.h.

You can spot several PCI device drivers in subdirectories under drivers/net/,
drivers/scsi/, and drivers/video/. To locate all PCI drivers, recursively grep the drivers/ tree for pci_register_driver().

If you do not find a good starting point to develop a custom PCI network driver,
begin with the skeletal PCI network driver drivers/net/pci-skeleton.c. For a brief tutorial on PCI programming, look at Documentation/pci.txt. For a description of the PCI DMA API, read Documentation/DMA-mapping.txt.

# SATA
Assume that your system is SATA-enabled via an Intel ICH7 South Bridge chipset. You need the following libATA components to access your disk:
1. The libATA core—To enable this, set CONFIG_ATA during kernel configuration. For a
list of library functions offered by the core, grep for EXPORT_SYMBOL_GPL inside the
drivers/ata/ directory.

# ONLINE SOURCE
https://embetronicx.com/tutorials/linux/device-drivers/export_symbol-in-linux-device-driver/

