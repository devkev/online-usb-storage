Online USB storage
===================

This guide explains how to setup a Raspberry Pi Zero W for as a kind of "online USB drive" --- that is, a device which appears like a regular USB stick/thumb drive ("USB removable mass storage device"), but where files saved onto it are accessable via the network.

I use this with a non-network-enabled scanner, so that it can save the scanned files onto (what it believes is) a USB storage device, and I (and others) can then easily access them without needing to unplug/replug an actual USB drive to transfer them to a computer.

It supports being permanently plugged into such a device (the "target device").  Files are saved into different directories for each (pre-configured) potential user of the system, who can then access those files via `rsync`.  Files saved into the top-level root directory are treated as though they were saved into the default user's directory (which is the first user listed in the `users` config file).

Setup
------

1. These instructions assume Linux is being used on the host that's installing the Raspberry Pi Zero W.  It should be possible to adjust them accordingly for other operating systems.

1. Download latest Rasberry Pi OS (Lite) installation image:

    https://www.raspberrypi.org/downloads/raspberry-pi-os/

1. Copy the filesystem onto micro sd card:

    ```
    $ unzip -p dl/raspberry-pi-os/2020-08-20-raspios-buster-armhf-lite.zip | sudo dd of=/dev/sdX bs=4M conv=fsync status=progress
    ```

1. Mount it:

    ```
    sudo mount /dev/sdX2 /mnt/tmp
    sudo mount /dev/sdX1 /mnt/tmp/boot
    ```

1. Install config to connect to wifi (2.4GHz):

    ```
    sudo vim /mnt/tmp/boot/wpa_supplicant.conf
    ```

    ```
    country=AU
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    network={
        ssid="MyWiFiNetwork"
        psk="aVeryStrongPassword"
        key_mgmt=WPA-PSK
    }
    ```

1. Enable ssh:

    ```
    sudo touch /mnt/tmp/boot/ssh
    ```

1. Enable `dwc2` usb driver device tree overlay:

    ```
    echo "dtoverlay=dwc2" | sudo tee -a /mnt/tmp/boot/config.txt
    ```

1. Enable the `dwc2` and `g_mass_storage` modules:

    ```
    echo "dwc2" | sudo tee -a /mnt/tmp/etc/modules
    echo "g_mass_storage" | sudo tee -a /mnt/tmp/etc/modules
    echo "options g_mass_storage removable=1" | sudo tee -a /mnt/tmp/etc/modprobe.d/gadget.conf
    ```

1. Set the hostname (I chose `scanner-storage`):

    ```
    echo "scanner-storage" | sudo tee /mnt/tmp/etc/hostname
    sudo vi /mnt/tmp/etc/hosts    # edit the last line to change "rasberrypi" to "scanner-storage"
    ```

1. Install your ssh key:

    ```
    sudo mkdir /mnt/tmp/home/pi/.ssh
    sudo chmod 700 /mnt/tmp/home/pi/.ssh
    sudo cp ~/.ssh/authorized_keys /mnt/tmp/home/pi/.ssh/authorized_keys
    sudo chmod 660 /mnt/tmp/home/pi/.ssh/authorized_keys
    sudo chown -R 1000 /mnt/tmp/home/pi/.ssh
    sudo chgrp -R 1000 /mnt/tmp/home/pi/.ssh
    ```

1. Unmount:

    ```
    sudo umount /mnt/tmp/boot
    sudo umount /mnt/tmp
    ```

1. Unplug the microsd card, plug it into the pi, and power up the pi.  Wait 1-2 minutes for it to boot.

1. Login:

    ```
    ssh pi@scanner-storage.local
    ```

1. Do local config:

    ```
    sudo raspi-config
    ```

    a. "1 Change User Password"

    a. "3 Boot Options" -> "B1 Desktop / CLI" -> "B1 Console"

    a. "4 Localisation Options" -> "I1 Change Locale" -> Scroll to "en_AU.UTF-8 UTF-8", press Space, then press Enter -> "en_AU.UTF-8"
    a. "4 Localisation Options" -> "I2 Change Time Zone" -> "Australia" -> "Sydney"

    a. "Finish" -> "Don't reboot"

1. Get this repo:

    ```
    git clone https://github.com/devkev/online-usb-storage
    chmod 711 . online-usb-storage
    cd online-usb-storage
    ```

1. Configure the users (one username per line):

    ```
    vi users
    ```

1. Create the user accounts on the pi:

    ```
    while read user; do
        sudo adduser --disabled-password --gecos '' "$user"
        sudo chmod 711 /home/"$user"
        sudo ln -s ~pi/online-usb-storage/files/"$user" /home/"$user"/files
        sudo mkdir /home/"$user"/.ssh
        sudo chown "$user" /home/"$user"/.ssh
        sudo chgrp "$user" /home/"$user"/.ssh
        sudo chmod 700 /home/"$user"/.ssh
    done < users
    # install the .ssh/authorized_keys files
    ```

1. Install required packages:

    ```
    sudo apt-get install -y lvm2 gawk
    ```

1. Create the template image:

    ```
    ./create-template
    ```

1. Install the python requirements:

    ```
    sudo apt-get install -y python3-pip
    pip3 install -r requirements.txt
    ```

1. Make the script run at boot:

    ```
    sudo sed -e '$i/bin/su - pi -c /home/pi/online-usb-storage/run &' -i /etc/rc.local
    ```

1. Reboot:

    ```
    sudo reboot
    ```


Known issues
-------------

1. There are a lot of hardcoded magic values sprinkled around.

1. There's a lot of non-DRY repeated code.

1. The code assumes that files are saved in a burst, with the device not otherwise accessing the USB storage (except for initial reads).  If the device does not quiesce its write behaviour, then an external trigger (eg. a physical button) may be necessary to trigger the Pi reading the files from the USB storage filesystem image, otherwise corruption may occur.

1. Files saved onto the USB device aren't encrypted, but should be before they leave RAM.

1. If further files are saved onto the USB filesystem, while previous files are being processed, they won't be handled (until the next time files are saved while the trigger is active and "listening" for writes).


References
-----------

* https://core-electronics.com.au/tutorials/raspberry-pi-zerow-headless-wifi-setup.html
* https://gist.github.com/gbaman/50b6cca61dd1c3f88f41
* https://www.raspberrypi.org/forums/viewtopic.php?t=213309
* https://www.raspberrypi.org/forums/viewtopic.php?p=1297312
* https://www.raspberrypi.org/forums/viewtopic.php?t=223127

* https://www.raspberrypi.org/documentation/hardware/raspberrypi/power/README.md
* https://blog.gbaman.info/?p=699
* http://www.isticktoit.net/?p=1383
* https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget
* https://www.instructables.com/NAS-Access-for-Non-Networked-Devices/
