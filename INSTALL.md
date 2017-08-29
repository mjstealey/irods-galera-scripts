# CentOS 7 - Galera VM - VirtualBox

## Installation

Using CentOS 7 [Minimal ISO](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1611.iso) as the base image, install MariaDB 10.1 and iRODS 4.2.1, then configure to use as Galera Cluster.

### Base image

VirtualBox image starting from only basic networking and a single user account with sudo rights.

Users (username / password):

- root / galera
- galera / galera (sudo rights)

Enable networking and update images

```
nmcli d
nmtui
... configure network settings
service network restart
yum clean all
yum update
```

Clear bash history for each user

```
cat /dev/null > ~/.bash_history && history -c && exit
```

Export appliance as .ova file ([centos-7.ova](./centos-7.ova))


### Prep work for VBox image

From VirtualBox: `File` > `Import appliance` > `centos-7.ova`

Update network settings based on host-only network configuration in VirtualBox settings.

- locally defined vboxnet2 to be non DHCP at 192.168.58.1/24
- Update: UUID between multiple VMs if identical using `uuidgen`
- File: `/etc/sysconfig/network-scripts/ifcfg-enp0s8`
- From:

    ```
    TYPE=Ethernet
    BOOTPROTO=dhcp
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=enp0s8
    UUID=803ea389-8557-4aed-a672-cb41760073f1
    DEVICE=enp0s8
    ONBOOT=yes
    PEERDNS=yes
    PEERROUTES=yes
    IPV6_PEERDNS=yes
    IPV6_PEERROUTES=yes
    ```
- To:

    ```
    TYPE=Ethernet
    BOOTPROTO=static
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=enp0s8
    UUID=803ea389-8557-4aed-a672-cb41760073f1
    DEVICE=enp0s8
    ONBOOT=yes
    IPADDR=192.168.58.101
    NETMASK=255.255.255.0
    PEERDNS=yes
    PEERROUTES=yes
    IPV6_PEERDNS=yes
    IPV6_PEERROUTES=yes
    ```

Install packages

```
yum install epel-release
yum makecache fast
yum install net-tools dkms 
yum groupinstall "Development Tools"
yum install kernel-devel
```

Set hostname for multiple instances. In example case using

- galera1.example.com (192.168.58.101)
- galera2.example.com (192.168.58.102)

### MariaDB 10.1

Assumes user has latest updates available to CentOS 7 Minimal ISO. All commands should be able to be made by any user with sudo rights.

```
sudo yum update
```

**Packages and Installation**

Create: `/etc/yum.repos.d/MariaDB.repo`

```config
# MariaDB 10.1 CentOS repository list - created 2017-04-23 13:24 UTC
# http://downloads.mariadb.org/mariadb/repositories/
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

```
sudo yum makecache fast
sudo yum install -y \
	epel-release \
	rsync \
	nmap \
	lsof \
	perl-DBI \
	nc \
	boost-program-options \
	iproute \
	iptables\
	libaio \
	libmnl \
	libnetfilter_conntrack \
	libnfnetlink \
	make \
	openssl \
	which \
	MariaDB-server \
	MariaDB-client \
	MariaDB-compat \
	galera \
	socat \
	jemalloc
```

**Firewalld port configuration**

Reference: [opening-ports-for-galera-cluster](http://galeracluster.com/documentation-webpages/firewalld.html#opening-ports-for-galera-cluster)

1. Enable the database service for FirewallD:

	```
	sudo firewall-cmd --zone=public --add-service=mysql
	```

2. Open the TCP ports for Galera Cluster:

	```
	sudo firewall-cmd --zone=public --add-port=3306/tcp
	sudo firewall-cmd --zone=public --add-port=4567/tcp
	sudo firewall-cmd --zone=public --add-port=4568/tcp
	sudo firewall-cmd --zone=public --add-port=4444/tcp
	```

3. Optionally, in the event that you would like to use multicast replication, run this command as well to open UDP transport on 4567:

	```
	sudo firewall-cmd --zone=public --add-port=4567/udp
	```

Make firewall changes persistent

1. Enable the database service for FirewallD:

	```
	sudo firewall-cmd --zone=public --permanent --add-service=mysql
	```

2. Open the TCP ports for Galera Cluster:

	```
	sudo firewall-cmd --zone=public --permanent --add-port=3306/tcp
	sudo firewall-cmd --zone=public --permanent --add-port=4567/tcp
	sudo firewall-cmd --zone=public --permanent --add-port=4568/tcp
	sudo firewall-cmd --zone=public --permanent --add-port=4444/tcp
	```

3. Optionally, in the event that you would like to use multicast replication, run this command as well to open UDP transport on 4567:

	```
	sudo firewall-cmd --zone=public --permanent --add-port=4567/udp
	```

4. Reload the firewall rules, maintaining the current state information:

	```
	sudo firewall-cmd --reload
	```

**Run the server**

```
sudo /etc/init.d/mysql start
sudo systemctl enable mariadb.service
mysql_secure_installation # password galera, everything else Y
```

File `initialize.sql`

```sql
CREATE DATABASE ICAT;
CREATE USER 'irods'@'localhost' IDENTIFIED BY 'temppassword';
GRANT ALL ON ICAT.* to 'irods'@'localhost';
SHOW GRANTS FOR 'irods'@'localhost';
```
Initialize database

```
mysql -uroot -pgalera < initialize.sql
```

### iRODS v.4.2.1

**Packages and Installation**

Create: `/etc/yum.repos.d/RENCI-iRODS.repo`

```config
# iRODS 4.2.1 packages list
# https://packages.irods.org
[renci-irods]
name=RENCI iRODS Repository
baseurl=https://packages.irods.org/yum/pool/centos$releasever/$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.irods.org/irods-signing-key.asc
```

```
sudo yum makecache fast -y
sudo yum install -y \
	irods-server-4.2.1 \
	irods-database-plugin-mysql-4.2.1
