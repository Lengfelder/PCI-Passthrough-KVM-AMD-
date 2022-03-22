# QEMU-KVM GPU- and NVME PCI-passthrough via OVMF
You're probably here for the same reason i visited [Bryansteiner's](https://github.com/bryansteiner/gpu-passthrough-tutorial)[^1] great tutorial on how to set up and configure KVM with GPU Passthrough. 
[YuriAlek's  Single GPU passthrough](https://gitlab.com/YuriAlek/vfio#start-here)-guide was also wholefully helpful, don't read mine. Just read his and you're good to go.

### Credits
Bryansteiner: https://github.com/bryansteiner/gpu-passthrough-tutorial  
Arch Wiki: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Dynamic_huge_pages  
RedHat on performance tuning: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/virtualization_tuning_and_optimization_guide/index#sect-specifying_VM_details


# Hardware 
## My sick gamestation Setup
- CPU: 
  - AMD Ryzen 5 3600 6c/12t
- Motherboard: 
  - Asus X570-F
- GPUs:
  - NVIDIA RTX 2070 Super (guest)
  - NVIDIA GT 1050ti (host)
- Memory:
  - Gskill DDR4 2133Mhz Dual-Rank 2x8GB
- Disk:
  - Intel 660p 2TB - M.2 NVMe (guest)
  - Samsung 970 EVO Plus SSD 1TB - M.2 NVMe (host)
- Screen
  - CRG9 49" 5120x1440 w/ KVM-switch
  - Looking Glass
- Extra keyboard/mouse


# PCI-Passthrough-KVM-AMD-
My private how-to on how to run Windows 10 virtualized inside Pop_OS!
This is only applicable to my specific system and level of understanding. 
Work in progress, hopefully it will turn in to something you can use.

# Prerequisites

### Hardware that supports IOMMU. 
Both CPU and Motherboard need support. I haven't seen unsupported hardware in years.  
Google your CPU- and Motherboard-model to verity support for Virtualization.  
Both your GPU's need to be UEFI Compatible. iGPU's normally is.

### OpenSSH
Sytem76 has a great guide on how to set up SSH on your system. This is recomended to do, just in case something goes wrong and you lose display output.
You can find it [here](https://support.system76.com/articles/server-setup/).
While we're at it, you don't need another computer to fix things. Just use your [Android](https://itsfoss.com/using-linux-terminal-android/) or [iPhone](https://www.noupe.com/inspiration/5-best-ssh-terminal-apps-for-iphone.html).

### Necessary(mostly) packages
``` 
$ sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils qemu-utils virt-manager ovmf
``` 
Make your user member of the kvm and libvirt groups (not sure this is necessary):

```
sudo usermod -a -G kvm myusername
```
```
sudo usermod -a -G libvirt myusername
```

### ISO-Files

You will need a Windows ISO and drivers to go with it.
virtIO drivers from [here](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md), and windows ISO from [here](https://www.microsoft.com/en-us/software-download/windows10ISO) is all you need.  
Move the files. This makes life easier in later steps.
`$ sudo mv Win10_21H2_EnglishInternational_x64.iso virtio-win-0.1.215.iso /var/lib/libvirt/images`

# Configuring IOMMU [<sup><sub>(What even is it?)</sup></sub>](https://www.quora.com/Memory-Management-computer-programming/Memory-Management-computer-programming-Could-you-explain-IOMMU-in-plain-English)
### Does your system support virtualization?
If you have a modern system, chances are your hardware is compatible with IOMMMU. If not, you need new hardware.  
IOMMU is something you can toggle on/off in UEFI/BIOS and is just a generic name for Intel VT-d and AMD-Vi
This setting will usually be called Virtualization-something or their actual names:
VT-d and AMD-Vi for Intel and AMD respectivly.  
Reboot to BIOS/UEFI and enable everything you deem related to virtualization (including `SVM`)

### Make sure IOMMU is enabled:
After booting back up, run `$ dmesg | grep IOMMU` to see if IOMMU is detected.

```
[    0.321158] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.326147] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
[    0.327382] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
``` 
If your output looks something like mine, you're good to go.

### To make sure virtualization is properly enabled
If you're a GRUB user, check out [Void's section on editing GRUB for Single GPU Passthrogh](https://gitlab.com/risingprismtv/single-gpu-passthrough/-/wikis/2)

Some system-updates may overwrite your bootloader, but luckily System76 thought of that and included [kernelstub](https://github.com/pop-os/kernelstub), an automatic manager for booting Linux on (U)EFI. 

So instead of adding the following options to the bootloader, we do them like this:  
`$ sudo kernelstub --add-options iommu=pt amd_iommu=on`  
If you're on Intel, just use `intel_iommu=on` instead of `amd_iommu=on`  
`iommu=pt` is recomended for security reasons and tells the system to not touch devices that cannot be passed throgh.  

#### Reboot and verify
run: `$ dmesg |grep AMD-Vi`or `$ dmesg |grep VT-d` for Intel.  
Output should look like this:  
``` 
[    0.321158] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.326147] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
[    0.326151] AMD-Vi: Extended features (0x58f77ef22294ade): PPR X2APIC NX GT IA GA PC GA_vAPIC
[    0.326155] AMD-Vi: Interrupt remapping enabled
[    0.326156] AMD-Vi: Virtual APIC enabled
[    0.326156] AMD-Vi: X2APIC enabled
```
#### Unsure which bootloader you use?
You can check which bootloader you have by running `$ efibootmgr -v`
In my case the output looks like this, and we see that i use systemd.
```
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0006,0005
Boot0001* Pop!_OS 21.10	HD.../File(\EFI\SYSTEMD\SYSTEMD-BOOTX64.EFI)
``` 


# Isolating passthrough-devices

## Isolating the GPU (and nvme-SSD)
It may not be nessecary to isolate the PCI-devices on kernel-level if you can get the hook to work. I couldt.
Now, in this modern day and age this should theoretically not be nessecary, but i coundt get the hooks[^3] to attach/detach the gpu properly. I found my system to be more stable with static parameters. Results may vary.

Now that it's verified your machine supports virtualization, we're going to prepare our PCIe-devices to be passed throgh to the VM.

### List out your PCI Devices
The first thing we need to to is identify our PCI-devices and which ones we want to pass though to out VM. 
 To do this, create a script `$ touch filename.sh` in a folder on your dekstop and paste in the code below. 
 ```
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
To make life easy, we print the output to a file like so: `sh filename.sh >> outputoffilename.txt`   
This will list all you iommu-groups and tell you if this will be a brease or a headache.
It will look something like this:

``` 
IOMMU Group 10:
-e 	00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1234]
-e 	00:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1233]
-e 	00:08.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1233]
-e 	00:08.3 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1233]
-e 	0c:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1223]
-e 	0c:00.1 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Cryptographic Coprocessor PSPCPP [1022:1222]
-e 	0c:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:1232]
-e 	0c:00.4 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse HD Audio Controller [1022:1122]
-e 	0d:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:2211] (rev 51)
-e 	0e:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:2211] (rev 51)
IOMMU Group 20:
-e 	04:00.0 Non-Volatile memory controller [0108]: Intel Corporation SSD 660P Series [8086:f1a8] (rev 03)
IOMMU Group 22:
-e 	09:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107 [GeForce GTX 1050 Ti] [10de:1c82] (rev a1)
-e 	09:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
IOMMU Group 23:
-e 	0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU104 [GeForce RTX 2070 SUPER] [10de:1e84] (rev a1)
-e 	0a:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f8] (rev a1)
-e 	0a:00.2 USB controller [0c03]: NVIDIA Corporation TU104 USB 3.1 Host Controller [10de:1ad8] (rev a1)
-e 	0a:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU104 USB Type-C UCSI Controller [10de:1ad9] (rev a1)
```
As you can see, i was lucky because my GPU(s) and NVME-drive are in seperate IOMMU-groups (group 20 and 23). In my case i had my keyboard connected to group 10, this would not let me boot the VM. You need all devices in a group passed through. Hence the importance of device isolation.
The RTX cards often have four devices, whereas earlier cards usually have two.

If this is not the case for you, check out the ACS Override Patch. A kernel-patch with some [security concerns](https://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html). Pre-patched kernels can be found [here](https://queuecumber.gitlab.io/linux-acs-override/).
You could also try to different PCI-e and NVME-slots on your motherboard to see if the other ports are isolated.

### Bind devices to vfio at boot (or hook them)
This is done so other drivers don't claim the device before you start your VM. I couldn't get the hooks to work properly so i decided to bind them permenemtly to vfio at boot for the sake of system stability and reliablity.

From the previous file, write down the Device ID and replace mine in the following command:
```
$ sudo kernelstub --add-options kvm.ignore_msrs=1 vfio-pci.ids=10de:1e84,10de:10f8,10de:1ad8,10de:1ad9,8086:f1a8"

