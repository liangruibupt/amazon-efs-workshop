# Step 1: Create Your Amazon Elastic File System Resources
Add a rule to the mount target's security group to allow inbound traffic to the Network File System (NFS) port (2049). 

# Step 2: Install the NFS Client
```
Red Hat Linux
$ sudo yum -y install nfs-utils

Ubuntu, install NFS with the following command
$ sudo apt-get -y install nfs-common
```

Launch a instance in ZHY region as simulation (EFS on BJS region)
```
sudo apt-get -y install nfs-common
```

# Step 3: Mount the Amazon EFS File System in other VPC
Mount your file system using one of the mount target IP addresses that your EC2 instance can access over the VPC peering connection. If your EC2 instance and your EFS file system are in the same AWS Region, use the mount target IP address that is in your Availability Zone.

```
ubuntu@ip-10-0-0-215:~$ mkdir ~/efs

Enable cross region VPC peering with vpc-0c67c73b8cdb0c429 EFS-Workshop-Vpc2-1JKSGQZJX7OR0 and vpc-00d4874adbfc118c0 client VPC in Ningxia
1. Enable cross region VPC peering 
2. Configure the routing table of EFS VPC and Client EC2 VPC to enable peering
3. Configure EFS SG to allow NFS 2049 port from client EC2 access

ubuntu@ip-10-0-0-215:~$ sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 172.31.0.145:/   ~/efs

ubuntu@ip-10-0-0-215:~$ cd efs
ubuntu@ip-10-0-0-215:~/efs$ ls
mike  sampledir  test-file.txt  test.html

ubuntu@ip-10-0-0-215:~/efs$ sudo mkdir getting-started
ubuntu@ip-10-0-0-215:~/efs$ sudo chown ubuntu getting-started
ubuntu@ip-10-0-0-215:~/efs$ cd getting-started
ubuntu@ip-10-0-0-215:~/efs/getting-started$ touch zhy-test-file.txt
ubuntu@ip-10-0-0-215:~/efs/getting-started$ echo "mount from zhy testing" > zhy-test-file.txt

In Beijing EC2 which mount same EFS target
[ec2-user@ip-172-31-0-87 ~]$ cd efs-mount-point/
[ec2-user@ip-172-31-0-87 efs-mount-point]$ ls
getting-started  mike  sampledir  test-file.txt  test.html
[ec2-user@ip-172-31-0-87 efs-mount-point]$ cd getting-started/
[ec2-user@ip-172-31-0-87 getting-started]$ ls
zhy-test-file.txt
[ec2-user@ip-172-31-0-87 getting-started]$ cat zhy-test-file.txt
mount from zhy testing
```

DNS name: fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn

Mount targets
VPC |  AZ  | Subnet | IP address | mount target ID | ENI | SG 
vpc-0c67c73b8cdb0c429 - VPC / EFS-Workshop-Vpc2-1JKSGQZJX7OR0 | cn-north-1a | subnet-018892dd2635577d7 - Public Subnet 0 | 172.31.0.145 | fsmt-7a39dce7 | eni-0f88c09a998b97131 | sg-065404e7207bb8560 - efs-walkthrough1-mt-sg
vpc-0c67c73b8cdb0c429 - VPC / EFS-Workshop-Vpc2-1JKSGQZJX7OR0 | cn-north-1b | subnet-05c62631603b6eec6 - Public Subnet 1 | 172.31.1.64 | fsmt-3705e0aa | eni-094d14247712c9612 | sg-065404e7207bb8560 - efs-walkthrough1-mt-sg

# Step 4: Encrypting Data in Transit
## install amazon-efs-utils
To encrypt data in transit, use the Amazon EFS mount helper, amazon-efs-utils, instead of the NFS client.
https://docs.aws.amazon.com/efs/latest/ug/using-amazon-efs-utils.html
```
ubuntu@ip-10-0-0-215:~$ git clone https://github.com/aws/efs-utils
ubuntu@ip-10-0-0-215:~$ cd efs-utils/
ubuntu@ip-10-0-0-215:~/efs-utils$ sudo apt-get -y install binutils
ubuntu@ip-10-0-0-215:~/efs-utils$ ./build-deb.sh
ubuntu@ip-10-0-0-215:~/efs-utils$ sudo apt-get -y install ./build/amazon-efs-utils*deb

The following packages have unmet dependencies:
 amazon-efs-utils : Depends: stunnel4 (>= 4.56) but it is not installable

sudo sh -c 'apt-get update && apt-get install stunnel4'
```

## To configure amazon-efs-utils for use in your AWS Region
```
Update the /etc/amazon/efs/efs-utils.conf
[mount]
dns_name_format = {fs_id}.efs.{region}.{dns_name_suffix}
dns_name_suffix = amazonaws.com.cn

To update /etc/hosts
mount-target-IP-Address file-system-ID.efs.region.amazonaws.com
172.31.0.145 fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn

Mount by TLS
ubuntu@ip-10-0-0-215:~/efs-utils$ sudo umount ~/efs
ubuntu@ip-10-0-0-215:~/efs-utils$ sudo mount -t efs -o tls fs-45f91ed8 ~/efs
ubuntu@ip-10-0-0-215:~/efs-utils$ cd ~/efs
ubuntu@ip-10-0-0-215:~/efs$ ls
getting-started  mike  sampledir  test-file.txt  test.html

```

# Step 5: Clean Up Resources and Protect Your AWS Account
sudo umount ~/efs
remove EFS SG entry which allowing NFS 2049 port from client EC2 access
rollback the routing table of EFS VPC and Client EC2 VPC to disable peering
delete cross region VPC peering







