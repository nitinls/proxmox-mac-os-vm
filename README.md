# proxmox-mac-os-vm

https://www.nicksherlock.com/2021/10/installing-macos-12-monterey-on-proxmox-7/

Installing macOS 12 “Monterey” on Proxmox 7
This tutorial for installing macOS 12 Monterey has been adapted for Proxmox from Kholia’s OSX-KVM project and Leoyzen’s OpenCore configuration for KVM. You can get the full sourcecode of my OpenCore release on my GitHub here.

Requirements
I’ll assume you already have Proxmox 7 installed. You also need a real Mac available in order to fetch the OSK key.

Your Proxmox host computer’s CPU must support SSE 4.2, so for Intel your CPU must be at least as new as Nehalem, which was the first CPU generation to bear the “Core” i5/i7 branding. Older CPUs will cause Illegal Instruction crashes when apps/extensions attempt to use these missing instructions.

Modern AMD CPUs also support SSE 4.2 and will work with this guide.

First step: Create an installation ISO
Download my copy of the OSX-KVM repository using the download button, and unzip it:

https://github.com/thenickdude/OSX-KVM

First we need to install some build requirements. If you will be building the installer ISO on macOS, open up the Terminal and run this command to install the commandline tools:

xcode-select --install
If you’re building the ISO on Linux, you instead need to run this command (these are the package names for Ubuntu or similar distributions, they may need adjustment on other distributions):

sudo apt install qemu-utils make
Now in the Terminal, from the root of OSX-KVM, run:

cd scripts/monterey
make Monterey-recovery.img
This will download the Monterey installer from Apple’s software distribution servers and build a Monterey-recovery.img file for you. Upload this file to your Proxmox’s ISO store directory (typically /var/lib/vz/template/iso). Although we’re putting it in the ISO directory so that we can use it with Proxmox’s ISO picker later, this a raw disk image rather than a true ISO.

If you’re building the installer on macOS, you can build a full installer instead of just a recovery, which will mean that macOS won’t have to download Monterey files during installation, and so won’t require an Internet connection. Simply ask it to build Monterey-full.img instead:

cd scripts/monterey
make Monterey-full.img
This option is not available when building the installer on Linux.

Prepare an OpenCore image
Download the OpenCore.iso.gz file from the newest release in my repository (you want v15 or newer), double click it to unpack it, and upload it to Proxmox’s ISO store at /var/lib/vz/template/iso. Although it has a .iso file extension, this is actually a hard disk image.

Fetch the OSK authentication key
macOS checks that it is running on real Mac hardware, and refuses to boot on third-party hardware. You can get around this by reading an authentication key out of your real Mac hardware (the OSK key). Save the first block of C code from this page as smc_read.c. In a command prompt, change into the same directory as that file and run:

xcode-select --install # If you don't already have gcc
gcc -o smc_read smc_read.c -framework IOKit
./smc_read
It’ll print out the 64 character OSK for you. Make a note of it.

Every Mac uses the same OSK, so don’t be surprised that it doesn’t look like a random string!

Create the VM
From the Proxmox web UI, create a new virtual machine as shown below.


Keep a note of your VM’s ID

Select the OpenCore ISO you uploaded and set OS type to “Other”

Set graphics to “VMWare Compatible”, set BIOS to OVMF (UEFI), set Machine to Q35, tick QEMU Agent, tick Add EFI Disk and pick storage for it. You must untick “pre-enroll keys”

Set the size of the hard disk (64GB or greater, 32GB is too small). Attach it to virtio0. Enable discard to support TRIM.

Set the number of cores for the VM, use a power of two (e.g. 1, 2, 4, 8). Set the CPU to Penryn.

I chose a memory size of 4096MB. Disable ballooning.

Choose VirtIO (paravirtualized) for the network model

In the Options page for the VM, ensure that “use tablet for pointer” is set to “Yes” (this is the default).

In the Hardware page for the VM, add a second DVD drive at IDE0, set it to use your Monterey-full.img or Monterey-recovery.img.

Don’t try to start the VM just yet. First, SSH into your Proxmox server so we can make some edits to the configuration files.

