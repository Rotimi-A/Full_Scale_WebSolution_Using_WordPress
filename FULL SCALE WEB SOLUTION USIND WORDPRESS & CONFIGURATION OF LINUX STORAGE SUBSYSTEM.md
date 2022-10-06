# FULL SCALE WEB SOLUTION USING WORDPRESS & CONFIGURATION OF LINUX STORAGE SUBSYSTEM

## STEP 1A - PREPARATION OF OUR WEB AND DATABASE SERVERS

WE WILL LAUNCH 2 REDHAT EC2 INSTANCES ON AWS:
1. THE WEBSERVER
2. THE DATABASE SERVER

<img width="965" alt="Screenshot 2022-10-04 at 11 16 05" src="https://user-images.githubusercontent.com/111234300/194236889-710c9a5e-7068-4b3a-b2c1-95c9cdf8c576.png">

<img width="1282" alt="Screenshot 2022-10-05 at 09 22 15" src="https://user-images.githubusercontent.com/111234300/194235999-d83258cc-bb15-4fe7-be4f-260fbdcd8fd6.png">

THEN WE WILL PROCEED TO CREATE VOLUMES IN THE SAME AVAILABILTY ZONES FOR OUR SERVERS 

<img width="1257" alt="Screenshot 2022-10-05 at 09 22 35" src="https://user-images.githubusercontent.com/111234300/194236337-448e0719-546e-4c80-b6d3-371eb0485a65.png">

AFTER THIS, WE WILL ATTACH EACH VOLUMES INDIVIDUALLY TO OUR INSTANCES 

<img width="748" alt="Screenshot 2022-10-04 at 11 32 16" src="https://user-images.githubusercontent.com/111234300/194237838-100bff61-6201-456a-bba6-a609c231bd12.png">

TIME TO OPEN THE LINUX TERMINAL AND BEGIN CONFIGURATION

USE `lsblk` commmand to inspect what block devices are attached to the server
<img width="428" alt="Screenshot 2022-10-04 at 11 34 31" src="https://user-images.githubusercontent.com/111234300/194240786-2224aa6e-abdf-4bf9-b18e-482329d44ffd.png">

All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there – their names will likely be xvdf, xvdh, xvdg.

NEXT WE USE THE `gdisk` TO CREATE A PARTITION ON EACH OF THE 3 DISKS

`sudo gdisk /dev/xvdf`

<img width="802" alt="Screenshot 2022-10-04 at 11 51 43" src="https://user-images.githubusercontent.com/111234300/194242058-74a4f8e5-c3ad-4e2c-b42a-3ff1eeedb24a.png">

ONCE THE PARTITION HAS BEEN SUCCESSFULLY CREATED, WE PARTITION WOULD APPEAR LIKE THIS using `lsblk`

<img width="419" alt="Screenshot 2022-10-04 at 11 56 27" src="https://user-images.githubusercontent.com/111234300/194245549-644d604c-ae00-4d23-a22a-e29fc03bbd35.png">

ON REDHAT CENTOS WE USE `yum` INSTEAD OF `apt` COMMAND TO INSTALL OUR PACKAGES.

So, Install `lvm2 package using sudo yum install lvm2`  Then run `sudo lvmdiskscan` command to check for available partitions.

Subsequently, Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

<img width="856" alt="Screenshot 2022-10-04 at 12 06 00" src="https://user-images.githubusercontent.com/111234300/194246688-af94bfa9-bf3c-4088-8574-d504576714de.png">

Verify that your Physical volume has been created successfully by running sudo pvs

<img width="832" alt="Screenshot 2022-10-04 at 12 09 29" src="https://user-images.githubusercontent.com/111234300/194246928-1d7f3f08-cd6d-4cc1-bc4a-ee52f5e3c1d1.png">

Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg
`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

<img width="832" alt="Screenshot 2022-10-04 at 12 09 29" src="https://user-images.githubusercontent.com/111234300/194247329-4a2420d3-b684-4c46-a865-112ce69beaae.png">

Verify that your VG has been created successfully by running sudo vgs

<img width="780" alt="Screenshot 2022-10-04 at 12 09 58" src="https://user-images.githubusercontent.com/111234300/194248474-b968d374-9a8f-48da-a613-69d220063406.png">

Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.
`sudo lvcreate -n apps-lv -L 14G webdata-vg`
`sudo lvcreate -n logs-lv -L 14G webdata-vg`

Verify that your Logical Volume has been created successfully by running `sudo lvs`

<img width="844" alt="Screenshot 2022-10-04 at 13 49 33" src="https://user-images.githubusercontent.com/111234300/194249078-f2696d45-b501-4a4d-81e2-b4d147aba981.png">

NOW, it's time we verify the entire setup using these codes:

`sudo vgdisplay -v #view complete setup - VG, PV, and LV`
`sudo lsblk`

Next, Use mkfs.ext4 to format the logical volumes with ext4 filesystem:

`sudo mkfs -t ext4 /dev/webdata-vg/apps-lv`
`sudo mkfs -t ext4 /dev/webdata-vg/logs-lv`

Create /var/www/html directory to store website files

