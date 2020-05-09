# Step1 Set up an Apache web server on an EC2 instance.
"MountTargetId": "fsmt-7a39dce7",
"FileSystemId": "fs-45f91ed8",
EFS DNS: fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn
EC2 DNS: ec2-52-80-3-21.cn-north-1.compute.amazonaws.com.cn

## Prepare
sudo umount  ~/efs-mount-point
sudo yum -y install httpd
sudo service httpd start

aws ec2 authorize-security-group-ingress \
--group-id sg-089de81c7918b07db --protocol tcp --port 80 --cidr 0.0.0.0/0 \
--region cn-north-1 --profile cn-north-1

## Mount EFS target
sudo mkdir /var/www/html/efs-mount-point
sudo yum install -y amazon-efs-utils

sudo mount -t efs fs-45f91ed8:/ /var/www/html/efs-mount-point 
[ec2-user@ip-172-31-0-87 ~]$ sudo mount -t efs fs-45f91ed8:/ /var/www/html/efs-mount-point
Failed to resolve "fs-45f91ed8.efs.cn-north-1.amazonaws.com" - check that your file system ID is correct.
Please update your efs-utils version > 1.19 
mount.efs --version
<!-- Fix: you need update the /etc/amazon/efs/efs-utils.conf
[mount]
dns_name_format = {fs_id}.efs.{region}.{dns_name_suffix}
dns_name_suffix = amazonaws.com.cn -->


## prepare HTML
cd /var/www/html/efs-mount-point 
sudo mkdir sampledir  
sudo chown  ec2-user sampledir
sudo chmod -R o+r sampledir
cd sampledir    
echo "<html><h1>Hello from Amazon EFS </h1></html>" > hello.html
public_hostname=`curl http://169.254.169.254/latest/meta-data/public-hostname`
mac=`curl http://169.254.169.254/latest/meta-data/mac`
vpc_id=`curl http://169.254.169.254/latest/meta-data/network/interfaces/macs/${mac}/vpc-id`
echo "<p>public_hostname: ${public_hostname}</p><p>vpc_id: ${vpc_id}</p>" >> hello.html
curl http://ec2-52-80-3-21.cn-north-1.compute.amazonaws.com.cn/efs-mount-point/sampledir/hello.html

# Step2 Set up an Apache web server on multiple EC2 instances by creating an Auto Scaling group
## Create a load balancer
Set the Ping Path value to /efs-mount-point/test.html
ELB DNS: EFS-WebServer-ELB-180589549.cn-north-1.elb.amazonaws.com.cn

## Create an Auto Scaling group with two EC2 instances
#cloud-config
package_upgrade: true
packages:
- nfs-utils
- httpd
runcmd:
- echo "$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone).fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn:/    /var/www/html/efs-mount-point   nfs4    defaults" >> /etc/fstab
- mkdir /var/www/html/efs-mount-point
- mount -a
- touch /var/www/html/efs-mount-point/test.html
- echo "<html><h1>Hello from Amazon EFS Test</h1></html>" > /var/www/html/efs-mount-point/test.html
- service httpd start
- chkconfig httpd on


## On first created EC2 (not by ASG)
cd /var/www/html/efs-mount-point 
sudo mkdir sampledir  
sudo chown  ec2-user sampledir
sudo chmod -R o+r sampledir
cd sampledir
echo "<html><h1>Hello from Amazon EFS Index</h1></html>" > index.html


http://EFS-WebServer-ELB-180589549.cn-north-1.elb.amazonaws.com.cn/efs-mount-point/sampledir/index.html
http://EFS-WebServer-ELB-180589549.cn-north-1.elb.amazonaws.com.cn/efs-mount-point/test.html

## Issue: Only the cn-north-1a can be mount, but failed with cn-north-1b. 
The reason is EFS mount target only create for AZ cn-north-1a
Fix: create the mount target on AZ cn-north1b
aws efs create-mount-target --file-system-id fs-45f91ed8 \
--subnet-id  subnet-05c62631603b6eec6 --security-group sg-065404e7207bb8560 \
--region cn-north-1 --profile cn-north-1 

{
    "OwnerId": "876820548815",
    "MountTargetId": "fsmt-3705e0aa",
    "FileSystemId": "fs-45f91ed8",
    "SubnetId": "subnet-05c62631603b6eec6",
    "LifeCycleState": "creating",
    "IpAddress": "172.31.1.64",
    "NetworkInterfaceId": "eni-094d14247712c9612"
}

On EC2 instance cn-north-1b, run sudo mount -a again. Then you can found the shared file system mounted