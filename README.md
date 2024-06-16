# GPU-Passthrough using OVMF + VFIO/IOMMU

Set up GPU passthrough on Debain &amp; Ubuntu hosts.

## CAUTION

- This script will modify `/etc/default/grub` and could cause your machine to enter and un-recoverable state. Do not run this script on a machine you are not willing to re-image.

- This script has only been tested on Intel and Nvidia hardware. 

- Do not install your GPU drivers on the host machine unless you are using a laptop with Nvidia Prime/Optimus.

## Quickstart

1. Enable IOMMU in grub

    ```bash
    # /etc/default/grub
    GRUB_DEFAULT=0
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
    GRUB_CMDLINE_LINUX_DEFAULT="quiet preempt=voluntary iommu=pt amd_iommu=on intel_iommu=on"
    GRUB_CMDLINE_LINUX=""
    ```
2. Update grub and reboot

    ```bash
    sudo update-grub
    sudo reboot now
    ``` 
    
3. Download and run `setup.sh`

    ```bash
    # Set up GPU passthrough for your distro
    # ubuntu
    bash setup.sh full NVIDIA ubuntu
    # debian
    bash setup.sh full NVIDIA debian
    
    # Reset 
    bash setup.sh reset
    ```

4. Reboot the machine


5. You are now ready to present the GPU to your VMM. An example using QEMU:

    ```bash
    > lspci |grep -ai nvidia |grep VGA | awk '{print $1}'
    > 02:00.0
    ```
    
    ```bash
    sudo qemu-system-x86_64 -machine accel=kvm,type=q35 \
        -cpu host,kvm=off,hv_vendor_id=null \
        -smp 4,sockets=1,cores=2,threads=2,maxcpus=4 \
        -m 8G \
        -vga virtio -serial stdio -parallel none \
        -device vfio-pci,host=02:00.0,multifunction=on,x-vga=on \
        -netdev user,id=network0,hostfwd=tcp::1234-:22 \
        -device virtio-net-pci,netdev=network0 \
        -drive if=virtio,format=qcow2,file=disk.qcow2,index=1,media=disk \
        -drive if=virtio,format=raw,file=seed.img,index=0,media=disk \
        -bios /usr/share/ovmf/OVMF.fd \
        -usbdevice tablet \
        -vnc 192.168.50.100:0
    ```

## Slow-Start

This is a long-form explanation of the `setup.sh` script that explains the underlying automation process. It is assumed that the reader has a fresh install of Debian 12 (no gui), or Ubuntnu Server and is using a NVIDIA GPU and an x86_64 Intel CPU. 

- AMD devices have NOT been tested. 
- ARM devices have NOT been tested.

### Enabling IOMMU

 - Enable IOMMU by changing the `GRUB_CMDLINE_LINUX_DEFAULT` line in your `/etc/default/grub` file to the following:

   ```bash
    GRUB_CMDLINE_LINUX_DEFAULT="quiet preempt=voluntary iommu=pt amd_iommu=on intel_iommu=on"
    ```
    Then run `sudo update-grub`.
    
    > The `preempt` option is also enabled here to reduce boot-times for systems with large amounts of RAM.
    
 - Install dependancies 

   ```bash
   sudo apt-get -y install \
         qemu-kvm \
         bridge-utils \
         virtinst \
         ovmf \
         qemu-utils \
         cloud-image-utils \
         curl
   ```

 - Reboot (Required)


