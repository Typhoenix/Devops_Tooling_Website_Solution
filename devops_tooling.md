# DevOps Tooling Website Solution

In this follow-up tutorial,  we'd introducing a single DevOps Tooling Solution that will consist of the following:

- Jenkins - free and open source automation server used to build CI/CD pipelines.
- Kubernetes - an open-source container-orchestration system for automating computer application deployment, scaling, and management.
- Jfrog Artifactory - Universal Repository Manager supporting all major packaging formats, build tools and CI servers. Artifactory.
- Rancher - an open source software platform that enables organizations to run and manage Docker and Kubernetes in production.
- Grafana - a multi-platform open source analytics and interactive visualization web application.
- Prometheus - An open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
- Kibana - Kibana is a free and open user interface that lets you visualize your Elasticsearch data and navigate the Elastic Stack.
  
In this project you will implement a solution that consists of following components:

* Infrastructure: AWS

* Webserver Linux: Red Hat Enterprise Linux 8

* Database Server: Ubuntu 20.04 + MySQL

* Storage Server: Red Hat Enterprise Linux 8 + NFS Server

* Programming Language: PHP

![](assets/1.png)

## Step 1 - Prepare the NFS Server
1. Spin up a new EC2 instance with RHEL Linux 8 Operating System.
2. Configure LVM on the Server (Follow the following steps to carry out the LVM config):

- Create volumes in the same availability zone as your instance and attach them.
Like so:

![](assets/3.png)

![](assets/2.png)
   
To verify, run:

`$ sudo lsblk`

![](assets/4.png)

- Use gdisk utility to create a single partition on each of the 3 disks
  
  `sudo gdisk /dev/xvdh`
```
Output:
GPT fdisk (gdisk) version 1.0.7

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries in memory.

Command (? for help): n
Partition number (1-128, default 1):
First sector (34-20971486, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-20971486, default = 20971486) or {+-}size{KMGTP}:
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/xvdh: 20971520 sectors, 10.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 7E5A9021-C3AF-4CB9-B5FD-A608BC24D915
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 20971486
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20971486   10.0 GiB    8300  Linux filesystem

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/xvdh.
The operation has completed successfully.
```
- verify by running: `lsblk`

![](assets/5.png)

3. Install lvm2 package using 
   `sudo yum install lvm2`
   
Run `sudo lvmdiskscan` command to check for available partitions

4.Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
```
sudo pvcreate /dev/xvdf1 
sudo pvcreate /dev/xvdg1 
sudo pvcreate /dev/xvdh1
```
```
Output:

Physical volume "/dev/xvdf1" successfully created.
  Creating devices file /etc/lvm/devices/system.devices
  Physical volume "/dev/xvdg1" successfully created.
  Physical volume "/dev/xvdh1" successfully created.
  ```
- Verify that your Physical volume has been created successfully by running : `sudo pvs`
```
Output:

   PV         VG Fmt  Attr PSize   PFree
  /dev/xvdf1    lvm2 ---  <10.00g <10.00g
  /dev/xvdg1    lvm2 ---  <10.00g <10.00g
  /dev/xvdh1    lvm2 ---  <10.00g <10.00g
  ```
4. Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg:

`sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

Verify that your VG has been created successfully by running: `sudo vgs`

```
Output:

VG         #PV #LV #SN Attr   VSize   VFree
  webdata-vg   3   0   0 wz--n- <29.99g <29.99g
```
5. Use lvcreate utility to create 3 logical volumes:lv-opt lv-apps, and lv-logs
   
`sudo lvcreate -n lv-opt -L 10G webdata-vg`

`sudo lvcreate -n lv-apps -L 10G webdata-vg`

`sudo lvcreate -n lv-logs -L 8G webdata-vg`

```
Output:
Logical volume "lv-opt" created.
Logical volume "lv-apps" created.
Logical volume "lv-logs" created.
```
run `sudo lvs`

```
Output:
  LV      VG         Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv-apps webdata-vg -wi-a----- 10.00g
  lv-logs webdata-vg -wi-a-----  9.00g
  lv-opt  webdata-vg -wi-a----- 10.00g
```
- Format the disks as xfs
  
```
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```
6. Create mount points on /mnt directory for the logical volumes as follow:

- Mount lv-apps on /mnt/apps – To be used by webservers
- Mount lv-logs on /mnt/logs – To be used by webserver logs
- Mount lv-opt on /mnt/opt – To be used by Jenkins server 
```
sudo mkdir /mnt/apps

sudo mkdir /mnt/logs

sudo mkdir /mnt/opt

sudo mount /dev/webdata-vg/lv-apps /mnt/apps

sudo mount /dev/webdata-vg/lv-logs /mnt/logs

sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```
>Once mount is completed run 
`sudo blkid` to get the UUID, edit the fstab file accordingly

`sudo vi /etc/fstab`

Verify the mount points

`sudo mount -a `
`sudo systemctl daemon-reload`

7. Install NFS server, configure it to start on reboot and make sure it is u and running

```
sudo yum -y update

sudo yum install nfs-utils -y

sudo systemctl start nfs-server.service

sudo systemctl enable nfs-server.service

sudo systemctl status nfs-server.service
```
