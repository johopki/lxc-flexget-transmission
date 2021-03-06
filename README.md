##Flexget and Transmission on Ubuntu 16.04 Container in LXC on an Arch/Antergos Host
This tutorial is a work in progress. Please let me know if you see anything out of place.

I decided to create this tutorial because I was having trouble building [Flexget](www.flexget.com) for [Antergos](www.antergos.com) which is my deskop OS of choice. It was also a good excuse to learn [LXC containers](https://linuxcontainers.org/lxc/introduction/) which I wanted to do for a while. Be kind. This is my first Github project and tutorial and don't harass me for using nano. Its been good for me for almost ten years now.

For these purposes, I will be logging in to my host as root.  

All commands entered on host system will have a `#` before the commands. All commands issued on the contianer will start with `root@container`.  

### Install LXC
Using pacman, install the lxc package  
`# pacman -S lxc debootstrap arch-install-scripts`

### Create container of Ubuntu 16.04
`# lxc-create -t download -n u1 -- -d ubuntu -r xenial -a amd64`  
Where "u1" is the name of your container

### Create a network bridge on host

`# nano /etc/systemd/network/MyBridge.netdev`  
~~~~
[NetDev]
Name=br0
Kind=bridge
~~~~~
`# systemctl restart systemd-networkd.service`

`# nano /etc/systemd/network/MyEth.network`
~~~~~
[Match]
Name=en*

[Network]
Bridge=br0
~~~

### Enable networking and share a host folder to the container config file 
`# nano /var/lib/lxc/u1/config`
~~~~
lxc.network.type = veth
lxc.network.link = br0
lxc.network.flags = up
lxc.network.name = eth0
#Uncomment the following lines to set a static IP and a MAC Address
#lxc.network.ipv4 = 192.168.0.3/24
#lxc.network.ipv4.gateway = 192.168.0.1
#lxc.network.hwaddr = ee:ec:fa:e9:56:7d

lxc.mount.entry = /[path on host]/[directory you]/[want to share]  /var/lib/lxc/[container name]/rootfs/[folder name] none bind 0 0
lxc.start.auto = 1
#lxc.start.delay = 0 (in seconds)
#lxc.start.order = 0 (higher means earlier)
~~~~

### Connect to container as root and update repository
Connect to the root container using: `# lxc-attach -n [container name]`.   
Update Ubuntu repositories using : `root@container: apt-get update`.  
If this works properly, you are connected to the internet. If you have trouble connecting, you have to troubleshoot your network.

#### Troubleshooting
Using `# lxc-ls --fancy` on your host will give you an output detailing your container(s) and if they have an IP address. 

### Install and set up Flexget
#### Install
Following the [Flexget install instructions](https://flexget.com/InstallWizard/Linux) is very straight forward. For my puropses, I used the following commands:  
`root@container apt-get update`  
`root@container apt-get install python3.5 python-pip`  
`root@container pip install --upgrade setuptools`  
`root@container pip install flexget`  

#### Configure

Following the [Flexget configuration instructions](https://flexget.com/Configuration) was pretty easy, but in the end I opted to use the [Transmission plugin](https://flexget.com/Plugins/transmission) and a prewritten config file. I altered it very little.

#### Test

By typing `root@container flexget` and pressing the tab button a couple of times in your bash shell, you can see the things that flexget can do. One of the import commands for troubleshooting is `root@container flexget database reset --sure` is useful if you are making changes to a config file and want to try to download media that you have already downloaded. Of coure you have to delete the files and clear them from transmission, but thats another step.  

Another is `root@container flexget check` which checks your config file and setup to make sure that you don't have any syntax errors.

### Running Transmission as Root

`root@container systemctl stop transmission-daemon`    
`root@container systemctl edit transmission-daemon.service`
~~~
[Service]
User=root
~~~~
`root@container systemctl daemon-reload`

Edit Transmission config file using `root@container nano ~/.config/transmission-daemon/settings.json`

`root@container systemctl enable transmission-daemon`

If you have already configured transmission settings:  
`root@container cp /var/lib/transmission-daemon/.config/transmission-daemon/settings.json ~/.config/transmission-daemon/settings.json` 

### Making things happen automagically

#### Set up systemd to run `flexget execute` as timer

Create a systemd service file and timer file:   
`root@container nano /etc/systemd/system/flexget.service`   
~~~~
[Unit]
Description=Flexget

[Service]
Type=simple
ExecStart=/bin/bash -c '/usr/local/bin/flexget execute'
~~~~   
`root@container nano /etc/systemd/system/flexget.timer`   
~~~~
[Unit]
Description=Runs myscript every hour

[Timer]
# Time to wait after booting before we run first time
OnBootSec=10min
# Every hour (*) and denomination in minutes (:00)
OnCalendar=*:00
Unit=flexget.service
~~~~   
Run command `root@container systemctl enable flexget.timer` to enable the timer at startup of LXC container and `root@container systemctl start flexget.timer` to start immediately.

### Troubleshooting and useful commands

- `# lxc-start -n [containername]`
- `# lxc-stop -n [containername]`
- `# lxc-ls --fancy`
- `root@container systemctl daemon-reload`

###Links for more detailed instructions which I used in this tutorial
- [Autostarting LXC Containers](https://coderwall.com/p/ysog_q/lxc-autostart-container-at-boot-choose-order-and-delay)
- [Networking with LXC containers](https://wiki.archlinux.org/index.php/Systemd-networkd#Basic_DHCP_network)
- [LXC Containers](https://wiki.archlinux.org/index.php/Linux_Containers)
- [Systemd Timers](https://jason.the-graham.com/2013/03/06/how-to-use-systemd-timers/)
- [Running services as different user](http://askubuntu.com/questions/261252/how-do-i-change-the-user-transmission-runs-under)