```
`kvm.ignore_msrs=1` is needed to prevent BSOD on Win10 1803 and later.
`vfio-pci.ids=xxxx:xxxx` tells the bootloader to bind those IDs to vfio-driver.

Reboot for the changes to take effect.

#### Verify

After boot, run `$ lspci -nnv` to verify that your devices use vfio.
Look for `Kernel driver in use: vfio-pci`. This should be on all devices you intend to pass through.


# Configure the VM

Now we'reready to actually create the VM that we will pass devices through. 
Go ahed and search for `kvm` in your applications to start the Virtual Machine Manager (virt-manager) 

We will need to edit the XML. Make sure you can by ticking the box

![Screenshot from 2022-03-22 16-00-47](https://user-images.githubusercontent.com/21096648/159518951-e243566c-4c25-4479-b961-fd28a68e9461.png)

Next, go to File -> Create New Virtual Machine

Select Local install media
![Screenshot from 2022-03-22 15-56-59](https://user-images.githubusercontent.com/21096648/159514753-6622a78e-9de4-44ab-aeb0-2757661f4c7e.png)

And use your Windows 10 ISO from previously. Make sure it detects the correct OS.
![Screenshot from 2022-03-22 16-00-01](https://user-images.githubusercontent.com/21096648/159515769-49416b91-8c31-4de4-8504-c89ec611f32c.png)

We'll revisit the amount of RAM and CPU cores later. Doesn't hurt to change it now. (4096x2=8192)
![Screenshot from 2022-03-22 16-00-13](https://user-images.githubusercontent.com/21096648/159517236-84cabba1-434c-41db-afa5-1b7fe1142000.png)

Leave this empty. We use a passed through drive, hence we don't need a virtual one. Leave it empty even if you're planning on using a virtual drive and create the drive properly yourself. Please see [Heiko Disk Tuning guide](https://www.heiko-sieger.info/tuning-vm-disk-performance/)
![Screenshot from 2022-03-22 16-00-19](https://user-images.githubusercontent.com/21096648/159517362-2f91a256-b79b-425e-bac0-4a31d1fc3b6d.png)

Name your VM and check `Customize configuration before install` 
![Screenshot from 2022-03-22 16-00-31](https://user-images.githubusercontent.com/21096648/159518551-e70ff0c0-752b-48d8-8861-3be869226d19.png)

Start by right clicking -> Remove hardware on what you don't need.
For me, its: 
- Tablet
- Display Spice
- Sound ich9
- Console 1
- Channel spice
- Video QXL

Also make sure to use Q35 for Chipset and OVMF_CODE for Firmware. Many guides tell you to use `OVMF_CODE.fd` which i did not have, but i had luck using the 4M.ms.fd variant. You can read more about the differences..where?
![Screenshot from 2022-03-22 16-34-16](https://user-images.githubusercontent.com/21096648/159519871-a798ec23-dd58-4dec-a460-5c9077401949.png)
Hit Apply.

Now we're gonna add our boot drive and virtIO-drivers by clicking Add Hardware in the downmost left cornet.



# Installing Windows
Next next OK Finish

# Tuning

### XML
```
<features>
    ...
    <hyperv>
        <relaxed state="on"/>
        <vapic state="on"/>
        <spinlocks state="on" retries="8191"/>
        <vendor_id state="on" value="kvm hyperv"/>
        
    </hyperv>
     <kvm>
      <hidden state="on"/>
    </kvm>
    ...
    <ioapic driver="kvm"/>
