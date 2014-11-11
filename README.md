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
bind-address = 127.0.0.1
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

A database is required to configure the identity service, this is all done on the controller node: 

```bash 
mysql -u root -p
CREATE DATABASE keystone; 
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY 'password';
exit
```

Generate a unique value to be used as the administration token, keep this value safe: 

```bash
openssl rand -hex 10
b2b4d7575f3da19b78a9
```

The following components will need to be installed and configured to enable the identity service: 

```bash
yum install openstack-keystone python-keystoneclient
```

Edit the ``/etc/keystone/keystone.conf`` file and commit the following edits: 

```bash
[DEFAULT]
...
admin_token = admintoken (generated in previous step)

[database]
...
connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
...
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.sql.Token
```

The following commands will: generic certificates, appropriate permissions and a population of the database, as well as enabling the keystone service and creating tentants, users and roles: 

```bash
keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
chown -R keystone:keystone /var/log/keystone
chown -R keystone:keystone /etc/keystone/ssl
chmod -R o-rwx /etc/keystone/ssl
systemctl enable openstack-keystone.service
systemctl start openstack-keystone.service
su -s /bin/sh -c "keystone-manage db_sync" keystone
systemctl enable openstack-keystone.service
systemctl start openstack-keystone.service
export OS_SERVICE_TOKEN=(admin token generated earlier)
export OS_SERVICE_ENDPOINT=http://178.62.59.79:35357/v2.0
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin_pass
export OS_AUTH_URL="http://localhost:5000/v2.0/"
keystone tenant-create --name admin --description "Admin Tenant"
```

Some basic user accounts are created and assigned to roles for testing purposes: 

```bash
keystone user-create --name admin --pass ADMIN_PASS --email EMAIL_ADDRESS
keystone role-create --name admin
keystone user-role-add --tenant admin --user admin --role admin
keystone role-create --name _member_
keystone user-role-add --tenant admin --user admin --role _member_
keystone tenant-create --name demo --description "Demo Tenant"
keystone user-create --name demo --pass DEMO_PASS --email EMAIL_ADDRESS
keystone user-role-add --tenant demo --user demo --role _member_
keystone tenant-create --name service --description "Service Tenant"
```

#### Create Service User

The test users and tenants are now created, so a service entity is required for the identity service. 

```bash
keystone service-create --name keystone --type identity --description "OpenStack Identity"
keystone endpoint-create \
  --service-id $(keystone service-list | awk '/ identity / {print $2}') \
  --publicurl http://127.0.0.1:5000/v2.0 \
  --internalurl http://127.0.0.1:5000/v2.0 \
  --adminurl http://127.0.0.1:35357/v2.0 \
  --region regionOne
```

#### Glance configuration

Some database configuration is required to set up Glance image storage: 

```bash
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
exit
```

The glance user requires appropriate permissions: 

```bash
keystone user-create --name glance --pass glance_pass
keystone user-role-add --user glance --tenant service --role admin
keystone service-create --name glance --type image --description "OpenStack Image Service"
keystone endpoint-create \
  --service-id $(keystone service-list | awk '/ image / {print $2}') \
  --publicurl http://127.0.0.1:9292 \
  --internalurl http://127.0.0.1:9292 \
  --adminurl http://127.0.0.1:9292 \
  --region regionOne
```

Now that the glance user has been created and placed in the right groups, the Glance service components can be installed and configured: 

```bash
yum install openstack-glance python-glanceclient
```

Make the following changes to the ``/etc/glance/glance-api.conf`` file: 

```bash
[database]
...
connection = mysql://glance:GLANCE_DBPASS@127.0.0.1/glance

[keystone_authtoken]
...
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = glance
admin_password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone

[glance_store]
...
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

Edit the ``/etc/glance/glance-registry.conf`` file:

```bash
[database]
...
connection = mysql://glance:GLANCE_DBPASS@127.0.0.1/glance

[keystone_authtoken]
...
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = glance
admin_password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone
```

To finalise the changes and verify the service works, run the following: 

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
mkdir /tmp/images
cd /tmp/images
wget http://cdn.download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img
glance image-create --name "cirros-0.3.3-x86_64" --file cirros-0.3.3-x86_64-disk.img \
  --disk-format qcow2 --container-format bare --is-public True --progress
```
