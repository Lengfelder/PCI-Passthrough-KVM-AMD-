# PCI-Passthrough-KVM-AMD-
My private how-to on how to run Windows 10 virtualized inside Pop_OS!

## Hardware Setup
- CPU: 
AMD Ryzen 5 3600 6/12
- Motherboard: 
Asus X570-F
- GPUs:
NVIDIA RTX 2070 Super
NVIDIA GT 1050ti
- Memory:
16GB Gskill DDR4 
- Disk:
Intel 660p 2TB - M.2 NVMe (guest)
Samsung 970 EVO Plus SSD 1TB - M.2 NVMe (host)

## Software setup
I will be using Pop_OS! 21.10 Nvidia Edition
This is Ubuntu-based, but instead of GRUB bootloader it uses systemd and has kernelstub preinstalled.

#### OpenSSH setup
This is recomended if something goes wrong
# Preparing Linux system





1. The first thing we need to to is identify our PCI-devices and which ones we want to pass though to out VM. 
 To do this, create a script (touch iommu.sh) in a folder on your dekstop and paste in the code below. 
 ```
 #!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```
   - Now you want to output this to a file: "sh ls-iommu.sh >> iommuoutput01.txt"
   - Open the file and locate the desired devices. In my case it's a RTX and NVME drive:

```
IOMMU Group 20 04:00.0 Non-Volatile memory controller [0108]: Intel Corporation SSD 660P Series [8086:f1a8] (rev 03)

IOMMU Group 22 09:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] [10de:1c82] (rev a1)
IOMMU Group 22 09:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)

IOMMU Group 23 0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU104 [GeForce RTX 2070 SUPER] [10de:1e84] (rev a1)
IOMMU Group 23 0a:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f8] (rev a1)
IOMMU Group 23 0a:00.2 USB controller [0c03]: NVIDIA Corporation TU104 USB 3.1 Host Controller [10de:1ad8] (rev a1)
IOMMU Group 23 0a:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU104 USB Type-C UCSI Controller [10de:1ad9] (rev a1)
```
As you can see, both my GPUs and one of my nvme-drives is listed. The RTX cards often have four devices, whereas earlier cards usually have two.

When the devices are identified, extract the [xxxx:xxxx] to a new file. This is the PCIe identifiers that are needed for this project.
In my case, they look like this:
```
sudo kernelstub --add-options "iommu=pt amd_iommu=on kvm.ignore_msrs=1 vfio-pci.ids=10de:1e84,10de:10f8,10de:1ad8,10de:1ad9,8086:f1a8"
(maybe use -o, not --add-o)

```
The first four are the GPU, and the last is the drive.
The important parameter are the ones in the options line: amd_iommu=on iommu=pt. iommu=pt will prevent Linux from touching devices which cannot be passed through and amd_iommu=on turns IOMMU on of course. Add them accordingly.


2. Now we'll need to install the required software:
```
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf
sudo apt install                                                             virt-manager ovmf
sudo apt install qemu-kvm qemu-utils libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf
sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients  virt-manager ovmf
```

Make your user member of the kvm and libvirt groups (not sure this is necessary):

```
sudo usermod -a -G kvm myusername
```
```
sudo usermod -a -G libvirt myusername
```

# Download ISOs
- We only need two .iso-files, one with Windows and one with drivers.
- Up-To date VIRTIO-drivers can be found here:
- https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md
- If you don't have one already, a plain W10-image can be downloaded directly from the source_
- https://www.microsoft.com/en-us/software-download/windows10ISO

# Configure KVM VM

# Installing Windows

# Tuning

### Hooks

# References:

https://www.tauceti.blog/posts/linux-amd-x570-nvidia-gpu-pci-passthrough-2-prepare-linux/
https://www.heiko-sieger.info/creating-a-windows-10-vm-on-the-amd-ryzen-9-3900x-using-qemu-4-0-and-vga-passthrough/

https://bbs.archlinux.org/viewtopic.php?id=250297
iommu=pt

https://www.oprocs.com/pop_os-gpu-passthrough-guide/
Easy to understand

https://forum.level1techs.com/t/voltage-control-on-a-kvm-passed-through-1080/181937/3
Voltage control nvidia gpu

https://www.reddit.com/r/VFIO/comments/thp3q2/hyperv_gpup_performance_tip/
Hyper-V display slow

https://www.reddit.com/r/VFIO/comments/teonsr/fan_speed_at_100_when_vm_offline/
fan speed

https://www.reddit.com/r/VFIO/comments/titakv/my_fully_almost_automatic_single_gpu_passthrough/
hooks