### Gathering IOMMU data

  Now that IOMMU is enabled we can look for devices in `/sys/kernel/iommu_groups`. The formatting is awful by default so here is a small script to list it in a more readable way courtesy of leduccc.medium.com 

   - [SOURCE](https://leduccc.medium.com/simple-dgpu-passthrough-on-a-dell-precision-7450-ebe65b2e648e)
   
   ```bash
   curl -fsSL "https://raw.githubusercontent.com/cloudymax/Scrap-Metal/main/virtual-machines/host-config-resources/iommu-groups.sh" | /bin/bash
   ```
   
   The output of the above script will list all IOMMU groups, as well as the PCI ID, a description of each of your PCI devices, and at the end of the line is the IOMMU ID that we require. You will need to find the group number that your graphics card belongs to, and the IOMMU IDs of each item in that group. In the case of the example blow, the IOMMU IDs we need are `10de:1f08`, `10de:10f9`, `10de:1ada`, and `10de:1adb`.


<details>
  <summary>Click to expand</summary>
   
   ```bash
   IOMMU Group 0 00:02.0 VGA compatible controller [0300]: Intel Corporation RocketLake-S GT1 [UHD Graphics 750] [8086:4c8a] (rev 04)
   IOMMU Group 1 00:00.0 PCI bridge [0604]: Intel Corporation Device [8086:4c43] (rev 01)
   IOMMU Group 2 00:01.0 PCI bridge [0604]: Intel Corporation Device [8086:4c01] (rev 01)
   IOMMU Group 3 00:04.0 Signal processing controller [1180]: Intel Corporation Device [8086:4c03] (rev 01)
   IOMMU Group 4 00:08.0 System peripheral [0880]: Intel Corporation Device [8086:4c11] (rev 01)
   IOMMU Group 5 00:12.0 Signal processing controller [1180]: Intel Corporation Comet Lake PCH Thermal Controller [8086:06f9]
   IOMMU Group 6 00:14.0 USB controller [0c03]: Intel Corporation Comet Lake USB 3.1 xHCI Host Controller [8086:06ed]
   IOMMU Group 6 00:14.2 RAM memory [0500]: Intel Corporation Comet Lake PCH Shared SRAM [8086:06ef]
   IOMMU Group 7 00:14.3 Network controller [0280]: Intel Corporation Comet Lake PCH CNVi WiFi [8086:06f0]
   IOMMU Group 8 00:15.0 Serial bus controller [0c80]: Intel Corporation Comet Lake PCH Serial IO I2C Controller #0 [8086:06e8]
   IOMMU Group 9 00:16.0 Communication controller [0780]: Intel Corporation Comet Lake HECI Controller [8086:06e0]
   IOMMU Group 10 00:17.0 SATA controller [0106]: Intel Corporation Comet Lake SATA AHCI Controller [8086:06d2]
   IOMMU Group 11 00:1b.0 PCI bridge [0604]: Intel Corporation Comet Lake PCI Express Root Port #21 [8086:06ac] (rev f0)
   IOMMU Group 12 00:1c.0 PCI bridge [0604]: Intel Corporation Device [8086:06bc] (rev f0)
   IOMMU Group 13 00:1f.0 ISA bridge [0601]: Intel Corporation H470 Chipset LPC/eSPI Controller [8086:0684]
   IOMMU Group 13 00:1f.3 Audio device [0403]: Intel Corporation Device [8086:f1c8]
   IOMMU Group 13 00:1f.4 SMBus [0c05]: Intel Corporation Comet Lake PCH SMBus Controller [8086:06a3]
   IOMMU Group 13 00:1f.5 Serial bus controller [0c80]: Intel Corporation Comet Lake PCH SPI Controller [8086:06a4]
   IOMMU Group 14 02:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106 [GeForce RTX 2060 Rev. A] [10de:1f08] (rev a1)
   IOMMU Group 14 02:00.1 Audio device [0403]: NVIDIA Corporation TU106 High Definition Audio Controller [10de:10f9] (rev a1)
   IOMMU Group 14 02:00.2 USB controller [0c03]: NVIDIA Corporation TU106 USB 3.1 Host Controller [10de:1ada] (rev a1)
   IOMMU Group 14 02:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU106 USB Type-C UCSI Controller [10de:1adb] (rev a1)
   IOMMU Group 15 03:00.0 Non-Volatile memory controller [0108]: ADATA Technology Co., Ltd. XPG SX8200 Pro PCIe Gen3x4 M.2 2280 Solid State Drive [1cc1:8201] (rev 03)
   ```
</details>


### Enable VFIO-PCI and disable conflicting kernel modules

In order to pass control of the GPU to the VM we will need to hand over control of the PCI devices to VFIO. This only works though if VFIO has control of ALL items in the GPU's IOMMU group.

   - edit/create `/etc/initramfs-tools/modules` (Debian), or `/etc/initram-fs/modules` (Ubuntu) to include the following:
   
      ```bash
      vfio
      vfio_iommu_type1
      vfio_pci
      vfio_virqfd
      options vfio-pci ids=<someID,someOtherID>
      ```
   
   - edit/create `/etc/modprobe.d/blacklist.conf` (Debian), or `/etc/modprobe.d/local.conf` (Ubuntu)
      
      ```bash
      options vfio-pci ids=<someID,someOtherID>
      ```
      
   - edit the `GRUB_CMDLINE_LINUX_DEFAULT` line of your `/etc/default/grub` file again to the following:
   
      ```bash
      GRUB_CMDLINE_LINUX_DEFAULT="quiet preempt=voluntary iommu=pt amd_iommu=on intel_iommu=on vfio-pci.ids=<<someID,someOtherID> rd.driver.pre=vfio-pci video=efifb:off kvm.ignore_msrs=1 kvm.report_ignored_msrs=0"
      ```
  
   - Now run `sudo update-grub`, `sudo update-initramfs -u`, `sudo depmod -ae` and then reboot. (Required)


### Disabling conflicting kernel drivers

A common issue I have seen others encounter with this process is that VFIO is not given control of all devices in the GPU's IOMMU group. Most often this is due to the xhci_hcd USB module retaining control of the GPU's USB controller. 

  > As per the [Debian Wiki](https://wiki.debian.org/KernelModuleBlacklisting) - to disable this kernel module, or any other you can:
  > 
  > - Create a file `/etc/modprobe.d/<modulename>.conf` containing `blacklist <modulename>`.
  > - Run `sudo depmod -ae` as root
  > - Recreate your initrd with `sudo update-initramfs -u`
 
So I will now edit/create `/etc/modprobe.d/xhci_hcd.conf` to contain
      
  ```bash
  blacklist xhci_hcd
  ```
   
Then I will run `sudo update-initramfs -u`, `sudo depmod -ae` and reboot. (Required)


### Verify VFIO control over PCI Devices

   After your machien reboots, run `lspci -nnk` to show which kernel driver has control over each PCI device. All devices should show `vfio-pci` as the kernel driver in use. If not, you will need to repeat the previous steps to disable that driver.


<details>
  <summary>Click to expand</summary>
   
   ```bash
   02:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106 [GeForce RTX 2060 Rev. A] [10de:1f08] (rev a1)
	   Subsystem: ASUSTeK Computer Inc. TU106 [GeForce RTX 2060 Rev. A] [1043:86f0]
	   Kernel driver in use: vfio-pci
	   Kernel modules: nouveau
   02:00.1 Audio device [0403]: NVIDIA Corporation TU106 High Definition Audio Controller [10de:10f9] (rev a1)
	   Subsystem: ASUSTeK Computer Inc. TU106 High Definition Audio Controller [1043:86f0]
	   Kernel driver in use: vfio-pci
	   Kernel modules: snd_hda_intel
   02:00.2 USB controller [0c03]: NVIDIA Corporation TU106 USB 3.1 Host Controller [10de:1ada] (rev a1)
	   Subsystem: ASUSTeK Computer Inc. TU106 USB 3.1 Host Controller [1043:86f0]
	   Kernel driver in use: vfio-pci
	   Kernel modules: xhci_pci
   02:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU106 USB Type-C UCSI Controller [10de:1adb] (rev a1)
	   Subsystem: ASUSTeK Computer Inc. TU106 USB Type-C UCSI Controller [1043:86f0]
	   Kernel driver in use: vfio-pci
   ```
</details>

At this point your host machine is ready to create GPU accelerated guest VMs.


## Resources and Help

This is not a new development or an original work. It's bits and pieces of knowledge from people much smarter than myself that have been cut-and-pasted into a slightly easier-to-use format.

GPU Passthrough resources:

- [GPU Passthrough on a Dell Precision 7540 and other high end laptops](https://leduccc.medium.com/simple-dgpu-passthrough-on-a-dell-precision-7450-ebe65b2e648e) - leduccc

- [Improving the performance of a Windows Guest on KVM/QEMU](https://leduccc.medium.com/improving-the-performance-of-a-windows-10-guest-on-qemu-a5b3f54d9cf5) - leduccc

- [Comprehensive guide to performance optimizations for gaming on virtual machines with KVM/QEMU and PCI passthrough](https://mathiashueber.com/performance-tweaks-gaming-on-virtual-machines/) - Mathias Hüber

- [Virtual machines with PCI passthrough on Ubuntu 20.04, straightforward guide for gaming on a virtual machine](https://mathiashueber.com/pci-passthrough-ubuntu-2004-virtual-machine/) - Mathias Hüber

- [gpu-virtualization-with-kvm-qemu](https://medium.com/@calerogers/gpu-virtualization-with-kvm-qemu-63ca98a6a172) - Cale Rogers

- [Faster Virtual Machines on Linux Hosts with GPU Acceleration](https://adamgradzki.com/2020/04/06/faster-virtual-machines-linux/) - Adam Gradzki

  |Author| Year |CPU | GPU | OS | modules method | pci-ids medthod |
  |--|--|--|--|--|--|--|
  | Leducc | 2020 | Intel | Nvidia | Manjaro |/etc/mkinitcpio.conf |GRUB_CMDLINE_LINUX_DEFAULT| 
  | Mathias Hüber | 2021 | AMD | Nvidia | Ubuntu 18.04 & 20.04 | /etc/initramfs-tools/modules|GRUB_CMDLINE_LINUX_DEFAULT and /etc/initramfs-tools/scripts/init-top/vfio.sh|
  | Cale Rogers | 2016 | Intel | Nvidia | Ubuntu 16.04 | GRUB_CMDLINE_LINUX and /etc/initram-fs/modules|/etc/modprobe.d/local.conf|
  | Adam Gradzki | 2020 | Intel | Intel | ?? | ---| created by i915-GVTg_V5_2 |

## Cloud-Init Resources:

- [My Magical Adventure With cloud-init](https://christine.website/blog/cloud-init-2021-06-04) - Xe Iaso

## Hypervisor Resources:

- [A Study of Performance and Security Across the Virtualization Spectrum](https://repository.tudelft.nl/islandora/object/uuid:34b3732e-2960-4374-94a2-1c1b3f3c4bd5/datastream/OBJ/download) - Vincent van Rijn

- [virtualization-hypervisors-explaining-qemu-kvm-libvirt](https://sumit-ghosh.com/articles/virtualization-hypervisors-explaining-qemu-kvm-libvirt/) by Sumit Ghosh

Kubernetes/Docker Resources:

- [Schedule GPUs in K8s](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#deploying-amd-gpu-device-plugin)

- [NVIDIA Container Toolkit install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

Kernel Options Docs:

- [Linux Kernel Params](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html)

- [Intel i915 Driver Options](https://www.kernel.org/doc/html/latest/gpu/i915.html?highlight=vfio%20pci)

- [PCI VFIO options](https://www.kernel.org/doc/html/latest/driver-api/vfio-pci-device-specific-driver-acceptance.html?highlight=vfio%20pci)

- [KMV Ignore MSRs](https://www.kernel.org/doc/html/latest/virt/kvm/x86/msr.html?highlight=kvm%20ignore%20msrs)

- [root device (rd) kernel options](https://man7.org/linux/man-pages/man7/dracut.cmdline.7.html)
