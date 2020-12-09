---
layout: post
title: Install OCP on IBM Cloud Baremetal (UPI)
categories: OpenShift
---

# Install OCP 4.5.6 on IBM Cloud Baremetal Servers (UPI)


## Introduction
Detailed step by step instructions for RH OCP on IBM Cloud Baremetal Servers.
No hypervisor is used in this setup.


## High Level Steps
- Setup Helper Node to run all the required tools. This can be Virtual Server.
- Setup Cloud Services (Load Balancer,Domain, VPN, vlan etc )
- Setup Baremetal Servers for OpenShift
- Install OpenShift 4.5.6
- Installation Validation
- Access the console

## Preparation
__Domain Name__

Make sure you decide on domain name for your cluster.
In this document, I have chosen __ibmgsi.com__.
It is good to have this domain name registered and you have control on update the DNS records. Though it is not mandatory. If you don't own the domain, then you may need to update clients /etc/hosts to point routes to LB external IP. In my case, I own the domain, and I would be able to access load balancers from any client. (For more details, check the Step # 3,Load Balancer configuration).

__Cluster Name__

In this case its __mycluster__.

Baremetal servers would have following hostnames.
- baremetal.mycluster.ibmgsi.com (Bootstrap Node)
- baremetal2.mycluster.ibmgsi.com (Master Node)
- baremetal3.mycluster.ibmgsi.com (Master Node)
- baremetal4.mycluster.ibmgsi.com (Master Node)
- baremetal5.mycluster.ibmgsi.com (Worker Node)
- baremetal6.mycluster.ibmgsi.com (Worker Node)
- baremetal7.mycluster.ibmgsi.com (Worker Node)


---------

## Step 1 Setup the Helper Node

### Get ssh key
From local mac machine, run following commands.
```
mkdir /root/cloudkey
cd /root/cloudkey
ssh-keygen
Enter file in which to save the key (/Users/vishal/.ssh/id_rsa): ./id_rsa
No need to provide passphrase. Enter Twice.

```

### Create a VLAN in your DC (Say Dallas13 in this case)
Create a new private vlan, using Catalog -> services -> VLANs -> Create. 
Select you data center (example Dallas13), name (example OCP45BMVLAN) -> create.


Please make sure all servers use this VLAN

Also please note down the CIDR, on the new VLAN page.

![subnet](/assets/images/subnet.png)


### Provision Virtual Server Instance as helper Node
- Select Catalog -> Services -> Compute -> Virtual Server for Classic
- Select public (Multi Tenant), bastion as name, Dal13 DC, 2 CPU, 4GB
- Select ssh key, generated above, Cent OS, 100GB Boot Disk
- Make sure you select private vlan as 'OCP45BMVLAN'.

Create the VSI Instance
_Please Note: In my environment, the internal IP of VSI instance is 10.209.85.195_


### Add ssh key in your mac machine for quick login
- From client (Mac), use ssh-add <private-key>
- Edit /etc/hosts (Map public IP of VSI = ocphelper)
- Create ~/.ssh/config file and add something like this : 
```
Host ocphelper
	    HostName ocphelper
	    IdentityFile ~/cloudkey/id_rsa
	    User root
```      
Now "ssh ocphelper" should work without password.


### Setup NAT router for Helper Node
- Establish the names of public and private interfaces with ip addr, we will assume eth1 is public and eth0 is private.
- Run sysctl -w net.ipv4.ip_forward=1 to enable IP forwarding
- Add the line net.ipv4.ip_forward = 1 to the file /etc/sysctl.conf to keep it enabled across reboots.
- systemctl start firewalld
- `firewall-cmd –permanent –direct –passthrough ipv4 -t nat -I POSTROUTING -o eth1 -j MASQUERADE -s XXX` where XXX is your subnet CIDR e.g. 10.209.85.192/26
- `systemctl enable firewalld`


### Setup Firewall on Helper Node
- Identify your public and private interfaces. On an IBM Cloud VSI the public interface should be eth1 and the private eth0.
- Add the private interface to the 'internal' group firewall-cmd --permanent --zone=internal --change-interface=eth0
- Add the public interface to the 'external' group firewall-cmd --permanent --zone=external --change-interface=eth1
- Add the SMB service to the internal zone firewall-cmd --permanent --zone=internal --add-service=samba
- Add HTTP to the internal zone firewall-cmd --permanent --zone=internal --add-service=http
- Add DNS to the internal zone firewall-cmd --permanent --zone=internal --add-service=dns
- Start or restart the firewall `systemctl start firewalld` or `systemctl restart firewalld`
- Enable the firewall permanently `systemctl enable firewalld`



