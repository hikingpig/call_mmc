- what is device?

# Device IDs
The DDI interfaces enable drivers to provide a persistent, unique identifier for a device. The device ID can be used to identify or locate a device. The ID is independent of the device's name or number (dev_t). Applications can use the functions defined in libdevid(3LIB) to read and manipulate the device IDs registered by the drivers.

# Interrupt Handling
The DDI/DKI addresses the following aspects of device interrupt handling:
■ Registering device interrupts with the system
■ Removing device interrupts
■ Dispatching interrupts to interrupt handlers
Device interrupt sources are contained in a property called interrupt, which is either provided by the PROM of a self-identifying device, in a hardware configuration file, or by the booting system on the x86 platform.

# Callback Functions
Certain DDI mechanisms provide a callback mechanism. DDI functions provide a mechanism for scheduling a callback when a condition is met. Callback functions can be used for the following typical conditions:
■ A transfer has completed
■ A resource has become available
■ A time-out period has expired

Callback functions are somewhat similar to entry points, for example, interrupt handlers. DDI functions that allow callbacks expect the callback function to perform certain tasks. In the case of DMA routines, a callback function must return a value indicating whether the callback function needs to be rescheduled in case of a failure.

Callback functions execute as a separate interrupt thread. Callbacks must handle all the usual multithreading issues.

# Driver Context
The driver context refers to the condition under which a driver is currently operating. The context limits the operations that a driver can perform. The driver context depends on the executing code that is invoked. Driver code executes in four contexts:

■ User context. A driver entry point has user context when invoked by a user thread in a synchronous fashion. That is, the user thread waits for the system to return from the entry point that was invoked. For example, the read(9E) entry point of the driver has user context when invoked by a read(2) system call. In this case, the driver has access to the user area for copying data into and out of the user thread.

■ Kernel context. A driver function has kernel context when invoked by some part of the kernel. In a block device driver, the strategy(9E) entry point can be called by the pageout daemon to write pages to the device. Because the page daemon has no relation to the current user thread, strategy(9E) has kernel context in this case.

■ Interrupt context.Interrupt context is a more restrictive form of kernel context. Interrupt context is invoked as a result of the servicing of an interrupt. Driver interrupt routines operate in interrupt context with an associated interrupt level. Callback routines also operate in an interrupt context. See Chapter 8, “Interrupt Handlers,” for more information.

■ High-level interrupt context. High-level interrupt context is a more restricted form of interrupt context. If ddi_intr_hilevel(9F) indicates that an interrupt is high level, the driver interrupt handler runs in high-level interrupt context. See Chapter 8, “Interrupt Handlers,” for more information.

# Dynamic Memory Allocation
Device drivers must be prepared to simultaneously handle all attached devices that the drivers claim to drive. The number of devices that the driver handles should not be limited. All per-device information must be dynamically allocated.
```cpp
void *kmem_alloc(size_t size, int flag);
```
The standard kernel memory allocation routine is kmem_alloc(9F). 
  - kmem_alloc() is similar to the C library routine malloc(3C), with the addition of the flag argument
  - The flag argument can be either KM_SLEEP or KM_NOSLEEP, indicating whether the caller is willing to block if the requested size is not available
    - If KM_NOSLEEP is set and memory is not available, kmem_alloc(9F) returns NULL.

kmem_zalloc(9F) is similar to kmem_alloc(9F), but also clears the contents of the allocated memory.

`Note` – Kernel memory is a limited resource, not pageable, and competes with user applications and the rest of the kernel for physical memory. Drivers that allocate a large amount of kernel memory can cause system performance to degrade.

```cpp
void kmem_free(void *cp, size_t size);
```
Memory allocated by kmem_alloc(9F) or by kmem_zalloc(9F) is returned to the system with kmem_free(9F). kmem_free() is similar to the C library routine free(3C), with the addition of the size argument. Drivers must keep track of the size of each allocated object in order to call kmem_free(9F) later

# Hotplugging
This manual does not highlight hotplugging information. If you follow the rules and suggestions for writing device drivers given in this book, your driver should be able to handle hotplugging. 

