KVMGT
=====

This test was done on an ArchLinux installation with Kernel 4.13.3-1

# Preparing the host

## Kernel cmdline

Make sure ```i915.enable_gvt=1``` is specified on your kernel cmdline. (Check the output of ```cat /proc/cmdline```)

When using grub this can be archived by modifying ```/etc/default/grub``` to contain

```
GRUB_CMDLINE_LINUX="i915.enable_gvt=1"
```

and updating ```/boot/grub/grub.cfg``` by executing ```grub-mkconfig -o /boot/grub/grub.cfg```

Reboot.

Now there should be a directory ```/sys/class/mdev_bus```. Inside it you should find another directory by the
name of the PCI ID of your Intel graphics card.

## Virtual GPU

Before using the KVMGT GPU it must be created. Inside ```/sys/class/mdev_bus/$PCI_ID/mdev_supported_types``` you should find at least one type of supported virtual GPU configuration.

Create a new uuid by calling ```uuidgen```. Now pick one of the virtual GPU configs and echo the UUID to ```/sys/class/mdev_bus/$PCI_ID/mdev_supported_types/$VGPU_TYPE/create```. After that there should be an entry by the name of the UUID in ```/sys/class/mdev_bus/$PCI_ID/mdev_supported_types/$VGPU_TYPE/devices```.

To use this device in libvirt it must be accessible by the QEMU user/group. Therefore the right VFIO node in ```/dev/vfio``` must have the correct permissions set. To find the id of the VFIO node execute ```readlink /sys/class/mdev_bus/$PCI_ID/mdev_supported_types/$VGPU_TYPE/devices/$UUID/iommu_group```. This should yield something much alike ```../../../../kernel/iommu_groups/7```. In this case the VFIO node would be ```/dev/vfio/7```. To grant qemu access to VFIO node either do ```chmod 666 /dev/vfio/$VFIO_ID``` or ```chown :kvm /dev/vfio/$VFIO_ID; chmod 660 /dev/vfio/$VFIO_ID```.

## Libvirt

By default libvirt won't allow accessing sysfs devices directly. This is enforced by cgroup restrictions. To loosen the cgroup restriction edit ```/etc/libvirt/qemu.conf``` and make sure it contains ```cgroup_controllers = [ "cpu", "memory", "blkio", "cpuset", "cpuacct" ]``` (note the absence of "devices"). This disables sysfs device access whitelisting.

Now create a new libvirt VM using the example configuration from ```domain.xml```. The most important parts of the configuration are
```
  <memoryBacking>
    <locked/>
  </memoryBacking>
```

and

```
  <qemu:commandline>
    <qemu:arg value='-device'/>
    <qemu:arg value='vfio-pci,sysfsdev=/sys/class/mdev_bus/0000:00:02.0/fc444019-54df-47bc-a0a0-7308ce9681a8,rombar=0'/>
    <qemu:arg value='-nographic'/>
  </qemu:commandline>
```
.
Note that ```<qemu:commandline>``` will fail to validate if the attribute ```xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'```is not set on the XML root ```domain``` node.

The ```memoryBacking``` part specifies that pages used by the virtual machine may not be swapped out by the host.

```qemu:commandline``` will need Adjustments to work with your setup. The uuid ```fc444019-54df-47bc-a0a0-7308ce9681a8``` must be replaced with the UUID used for creating the virtual GPU.