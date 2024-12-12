# Raspberry Pi 3B+ 1GB RAM
(Moving from a Raspberry Pi 2 Model B 1GB RAM, this is an upgrade to a 64-bit version!)

After getting Raspberry Pi OS (Legacy, 64-bit) Lite image (with ssh enabled and a user/password created) on to micro SD card through Raspberry Pi Imager:

1) inserted the micro SD card into Raspberry Pi, connected it to my ISP-provided router with an ethernet cable, and powered it on. After waiting for a minute or so, the Pi showed up on the admin console of the router. It was given the IP address, 192.168.1.66

2) logged into Pi from a terminal on my Mac:

    ```ssh rpi3b1gb@192.168.1.66```

3) updated packages. (the following update / upgrade / autoremove can be run later also when the system needs to be updated with the latest Docker packages and/or other packages)

    ```
    sudo apt update && sudo apt upgrade -y && sudo apt autoremove
    ```

4) set the IP address (from step 1) above) as static IP address on Pi:
    ```
    sudo systemctl enable NetworkManager
    sudo systemctl start NetworkManager
    nmcli device show eth0
    sudo nmtui edit “Wired connection 1” # Pi / Gateway / Router IP Address are set here
    sudo shutdown -h reboot # if that doesn't work, try: sudo shutdown -r now
    ```
    ![edits in nmtui](images/screenshots/nmtui.jpg)
   
6) Mounted a USB drive (which till now I had on my OpenWRT router and accessible on home network through Samba) with FLAC files of my (approximately 250) audio CD collection:
    ```
    # if needed, to add, delete, or modify disk partitions
    # sudo fdisk /dev/sda and
    # sudo mkfs.ext4 /dev/sda1
    sudo mkdir /mnt/usb128gb
    sudo mount /dev/sda1 /mnt/usb128gb
    lsblk -d -fs /dev/sda1 #use info from this command in the next step
    ```

7) Using information from 'lsblk' above, added the following to /etc/fstab so that the USB drive is automatically mounted whenever a reboot occurs:
    ```
    UUID=58442e0d-2fd0-41a3-8cb1-9a321c4638e4 /mnt/usb128gb ext4 noauto,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.device-timeout=2
    ```

8) Installed and configured Samba so that the files on USB drive are accessible on home network:
    ```
    sudo apt install samba -y
    # [music]
    #    comment = All My Music
    #    path = /mnt/usb128gb/music
    #    browseable = yes
    #    read only = yes
    #    guest ok = yes
    #
    # [nobody]
    #    comment = so as to not display
    #    browseable = no
    # added above 10 lines to the end of /etc/samba/smb.conf
    vi /etc/samba/smb.conf
    sudo service smbd restart
    sudo apt install ufw -y
    sudo ufw allow samba
    ```
9) Installed Docker and Portainer (I'll be using Portainer web UI to manage Docker containers):
    ```
    curl -sSL https://get.docker.com | sh
    docker version
    sudo systemctl enable docker
    sudo usermod -aG docker rpi3b1gb
    sudo docker pull portainer/portainer-ce:linux-arm
    sudo docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:linux-arm
    ```
10) With [docker-compose.yaml](docker-compose.yaml) loaded as stack in Portainer (at 192.168.1.66:9000) with environment variables from [rpi3b1gb.env](rpi3b1gb.env), I now have this running on my Pi:

    ![containers in portainer](images/screenshots/portainer_3.png)

11) Portainer can be upgraded later when there is a newer version:
    ```
    sudo docker stop portainer && sudo docker rm portainer && sudo docker pull portainer/portainer-ce:linux-arm && sudo docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:linux-arm
    ```
12) One can't trust all open source software. Never hurts to have some antivirus software (from a reputable company like Cisco, no less!) run periodiclaly looking for trojans, viruses, malware & other malicious threats.
    ```
    sudo apt-get install clamav
    sudo systemctl stop clamav-freshclam
    sudo freshclam
    sudo systemctl start clamav-freshclam
    sudo systemctl enable clamav-freshclam
    #exclude multiple directories repeating --exclude flag as many times as there are directories to be excluded
    sudo clamscan --log=/tmp/clamscan.log --exclude=/mnt/usb128gb --suppress-ok-results --infected --recursive  / > /dev/null 2>&1 &
    ```
    Takes some time for scan to complete. Check the log file, /tmp/clamscan.log, later.
    
13) Add the following to root's crontab to run the above periodically.
    ```
    # update virus definition files and then scan for viruses
    # at midnight, on first day of every month
    0 0 1 * *   rm -f /tmp/clamscan.log; /usr/bin/freshclam > /dev/null 2>&1; /usr/bin/clamscan --log=/tmp/clamscan.log --exclude=/mnt/usb128gb --suppress-ok-results --infected --recursive / > /dev/null 2>&1
    ```    
