# How to Install Windows 10 Legacy on a 2009-2011 iMac

Apple has released EFI based support for **BootCamp** for 2012+ machines. While it is possible to install EFI based OS applications in pre 2011 machines, these are often met with hardware level incompatibilites due to the early nature of EFI implementation in these machines. Here I present a way to install Windows 10 in LEGACY MODE with full compatibilty and reliablility of all native hardware and upgraded video cards. 

## Compatibility list:
* iMac mid-2011 (iMac12,2 & iMac12,1)
* iMac mid-2010 (iMac11,3 & iMac11,2)
* iMac late-2009 (iMac11,1)

> **IMPORTANT:** Unplug all external hard drives to avoid accidental erasure. It will be important to keep track of Drives and Partitions throughout this process. Please note this process involves partition rewrites to the main disk and could potentially cause corruption; Please make a backup. I take no responsibility for accidental erasure. 

# Requirements

* A 2009-2011 iMac running MacOS High Sierra and above (as my base system, I am starting with Catalina 10.5.7 - Dosdude1 patched)
* A Metal-GPU capable video card, Nvidia or AMD will work (Native or Upgraded) 
* An 8GB or larger USB drive (only for the BootCamp support software you will need to download)
* An ISO file containing Windows 10. I used [Windows 10 Home Edition](https://archive.org/details/win-10-1803-english-x-64)
* An functional OEM 8X DL "SuperDrive" slot-loading DVD reader in your iMac 2011.  
* 45GB+ free disk space for Windows OS.

## Step 1: Disable MacOS System Integrity Protection (SIP)

High Sierra and Catalina ship with System Integrity Protection (SIP), also known as "rootless" mode. 

1. Check to see if you have it enabled: ```csrtuil status```. If already disabled skip Step 1.
1. Restart your Mac.
1. Hold down Command ⌘-R and keep it held down until you see an Apple logo & progress bar. This boots you into Recovery mode.
1. From the **Utilities** menu, select **Terminal**.
1. At the prompt type exactly the following and then press Return: ```csrutil disable```
1. Terminal should display a message that SIP was disabled.
1. From the top menu, select **Restart**.
1. For Catalina, you will need [Hackintool V3.05](https://github.com/headkaze/Hackintool), go to **tools** menu and select the 'house' icon at the bottom to 'Disable Gatekeeper' and mount the disk in read/write mode.
1. To re-enable SIP protection, repeat steps above but instead type: ```csrutil enable```

## Step 2: Download the Bootcamp Windows Support Software

This download will contain the Windows drivers you will need to run Windows on your Mac. Allow the computer to do this for you via the **Boot Camp Assistant application**:

1. Open **Boot Camp Assistant** from Applications -> Utilities
1. Click continue at the introduction, you may see 2 or 3 options: 
   "Create a Windows 7 or later version install disk"
   "Download the latest Windows support software from Apple"
   "Install Windows 7 or later version"
1. Unclick these options; We will not use them  
1. Instead, from the **Actions** menu select **Download Windows Support Software**
1. Select your Macs *Desktop* or *Downloads* folder as the destination for the download
1. Press *Quit* once you are done. The download is 1.35GB, drag it to a USB drive to use later when we've booted into the newly installed Win10 Desktop

## Step 3: Obtain a bootable Windows 10 Legacy DVD .iso

Premable:
The .iso needs to be <4.7GB in total size to fit on the DVD. The .iso images have grown over the years as they incorporate more and more updates. Furthermore, the **install.wim** file must be <4Gb to fit in a FAT partition for copying.
In another segment I will show you how to reduce the size of the install.wim to only carry the actual version of Windows that you want and therefore adhere to the <4Gb file limitation size of FAT. 

1. I used [Win10_1803_English_x64.iso](https://archive.org/details/win-10-1803-english-x-64). The download is slow, but it will complete successfully.
1. This .iso represents the absolute latest one that can perfectly fit on a 4.7GB DVD-R disc without a hastle. 
1. In MacOS, you can right click the .iso file and choose "Burn to Disc..." 

## Step 4: Create the Win10 MS-DOS FAT partition (in MacOS)

1. Launch **Disk Utility**; On the **Internal drive**, under the 'View' drop down, make sure 'Show All Devices' is selected. 
1. Click the **Partition** button, and then the **"+"** sign, adjust the size of the partition you want to dedicate for Windows.
1. Select **MS-DOS (FAT)** for the Format and name it "win10", click **Apply**. It will format it under the GUID scheme.
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/diskutility-FAT-win10.png)
1. Now you have a Windows FAT/GUID partition.
1. Note: you may get a final partition name of "10" instead of "win10", this is a known bug of Disk Utility, ignore it, the name won't matter.

## Step 5: Create the Hybrid MBR (in MacOS)

Premable:
A conventional GPT disk contains a protective MBR with a single partition, of type 0xEE (EFI GPT). This can span the entire size of the disk. Our Legacy Windows 10 will not install on a GPT formated partition. In operating systems that support GPT-based boot through BIOS services rather than EFI, the first sector may also still be used to store the first stage of the bootloader code, but modified to recognize GPT partitions. 
This is a little dangerous because Hybrid MBRs are not part of Intel's EFI GPT standard. In fact, beginning with Windows 8, Microsoft's OS installs on most Intel-based Macs in EFI mode, so a hybrid MBR is not required. Unfortunately if you try to install Windows 10 in EFI mode in a 2011 iMac, you will face random reboots, Cirrus Logic Audio not working (internal speakers) and possibly video problems involving igdkmd64.sys (Intel HD3000 internal GPU incompatibilities). As I've stated, these issues were sorted out with the introduction of proper efi drivers starting in 2012. So for us, Windows 10 will never have full Protected MBR status like a normal PC; it will have to exist in a hybrid MBR environment.

If this step is not completed you will get this error when installing Windows 10:
> Windows cannot be installed to this disk. The selected disk is not of the GPT partition style

Let's begin:
* download [gptfdisk](https://sourceforge.net/projects/gptfdisk/) and install it.
* Open Terminal
* ```sudo gpt -r -vv show disk0```
* This will show you the partitions in disk0, notice how it uses a protected MBR (PMBR) at sector 0.
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/GPT-r-vv_show_disk0.png)
* From this command, we need to verify which index our FAT partition is located in. Do not use Disk Utility for this, it could be misleading.
* Look for ```GPT part - EBD0A0A2-B9E5-4433-87C0-68B6B72699C7```, this is the GUID pointing to the FAT Windows partition you created in Step 4.
* Make note of the **index** that it is connected to. You will need that in the next steps.
* ```sudo gdisk /dev/disk0```<br/>
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/sudo_gdisk_dev_disk0_protective.png)
* type ```r```  
* type ```h```  
* type ```3``` (or whatever index your FAT partition was created it)  
* "Place EFI GPT (0xEE) partition first in MBR (good for GRUB)?" type ```y```  
* "Enter an MBR hex code (default 07):" ```return```  
* "set bootable flag? (Y/N)" type ```y```  
* "do not protect more partitions?" type ```n```  
* type ```o``` ; This will view your new assignments  
* type ```w``` ; This is write or comit our changes 
* proceed? type ```y```<br/>
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/sudo_gdisk_dev_disk0_proceed.png)

