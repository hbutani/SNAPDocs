<!-- --- title: EC2 SNAP -->

#### Ensure Instance Disks are mounted

If the ec2 instance disk is not formatted and mounted then you have do this manually

```
# Use df -h, to check mount points and space
# there should be at least 1 mount point with over 50GB of free space

# to format and mount a disk
# follow instructions from here: https://blogs.oracle.com/AlejandroVargas/entry/how_to_create_configure_and_mo

# for e.g.
sudo fdisk
# then create a new partition on /dev/xvdca1
sudo mkfs -t ext4 /dev/xvdca1
sudo mount /dev/xvdca1 /mnt

sudo chmod 777 /mnt
```

#### Install jdk 8

```
cd /opt/
sudo wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u121-b13/e9e7ea248e2c4826b92b3f075a80e441/jdk-8u121-linux-x64.tar.gz"
sudo tar xzf jdk-8u121-linux-x64.tar.gz
sudo rm jdk-8u121-linux-x64.tar.gz
cd jdk1.8.0_121/
sudo alternatives --install /usr/bin/java java /opt/jdk1.8.0_121/bin/java 2
sudo alternatives --config java
```

#### S3 Setup

```
aws configure
aws s3 cp s3://splsnap/snap-assembly-1.0.0-SNAPSHOT.jar .
```

#### Install Spark

```
wget  http://d3kbcqa49mib13.cloudfront.net/spark-2.0.2-bin-hadoop2.7.tgz
```

#### Copy conf, sh files

```
cd spark-2.0.2-bin-hadoop2.7
cd sbin
aws s3 cp s3://splsnap/start-sparklinedatathriftserver.sh .
aws s3 cp s3://splsnap/stop-sparklinedatathriftserver.sh .
chmod +x *.sh
cd ../conf/
aws s3 cp s3://splsnap/fairscheduler.xml .
```

#### YourKit Setup

```
wget https://www.yourkit.com/download/yjp-2017.02-b57.zip

unzip yjp-2017.02-b57.zip
cd yjp-2017.02
bin/yjp.sh -integrate
```

#### Start Stop commands

```
export JAVA_HOME=/opt/jdk1.8.0_121/

sbin/start-sparklinedatathriftserver.sh ~/snap-assembly-1.0.0-SNAPSHOT.jar --properties-file ~/sparkline.properties --master local-cluster[1,7,14336]
# or local-cluster[1,14,14336]

sbin/stop-sparklinedatathriftserver.sh
```

#### YourKit Remote Attach

> bin/yjp.sh -attach



