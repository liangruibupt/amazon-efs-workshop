# Step 1: Create Amazon EC2 Resources
## Create a security group (efs-walkthrough1-ec2-sg) for your EC2 instance
aws ec2 create-security-group --group-name efs-walkthrough1-ec2-sg \
--description "Amazon EFS walkthrough 1, SG for EC2 instance" \
--vpc-id vpc-0c67c73b8cdb0c429 --region cn-north-1 --profile cn-north-1 
{
    "GroupId": "sg-089de81c7918b07db"
}

aws ec2 authorize-security-group-ingress \
--group-id sg-089de81c7918b07db --protocol tcp --port 22 --cidr 0.0.0.0/0 \
--region cn-north-1 --profile cn-north-1 

aws ec2 create-tags \
--resources  sg-089de81c7918b07db \
--tags Key=Name,Value=efs-walkthrough1-ec2-sg \
--region cn-north-1 --profile cn-north-1 

## Create a security group (efs-walkthrough1-mt-sg) for your Amazon EFS mount target
aws ec2 create-security-group --group-name efs-walkthrough1-mt-sg \
--description "Amazon EFS walkthrough 1, SG for mount target" \
--vpc-id vpc-0c67c73b8cdb0c429 --region cn-north-1 --profile cn-north-1 
{
    "GroupId": "sg-065404e7207bb8560"
}

aws ec2 authorize-security-group-ingress \
--group-id sg-065404e7207bb8560 --protocol tcp --port 2049 \
--source-group sg-089de81c7918b07db \
--region cn-north-1 --profile cn-north-1 

aws ec2 create-tags \
--resources  sg-065404e7207bb8560 \
--tags Key=Name,Value=efs-walkthrough1-mt-sg \
--region cn-north-1 --profile cn-north-1 

aws ec2 describe-security-groups --group-ids sg-089de81c7918b07db sg-065404e7207bb8560 \
--region cn-north-1 --profile cn-north-1 

## Launch an EC2 Instance
EFS-Workshop-Vpc2
aws ec2 run-instances --image-id ami-0242c2cc3f754a0e9 \
--count 1 --instance-type t2.micro \
--associate-public-ip-address --key-name ruiliang-lab-key-pair-cn-north1 \
--security-group-ids sg-089de81c7918b07db --subnet-id subnet-018892dd2635577d7 \
--region cn-north-1 --profile cn-north-1 

"InstanceId": "i-0a53fc6ed6d2ddcf3"

aws ec2 create-tags --resources  i-0a53fc6ed6d2ddcf3 \
--tags Key=Name,Value=efs-walkthrough1-ec2 \
--region cn-north-1 --profile cn-north-1 


# Step 2: Create Amazon EFS
## Create Amazon EFS File System
aws efs create-file-system --creation-token FileSystemForWalkthrough1 \
--tags Key=Name,Value=FileSystemForWalkthrough1 \
--region cn-north-1 --profile cn-north-1 
{
    "CreationToken": "FileSystemForWalkthrough1",
    "FileSystemId": "fs-45f91ed8",
    "Name": "FileSystemForWalkthrough1"
}


## Enable Lifecycle Management
aws efs put-lifecycle-configuration --file-system-id fs-45f91ed8 \
--lifecycle-policies TransitionToIA=AFTER_7_DAYS \
--region cn-north-1 --profile cn-north-1 

## Create a Mount Target
aws efs create-mount-target --file-system-id fs-45f91ed8 \
--subnet-id  subnet-018892dd2635577d7 --security-group sg-065404e7207bb8560 \
--region cn-north-1 --profile cn-north-1 

{
    "OwnerId": "876820548815",
    "MountTargetId": "fsmt-7a39dce7",
    "FileSystemId": "fs-45f91ed8",
    "SubnetId": "subnet-018892dd2635577d7",
    "LifeCycleState": "creating",
    "IpAddress": "172.31.0.145",
    "NetworkInterfaceId": "eni-0f88c09a998b97131"
}

EFS DNS: fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn
EC2 DNS: ec2-52-80-3-21.cn-north-1.compute.amazonaws.com.cn

# Step3: Mount and Test the File system
## Install the NFS Client on Your EC2 Instance
sudo yum -y update  
sudo reboot  
sudo yum -y install nfs-utils

## Mount File System on Your EC2 Instance and Test
mkdir ~/efs-mount-point 
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn:/   ~/efs-mount-point  
cd ~/efs-mount-point  
[ec2-user@ip-172-31-0-87 efs-mount-point]$ ls -la
total 8
drwxr-xr-x 2 root     root     6144 Jan 23 05:03 .
drwx------ 4 ec2-user ec2-user 4096 Jan 23 05:15 ..
sudo chmod go+rw .
touch test-file.txt 
[ec2-user@ip-172-31-0-87 efs-mount-point]$ ls -l
total 4
-rw-rw-r-- 1 ec2-user ec2-user 0 Jan 23 05:16 test-file.txt 



# Step4: CLearn up
aws ec2 terminate-instances --instance-ids i-0a53fc6ed6d2ddcf3 \
--region cn-north-1 --profile cn-north-1 

aws efs delete-mount-target --mount-target-id fsmt-7a39dce7 \
--region cn-north-1 --profile cn-north-1 

aws efs delete-file-system --file-system-id fs-45f91ed8 \
--region cn-north-1 --profile cn-north-1 


# lower than version 1.19 will unable to mount the China region EFS
## EC2 wizard automatically mount user data
```
#cloud-config
repo_update: true
repo_upgrade: all
runcmd:
- yum install -y amazon-efs-utils
- apt-get -y install amazon-efs-utils
- yum install -y nfs-utils
- apt-get -y install nfs-common
- file_system_id_1=fs-dc47a539
- efs_mount_point_1=/mnt/efs/fs1
- mkdir -p "${efs_mount_point_1}"
- chmod go+rw "${efs_mount_point_1}"
- test -f "/sbin/mount.efs" && echo "${file_system_id_1}:/ ${efs_mount_point_1} efs tls,_netdev" >> /etc/fstab || echo "${file_system_id_1}.efs.cn-northwest-1.amazonaws.com.cn:/ ${efs_mount_point_1} nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0" >> /etc/fstab
- test -f "/sbin/mount.efs" && echo -e "\n[client-info]\nsource=liw" >> /etc/amazon/efs/efs-utils.conf
- mount -a -t efs,nfs4 defaults

```
##Install efs-utils EFS on centos
```
$ git clone https://github.com/aws/efs-utils
$ sudo yum -y install rpm-build
$ cd efs-utils && make rpm
$ sudo yum -y install build/amazon-efs-utils*rpm

$ mount.efs --version
# version of mount.efs show as below
/usr/sbin/mount.efs Version: 1.21

$ sudo mkdir -p /mnt/efs
$ sudo mount -t efs fs-<efs_id>:/ /mnt/efs
# add efs path to fstab
$ sudo vim /etc/fstab
fs-<fs_id>:/ /mnt/efs efs defaults,_netdev 0 0
```

##Install efs-utils on ubuntu
```
$ sudo apt-get update
$ sudo apt-get -y install git binutils
$ git clone https://github.com/aws/efs-utils
$ cd efs-utils
$ ./build-deb.sh
$ sudo apt-get -y install ./build/amazon-efs-utils*deb

##Copy original /var/bigbluebutton
$ sudo mkdir /mnt/efs-bbb
$ sudo mount -t efs fs-<efs_id>:/ /mnt/efs-bbb
```