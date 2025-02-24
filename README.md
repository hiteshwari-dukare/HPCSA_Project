#HPCSA_Project
#Deployment HPC Cluster on Container

Warewulf:

Step 1: Setting up the os:

setenforce 0
sed -i 's/^SELINUX=.*$/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
nmcli con up ens33
nmcli con up ens34
hostnamectl set-hostname master
echo "`ip a | grep ens34 | tail -1 | cut -d' ' -f6|cut -c 1-12` master" >> /etc/hosts
su


Step2: Intalling the warewulf :


# adding the repository
yum install -y https://repo.ctrliq.com/rhel/8/ciq-release.rpm

# installing the warewulf
yum install -y warewulf


Step3: Making changes for warewulf configuration file i.e. warewulf.conf :


vim /etc/warewulf/warewulf.conf

    Line no 2: change ip address
    Line no 4: change network
    Line no 14: change DHCP starting ip address
    Line no 15: change DHCP ending ip address


# configure the warewulf
wwctl configure --all


# check if all the services are running or not
systemctl status dhcpd tftp nfs-server warewulfd


# if services are not running then restart the services
systemctl start dhcpd tftp nfs-server warewulfd


# enable the services
systemctl enable dhcpd tftp nfs-server warewulfd


# check logs if necessary
cat /var/log/warewulfd.log

