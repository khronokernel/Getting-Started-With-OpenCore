# Getting-Started-With-OpenCore
A brief guide to using the OpenCore bootloader for hackintoshes

# So what is OpenCore?

OpenCore is a replacement to Clover which tries to fix the issues that plague it, specifically by being more modular and open. OpenCore is not only for Hackintoshes, but can be used for other purposes that require an emulated EFI. And please remember we’re still in very early infancy so there will be issues. I also won't be going into detail with vault.plist as there's still quite a bit of development being done there

# Current issues with OpenCore

* Z97 based systems require pure UEFI mode for booting(also known as Windows 8/10 mode)
* Currently minimal support for emulated NVRAM(sorry z390 users, EmuVariableRuntimeDxe may work but won't save on reboot)
* fakeSMC sensors can't be injected, alternative is [HWSensors3](https://github.com/warexify/HWSensors3)
* VoodooPS2Controller needs to be injected first, Keyboard second and Mouse/Trackpad third
* NVMe issues if setup as SATA device in BIOS

# Setting up OpenCore

So what you need for this:

* [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) (recommend to build from scratch instead of using the prebuilt package as OpenCore is constantly being updated)
* [AppleSupportPkg](https://github.com/acidanthera/AppleSupportPkg/releases)
* [AptioFixPkg](https://github.com/acidanthera/AptioFixPkg/releases)
* [mountEFI](https://github.com/corpnewt/MountEFI) or some form of EFI mounting. Clover Configurator works just as well
* Xcode to edit .plist files([OpenCore Configurator](https://www.insanelymac.com/forum/topic/338686-opencore-configurator/) is another tool but vit9696 has stated multiple times he does not support these tools and they even break OpenCore's specifications. Use at own risk)
* USB formatted as MacOS Journaled with GUID partition map
* Knowledge of how a hackintosh works and what files yours requires
* A working Hackintosh to test on

# Creating the USB

So it's actually quite simple to make the USB all you need to do is format it as MacOS Journaled with GUID partition map. There is no real size requirement for the USB as OpenCore's entire EFI is less than 5MB

https://i.imgur.com/5uTJbgI.png

Next we'll want to mount the EFI partition on the USB with mountEFI or Clover Configurator

https://i.imgur.com/4l1oK8i.png

And you'll notice that once we open the EFI partition, it's empty. This is where the fun begins

https://i.imgur.com/EDeZB3u.png

# Base folder structure

So lets setup OpenCore’s folder structure, you’ll want to grab those files from OpenCorePkg and construct your EFI to look like the one below:

https://i.imgur.com/VTd0OYI.png

Now you can place your necessary .efi drivers from AppleSupportPkg and AptioFixPkg into drivers and kexts/ACPI into their respective folders. Please note that UEFI drivers are not supported with OpenCore

Here's what mine looks like(ignore my odd choice of kexts):

https://i.imgur.com/ybRtoTi.png

# Setting up your config.plist

So things to keep in mind with Config.plist in OpenCore, they are different from Clover’s config.plists, they cannot be mixed and matched. And you'll also need to read the documentation what what everything does if you don't know

First let’s duplicate the sample.plist and rename the duplicate to config.plist. Now lets open it up in Xcode

&#x200B;

So you've probably noticed there's a bunch of groups:

* ACPI: this is for loading, blocking and patching the ACPI
* DeviceProperties: this is where you'd set PCI device patches like the intel Framebuffer patch
* Kernel: Where we tell OpenCore what kexts to load, what order to load and which to block
* Misc: Setting for OpenCore's boot loader itself
* NVRAM: This is where we set NVRAM properties like boot flags and SIP
* Platforminfo: This is where we setup your SMBIOS
* UEFI: Where we tell OpenCore which drivers to load and which order

We can delete #WARNING -1 and  #WARNING -2 just to clean it up a bit

# ACPI

**Add:** You'll want to go through and disable all of them or rename them to the files you have under EFI/OC/ACPI/Custom (set enabled to no or delete)

**Block**: We won't be doing anything here

**Patch**: Here I'll be adding some USB and SATA patches, follow the vanilla guide for what patches your system may need

**Quirk**: settings for ACPI

* FadtEnableReset: NO (enable reboot and shutdown on legacy hardware, not recommended unless needed)
* IgnoreForWindows: NO (Disable ACPI modifications when booting Windows, only for those who made broken ACPI tables)
* NormalizeHeaders: NO (Cleanup ACPI header fields, irrelevant in 10.14)
* RebaseRegions: NO (Attempt to heuristically relocate ACPI memory regions)
* RestLogoStatus: NO 

&#x200B;

# DeviceProperties

**Add**: Sets device properties from a map

PciRoot(0x0)/Pci(0x2,0x0) -> AAPL,ig-platform-id

* Applies Framebuffer patch, insert required value from Framebuffer guide [here](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271). Don't forget to add Stolemen and patch-enable

PciRoot(0x0)/Pci(0x1b,0x0) -> Layout-id

* Applies AppleALC audio injection, insert required value from AppleALC documentation [here](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=2ahUKEwj9t8qklJziAhVPoZ4KHb1sBPIQjBAwAnoECAUQDQ&url=https%3A%2F%2Fgithub.com%2Facidanthera%2FAppleALC%2Fwiki%2FSupported-codecs&usg=AOvVaw3QqKGaYwfJ7OkT3YIOmoPz)

**Block**: Removes device properties from map(can delete, irrelevant for most users)

# Kernel

**Add**: Here's where you specify which kexts to load, order matters here so make sure Lilu.kext is always first and other higher priority kexts come after as Lilu provides functionality to VirtualSMC, AppleALC, WhateverGreen and others

**Block**: Blocks kexts from loading, sometime needed for disabling Apple's trackpad driver for some laptops

**Patch**: Patches kexts (this is where you add USB port limit patches and AMD CPU patches)

**Quirks**:

* AppleCpuPmCfgLock: NO (only needed when CFG-Lock can't be disabled in BIOS)
* AppleXcpmCfgLock: NO (only needed when CFG-Lock can't be disabled in BIOS)
* ExternalDiskIcons: YES (External Icons Patch, for when internal drives are treated as external drives)
* ThirdPartyTrim: NO (enables TRIM, not needed for AHCI or NVMe SSDs)
* XhciPortLimit: YES (This is actually the 15 port limit patch, don't rely on it as it's not a guaranteed solution to USB. Please create a [USB map](https://usb-map.gitbook.io/project/) when possible but perfect for those who don't have a USBmap yet)

# Misc

**Boot**:settings for boot screen(leave as-is unless you know what you're doing)
* Timeout: 5 (this sets how long it'll wait till it automatically boots from the picker)

**Debug**: debug, pretty self-explanatory(leave as-is unless you know what you're doing)
* DisableWatchDog: NO (may need to be set for yes if MacOS is stalling on something while booting)

**Security**: security, pretty self-explanatory

* RequireSignature: NO (we won't be dealing vault.plist so we can ignore)
* RequireVault: NO (we won't be dealing vault.plist so we can ignore as well)

# NVRAM

**Add**: 7C436110-AB2A-4BBB-A880-FE41995C9F82 (System Integrity Protection bitmask)

* boot-args: -v dart=0 debug=0x100 keepsyms=1 , etc (boot flags)
* csr-active-config: <00000000> (settings for SIP)
* nvda\_drv:  <> (for enabling WebDrivers)
* prev-lang:kbd: <> (needed for non-latin keyboards)

**Block**: blocks NVRAM variables, not needed for us. Delete the entires there

# Platforminfo

**Automatic**: YES (generates PlatformInfo based on Generic section instead of  DataHub, NVRAM, and SMBIOS sections)

**Generic**:

* SpoofVendor: YES
* SystemUUID: can be generated with MacSerial or use pervious from Clover's config.plist
* MLB:  can be generated with MacSerial or use pervious from Clover's config.plist
* ROM: <> (automatically filled in)
* SystemProductName: can be generated with MacSerial or use pervious from Clover's config.plist
* SystemSerialNumber: can be generated with MacSerial or use pervious from Clover's config.plist

**UpdateDataHub**: YES (Update Data Hub fields)

**UpdateNVRAM**: YES (Update NVRAM fields)

**UpdateSMBIOS**: YES (Update SMBIOS fields)

**UpdateSMBIOSMode**: Create (Replace the tables with newly allocated EfiReservedMemoryType)

# UEFI

**ConnectDrivers**: YES (forces .efi drivers)

**Drivers**: add your .efi drivers here

* ApfsDriverLoader.efi (for example)

**Protocols**:

* AppleBootPolicy: NO
* ConsoleControl: NO
* DataHub: NO
* DeviceProperties: NO

**Quirks**:

* ExitBootServicesDelay: 0 (switch to 5 if running ASUS Z87-Pro with FileVault2)
* IgnoreInvalidFlexRatio: NO
* IgnoreTextInGraphics: NO
* ProvideConsoleGop: NO
* ReleaseUsbOwnership: NO
* RequestBootVarRouting: NO
* SanitiseClearScreen: NO

&#x200B;

# And now you're ready to boot!
