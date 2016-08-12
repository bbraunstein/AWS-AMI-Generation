# AWS-AMI-Generation
## Deploy EC2 AMI & API Tools ##
<pre><code>
mkdir /tmp/packages
cd /tmp/packages/
wget http://s3.amazonaws.com/ec2-downloads/:
sudo mkdir /usr/local/ec2
sudo unzip ec2-api-tools.zip -d /usr/local/ec2
yum install -y ruby
wget http://s3.amazonaws.com/ec2-downloads/ec2-ami-tools.noarch.rpm
yum install -y ec2-ami-tools.noarch.rpm
yum install -y java
export JAVA_HOME=$(file $(file $(which java) | awk '{print $5}' | tr -d \`\') | awk '{print $5}' | tr -d \`\'\ | sed -e 's#/bin/java##')
export EC2_HOME=/usr/local/ec2/$(ls /usr/local/ec2/ | grep ec2-api-tools-*)
export PATH=$PATH:$EC2_HOME/bin
export AWS_ACCESS_KEY=
export AWS_SECRET_KEY=
export AWS_ACCOUNT_NUMBER=
cd
rm -rf /tmp/packages/
</pre></code>

---
## Creating the Image ##
<pre><code>
mkdir /opt/ec2/images/
dd if=/dev/zero of=/opt/ec2/images/CentOS-6.7-x86_64-LeadRouter-PV-IS.img bs=1M count=8192
mkfs.ext4 -F -j /opt/ec2/images/CentOS-6.7-x86_64-LeadRouter-PV-IS.img
mkdir /mnt/ec2-image
mount -o loop /opt/ec2/images/CentOS-6.7-x86_64-LeadRouter-PV-IS.img /mnt/ec2-image/
mkdir -p /mnt/ec2-image/{dev,etc,proc,sys}
mkdir -p /mnt/ec2-image/var/{cache,log,lock,lib/rpm}

/sbin/MAKEDEV -d /mnt/ec2-image/dev -x console
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x null
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x zero
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x urandom

mount -o bind /dev /mnt/ec2-image/dev
mount -o bind /dev/shm /mnt/ec2-image/dev/shm
mount -o bind /proc /mnt/ec2-image/proc
mount -o bind /sys /mnt/ec2-image/sys
mount -o bind /dev/pts /mnt/ec2-image/dev/pts

yum -c /opt/ec2/yum/CentOS-Base.repo --installroot=/mnt/ec2-image -y groupinstall Base
yum -c /opt/ec2/yum/CentOS-Base.repo --installroot=/mnt/ec2-image -y install *openssh* dhclient gdisk
yum -c /opt/ec2/yum/CentOS-Base.repo --installroot=/mnt/ec2-image -y install grub e2fsprogs yum-plugin-fastestmirror.noarch vim sudo cloud-init.x86_64

cat <<EOF >/mnt/ec2-image/etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
NM_CONTROLLED="yes"
ONBOOT=yes
TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
EOF

/usr/sbin/chroot /mnt/ec2-image /sbin/chkconfig --level 2345 network on

cat <<EOF > /mnt/ec2-image/etc/sysconfig/network
NETWORKING=yes
HOSTNAME=localhost.localdomain
EOF

cp /etc/skel/.bash* /mnt/ec2-image/root/
mkdir -p /mnt/ec2-image/mnt/vol1

echo -e "/dev/xvde1\t/\text4\tdefaults\t1\t1" > /mnt/ec2-image/etc/fstab
echo -e "none\t/dev/pts\tdevpts\tgid=5,mode=620\t0\t0" >> /mnt/ec2-image/etc/fstab
echo -e "none\t/dev/shm\ttmpfs\tdefaults\t0\t0" >> /mnt/ec2-image/etc/fstab
echo -e "none\t/proc\tproc\tdefaults\t0\t0" >> /mnt/ec2-image/etc/fstab
echo -e "none\t/sys\tsysfs\tdefaults\t0\t0" >> /mnt/ec2-image/etc/fstab
echo -e "/dev/xvde2\t/mnt/vol1\text4\tdefaults\t0\t0" >> /mnt/ec2-image/etc/fstab
echo -e "/dev/xvde3\tswap\tswap\tdefaults\t0\t0" >> /mnt/ec2-image/etc/fstab

vim /mnt/ec2-image/etc/fstab    # (check formatting is correct, i.e. no red blocks)
cat <<EOF >/mnt/ec2-image/boot/grub/grub.conf
default=0
timeout=0
title CentOS 6.6 (Modus AMI)
root (hd0)
kernel /boot/vmlinuz ro root=/dev/xvde1 rd_NO_PLYMOUTH
initrd /boot/initramfs
EOF

ln -s /boot/grub/grub.conf /mnt/ec2-image/boot/grub/menu.lst
kern=$(ls /mnt/ec2-image/boot/vmlin*|awk -F/ '{print $NF}')
ird=$(ls /mnt/ec2-image/boot/initramfs*.img|awk -F/ '{print $NF}')
sed -ie "s/vmlinuz/$kern/" /mnt/ec2-image/boot/grub/grub.conf
sed -ie "s/initramfs/$ird/" /mnt/ec2-image/boot/grub/grub.conf


cat /mnt/ec2-image/boot/grub/grub.conf
default=0
timeout=0
title CentOS 6.2 (Custom AMI)
root (hd0)
kernel /boot/vmlinuz-2.6.32-504.16.2.el6.x86_64 ro root=/dev/xvde1 rd_NO_PLYMOUTH
initrd /boot/initramfs-2.6.32-504.16.2.el6.x86_64.img
</pre></code>

### AMI Modifications ###
Update default user in cloud.cfg
vim /mnt/ec2-image/etc/cloud/cloud.cfg
Limit init processes. An ideal example:
<pre><code>
acpid          	0:off	1:off	2:on	3:on	4:on	5:on	6:off
cloud-config   	0:off	1:off	2:on	3:on	4:on	5:on	6:off
cloud-final    	0:off	1:off	2:on	3:on	4:on	5:on	6:off
cloud-init     	0:off	1:off	2:on	3:on	4:on	5:on	6:off
cloud-init-local	0:off	1:off	2:on	3:on	4:on	5:on	6:off
crond          	0:off	1:off	2:on	3:on	4:on	5:on	6:off
netfs          	0:off	1:off	2:off	3:on	4:on	5:on	6:off
network        	0:off	1:off	2:on	3:on	4:on	5:on	6:off
rsyslog        	0:off	1:off	2:on	3:on	4:on	5:on	6:off
sshd           	0:off	1:off	2:on	3:on	4:on	5:on	6:off
sysstat        	0:off	1:on	2:on	3:on	4:on	5:on	6:off
udev-post      	0:off	1:on	2:on	3:on	4:on	5:on	6:off
Plus any applicable specifications related to the project
</pre></code>

Clean up
<pre><code>
yum -c /opt/ec2/yum/CentOS-Base.repo --installroot=/mnt/ec2-image -y clean packages
rm -rf /mnt/ec2-image/root/.bash_history
rm -rf /mnt/ec2-image/var/cache/yum
rm -rf /mnt/ec2-image/var/lib/yum
sync; sync; sync; sync
umount /mnt/ec2-image/dev/shm
umount /mnt/ec2-image/dev/pts
umount /mnt/ec2-image/dev
umount /mnt/ec2-image/sys
umount /mnt/ec2-image/proc
umount /mnt/ec2-image
</pre></code>

---
# Generating the AMI #
Kernel aki-88aa75e1 is the correct ID for CentOS 6.5 --> Centos 6.8
<pre><code>
ec2-bundle-image --cert /tmp/cert/Leadrouter-crt.pem --privatekey /tmp/cert/LeadRouter-key.pem \
--image /opt/ec2/images/CentOS-6.7-x86_64-LeadRouter-PV-IS.img
--prefix CentOS-6.7-x86_64-LeadRouter-PV-IS
--user $AWS_ACCOUNT_NUMBER --destination /opt/ec2/images --arch x86_64 --kernel aki-88aa75e1

ec2-upload-bundle --manifest /opt/ec2/images/CentOS-6.7-x86_64-LeadRouter-PV-IS.manifest.xml \
--bucket leadrouter/amis/CentOS-6.7-x86_64-LeadRouter-PV-IS --access-key $AWS_ACCESS_KEY \
--secret-key $AWS_SECRET_KEY

ec2-register leadrouter/amis/CentOS-6.7-x86_64-LeadRouter-PV-IS/CentOS-6.7-x86_64-LeadRouter-PV-IS.manifest.xml \
--name "CentOS-6.7-x86_64-LeadRouter-PV-IS” --description "CentOS 6.7 (x86_64) PV-IS Base AMI" \
--architecture x86_64 --kernel aki-88aa75e1
</pre></code>


---
# Converting IS to EBS-backed AMI (PVM & HVM) #
<b>Prerequisites</b>
* Fully functioning IS image
* 8GB target volume
* 10+ GB for workspace
* AWS privatekey
* EC2 API & AMI tools (see first section)

<ol>
<li>Spin up an IS image</li>
<li>Create two EBS volumes in the same availability zone as the running instance</li>
<li>8 GB, target volume for EBS conversion</li>
<li>10+ GB, workspace for downloading and bundling the manifest</li>
<li>Attach the two volumes to your running IS image</li>
</li>Copy over the privatekey to the running IS image</li>
</ol>

---
## Starting the Conversion Process 
Create a folder for your bundle and download the bundle for your IS AMI from S3::
<pre><code>
mkdir /tmp/bundle
ec2-download-bundle -b leadrouter/amis/CentOS-6.7-x86_64-LeadRouter-HVM-IS \
-m CentOS-6.7-x86_64-LeadRouter-HVM-IS.manifest.xml -a $AWS_ACCESS_KEY -s $AWS_SECRET_KEY \
--privatekey /tmp/bundle/LeadRouter-key.pem -d /tmp/bundle/
</pre></code>

Change dir to the bundle folder and reconstitute the image:
<pre><code>cd /tmp/bundle
ec2-unbundle -m CentOS-6.7-x86_64-LeadRouter-HVM-IS.manifest.xml --privatekey /tmp/bundle/LeadRouter-key.pem
</pre></code>

Copy the files from the unbundled image to the new Amazon EBS volume
<pre><code>dd if=CentOS-6.7-x86_64-LeadRouter-HVM-IS of=/dev/xvdf bs=1M</pre></code>

Probe the volume for any new partitions that were unbundled:
<pre><code>
[root@ip-10-200-99-160 ebs]# partprobe /dev/xvdf 
[root@ip-10-200-99-160 ebs]# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0  7.8G  0 disk 
└─xvda1 202:1    0  7.8G  0 part /
xvdb    202:16   0    4G  0 disk /mnt
xvdf    202:80   0    8G  0 disk 
└─xvdf1 202:81   0  7.8G  0 part 
xvdg    202:96   0   20G  0 disk /mnt/ebs
</pre></code>


(Optional) If you need to make local changes to the AMI:
<pre><code>
mkdir /mnt/ebs
mount /dev/xvdf1 /mnt/ebs

[ ... Update configs or chroot, as needed ... ]

umount /mnt/ebs
</pre></code>

---
## Finalizing the EBS Conversion Process
* Go to the AWS Console, right click the 8GB volume and select ‘Create Snapshot’
* After the Snapshot has completed, right click the snapshot and select ‘Create Image’
* For Virtualization Type -
  * Paravirtual = Paravirtual
  * HVM = Hardware-assisted virtualization
* For Kernel ID -
  * Paravirtual = match AKI that was specified with IS creation
  * HVM = N/A (Use default)

## Tagging & Review
Once the AMIs are all created, be sure to verify they are all in functioning order. 
Afterwards, go through the process of accurately tagging the necessary fields for Volumes, Snapshots, and AMIs.