In particular, make sure that both autoconfiguration (see Chapter 6, “Driver Autoconfiguration”) and detach(9E) work correctly in your driver. 

In addition, if you are designing a driver that uses power management, you should follow the information given in Chapter 12, “Power Management.” SCSI HBA drivers might need to add a cb_ops structure to their dev_ops structure (see Chapter 18, “SCSI Host Bus Adapter Drivers”) to take advantage of hotplugging capabilities.

# What Is the Kernel?
A program that manages system resources
  - insulates applications from the system hardware
  - provides them with essential system services such as
    - input/output (I/O) management, virtual memory, and scheduling

  - consists of object modules that are dynamically loaded into memory when needed
  
divided logically into two parts
  - kernel, manages file systems, scheduling, and virtual memory
  - I/O subsystem, manages the physical components

- provides a set of interfaces for applications to use that are accessible through system calls
  - Some system calls are used to invoke device drivers to perform I/O

`Device drivers` are loadable kernel modules that `manage data transfers while insulating the rest of the kernel from the device hardware`.
  
To be compatible with the operating system, device drivers need to be able
to accommodate such features as multithreading, virtual memory addressing, and both 32-bit and 64-bit operation.

The kernel provides access to device drivers through the following features:

■ Device-to-driver mapping. The kernel maintains the device tree. Each node in the tree represents a virtual or a physical device. The kernel binds each node to a driver by matching the device node name with the set of drivers installed in the system. The device is made accessible to applications only if there is a driver binding.

■ DDI/DKI interfaces. DDI/DKI (Device Driver Interface/Driver-Kernel Interface)
interfaces standardize interactions between the driver and the kernel, the device hardware, and the boot/configuration software. These interfaces keep the driver independent from the kernel and improve the driver's portability across successive releases of the operating system on a particular machine.

■ `LDI`. The LDI (Layered Driver Interface) is an extension of the DDI/DKI. The LDI `enables a kernel module to access other devices in the system.` The LDI also enables you to `determine which devices are currently being used by the kernel`.

# Devices as Special Files
Devices are represented in the file system by special files

Special files can be of type block or character.
  - The type indicates which kind of device driver operates the device.
  - Drivers can be implemented to operate on both types.
    - disk drivers export a character interface for use by the fsck(1) and mkfs(1) utilities, and a block interface for use by the file system.

Associated with each special file is a device number (dev_t).
  - A device number consists of a major number and a minor number.
    - The major number identifies the device driver associated with the special file. 
    - The minor number is created and used by the device driver to further identify the special file.
      - Usually, the minor number is an encoding that is used to identify which device instance the driver should access and which type of access should be performed
        - For example, the minor number can identify a tape device used for backup and can specify that the tape needs to be rewound when the backup operation is complete.

# DDI/DKI Interfaces
- documents
  - driver entry points
  - driver-callable functions
  - kernel data structures used by device drivers
  
- DDI/DKI is intended to standardize and document all interfaces between device drivers and the rest of the kernel. 

- In addition, the DDI/DKI enables source and binary compatibility for
drivers on any machine that runs the OS, regardless of the processor architecture

- Drivers that use only kernel facilities that are part of the DDI/DKI are known as DDI/DKI-compliant device drivers.

- The DDI/DKI enables you to write platform-independent device drivers
  - These binary-compatible drivers enable you to more easily integrate third-party hardware and software into any machine
  - The DDI/DKI is architecture independent, which enables the same driver to work across a diverse set of machine architectures.

Platform independence is accomplished by the design of DDI in the following areas:
■ Dynamic loading and unloading of modules
■ Power management
■ Interrupt handling
■ Accessing the device space from the kernel or a user process, that is, register mapping and memory mapping
■ Accessing kernel or user process space from the device using DMA services
■ Managing device properties


