# IMPLEMENTING WORDPRESS WEBSITE WITH LVM STORAGE

## Step 1

firstly i created two EC2 `REDHAT` server instances on aws one will be the `Webserver` and the other will be the `Dataserver`

![instances created](<img/1 instance created.png>)

#### Working on the webserver EC2 instance

i created three EBS volumes on EC2 instance following the steps on the image below

![volume 1](<img/2 select volume.png>)

![volume 2](<img/3 click create volume.png>)

![volume 3](<img/4 three new volumes created.png>)

next i attached the new folumes to the webserver EC2 instance using the pictorial steps below

![attach 1](<img/5 attach volume 1.png>)

![attach 2](<img/6 first attachment.png>)

![attach 3](<img/7 second attachment.png>)

<br />

Next i ssh into my webserver EC2 instance 

![ssh into EC2 instance](<img/8 connect to redhat instance.png>)

To identify what block devices that are attached to the server i used the command below

```
lsblk
```
![available blocks](<img/9 available volumes in the server.png>)
 
from the image above i have the following devices attached 
`xvdf`

`xvdg`

`xvdh`

### take note of the devices listed above 


to know the available mount point i used the command below

```
df -h
```
![mount points available](<img/11 list devices.png>)

for the next step i will have to create a partition and to do that i used the command below 

```
sudo gdisk /dev/xvdf
```

`xvdf` is one of the devices attached as stated earlier

![xvdf partition](<img/15 xvdf partition.png>)

![xvdf partition](<img/16 xvdf partition.png>)

___STEPS FOLLOWED IN THE IMAGE ABOVE___

i typed `n` to create a new partition next i gave a partition number `1` (you can choose to give any number) 

next i did not type any number because i wanted to use all the allocated 10G of available storage 

next i chose the hex code `8e00` to change the partition type to `Linux LVM` which is `linux logical volume management`

afterward i typed `p` confirm what i did afterwhich i typed `w` to write and type `y` to confirm.

<br />

Next i repeated the partition step above for `xvdg` and `xvdh` as seen in the images below

![xvdg](<img/17 xvdg repeated the same steps above.png>)

![xvdh](<img/18 xvdh repeated the same steps as done for xvdf.png>)

<br />
to see the new logical volume created i used the code below
<br />

```
lsblk
```
the new logical volumes are 

`xvdf1`

`avdg1`

`xvdh1`


![new lvm created](<img/19 new partition listed.png>)


next i installed `lvm2` using the code below

```
sudo yum install lvm2
```

![install lvm2 package](<img/20 sudo yum install lvm2.png>)

to confirm `lvm2` is installed i use the command below

```
which lvm
```

![confirm lvm is installed](<img/21 confirmation that lvm is installed.png>)


Next i need to create a physical volume for each partition to do that i used the `pvcreate` utility to mark each of 3 disks as physical volumes to be used by Logical Volume Management to do this i used the command below

```
sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
```

![using pvcreate](<img/23 create physical volume.png>)

to confirm the creation of PV i used the command below

```
sudo pvs
```

![confirm pv has been created](<img/24 confirm physical volume has been created.png>)


next i added all the PV to a volume group using the command below

```
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
```

![create volume group](<img/25 volume group created.png>)

The name of the volume group created is `webdata-vg`


To confirm the creation of volume groups i used the command below

```
sudo vgs
```

![confirm volume group is created](<img/26 confirm volume group is created.png>)


next i created two(2) logical volume named

`apps-lv`

`logs-lv`

using the commands below

```
sudo lvcreate -n apps-lv -L 14G webdata-vg
```
![logical volume apps-lv](<img/27 logical volume for apps-lv created.png>)


```
sudo lvcreate -n logs-lv -L 15G webdata-vg
```

![logical volume logs-lv](<img/28 logical volume for logs-lv created.png>)


To verify that the logical volume is created i use the command below

```
sudo lvs
```

![confirm logical volume](<img/29 confirm two logical volume has been created.png>)


next i formated the logical volume with `ext4` filesystem using the command below

```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
```

![ext4 file system formating on apps-lv](<img/30 file system fomat apps-lv ext4.png>)


```
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

![ext4 file system formating on logs-lv](<img/31 file system fomat logs-lv ext4.png>)


the next step is to create various mount points. it is important to note that when a mount point is created the destination of the mount point is formated hence it is a good practice to first check if the location contains any files and back it up if there is any to avoid any issues. 

Mount point `apps-lv` does not contain any files hence no need for backup but `logs-lv` contains important files hence the need to back it up first mount on it and thereafter restore the backed up files back to its original location

next i created a `html` directory at `var/www/html` using the command below

```
sudo mkdir -p /var/www/html
```

![html directory created](<img/32 create html directory.png>)

next i created a logs directory that will be used to back up my important log files using the command 

```
sudo mkdir -p /home/recovery/logs
```

![create logs backup directory](<img/33 create logs directory.png>)


next i mount `/var/www/html` onto `apps-lv` logical volume using the command below

```
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
![mount unto apps-lv](<img/34 mount apps-lv onto html.png>)