### Configure Cluster DNS Service on Helper Node

Use BIND set up on the VSI. 
1. Install bind on Helper Node <code>yum install bind bind-utils</code>
2. Edit /etc/named.conf and create an access control list by adding the following to the top of the file, where XXX is your private subnet CIDR:
```
acl openshift {
        XXX;
};
```
3. In the options section, add or change the following lines:
```
listen-on port 53 { 10.209.85.195; };
allow-query     { localhost; openshift; };
```

4. At the end of the file add the following for the forward zone:
```
zone "mycluster.ibmgsi.com" IN {
 type master;
 file "mycluster.ibmgsi.com.db";
 allow-update { none; };
};
```

5. Reverse zones are named with the significant IP octets in reverse, so if your subnet is 10.1.2.x then the zone shoudl be 2.1.10.in-addr-arpa
```
zone "85.209.10.in-addr.arpa" IN {
  type master;
  file "85.209.10.db";
  allow-update { none; };
};
```

Check out the file from my working environment: [named.conf](/files/named.conf)

6. Create the zone file in for the forward zone in /var/named called mycluster.mydomain.com.db with the following content including your own values. Note that IBM Cloud load balancers have multiple redundant IP addresses, so two records are included.

```
@ IN SOA ns1.softlayer.com. root.mycluster.mydomain.com. (
                       2020030800        ; Serial
                       7200              ; Refresh
                       600               ; Retry
                       1728000           ; Expire
                       3600)             ; Minimum

@                      86400    IN NS    ns1.softlayer.com.
@                      86400    IN NS    ns2.softlayer.com.

*.apps                 900      IN A     <internal proxy LB address 1>
*.apps                 900      IN A     <internal proxy LB address 2>
api                    900      IN A     <internal API LB address 1>
api                    900      IN A     <internal API LB address 2>
api-int                900      IN A     <internal API LB address 1>
api-int                900      IN A     <internal API LB address 2>
etcd-0                 900      IN A     <master 1 address>
etcd-1                 900      IN A     <master 2 address>
etcd-2                 900      IN A     <master 3 address>
<bootstrap name>       900      IN A     <bootstrap address>
<master 1 name>        900      IN A     <master 1 address>
<master 2 name>        900      IN A     <master 2 address>
<master 3 name>        900      IN A     <master 3 address>
<worker 1 name>        900      IN A     <worker 1 address>
<worker 2 name>        900      IN A     <worker 2 address>
<worker 3 name>        900      IN A     <worker 3 address>
<VSI name>             900      IN A     <VSI IP>
_etcd-server-ssl._tcp  900      IN SRV   0 10 2380 etcd-0
_etcd-server-ssl._tcp  900      IN SRV   0 10 2380 etcd-1
_etcd-server-ssl._tcp  900      IN SRV   0 10 2380 etcd-2
```

Checkout working file from my environment: [mycluster.ibmgsi.com.db](/files/mycluster.ibmgsi.com.db)

7. Then create a reverse file matching the reverse zone definition above e.g. 2.1.10.db:

```
$TTL 86400
@ IN SOA ns1.softlayer.com. root.mycluster.mydomain.com. (
                       2020030722        ; Serial
                       7200              ; Refresh
                       600               ; Retry
                       1728000           ; Expire
                       3600)             ; Minimum

@                      86400    IN NS    ns1.softlayer.com.
@                      86400    IN NS    ns2.softlayer.com.

<bootstrap last octet>  IN PTR <bootstrap name>.;
<master 1 last octet>   IN PTR <master 1 name>.;
<master 2 past octet>   IN PTR <master 2 name>.;
<master 3 last octet>   IN PTR <master 3 name>.;
<worker 1 last octet>   IN PTR <worker 1 name>.;
<worker 2 last octet>   IN PTR <worker 2 name>.;
<worker 3 last octet>   IN PTR <worker 3 name>.;
```

Checkout working file from my environment: [85.209.10.db](/files/85.209.10.db)

