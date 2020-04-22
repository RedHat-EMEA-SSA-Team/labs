# Openshift 4 - Encryption at Rest using KVM vTPM
- Lab overview
    - Nodes:
        - 1x bootstrap
        - 3x master
    - Cluster name: ocp
    - Domain name: ocp4.local
    - Libvirt Network: ocp4
        - cidr: 192.168.150.0/24

## Encrypting disks during installation

During OpenShift Container Platform installation, you can enable disk encryption on all master and worker nodes. This feature:

- Is available for installer provisioned infrastructure and user provisioned infrastructure deployments
- Is supported on Red Hat Enterprise Linux CoreOS (RHCOS) systems only
- Sets up disk encryption during the manifest installation phase so all data written to disk, from first boot forward, is encrypted
- Encrypts data on the root filesystem only (/dev/mapper/coreos-luks-root on /)
- Requires no user intervention for providing passphrases
- Uses AES-256-CBC encryption
- Should be enabled for your cluster to support FIPS.

There are two different supported encryption modes:

- TPM v2: This is the preferred mode. TPM v2 stores passphrases in a secure cryptoprocessor. To implement TPM v2 disk encryption, create an Ignition config file as described below.
- Tang: To use Tang to encrypt your cluster, you need to use a Tang server. Clevis implements decryption on the client side. Tang encryption mode is only supported for bare metal installs.

The follwoing procedures will describe how to test disk encryption for the nodes using KVM vTPM2.

### Enabling Advanced Virtualization Repository 
```
dnf module disable virt -y
```
#### In case of Centos 8
```
cat > /etc/yum.repos.d/CentOS-Virt.repo << EOF
[Advanced_Virt]
name=CentOS-$releasever - Advanced Virt 
baseurl=http://mirror.centos.org/centos/\$releasever/virt/x86_64/advanced-virtualization/
gpgcheck=0
enabled=1
EOF
```
#### In case of RHEL8
```
subscription-manager repos --enable advanced-virt-for-rhel-8-x86_64-rpms
```
### Installing Virtualization Packages
```
yum groupinstall "Virtualization Host" -y
yum install virt-install libguestfs-tools swtpm swtpm-tools @container-tools -y
systemctl enable --now libvirtd
```

### Creating libvirt Network ###
```
cat > ocp4.xml << EOF
<network xmlns:dnsmasq='http://libvirt.org/schemas/network/dnsmasq/1.0'>
  <name>ocp4</name>
  <uuid>c6ffdbe6-23f6-4b6b-b2f6-c2c178bf77d8</uuid>
  <forward mode='route'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:33:2e:f6'/>
  <domain name='ocp.ocp4.local'/>
  <dns>
    <srv service='etcd-server-ssl' protocol='tcp' domain='ocp.ocp4.local' target='etcd-0.ocp.ocp4.local' port='2380' weight='10'/>
    <srv service='etcd-server-ssl' protocol='tcp' domain='ocp.ocp4.local' target='etcd-1.ocp.ocp4.local' port='2380' weight='10'/>
    <srv service='etcd-server-ssl' protocol='tcp' domain='ocp.ocp4.local' target='etcd-2.ocp.ocp4.local' port='2380' weight='10'/>
    <host ip='192.168.150.20'>
      <hostname>bootstrap.ocp.ocp4.local</hostname>
    </host>
    <host ip='192.168.150.30'>
      <hostname>master-0.ocp.ocp4.local</hostname>
      <hostname>etcd-0.ocp.ocp4.local</hostname>
    </host>
    <host ip='192.168.150.31'>
      <hostname>master-1.ocp.ocp4.local</hostname>
      <hostname>etcd-1.ocp.ocp4.local</hostname>
    </host>
    <host ip='192.168.150.32'>
      <hostname>master-2.ocp.ocp4.local</hostname>
      <hostname>etcd-2.ocp.ocp4.local</hostname>
    </host>
    <host ip='192.168.150.1'>
      <hostname>api-int.ocp.ocp4.local</hostname>
      <hostname>api.ocp.ocp4.local</hostname>
    </host>
  </dns>
  <ip address='192.168.150.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.150.128' end='192.168.150.254'/>
      <host mac='54:52:00:01:02:03' name='bootstrap.ocp.ocp4.local' ip='192.168.150.20'/>
      <host mac='54:52:00:01:02:04' name='master-0.ocp.ocp4.local' ip='192.168.150.30'/>
      <host mac='54:52:00:01:02:05' name='master-1.ocp.ocp4.local' ip='192.168.150.31'/>
      <host mac='54:52:00:01:02:06' name='master-2.ocp.ocp4.local' ip='192.168.150.32'/>
    </dhcp>
  </ip>
  <dnsmasq:options>
    <dnsmasq:option value='address=/apps.ocp.ocp4.local/192.168.150.1'/>
  </dnsmasq:options>
</network>
EOF
```