Edit /etc/pve/qemu-server/YOUR-VM-ID-HERE.conf (e.g. with nano or vim). Add this line, being sure to substitute the OSK you extracted earlier into the right place:

args: -device isa-applesmc,osk="THE-OSK-YOU-EXTRACTED-GOES-HERE" -smbios type=2 -device usb-kbd,bus=ehci.0,port=2 -global nec-usb-xhci.msi=off -global ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off
A USB keyboard is added here because macOS doesn’t support QEMU’s default PS/2 keyboard.

MSI has been disabled for the USB controller to fix QEMU panics when USB 3 devices are passed through.

ACPI PCIe hotplug support needs to be disabled for VMs with passthrough devices since QEMU Q35 machine model 6.1, which is what that final argument does. (You could instead choose Q35 6.0 for your Machine type on the Hardware tab). If you haven’t updated your QEMU to version 6.1 yet then remove this argument from the args. (Proxmox 7 ships with 6.0 and updated to 6.1 later on)

Ensure the args are all on a single line!

We also need to add a -cpu argument. If your host CPU is Intel, add this to the end of the “args” line:

-cpu host,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc
This will pass through all of the features that your CPU supports. OpenCore’s config will pretend to macOS that the CPU’s model name is Penryn for compatibility.

If your host CPU is AMD, or the above argument doesn’t work for you, use this more-compatible alternative:

-cpu Penryn,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc,+pcid,+ssse3,+sse4.2,+popcnt,+avx,+avx2,+aes,+fma,+fma4,+bmi1,+bmi2,+xsave,+xsaveopt,+rdrand,check
This pretends that your CPU is Penryn, which will keep macOS happy even if your host CPU is AMD, and adds a bunch of newer required and optional CPU features on top. Features that your host CPU doesn’t support will be ignored (a warning will be printed to the console during launch with qm start 1xx), but note that macOS won’t work without SSE4.2 support.

You can remove the “+invtsc” feature from the list if your CPU doesn’t support it, or if you want to be able to migrate a running VM between Proxmox nodes.

Now find the lines that define the two “ISOs” (ide0 and ide2), and remove the “,media=cdrom” part from them. Add “,cache=unsafe” in its place. This will treat these as hard disks rather than DVD drives.

Save your changes, return to the Options tab, and change the boot order to put IDE2 (the OpenCore image) first. Your final VM configuration file should resemble this:

args: -device isa-applesmc,osk="..." -smbios type=2 -device usb-kbd,bus=ehci.0,port=2 -cpu host,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc -global nec-usb-xhci.msi=off -global ICH9-LPC.acpi-pci-hotplug-with-bridge-support=off
agent: 1
balloon: 0
bios: ovmf
boot: order=ide2
cores: 4
cpu: Penryn
efidisk0: vms-ssd-enc:vm-171-disk-1,efitype=4m,size=1M
ide0: isos:iso/Monterey-full.img,cache=unsafe,size=14G
ide2: isos:iso/OpenCore-v15.img,cache=unsafe,size=150M
machine: q35
memory: 4096
name: macos-monterey
net0: virtio=...,bridge=vmbr0,firewall=1
numa: 1
ostype: other
virtio0: vms:vm-171-disk-0,cache=unsafe,discard=on,size=64G
scsihw: virtio-scsi-pci
smbios1: uuid=...
sockets: 1
vga: vmware
Configure Proxmox
On Proxmox, run “echo 1 > /sys/module/kvm/parameters/ignore_msrs” to avoid a bootloop during macOS boot. To make this change persist across Proxmox reboots, run:

echo "options kvm ignore_msrs=Y" >> /etc/modprobe.d/kvm.conf && update-initramfs -k all -u
Install Monterey
Now start up your VM, it should boot to the OpenCore boot picker:

Press enter to boot the “Install macOS Monterey” entry and the installer should appear. (If your keyboard isn’t working, leave the Proxmox Console page and re-enter it)


OpenCore’s “OpenCanopy” boot picker
If you built a recovery installer, the icon will instead be an image of a hard disk and be labelled “MacOS Base System”.

Our virtual hard drive needs to be erased/formatted before we can install to it, so select the Disk Utility option. Follow the steps below to format the disk:


Select Disk Utility