`sudo mkdir -p /var/www/html`

Create /home/recovery/logs to store backup of log data

`sudo mkdir -p /home/recovery/logs`

Mount /var/www/html on apps-lv logical volume

`sudo mount /dev/webdata-vg/apps-lv /var/www/html/`

<img width="706" alt="Screenshot 2022-10-04 at 12 41 44" src="https://user-images.githubusercontent.com/111234300/194258134-381f2b72-fc06-4baf-8bcc-cdbf06dd3287.png">

Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)

`sudo rsync -av /var/log/. /home/recovery/logs/`

<img width="703" alt="Screenshot 2022-10-04 at 12 45 44" src="https://user-images.githubusercontent.com/111234300/194257905-6ce6a682-74d2-4b20-8473-c248f1e3db23.png">

Mount /var/log on logs-lv logical volume. (Note that all the existing data on /var/log will be deleted. That is why step 15 above is very
important)

`sudo mount /dev/webdata-vg/logs-lv /var/log`

Restore log files back into /var/log directory

`sudo rsync -av /home/recovery/logs/. /var/log`

Update /etc/fstab file so that the mount configuration will persist after restart of the server.
 and test the configuration and reload the daemon
`sudo mount -a`
`sudo systemctl daemon-reload`

<img width="601" alt="Screenshot 2022-10-04 at 15 01 31" src="https://user-images.githubusercontent.com/111234300/194258501-14ca06e3-850a-4d2e-8521-e5310bc2cad3.png">

## STEP 1B - WE REPEAT THE SAME PROCESS FOR THE DATABASE SERVER

Now, we repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it to /db directory instead of /var/www/html/.

<img width="784" alt="Screenshot 2022-10-04 at 14 37 51" src="https://user-images.githubusercontent.com/111234300/194262245-d248a768-483d-47aa-88cc-27af08fb4169.png">

## STEP 2 - INSTALL WORDPRESS ON OUR WEB SERVER EC2

WordPress is a free and open-source content management system written in PHP and paired with MySQL or MariaDB (we use MuSQL for this project) as its backend Relational Database Management System (RDBMS).

WordPress requires the latest version of php to work, so at the time of this documentation, that is version 8.0.24.

So, let's proceed with installing PHP with these codes:

Update the repository

`sudo yum -y update`

Install wget, Apache and it’s dependencies

`sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

Start Apache

`sudo systemctl enable httpd`
`sudo systemctl start httpd`

To install PHP and it’s dependencies

`sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm`

`sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm`

`sudo yum module list php`

`sudo yum module reset php`

`sudo yum module enable php:remi-8.0`

<img width="1018" alt="Screenshot 2022-10-04 at 15 34 06" src="https://user-images.githubusercontent.com/111234300/194265764-2762f654-7d2e-4e13-bdd6-61e7e7a61331.png">

`sudo yum install php php-opcache php-gd php-curl php-mysqlnd`

`sudo systemctl start php-fpm`

`sudo systemctl enable php-fpm`

`setsebool -P httpd_execmem 1`

Now, after running all these codes, We Restart Apache

`sudo systemctl restart httpd`

TIME TO DOWNLOAD WORDPRESS AND INSTALL IT IN THE REQUIRED DIRECTORY var/www/html

 `mkdir wordpress`
 
 `cd   wordpress`
 
 `sudo wget http://wordpress.org/latest.tar.gz`
 
 `sudo tar xzvf latest.tar.gz`
 
 `sudo rm -rf latest.tar.gz`
 
 `cp wordpress/wp-config-sample.php wordpress/wp-config.php`
 
 `cp -R wordpress /var/www/html/`
 
 
 NOW WE RUN SELinux Policies to enable apache have access to the wordpress file by changing ownership and SElinux context of our WordPress from root to apache
 
 <img width="1054" alt="Screenshot 2022-10-04 at 16 43 10" src="https://user-images.githubusercontent.com/111234300/194268011-3a67e781-d1de-4f39-b192-afea40547acc.png">
 
  `sudo chown -R apache:apache /var/www/html/wordpress`
  
  `sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R`
  
  `sudo setsebool -P httpd_can_network_connect=1`
  
  WITH HOW FAR WE'VE GONE, SINCE WE HAVE'T ALTERED OUR DEFAULT APACHE PAGE,
  WE CAN ACCESS IT WITH OUR PUBLIC <IPV4 DNS> OR PUBLIC EC2 INSTANCE <IP addr> AND PUTTING IT IN OUR WEB BROWSER
  
  <img width="1440" alt="Screenshot 2022-10-05 at 05 50 50" src="https://user-images.githubusercontent.com/111234300/194270625-80a1077b-5f68-4de0-b4f9-ef41ddbbeac4.png">
<img width="1440" alt="Screenshot 2022-10-05 at 05 50 59" src="https://user-images.githubusercontent.com/111234300/194270702-bcf519a1-4d8b-4404-be61-1007edf1ff66.png">
  
  GREAT! 👍 
  SO, NOW WE MOVE OVER TO THE NEXT STEP
  
## STEP 3 - INSTALL MYSQL ON OUR DB SERVER EC2
  
  
`sudo yum update`
  
