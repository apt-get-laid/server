# Installing Proxmox 
## Troubleshooting
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
## Setting Up the server in PROXMOX
### GPU Passthrough
First, some changes need to be applied in the *BIOS* and then you have to allow these changes from the Bootloader:

| BIOS Input  | GRUB Entry | Reasoning |
| ------------- | ------------- | ------------- |
| AMD CBS > NBIO Common Options > IOMMU = Enable | amd_iommu=on | Enable the IOMMU on the kernel. |
| N/A | iommu=pt | If the HW supports IOMMU, this will bypass the default DMA translation normally performed by the hypervisor and instead pass DMA requests directly to the hardware IOMMU. |
| AMD CBS > ACS Enable = Enable | pcie_acs_override=downstream | ACS (Access Control Services) to have separate IOMMU Groups. |
| AMD CBS > NBIO Common Options > Enable AER Cap = Enable | " | Enables Advanced Error Reporting, needed for ACS. |
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

## Avoiding Windows and EasyAntiCheat to detect the VM
So the Machine does not know it is virtualised.
Blacklist the GPU Drivers in `etc/modprobe.d` and edit `pve-blacklist.conf`
```
blacklist nvidiafb
blacklist nvidia
blacklist radeon
blacklist nouveau
```
### Create an undetectable VM

Click on "Create VM" (top right in the GUI)
1. (General): Give your VM a name and an ID. Also select the "Advanced" option at the bottom of the window. We'll need some advanced settings later.
2. (OS): Select the ISO you uploaded and select "Microsoft Windows" as the Guest OS type. Note: We do not need any VirtIO drivers!
3. (System):
```
    Graphics card: Default
    Machine: q35
    BIOS: OVMF (UEFI)
    Add EFI Disk: yes
    SCSI Controller: LSI 53C895A
    Qemu Agent: no
    Add TPM: yes
```
4. (Disks):
```
    Bus/Device: SATA
    Cache: Write back
```
5. (CPU):
```
    Sockets: 1
    Cores: However many cores you want
    Type: host
```
6. (Memory):
```
    Memory: however much memory you want
    Ballooning Device: no
```
7. (Network):
```
   Model: Intel E1000
```
8.: Confirm your settings, create the VM, **but don't start it yet!**

Note: The reason why we choose LSI for our SCSI controller, SATA for your disks, and the Intel E1000 network card is because we don't need any virtio drivers for any of those. In my experience, as soon as you add the Qemu gest agent or add any virtio drivers, Windows knows it's a VM.

Now, we add our GPU to the VM.
Select your VM in the menu on the left, then click on "Hardware". Now click "add" and select "PCI Device". Select "Raw Device" and find your GPU in the drop down menu. Make sure you select the GPU and not the audio controller! Tick the "Primary GPU" and "PCI-Express" boxes. For the "PCI-Express" box you need to select "Advanced" at the bottom of the window. Do not select the "All Functions" checkbox!
Repeat the process for the GPUs audio device but this time don't tick the "Primary GPU" checkbox.

Do not start the VM yet! We need some additional settings.
Run dmidecode -t 0 and dmidecode -t 1. This gives you some information about your mainboard.
Navigate to your VM in the webGUI and select Options -> SMBIOS settings (type1). Enter as much information as you can find there. For me this is:

    UUID: <redacted>
    Manufacturer: Gigabyte Technology Co., Ltd.
    Product: X670 GAMING X AX
    Family: X670 MB

Then, add the following line at the top of the VM config file (/etc/pve/qemu-server/<your_vmid>.conf).

    args: -cpu host,-hypervisor,kvm=off, -smbios type=0,vendor=<vendor>,version=<version>,date=<date>

For me this looks like this:

    args: -cpu host,-hypervisor,kvm=off, -smbios type=0,vendor="American Megatrends International",version=F8,date="07/10/2023"

Note: Don't forget the quotes for strings with spaces!
Note2: For the Manufacturer, you have to drop the comma itself and what comes after it.
Finally, add the hidden=1 option to the cpu. That is, change the line

    cpu: host,hidden=1 

Now the VM can be started.

# Host Configuration
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
## Configuring the VM 
Inside `/etc/pve/qemu-server/` choose the file `<VMID>.conf`

```
nano <VMID>.conf
# Working Example
agent: 1
bios: ovmf
boot: order=virtio0;ide0;ide2;net0
cores: 16
cpu: host
efidisk0: local-btrfs:100/vm-100-disk-0.raw,efitype=4m,pre-enrolled-keys=1,size=528K
hostpci0: 0000:06:00,pcie=1
ide0: local-btrfs:iso/virtio-win.iso,media=cdrom,size=612812K
ide2: local-btrfs:iso/Windows.iso,media=cdrom,size=5529856K
machine: pc-q35-8.1
memory: 16384
meta: creation-qemu=8.1.2,ctime=1706735653
name: W11
net0: virtio=BC:24:11:3D:25:BC,bridge=vmbr0,firewall=1
numa: 0
ostype: win11
scsihw: virtio-scsi-single
smbios1: uuid=049318e0-b290-4f57-8fcb-389e30db35fa
sockets: 1
tpmstate0: local-btrfs:100/vm-100-disk-1.raw,size=4M,version=v2.0
usb0: host=11c1:9101
usb1: host=145f:02ca
usb2: host=24ae:2013
usb3: host=0781:55a8
usb4: host=0bc2:2323
usb5: host=054c:05bf
usb6: host=0bda:0852
vga: virtio
virtio0: local-btrfs:100/vm-100-disk-2.raw,iothread=1,size=256G
vmgenid: e3bdd6b3-5784-4164-beb7-a19e3b55f8e3
```
## Setting the best Remote Desktop performance
Go to Local Group Policy Editor ->github.com
Computer Configuration -> Administrative Templates -> Windows Components -> Remote Desktop Services -> Remote Desktop Session Host -> Remote Session Environment.

### Limit maximum color depth
`Disabled`
### Use Hardware graphics adapters for all Remote Desktop Services sessions
`Enabled`
### Limit maximum display resolution
`Disabled`
### Configure H264/AVC hardware encoding for Remote Desktop Connections
`Enabled`
### Configure compression for RemoteFX data
`Enabled`
Options: `RDP Compression algorithm: Optimized to use less memory`
### Configure image quality for RemoteFX Adaptative Graphics
`Enabled`
Options: `Image quality: High`
### Configure RemoteFX Adaptative Graphics
`Enabled`
Options: `RDP experience: Let the system choose the experience for the network condition`
```
## Install all the must-have applications
All the applications listed below are freeware.
```
# Install 7zip [compressed files manager]
winget install -e --id 7zip.7zip --accept-source-agreements --accept-package-agreements
# Install Steam
winget install -e --id Valve.Steam
```