```

Update mysql-connector-odbc

```
sudo yum install -y wget
wget https://dev.mysql.com/get/Downloads/Connector-ODBC/5.3/mysql-connector-odbc-5.3.7-1.el7.x86_64.rpm
sudo yum --nogpgcheck localinstall -y mysql-connector-odbc-5.3.7-1.el7.x86_64.rpm
```

**Firewalld port configuration**

Reference: [www.firewalld.org](http://www.firewalld.org/documentation/)

1. Open the TCP ports for iRODS services:

	```
	sudo firewall-cmd --zone=public --add-port=1247/tcp
	sudo firewall-cmd --zone=public --add-port=1248/tcp
	sudo firewall-cmd --zone=public --add-port=20000-20199/tcp
	```

Make firewall changes persistent across reboots

1. Open the TCP ports for iRODS services:

	```
	sudo firewall-cmd --zone=public --permanent --add-port=1247/tcp
	sudo firewall-cmd --zone=public --permanent --add-port=1248/tcp
	sudo firewall-cmd --zone=public --permanent --add-port=20000-20199/tcp    
	```

2. Reload the firewall rules, maintaining the current state information:

	```
	sudo firewall-cmd --reload
	```

**my.cnf**

```
sudo /etc/init.d/mysql stop
```

Update `/etc/my.cnf`:

- Add `[mysqld]` section:

	```
	[mysqld]
	log_bin_trust_function_creators=1
	```

- Final file should look like:

	```
	$ sudo cat /etc/my.cnf
	#
	# This group is read both both by the client and the server
	# use it for options that affect everything
	#
	[client-server]
	
	#
	# include all files from the config directory
	#
	!includedir /etc/my.cnf.d
	[mysqld]
	log_bin_trust_function_creators=1
	```
	
```
sudo /etc/init.d/mysql start
```

**iRODS Configuration**

The `setup_irods.py` script will ask for information in four (possibly five) sections:

1. Service Account
	- Service Account Name: **IRODS\_SERVICE\_ACCOUNT\_NAME** 
	- Service Account Group: **IRODS\_SERVICE\_ACCOUNT\_GROUP**
	- Catalog Service Role: **IRODS\_SERVER\_ROLE** # 1. provider, 2. consumer
2. Database Connection (if installing a 'provider')
	- ODBC Driver: **ODBC\_DRIVER\_FOR\_MYSQL** # 1. MySQL, 2. MySQL ODBC 5.3 Unicode Driver, 3. MySQL ODBC 5.3 ANSI Driver
	- Database Server's Hostname or IP: **IRODS\_DATABASE\_SERVER\_HOSTNAME**
	- Database Server's Port: **IRODS\_DATABASE\_SERVER\_PORT**
	- Database Name: **IRODS\_DATABASE\_NAME** 
	- Database User: **IRODS\_DATABASE\_USER\_NAME**
	- Database Password: **IRODS\_DATABASE\_PASSWORD**
	- Stored Passwords Salt: **IRODS\_DATABASE\_USER\_PASSWORD\_SALT**
3. iRODS Server Options
	- Zone Name: **IRODS\_ZONE\_NAME**
	- Zone Port: **IRODS\_PORT**
	- Parallel Port Range (Begin): **IRODS\_PORT\_RANGE\_BEGIN**
	- Parallel Port Range (End): **IRODS\_PORT\_RANGE\_END**
	- Control Plane Port: **IRODS\_CONTROL\_PLANE\_PORT**
	- Schema Validation Base URI: **IRODS\_SCHEMA\_VALIDATION**
	- iRODS Administrator Username: **IRODS\_SERVER\_ADMINISTRATOR\_USER\_NAME**
4. Keys and Passwords
	- zone_key: **IRODS\_SERVER\_ZONE\_KEY**
	- negotiation_key: **IRODS\_SERVER\_NEGOTIATION\_KEY**
	- Control Plane Key: **IRODS\_CONTROL\_PLANE\_KEY**
	- iRODS Administrator Password: **IRODS\_SERVER\_ADMINISTRATOR\_PASSWORD**
5. Vault Directory: **IRODS\_VAULT\_DIRECTORY**


This information can be saved as a text file and provided to the `setup_irods.py` script at runtime.

Example `irods.config` file:

```
irods
irods
1
2
localhost
3306
ICAT
irods
yes
temppassword
tempsalt
tempZone
1247
20000
20199
1248
file:///var/lib/irods/configuration_schemas
rods
yes
TEMPORARY_zone_key
TEMPORARY_32byte_negotiation_key
TEMPORARY__32byte_ctrl_plane_key
rodspassword
/var/lib/irods/Vault
```

Symlink `mysql.sock` to correspond with expected location.

```
sudo ln -s /var/lib/mysql/mysql.sock /tmp/mysql.sock
```

Run iRODS setup script using `irods.config` file:

```
sudo python /var/lib/irods/scripts/setup_irods.py < irods.config
sudo systemctl enable irods
```

Dump the entire database as `db.sql`

```
mysqldump -uroot -pgalera --all-databases > db.sql
```

### Enable Galera Cluster

Ensure that SELinux is **disabled**

```
sudo /etc/init.d/mysql stop
```

Update `[galera]` settings in: `/etc/my.cnf.d/server.cnf`

```
[galera]
# Mandatory settings
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address='gcomm://192.168.58.101,192.168.58.102'
wsrep_cluster_name='galera'
wsrep_node_address='192.168.58.101'
wsrep_node_name='galera-1'
wsrep_sst_method=rsync

binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
```

Start the database with the `[galera]` configuration.

1. Execute `/usr/bin/mysqld_safe --wsrep-new-cluster` on the cluster's first master node
2. Bring up the other nodes in the cluster by executing
    - `sudo systemctl start mariadb.service`
    - `sudo systemctl start irods`
3. Execute `sudo pkill -SIGQUIT mysqld` on the master
4. Execute `sudo systemctl start mariadb.service` and `sudo systemctl start irods` on the master

After two node cluster is up it should show size = 2

```
$ mysql -uroot -pgalera -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 2     |
+--------------------+-------+
```


**NOTE**:

If the start command fails, check SELinux settings.

```
$ sudo systemctl start mariadb.service
Starting mysql (via systemctl):  Job for mariadb.service failed because the control process exited with error code. See "systemctl status mariadb.service" and "journalctl -xe" for details.
                                                           [FAILED]
```

Check SELinux (Want disabled)

```
$ sudo cat /etc/sysconfig/selinux

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

Update the `selinux` settings:

```
$ sudo vi /etc/sysconfig/selinux

## From: SELINUX=enforcing
## To:   SELINUX=disabled

$ sudo reboot
```

After reboot:

```
$ sudo systemctl start mariadb.service
[sudo] password for galera:
Starting mysql (via systemctl):                            [  OK  ]
```

For error checking: `journalctl -u mariadb`



---

### Notes

Expected iRODS installation output:

