## Linux kernel with Microsoft Hypervisor (MSHV) support

This branch contains cherry-picked commits from Azure Linux MSHV kernel tag [rolling-lts/kata/6.6.100.mshv1](https://github.com/microsoft/CBL-Mariner-Linux-Kernel/tree/rolling-lts/kata/6.6.100.mshv1) which enables support to boot Microsoft Hypervisor (MSHV) after you download and extract those binaries with script like this:
```bash
#!/bin/bash
set -e

# =============================================================================
# Install Microsoft Hypervisor (MSHV)
# - EFI boot via HvLoader.efi → lxhvloader.dll → hypervisor → Linux root partition
# - /dev/mshv available after boot through mshv_root module
#
# NOTE: Secure Boot is NOT supported at the moment.
# =============================================================================

HVLOADER_RPM="https://packages.microsoft.com/azurelinux/3.0/prod/base/x86_64/Packages/h/hvloader-1.0.1-5.azl3.x86_64.rpm"
MSHV_BOOTLOADER_RPM="https://packages.microsoft.com/azurelinux/3.0/prod/ms-non-oss/x86_64/Packages/m/mshv-bootloader-lx-26100.7436.2511151828.1-1.azl3.x86_64.rpm"
MSHV_RPM="https://packages.microsoft.com/azurelinux/3.0/prod/ms-non-oss/x86_64/Packages/m/mshv-26100.7436.2511151828.1-1.azl3.x86_64.rpm"

echo "Extracting Microsoft MSHV packages..."
WORK_DIR=$(pwd)
cd /
for rpm in "$HVLOADER_RPM" "$MSHV_BOOTLOADER_RPM" "$MSHV_RPM"; do
  curl -sL -O "$rpm"
done
for rpm in $(ls *.rpm); do
  echo "Extracting $rpm"
  rpm2cpio $rpm | cpio -idm --no-absolute-filenames
  rm $rpm
done

echo "Config modules"
echo mshv_root > /etc/modules-load.d/mshv_root.conf
echo blacklist kvm > /etc/modprobe.d/blacklist-kvm.conf
echo blacklist kvm_amd >> /etc/modprobe.d/blacklist-kvm.conf
echo blacklist kvm_intel >> /etc/modprobe.d/blacklist-kvm.conf
update-initramfs -u

ROOT_UUID=$(findmnt -no UUID /)
cat >> /etc/grub.d/40_custom <<EOF
menuentry "Microsoft Hypervisor (MSHV)" {
  search --no-floppy --set=root --file /HvLoader.efi
  chainloader /HvLoader.efi lxhvloader.dll MSHV_ROOT=\\\\Windows MSHV_ENABLE=TRUE MSHV_SCHEDULER_TYPE=ROOT MSHV_X2APIC_POLICY=ENABLE MSHV_SEV_SNP=TRUE MSHV_LOAD_OPTION=INCLUDETRACEMETADATA=1
  boot
  insmod part_gpt
  insmod ext2
  set root='hd0,gpt2'
  search --no-floppy --fs-uuid --set=root $ROOT_UUID
  linux /boot/vmlinuz root=UUID=$ROOT_UUID ro rd.auto=1 net.ifnames=0 lockdown=integrity audit=0 console=ttyS0,115200n8 earlyprintk
  initrd /boot/initrd.img
}
EOF
sed -i -e 's/GRUB_DEFAULT=0/GRUB_DEFAULT=3/' /etc/default/grub
sed -i -e 's/GRUB_TIMEOUT_STYLE=hidden/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
sed -i -e 's/GRUB_TIMEOUT=0/GRUB_TIMEOUT=30/' /etc/default/grub
update-grub
```