Restart the DNS service with <code> systemctl named restart </code>

8. Make sure you update the DNS Settings. Edit /etc/resolv.conf and add following lines

```
nameserver  10.209.85.195  # ns1 private IP address
nameserver 10.0.80.11
```
Use Helper Node Private IP as nameserver.
Test name resolution from your local server.


----------

## Step 2 Provision 7 Baremetal servers.

Use following process: 

- Click on Catalog, Left navigation compute -> Bare Metal Servers
- Select the servers, as mentioned in the screenshots
- 7 Numbers of servers, baremetal as name, mycluster.ibmgsi.com as domain name.
- Follow screenshots for more details

- ![Order Device](/assets/images/bm1.png)

- ![Order Device2](/assets/images/bm2.png)

- ![Order Device3](/assets/images/bm3.png)

- Select correct data center, vlan, ssh key, "No OS" option, etc.
- Accept License and provision bare metals.
-------

## Step 3 Configure Load balancers and Domain

Openshift requires load balancers for bootstrap related traffic, for control plane traffic, and for ingress traffic i.e. access to the user applications.  This guide uses the IBM local load balancer service and will create four instances:

| Name             | Type                | Servers            | Ports       |
| -----------------| --------------------|--------------------|-------------|
| Internal Proxy   | Private to private  | Workers            | 80, 443     |
| External Proxy   | Public to private   | Workers            | 80, 443     |
| Internal API     | Private to private  | Bootstrap, masters | 6443, 22623*|
| External API     | Public to private   | Bootstrap, workers | 6443        |

 \* This rule should be removed after installation is complete along with the bootstrap server.

Go to Catalog -> Left Navigation -> Classic Infrastructure -> Load Balancing
Create these four entries:

![Load Balancer](/assets/images/lb3.png)


Please make sure you use TCP as a protocol for LBs.

![Load Balancer](/assets/images/lb2.png)

### Accessing Load Balancers Externally
We also need to make sure, domain names are updated with the external IP address of LB.

Create the following A records in your zone, assuming your cluster name will be mycluster and your domain is ibmgsi.com:

| Name | IP |
|------| ---|
| api.mycluster.ibmgsi.com | IP of external API load balancer |
| *.apps.mycluster.ibmgsi.com | IP of external proxy load balancer |

-------


## Step 4 Setup SMB Server on Helper Node

The VSI needs to host an SMB share to allow the bare metal server management system to mount it. It is also possible to mount ISOs from your local workstation, but it is less convenient in most cases as it requires extra software to be installed.

To set up SMB on the VSI:

- Install the packages with `yum install samba samba-client samba-common`
- Create a group for SMB users `groupadd smbgrp`
- Create an SMB user called e.g. coreos with the command `useradd coreos -G smbgrp`
- Set an SMB password for the user with `smbpasswd -a coreos`
- Create a folder to hold the shared files e.g. `/share/coreos`
- Change the permissions of the shared folder: `chmod 777 /share/coreos`
- Add the following to /etc/samba/smb.conf:

```
[coreos]
	path = /share/coreos
	valid users = @smbgrp
	guest ok = no
	writable = yes
	browsable = no
```
- Under the `[global]` section in smb.conf, add the line `ntlm auth = yes`.  This is an insecure protocol, but it is required because the IPMI console will try to use it when it mounts the ISO as a virtual DVD.
- Start the SMB services with `systemctl start smb.service` and `systemctl start nmb.service`
- Enable the services to start on boot with `systemctl enable smb.service` and `systemctl enable nmb.service`

----


## Step 5 Setup HTTP Servers on Helper Node
This is required to allow the nodes to access their configuration files when they are installing.
- On the VSI, install apache `yum install httpd`
- Start the server 'systemctl start httpd'
- Enable the service permanently `systemctl enable httpd`

-----


## Step 6 Setup Docker on Helper Node
Docker is not specifically required for the installation process, however these instructions use a utility called filetranspiler which is written in Python, and distributed as both Python source and a container image.  In the spirit of the modern containerised world it was decided to use the container version which will be described here.  However those familiar with Python may wish to use the source directly.  Consult the github repo at https://github.com/ashcrow/filetranspiler for instructions on using Python directly.  We will proceed with docker here.

