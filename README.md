# Installing Proxmox Torubleshooting
As the Graphival installation of Proxmox fails to load, a generic dummy VGA driver needs to be created. We will do this from the "Graphical Deubg Installation"
First needs to be indentified that the error is the one that we think. For that we hit `Ctrl + Alt + F2` and check that the error is *"Cannot run in framebuffer mode. Please specify busIDs for all framebuffer devices"*. After that we go back to the console with `Ctrl + Alt + F1`.
Then we get the PCI ID of our GPU:
```
# lspci | grep -i vga
```
For this tutorial, we are going to assume that the ID is *06:0:0*.
Then we create the dummy driver:
```
# cd /usr/share/X11/xorg.conf.d/
# nano driver-nvidia.conf
```
The contetnts of this file would be:
```
Section "Device"
    Identifier "Card0"
    Driver "fbdev"
    BusID "pci0:01:0:0:"
EndSection
```
Finally, we restat X:
```
# xinit -- -dpi 96 >/dev/tty2 2>&1
```
# Setting Up the server in PROXMOX
## GPU Passthrough
First, some changes need to be applied in the *BIOS* and then you have to allow these changes from the Bootloader:

| BIOS Input  | GRUB Entry | Reasoning |
| ------------- | ------------- | ------------- |
| AMD CBS > NBIO Common Options > IOMMU = Enable | amd_iommu=on | Enable the IOMMU on the kernel. |
| N/A | iommu=pt | If the HW supports IOMMU, this will bypass the default DMA translation normally performed by the hypervisor and instead pass DMA requests directly to the hardware IOMMU. |
| AMD CBS > ACS Enable = Enable | pcie_acs_override=downstream | ACS (Access Control Services) to have separate IOMMU Groups. |
| AMD CBS > CPU Common Options > Enable AER Cap = Enable | " | Enables Advanced Error Reporting, needed for ACS. |
| N/A | video=efifb:off | Disable the generic EFI platform driver for systems with UEFI firmware. |
| AMD CBS > CPU Common Options > Local APIC Mode = x2APIC | helps operating systems run more efficiently on high core count configurations and optimizes interrupt distribution in virtualized environments. |
DMAr Support = Enabled

To apply the chagnes in the bootloader, edit inside `/etc/default/` the file called `grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction video=efifb:off"
```
And then run `update-grub` and `reboot`
After rebooting check that the ouptut of:
```
# dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
```
verifies that the config is set properly.
Edit within `/etc/default/` the file called `modules`:
```
vfio
vfio_iommu_type1
vfio_pci
```
Then run:
```
$update-initramfs -u -k all
$lsmod | grep vfio
```
and verify that the modules have been successfully loaded.
Finally edit in `/etc/pve/qemu-server/` your VM `.conf` file:
```
cpu: host,hidden=1,flags=+pcid
```
so the Machine does not know it is virtualised.
Blacklist the GPU Drivers in `etc/modprobe.d` and edit `pve-blacklist.conf`
```
blacklist nvidiafb
blacklist nvidia
blacklist radeon
blacklist nouveau
```

### Windows Package Manager (winget)
In the Powershell, insert the following code:
```
# Get latest download url
$URL = "https://api.github.com/repos/microsoft/winget-cli/releases/latest"
$URL = (Invoke-WebRequest -Uri $URL).Content | ConvertFrom-Json |
        Select-Object -ExpandProperty "assets" |
        Where-Object "browser_download_url" -Match '.msixbundle' |
        Select-Object -ExpandProperty "browser_download_url"

# Download
Invoke-WebRequest -Uri $URL -OutFile "Setup.msix" -UseBasicParsing

# Install
Add-AppxPackage -Path "Setup.msix"

# Delete installation file
Remove-Item "Setup.msix"
```
## Install all the must-have applications
All the applications listed below are freeware.
```
# Install 7zip [compressed files manager]
winget install -e --id 7zip.7zip --accept-source-agreements --accept-package-agreements
# Install Steam
winget install -e --id Valve.Steam
```