</features>
```
```
 
```
```
ignore_msrs=1
```
```
```



# Hooks
## CPU Hooks
```
#!/bin/sh

command=$2

if [ "$command" = "started" ]; then
    systemctl set-property --runtime -- system.slice AllowedCPUs=0,1,6,7
    systemctl set-property --runtime -- user.slice AllowedCPUs=0,1,6,7
    systemctl set-property --runtime -- init.scope AllowedCPUs=0,1,6,7
elif [ "$command" = "release" ]; then
    systemctl set-property --runtime -- system.slice AllowedCPUs=0-11
    systemctl set-property --runtime -- user.slice AllowedCPUs=0-11
    systemctl set-property --runtime -- init.scope AllowedCPUs=0-11
fi
```

### Hugepages

```
<memory unit="KiB">16777216</memory>
<currentMemory unit="KiB">16777216</currentMemory>
<memoryBacking>
    <hugepages/>
</memoryBacking>
```

### CPU pinning

```
<cpu mode="host-passthrough" check="none">
  <topology sockets="1" cores="6" threads="2"/>
  <cache mode='passthrough'/>
  <feature policy='require' name='topoext'/>
</cpu>
```


# References:

### Initial Configuration

[^1]: [OpenSSH-Server setup](https://support.system76.com/articles/server-setup/)

### Virtual Machine configuration

### System Tuning

     
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

https://www.redhat.com/en/blog/using-kvm-simulate-numa-configurations-openstack
On NUMA nodes cpu//ram

https://github.com/joeknock90/Single-GPU-Passthrough#libvirt-hook-scripts
another hook guide