Erase the main disk (it’ll appear as VirtIO Block Media). Choose APFS. I called mine “Main”
Now you can quit Disk Utility from the top menu (Disk Utility > Quit Disk Utility), and we’re ready to begin installation!


Install macOS Monterey

Choose the disk you erased

It’s installing!
After the first stage of installation, the VM will reboot 2 or 3 times in quick succession, and each time you must manually pick the “macOS Installer” entry (the second one here, with the hard disk icon) to continue installation. It will not be selected for you automatically:


If your keyboard isn’t responding on this screen, exit the Console tab in Proxmox and re-enter it. If you get a “prohibited” sign like this appearing, hit the Reset button on the VM to try again:


Now the installation is nearly complete and the macOS Installer entry disappears, so pick the name of your main disk to boot (mine’s called Main).


There’s just one more reboot to come. Afterwards pick the “Main” entry again and this time you’ll finally boot into Monterey!

Answer the initial install questions, and you’ll be logged on! Note that you will want to hold off on logging into your Apple ID until you’ve configured your Mac’s serial number in OpenCore (because otherwise a Mac with the default shared serial number in my OpenCore image will be added to your Apple ID).


It works!
Note that it will be really sluggish for a few minutes after the first boot while the system performs housekeeping tasks.

Make the OpenCore install permanent
We’re currently booting using OpenCore from the attached OpenCore ISO. Let’s install that to the hard drive instead. Pop open Terminal and run “diskutil list” to see what drives we have available.


Use “sudo dd if=<source> of=<dest>” to copy the “EFI” partition from the OpenCore CD and overwrite the EFI partition on the hard disk. The OpenCore CD is the small disk (~150MB) that only has an EFI partition on it, and the main hard disk is the one with the large (>30GB) Apple_APFS “Container” partition on it.

In my case these EFI partitions ended up being called disk1s1 and disk0s1 respectively, so I ran “sudo dd if=/dev/disk1s1 of=/dev/disk0s1” (note that if you get these names wrong, you will overwrite the wrong disk and you’ll have to start the installation over again!).

Now shut down the VM, and remove both the OpenCore and the Monterey installer drives from the Hardware tab. On the Options tab, edit the boot order to place your virtio0 disk as the first disk. Boot up. If everything went well, you should see the OpenCore boot menu, and you can select your “Main” disk to boot Monterey:


Sleep management
I found that I was unable to wake Monterey from sleep using my mouse or keyboard. If you encounter the same problem, you can either disable system sleep in Monterey’s Energy Saver settings to avoid the issue, or you can manually wake the VM up from sleep from Proxmox by running:

qm monitor YOUR-VM-ID-HERE 
system_wakeup 
quit
Editing your OpenCore/EFI settings
The Configuration.pdf that explains the OpenCore config.plist file can be found along with the OpenCore release on my GitHub.

To mount your EFI partition in macOS so you can edit your config.plist, first check the “identifier” of your EFI partition in the terminal:

~$ diskutil list
/dev/disk0 (external, physical):
   #:                   TYPE NAME              SIZE       IDENTIFIER
   0:  GUID_partition_scheme                  *512.1 GB   disk0
   1:                    EFI EFI               209.7 MB   disk0s1
   2:             Apple_APFS Container disk1   511.9 GB   disk0s2
Then you can mount it like so:

sudo mkdir /Volumes/EFI
sudo mount -t msdos /dev/disk0s1 /Volumes/EFI
Now you can edit /Volumes/EFI/OC/config.plist with your favourite text editor to make your changes. (TextEdit is not a great choice because it likes to insert curly quotes into the file and otherwise break things, there are some dedicated plist editors available such as XCode).

If you’re unable to boot macOS, you can edit the config.plist using the “UEFI Shell” option in the OpenCore boot menu instead.

Enter “FS0:” and press enter to open up the first available filesystem, then run “edit EFI\OC\config.plist” (if the file isn’t found, try switching to another filesystem like fs1:). When you’re done editing, press control+Q to exit, “Y” to save, then run “exit” to return to the OpenCore menu. You need to reboot for your changes to take effect.

