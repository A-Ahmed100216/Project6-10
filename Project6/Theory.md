# Project 6 - Web solution with Wordpress

## Introduction
Project 6 consists of 2 parts:
1. Configure storage subsystem for Web and Database servers based on Linux OS.
2. Install WordPress and connect it to a remote MySQL database server.
### 3 Tier Architecture
* Web and mobile solutions typically use a three-tier architecture 
* A three-tier architecture consists of the following components:
1. Presentation Tier - Client Server - User Interface
2. Application Tier - Webserver - Backend implements the logic
3. Data Tier - Database Server - Data storage and access


## Project Implementation 
### 1. Prepare Webserver
1. Launch an EC2 Instance - use RedHat. This instance will serve as the webserver.
2. Create 3 volumes, each of 10 GiB, ensuring they have been created in the same AZ as the instance.
![Create volumes](/Project6/images/create_volumes.png)   
3. Attach the volumes to the webserver instance.
![Attach volumes](/Project6/images/attach_volumes.png)   
4. Connect to the instance 
5. Use the command `lsblk` to inspect block devices attached. The attached volumes should be be displayed, alongside any other block devices.   
![lsblk](/Project6/images/lsblk.png)
6. All devices are located in the /dev/ directory. Confirm the block devices are in this directory.
![ls /dev/](/Project6/images/lsdev.png)
7. Use the command `df -h` to view mounts and free space
![df -h command](/Project6/images/df.png)
8. Create a single partition using the `gdisk` utility:   
    a. `sudo gdisk /dev/xvdf`   
    b. Add a partition - `n`      
    c. Hit enter to use default values for all except the HEX code `8e00`     
    d. View partition table - `p`    
    e. Write the table to disk and exit - `w`     
    f. Repeat for remaining disks `sudo gdisk /dev/xvdg` `sudo gdisk /dev/xvdh`     
9. Confirm partitions by running `lsblk`      
![lsblk after partitions](/Project6/images/lsblk_with_partitions.png)
10. Install lvm2 package via yum, the package manager for RedHat/CentOS.
```bash
sudo yum install lvm2
```
![install lvm2](/Project6/images/install_lvm2.png)        
11. Run `sudo lvmdiskscan` to check for available partitions.
![lvmdiskscan](/Project6/images/lvmdiskscan.png)       
12. Use `pvcreate` to mark disks as physical volumes (PVs) to be used by LVM
```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
13. Confirm the creation of the physical volumes `sudo pvs`
![pvcreate](/Project6/images/pvcreate.png)       
14. Use `vgcreate` utility to add all 3 physical volumes to a volume group (VG). Name this VG `webdata-vg` and verify the creation.
```
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
sudo vgs
```
![vgcreate](/Project6/images/vgcreate.png)
15. Next create two logical volumes each using half the size of the physical volume (14G). One will be called `apps-lv` which will store website data. The other,`logs-lv` will store log data. 
```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
16. Verify creation of logical volumes by running `sudo lvs`
![lvcreate](/Project6/images/lvcreate.png)
17. View the entire setup 
```
sudo vgdisplay -v
sudo lsblk
```
![setup web](/Project6/images/full_setup_web.png)
18. Use `mkfs.ext4` to format the logical volumes with ext4 filesystem. ext4 is a successor of previous Linux filesystems (ext3) with a focus on improving performance, reliability, and capacity. 
```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
![ext4](/Project6/images/ext4.png)
19. Next, create a directory to store the website files in /var/www/html
```
sudo mkdir -p /var/www/html
```
20. Similarly, create a directory to store backup logs in /home/recovery/logs
```
sudo mkdir -p /home/recovery/logs
```
![directories](/Project6/images/directories.png)
21. Mount `/var/www/html` on the `apps-lv` logical volume
```
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
22. Use the `rsync` utility to backup all the files in the log directory `/var/log` into `/home/recovery/logs`. This is required before mounting as the following step deletes all existing data in `/var/log`
```
sudo rsync -av /var/log/. /home/recovery/logs/
```
![mount app](/Project6/images/mount_app.png)
23. Mount `/var/log` onto the `logs-lv` logical volume. 
```
sudo mount /dev/webdata-vg/logs-lv /var/log
```
24. Restore log files back into /var/log directory
```
sudo rsync -av /home/recovery/logs/. /var/log
```
![mount logs](/Project6/images/mount_log.png)
25. In order to retain the mount configuration after restarting the server, update /etc/fstab file. 

