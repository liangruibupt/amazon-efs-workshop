# Create Writable Per-User Subdirectories and Configure Automatic Remounting on Reboot

Create a "writable" subdirectory under this file system root for each user you create on the EC2 instance, and mount it on the user's home directory. All files and subdirectories the user creates in their home directory are then created on the Amazon EFS file system.
The walkthrough also explains how to configure automatic remounting of subdirectories if the system reboots.

# Create new EC2 user
## Create user mike
sudo useradd -c "Mike Smith" mike
sudo passwd mike
Password

## Create a subdirectory under EFSroot for user mike
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn:/   ~/efs-mount-point
cd ~/efs-mount-point

sudo mkdir ~/efs-mount-point/mike
sudo chown mike:mike ~/efs-mount-point/mike 

# mount the EFSroot/mike subdirectory onto mike's home directory
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn:/mike  /home/mike

# Automatic Remounting on Reboot
Edit /etc/fstab
sudo sh -c 'echo "fs-45f91ed8:/   ~/efs-mount-point efs defaults,_netdev 0 0" >> /etc/fstab'
sudo sh -c 'echo "fs-45f91ed8:/mike  /home/mike efs defaults,_netdev 0 0" >> /etc/fstab'

sudo sh -c 'echo "$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone).fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn:/   ~/efs-mount-point nfs4    defaults" >> /etc/fstab'
sudo sh -c 'echo "$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone).fs-45f91ed8.efs.cn-north-1.amazonaws.com.cn:/mike  /home/mike nfs4    defaults" >> /etc/fstab'

sudo mount -fav

sudo reboot

# ssh back and check the mount