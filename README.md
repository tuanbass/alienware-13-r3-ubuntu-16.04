# alienware-13-r3-ubuntu-16.04

This is guide where I have dumped my personal steps to getting Ubuntu 16.04 running smoothly with my Alienware 13 R3 (Early 2017 with Kaby Lake) with OLED.

Please be aware that your mileage my vary. I am definitely no Linux/Ubuntu expert. If you have any feedback, please raise an issue!

**I take no responsibility for any damage or problems that arise as a result from following this guide.**

# READ THIS FIRST
Running Linux on a "hybrid" GPU (e.g. Nvidia Optimus) is a little trickier than without. It's important that you understand the technology and what trade-offs you want to make by using Linux.

I highly recommend that you read the following before continuing:

* [Dell's Knowledge Base article on Nvidia Optimus and Ubuntu](http://www.dell.com/support/Article/au/en/aubsdt1/SLN298431/EN)
* [A great summary of Nvidia Optimus (from one of the bumblebee devs)](http://askubuntu.com/questions/766745/do-i-need-to-install-bumblebee-for-hybrid-graphics-system-to-enable-optimus-on-u/775161)

This guide will result in **primarily using the integrated GPU (intel)** with the option of invoking the discrete (NVIDIA GTX 1060) card via bumblebee.

My reasoning was:
* get the most out of the 76 watt-hour-battery by using the integrated GPU as much as possible
* able to be explicit about when I want to use the discrete GPU (e.g. When I'm using Blender)
* primarily do my gaming in Windows
* currently the most stable configuration that I've been able to achieve

Please be aware that running permanently on the discrete GPU via `sudo prime-select nvidia` **will not work** if you follow the steps in this guide.

# Preparation

## BIOS
Turn off `Secure Boot` in the BIOS. This will make it easier to install several 3rd party drivers.

I recommend leaving `UEFI` on. Be aware that to get entries in your bios, you will need to manually add a new entry and select the appropriate *.efi file.

I believe that you need to switch the SATA interface from `RAID` to `AHCI` before you're able to install Ubuntu.
(I couldn't get the Ubuntu installer to detect the drive when using RAID but maybe this could be resolved with extra drivers). 

**WARNING: If Windows was installed with `RAID`, switching the SATA interface to `AHCI` will cause Windows to BSOD.**

Be very wary of some guides that give instruction on to continue using Windows even after switching to `ACHI`. You can very easily stop Windows booting properly. Even with recovery points, don't expect this to be easy to undo. I highly recommend that you backup any important data.

[trunet](https://github.com/trunet) has advised me that [this solution works](https://gist.github.com/Sjeanpierre/81ce440458b6c46c5608).


## Dual-boot with Windows
It is recommended that you have Windows 10 installed first.
If you need to reinstall, **you shouldn't need a product key** as it should be embedded in the BIOS.
If necessary, [shrink the partition](https://www.google.com.au/search?q=shrink+partition+windows+10) to make room for Ubuntu.
   
Installing Windows 10 is easy. Simply create a bootable USB using the Windows 10 Media Creation Tool. Get the [drivers from Dell's website](http://www.dell.com/support/home/us/en/4/product-support/product/alienware-13/drivers) and get it all working. 

## Ubuntu installer
Creating a bootable USB Ubuntu installer can be done using Rufus, using [these instructions](https://www.ubuntu.com/download/desktop/create-a-usb-stick-on-windows).

## Wifi not working after suspending in Ubuntu
I had an intermittent issue where Wifi wouldn't always come back after suspending. I'm yet to resolve this, but running `sudo service network-manager restart` resolves the problem.
# Instructions

## Ubuntu installer
Install Ubuntu 16.04 from the bootable USB.

## Disable the screensaver
Go into `System Settings...`. In `Brightness & Lock`, change `Turn off screen when inactive for:` to `Never`.

Otherwise your screen may get stuck on a black screen (but the system is still responsive).

## Fix the DPI
Go into `System Settings...` again. In `Screen Display`, change `Scale for menu and title bars:` to `1.5`.
 
## Install Nvidia 367

````
sudo apt purge 'nvidia.*'
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-367 nvidia-prime
````

~~I can't confirm if a newer version works, but 367 worked for me.~~

Edit: I'm actually using the 378.09 drivers now, but upgraded after following these steps.

## Indicate that you want to primarily use the integrated graphics card

````
sudo prime-select intel
````

## Do a general update/upgrade

````
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
````

## Intel Graphics Update Tool
Install v2.0.2 and run the [Intel Graphics Update Tool](https://01.org/linuxgraphics/downloads/update-tool).

## Reboot and verify that you can login correctly

Verify that you're using the integrated GPU. Run the following and confirm it returns `intel`:

````
prime-select query
````

## Modify grub defaults
We're going to fix 2 things:

* a bug which causes `lspci` or `lshw` to hang
* disable gpu-manager, which crashes resuming after suspending when using the integrated GPU

````
sudo vi /etc/default/grub
````

Modify `GRUB_CMD_LINUX_DEFAULT` and add options `nogpumanager` and `acpi_osi=! acpi_osi="Windows 2009"`.

For example, mine looks like this now:

````
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nogpumanager acpi_osi=! acpi_osi=\"Windows 2009\""
````

Regenerate/update:

````
sudo update-grub
````

## Reboot
After rebooting, check

* that you can correctly suspend/resume.
* running `lspci` or `lshw` doesn't crash

## Install Bumblebee
These steps here were taken from [this guide.](http://www.webupd8.org/2016/08/how-to-install-and-configure-bumblebee.html).

### Add the Bumblebee testing repository and install bumblebee:

````
sudo add-apt-repository ppa:bumblebee/testing
sudo apt update
sudo apt install bumblebee
````

### Blacklist nvidia-367:

````
sudo vi /etc/modprobe.d/bumblebee.conf
````

### Insert the following lines:

````
# 367
blacklist nvidia-367
blacklist nvidia-367-updates
blacklist nvidia-experimental-367
````

## Configure Bumblebee:

````
sudo vi /etc/bumblebee/bumblebee.conf
````

Change to the following:

* Driver=nvidia
* KernelDriver=nvidia-367
* LibraryPath=/usr/lib/nvidia-367:/usr/lib32/nvidia-367
* XorgModulePath=/usr/lib/nvidia-367/xorg,/usr/lib/xorg/modules

## Reboot
Reboot and afterwards confirm the following returns `OFF`:

````
cat /proc/acpi/bbswitch
````

## optirun & nvidia-settings

You're able to configure the nvidia settings using the below command:

````
optirun -b none /usr/bin/nvidia-settings  -c :8
````

Whilst this is running `cat /proc/acpi/bbswitch` should now return `ON`.

## Install Nvidia Power Indicator
This is a great little indicator that helps you know if the discrete GPU is being used.

````
sudo add-apt-repository ppa:nilarimogard/webupd8
sudo apt update
sudo apt install nvidia-power-indicator
````

# Other tips

## Fix lag when resizing windows
Follow [this guide](http://www.howtogeek.com/101006/how-to-tweak-unity-on-ubuntu-with-the-compizconfig-settings-manager/). Personally, instead of `outline`, I selected `rectangle`.

## Fix flickering in Chrome
When the window is maximized and you scroll, you can sometimes see flicker.

The solution was to modify `/usr/share/applications/google-chrome.desktop` and update to the following:

````
Exec=/usr/bin/google-chrome-stable --disable-gpu-driver-bug-workarounds --enable-native-gpu-memory-buffers %U
````
## Adjusting brightness
There's some great scripts on a [reddit thread](https://www.reddit.com/r/Alienware/comments/5l57hu/aw13_r3_oled_ubuntu_everything_good_but_the/dc7wcvd/) which can continously monitor and restore brightness.

Personally, I just use a script that executes the following:

````
xrandr --output eDP1 --brightness 0.35
````

Note: Try the script a few times if it doesn't work at first.

# Unresolved

### "Turn screen off when inactive for" option not reliably resuming

I'm intermittently ending up with a permanent black screen when trying to resume using the computer.

I initially thought this was causing the computer to freeze/crash, but ssh sessions from another machine were still responsive.

The following script (even run via an external ssh session) seems to fix the problem:

````
xrandr -d :0.0 --output eDP1 --off && xrandr -d :0.0 --output eDP1 --auto
````

Note: Try the script a few times if it doesn't work at first.

### Tearing while watching videos.

This [solution](http://askubuntu.com/questions/667466/screen-tearing-in-ubuntu-with-nvidia-intel-graphics) seemed to work for me.

Create `/usr/share/X11/xorg.conf.d/20-intel.conf` and add the following contents:

````
Section "Device"
   Identifier  "Intel Graphics"
   Driver      "intel"
   Option      "TearFree"    "true"
EndSection
````

Log out and back in to see changes.
 
 
### Wifi adapter is not recognized. 
````
sudo apt-get install git build-essential
git clone https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/backport-iwlwifi.git/
cd backport-iwlwifi
make defconfig-iwlwifi-public
sed -i 's/CPTCFG_IWLMVM_VENDOR_CMDS=y/# CPTCFG_IWLMVM_VENDOR_CMDS is not set/' .config
make -j4
sudo make install
````
Restart, disable securedboot 

````
cd backport-iwlwifi
make clean
make defconfig-iwlwifi-public
make -j4
sudo make install
````
restart 



##### optirun 
optirun -b primus -vv env DISPLAY=:8 /opt/VirtualGL/bin/glxspheres64 

tham khao https://github.com/Bumblebee-Project/Bumblebee/issues/971  
vblank_mode=0 optirun xxx to avoid fps limitation 

You can install Bumblebee to Linux Mint 19 as following

1. Installation of required packages

sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update
sudo apt install bumblebee nvidia-390 primus  primus-libs-ia32 linux-headers-$(uname -r) -y
2. Post-installation configuration

Change following lines from

LibraryPath=/usr/lib/nvidia-current:/usr/lib32/nvidia-current
XorgModulePath=/usr/lib/nvidia-current/xorg,/usr/lib/xorg/modules
to

LibraryPath=/usr/lib/x86_64-linux-gnu:/usr/lib/i386-linux-gnu
XorgModulePath=/usr/lib/x86_64-linux-gnu/nvidia/xorg,/usr/lib/xorg/modules,/usr/lib/xorg/modules/input
in /etc/bumblebee/bumblebee.conf file.

Add following line to /etc/environment file.

__GLVND_DISALLOW_PATCHING=1

Add following lines to /etc/modprobe.d/blacklist-nvidia.conf file.

blacklist nouveau
blacklist nvidia
blacklist nvidia-drm
blacklist nvidia-modeset
alias nouveau off
alias nvidia-drm off
alias nvidia-modeset off
3. Disable gpumanager

You can disable gpumanager with following commands.

echo 'GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX nogpumanager"' | sudo tee /etc/default/grub.d/nogpumanager.cfg
sudo update-grub

3a. Add acpi parameter for some BIOS types with following commands.

echo 'GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX acpi_osi=! acpi_osi=\"Windows 2009\""' | sudo tee /etc/default/grub.d/nvidia-acpi.cfg
sudo update-grub

4. Disable unnecessary services

sudo systemctl disable nvidia-fallback.service
sudo systemctl disable nvidia-persistenced.service

5. Reboot

Tip: You can use optirun -b primus instead of primusrun.



