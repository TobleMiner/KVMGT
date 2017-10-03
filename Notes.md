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

Before using the KVMGT GPU it must be created.