![1 warewulf_conf](https://github.com/user-attachments/assets/cd36bb1f-337f-4941-b2d1-7210196163bd)

![2 wwctl-configure-all](https://github.com/user-attachments/assets/d9770d6e-2820-4479-b7b8-1937d163fed6)

![3 dhcp-service](https://github.com/user-attachments/assets/d2459cca-5056-4d1d-bd2a-be922660f318)

![4 nfs-server-service](https://github.com/user-attachments/assets/2e4d4107-9abc-491f-84ea-40ba998da194)


Step 3: Booting the node using container :


# import container from docker hub
wwctl container import docker://warewulf/rocky rocky-8

# to check container list
wwctl container list

# to modify the container
wwctl container exec rocky-8 /bin/bash
    dnf install -y passwd
    passwd root
        new password : root
        re enter password : root
    exit
    
# sync host uid & gid with the container
wwctl overlay build
wwctl container syncuser --write rocky-8

Step 4: Adding node :

# to add a node without knowing MAC address
wwctl node add node1 --ipaddr 10.10.10.232 --discoverable ens34

# to add a container to a given node
wwctl node set --container rocky-8 node1

#Note: If this error shows -: hub_port_status failed (err = 110).
#Solution : Disable the USB Device or remove.
![5 solution-for-error](https://github.com/user-attachments/assets/ca75d889-fb56-44f8-af9b-a8ddfc94c9b7)

Installing Slurm :

Step 1: Install munge service on both master and container :


# installing epel-release repository
dnf install epel-release -y

# adding powertools repository
dnf config-manager --set-enabled powertools

# installing munge packages
dnf install munge* -y

# intalling rng-tools used for gathering harware entropy
dnf install rng-tools.x86_64 -y
dnf repolist

# starting the random number generator demon
systemctl start rngd
systemctl enable rngd
systemctl status rngd

Randome number generator
to generate secure cryptographic keys that cannot be easily broken, a source of random numbers is required. 
Generally, the more random the numbers are, the better the chance of obtaining unique keys. 
Entropy for generating random numbers is usually obtained from computing environmental “noise” or using a hardware random number generator.

The rngd daemon, which is a part of the rng-tools package, is capable of using both environmental noise 
and hardware random number generators for extracting entropy.

![6 epel-release](https://github.com/user-attachments/assets/48187281-2686-42da-9fb8-22f90ee02cfc)

![7 yum-repolist](https://github.com/user-attachments/assets/964fe43c-e217-4490-a6db-52c2d7d34d34)

![8 dnf-install-munge](https://github.com/user-attachments/assets/17916c8a-a754-4b58-a3f3-193fe47922ad)

![9 dnf-install-rng-tools](https://github.com/user-attachments/assets/c5a2523b-1268-4712-b332-da1b58386bb2)

![10 rngd-service](https://github.com/user-attachments/assets/f6aa4033-8881-476f-a86b-6372ecd51825)


step 2: Generate munge key :

/usr/sbin/create-munge-key -r

# copy the monge key to container
wwctl container exec --bind [path on master]:[path on container] [container name] /bin/bash

# copy the munge file 
cp /etc/munge/munge.key root/Documents/temp

![11 copy-munge key](https://github.com/user-attachments/assets/edbaff38-0733-43cf-bac4-dba4e7061fa5)

Resolving Error ( glibc )
# install language to resolve this error
dnf install langpacks-en glibc-all-langpacks -y

![12 error-glibc](https://github.com/user-attachments/assets/e89af7da-ec41-44a3-b6b1-9f16d8f3e0f0)

Step 3: Setting up the munge key in container :

# copy the munge key in the default location
cp 
chown munge:munge /etc/munge/munge.key

Step 4: Installing Slurmd

# first download the tar file 
wget https://download.schedmd.com/slurm/slurm-20.11.9.tar.bz2

# download the rpm-build package
dnf install rpm-build make -y
# build the packages from tar file
rpmbuild -ta slurm-20.11.9.tar.bz2
# to install the dependencies
dnf install pam-devel python3 readline-devel perl-ExtUtils-MakeMaker gcc mysql-devel -y
# now try to build the package again
rpmbuild -ta slurm-20.11.9.tar.bz2

![13 slurm-package-get](https://github.com/user-attachments/assets/0ce164c9-3ec0-4771-8396-ca9975b3862c)

![14 slurm-dependencies](https://github.com/user-attachments/assets/6ca82f3b-8b47-42d4-b26e-eeead45757e8)

![15 dependencies-install](https://github.com/user-attachments/assets/b4c97972-d865-47b4-a356-4af7f9c37d91)

Step 5: Create Slurm User for SLURM Service ;
export SLURMUSER=900;
# add group
groupadd -g $SLURMUSER slurm;
# create user
useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm;

cat /etc/passwd | grep slurm;

![16 create-slurm-user](https://github.com/user-attachments/assets/8838a40d-9323-42e2-867f-41ff6cff37b8)

![17 slurm-user-container](https://github.com/user-attachments/assets/317bf071-78ce-400f-b50e-e4dba329df9d)

Step 6: installing packages from rpm builds :

# navigate to the directory
cd /root/rpmbuild/RPMS/x86_64/

# installing local packages
dnf --nogpgcheck localinstall * -y

# to confirm if packages has been installed or not
rpm -qa | grep slurm | wc -l

# copy packages to shared folder with container
cp * /root/test/;

![18 rpm-packages](https://github.com/user-attachments/assets/3015dd07-8a41-4cec-9afd-162733ed1d6c)

![19 confirm-rpm-packages](https://github.com/user-attachments/assets/89828fac-6073-46b9-9f9e-065f756f4460)

![20 installing-slurm-packages-on-container](https://github.com/user-attachments/assets/2180125e-5204-4e06-9583-9501b00ff6ce)

Step 7: creating repositories for slurm on master and node :

# create folder 
mkdir /var/spool/slurm

# change ownership of the slurm service on both client and master
chown slurm:slurm /var/spool/slurm

# change mod
chmod 755 /var/spool/slurm

# create folder 
mkdir /var/log/slurm

# change ownership of the slurm service on both client and master
chown slurm:slurm /var/log/slurm

# create log files
touch /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log

# change ownership
chown slurm:slurm /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log

![21 changing-permissions-slurms](https://github.com/user-attachments/assets/284a5a10-f67a-4536-9f85-46e087a2d1cf)


Step 8: Edit the configuration file :

# copy the example config file 
cp /etc/slurm/slurm.conf.example /etc/slurm/slurm.conf

# edit the config file and add node entry
vi /etc/slurm/slurm.conf

    edit : clusterName=warewulf-cluster
    edit : clusterMachine=master
    edit : slurmUser=slurm
    edit : #COMPUTE NODES
            #NodeName=node[1-2] Procs=1 State=UNKOWN
            NodeName=node1 CPUs=2 Boards=1 SocketsPerBoard=2 CoresPerSocket=1 ThreadsPerCore=1 RealMemory=7928
            ParitionName=standard NODES=ALL Default=YES MaxTime=INFINITE State=UP
            :wq # save and exit

# run on client and get the satus of the node(but node has to be in booted state) and paste this information in the configuration file
slurmd -C

# give ownership to slurm 
chown slurm:slurm /etc/slurm/slurm.conf

# copy this config file to shared folder on the nodes
cp /etc/slurm/slurm.conf /root/Documents/slurm/

# start slurmctld service on master
systemctl start slurmctld;systemctl enable slurmctld;

# start slurmd service on node1
systemctl start slurmd;systemctl enable slurmd;

![slurm_conf](https://github.com/user-attachments/assets/4936d334-d9da-4fda-a309-91dc2c0f423b)


Step 9: To sync the the user with containers we use the command :

# to rebuild the overlays 
wwctl overlay build
wwctl container syncuser --write rocky-8

# to debug error in slurmd 
slurmd -Dvv


Installing Ganglia :

Step 1: Installing the Ganglia on master node

dnf install epel-release -y
dnf install ganglia ganglia-gmetad ganglia-gmond ganglia-web -y

![23](https://github.com/user-attachments/assets/cc5b9296-57eb-4a5a-857c-16ffb074a590)

![24](https://github.com/user-attachments/assets/fab35def-bc9d-42fe-bc6f-6dbfe8773ac6)

Step 2: Edit the conf files on master node :

# on master 
vim /etc/ganglia/gmetad.conf
    line no 44: Change the cluster name


vim /etc/ganglia/gmond.conf

    line no 30: give the cluster name
    line no 31: set the owner name
    line no 50: give the master ip address
    line no 57: comment this line
    line no 59: comment this line

![25](https://github.com/user-attachments/assets/889ecfda-b7fc-41e9-b395-88d1aa306d89)

![26](https://github.com/user-attachments/assets/fb7ceb44-ec07-448e-8ccf-529dba60f68e)

![27](https://github.com/user-attachments/assets/2d1f1a88-0440-4003-b855-2945391e2fba)


Step 3: Start the services on master:

systemctl start gmetad gmond httpd
systemctl enable gmetad gmond httpd
systemctl status gmetad gmond httpd

![28](https://github.com/user-attachments/assets/d3ad643f-16bd-45fb-a416-a6516a8a7313)

![29](https://github.com/user-attachments/assets/c6dd5fa0-a873-4883-ba61-7c315a946955)


Step 4: Installing the Ganglia on Container :

dnf install epel-release -y
dnf install ganglia ganglia-gmond -y

![30](https://github.com/user-attachments/assets/5da26fb1-da79-415c-a791-6881286e1258)

![31](https://github.com/user-attachments/assets/a219d7df-1e3f-462d-85de-a8eeab950622)


Step 5:Edit the gmond.conf file on Container :

vim /etc/ganglia/gmond.conf

    line no 30: give the cluster name
    line no 31: set the owner name
    line no 50: give the master ip address
    line no 57: comment this line
    line no 59: comment this line

![32](https://github.com/user-attachments/assets/a75f2262-a83b-4297-9eaf-6a26abaab3bd)

![33](https://github.com/user-attachments/assets/7b9dfbc4-4183-4fcb-9924-412e9dfb4cca)


Step 6 :Start the services on container:

# service get started after nodes get booted
systemctl start gmond 
systemctl enable gmond 
systemctl status gmond 

Step 7: Rebuild container :

# this command will sync host user to container 
wwctl container syncuser --write rocky-8
# this command will build overlay for given container
wwctl overlay build 
# this command rebuild the container forcefully
wwctl container build -f rocky-8

![34](https://github.com/user-attachments/assets/53db1e12-8327-4ef1-a112-3c90123add38)


Step 8: After successfully booting up the node :

![35](https://github.com/user-attachments/assets/4120d184-4d54-48b0-96b8-e3fb5772e44b)

![36](https://github.com/user-attachments/assets/0d4cd67d-6d28-4fed-aec8-c589bc92cfc2)