next i backed up the files in the log directory `/var/log` unto `/home/recovery/logs` using the `rsync` utility with the command below

```
sudo rsync -av /var/log/. /home/recovery/logs/
```

![backing up log files](<img/37 backing up of log files using rsync command.png>)


next i mount `/var/log` onto `logs-lv` using the command below

```
sudo mount /dev/webdata-vg/logs-lv /var/log
```

![mount log onto logs-lv](<img/38 mount logs-lv onto log.png>)


next i restore the backed up log files back into `/var/log` using the command below

```
sudo rsync -av /home/recovery/logs/log/. /var/log
```

![restore baced up log files back](<img/39 copy log files back from backup folder.png>)

to confirm the log files are restored back to `/var/log` i use the command below

```
sudo ls -l /var/log
```

![confirm restored logs](<img/40 confirm that files are copied back.png>)

<br />

next i update `/etc/fstab` file to ensure the mount configuraion will persist after a restart of the server. 

before updating `/etc/fstab` file i have to get the `UUID` number of `apps-lv` and `logs-lv`.

to get the number type the command below

```
sudo blkid
```

![blkid number](<img/41 sudo blkid for webdata.png>)

next i copy the hilighted numbers and copy them into the `/etc/fstab` file

To open the `/etc/fstab` file i use the command below

```
sudo vi /etc/fstab
```

![edit fstab file](<img/42 edit fstab using vi.png>)

next i edit the file as shown below

![fstab uuid edit](<img/43 uuid for webserver.png>)


to ensure my configuration on `/etc/fstab` good i run the command below. 

If i get an error i will go back and correct it. if no error then the configuration is good.

```
sudo mount -a
```

![check fstab configuration is ok](<img/44 sudo mount a check if all is alright.png>)


next i reload the configuration using the command below

```
sudo systemctl daemon-reload
```

![reload the daemon](<img/45 sudo systemctl daemon update fstab.png>)

to verify my setup i run the command below

```
df -h
```

![verify running setup](<img/46 verify my setup is running.png>)


### INSTALLING WORDPRESS AND CONFIGURING TO USE MYSQL DATABASE

###  STEP 2

#### WORKING ON THE DATABASE EC2 SERVER

i repeated the process above on the database server with some little changes which are

> instead of `apps-lv` i created `db-lv`

![dbvl](<img/48 create db-lv logical database.png>)

> instead of `/var/www/html` i created `/db` directory

![db directory](<img/49 create a mount point db.png>)

> made it an ext4 file system

![file system](<img/50 before mounting make it a file system first.png>)

> i mounted `db-lv` unto `/db` directory and also edit `/etc/fstab` configuration file

![mount](<img/51 mount and afterward get uuid.png>)


### STEP 3 - INSTALL WORDPRESS ON MY WEBSERVER

#### ON THE WEBSERVER EC2

first i updated my repository using the command below

```
sudo yum -y update
```

![update repository](<img/52 update my repository installing wordpress step.png>)

next  i installed `mysql server` using the command below

next i typed the following commands

```
sudo systemctl start mysqld
```

```
sudo systemctl enable mysqld

```

```
sudo systemctl status mysqld
```


```
sudo yum install mysql-server
```

next i installed `wget`, `apache` and their dependencies using the command below

```
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```

![install wget and htppd](<img/53 install mysql php etc.png>)

next i started Apache using the command below

```
sudo systemctl enable httpd
```

```
sudo systemctl start httpd
```

![enable apache](<img/54 enable and start httpd.png>)


to verify apache is running i used the command below

```
sudo systemctl status httpd 
```

![confirm httpd is running](<img/55 connfirm httpd is running.png>)


next i installed PHP and it's dependencies using the commands below

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

![php 1](<img/56 sudo yum install https ==dl fedoraproject org-pub-epel-epel-release-latest-8 noarch rpm.png>)


```
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
```

![php 2](<img/57 sudo yum install yum-utils http --rpms remirepo net-enterprise-remi-release-8 rpm.png>)


```
sudo yum module list php
```

![php 3](<img/58 sudo yum module list php.png>)


```
sudo yum module reset php
```

![php 4](<img/59 sudo yum module reset php.png>)


```
sudo yum module enable php:remi-7.4
```

![php 5](<img/60 sudo yum module enable php remi-7 4.png>)


```
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
```

![php 6](<img/61 sudo yum install php php-opcache php-gd php-curl php-mysqlnd.png>)


