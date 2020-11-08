## How to build wireless connection on Debian

In this article i will show you how to set up network in Debian Buster. There is a few ways to establish a connection. 

1. Wireless network interface can be configurate on a Debian system using a connection manager (such as NetworkManager). 
2. You can use  Debian's /etc/network/interfaces file with a special purpose utility.

I prefer command line solution and use /etc/network/interfaces file and wpa_supplicant package. 

#### Note! 

If your are going to use /etc/network/interfaces you need to delete "network-manager" package: 

```
"apt purge network-manager"
```
I found that after deleting this package via command line config files is still here. So you should search for NetworkManager config file and  delete it manually.

```
root@name:~#find / -name "*NetworkManager*" -print
/etc/NetworkManager/conf.d

root@name:~#rm -rf /etc/NetworkManager/conf.d
```

After that we are ready to building network interface. So, let's start. 


### Check network interface

Make shure that wireless network interface is avaliable on your system and verify it state running "ip a" or "iwconfig" on the command line. 
If you see only lo and eno1 interfaces it means that your wifi card is not detected because of missing drivers for firmware. 

### Find and install missing driver

Run
```
"lspci -k" 
```
to see kernel module for Network controller. 

For example:

```
root@name:~#lspci -k

Network controller: Intel Corporation Wireless 3165 (rev 81)

Subsystem: Intel Corporation Dual Band Wireless AC 3165

Kernel driver in use: iwlwifi

Kernel modules: iwlwifi 
```

So, my wifi card working on iwlwifi firmware. 

Next step is to check if this firmware exist in the system. 

Go to page https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-i-o/wireless-networking.html and find iwlwifi firmware for your Network controller (in my case it is Intel Wireless AC 3165 and iwlwifi-7265-ucode which is equal to 3165 firmware).

Check /lib/firmware/ folder to make shure that firmware exist. if so, check if it is downloaded with "lsmod | grep iwl" command. (Replace "iwl" with your module).If nothing is shown it means that driver for your wifi card firmware is not presented in the system. In order to find what driver is missing run "modinfo iwlwifi" and look at the "depends" line. In my output it was cfg80211 driver.  "Dpkg -S cfg80211" command shows you that driver is located in /lib/modules/4.19.0-5-amd64/kernel/net/wireless/ folder. If folder is empty you need download and install driver.  

The easiest way to install missing drivers is reinstalling kernel via Synaptic. To find kernel images run:

```
root@name:~# apt-cache search linux-image

linux-image-4.19.0-5-amd64 - Linux 4.19 for 64-bit PCs (signed)

linux-image-amd64 - Linux for 64-bit PCs (meta-package)

linux-headers-4.19.0-5-amd64 - Header files for Linux 4.19.0-5-amd64
```

Now open Synaptic and run in search box "linux-image" keyword. THe result was three packages shown above. You need reinstall only two of them: linux-image-4.19.0-5-amd64 - Linux 4.19 for 64-bit PCs (signed) and linux-image-amd64 - Linux for 64-bit PCs (meta-package).

Then check again if firmware and driver for your wifi card is presented in system.

```
root@name:~# lsmod | grep iwl

iwlmvm                299008  0

mac80211              815104  1 iwlmvm

iwlwifi               241664  1 iwlmvm

cfg80211              761856  3 iwlmvm,iwlwifi,mac80211
```

So, you can see that iwlwifi firmware and cfg80211 driver are installed and downloaded. 

Then check again if wifi interface exist with "ip a" command. If network interface is shown but has state "DOWN" don't worry. We will solve this problem later. 

Make shure that your machine connected to the router:

```
root@name:~# sudo iw dev wlo1 scan | grep your SSID
```

If so everything is going okey. 

Another way to install missing drivers can be found here: https://wiki.debian.org/iwlwifi# But it didn't help me. 


### Establish a connection


1. Install wpa_supplicant package

Make shure that you have "wpa_supplicant" package installed: 

```
root@name:~# dpkg -S wpa_supplicant
```

