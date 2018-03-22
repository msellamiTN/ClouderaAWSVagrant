# -*- mode: ruby -*-
# vi: set ft=ruby :
 
$master_script = <<SCRIPT
#!/bin/bash
cat > /etc/hosts <<EOF
127.0.0.1       localhost
 
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
 
10.211.55.100   vm-cluster-node1.vmcluster  vm-cluster-node1  
10.211.55.102   ad.vmcluster  ad  
EOF
 
# common part for cloudera
yum -y install wget
wget http://archive-primary.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo -O /etc/yum.repos.d/cloudera-manager.repo
#sed -i 's#archive.cloudera.com#10.0.2.2#' /etc/yum.repos.d/cloudera-manager.repo
sed -i 's#cm/5#cm/5.4#' /etc/yum.repos.d/cloudera-manager.repo
echo 'vm.swappiness = 1' >> /etc/sysctl.conf
echo 1 > /proc/sys/vm/swappiness
 
# master part for cloudera
yum -y install oracle-j2sdk1.7 cloudera-manager-server cloudera-manager-server-db-2 cloudera-manager-agent openldap-clients krb5-server krb5-workstation
service cloudera-scm-server-db start
service cloudera-scm-server start
service cloudera-scm-agent start
sudo ln /dev/random /dev/randombkp
sudo ln -sf /dev/urandom /dev/random
 
# worker part for cloudera
yum -y install oracle-j2sdk1.7 cloudera-manager-agent krb5-workstation
sed -i 's/server_host=.*/server_host=10.0.55.2/' /etc/cloudera-scm-agent/config.ini
service cloudera-scm-agent start 
 
# security part
yum install -y krb5-server krb5-libs krb5-auth-dialog krb5-workstation authconfig
 
cat > /etc/krb5.conf <<EOF
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log
 
[libdefaults]
 default_realm = VMCLUSTER
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 
[realms]
 VMCLUSTER = {
  kdc = vm-cluster-node1
  admin_server = vm-cluster-node1
 }
 
[domain_realm]
 vmcluster = VMCLUSTER
 .vmcluster = VMCLUSTER
 
EOF
 
# now create the kerberos database
kdb5_util create -s -P vagrant
 
# update the kdc.conf file
sed -i 's/EXAMPLE.COM/VMCLUSTER/g' /var/kerberos/krb5kdc/kdc.conf
# this will add a line to the file with ticket life
sed -i '/dict_file/a max_life = 1d' /var/kerberos/krb5kdc/kdc.conf
# add a max renewable life
sed -i '/dict_file/a max_renewable_life = 7d' /var/kerberos/krb5kdc/kdc.conf
# indent the two new lines in the file
sed -i 's/^max_/  max_/' /var/kerberos/krb5kdc/kdc.conf
 
# the acl file needs to be updated so the */admin
# is enabled with admin privileges 
sed -i 's/EXAMPLE.COM/VMCLUSTER/' /var/kerberos/krb5kdc/kadm5.acl
 
 
# The kerberos authorization tickets need to be renewable
# if not the Hue service will show bad (red) status
# and the Hue “Kerberos Ticket Renewer” will not start
# the error message in the log will look like this:
#  kt_renewer   ERROR    Couldn't renew # kerberos ticket in 
#  order to work around Kerberos 1.8.1 issue.
 
# update the kdc.conf file to allow renewable
sed -i '/supported_enctypes/a default_principal_flags = +renewable, +forwardable' /var/kerberos/krb5kdc/kdc.conf
# fix the indenting
sed -i 's/^default_principal_flags/  default_principal_flags/' /var/kerberos/krb5kdc/kdc.conf
 
# start up the kdc server and the admin server
service krb5kdc start
service kadmin start
 
chkconfig krb5kdc on
chkconfig kadmin on
 
kadmin.local <<eoj
addprinc -pw cloudera cloudera-scm/admin@VMCLUSTER
modprinc -maxrenewlife 1week cloudera-scm/admin@VMCLUSTER
eoj
 
kadmin.local <<eoj
addprinc -pw hdfs hdfs@VMCLUSTER
modprinc -maxrenewlife 1week hdfs@VMCLUSTER
eoj
 
SCRIPT
 
Vagrant.configure("2") do |config|
 
  config.vm.define :master do |master|
    master.vm.box = "chef/centos-6.6"
    master.vm.provider :vmware_fusion do |v|
      v.vmx["memsize"]  = "8192"
    end
    master.vm.provider :virtualbox do |v|
      v.name = "vm-cluster-node1.vmcluster"
      v.customize ["modifyvm", :id, "--memory", "8192"]
    end
    master.vm.network :private_network, ip: "10.211.55.100"
    master.vm.hostname = "vm-cluster-node1.vmcluster"
    master.vm.provision :shell, :inline => $master_script
  end
end