- Install docker with `yum install docker`
- Start docker with `systemctl start docker`
- Enable docker on restart with `systemctl enable docker`

-----

## Step 7 Obtaining the installation files
Openshift 4.x uses Red Hat Core OS.  To install CoreOS on a machine, the machine must boot from an ISO with configuration parameters suppled to a GRUB-like bootloader.  The ISO will then download an ignition file and an OS image which it will will write to the disk.  The ignition file contains configuration for the server.  When the machine is rebooted CoreOS will boot and join the cluster as specified in the ignition file.

There are two version of the OS disk image - one for BIOS and one for UEFI.  These instructions will use the BIOS version but the process should be the same for UEFI.

The ignition files are created by a utility called `openshift-install` which along with the ISO and the OS images is downloadable from Red Hat.

These instructions will conduct the installation from the VSI although it is possible to do this from a local workstation if desired, however the VPN must be correctly configured to allow this.

Visit https://cloud.redhat.com/openshift/install/metal/user-provisioned to download the following:
- The Openshift installer (Linux version)
- Your pull secret
- The Openshift command line tool (oc) (Linux version)

Run all these files and make sure its installed on Helper Node.

From the same page, click on the Download RHCOS link to get to the download archive.  From this page, download the following, where <version> is the appropriate version:
- rhcos-<version>-x86_64-installer.x86_64.iso
- rhcos-<version>-x86_64-metal.x86_64.raw.gz

Place the files in the appropriate locations
- Copy the installer ISO `rhcos-<version>-x86_64-installer.x86_64.iso` to the SMB share e.g. `/share/coreos`
- Copy the disc image `rhcos-<version>-x86_64-metal.x86_64.raw.gz` to the HTML server directory `/var/www/html`

In my installation, I have used OpenShift 4.5.6. 
Files : 
- `rhcos-4.5.6-x86_64-installer.x86_64.iso`
- `rhcos-4.5.6-x86_64-metal.x86_64.raw.gz`

-----

## Step 8 Configure VPN Connectivity from your local mac machine

The web-based IPMI console for managing the bare metal servers is available via a management IP address which is on the same subnet as the servers themselves.  To access this from your local workstation a VPN connection is required.

### Configure VPN for your subnet
- Click on Catalog -> Classic Infrastructure -> Network -> IPSec VPN.
- In the top right, click 'Order VPN', and select your data center from the list.
- Once the VPN is available, click on its name in the list.  On the following configuration page it is not necessary to enter anything in the Remote Peer Address field.
- The Customer Subnets list is a white-list of the IPs from which connection is allowed.  Enter a suitable CIDR.
- The Hosted Private Subnets section is a list of the subnets to which this VPN will allow access.  Click on 'Add Hosted Private Subnet', then in the dialog box find the subnet on which your servers are and click the Add link.
- In my setup its `10.209.85.192/26`.
- Click on Update Device in the bottom right.

### Get Credentials for VPN
Now the endpoint is configured, a username and password will be required to connect.  VPN access is granted to users via the access management page.

- At the top of the screen click on the Manage drop-down and select Access (IAM).  On the next screen click on Users.
- Select your user from the list.
- On the user details page, scroll down to VPN password and enter a password.
- By default you have access to all subnets on which the devices to which you have access are situated.  The account owner has access to all the devices.

To connect to the VPN, an IPSec VPN client is required such as __MotionPro__ which is a free download.  The software requires an endpoint address, a username and password.

- To find your endpoint address, go to https://cloud.ibm.com/docs/iaas-vpn?topic=iaas-vpn-available-vpn-endpoints and select the appropriate endpoint for your data centre.
- The username and password are those configured in the IAM screen earlier in this section.

In my case, where my vlan is on Dallas13, I should use 'vpn.dal.softlayer.com'

-------


## Step 9 Configuring installation
Create a file called install-config.yaml, and add the following content, substituting the appropriate values.  Set the number of worker nodes in this file to zero even though we are adding worker nodes later.  Make sure that the cidr of the cluster network does NOT overlap with the subnet of your bare metal machines.  The service network can be left as it is.  The SSH key is the public part of the one created in step 1.