### 2. Update /etc/fstab file
1. Obtain the Universally Unique Identifier (UUID) of the device 
```
sudo blkid
```
![UUID](/Project6/images/UUID.png)
2. Open the /etc/fstab file and add the mounts for the webserver (see image for format)
```
sudo vi /etc/fstab
```
![update etc/fstab](/Project6/images/update_etc_fstab.png)
3. Exit and reload the daemon 
```
sudo mount -a sudo systemctl daemon-reload
```
4. Verify setup by running `df -h`
![reload and verify](/Project6/images/reload_daemon.png)

### 3. Prepare the Database Server
1. Launch another EC2 instance (also RedHat). This will serve as the db server. 
2. Repeat steps from section 1 - Prepare Webserver but instead of `apps-lv`,create `db-lv` and mount to `/db`
![db initial partitions](/Project6/images/db_partitions.png)
![db lvs](/Project6/images/db_lvs.png)
![db setup](/Project6/images/db_setup.png)

### 4. Install Wordpress on your Webserver
1. Install Apache and its dependencies as well as wget, a tool used to retrieve web content. 
```bash
# Update the repository
sudo yum -y update

# Install wget, Apache and its dependencies
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
2. Use the `systemctl` utility to enable and start Apache
```
sudo systemctl enable httpd 
sudo systemctl start httpd
```
3. Install PHP and its dependencies
```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm 
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm 
sudo yum module list php 
sudo yum module reset php 
sudo yum module enable php:remi-7.4 
sudo yum install php php-opcache php-gd php-curl php-mysqlnd 
sudo systemctl start php-fpm 
sudo systemctl enable php-fpm 
sudo setsebool -P httpd_execmem 1
```
4. Restart Apache
```bash
sudo systemctl restart httpd
```
5. Download wordpress and copy wordpress to var/www/html 
```bash
mkdir wordpress 
cd wordpress 
sudo wget http://wordpress.org/latest.tar.gz 
sudo tar xzvf latest.tar.gz 
sudo rm -rf latest.tar.gz 
cp wordpress/wp-config-sample.php wordpress/wp-config.php cp -R wordpress /var/www/html/
# The last command may not work if wp-congif.php is not present. In this case, use the command below:
cp wordpress/wp-config-sample.php  cp -R wordpress /var/www/html/
```
6. Configure SELinux Policies - a set of rules which guide the SeLinux Security Engine by defining types of file objects and domains for processes. 
```bash
# Make Apache the owner
sudo chown -R apache:apache /var/www/html/wordpress 
# Change SELinux context 
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R 
# Modify SELinux bool value
sudo setsebool -P httpd_can_network_connect=1
```

### 5. Install MySQL on DB Server
1. Install `mysql-server`
```
sudo yum update
sudo yum install mysql-server
```
2. Confirm Service is running, if not, restart the service and enable it. 
```bash
sudo systemctl status mysqld

# If not running, restart service and enable. 
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
![mysql status](/Project6/images/mysql_status.png)

3. Configure the DB to work with Wordpress
```bash
sudo mysql
```
```sql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```
![configure db](/Project6/images/db_config.png)

### 6. Configure WordPress to connect to remote database.
1. Amend Security Groups 
    a. DB Instance - Open port 3306 (MySQL/Aurora) to the Web Server's IP address. 
    b. Webserver Instance - Allow all incoming connections to port 80 (HTTP)
![webserver sg](/Project6/images/webserver_sg.png)
2. Navigate to DB Instance and install `mysql-client`
```bash
sudo yum install mysql
sudo mysql -u myuser -p -h <DB-Server-Private-IP-address>
```
3. Confirm connection my running the command `SHOW DATABASES;`. The wordpress database should be displayed.
![mysql connect via client](/Project6/images/mysql_client.png)

4. Navigate to a browser and access the following url `http://<Web-Server-Public-IP-Address>/wordpress/`. 
5. Click `Let's Go` and fill in your DB connection details 
![db connection](/Project6/images/db_connection.png)
6. Provided the connection is successful, the following page should be displayed. 
![final webpage](/Project6/images/final_webpage.png)
7. **Terminate instances and delete volumes to avoid incurring extra charges.**



