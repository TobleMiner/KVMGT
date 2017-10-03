KVMGT
=====

This test was done on an ArchLinux installation with Kernel 4.13.3-1

Also I will be using the terms vGPU, KVMGT graphics card and (Intel) virtual GPU interchangeably. The do all refer to Intel GVT-G KVMGT.

In contrast the term emulated GPU or emulated graphics card refer to generic software GPU solutions like QXL or virtio-gpu.

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

The different virtual GPU configurations differ in the amount of video memory available and the maximum screen resolution available.

Create a new uuid by calling ```uuidgen```. Now pick one of the virtual GPU configs and echo the UUID to ```/sys/class/mdev_bus/$PCI_ID/mdev_supported_types/$VGPU_TYPE/create```. After that there should be an entry by the name of the UUID in ```/sys/class/mdev_bus/$PCI_ID/mdev_supported_types/$VGPU_TYPE/devices```.

To use this device in libvirt it must be accessible by the QEMU user/group. Therefore the right VFIO node in ```/dev/vfio``` must have the correct permissions set. To find the id of the VFIO node execute ```readlink /sys/class/mdev_bus/$PCI_ID/mdev_supported_types/$VGPU_TYPE/devices/$UUID/iommu_group```. This should yield something much alike ```../../../../kernel/iommu_groups/7```. In this case the VFIO node would be ```/dev/vfio/7```. To grant qemu access to VFIO node either do ```chmod 666 /dev/vfio/$VFIO_ID``` or ```chown :kvm /dev/vfio/$VFIO_ID; chmod 660 /dev/vfio/$VFIO_ID```.

## Libvirt

I'd recommend installing a new virtual machine with an emulated graphics adapter to create the bootable base image. Using KVMGT for installation will prove very challenging since at the time of writing there is no way to forward output of a KVMGT vGPU to a real display. Thus the only way of installing the OS with no emulated graphics would be to use a serial console which is often not supported out of the box.

Note that I was not able to use both an Intel KVMGT vGPU and an emulated graphics adapter at the same time. As soon as I tried to start Xorg on the guest system it would segfault inside OsLookupColor.

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

Note that ```<qemu:commandline>``` will fail to validate if the attribute ```xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'```is not set on the XML root ```domain``` node.

The ```memoryBacking``` part specifies that pages used by the virtual machine may not be swapped out by the host. If not specified qemu will try to lock all pages (for the graphics memory?) individually resulting in exceeding the RLIMIT_MEMLOCK limit.

```qemu:commandline``` will need adjustments to work with your setup. The uuid ```fc444019-54df-47bc-a0a0-7308ce9681a8``` must be replaced with the UUID used for creating the virtual GPU.

## Guest setup

Gust setup should be pretty straight forward. No additional configuration was required to start Xorg.

I was unable to start Xorg with an additional emulated graphics card though. Any attempt to start Xorg or even run ```Xorg -configure``` resulted in a segfault inside OsLookupColor. Using only the Intel KVMGT card "fixed" that issue though.

## EFI

I have not been able to get KVMGT to work with UEFI on the guest system. I tried booting with various versions of OVMF but OVMF startup always hangs as soon as it starts probing the vGPU PCI device. The debug log output of OVMF is attached as ```ovmf.log```.
