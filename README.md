# smol-gpu-passthrough

Set up GPU passthrough on Debain &amp; Ubuntu hosts.

## CAUTION

- This script will modify `/etc/default/grub` and could cause your machine to enter and un-recoverable state. Do not run this script on a machine you are not willing to re-image.

- This script has only been tested on Intel and Nvidia hardware. 

## Usage

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
    # Set up GPU passthrough
    bash setup.sh full_run NVIDIA

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
    -device vfio-pci,host=02:00.0,multifunction=on,x-vga=on
    ```


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

Cloud-Init Resources:

- [My Magical Adventure With cloud-init](https://christine.website/blog/cloud-init-2021-06-04) - Xe Iaso

Hypervisor Resources:

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