```
sudo systemctl start php-fpm
```

![php 7](<img/62 sudo systemctl start php-fpm.png>)


```
sudo systemctl enable php-fpm
```

![php 8](<img/63 sudo systemctl enable php-fpm.png>)


```
setsebool -P httpd_execmem 1
```

![php 8](<img/64 setsebool -P httpd_execmem 1.png>)


next i restarted apache using the command below

```
sudo systemctl restart httpd
```

![restert httpd](<img/65 sudo systemctl restart httpd.png>)

<br />


Next i downloaded wordpress and copied it to `/var/www/html` following the steps below

```
mkdir wordpress
``` 

![mkdir wordpress](<img/66 mkdir wordpress.png>)


```
cd   wordpress
```

![change directory to wordpress](<img/67 cd wordpress.png>)


```
sudo wget http://wordpress.org/latest.tar.gz
```

![download wordpress](<img/68 sudo wget http wordpress.org latest tar gz.png>)


```
sudo tar xzvf latest.tar.gz
```

![extract downloaded wordpress zip file](<img/69 sudo tar xzvf latest tar gz.png>)


```
sudo rm -rf latest.tar.gz
```

![remove zipped file after unzippng content](<img/70 sudo rm -rf latest tar gz.png>)


```
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
```

![wordpress config](<img/71 sudo cp wordpress-wp-config-sample.php wordpress-wp-config php.png>)


![file created](<img/72 confirm wp-config php file creation.png>)


```
cp -R wordpress/. /var/www/html/
```

![copy](<img/73 sudo cp -R wordpress -var-www-html.png>)

![confirmed copy](<img/74 confirm contents in var-www-html.png>)


### STEP 4 - INSTALL MYSQL ON MY DATABASE EC2 SERVER

I installed mysql server on my database server using the folowing commands

```
sudo yum update
```

```
sudo yum install mysql-server
```

![install mysql server](<img/75 sudo yum install mysql-server.png>)


To `restart`, `enable` and check the `status` of mysql on the database server i used the commands below

```
sudo systemctl restart mysqld
```

```
sudo systemctl enable mysqld
```

```
sudo systemctl status mysqld
```

![restart enable status check](<img/77 restart ensble and check status of mysql.png>)


### STEP 5 - CONFIGURE DATABASE TO WORK WITH WORDPRESS

i created and configured my database using these commands below

```
sudo mysql
```

```
CREATE DATABASE wordpress;
```

```
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
```

```
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
```

```
FLUSH PRIVILEGES;
```

```
SHOW DATABASES;
```

```
exit
```

![database creation](<img/87 server database table creation.png>)



next i set the bind address to do so i open and edit `/etc/my.cnf` file using the command below

```
sudo vi /etc/my.cnf
```
i can set or add any and multiple IP address to this file as bind address


![file for bind address](<img/82 sudo vi -etc-my cnf.png>)


![set bind address](<img/83 sudo vi -etc-my cnf edited.png>)

next i restart mysql using the command below


```
sudo systemctl restart mysqld
```

![restart mysql](<img/84 sudo systemctl restart mysqld.png>)


next i edit the `wp-config.php` file

![edit wp config php file](<img/85 sudo vi wp-config php.png>)


![input relevant infor in the wpconfig file](<img/86 wp config edit.png>)


> i inputed the database user `myuser`, password `mypass` and DB host ip address `172.31.80.160` note the information is not fixed but depends on the information you entered in your database creation and the server ip address


### STEP 6 - CONFIGURE WORDPRESS TO CONNECT TO REMOTE DATABASE

to enable remote connection i added the remote  IP address to the inbound rules on the EC2 console. i opended port 3306 on the server.

![inbound rules](<img/81 server security group.png>)


To remotely connect to my remote EC2 database server i use the command below

```
sudo mysql -h <IP ADDRESS> -u <USERNAME> -p
```

aftertyping the command and pressing enter i typed my password before the database was shown


![remote connection](<img/88 remotely connect to database from webserver.png>)


Next i changed configuration a SELinux Policies 

``` 
sudo chown -R apache:apache /var/www/html/
```

```
sudo chcon -t httpd_sys_rw_content_t /var/www/html/ -R
```

```
sudo setsebool -P httpd_can_network_connect=1
```

```
sudo setsebool -P httpd_can_network_connect_db 1
```


next i visit the webserver page using its public IP address using the command below

```
http://<Public Ip Address>/wp-admin/install.php
```

![browser 1](<img/90 web browser wordpress view.png>)

![browser 2](<img/91 web browser wordpress view.png>)

![browser 3](<img/92 web browser wordpress view.png>)

![browser 4](<img/93 web browser wordpress view.png>)

![browser 5](<img/94 web browser wordpress view.png>)