Here is example of my environment install-config.yaml
```
apiVersion: v1
baseDomain: ibmgsi.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: mycluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/16
    hostPrefix: 26
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: 'xxx'
sshKey: 'ssh-rsa yyy'

```
- xxx = Use pullSecret from previous step.
- yyy = Use ssh key which we created at the beginning of this document, its public part.

_Please note that CIDR in above file shouldn't clash with existing CIDR. Hence we have choose to use 10.128.0.0/16. If you use /8, there would be a problem_

- Create an install directory e.g. `install01`.  This is the working directory for the openshift installer.  **Make a copy** of install-config.yaml and place it in this directory.
- Create manifests `openshift-install create manifests --dir=install01`.  Ignore the warning about no compute nodes.
- Edit `install01/manifests/cluster-scheduler-02-config.yml` and change the parameter `mastersSchedulable` to `false`.
- Create ignition config files with `openshift-install create ignition-configs --dir=install01`

This will create bootstrap, master and worker ignition files that are suitable for use with DHCP.  However this cluster must use static IPs, so each node must have its own ignition file with that static IP written into it.  The static IP must be the same IP as assigned to the server by IBM Cloud.

Ignition files are not human readable, so it is best to use a utility for this.  As mentioned above, these instructions will use filetranspiler available from https://github.com/ashcrow/filetranspiler - but other tools may be available.  Filetranspiler works by taking files from a 'fakeroot' directory and including them into the ignition file.

- Make a directory e.g. `filetranspiler` and change to it.
- `git clone https://github.com/ashcrow/filetranspiler`
- Build the docker image `docker build . -t filetranspiler:latest`
- Make another directory to contain the fake roots e.g. `fakeroots`

The following tasks are repetitive and may be scripted, and a script to help is available in this repository at http://XXXX.  The steps in the script will be described below.  For *each machine* i.e. bootstrap, three masters and three workers, do the following steps:

- Create a standard interface configuration file for the static network in the  fakeroot directory
```
cat > fakeroots/<machine name>/etc/sysconfig/network-scripts/ifcfg-eno1 << EOF
DEVICE=eno1
BOOTPROTO=static
ONBOOT=yes
IPADDR=<ip address>
PREFIX=<subnet prefix>
NETMASK=<netmask>
GATEWAY=<VSI IP>
DNS1=<VSI IP>
DNS2=10.0.80.11
DOMAIN=mycluster.mydomain.com
DEFROUTE=yes
IPV6INIT=no
HOSTNAME=<node name>.mycluster.mydomain.com
EOF
```
In my environment, here is one of file: [ifcfg-eno1](/files/ifcfg-eno1) for bootstrap host.<br/>
Similarly create total 7 files.<br/>
- Run filetranspiler, substituting machine type for boostrap, master or worker:
- That means create total 7 ifcg-eno1 files.
```
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/<machine type>.ign -f fakeroots/ocpmaster41 -o install01/<machine name>.ign
```

In my case of installation, Here is what I ran:
```
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/bootstrap.ign -f fakeroots/baremetal -o install01/baremetal.ign
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/master.ign -f fakeroots/baremetal2 -o install01/baremetal2.ign
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/master.ign -f fakeroots/baremetal3 -o install01/baremetal3.ign
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/master.ign -f fakeroots/baremetal4 -o install01/baremetal4.ign
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/worker.ign -f fakeroots/baremetal5 -o install01/baremetal5.ign
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/worker.ign -f fakeroots/baremetal6 -o install01/baremetal6.ign
docker run --rm -ti --volume `pwd`:/srv:z filetranspiler -i install01/worker.ign -f fakeroots/baremetal7 -o install01/baremetal7.ign
```
After this there should be seven new ign files in the install01 directory.<br/>  
- Copy these files to /var/www/html

------

## Step 10 CoreOS installation on all 7 Bare Metal servers
Installing CoreOS is a two stage process. Firstly, the system boots from the ISO. At this point, basic installation parameters must be entered on the command line to enable networking. The system then downloads an ign file and an OS image from the HTTP server, writes the image to the specified disk and configures the system according to the ign file with networking and the cluster details. When the system reboots from the disk, the system will adopt the role defined in the ign file, either bootstrap, master or worker. The master and worker nodes will contact the bootstrap node which will orchestrate the assembly of the cluster.


__Before you begin here, please make sure to connect the VPN via your IPsec client mentioned in step 8__


For each node - bootstrap, three masters and three workers (in that order), complete the following steps:


