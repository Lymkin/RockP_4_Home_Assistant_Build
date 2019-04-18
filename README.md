<h1> Install Ubuntu, Docker and Home Assistant on Rock Pi 4b </h1>

<h2>1) Download and Install Ubuntu Image on eMMC Card </h2>

Download here - https://wiki.radxa.com/Rockpi4/downloads

Follow the instructions to install the image to eMMC Card - https://wiki.radxa.com/Rockpi4/getting_started

<h2>2) Initial Setup</h2>

>Login to your Rock Pi
	
	Default username: rock
	Default password: rock

>Install openssh-server, nano and get dhcp assigned IP

	$ sudo apt-get update
	$ sudo apt-get install openssh-server nano
	$ ifconfig

>SSH to Rock Pi

	ssh rock@x.x.x.x

<h2>3) Move root filesystem to NVMe Drive (Optional)</h2>

>Check that NVMe drive is recognized

	$ lsblk

		Output:
			NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
			mmcblk1      179:0    0  14.5G  0 disk
			|-mmcblk1p1  179:1    0   3.9M  0 part
			|-mmcblk1p2  179:2    0     4M  0 part
			|-mmcblk1p3  179:3    0     4M  0 part
			|-mmcblk1p4  179:4    0   112M  0 part /boot
			`-mmcblk1p5  179:5    0  14.3G  0 part /
			mmcblk1boot0 179:32   0     4M  1 disk
			mmcblk1boot1 179:64   0     4M  1 disk
			mmcblk1rpmb  179:96   0     4M  0 disk
			nvme0n1      259:0    0 232.9G  0 disk  <-----NVMe Drive

>Partition and format NVMe Drive

	$ sudo fdisk /dev/nvme0n1

- Enter **"n"** to create new partition.<br>
- Enter **"p"** for primary partition.<br>
- Enter **"1"** for partition number.<br>
- Select default for **"First Sector"** and **"Last Sector"**.<br>
- If prompted enter **"y"** to remove existing signature.<br>
- Enter **"w"** to write partition information.

>Check that new partition was created

	$ lsblk

		Output:
			NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
			mmcblk1      179:0    0  14.5G  0 disk
			|-mmcblk1p1  179:1    0   3.9M  0 part
			|-mmcblk1p2  179:2    0     4M  0 part
			|-mmcblk1p3  179:3    0     4M  0 part
			|-mmcblk1p4  179:4    0   112M  0 part /boot
			`-mmcblk1p5  179:5    0  14.3G  0 part /
			mmcblk1boot0 179:32   0     4M  1 disk
			mmcblk1boot1 179:64   0     4M  1 disk
			mmcblk1rpmb  179:96   0     4M  0 disk
			nvme0n1      259:0    0 232.9G  0 disk
			`-nvme0n1p1  259:2    0 232.9G  0 part  <----- New Partition

>Move root filesystem to NVMe

    $ sudo dd if=/dev/disk/by-partuuid/b921b045-1df0-41c3-af44-4c6f280d3fae conv=sync,noerror bs=4M of=/dev/nvme0n1p1

>Make sure NVMe drive has no errors

    $ sudo e2fsck -f /dev/nvme0n1p1

>Extend filesystem to use entire NVMe drive

    $ sudo resize2fs /dev/nvme0n1p1

>Make sure NVMe drive has no errors after resize

    $ sudo e2fsck -f /dev/nvme0n1p1

>Change extlinux.conf to mount root to NVMe drive

    $ sudo nano /boot/extlinux/extlinux.conf
		change - root=PARTUUID=b921b045-1d
	    	to - root=/dev/nvme0n1p1

>Reboot

	$ sudo reboot

>Check that root file system is mounted to NvME

	$ lsblk

		Output:
			NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
			mmcblk1      179:0    0  14.5G  0 disk
			|-mmcblk1p1  179:1    0   3.9M  0 part
			|-mmcblk1p2  179:2    0     4M  0 part
			|-mmcblk1p3  179:3    0     4M  0 part
			|-mmcblk1p4  179:4    0   112M  0 part /boot
			`-mmcblk1p5  179:5    0  14.3G  0 part 
			mmcblk1boot0 179:32   0     4M  1 disk
			mmcblk1boot1 179:64   0     4M  1 disk
			mmcblk1rpmb  179:96   0     4M  0 disk
			nvme0n1      259:0    0 232.9G  0 disk
			`-nvme0n1p1  259:2    0 232.9G  0 part /  <----- New root mount point

<h2>4) Setup System</h2>

>Unminimize the ubuntu image

	$ sudo unminimize
	$ sudo reboot

>Disable WiFi Interface (Optional)

    $ sudo nano /etc/modprobe.d/blacklist.conf

		Add the following two lines to the end of the file
            # Wireless Networking
            blacklist bcmdhd

	$ sudo reboot

>Create the "hosts" file

	$ sudo nano /etc/hosts

		Add:
			127.0.0.1       localhost
			127.0.1.1       rockpi.yourdomain.local       rockpi
			192.168.1.14    rockpi.yourdomain.local       rockpi

>Change "hostname" file

	$ sudo nano /etc/hostname

		Change: localhost.localdomain To: rockpi

>Setup Static IP

	$ sudo nano /etc/netplan/01-netcfg.yaml

		Add:
			# This file describes the network interfaces available on your system
			# For more information, see netplan(5).
			network:
			  version: 2
			  renderer: networkd
			  ethernets:
			    eth0:
			      addresses: [ 192.168.1.10/24 ] <----- Enter your IP
			      gateway4: 192.168.1.1
			      nameservers:
			          search: [ yourdomain.local ]
			          addresses:
			              - "8.8.8.8"  <----- Your DNS Servers
			              - "8.8.4.4"

	$ sudo reboot

>Reconnet using the new static IP address

<h2>5) Install Docker</h2>

>Install Docker and prerequisites

	$ sudo apt-get update
	
	$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
	
	$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	
	$ sudo apt-key fingerprint 0EBFCD88
	
	$ sudo add-apt-repository  \
	"deb [arch=arm64] https://download.docker.com/linux/ubuntu \
	$(lsb_release -cs) \
	stable"
	
	$ sudo apt-get update
	
	$ sudo apt-get install docker-ce
	
	$ sudo apt-cache madison docker-ce  <----- Potentially Optional (I did not do this)
	
	$ sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu  <----- Potentially Optional (I did not do this)

<h2>6) Install Home Assistant on Docker</h2>

>Install Home Assistant and prerequisite libraries

	$ sudo su

	$ apt-get install jq avahi-daemon avahi-discover avahi-utils libnss-mdns mdns-scan

	$ wget https://raw.githubusercontent.com/home-assistant/hassio-build/master/install/hassio_install

	$ chmod +x ./hassio_install

	$ ./hassio_install -m qemuarm-64

	$ exit

<h2>7) Monitor installation (Optional)</h2>

	$ watch -n 2 'sudo docker ps'

<h4>The installation is pretty quick and should be up and running in less than 5 mins.</h4>