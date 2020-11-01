# ubuntu-pies
# Goal
Automate install and configure Raspberry Pi with Ubuntu 20.04 LTS 64-bit OS as a Kubernetes cluster.
At current time, this means only Raspberry Pi 3 and 4 are compatible with the 64-bit OS.
This will also perform the installs in a headless configuration.
# Note
The steps were run on a Mac using Ubuntu imager (to build the prepare the SD card), ssh, and ansible.
Hopefully these tasks remain OS neutral, but YMMV.
# Steps
1. Per the [Raspberry Pi Ubuntu installation instructions](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview),
prepare the SD card with [Step #2 Prepare the SD Card](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#2-prepare-the-sd-card)
1. If your raspberry pi will only be connected via wifi, keep the SD card mounted to your host.
If the you or the Ubuntu imager has already unmounted it, remount it by reinserting the SD card back into your host.
   ```
   cd /Volumes/system-boot
   # edit the network-config file and add your wifi configuration (plenty of examples on the web)

   # this should take you out of the previous directory and to your home directory
   cd

   # on a mac, to unmount the SD card do something like this (YMMV for the disk2):
   diskutil unmountDisk /dev/disk2
   ```
1. Insert the SD card into your Raspberry Pi, provide network to your pi and then power.
It should take quite a while, but you pi should boot, join the network, and then install updates.
1. Find your pi's IP address on the network.
This is problematic.
In the past, I would have recommended using **arp**, but you may need to install a tool like **nmap** and run something like:
   ```
   nmap -sV -p 22 10.0.4.0/24 -open
   ```
   and try to locate your pi.
1. Connect remotely with your pi:
   ```
    ssh ubuntu@YOUR_PI_IP_ADDRESS_FROM_PREVIOUS_STEP
   ```
   Initially, the password for the ubuntu user will be *ubuntu*, and you will be asked to change it on first entry.
1. Allow the pi to finish its unattended updates.
Using a tool like *htop*, *top*, or a form of the *ps* command, monitor the progress of the unattended updates, and let finish.
Once it seems the updates have finished, logout, log back in, and the login banner should advise you of the need to reboot the pi to finish installing the updates:
   ```
   # to reboot your pi
   sudo reboot
   ```
   It will probably take a minute or so for the pi to reboot and then *ssh* back into your pi:
   ```
   ssh ubuntu@YOUR_PI_IP_ADDRESS_FROM_PREVIOUS_STEP
   ```