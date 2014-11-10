# Initial Setup
To begin deploying a single node OpenStack test environment, create a VM with ~4GB RAM, 4 physical cores and >10GB disk running RedHat 7/Opensource derivative. 
Once the VM is created, it must have swap enabled to complete the install. To create swap on RedHat derivative operating systems, perform the following steps. 

## Swap Creation
To create a swap file of 4GB, run the following command: 

```bash
sudo fallocate -l 4G /swapfile
```

Adjust the permissions of the created swapfile to deny reads from any user other than root:

```bash
sudo chmod 0600 /swapfile
```

Confirm the permissions have been set correctly with an ``ls -lh``, which should provide the following output: 

```bash
-rw------- 1 root root 4.0G Oct 30 11:00 /swapfile
```

The swap file is now secure, and can be configured for use with the following command: 

```bash
sudo mkswap /swapfile
sudo swapon /swapfile
```

To ensure the swap file is persistent through reboot, edit the ``/etc/fstab`` file with the following line: 

```bash
/swapfile   swap    swap    sw  0   0
```

# OpenStack Deployment

## OpenStack Installation

### Controller node

The OpenStack installation follows the guide published on OpenStack.org for a *Juno* installation with tweaks to allow for installation in a VM. 

``vi /etc/sysconfig/network-scripts/ifcfg-eth1`` and ensure the file has the following configuration: 

```bash
DEVICE='eth1'
TYPE=Ethernet
BOOTPROTO=none
ONBOOT='yes'
```

Install and enable NTP: 

```bash
yum install -y ntp
systemctl enable ntpd.service
systemctl start ntpd.service
```

The EPEL repository is required, and can be enabled with the following:

```bash
yum install -y yum-plugin-priorities
yum install http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-2.noarch.rpm
yum install http://rdo.fedorapeople.org/openstack-juno/rdo-release-juno.rpm
yum upgrade
yum install openstack-selinux
```

#### OpenStack Database

OpenStack requires a database service to store various information - usually run on the headnode. To install MariaDB/MySQL run the following: 

```bash
yum install mariadb mariadb-server MySQL-python
```

The ``/etc/my.cnf`` file needs to be edited to set the bind-address to the *management* IP of the controller node, and make some changes to character encoding, add the following lines into the appropriate sections: 

```bash
[mysqld]
bind-address = 10.131.249.26
default-storage-engine = innodb
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```

Start the MariaDB service and enable it to run across reboots, as well as perform a secure mysql installation (removing test accounts and databases etc.): 

```bash
systemctl enable mariadb.service
systemctl start mariadb.service
mysql_secure_installation
```

#### Messaging Server

Typically, the controller node would run the messaging server required by OpenStack to pass information between services, to install RabbitMQ and enable its services: 

```bash
yum install rabbitmq-server
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
systemctl restart rabbitmq-server.service
rabbitmqctl change_password guest RABBIT_PASS
```

#### Identity Service