`sudo yum install mysql-server`
  
Verify that the service is up and running by using sudo systemctl status mysqld, if it is not running, restart the service and enable it so it will be running even after reboot:

`sudo systemctl restart mysqld`
  
`sudo systemctl enable mysqld`
  
  <img width="812" alt="Screenshot 2022-10-04 at 16 53 25" src="https://user-images.githubusercontent.com/111234300/194273599-f8ebf6c7-8666-4160-b9dd-9d108320a8be.png">
  

## STEP 4 — CONFIGURE DB TO WORK WITH WORDPRESS
  
`sudo mysql_secure_installation`
  
  <img width="728" alt="Screenshot 2022-10-04 at 16 57 20" src="https://user-images.githubusercontent.com/111234300/194274190-a91c158e-aec2-453e-b60f-c90e90365c74.png">

 Then connect to MySQL using `sudo mysql -u root -p`
  
  <img width="710" alt="Screenshot 2022-10-04 at 16 59 12" src="https://user-images.githubusercontent.com/111234300/194274347-e182e6af-2f15-450e-9a8f-70afd3cba293.png">
  
`CREATE DATABASE wordpress;`
  
`CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';`
  
`GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';`
  
  <img width="768" alt="Screenshot 2022-10-04 at 17 17 13" src="https://user-images.githubusercontent.com/111234300/194275361-271cef54-2302-4253-8c67-32b995d86699.png">
  
  
`FLUSH PRIVILEGES;`
  
`SHOW DATABASES;`
  
<img width="329" alt="Screenshot 2022-10-04 at 17 00 58" src="https://user-images.githubusercontent.com/111234300/194274734-0d40257c-91ba-4b71-a36d-656789575409.png">
<img width="323" alt="Screenshot 2022-10-04 at 17 17 04" src="https://user-images.githubusercontent.com/111234300/194274742-518c564c-a3ee-4789-b77a-8e2b7553c4ea.png">
  

  Then, exit MySQL
  
 
  
  STEP 5 - CONFIGURE WORDPRESS TO CONNECT TO REMOTE DATABASE
 This is done by allowing access through our security group on AWS by editing the inbound rule and to open MySQL port 3306 with our specific's server IP address
  
 Now, we install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client

  `sudo yum install mysql`
`sudo mysql -u admin -p -h <DB-Server-Private-IP-address>`
  
  <img width="883" alt="Screenshot 2022-10-05 at 08 40 56 copy" src="https://user-images.githubusercontent.com/111234300/194277604-e8705cb8-87ef-41bb-b34c-ec1a4a36c3c2.png">
  
Verify if you can successfully execute SHOW DATABASES; command and see a list of existing databases.
  
  <img width="788" alt="Screenshot 2022-10-05 at 08 48 06" src="https://user-images.githubusercontent.com/111234300/194277735-848fbe33-5625-4c9e-8606-d3e51890b64a.png">
  
WE ARE ALMOST THERE, BUT WE NEED TO EDIT APACHE DEFAULT HOMEPAGE AND EDIT OUR WP CONFIG FILE, so that apache can use WordPress
  
 <img width="903" alt="Screenshot 2022-10-05 at 08 40 56" src="https://user-images.githubusercontent.com/111234300/194280390-b478e414-b1a1-4582-8275-f25f3351ea3d.png">
  
  
  Try to access from your browser the link to your WordPress http://<Web-Server-Public-IP-Address>/wordpress/
  
 AND VOILA!
  <img width="1440" alt="Screenshot 2022-10-05 at 08 51 51" src="https://user-images.githubusercontent.com/111234300/194278523-0cbf3689-3aaa-492a-b818-ea67391b295a.png">
<img width="1440" alt="Screenshot 2022-10-05 at 08 55 35" src="https://user-images.githubusercontent.com/111234300/194278535-6a55ee65-db20-4bf6-9a38-f7bb2504b4bb.png">
<img width="1440" alt="Screenshot 2022-10-05 at 08 56 02" src="https://user-images.githubusercontent.com/111234300/194278549-cb601a24-e43c-43a7-9caa-a7d4ed46e92f.png">
<img width="1440" alt="Screenshot 2022-10-05 at 08 56 23" src="https://user-images.githubusercontent.com/111234300/194278556-5480f993-0b30-4107-8bfd-31c8e4cb3d75.png">
<img width="1440" alt="Screenshot 2022-10-05 at 08 57 11" src="https://user-images.githubusercontent.com/111234300/194278563-9d2255a2-d304-4ea6-91ba-8fd3c3759c05.png">
<img width="1440" alt="Screenshot 2022-10-05 at 08 57 26" src="https://user-images.githubusercontent.com/111234300/194278568-50f7d82f-7c50-4a41-886d-0f372a06550f.png">
<img width="1440" alt="Screenshot 2022-10-05 at 08 59 16" src="https://user-images.githubusercontent.com/111234300/194278574-a130b6ab-eb57-44e7-9494-70e9d0bf4db3.png">

We have successfully configured Linux storage susbystem and have also deployed a full-scale Web Solution using WordPress CMS and MySQL RDBMS!
 
 