If you prefer, you can edit config.plist from the comfort of your Proxmox host instead. If you’re booting from an attached OpenCore.img file, you can mount that file as a disk on the host. If you’re booting from the VM’s disk instead, it must be in raw format in order to be mounted (e.g. typical LVM or ZFS usage) rather than qcow2.

# Mount an OpenCore image:
losetup --partscan /dev/loop0 /var/lib/vz/template/iso/OpenCore-v15.img
# or a VM boot disk:
losetup --partscan /dev/loop0 /dev/zvol/tank/vms/vm-100-disk-1

mount /dev/loop0p1 /mnt
Now the contents of that first partition are available in /mnt, so you can edit /mnt/EFI/OC/config.plist in your favourite editor. When you’re done, do this to unmount the disk:

umount /mnt
losetup --detach /dev/loop0
Automatic boot
In config.plist, you can set Misc/Boot/Timeout to a non-zero value to allow the default boot option be chosen automatically after that delay in seconds. I’ve disabled this by default because it causes the installer ISO to re-enter its main menu instead of continuing the second stage of installation.

You can set the default boot option by pressing control+enter on it.

Verbose boot
To boot macOS in Verbose mode to diagnose boot problems, at the OpenCore boot menu press Cmd+V before pressing enter to boot macOS (you don’t need to hold it down).

If there is a kernel panic during boot and it reboots too quickly to be read, edit config.plist to add “debug=0x100” to the kernel arguments.

Changing screen resolution
To change macOS’ screen resolution, you need to edit the UEFI/Output/Resolution setting in config.plist, the default is 1920×1080@32.

You should be able to change this to any of the modes that the system OVMF menu offers (hit F2 at the start of guest boot and choose “Device Manager/OVMF Platform Configuration” to see which resolutions are available).

Video performance
Because there is no guest video acceleration available for macOS, video performance is poor.

In Google Chrome in the guest you will need to toggle off the setting to “use hardware acceleration when available” to improve issues with elements not being drawn or flickering (especially video). Safari may be a better choice.

macOS’s built in “Screen Sharing” feature offers dramatically better framerates and latency than Proxmox’s browser-based VNC console, so if you have a real Mac to act as a viewing console, you can enable that in the VM’s “Sharing” settings and connect to the VM using the Screen Sharing app from your Mac instead:


Apparently Screen Sharing is also compatible with VNC clients like RealVNC, so you should be able to connect to it from Linux or Windows consoles using RealVNC.

The real magic bullet for video performance is to pass through a compatible video card using PCIe passthrough (though note that macOS now only supports a handful of very old NVidia cards). This offers near-native performance. You can read more about how I’m using PCIe passthrough on my own installation here.

USB passthrough
Since I want to use this as my primary computer, I want to use a USB keyboard and mouse plugged directly into Proxmox, rather than sending my input through the web VNC console.

Proxmox has good documentation for USB passthrough. Basically, run “qm monitor YOUR-VM-ID-HERE”, then “info usbhost” to get a list of the USB devices connected to Proxmox:

qm> info usbhost
 Bus 3, Addr 12, Port 6, Speed 480 Mb/s
 Class 00: USB device 8564:1000, Mass Storage Device
 Bus 3, Addr 11, Port 5.4, Speed 12 Mb/s
 Class 00: USB device 04d9:0141, USB Keyboard
 Bus 3, Addr 10, Port 5.1.2, Speed 12 Mb/s
 Class 00: USB device 046d:c52b, USB Receiver
 Bus 3, Addr 9, Port 14.4, Speed 12 Mb/s
 Class 00: USB device 046d:c227, G15 GamePanel LCD
 Bus 3, Addr 8, Port 14.1, Speed 1.5 Mb/s
 Class 00: USB device 046d:c226, G15 Gaming Keyboard
In this case I can add my keyboard and mouse to USB passthrough by quitting qm, then running:

qm set YOUR-VM-ID-HERE -usb1 host=04d9:0141
qm set YOUR-VM-ID-HERE -usb2 host=046d:c52b
This saves the devices to the VM configuration for you. You need to reboot to have the new settings apply.

You can also pass through USB devices by passing through an entire USB controller using Proxmox’s PCIe passthrough feature, which gives much better compatibility.