If you don't see files listed you can download package here: https://packages.debian.org/buster/wpasupplicant

```
root@name:~# apt install /home/username/Downloads/wpa_supplicant.deb
```

2. Restrict the permissions of /etc/network/interfaces:

```
root@name:~# chmod 0600 /etc/network/interfaces
```

3. Use the WPA passphrase to calculate the correct WPA PSK hash for your SSID:

```
root@name:~# su -l -c "wpa_passphrase myssid my_very_secret_passphrase > /etc/wpa_supplicant/wpa_supplicant.conf"
```

The above command gives the following output to "/etc/wpa_supplicant/wpa_supplicant.conf":
```
network={

        ssid="myssid"
	
        #psk="my_very_secret_passphrase"
	
        psk=numbers
}
```
Since wpa_supplicant v2.6, you need to add following in your /etc/wpa_supplicant/wpa_supplicant.conf in order to function sudo wpa_cli:

```
ctrl_interface=/run/wpa_supplicant 

update_config=1
```

I have the following wpa_supplicant.conf file:

```
ctrl_interface=/run/wpa_supplicant

update_config=1

network={

	ssid="MySSID"
	
	#psk="my_very_secret_passphrase"
	
	psk=numbers
}
```

You'll need to copy from "psk=" to the end of the line, to put in your /etc/network/interfaces file. 


### Static IP

It's better to set up static IP address. Let's edit /etc/network/interfaces file:

```
#The loopback network interface

auto lo
iface lo inet loopback

#The primary network interface

auto wlo1
allow-hotplug wlo1
iface wlo1 inet static
        address 192.168.1.2
        netmask 255.255.255.252
        #netmask 255.255.255.0
        network 192.168.1.0
        gateway 192.168.1.1
        #broadcast 192.168.1.255
        broadcast 192.168.1.3
        dns-nameservers 192.168.1.1
        #dns-nameservers 8.8.8.8 8.8.4.4
        wpa-ssid myssid
        wpa-psk numbers

```
Connect to the internet:

```
sudo systemctl reenable wpa_supplicant.service

sudo systemctl restart wpa_supplicant.service

sudo wpa_supplicant -B -Dwext -i <interface> -c/etc/wpa_supplicant/wpa_supplicant.conf

I noticed that it's better don't make a daemon (don't add "B"). It's causing errors.
```

### Dynamic IP

If you prefer dynamic IP your /etc/network/interfaces file should look like that: 

```
#The loopback network interface

auto lo

iface lo inet loopback

#The primary network interface

auto wlo1
allow-hotplug wlo1
iface wlo1 inet dhcp
        wpa-ssid myssid
        wpa-psk numbers
```

Add "sudo systemctl restart "dhcpcd.service" line after "sudo systemctl restart wpa_supplicant.service"

But I noticed that ISP can provide the same dhcp address for a long time (about a year or even more). In documents of your ISP you can read that dhcp address is changing every 12-24 houres. But in reality it has the different behavour. So, having dhcp address is nonsensical and non-stable. 

Reboot! You can run: 

```
systemctl restart networking  
```
but it's not enough in some cases. 

### Bringing up a connection

Bring up your interface and verify the connection: 


```
root@name:~# ifup wlp2s0

root@name:~# iw wlp2s0 link

root@name:~# ip a
```

If your network interface is still "DOWN" try reinstall some packages. Open Synaptic and enter in search box "wireless" keyword. 
At first reinstall "wireless-regdb" package. If it's not help try reinstall also "netbase", "wireless-tools", "iw", "crda", "firmware-iwlwifi", "firmware-linux-free", "systemd", "systemd-sysv". If you use DHCP reinstall also isc-dhcp-client. If you still have no internet connection open router's home page: 192.168.1.1 and check if you have desired device listed. Sometimes just open router's home page can help, i don't know why. 

You can check mistake in dmesg, journalctl, look if exclamation mark is present in network bios settings (f10 during boot). 

Reboot!

Enjoy your internet! :)
