One of the design goals for Xenon is the ability to run in low-cost, memory-constrained environments. To verify this, we have setup a Xenon cluster on 5 Raspberry PIs at VMware that is integrated with Xenon CI and gets used to run Xenon multi-node tests.

![](https://lh5.googleusercontent.com/-Y_WkxINFo2s/V2uKgMIGZ3I/AAAAAAAAKMw/cB-8XUCIxxw-M1XsGdvNt1OEN3EynJMawCL0B/w2618-h1546-no/DSC_1466.JPG)
![](https://lh3.googleusercontent.com/-7ysWshxfHjI/V2uKeeqz7CI/AAAAAAAAKM0/HwbQ1dzzbeMei7L9T22t2ExPVx6fOyixQCL0B/w2620-h1662-no/DSC_1462.JPG)

## Instructions
This article documents the steps involved in setting up a Raspberry PI 3 to run Xenon:

### Choose and Install the **Operating System**
You can find detailed instructions for how to install the Operating System from Raspberry's website [Link](https://www.raspberrypi.org/documentation/installation/installing-images/). For our 5 node setup, we chose **Raspbian Jessie** [Link](https://downloads.raspberrypi.org/raspbian_latest). OS installation on a Raspberry PI only involves setting up the micro-SD card with the OS image and should take a few minutes depending on the OS you are installing. 

### Boot-up the Raspberry Pi
Make sure it is connected to a network with internet connectivity. After booting the Pi, change basic settings including Locale, Timezone, Password. 

### Networking
You can rely on DHCP for the raspberry Pi. However, DHCP ip addresses can change that can cause issues when running Xenon as part of a cluster. We recommend you instead use static-ip addresses on the raspberry pi in such scenarios. Raspbian Jessie uses dhcpcd5 by default. To set a static-ip address for dhcpcd5, you can do the following: Edit /etc/dhcpcd.conf and add the below lines at the end of the configuraiton file: 

```
interface eth0
static ip_address=10.118.96.72/23
static routers=10.118.97.253
static domain_name_servers=10.118.97.253
```

Once this is done, delete the below two files and reboot the raspberry pi:
* /var/lib/dhcp/dhclient.leases
* /var/lib/dhcpcd5/dhcpcd-eth0.lease

### Setting up NTP
When running a Xenon cluster, it is _critical_ that all nodes are in-sync with respect to Time. Since a Raspberry Pi does not have a system clock, you will have to rely on NTP or something similar to keep the time updated on the PI. Raspbian Jessie already has NTP installed on it. In-addition we also installed ntpdate through: `apt-get install ntpdate`. NTP relies on UDP port 123, so make sure the port is available on your network and is not blocked by Firewall. If the port is blocked, NTP will not will be able to update the time locally. Also, update the ntp pool servers appropriately based on your location. This can be done by updating the file under `/etc/ntp.conf`. For US, the servers should be set to:
* server 0.us.pool.ntp.org
* server 1.us.pool.ntp.org
* server 2.us.pool.ntp.org
* server 3.us.pool.ntp.org

Once the pools have been updated, you will restart the ntp service through /etc/init.d/ntp restart.

### Copy and Start Xenon host
Raspbian Jessie by default contains JRE/JDK. Once the above changes have been made, the Raspberry Pi should be ready to run your Xenon-Host. You can copy your Xenon jars manually and kick-off the host. For convenience, we have added some bash scripts: [cluster-setup](https://github.com/vmware/xenon/blob/master/contrib/cluster-setup.sh), [cluster-cleanup](https://github.com/vmware/xenon/blob/master/contrib/cluster-cleanup.sh) that will deploy and start the xenon-host jars.