```
$ sudo python /var/lib/irods/scripts/setup_irods.py < irods.config
Warning: Hostname `localhost` should be a fully qualified domain name.
The iRODS service account name needs to be defined.
iRODS user [irods]:
iRODS group [irods]:

+--------------------------------+
| Setting up the service account |
+--------------------------------+

Existing Group Detected: irods
Existing Account Detected: irods
Setting owner of /var/lib/irods to irods:irods
Setting owner of /etc/irods to irods:irods
iRODS server's role:
1. provider
2. consumer
Please select a number or choose 0 to enter a new value [1]:
Updating /etc/irods/server_config.json...

+-----------------------------------------+
| Configuring the database communications |
+-----------------------------------------+

You are configuring an iRODS database plugin. The iRODS server cannot be started until its database has been properly configured.

ODBC driver for mysql:
1. MySQL
2. MySQL ODBC 5.3 Unicode Driver
3. MySQL ODBC 5.3 ANSI Driver
Please select a number or choose 0 to enter a new value [1]:
Database server's hostname or IP address [localhost]:
Database server's port [3306]:
Database name [ICAT]:
Database username [irods]:

-------------------------------------------
Database Type: mysql
ODBC Driver:   MySQL ODBC 5.3 Unicode Driver
Database Host: localhost
Database Port: 3306
Database Name: ICAT
Database User: irods
-------------------------------------------

Please confirm [yes]:
Warning: Cannot control echo output on the terminal (stdin is not a tty). Input may be echoed.

Updating /etc/irods/server_config.json...
Listing database tables...
Warning: Cannot control echo output on the terminal (stdin is not a tty). Input may be echoed.

Updating /etc/irods/server_config.json...

+--------------------------------+
| Configuring the server options |
+--------------------------------+

iRODS server's zone name [tempZone]:
iRODS server's port [1247]:
iRODS port range (begin) [20000]:
iRODS port range (end) [20199]:
Control Plane port [1248]:
Schema Validation Base URI (or off) [file:///var/lib/irods/configuration_schemas]:
iRODS server's administrator username [rods]:

-------------------------------------------
Zone name:                  tempZone
iRODS server port:          1247
iRODS port range (begin):   20000
iRODS port range (end):     20199
Control plane port:         1248
Schema validation base URI: file:///var/lib/irods/configuration_schemas
iRODS server administrator: rods
-------------------------------------------

Please confirm [yes]:
Warning: Cannot control echo output on the terminal (stdin is not a tty). Input may be echoed.

Warning: Cannot control echo output on the terminal (stdin is not a tty). Input may be echoed.

Warning: Cannot control echo output on the terminal (stdin is not a tty). Input may be echoed.

Updating /etc/irods/server_config.json...

+-----------------------------------+
| Setting up the client environment |
+-----------------------------------+


Warning: Cannot control echo output on the terminal (stdin is not a tty). Input may be echoed.


Updating /var/lib/irods/.irods/irods_environment.json...

+--------------------------+
| Setting up default vault |
+--------------------------+

iRODS Vault directory [/var/lib/irods/Vault]:

+-------------------------+
| Setting up the database |
+-------------------------+

Listing database tables...
Defining mysql functions...
Creating database tables...

+-------------------+
| Starting iRODS... |
+-------------------+

Validating [/var/lib/irods/.irods/irods_environment.json]... Success
Validating [/var/lib/irods/VERSION.json]... Success
Validating [/etc/irods/server_config.json]... Success
Validating [/etc/irods/host_access_control_config.json]... Success
Validating [/etc/irods/hosts_config.json]... Success
Ensuring catalog schema is up-to-date...
Updating to schema version 2...
Updating to schema version 3...
Updating to schema version 4...
Updating to schema version 5...
Catalog schema is up-to-date.
Starting iRODS server...
Success

+---------------------+
| Attempting test put |
+---------------------+

Putting the test file into iRODS...
Getting the test file from iRODS...
Removing the test file from iRODS...
Success.

+--------------------------------+
| iRODS is installed and running |
+--------------------------------+
```

```
$ sudo su - irods
$ ienv
irods_version - 4.2.1
irods_server_control_plane_encryption_algorithm - AES-256-CBC
schema_name - irods_environment
irods_transfer_buffer_size_for_parallel_transfer_in_megabytes - 4
irods_host - localhost
irods_zone_name - tempZone
irods_user_name - rods
irods_server_control_plane_encryption_num_hash_rounds - 16
irods_session_environment_file - /var/lib/irods/.irods/irods_environment.json.10952
irods_port - 1247
irods_match_hash_policy - compatible
irods_home - /tempZone/home/rods
irods_default_resource - demoResc
irods_encryption_num_hash_rounds - 16
irods_maximum_size_for_single_buffer_in_megabytes - 32
irods_encryption_algorithm - AES-256-CBC
irods_cwd - /tempZone/home/rods
irods_client_server_policy - CS_NEG_REFUSE
irods_default_hash_scheme - SHA256
irods_encryption_key_size - 32
irods_server_control_plane_key - TEMPORARY__32byte_ctrl_plane_key
irods_client_server_negotiation - request_server_negotiation
irods_server_control_plane_port - 1248
schema_version - v3
irods_encryption_salt_size - 8
irods_environment_file - /var/lib/irods/.irods/irods_environment.json
irods_default_number_of_transfer_threads - 4
```

### Items that need to be the same across all deployments

1. iRODS database user and password, shown as **'irods'@'localhost'** and **'temppassword'** below

    ```
    CREATE DATABASE ICAT;
    CREATE USER 'irods'@'localhost' IDENTIFIED BY 'temppassword';
    GRANT ALL ON ICAT.* to 'irods'@'localhost';
    SHOW GRANTS FOR 'irods'@'localhost';
    ```
2. iRODS configuration

    ```
    irods
    irods
    1
    2
    localhost
    3306
    ICAT
    irods
    yes
    temppassword
    tempsalt
    tempZone
    1247
    20000
    20199
    1248
    file:///var/lib/irods/configuration_schemas
    rods
    yes
    TEMPORARY_zone_key
    TEMPORARY_32byte_negotiation_key
    TEMPORARY__32byte_ctrl_plane_key
    rodspassword
    /var/lib/irods/Vault
    ```