### Step 10.1 Prepare Bootstrap Server for CoreOS

- Go to the Classic Infrastructure page on IBM Cloud and select Devices then Device List. Select a device - starting with the bootstrap device.
- Select Remote Management from the left hand menu. Under Management Details there is a password - select Show and then copy the password.
- From the Actions drop-down in the top right, select KVM Console. In the resulting log in screen, enter 'root' for the user and paste the password. The IPMI console should appear.

![Baremetal Management Console](/assets/images/bmc1.png)


### Step 10.2 IPMI Console, Setup CDROM with CoreOS iso Image
- Under Remote Control select Power Control. Use the menu to power the server off if it is running.
- Under Virtual Media, select CD-ROM image. There should be listed three devices with no disk emulation set. If not, contact support to un-mount any unwanted images.
- Enter the details of the SMB share: for host, enter the VSI IP; for the path enter the share name and filename of the ISO e.g. /coreos/rhcos--x86_64-installer.x86_64.iso; enter the username and password set for the SMB user in step

![Baremetal Management Console](/assets/images/bmc2.png)

- Click Save then Mount. If the mount was successful, Device 1 in the list of devices should report that an image is mounted.
- Under Remote Control at the top of the screen, select iKVM/HTML5. Then on the next screen click the iKVM/HTML5 button.
- A pop-up window will appear with a menu and a blank screen.

### Step 10.3 IPMI Console, boot from CDROM and add boot options.
- In the menu select Power Control then Set Power On.
- The CoreOS installer should boot.

_Please note: In some cases, you may need to press F11 (or some other function key) key to enter to boot menu_
_Please use on screen key board as mentioned in below image_


- ![Baremetal Management Console](/assets/images/bmc3.png)

- Hit enter for CDROM option.
- You will be prompted for option of 'Install RHEL CoreOS'
- Use tab, to append addition parameters to the installer.

- ![Baremetal Management Console](/assets/images/bmc4.png)

- Add following additional parameters depending on which Node is being set.
- Options are:

```
ip=machine IP::gateway:subnet:hostname:network adapter:none:nameserver coreos.inst.install_dev=install device coreos.inst.image_url=OS image URL coreos.inst.ignition_url=ignition file URL
```