This gets us to a Hybrid MBR (see below) which Windows 10 will use to format over to NTFS, and finally install windows 10 in.<br/>
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/sudo_gdisk_dev_disk0_hybrid.png)

## Step 6: Install Windows 10 Legacy using the DVD ROM Drive:

1. Reboot your Mac with the bootable Windows 10 Installer DVD inserted in the SuperDrive.
1. Press and hold down the **Option (⌥ alt)** until you see the boot selection options.
1. You should see an option with 2 DVD icons, one is a **Windows** and the other is **EFI Boot**, choose the "Windows" one.
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/mac-boot-options.jpg)
1. The Windows 10 installation will now start, follow the steps, selecting **Custom Installation**.
1. "Press any key to boot from CD or DVD....." make sure to press a key!
1. At the *"Windows Setup"* screen select **Next**, then **Install now**
1. Put in your product code or select *"I don't have a product key"*
1. Select the version of Windows you want to install, I selected *"Windows 10 Home"*
1. Select **Custom: Install Windows only (advanced)**
1. On the screen where you select your partition be careful, and ensure you select the "win10" partition you created earlier before proceeding with installation.
1. You will see an exclamation point warning that "**! Windows can't be installed on drive 0 partition 2. (Show details)**". This is expected because remember, so far, you only have a FAT-32 partition there.
1. Select **Format** (The installer will format it to NTFS); The installation should start
1. The error message is now cleared and the **Next** button will be lit up. Press it.
1. Windows will restart 4 or so times during installation. Be ready to hold down the **Option (⌥ alt)** key after each reboot, but instead of selecting the DVD device named "Windows" select the newly created hard disk labelled "Windows" instead to ensure the installation continues 
1. This time, ignore: "Press any key to boot from CD or DVD....." since you want to continue the installation from the hard disk now.
1. Finish installing Windows until you get to the desktop.