# DeviceTree Components
The system builds a tree structure that contains information about the devices connected to the machine at boot time. The device tree can also be modified by dynamic reconfiguration operations while the system is in normal operation. The tree begins at the root device node, which represents the platform.

  - Below the root node are the branches of the device tree. A branch consists of one or more bus nexus devices and a terminating leaf device.
  - A bus nexus device provides bus mapping and translation services to subordinate devices in the device tree. 
    - PCI - PCI bridges, PCMCIA adapters, and SCSI HBAs are all examples of nexus devices.
    - The discussion of writing drivers for nexus devices is limited to the development of SCSI HBA drivers (“SCSI Host Bus Adapter Drivers”).

  - Leaf devices are typically peripheral devices such as disks, tapes, network adapters, frame buffers, and so forth. 
    - Leaf device drivers export the traditional character driver interfaces and block driver interfaces.
    - The interfaces enable user processes to read data from and write data to either storage or communication devices.

The system goes through the following steps to build the tree:

1. The CPU is initialized and searches for firmware.

2. The main firmware (OpenBoot, Basic Input/Output System (BIOS), or Bootconf) initializes and creates the device tree with known or self-identifying hardware.

3. When the main firmware finds compatible firmware on a device, the main firmware
initializes the device and retrieves the device's properties.

4. The firmware locates and boots the operating system.

5. The kernel starts at the root node of the tree, searches for a matching device driver, and binds that driver to the device.

6. If the device is a nexus, the kernel looks for child devices that have not been detected by the firmware. The kernel adds any child devices to the tree below the nexus node. 

7. The kernel repeats the process from Step 5 until no further device nodes need to be created.

Each driver exports a device operations structure `dev_ops` to define the operations that the device driver can perform. 
The device operations structure contains function pointers for generic operations such as attach(9E), detach(9E), and getinfo(9E). 
The structure also contains a pointer to a set of operations specific to bus nexus drivers and `a pointer to a set of operations specific to leaf drivers`.

The tree structure creates a parent-child relationship between nodes. This parent-child relationship is the key to architectural independence. When a leaf or bus nexus driver requires a service that is architecturally dependent in nature, that driver requests its parent to provide the service. This approach enables drivers to function regardless of the architecture of the machine or the processor.

# Displaying the DeviceTree
The device tree can be displayed in three ways:
■ The `libdevinfo` library provides interfaces to access the contents of the device tree programmatically.
■ The `prtconf` command displays the complete contents of the device tree.
■ The /devices hierarchy is a representation of the device tree. Use the `ls` command to view the hierarchy.

# Binding a Driver to a Device
In addition to constructing the device tree, the kernel determines the drivers that are used to manage the devices.

Binding a driver to a device refers to the process by which the system selects a driver to manage a particular device. The binding name is the name that links a driver to a unique device node in the device information tree. For each device in the device tree, the system attempts to choose a driver from a list of installed drivers.

Each device node has an associated name property. This property can be assigned either from an external agent, such as the PROM, during system boot or from a driver.conf configuration file. In any case, the name property represents the node name assigned to a device in the device tree. The node name is the name visible in /devices and listed in the prtconf output.

A device node can have an associated compatible property as well. The compatible property contains an ordered list of one or more possible driver names or driver aliases for the device.

The system uses both the compatible and the name properties to select a driver for the device. The system first attempts to match the contents of the compatible property, if the compatible property exists, to a driver on the system. Beginning with the first driver name on the compatible property list, the system attempts to match the driver name to a known driver on the system. Each entry on the list is processed until the system either finds a match or reaches the end of the list.

If the contents of either the name property or the compatible property match a driver on the system, then that driver is bound to the device node. If no match is found, no driver is bound to the device node.

# Generic Device Names
Some devices specify a `generic device name` as the value for the name property. 

Generic device names `describe the function of a device without actually identifying a specific driver` for the device. 
  - a SCSI host bus adapter might have a generic device name of scsi
  - an Ethernet device might have a generic device name of ethernet

The compatible property enables the system to determine alternate driver names for devices with a generic device name, for example, glm for scsi HBA device drivers or hme for ethernet device drivers.

Devices with generic device names are required to supply a `compatible property`.

Note – For a complete description of generic device names, see the IEEE 1275 Open Firmware Boot Standard. 

A device node with the generic device name display. The driver binding name SUNW,ffb is the first name on the compatible property driver list that matches a
driver on the system driver list. In this case, display is a generic device name for frame buffers.

