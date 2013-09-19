## OpenStack (Grizzly) - Glance Management Instructions

### Creating and Uploading VMs to Glance

*The execution of the __glance__ commands require you to be on a system that have the python-glance package installed*

### Hyper-V (VHDs)

Navigate to the folder where your VMs are located and execute the following commands:

1. $ cp your_vm\_image.vhd image.vhd
2. $ tar -czf image.vhd.gz image.vhd
3. Import the VHD into Glance:
    * Windows Base VM
        * $ glance add name=“My VHD image” is\_public=True disk\_format=vhd container\_format=ovf **vm\_mode=hvm** image\_state=available architecture=x86\_64 < image.vhd.gz
    * Linux VM
        * $ glance add name=“My VHD image” is\_public=True disk\_format=vhd container\_format=ovf **vm_mode=xen** image\_state=available architecture=x86\_64 < image.vhd.gz
4. $ glance image-list