# KIKA netboot

Services used are: dnsmasq, nginx, syslinux (for pxelinux, menu, vesamenu, etc..), and nfs for ubuntu live.
Supports BIOS or UEFI net boot.

Boot options:

* Archlinux net install
* Ubuntu live boot
* Fedora live boot
* Debian net install
* CentOS net install
* Memtest86+
* SystemRescueCD
* Darik's boot and nuke

Everything lives in `/home/netboot`:
* `/home/netboot/iso` - distro `.iso` image files are downloaded here
* `/home/netboot/tftproot` - the pxelinux config files
* `/home/netboot/nfs` - `.iso` image files are loop mounted here

## Dnsmasq (dhcp + tftp)

The dnsmasq config (in `/etc/dnsmasq.d/pxe.conf`) is:

```
enable-tftp
tftp-root=/home/netboot/tftproot

dhcp-match=x86PC, option:client-arch,0
dhcp-match=BC_EFI, option:client-arch,7
dhcp-match=BC_EFI, option:client-arch,9
dhcp-boot=tag:x86PC,/syslinux/bios/lpxelinux.0
dhcp-boot=tag:BC_EFI,/syslinux/efi64/syslinux.efi
dhcp-option-force=tag:x86PC,209,::/pxelinux.cfg/default
dhcp-option-force=tag:BC_EFI,209,::/pxelinux.cfg/default
```

This config supports both BIOS and UEFI machines, giving them the coresponding pxelinux bootloader.

To start and enable dnsmasq run:
```
sudo systemctl enable --now dnsmasq.service
```
## PXELinux

In our case `/home/netboot/tftproot/syslinux` is a copy of `/usr/lib/syslinux/` from Archlinux - cause it was much newer than the syslinux from Debian jessie.
But one can also use the syslinux official binary releasess or a symlink to `/usr/lib/syslinux/` - mind the different structure though.

On BIOS systems we use `bios/lpxelinux.0` - has the most of features, on UEFI we use `efi64/syslinux.efi`.

Check the other files in this directory, starting from `default` to see the syslinux menu configuration used.

## NFS

To setup a read-only export of the whole `/home/netboot/nfs` directory, put the following line in `/etc/exports`:
```
/home/netboot/nfs *(ro,async,no_subtree_check,no_root_squash,insecure,crossmnt)
```
and then just enable and start the nfs-server service:
```
sudo systemctl enable --now nfs-server.service
```

## nginx

need to expose these two with http:
```
  location /tftproot {
      alias /home/netboot/tftproot/;
      autoindex off;
      allow all;
  }
  location /nfs {
      alias /home/netboot/nfs/;
      autoindex off;
      allow all;
  }
```

Don't forget to enable and start nginx too:
```
sudo systemctl enable --now nginx.service
```

## Distros

For ubuntu and other casper based distros (Mint etc), Fedora and Arch you only need the `.iso` image file loop mounted.
For example (the fstab entry):

```
/home/netboot/iso/ubuntu-16.04.2-desktop-amd64.iso /home/netboot/nfs/ubuntu-16.04.2-desktop-amd64 auto loop,nofail,ro 0 0
```

If running a lot of distros, make sure to enlarge the maximum allowed loop devices:
```
#/etc/modprobe.d/local.conf 
options loop max_loop=32
```

# Booting explained for dummies

How booting works, step by step:

1. Computer is turned on
1. BIOS is loaded and run
1. We select network boot
1. PXE part of BIOS is loaded and run
1. PXE BIOS requests a DHCP address
1. dnsmasq replies with an address and boot server and filename (pxelinux.0)
1. PXE BIOS uses tftp to download pxelinux.0 and runs it
1. pxelinux (inherits the IP setup from the BIOS)
   it uses tftp to download a config file (typically pxelinux.cfg/default)
1. the config file is a typical syslinux configuration specifying a 
   kernel and initial ramfs (initrd) an archive unpacked in a ram filesystem
1. pxelinux.0 tftp downloads the choosen kernel and initrd in RAM
1. the kernel runs, initializes itself, and finds the ram filesystem
1. /init in the ramfs is run as process 1 (ussually a script) 
