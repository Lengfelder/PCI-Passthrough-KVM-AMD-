# PCI-Passthrough-KVM-AMD-
My private how-to on how to run Windows 10 virtualized inside Pop_OS!

# Hardware Setup
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



´´´
 #!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
´´´


> sudo kernelstub --add-options "amd_iommu=on kvm.ignore_msrs=1 vfio-pci.ids=10de:1e84,10de:10f8,10de:1ad8,10de:1ad9,8086:f1a8"
