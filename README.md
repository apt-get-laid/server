# Setting Up the server
# Configuring for PROXMOX
## GPU Passthrough
First, some changes need to be applied in the *BIOS* and then you have to allow these changes from the Bootloader:

| BIOS Input  | GRUB Entry | Reasoning |
| ------------- | ------------- | ------------- |
| IOMMU = Enable | amd_iommu=on | Enable the IOMMU on the kernel. |
| N/A | iommu=pt | If the HW supports IOMMU, this will bypass the default DMA translation normally performed by the hypervisor and instead pass DMA requests directly to the hardware IOMMU.|
| ACS Enable = Enable | pcie_acs_override=downstream | ACS (Access Control Services) to have separate IOMMU Groups |
| N/A | video=efifb:off | Disable the generic EFI platform driver for systems with UEFI firmware |

To apply the chagnes in the bootloader, edit inside `/etc/default/` the file called `grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction video=efifb:off"
```
And then run `update-grub` and `reboot`
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
All the applications listed below are Open Source.
```
# Install 7zip [compressed files manager]
winget install -e --id 7zip.7zip --accept-source-agreements --accept-package-agreements
```
