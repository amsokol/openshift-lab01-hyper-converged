# OpenShift Origin v1.5.1 based hyper-converged infrastructure deployment tutorial (deploying containerized Gluster storage with Atomic Host and OpenShift)
Step by step tutorial how to deploy hyper-converged infrustructure by OpenShift Origin v1.5.1 + Gluster for CentOS Atomic Host

## Materials are used to prepare this tutorial:
- [OpenShift Origin Advanced Installation](https://docs.openshift.org/latest/install_config/install/advanced_install.html)
- [Heketi, OpenShift Integration Project Aplo](https://github.com/heketi/heketi/wiki/OpenShift-Integration---Project-Aplo)

## Environment:
| Host                              | OS             | IP              | Cores | RAM     | dev/vda (system) | dev/vdb (docker) | dev/vdc (gluster) |
|-----------------------------------|----------------|-----------------|-------|---------|------------------|------------------|-------------------|
| installer.openshift151.amsokol.me | CentOS Minimal | 192.168.151.10  |   2   | 2048 MB |       64 GB      |         -        |         -         |
| master-01.openshift151.amsokol.me | CentOS Atomic  | 192.168.151.11  |   2   | 4096 MB |       64 GB      |       128 GB     |         -         |
| node-1-01.openshift151.amsokol.me | CentOS Atomic  | 192.168.151.101 |   2   | 4096 MB |       64 GB      |       128 GB     |       256 GB      |
| node-1-02.openshift151.amsokol.me | CentOS Atomic  | 192.168.151.102 |   2   | 4096 MB |       64 GB      |       128 GB     |       256 GB      |
| node-2-01.openshift151.amsokol.me | CentOS Atomic  | 192.168.151.201 |   2   | 4096 MB |       64 GB      |       128 GB     |       256 GB      |
| node-2-02.openshift151.amsokol.me | CentOS Atomic  | 192.168.151.202 |   2   | 4096 MB |       64 GB      |       128 GB     |       256 GB      |

1. CentOS Atomic (tested for `CentOS-Atomic-Host-7.1704-Installer.iso`): [http://cloud.centos.org/centos/7/atomic/images/](http://cloud.centos.org/centos/7/atomic/images/)

2. CentOS Minimal (tested for `CentOS-7-x86_64-Minimal-1704-01.iso`): [https://buildlogs.centos.org/rolling/7/isos/x86_64/](https://buildlogs.centos.org/rolling/7/isos/x86_64/)

## Configure DNS:
1. Set DNS records from table above.

2. Set `*.app.openshift151.amsokol.me` to `192.168.151.101`

3. Set `openshift151.amsokol.me` to `192.168.151.11`

## Users:
You need only root account on `installer` and `master-01`.
All command should be run under `root`!

## Configure `master-01`, `node-1-01`, `node-1-02`, `node-2-01`, `node-2-02` hosts (run for each server):
1. Install OS

2. SSH as root and run:
```
# atomic host upgrade

# reboot
```
3. SSH as root and run:
```
# systemctl stop docker

# atomic storage reset

# atomic storage modify --driver devicemapper --add-device /dev/sdb --vgroup vg-docker

# systemctl start docker
```

4. Run as root:
```
# cat <<EOF >> /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# reboot

# docker info
```

## Configure `installer` host:
1. Install OS

2. SSH as root and run:
```
# yum -y update && yum -y clean all

# reboot
```
3. SSH as root

4. Run (leave all passwords empty):
```
# ssh-keygen
```
5. Run (enter root password for for each server):
```
# for host in master-01.openshift151.amsokol.me \
    node-1-01.openshift151.amsokol.me \
    node-1-02.openshift151.amsokol.me \
    node-2-01.openshift151.amsokol.me \
    node-2-02.openshift151.amsokol.me; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
    done
```
6. Run:
```
# yum -y install centos-release-openshift-origin

# yum -y install git python-cryptography pyOpenSSL httpd-tools ansible

# yum -y clean all

# cd ~

# git clone https://github.com/openshift/openshift-ansible

# git clone https://github.com/amsokol/openshift-lab01-hyper-converged.git

```

## Installation:
1. SSH as root to `installer`

2. Check if all nodes are ready:
```
# cd ~

# ansible -i openshift-lab01-hyper-converged/inventory-lab02.toml nodes -a '/usr/bin/rpm-ostree status'
```

3. Start installation:
```
# ansible-playbook -i openshift-lab01-hyper-converged/inventory-lab02.toml openshift-ansible/playbooks/byo/config.yml
```

## [Optional, just FYI] Redeploy master certificates  (you need to have your own domain instead of amsokol.me):
1. SSH as root to `installer`

2. Uncomment two lines below `"# Redeploy master certificates"` in `inventory-lab02.properties` file:
```
openshift_master_named_certificates=[{"certfile": "/root/openshift.amsokol.me.crt", "keyfile": "/root/openshift.amsokol.me.key", "names":["openshift.amsokol.me"]}]
openshift_master_overwrite_named_certificates=true
```

3. Create `openshift-master.pem` and `openshift-master.pem` on `https://www.startssl.com/`

4. Copy `openshift-master.pem` and `openshift-master.pem` to `installer` /root folder

5. Run installation: 
```
# ansible-playbook -i openshift-lab01-hyper-converged/inventory-lab02.toml openshift-ansible/playbooks/byo/openshift-cluster/redeploy-master-certificates.yml
```

## Add administrator user account:
1. SSH as root to `installer`

2. Add `admin` with password:
```
# ansible -i openshift-lab01-hyper-converged/inventory-lab02.toml masters -a "sed -i '$ a `htpasswd -n admin`' /etc/origin/master/htpasswd"

# ansible -i openshift-lab01-hyper-converged/inventory-lab02.toml masters -a 'oc adm policy add-cluster-role-to-user cluster-admin admin'
```

## [Optional, just FYI] Add user developer account (with name `amsokol` as an example) 
1. SSH as root to `installer`

2. Add `amsokol` with password 
```
# ansible -i openshift-lab01-hyper-converged/inventory-lab02.toml masters -a "sed -i '$ a `htpasswd -n amsokol`' /etc/origin/master/htpasswd"
```
3. [Optional] Give `amsokol` direct access to OpenShift's Docker registry:
```
# ansible -i openshift-lab01-hyper-converged/inventory-lab02.toml masters -a "oc adm policy add-role-to-user system:registry amsokol"

# ansible -i openshift-lab01-hyper-converged/inventory-lab02.toml masters -a "oc adm policy add-role-to-user system:image-builder amsokol"
```

## Install Gluster cluster to OpenShift
1. SSH as root to `installer` and run:
 ```
# yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# yum -y install heketi-templates heketi-client
```

2. Copy all files from `/usr/share/heketi/templates` (on `installer`) to `/root/heketi/templates` (on `master-01` where you need to create `/root/heketi/templates` before)

3. For each `node-1-01`, `node-1-02`, `node-2-01`, `node-2-02` hosts add the following rules to `/etc/sysconfig/iptables` and reboot:
```
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 24007 -j ACCEPT
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 24008 -j ACCEPT
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2222 -j ACCEPT
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m multiport --dports 49152:49251 -j ACCEPT
```

4. [Workaround due to issue [#656](https://github.com/heketi/heketi/issues/656) in Heketi] For each `node-1-01`, `node-1-02`, `node-2-01`, `node-2-02` run the following as root:
```
# systemctl stop rpcbind.socket

# systemctl disable rpcbind.socket
```

5. SSH as root to `master-01` and run:
```
# oc new-project aplo

# oc project aplo

# oc adm policy add-scc-to-user privileged -z default

# oc create -f /root/heketi/templates

# oc process glusterfs -p GLUSTERFS_NODE=node-1-01.openshift151.amsokol.me | oc create -f -

# oc process glusterfs -p GLUSTERFS_NODE=node-1-02.openshift151.amsokol.me | oc create -f -

# oc process glusterfs -p GLUSTERFS_NODE=node-2-01.openshift151.amsokol.me | oc create -f -

# oc process glusterfs -p GLUSTERFS_NODE=node-2-02.openshift151.amsokol.me | oc create -f -
```

6. Wait while all pods are created

7. Run (replace `<admin_password>` by `admin` password you set when created account):
```
# oc process deploy-heketi \
         -p HEKETI_KUBE_NAMESPACE=aplo \
         -p HEKETI_KUBE_APIHOST=https://openshift151.amsokol.me:8443 \
         -p HEKETI_KUBE_INSECURE=y \
         -p HEKETI_KUBE_USER=admin \
         -p HEKETI_KUBE_PASSWORD=<admin_password> | oc create -f -
```

8. Wait while pod is created and test result:
```
# curl http://deploy-heketi-aplo.app.openshift151.amsokol.me/hello
```

9. Run:
```
# oc adm policy add-role-to-user admin system:serviceaccount:aplo:default -n aplo
```

10. SSH as root to `installer` and run:
```
# export HEKETI_CLI_SERVER=http://deploy-heketi-aplo.app.openshift151.amsokol.me:80

# heketi-cli topology load --json=openshift-lab01-hyper-converged/gluster-topology.json

# heketi-cli setup-openshift-heketi-storage
```

11. Copy `heketi-storage.json` from `/root` (on `installer`) to `/root` (on `master-01`)

12. SSH as root to `master-01` and run:
```
# oc create -f heketi-storage.json

# oc delete all,job,template,secret --selector="deploy-heketi"
```
13. Run (replace `<admin_password>` by `admin` password you set when created account):
```
# oc process heketi \
         -p HEKETI_KUBE_NAMESPACE=aplo \
         -p HEKETI_KUBE_APIHOST=https://openshift151.amsokol.me:8443 \
         -p HEKETI_KUBE_INSECURE=y \
         -p HEKETI_KUBE_USER=admin \
         -p HEKETI_KUBE_PASSWORD=<admin_password> | oc create -f -
```

14. Wait while pod is created and test result:
```
# curl http://heketi-aplo.app.openshift151.amsokol.me/hello
```

15. SSH as root to `installer` and run:
```
# export HEKETI_CLI_SERVER=http://heketi-aplo.app.openshift151.amsokol.me:80

# heketi-cli topology info
```

16. Copy `glusterfs-storageclass.yaml` from `/root/openshift-lab01-hyper-converged` (on `installer`) to `/root` (on `master-01`)

17. SSH as root to `master-01` and run:
```
oc create -f glusterfs-storageclass.yaml
```

## Configure Gluster cluster storage for internal Docker registry
1. Login as `admin` (account you created above) to `https://openshift151.amsokol.me:8443`

2. Open `default` project

3. Create storage (`'Storage Classes'`=`'slow'`, `'Name'`=`'docker-registry-claim'`, `'Access Mode'`=`'Shared Access'`, `'Size'`=`50GiB`)

4. SSH as root to `master-01` and run:
```
# oc project default

# oc volume deploymentconfigs/docker-registry --add --name=registry-storage -t pvc --claim-name=docker-registry-claim --overwrite
```