Odd core counts
If your host CPU is AMD, you might have a core count which is divisible by 3, like 6 cores. macOS doesn’t like it if the core count isn’t a power of 2 (1, 2, 4, 8), but it doesn’t mind odd socket counts.

So in order to pass 6 total cores, configure the VM for 3 sockets and 2 cores per socket.

Fixing “guest boots to UEFI shell”
If your guest ends up booting to the UEFI shell instead of showing the OpenCore boot menu, especially if you’ve just updated OpenCore to a new version, you’ll need to edit the guest’s UEFI boot entries to fix this.

At the very start of guest boot, hit F2 to enter guest UEFI settings.

First we’ll remove the old entries. Choose the Boot Maintenance option, then Boot Options -> Delete Boot Option. Use the spacebar to tick any old Clover or OpenCore entries (avoid ticking the EFI Internal Shell option, you want to keep that!). Select “Commit Changes and Exit”.

Now we’ll add the correct entry for OpenCore back in. Select Add Boot Option. Navigate through the device tree to EFI/OC/OpenCore.efi and select it, name this new option “OpenCore” or similar. Again Commit Changes and Exit.

Go to the Change Boot Order and move OpenCore to the top. Commit Changes and Exit.

Now back out to the main menu and choose Reset, and you should successfully boot into OpenCore this time.

Fixing iMessage
iCloud and the App Store should already be working for you, but for iMessage support you must follow these steps to mark your network adapter as built-in:

https://dortania.github.io/OpenCore-Post-Install/universal/iservices.html#fixing-en0

Disabling SIP (System Integrity Protection)
You can disable SIP by selecting the Recovery option from the OpenCore boot menu, then use the top menu to open the Terminal and run csrutil disable --no-internal. Then reboot.

This may be needed to run unsigned kexts or perform other hacks.

Upgrading OpenCore
Sometimes you need to update OpenCore to a new release in order to support a new macOS update. I’ll assume you don’t have any customisations to config.plist you want to save.

First take a snapshot! It’s great to be able to roll back if something goes wrong.

If you’re still able to boot macOS, you can update it from within the guest. Follow the instructions in the “Editing your OpenCore/EFI settings” section to mount your EFI partition. Then you can delete the EFI folder in there and replace it with the one from the OpenCoreEFIFolder.zip file from my OpenCore release (you’ll probably need to empty the trash first to make room for the new folder). You’re done!

If you aren’t able to boot macOS, unpack and upload the new OpenCore ISO to Proxmox’s ISO store instead. Add a new CD drive to the VM that uses that ISO. Then in Proxmox’s terminal edit the VM’s config (in /etc/pve/qemu-server) to replace “media=cdrom” with “cache=unsafe” for the OpenCore drive. Now on the “Options” tab, change the boot order to put the new OpenCore drive first.

Start the VM and boot into macOS using the new OpenCore drive. From within macOS you can now follow the instructions from the “Make the OpenCore install permanent” section to install the new OpenCore image to your main macOS disk, after which the OpenCore drive can be detached from the VM.

Upgrading from macOS Big Sur
First make a backup or snapshot of your system! Being able to roll back when the upgrade goes wrong is a real lifesaver.

You’ll need to update OpenCore to my v15 release before the upgrade. You can follow the instructions in the “Editing your OpenCore settings” section above to mount your EFI disk. Then you can replace the OpenCore files in the mounted “EFI” disk with the ones from the OpenCoreEFIFolder.zip file in my newest OpenCore release.

Reboot to make sure that you can still boot Big Sir.

If you’re using any PCIe passthrough devices (particularly video cards) you’ll want to disable those and set “vga: vmware” instead, so you can install using Proxmox’s web console from a different machine during the upgrade. This avoids installer problems triggered by flaky video card passthrough, especially host lockups caused by the AMD Reset Bug.

Now you can upgrade to Monterey using Software Update or from the App Store like you would on a real Mac.

If the update fails to appear, your SIP mode might be set to “Apple Internal”. You can check this by running “csrutil status” in the Terminal.

To fix this, reboot your VM and at the OpenCore menu select Recovery. Open up the Terminal, and in there run “csrutil enable --no-internal” (“csrutil disable --no-internal” is fine too), then reboot back into macOS and try again.