In my environment, here are the parameters, for Bootstrap Node :
```
ip=10.209.85.219::10.209.85.195:255.255.255.192:baremetal:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal.ign
```
__Please note: If you hate typing such big command everytime, you can use mac 'Script Editor'__
__Look at this page for more information__
[How to reduce typing on IPMI Console](https://www.sythe.org/threads/auto-typer-script-for-mac/)


### Step 10.4 IPMI Console, complete the installation and make sure server boots with CoreOS.

Once the installation is complete, the server would reboot.
During re-boot, make sure you unmount CDROM, so that server would follow normal boot sequence of CoreOS.


At this moment, your bootstrap server has completed installation.

- Monitor the bootstrap process with `openshift-install --dir=install01 wait-for bootstrap-complete --log-level=debug`

------

## Step 11 Install CoreOS on Master and Worker Nodes.

- Follow 10.1, 10.2, 10.3 for all masters and worker nodes.
- Use following parameters depending on which node you are Configuring.
- For Baremetal2 which is master node

```
BAREMETAL2 => Master Node 1
ip=10.209.85.242::10.209.85.195:255.255.255.192:baremetal2:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal2.ign


BAREMETAL3 => Master Node 2
ip=10.209.85.202::10.209.85.195:255.255.255.192:baremetal3:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal3.ign


BAREMETAL4 => Master Node 3
ip=10.209.85.236::10.209.85.195:255.255.255.192:baremetal4:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal4.ign


BAREMETAL5 => Worker Node 1
ip=10.209.85.211::10.209.85.195:255.255.255.192:baremetal5:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal5.ign

BAREMETAL6 => Worker Node 2
ip=10.209.85.218::10.209.85.195:255.255.255.192:baremetal6:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal6.ign

BAREMETAL7 => Worker Node 3
ip=10.209.85.224::10.209.85.195:255.255.255.192:baremetal7:eno1:none:10.209.85.195 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.209.85.195/rhcos-4.5.2-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://10.209.85.195/baremetal7.ign

```

- Monitor the bootstrap process with `./openshift-install --dir=install01 wait-for install-complete --log-level=debug`

------

## Step 12 validation of environment and some of manual steps to complete the installation.

### Connect to OCP Cluster from Helper Node.

- On the Helper Node, change directory to install01/auth
- You will find a file named 'kubeconfig'
- On command prompt. `export KUBECONFIG=/root/OCPInstall3/install01/auth/kubeconfig` (Select correct directory for kuebconfig)
- Run command `oc get nodes` Check if all 6 nodes are ready or not.
- Run command `oc get pods -A` Check if any issue to any pod.
- Run command `oc get clusteroperator` Check if AVAILABLE is true for all operators.

In my installation I got this, where many clusteroperator were showing unknowm or False.
```
[root@ocp-bastion OCPInstall3]# oc --kubeconfig ./install01/auth/kubeconfig get clusteroperator
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                                       Unknown     Unknown       True       8h
cloud-credential                           4.5.7     True        False         False      8h
cluster-autoscaler                         4.5.7     True        False         False      3h52m
config-operator                            4.5.7     True        False         False      3h52m
console                                    4.5.7     Unknown     True          False      3h56m
csi-snapshot-controller                                                                   
dns                                        4.5.7     True        False         False      8h
etcd                                       4.5.7     True        False         False      4h
image-registry                             4.5.7     True        False         False      3h54m
ingress                                              False       True          True       3h54m
insights                                   4.5.7     True        False         False      3h54m
kube-apiserver                             4.5.7     True        False         False      3h57m
kube-controller-manager                    4.5.7     True        False         False      8h
kube-scheduler                             4.5.7     True        False         False      8h
kube-storage-version-migrator              4.5.7     False       False         False      8h
machine-api                                4.5.7     True        False         False      3h54m
machine-approver                           4.5.7     True        False         False      4h2m
machine-config                             4.5.7     True        False         False      8h
marketplace                                4.5.7     True        False         False      3h54m
monitoring                                           False       True          True       3h49m
network                                    4.5.7     True        False         False      8h
node-tuning                                4.5.7     True        False         False      8h
openshift-apiserver                        4.5.7     True        False         False      3h56m
openshift-controller-manager               4.5.7     True        False         False      3h54m
openshift-samples                          4.5.7     True        False         False      3h50m
operator-lifecycle-manager                 4.5.7     True        False         False      8h
operator-lifecycle-manager-catalog         4.5.7     True        False         False      8h
operator-lifecycle-manager-packageserver   4.5.7     True        False         False      4h1m
service-ca                                 4.5.7     True        False         False      8h
storage                                    4.5.7     True        False         False      3h54m

```

If you run into same issue:
- Run command 'oc get csr' Check if all csr are Approved or Pending.
- If Pending, you can approve them manually using following command.
- "oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve"
- Please check the csr status few times, if new csr is created, you need to approve that as well.
- Run command 'oc get clusteroperator' Check if AVAILABLE is true for all operators.

------

## Step 13 : Access the OCP console

In previous step we ran following command.
- 'openshift-install --dir=install01 wait-for install-complete --log-level=debug'
- This should show you following message on successful install.

```

DEBUG Cluster is initialized                       
INFO Waiting up to 10m0s for the openshift-console route to be created...
DEBUG Route found in openshift-console namespace: console
DEBUG Route found in openshift-console namespace: downloads
DEBUG OpenShift console route is created           
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/OCPInstall3/install01/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.mycluster.ibmgsi.com
INFO Login to the console with user: "kubeadmin", and password: "xxxxx-xxxxx-xxxxx-xxxxx"
INFO Time elapsed: 0s                 

```

If you own the Domain Name and you have updated the DNS records, as mentioned in Step # 3, you are good to connect to console or access any route. Else you may need to update /etc/hosts, on your client machine (e.g.mac) and add console route with external load balancer IP address.

------

## Resources
- Installation Document from RH
- https://docs.openshift.com/container-platform/4.5/installing/installing_bare_metal/installing-bare-metal.html

- Special Thanks for the following document. Most of content is reused from following two URLs. 
https://github.com/IBMIntegration/IBM-CP4I-OpenShift-BareMetal-Native 
https://www.openshift.com/blog/openshift-4-bare-metal-install-quickstart 

-------
