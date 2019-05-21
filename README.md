# Getting Started With OpenCore
A brief guide to using the OpenCore bootloader for hackintoshes

# So what is OpenCore?

OpenCore is a replacement for Clover. By design, OpenCore is versatile by being more modular and open as it aims to resolve the constraints and issues that Clover brings. It is not only for Hackintoshes as it can be used for other purposes that require an emulated EFI. Please remember we’re still in very early infancy so there will be issues. This specifc guide will be omiting Vault.plist and Vault.sig as there's still quite a bit of development happening there. OpenCore should be considered in Aplha stage at this time. If you have a working, stable system you should not migrate unless you prefer "bleeding edge" development, want to contribute, and don't mind recovering your system should it fail to boot.

# Current issues with OpenCore

* Z97 based systems require pure UEFI mode for booting (also known as Windows 8/10 mode).
* Currently minimal support for emulated NVRAM (sorry z390 users, EmuVariableRuntimeDxe may work but won't save on reboot).
* FakeSMC sensors can't be injected, alternative is [HWSensors3](https://github.com/warexify/HWSensors3).
* VoodooPS2Controller needs to be injected first, Keyboard second and Mouse/Trackpad third.
* NVMe issues if setup as a SATA device in BIOS.

# Setting up OpenCore

Requirements:

* [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) (Recommend to build from scratch instead of using the prebuilt package as OpenCore is constantly being updated. As of writing we're on Version 0.0.3 even though the current offical release is 0.0.1)
* [AppleSupportPkg](https://github.com/acidanthera/AppleSupportPkg/releases)
* [AptioFixPkg](https://github.com/acidanthera/AptioFixPkg/releases)
* [mountEFI](https://github.com/corpnewt/MountEFI) or some form of EFI mounting. Clover Configurator works just as well
* Xcode to edit .plist files ([OpenCore Configurator](https://www.insanelymac.com/forum/topic/338686-opencore-configurator/) is another tool, but vit9696 has stated multiple times he does not support these tools and they even break OpenCore's specifications. Use at own risk!)
* USB formatted as MacOS Journaled with GUID partition map.
* Knowledge of how a hackintosh works and what files yours requires.
* A working Hackintosh to test on.

# Creating the USB

Creating the USB is simple. All you need to do is format it as MacOS Journaled with GUID partition map. There is no real size requirement for the USB as OpenCore's entire EFI is less than 5MB.

![Formatting the USB](https://i.imgur.com/5uTJbgI.png)

Next we'll want to mount the EFI partition on the USB with mountEFI or Clover Configurator.

![mountEFI](https://i.imgur.com/4l1oK8i.png)

You'll notice that once we open the EFI partition, it's empty. This is where the fun begins.

![Empty EFI partition](https://i.imgur.com/EDeZB3u.png)

# Base folder structure

To setup OpenCore’s folder structure, you’ll want to grab those files from OpenCorePkg and construct your EFI to look like the one below:

![base EFI folder](https://i.imgur.com/VTd0OYI.png)

Now you can place your necessary .efi drivers from AppleSupportPkg and AptioFixPkg into the *drivers* folder and kexts/ACPI into their respective folders. Please note that UEFI drivers are not supported with OpenCore!

Here's what mine looks like (ignore my odd choice of kexts):

![Populated EFI folder](https://i.imgur.com/ybRtoTi.png)

# Setting up your config.plist

Keep in mind with Config.plist in OpenCore, they are different from Clover’s config.plists. They cannot be mixed and matched. And you'll also need to read the documentation in regards to what everything does if you don't know.

First let’s duplicate the `sample.plist`, rename the duplicate to `config.plist` and open in Xcode.

![Base Config.plist](https://i.imgur.com/MklVb2Z.png)

At this point you've noticed there are a bunch of groups:

* ACPI: This is for loading, blocking and patching the ACPI.
* DeviceProperties: This is where you'd set PCI device patches like the Intel Framebuffer patch.
* Kernel: Where we tell OpenCore what kexts to load, what order to load and which to block.
* Misc: Setting for OpenCore's boot loader itself.
* NVRAM: This is where we set NVRAM properties like boot flags and SIP.
* Platforminfo: This is where we setup your SMBIOS.
* UEFI: Where we tell OpenCore which drivers to load and in which order.

We can delete *#WARNING -1* and  *#WARNING -2* just to clean it up a bit.

# ACPI

**Add:** You'll want to go through and disable all of them or rename them to the files you have under EFI/OC/ACPI/Custom (set enabled to no or delete).

**Block**: We won't be doing anything here.

**Patch**: Here you'll be adding some USB and SATA patches, follow the vanilla guide for what patches your system may need.

**Quirk**: Settings for ACPI.

* FadtEnableReset: NO (enable reboot and shutdown on legacy hardware, not recommended unless needed)
* IgnoreForWindows: NO (Disable ACPI modifications when booting Windows, only for those who made broken ACPI tables)
* NormalizeHeaders: NO (Cleanup ACPI header fields, irrelevant in 10.14)
* RebaseRegions: NO (Attempt to heuristically relocate ACPI memory regions)
* ResetLogoStatus: NO (Workaround for systems running BGRT tables)

![ACPI](https://i.imgur.com/IDZZoFc.png)

&#x200B;

# DeviceProperties

**Add**: Sets device properties from a map.

`PciRoot(0x0)/Pci(0x2,0x0)` -> `AAPL,ig-platform-id`

* Appies Framebuffer patch, insert required value from Framebuffer guide [here](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271). Don't forget to add Stolemen and patch-enable.

`PciRoot(0x0)/Pci(0x1b,0x0)` -> `Layout-id`

* Applies AppleALC audio injection, insert required value from AppleALC documentation [here](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=2ahUKEwj9t8qklJziAhVPoZ4KHb1sBPIQjBAwAnoECAUQDQ&url=https%3A%2F%2Fgithub.com%2Facidanthera%2FAppleALC%2Fwiki%2FSupported-codecs&usg=AOvVaw3QqKGaYwfJ7OkT3YIOmoPz).

**Block**: Removes device properties from map (can delete, irrelevant for most users).

![DeviceProperties](https://i.imgur.com/8gujqhJ.png)

# Kernel

**Add**: Here's where you specify which kexts to load, order matters here so make sure Lilu.kext is always first! Other higher priority kexts come after as Lilu such as, VirtualSMC, AppleALC, WhateverGreen, etc.

**Block**: Blocks kexts from loading. Sometimes needed for disabling Apple's trackpad driver for some laptops.

**Patch**: Patches kexts (this is where you add USB port limit patches and AMD CPU patches).

**Quirks**:

* AppleCpuPmCfgLock: NO (Only needed when CFG-Lock can't be disabled in BIOS)
* AppleXcpmCfgLock: NO (Only needed when CFG-Lock can't be disabled in BIOS)
* ExternalDiskIcons: YES (External Icons Patch, for when internal drives are treated as external drives)
* ThirdPartyTrim: NO (Enables TRIM, not needed for AHCI or NVMe SSDs)
* XhciPortLimit: YES (This is actually the 15 port limit patch, don't rely on it as it's not a guaranteed solution to USB. Please create a [USB map](https://usb-map.gitbook.io/project/) when possible. It's intended use is for those that do not have a USB map.)

![Kernel](https://i.imgur.com/TbkQvwb.png)

# Misc

**Boot**: Settings for boot screen (leave as-is unless you know what you're doing).
* Timeout: 5 (This sets how long OpenCore will wait until it automatically boots from the default selection).

**Debug**: Debug has special use cases, leave as-is unless you know what you're doing.
* DisableWatchDog: NO (May need to be set for yes if MacOS is stalling on something while booting)

**Security**: Security is pretty self-explanatory.

* RequireSignature: NO (We won't be dealing vault.plist so we can ignore)
* RequireVault: NO (We won't be dealing vault.plist so we can ignore as well)

![Misc](https://i.imgur.com/Y2AbXMY.png)

# NVRAM

**Add**: 7C436110-AB2A-4BBB-A880-FE41995C9F82 (System Integrity Protection bitmask)

* boot-args: -v dart=0 debug=0x100 keepsyms=1 , etc (Boot flags)
* csr-active-config: <00000000> (Settings for SIP)
* nvda\_drv:  <> (For enabling WebDrivers)
* prev-lang:kbd: <> (Needed for non-latin keyboards)

**Block**: Blocks NVRAM variables, not needed for us. Delete the entires there.

![NVRAM](https://i.imgur.com/F63KIYS.png)

# Platforminfo

**Automatic**: YES (Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections)

**Generic**:

* SpoofVendor: YES
* SystemUUID: Can be generated with MacSerial or use pervious from Clover's config.plist.
* MLB: Can be generated with MacSerial or use pervious from Clover's config.plist.
* ROM: <> (Automatically filled in)
* SystemProductName: Can be generated with MacSerial or use pervious from Clover's config.plist.
* SystemSerialNumber: Can be generated with MacSerial or use pervious from Clover's config.plist.

**UpdateDataHub**: YES (Update Data Hub fields)

**UpdateNVRAM**: YES (Update NVRAM fields)

**UpdateSMBIOS**: YES (Update SMBIOS fields)

**UpdateSMBIOSMode**: Create (Replace the tables with newly allocated EfiReservedMemoryType)

![PlatformInfo](https://i.imgur.com/dIKAlhj.png)

# UEFI

**ConnectDrivers**: YES (Forces .efi drivers, change to NO for faster boot times)

**Drivers**: Add your .efi drivers here.

**Protocols**:

* AppleBootPolicy: NO (Ensures APFS compatibility on VMs or legacy Macs)
* ConsoleControl: NO (Replaces Console Control protocol with a builtin version, needed for when firmware doens't support text output mode)
* DataHub: NO (Reinstalls Data Hub)
* DeviceProperties: NO (Ensures full compatibility on VMs or legacy Macs)

**Quirks**:

* ExitBootServicesDelay: 0 (Switch to 5 if running ASUS Z87-Pro with FileVault2)
* IgnoreInvalidFlexRatio: NO (Fix for when MSR_FLEX_RATIO (0x194) can't be disabled in the BIOS)
* IgnoreTextInGraphics: NO (Fix for UI corruption when both text and graphics outputs happen)
* ProvideConsoleGop: NO (Enables GOP, AptioMemeoryFix already has this)
* ReleaseUsbOwnership: NO (Releases USB controller from firmware driver)
* RequestBootVarRouting: NO (Redirects AptioMemeoryFix from EFI_GLOBAL_VARIABLE_G to OC_VENDOR_VARIABLE_GUID. Needed for when firmware tries to delete boot entiries)
* SanitiseClearScreen: NO (Fixes High resoltuions displays that display OpenCore in 1024x768)

![UEFI](https://i.imgur.com/acZ1PUA.png)

# What your EFI should now look like:

![Finished EFI](https://i.imgur.com/pJlP6C3.png)

# And now you're ready to boot!

![AboutThisMac](https://i.imgur.com/MFq1qGr.png)