```
virsh net-define --file ocp4.xml
virsh net-autostart ocp4
virsh net-start ocp4
```

## Downloading Openshift Qcow2 Image, Installer and Client

#### In case of 4.3.8
```
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2.gz -O /var/lib/libvirt/images/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2.gz
gunzip /var/lib/libvirt/images/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2.gz
```

```
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.8/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.3.8/openshift-install-linux.tar.gz
mkdir bin
tar xzvf openshift-client-linux.tar.gz -C bin
tar xzvf openshift-install-linux.tar.gz -C bin
```

#### In case of 4.4.0-rc1
```
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/latest/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2.gz -O /var/lib/libvirt/images/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2.gz
gunzip /var/lib/libvirt/images/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2.gz
```

```
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.4.0-rc.1/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.4.0-rc.1/openshift-install-linux.tar.gz
mkdir bin
tar xzvf openshift-client-linux.tar.gz -C bin
tar xzvf openshift-install-linux.tar.gz -C bin
```

### Installing Loadbalancer
```
/usr/bin/podman run -d --name 3nodes --net host \
    -e API="bootstrap=192.168.150.20:6443,master-0=192.168.150.30:6443,master-1=192.168.150.31:6443,master-2=192.168.150.32:6443" \
    -e API_LISTEN="192.168.150.1:6443" \
    -e INGRESS_HTTP="master-0=192.168.150.30:80,master-1=192.168.150.31:80,master-2=192.168.150.32:80" \
    -e INGRESS_HTTP_LISTEN="192.168.150.1:80" \
    -e INGRESS_HTTPS="master-0=192.168.150.30:443,master-1=192.168.150.31:443,master-2=192.168.150.32:443" \
    -e INGRESS_HTTPS_LISTEN="192.168.150.1:443" \
    -e MACHINE_CONFIG_SERVER="bootstrap=192.168.150.20:22623,master-0=192.168.150.30:22623,master-1=192.168.150.31:22623,master-2=192.168.150.32:22623" \
    -e MACHINE_CONFIG_SERVER_LISTEN="192.168.150.1:22623" \
    quay.io/redhat-emea-ssa-team/openshift-4-loadbalancer
```

### Firewall

```
firewall-cmd --add-masquerade --permanent -q
firewall-cmd --add-source=192.168.150.0/24 --permanent -q
firewall-cmd --permanent --add-service=http -q
firewall-cmd --permanent --add-service=dns -q
firewall-cmd --permanent --add-service=https -q
firewall-cmd --permanent --add-port=6443/tcp -q
firewall-cmd --permanent --add-port=22623/tcp -q
firewall-cmd --reload -q
```

# Set up DNS overlay
This step allows installer and users to resolve cluster-internal hostnames from your host.

1. Tell NetworkManager to use dnsmasq:

```
yum install dnsmasq
echo -e "[main]\ndns=dnsmasq" | sudo tee /etc/NetworkManager/conf.d/openshift.conf
```

2. Tell dnsmasq to use your cluster.

```
echo listen-address=127.0.0.1 > /etc/NetworkManager/dnsmasq.d/openshift.conf
echo bind-interfaces >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo server=8.8.8.8 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo address=/ocp.ocp4.local/192.168.150.1 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
```

3. Reload NetworkManager to pick up the dns configuration change: 

```
systemctl reload NetworkManager
```

### Creating install-config.yaml and ignition files

You can download your pull secret from https://cloud.redhat.com/openshift/install/metal/user-provisioned and copy to the file $HOME/redhat-registry-pullsecret.json

#### Generate the ssh key
```
ssh-keygen
```

#### Creating install-config.yaml template file
```
export PULL_SECRET=$(cat $HOME/redhat-registry-pullsecret.json)
export SSH_KEY=$(cat $HOME/.ssh/id_rsa.pub)

cat > install-config.yaml << EOF
apiVersion: v1
baseDomain: ocp4.local
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 3
metadata:
  name: ocp
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: |
  ${PULL_SECRET}
sshKey: |
  ${SSH_KEY}
EOF
```
#### Generate the Kubernetes manifests for the cluster
```
rm -rf /root/ocp4
mkdir /root/ocp4
cp -rf /root/install-config.yaml /root/ocp4
openshift-install create manifests --dir=/root/ocp4
```
#### In the openshift directory, create a master and/or worker file to encrypt disks for those nodes. 

