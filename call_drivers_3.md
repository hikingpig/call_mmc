# Initialization
  - must initialize:
    - the name and bus fields
  - should initialize 
    - the devclass field (when it arrives)
    - as many of the callbacks as possible

# Declaration
- if the driver can be completely converted to new model

```cpp
static struct device_driver eepro100_driver = {
         .name		= "eepro100",
         .bus		= &pci_bus_type,

         .probe		= eepro100_probe,
         .remove		= eepro100_remove,
         .suspend		= eepro100_suspend,
         .resume		= eepro100_resume,
  };
```
- if the driver can not be converted completely to new model: A driver typically defines an array of device IDs that it supports. The format of these structures and the semantics for comparing device IDs are completely bus-specific.

```cpp
  struct pci_driver {
         const struct pci_device_id *id_table;
         struct device_driver	  driver;
  };
```
- more specific
```cpp
  static struct pci_driver eepro100_driver = {
         .id_table       = eepro100_pci_tbl,
         .driver	       = {
		.name		= "eepro100",
		.bus		= &pci_bus_type,
		.probe		= eepro100_probe,
		.remove		= eepro100_remove,
		.suspend	= eepro100_suspend,
		.resume		= eepro100_resume,
         },
  };
```
# Registration
```cpp
int driver_register(struct device_driver *drv);
```
The driver registers the structure on startup.

For drivers that have no bus-specific fields (i.e. don't have a bus-specific driver structure), they would use driver_register and pass a pointer to their struct device_driver object.

Most drivers, however, will have a bus-specific structure and will need to register with the bus using something like `pci_driver_register`.

# Access

Once the object has been registered, it may access the common fields of the object, like the lock and the list of devices:
```cpp
  int driver_for_each_dev(struct device_driver *drv, void *data,
			  int (*callback)(struct device *dev, void *data));
```

The devices field is a list of all the devices that have been bound to the driver.

# Callbacks

1. probe
```cpp
	int	(*probe)	(struct device *dev);
```
  - called in task context: the bus's rwsem locked and the driver partially bound to the device
  - commonly use `container_of()` to convert "dev" to a bus-specific type
  - check if
    - the device is present
    - a version the driver can handle
    - driver data structures can be allocated and initialized
    - any hardware can be initialized.

- drivers often store a pointer to their state with dev_set_drvdata()
  - When the driver has successfully bound itself to that device, then probe() returns zero and the driver model code will finish its part of binding the driver to that device
  - return a negative errno value to indicate that the driver did not bind to this device, in which case it should have released all resources it allocated

2. sync_state
```cpp
	void	(*sync_state)	(struct device *dev);
```
  - called only once for a device when all the consumer devices of the device have successfully probed

3. remove
```cpp
  int 	(*remove)	(struct device *dev);
```
  - called to unbind a driver from a device when
    - a device is physically removed from the system
    - driver module is being unloaded, during a reboot sequence and other cases
4. suspend
```cpp
 	int	(*suspend)	(struct device *dev, pm_message_t state);
```
  - put the device in a low power state.

- resume
```cpp
	int	(*resume)	(struct device *dev);
```
  - bring a device back from a low power state.

  
  
