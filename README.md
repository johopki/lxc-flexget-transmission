##Flexget and Transmission on Ubuntu 16.04 Container in LXC

For these purposes, I will be logging in to my host as root

###Install LXC
Using pacman, install the lxc package  
`# pacman -S lxc debootstrap`

### Create container of Ubuntu 16.04
`# lxc-create -t download -n u1 -- -d ubuntu -r xenial -a amd64`  
Where "u1" is the name of your container

### Create a network bridge on host

`# nano /etc/systemd/network/MyBridge.netdev`  
```
[NetDev]
Name=br0
Kind=bridge
~~~~~
`# systemctl restart systemd-networkd.service`

`# /etc/systemd/network/MyEth.network`
~~~~~
[Match]
Name=en*

[Network]
Bridge=br0
~~~

### Enable networking and share a host folder to the container config file 
/var/lib/lxc/u1/config
~~~~
lxc.network.type = veth
lxc.network.link = br0
lxc.network.flags = up
lxc.network.name = eth0

lxc.mount.entry = /[path on host]  /var/lib/lxc/u1/rootfs/[folder name] none bind 0 0
lxc.start.auto = 1
~~~~

https://wiki.archlinux.org/index.php/Systemd-networkd#Basic_DHCP_network
https://wiki.archlinux.org/index.php/Linux_Containers

### E

#3# Install and set up Flexget
  Install
  Configure
  Test
  Clear db if problem
  Set up systemd to run command as timer
  

lxc-start -n containername
 lxc-stop -n containername
 ###Running Transmission as Root
 http://askubuntu.com/questions/261252/how-do-i-change-the-user-transmission-runs-under
 
 sudo systemctl stop transmission-daemon
systemctl edit transmission-daemon.service

[Service]
User=codon

sudo systemctl daemon-reload

edit: ~/.config/transmission-daemon/settings.json

sudo systemctl start transmission-daemon

If you have already configured transmission settings: cp /var/lib/transmission-daemon/.config/transmission-daemon/settings.json ~/.config/transmission-daemon/settings.json 

### autostart lxc 

https://coderwall.com/p/ysog_q/lxc-autostart-container-at-boot-choose-order-and-delay

/var/lib/lxc/myfirstcontainer/config) :

lxc.start.auto = 1
#lxc.start.delay = 0 (in seconds)
#lxc.start.order = 0 (higher means earlier)

Create a Systemd service file and timer file in /etc/systemd/system/

flexget.service
####################
[Unit]
Description=Flexget

[Service]
Type=simple
ExecStart=/usr/bin/flexget execute
------
flexget.timer
####################
[Unit]
Description=Runs myscript every hour

[Timer]
# Time to wait after booting before we run first time
OnBootSec=10min
# Every hour (*) and denomination in minutes
OnCalendar=*:0
Unit=flexget.service
----------------------
sudo systemctl enable flexget.timer

https://jason.the-graham.com/2013/03/06/how-to-use-systemd-timers/