Here are examples of those two files: 

```
cat << EOF > /root/ocp4/manifests/99_openshift-worker-tpmv2-encryption.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: worker-vtpm
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  kernelArguments:
    - rd.neednet
    - ip=dhcp
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;base64,e30K
        filesystem: root
        mode: 420
        path: /etc/clevis.json
EOF

cat << EOF > /root/ocp4/manifests/99_openshift-master-tpmv2-encryption.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: master-vtpm
  labels:
    machineconfiguration.openshift.io/role: master
spec:
  kernelArguments:
    - rd.neednet
    - ip=dhcp
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;base64,e30K
        filesystem: root
        mode: 420
        path: /etc/clevis.json
EOF
```

Continue with the remainder of the OpenShift Container Platform deployment.

```
openshift-install create ignition-configs --dir=/root/ocp4
cat /root/ocp4/bootstrap.ign > /tmp/bootstrap.ign
cat /root/ocp4/master.ign > /tmp/master.ign
```

### Creating Virtual Machines

#### In Case of 4.3.8
```
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/bootstrap.qcow2 60G
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/master-0.qcow2 60G
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/master-1.qcow2 60G
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.3.8-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/master-2.qcow2 60G
```

#### In Case of 4.4.0-rc1
```
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/bootstrap.qcow2 60G
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/master-0.qcow2 60G
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/master-1.qcow2 60G
qemu-img create -f qcow2 -b /var/lib/libvirt/images/rhcos-4.4.0-rc.1-x86_64-qemu.x86_64.qcow2 /var/lib/libvirt/images/master-2.qcow2 60G
```

```
virt-install -q --name="bootstrap" \
    --vcpus=2 --ram=4096 \
    --cpu host \
    --clock hpet_present=yes \
    --noautoconsole \
    --import \
    --memballoon virtio \
    --graphics vnc --console pty,target_type=serial \
    --disk path=/var/lib/libvirt/images/bootstrap.qcow2,format=qcow2,bus=virtio \
    --os-variant rhel8.0 \
    --network network=ocp4,model=virtio \
    --boot menu=on \
    -m 54:52:00:01:02:03 \
    --qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=/tmp/bootstrap.ign"

virt-install -q --name="master-0" \
    --vcpus=4 --ram=16384 \
    --cpu host \
    --clock hpet_present=yes \
    --noautoconsole \
    --import \
    --memballoon virtio \
    --graphics vnc --console pty,target_type=serial \
    --disk path=/var/lib/libvirt/images/master-0.qcow2,format=qcow2,bus=virtio \
    --os-variant rhel8.0 \
    --network network=ocp4,model=virtio \
    --boot menu=on \
    -m 54:52:00:01:02:04 \
    --tpm emulator,model=tpm-tis,version=2.0 \
    --qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=/tmp/master.ign"

virt-install -q --name="master-1" \
    --vcpus=4 --ram=16384 \
    --cpu host \
    --clock hpet_present=yes \
    --noautoconsole \
    --import \
    --memballoon virtio \
    --graphics vnc --console pty,target_type=serial \
    --disk path=/var/lib/libvirt/images/master-1.qcow2,format=qcow2,bus=virtio \
    --os-variant rhel8.0 \
    --network network=ocp4,model=virtio \
    --boot menu=on \
    -m 54:52:00:01:02:05 \
    --tpm emulator,model=tpm-tis,version=2.0 \
    --qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=/tmp/master.ign"

virt-install -q --name="master-2" \
    --vcpus=4 --ram=16384 \
    --cpu host \
    --clock hpet_present=yes \
    --noautoconsole \
    --import \
    --memballoon virtio \
    --graphics vnc --console pty,target_type=serial \
    --disk path=/var/lib/libvirt/images/master-2.qcow2,format=qcow2,bus=virtio \
    --os-variant rhel8.0 \
    --network network=ocp4,model=virtio \
    --boot menu=on \
    -m 54:52:00:01:02:06 \
    --tpm emulator,model=tpm-tis,version=2.0 \
    --qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=/tmp/master.ign"
```

```
openshift-install wait-for --log-level debug bootstrap-complete --dir=ocp4/
```

```
virsh destroy bootstrap
virsh undefine --remove-all-storage bootstrap 
```

```
openshift-install wait-for install-complete --log-level debug --dir=/root/ocp4
```

## Checking the Disks
```
ssh core@<server_IP> lsblk --fs
```

