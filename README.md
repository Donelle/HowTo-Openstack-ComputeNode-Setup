# HowTo Setup Openstack Compute Node (Grizzly)


This tutorial attempts to describes how to setup an Openstack (Grizzly) Compute Node using Xenserver 6.2.0. 

**Disclaimer:** These are the notes I took during my month long `Hacking Openstack` venture trying to get it functioning (which I manged to do). I do not claim to be an expert on Openstack therefore I  take no responsibility for any hardware or software damages that may occur to your system. I recommend that you perform this tutorial in a sandbox environment.

The configuration setup described in this tutorial is adapted from [Bilel Msekni](http://www.linkedin.com/profile/view?id=136237741&trk=tab_pro) 's [OpenStack Grizzly Install Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide) which focuses on the **Compute Node** portion of the setup. The following diagram shows the architectural setup:

![](http://i.imgur.com/Frsughe.jpg)

It's assumed that you've already have a functioning **Controller Node** setup somewhere on your network.

## Directory Map

* quantum-agent/ - (root directory for quantum resources)
	* nova-agent-Linux-x86_64-0.0.1.37.tar.gz - *(Agent service ran on instances)*
	* quantum-xen-plugins.noarch.rpm - *(quantum plugins that run in XenServer Dom0)*
	* quantum-plugin-agent.patch.orig - *(Original patch for quantum plugin see: [Change I7795446e: Add support for OVS l2 agent in XS/XCP domU](https://review.openstack.org/#/c/15022/))*
	* quantum-plugin-agent.patch - *(modified patch for quantum plugin)*


## Server Prerequesits

The following set of instructions assumes that the server:

* has atleast 3 NIC
* has atleast 16GB of RAM
* has atleast 2 HD, (1st HD runs the OS, Nth HD storage for VMs)
* BIOS settings has Virtualization turned on

## XenServer Installation

1. Pop in the XenServer 6.2.0 installation medium into your server and allow the system to boot from CDROM
2. Following the initial boot messages and the Welcome to XenServer screen, select the keyboard layout to use.
3. Next, the XenServer End User License Agreement (EULA) is displayed, **accept it**.
4. Next, select **Perform clean installation**
5. If you have multiple local hard disks, choose a Primary Disk for the installation. Select Ok.
6. Next, choose which disk(s) you would like to use for virtual machine storage. (Information about a specific disk can be viewed by pressing F5)
    * After choosing your disk(s) select **Enable thin provisioning**
7. Select your installation media source (which will be CDROM)
8. Skip the **Verify installation source** option
9. Next, set and confirm a root password, which XenCenter will use to connect to the XenServer host.
10. Next, set up the primary management interface that will be used to connect to XenCenter. (This will be the first NIC on the system)
11. Next, specify the hostname and the DNS configuration, manually or automatically via DHCP. (Choose **DHCP**)
12. Next, select your time zone
13. Next, skip NTP option (We will do that differently later on)
14. Finally, select **Install XenServer**, wait until installation completes then reboot the system

### XenServer Configurations

*The following set of instructions require you to be on the XenServer*

1. After the XenServer has booted, at the **Configuration** console select **Network and Management Interface**
    * Select **Configure Management Interface** and press [enter]
    * Next, select the device labeled with **eth0**
    * Next, select **Static** option from the list
    * On the next screen fill in the following:
        * IP Address: 192.168.1.232 (Replace this with the appropriate IP on the 192.168.1.0/24 network)
        * Netmask: 255.255.255.0
        * Gateway: 192.168.1.1
        * Hostname: n01-openstack-node
2. Next, select the **Network Time** option
    * Next, select **Enable NTP Time Synchronization**
    * Next, select **Add an NTP Server** and on the next screen enter 192.168.1.231 (Replace with Controller Node IP) and press [enter]
    * Next, select **Add an NTP Server** (again) and on the next screen enter 10.10.7.1 (Replace with Controller Node IP) and press [enter]

### Install Openstack Plugins

1. Press **alt+F3**, which should bring you to the terminal console, then **login as root**
2. Download the plugins package from the **quantum-agent** folder in this repo and install the package *(a little **wget** magic should do the trick)*
    * $ rpm -i openstack-quantum-xen-plugins.noarch.rpm


### Create an ISO Storage Repository (Optional)

***Note:** Remember to replace UUID (12345678-1234-5678-1234-567812345678) with the actual ID*

1. $ cd /var/run/sr-mount/12345678-1234-5678-1234-567812345678 && mkdir iso
2. $ sr_uuid=$(xe sr-create name-label=LocalISO type=iso device-config:location=/var/run/sr-mount/12345678-1234-5678-1234-567812345678/iso device-config:legacy_mode=true content-type=iso)
3. $ xe sr-param-set other-config:i18n-key=local-storage-iso uuid=$sr_uuid


### Create Openstack Tenant Network

*The following set of instructions require **XenCenter***

1. Open **XenCenter** and connect this server
2. Next, select the Networking tab
3. Under Networks list, click the **Add Network..** button
    * In the next screen, select the **Single-Server Private Network** option and press [next]
    * Next, enter a name to represent this network: n01-tenant-network , and press [next]
    * Next, press [finish]
4. Under IP Address Configuration on the same tab, click the **Configure** button
    * On the left pane, click **Add IP address** button
    * On the right pane enter the following and then press [ok]:
        * Name: Openstack Management
        * Network: Network 1
        * IP Address: 10.10.7.100 (Replace this with the appropriate IP on the 10.10.7.0/24 network)
        * Subnet Mask: 255.255.255.0
5. Next, select the **Console** tab
    * At the command prompt press [enter]
    * $ xe network-list
    * On the screen search for the name you created for the tenant network in step #3 and make note of the **bridge name**:
  
    	               uuid ( RO) : 7a6c5e4b-81cf-4bed-3059-1a20d10f16ae
                  name-label ( RW): n01-tenant-network
            name-description ( RW): 
                      bridge ( RO): xapi0 

6. Next, add the integration bridge:
    * $ ovs-vsctl add-br xapi0  


## Openstack VM Installation

*The following set of instructions assumes that you’ve already created a VM instance with Ubuntu Server installed and that you’ve setup the network interfaces **(Network-0 and Network-1)** to be attached to this VM. Also the instructions require you to be running in privilege mode, either as **root** or executing **sudo** before the command*

1. Login to your Ubuntu instance and execute the following:
    * $ apt-get update -y
    * $ apt-get upgrade -y
    * $ apt-get dist-upgrade -y
    * Reboot (you might have new kernel)
2. Download the **quantum-agent** folder in this repo
    * $ apt-get install git
    * $ git clone git@github.com:Donelle/HowTo-Openstack-ComputeNode-Setup.git
3. Install NTP service
    * $ apt-get install -y ntp
    * edit **/etc/ntp.conf** with the following:
        * #server 0.ubuntu.pool.ntp.org <- Comment out this line
        * #server 1.ubuntu.pool.ntp.org <- Comment out this line
        * #server 2.ubuntu.pool.ntp.org <- Comment out this line
        * #server 3.ubuntu.pool.ntp.org <- Comment out this line
        * server [your controller node ip address goes here] i.e. 10.10.7.1
4. Install **vlan** and **bridge-utils**
    * $ apt-get install -y vlan bridge-utils
    * edit /etc/sysctrl.conf with the following:
        * net.ipv4.ip_forward=1 <— Uncomment this line by removing the ‘#’
5. Setup the network intefaces in **/etc/network/interfaces** : 

        # Internet access interface
        auto eth0
        iface eth0 inet dhcp

        # Openstack Management interface
        auto eth1
        iface eth1 inet static
        # replace this ip with the appropriate ip that
        # the controller will communicate with this node on
        address 10.10.7.3
        gateway 255.255.255.0

6. Now reboot the system


### Quantum Installation

1. Install the **openVSwitch** and add your bridge:
    * $ apt-get install -y openvswitch-switch openvswitch-datapath-dkms openvswitch-datapath-source
2. Install the **Quantum** openvswitch agent:
    * $ apt-get install -y quantum-plugin-openvswitch-agent
3. Configure the quantum plugin:
    1. $ cd /etc/quantum/plugins/openvswitch/
    2. $ scp root@10.10.7.1:/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini . # Replace the IP with your Controller’s IP
    3. edit ovs_**quantum_plugin.ini**:
        * tenant\_network\_type = vlan
        * network\_vlan\_ranges = physnet2:100:2999
        * integration_bridge = xapi0 # This represents the bridge associated with the tenant network created earlier
        * bridge_mappings = physnet2:xenbr2 # The bridge used for VM traffic (It will be the bridge associated with the 3rd NIC)
    4. $ cd /etc/quantum/
    5. $ scp root@10.10.7.1:/etc/quantum/quantum.conf . # Replace the IP with your Controller’s IP
    6. edit **quantum.conf**:
        * rabbit_host = 10.10.7.1 <— Uncomment this line by removing the ‘#’ and replace the IP with your Controller’s IP
        * root_helper = sudo quantum-rootwrap-xen-dom0 /etc/quantum/rootwrap.conf
    7. $ scp root@10.10.7.1:/etc/quantum/api-paste.ini . # Replace the IP with your Controller’s IP
    8. edit **/etc/sudoers.d/quantum_sudoers**
        * quantum ALL = NOPASSWD: ALL
4. Patch Quantum to run agent in dom0 on the XenServer
    * $ cd /usr/lib/python2.7/dist-packages/quantum && patch -p1 < ~/HowTo-Openstack-ComputeNode-Setup/quantum-agent/quantum-plugin-agent.patch
    * Enter the following at the next prompt:
        * $ /etc/quantum/rootwrap.conf [press enter]
    * $ chmod +x bin/quantum-rootwrap-xen-dom0 && cp bin/quantum-rootwrap-xen-dom0 /usr/bin/
5. Configure quantum rootwrap:
    * $ cd /etc/quantum/
    * edit **rootwrap.conf**:
        * xenapi\_connection\_url=http://10.10.7.101
        * xenapi\_connection\_username=root
        * xenapi\_connection\_password=yourpassword
6. Restart service
    * $ service quantum-plugin-openvswitch-agent restart
 


### Nova Compute Installation

1. Install **Nova Compute** and **XenAPI**:
    * $ apt-get install -y nova-compute python-xenapi
2. Configure nova-compute:
    * $ cd /etc/nova
    * $ scp root@10.10.7.1:/etc/nova/api-paste.ini . # Replace the IP with your Controller’s IP
    * $ scp root@10.10.7.1:/etc/nova/nova.conf . # Replace the IP with your Controller’s IP
    * edit **nova.conf**:
        * vncserver\_proxyclient\_address=192.168.1.232 # Replace the IP with our host’s XenServer Management IP
        * public_interface=eth0
        * vlan_interface=eth2
        * flat_injected=False
        * network_host=10.10.7.2 # Replace this with the network node IP
        * send_arp_for_ha=True
        * multi_host=True
    * edit **nova-compute.conf** :
                  
            [DEFAULT]
            #debug=True
            libvirt_type=xen
            compute_driver=xenapi.XenAPIDriver
            xenapi_connection_url=http://10.10.7.100 # Replace this with this current's host Openstack Management IP
            xenapi_connection_username=root
            xenapi_connection_password=yourpassword
            xenapi_vif_driver=nova.virt.xenapi.vif.XenAPIOpenVswitchDriver
            xenapi_ovs_integration_bridge=xapi0 # This represents the bridge associated with the tenant network created earlier
            xenapi_torrent_images=none
            sr_matching_filter=default-sr:true
3. Restart service
    * $ service nova-compute restart

### Nova Compute Agent Installation

1. $ cd ./HowTo-Openstack-ComputeNode-Setup/quantum-agent
1. $ mkdir nova-agent && tar -xzf nova-agent-Linux-x86_64-0.0.1.37.tar.gz -C ./nova-agent
2. $ cd nova-agent
3. $ ./installer


### XenServer Tools Installation

1. Open XenCenter and select the the VM running OpenStack *(I named the vm **OpenStack-VM**)*
2. Select the **Console** tab and then select xs-tools.iso from the drop down list labeled DVD Drive
3. Login into the VM from the console and issue the following commands:
    * $ mkdir /mnt/xs-tools
    * $ mount /dev/xvdd /mnt/xs-tools
    * $ cd /mnt/xs-tools/Linux/
    * $ bash install.sh *(If you get an error run **dpkg -i xe-guest-utilities_6.2.0-1120_amd64.deb**)*
    * $ reboot
 
## Questions?

If you have any questions or comments about the tutorial please feel free to drop me a line :-).

Email: <donellesanders@thepottersden.com>
Follow Me: [@DonelleJr](https://twitter.com/DonelleJr)