## Step 7: Install the Bootcamp Windows Support Software (in Bootcamp Windows 10)

These drivers are installed as part of the Bootcamp Windows Support Software and will allow Windows 10 to work with the Mac specific devices: WiFi, Graphics, External Monitors, Webcam, Bluetooth and Audio. 
The Bootcamp Windows Support Software should be on the Windows 10 installer USB you created earlier.

1. Insert your USB drive 
1. Open **WindowsSupport** -> **Bootcamp** -> **setup.exe**.
1. This will install all the required drivers and the bootcamp utility for Windows.
1. Reboot the system, and get back into Windows.
1. This is a good time to go into **Windows Update** and allow multiple security and feature installs to bring you up to Version 21H2
1. You may wish to allow 'other device updates' in Win10, as this will ensure you are using the latest video drivers as well

## Step 8: 'Regedit' patches to allow for brightness control (Upgraded Nvidia _Metal_ video cards)

1. run the Registry Editor, **regedit**
1. navigate to: ```HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Video\{**UUID**}\0002```
1. The ```UUID``` identifier may be different for your video card. Check each entry for the ```HardwareInformation.AdapterString``` that matches your hardware.
1. The entries will be clustered in groups of 4 numbers, eg: 0000, 0001, 0002, 0003, 0004. 
1. Right click, Add a **REG_DWORD** labelled ```EnableBrightnessControl``` and set it to ```1```
1. Right click, Add a **REG_DWORD** labelled ```RMBrightnessControlFlags``` and set it to ```400``` in Hex, or ```1024``` in Decimal.
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/K4100M_brightness_control_registry_entries_win10Legacy.png)
3. Reboot, you should now have a slider for brightness control back.
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/K4100M_brightness_control_win10Legacy.PNG)

## Step 9: Verify "Legacy" Installation
1. run **msinfo32** via the Windows panel, you should see verification of your install:
![image](https://github.com/nikey22/imac-win10-legacy-install/blob/main/images/msinfo32_bootmode_win10Legacy.PNG)

## Step 10: Re-Enable SIP protection
see Step 1, type: ```csrutil enable```, reboot

# Issues

## Separate SSD for Windows 10 
If you want to install Windows 10 on separate SSD, it can be done. The SSD needs to be part of the SATA chain and not a USB device. 
You will need to create an MS-DOS FAT, GUID formatting as we did with our partitioned drive; You will need to also transform the PMBR to Hybrid MBR.
The rest of the procedure is similar.

## Opencore (OCLP):
Opencore does not yet allow for MBR or Hybrid-MBR bootup recognition as far as I know. You will need to subload the REFInd manager from within Opencore to finally see the Windows icon and load it. I use an SD card with Opencore on it to achieve this.


# Sources & Acknowledgements

* [Oznu, How to install windows 10 EFI on imac 2011](https://gist.github.com/oznu/8796d08d73315483c3b26e79a8e3d350)
* [Dual Booting on an iMac 27" Mid 2011](https://gist.githubusercontent.com/balupton/1603bb4b7769d1af0fd7/raw/README.md)
* [Repair Bootcamp partition boot entry missing after disk resize](https://www.amirootyet.com/post/repair-bootcamp-partition-boot-entry/)
