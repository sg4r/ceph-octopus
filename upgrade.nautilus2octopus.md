# TP de nautilus à octopus
l'exercice est de migrer un cluster ceph de la version nautilus et déployé avec ceph-deploy vers ceph octopus
```
$ cp Vagrantfile.ct7 Vagrantfile
$ vagrant up
$ vagrant status
Current machine states:

cephclt                   running (libvirt)
cn1                       running (libvirt)
cn2                       running (libvirt)
cn3                       running (libvirt)
cn4                       running (libvirt)
```
le TP va utiliser des vms en CentOS7 qu'il faudra mettre a jours.
## connexion à l'environement
```
$ vagrant ssh cn1
```
# Installation ceph nautilus sous CentOS7
création d'un cluster ceph minimal pour tester les étapes de mises a jours vers ceph octopus
```
sudo  vi /etc/yum.repos.d/ceph.repo
[Ceph]
name=Ceph packages for $basearch
baseurl=http://download.ceph.com/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-nautilus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://download.ceph.com/rpm-nautilus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1


sudo yum install ceph-deploy
sudo yum -y install python2-pip

mkdir ceph
cd ceph

for i in cn1 cn2 cn3 cephclt; do ssh $i sudo systemctl disable firewalld; ssh $i sudo systemctl stop firewalld; done

ceph-deploy install --release nautilus cn1 cn2 cn3
ceph-deploy new --public-network 192.168.0.0/24 cn1 cn2 cn3
ceph-deploy mon create-initial

sudo ceph -s
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 71s)
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     

ceph-deploy disk list cn1 cn2 cn3
for i in cn1 cn2 cn3; do ceph-deploy osd create --data /dev/vdb $i; done
[vagrant@cn1 ceph]$ sudo ceph -s
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_WARN
            no active mgr
            mons are allowing insecure global_id reclaim
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 6m)
    mgr: no daemons active
    osd: 3 osds: 3 up (since 12s), 3 in (since 12s)
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     

sudo ceph-deploy mgr create cn1 cn2 cn
sudo ceph config set mon auth_allow_insecure_global_id_reclaim false
[vagrant@cn1 ceph]$ sudo ceph -s
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 13m)
    mgr: cn1(active, since 2m), standbys: cn2, cn3
    osd: 3 osds: 3 up (since 9h), 3 in (since 9h)
 
  data:
    pools:   1 pools, 64 pgs
    objects: 4 objects, 35 B
    usage:   3.0 GiB used, 117 GiB / 120 GiB avail
    pgs:     64 active+clean


#cluster OK ;)  health_OK
```
# Export d'une image
```

# création pool et image

sudo ceph osd pool create prbd 64 64
sudo rbd pool init prbd
sudo rbd create --size 20G --image-feature layering,exclusive-lock prbd/foo


# préparation du client cephclt
scp /etc/yum.repos.d/ceph.repo root@cephclt:/etc/yum.repos.d/ceph.repo
ssh root@cephclt yum install -y ceph-common
ceph-deploy admin cephclt

# mountage rbd
ssh root@cephclt
[root@cephclt ~]# rbd  device map prbd/foo
/dev/rbd0
[root@cephclt ~]# mkfs.xfs /dev/rbd/prbd/foo
..
[root@cephclt ~]# mount /dev/rbd0 /mnt
[root@cephclt ~]# df -h /mnt
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0        20G   33M   20G   1% /mnt

[root@cephclt ~]# ls -l /etc/ceph/ >/mnt/lsceph.txt
[root@cephclt ~]# cat /mnt/lsceph.txt
total 12
-rw-------. 1 root root 151 31 mai   04:46 ceph.client.admin.keyring
-rw-r--r--. 1 root root 261 31 mai   04:46 ceph.conf
-rw-r--r--. 1 root root  92 13 mai   17:50 rbdmap
-rw-------. 1 root root   0 31 mai   04:46 tmp7PDYYz
[root@cephclt ~]# exit

# cluster Nautilus ok et un client en version Nautilus a monté une image rbd
```
# Mise a jours vers ceph octopus
Avec le changement d'outils de management du cluster Ceph de ceph-deploy qui est en python2 vers cehadm codé en python3, l'équipe Ceph a rejouter de nouvelles fonctionnalitées tres intéréssantes que l'on va découvrir ici.
## Documentation
Pour plus d'information voir la documentation officiel.
https://docs.ceph.com/en/latest/cephadm/adoption/
## Première étape, cephadm
```
#install cephadm
[vagrant@cn1 ceph]$ sudo yum install https://download.ceph.com/rpm-octopus/el7/noarch/cephadm-15.2.13-0.el7.noarch.rpm
[vagrant@cn1 ceph]$ sudo ceph config assimilate-conf -i /etc/ceph/ceph.conf

[global]
	fsid = dbb88f95-1cd6-482a-a735-23c832ee7795
	mon_host = 192.168.0.11,192.168.0.12,192.168.0.13
	mon_initial_members = cn1, cn2, cn3
[vagrant@cn1 ceph]$ sudo ceph config dump
WHO    MASK LEVEL    OPTION                                VALUE          RO 
global      advanced auth_client_required                  cephx          *  
global      advanced auth_cluster_required                 cephx          *  
global      advanced auth_service_required                 cephx          *  
global      advanced public_network                        192.168.0.0/24 *  
  mon       advanced auth_allow_insecure_global_id_reclaim false             

[vagrant@cn1 ceph]$ sudo cephadm prepare-host
Verifying podman|docker is present...
Installing packages ['podman']...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
podman|docker (/bin/podman) is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK

[vagrant@cn1 ceph]$ sudo cephadm ls
[
    {
        "style": "legacy",
        "name": "mgr.cn1",
        "fsid": "dbb88f95-1cd6-482a-a735-23c832ee7795",
        "systemd_unit": "ceph-mgr@cn1",
        "enabled": true,
        "state": "running",
        "host_version": "14.2.21"
    },
    {
        "style": "legacy",
        "name": "osd.0",
        "fsid": "dbb88f95-1cd6-482a-a735-23c832ee7795",
        "systemd_unit": "ceph-osd@0",
        "enabled": true,
        "state": "running",
        "host_version": "14.2.21"
    },
    {
        "style": "legacy",
        "name": "mon.cn1",
        "fsid": "dbb88f95-1cd6-482a-a735-23c832ee7795",
        "systemd_unit": "ceph-mon@cn1",
        "enabled": true,
        "state": "running",
        "host_version": "14.2.21"
    }
]

[vagrant@cn1 ceph]$ cephadm adopt --style legacy --name mon.cn1
ERROR: cephadm should be run as root
[vagrant@cn1 ceph]$ sudo cephadm adopt --style legacy --name mon.cn1
Pulling container image docker.io/ceph/ceph:v15...
Stopping old systemd unit ceph-mon@cn1...
Disabling old systemd unit ceph-mon@cn1...
Moving data...
Chowning content...
Moving logs...
Creating new units...
[vagrant@cn1 ceph]$ sudo cephadm adopt --style legacy --name mgr.cn1
Pulling container image docker.io/ceph/ceph:v15...
Stopping old systemd unit ceph-mgr@cn1...
Disabling old systemd unit ceph-mgr@cn1...
Moving data...
Chowning content...
Moving logs...
Creating new units...
[vagrant@cn1 ceph]$ sudo cephadm adopt --style legacy --name osd.0
Pulling container image docker.io/ceph/ceph:v15...
Found online OSD at //var/lib/ceph/osd/ceph-0/fsid
objectstore_type is bluestore
Stopping old systemd unit ceph-osd@0...
Disabling old systemd unit ceph-osd@0...
Moving data...
Chowning content...
Chowning /var/lib/ceph/dbb88f95-1cd6-482a-a735-23c832ee7795/osd.0/block...
Disabling host unit ceph-volume@ lvm unit...
Moving logs...
Creating new units...
# vérification

[vagrant@cn1 ceph]$ sudo cephadm ls
[
    {
        "style": "cephadm:v1",
        "name": "mon.cn1",
        "fsid": "dbb88f95-1cd6-482a-a735-23c832ee7795",
        "systemd_unit": "ceph-dbb88f95-1cd6-482a-a735-23c832ee7795@mon.cn1",
        "enabled": true,
        "state": "running",
        "container_id": "bff612447838aaaf9a63f5dacdf59138c8301433dc0f0cad91b15916b13a10fb",
        "container_image_name": "docker.io/ceph/ceph:v15",
        "container_image_id": "2cf504fded3980c76b59a354fca8f301941f86e369215a08752874d1ddb69b73",
        "version": "15.2.13",
        "started": "2021-05-31T05:28:34.498631Z",
        "created": null,
        "deployed": "2021-05-31T05:28:33.768721Z",
        "configured": null
    },
    {
        "style": "cephadm:v1",
        "name": "mgr.cn1",
        "fsid": "dbb88f95-1cd6-482a-a735-23c832ee7795",
        "systemd_unit": "ceph-dbb88f95-1cd6-482a-a735-23c832ee7795@mgr.cn1",
        "enabled": true,
        "state": "running",
        "container_id": "4f9ecdda9d0d14b69c77c278e5b0e733efa22f1c7f134b9aec5d8637576deb38",
        "container_image_name": "docker.io/ceph/ceph:v15",
        "container_image_id": "2cf504fded3980c76b59a354fca8f301941f86e369215a08752874d1ddb69b73",
        "version": "15.2.13",
        "started": "2021-05-31T05:33:32.667534Z",
        "created": null,
        "deployed": "2021-05-31T05:33:32.286334Z",
        "configured": null
    },
    {
        "style": "cephadm:v1",
        "name": "osd.0",
        "fsid": "dbb88f95-1cd6-482a-a735-23c832ee7795",
        "systemd_unit": "ceph-dbb88f95-1cd6-482a-a735-23c832ee7795@osd.0",
        "enabled": true,
        "state": "running",
        "container_id": "5fe7219bb9c93f2b81017c9f1519171218d1ceca0d987979a8faafde751a7b41",
        "container_image_name": "docker.io/ceph/ceph:v15",
        "container_image_id": "2cf504fded3980c76b59a354fca8f301941f86e369215a08752874d1ddb69b73",
        "version": "15.2.13",
        "started": "2021-05-31T05:33:49.020766Z",
        "created": null,
        "deployed": "2021-05-31T05:33:47.502214Z",
        "configured": null
    }
]

```
## Activation de la gestion de l'orchestrateur cephadm
```
# activer cephadm
[vagrant@cn1 ceph]$ sudo ceph mgr module enable cephadm --force
[vagrant@cn1 ceph]$ sudo ceph orch set backend cephadm

# generer la clé SSH
[vagrant@cn1 ceph]$ sudo ceph cephadm generate-key
[vagrant@cn1 ceph]$ sudo ceph cephadm get-pub-key > ~/ceph.pub
# copie la clé SSH sur tous les nodes du cluster, pas sur les clients
[vagrant@cn1 ceph]$ sudo ssh-copy-id -f -i ~/ceph.pub root@cn1
Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@cn1'"
and check to make sure that only the key(s) you wanted were added.

[vagrant@cn1 ceph]$ sudo ssh-copy-id -f -i ~/ceph.pub root@cn2
...
Now try logging into the machine, with:   "ssh 'root@cn2'"

[vagrant@cn1 ceph]$ sudo ssh-copy-id -f -i ~/ceph.pub root@cn3
...
Now try logging into the machine, with:   "ssh 'root@cn3'"

[vagrant@cn1 ceph]$ sudo ssh-copy-id -f -i ~/ceph.pub root@cn4
...
Now try logging into the machine, with:   "ssh 'root@cn4'"

# ajout premier node dans l'orchestrateur ceph
[vagrant@cn1 ceph]$ sudo ceph orch host add cn1
Added host 'cn1'
# verification
[vagrant@cn1 ceph]$ sudo ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
[vagrant@cn1 ceph]$ sudo ceph orch ps
NAME     HOST  STATUS         REFRESHED  AGE  VERSION  IMAGE NAME               IMAGE ID      CONTAINER ID  
mgr.cn1  cn1   running (12m)  2m ago     2m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  4f9ecdda9d0d  
mon.cn1  cn1   running (17m)  2m ago     2m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  bff612447838  
osd.0    cn1   running (12m)  2m ago     2m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  5fe7219bb9c9  
[vagrant@cn1 ceph]$ sudo ceph versions
{
    "mon": {
        "ceph version 14.2.21 (5ef401921d7a88aea18ec7558f7f9374ebd8f5a6) nautilus (stable)": 2,
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 1
    },
    "mgr": {
        "ceph version 14.2.21 (5ef401921d7a88aea18ec7558f7f9374ebd8f5a6) nautilus (stable)": 2,
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 1
    },
    "osd": {
        "ceph version 14.2.21 (5ef401921d7a88aea18ec7558f7f9374ebd8f5a6) nautilus (stable)": 2,
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 1
    },
    "mds": {},
    "overall": {
        "ceph version 14.2.21 (5ef401921d7a88aea18ec7558f7f9374ebd8f5a6) nautilus (stable)": 6,
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 3
    }
}


# commentaire: le premier node est migré sous octopus avec le support podman sous CT7
```
## Adoption des autres nodes du cluster
```
# ajouter les nodes suivants
[vagrant@cn1 ceph]$ ssh root@cn2
[root@cn2 ~]# yum install https://download.ceph.com/rpm-octopus/el7/noarch/cephadm-15.2.13-0.el7.noarch.rpm
[root@cn2 ~]# cephadm prepare-host
[root@cn2 ~]# cephadm ls
[root@cn2 ~]# cephadm adopt --style legacy --name mon.cn2
[root@cn2 ~]# cephadm adopt --style legacy --name mgr.cn2
[root@cn2 ~]# cephadm adopt --style legacy --name osd.1
[root@cn2 ~]# ceph orch host add cn2
Added host 'cn2'
[root@cn2 ~]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2      
#verification
[root@cn2 ~]# ceph -s
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_WARN
            3 stray daemon(s) not managed by cephadm
            1 stray host(s) with 3 daemon(s) not managed by cephadm
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 2m)
    mgr: cn1(active, since 27m), standbys: cn3, cn2
    osd: 3 osds: 3 up (since 2m), 3 in (since 10h)
 
  data:
    pools:   2 pools, 65 pgs
    objects: 24 objects, 14 MiB
    usage:   3.1 GiB used, 117 GiB / 120 GiB avail
    pgs:     65 active+clean
  
# remarque : ca progresse, reste a faire le dernier node

[root@cn2 ~]# exit
logout
Connection to cn2 closed.

# ajout du dernier node
[vagrant@cn1 ceph]$ ssh root@cn3
[root@cn3 ~]# 
[root@cn3 ~]# yum install https://download.ceph.com/rpm-octopus/el7/noarch/cephadm-15.2.13-0.el7.noarch.rpm
[root@cn3 ~]# cephadm prepare-host
[root@cn3 ~]# cephadm ls
[root@cn3 ~]# cephadm adopt --style legacy --name mon.cn3
[root@cn3 ~]# cephadm adopt --style legacy --name mgr.cn3
[root@cn3 ~]# cephadm adopt --style legacy --name osd.2
[root@cn3 ~]# ceph orch host add cn3
Added host 'cn3'
[root@cn3 ~]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3     
```
## Vérification de la migration
```
# verification
[root@cn3 ~]# ceph -s
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 97s)
    mgr: cn1(active, since 35m), standbys: cn2, cn3
    osd: 3 osds: 3 up (since 85s), 3 in (since 10h)
 
  data:
    pools:   2 pools, 65 pgs
    objects: 24 objects, 14 MiB
    usage:   3.1 GiB used, 117 GiB / 120 GiB avail
    pgs:     65 active+clean
 

# remarque : youpi cluster est health_ok et on a toujours nos 3 mon 3 mgr et 3 osd
[root@cn3 ~]# ceph versions
{
    "mon": {
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 3
    },
    "mgr": {
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 3
    },
    "osd": {
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 3
    },
    "mds": {},
    "overall": {
        "ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)": 9
    }
}

# remarque: le cluster est en version octopus

[vagrant@cn1 ceph]$ ceph --version
ceph version 14.2.21 (5ef401921d7a88aea18ec7558f7f9374ebd8f5a6) nautilus (stable)

# remarque: le binaire de ceph est toujours en version nautilus , car la version octopus est dans les conteneurs
[vagrant@cn1 ceph]$ sudo podman ps
CONTAINER ID  IMAGE                    COMMAND               CREATED         STATUS             PORTS  NAMES
5fe7219bb9c9  docker.io/ceph/ceph:v15  -n osd.0 -f --set...  45 minutes ago  Up 45 minutes ago         ceph-dbb88f95-1cd6-482a-a735-23c832ee7795-osd.0
4f9ecdda9d0d  docker.io/ceph/ceph:v15  -n mgr.cn1 -f --s...  45 minutes ago  Up 45 minutes ago         ceph-dbb88f95-1cd6-482a-a735-23c832ee7795-mgr.cn1
bff612447838  docker.io/ceph/ceph:v15  -n mon.cn1 -f --s...  50 minutes ago  Up 50 minutes ago         ceph-dbb88f95-1cd6-482a-a735-23c832ee7795-mon.cn1
```
# Upgrade du client
```
[vagrant@cn1 ceph]$ ssh root@cephclt
Last login: Mon May 31 05:09:33 2021 from 192.168.0.11
[root@cephclt ~]# 
[root@cephclt ~]# cat /mnt/lsceph.txt
total 12
-rw-------. 1 root root 151 31 mai   04:46 ceph.client.admin.keyring
-rw-r--r--. 1 root root 261 31 mai   04:46 ceph.conf
-rw-r--r--. 1 root root  92 13 mai   17:50 rbdmap
-rw-------. 1 root root   0 31 mai   04:46 tmp7PDYYz

[root@cephclt ~]# sed -i 's/rpm-nautilus/rpm-octopus/g' /etc/yum.repos.d/ceph.repo
[root@cephclt ~]# yum -y update
[root@cephclt ~]# ceph --version
ceph version 15.2.13 (c44bc49e7a57a87d84dfff2a077a2058aa2172e2) octopus (stable)
[root@cephclt ~]# 
[root@cephclt ~]# cat /mnt/lsceph.txt 
total 12
-rw-------. 1 root root 151 31 mai   04:46 ceph.client.admin.keyring
-rw-r--r--. 1 root root 261 31 mai   04:46 ceph.conf
-rw-r--r--. 1 root root  92 13 mai   17:50 rbdmap
-rw-------. 1 root root   0 31 mai   04:46 tmp7PDYYz

# remarque : les binnaires sont mise a jour. pensez a remoter les volumes pour qu'ils utilisent les bons binnaires.
[root@cephclt ~]# umount /mnt
[root@cephclt ~]# rbd device list
id  pool  namespace  image  snap  device   
0   prbd             foo    -     /dev/rbd0
[root@cephclt ~]# rbd device unmap /dev/rbd0
[root@cephclt ~]# rbd  device map prbd/foo
/dev/rbd0
[root@cephclt ~]# mount /dev/rbd/prbd/foo  /mnt
[root@cephclt ~]# df -h /mnt
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0        20G   33M   20G   1% /mnt

# remarque : c'est remonté en version octopus.

[root@cephclt ~]# exit
logout
Connection to cephclt closed.
[vagrant@cn1 ceph]$ 
```
# Ajout d'un nouveau node
A présent vérifiont qu'il soit possible d'agrandir ce cluster avec les commandes cephadm. le nouveau node sera en CentOS8
```
[vagrant@cn1 ceph]$ sudo ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
# remarque : il y a 3 nodes en CT7
[vagrant@cn1 ceph]$ sudo ceph orch host add cn4
Error EINVAL: New host cn4 (cn4) failed check: ['systemctl is present', 'lvcreate is present', 'Unit chronyd.service is enabled and running', 'Hostname "cn4" matches what is expected.', "ERROR: Unable to locate any of ['podman', 'docker']"]
# remarque: il faut préparer le node avant
[vagrant@cn1 ceph]$ ssh root@cn4
Last login: Mon May 31 10:01:11 2021 from 192.168.0.11
[root@cn4 ~]# cat /etc/redhat-release 
CentOS Linux release 8.3.2011
[root@cn4 ~]# dnf install https://download.ceph.com/rpm-octopus/el8/noarch/cephadm-15.2.13-0.el8.noarch.rpm
# remarque : install de podman, runc, python3
[root@cn4 ~]# cephadm  prepare-host
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
podman|docker (/usr/bin/podman) is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK
# remarque: ca devrait etre ok maintenant. la clé ssh avait été ajouté précédament.
[root@cn4 ~]# exit
logout
Connection to cn4 closed.
[vagrant@cn1 ceph]$ sudo ceph orch host add cn4
Added host 'cn4'
[vagrant@cn1 ceph]$ sudo ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
cn4   cn4                   
[vagrant@cn1 ceph]$ sudo ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...
[vagrant@cn1 ceph]$ sudo ceph -w
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 4h)
    mgr: cn1(active, since 4h), standbys: cn2, cn3
    osd: 3 osds: 3 up (since 4h), 3 in (since 14h)
 
  data:
    pools:   2 pools, 65 pgs
    objects: 24 objects, 14 MiB
    usage:   3.1 GiB used, 117 GiB / 120 GiB avail
    pgs:     65 active+clean
 

2021-05-31 10:22:17.657291 mon.cn1 [INF] osd.6 [v2:192.168.0.11:6810/1937893815,v1:192.168.0.11:6811/1937893815] boot
2021-05-31 10:22:17.657342 mon.cn1 [INF] osd.3 [v2:192.168.0.13:6808/1544026126,v1:192.168.0.13:6809/1544026126] boot
2021-05-31 10:22:17.657355 mon.cn1 [INF] osd.4 [v2:192.168.0.12:6808/2920888063,v1:192.168.0.12:6809/2920888063] boot
2021-05-31 10:22:20.847034 mon.cn1 [INF] osd.5 [v2:192.168.0.14:6800/1970775612,v1:192.168.0.14:6801/1970775612] boot
2021-05-31 10:22:21.830094 mon.cn1 [WRN] Health check failed: Reduced data availability: 2 pgs peering (PG_AVAILABILITY)
2021-05-31 10:22:23.427665 mon.cn1 [INF] osd.7 [v2:192.168.0.14:6808/4259384539,v1:192.168.0.14:6809/4259384539] boot
2021-05-31 10:22:24.508468 osd.5 [INF] 1.16 continuing backfill to osd.0 from (0'0,28'94] MIN to 28'94
2021-05-31 10:22:26.044496 mon.cn1 [WRN] Health check failed: Degraded data redundancy: 1 pg degraded (PG_DEGRADED)
2021-05-31 10:22:29.266863 mon.cn1 [WRN] Health check update: Reduced data availability: 1 pg peering (PG_AVAILABILITY)
2021-05-31 10:22:31.384020 mon.cn1 [INF] Health check cleared: PG_AVAILABILITY (was: Reduced data availability: 1 pg peering)
2021-05-31 10:22:31.385396 mon.cn1 [INF] Health check cleared: PG_DEGRADED (was: Degraded data redundancy: 1 pg degraded)
2021-05-31 10:22:31.385471 mon.cn1 [INF] Cluster is now healthy
[vagrant@cn1 ceph]$ sudo ceph orch ls
NAME                       RUNNING  REFRESHED  AGE  PLACEMENT    IMAGE NAME               IMAGE ID      
mgr                            3/0  3m ago     -    <unmanaged>  docker.io/ceph/ceph:v15  2cf504fded39  
mon                            3/0  3m ago     -    <unmanaged>  docker.io/ceph/ceph:v15  2cf504fded39  
osd.all-available-devices      4/4  3m ago     4m   *            docker.io/ceph/ceph:v15  2cf504fded39  
[vagrant@cn1 ceph]$ sudo ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME      STATUS  REWEIGHT  PRI-AFF
-1         0.35156  root default                            
-3         0.08789      host cn1                            
 0    hdd  0.03909          osd.0      up   1.00000  1.00000
 6    hdd  0.04880          osd.6      up   1.00000  1.00000
-5         0.08789      host cn2                            
 1    hdd  0.03909          osd.1      up   1.00000  1.00000
 4    hdd  0.04880          osd.4      up   1.00000  1.00000
-7         0.08789      host cn3                            
 2    hdd  0.03909          osd.2      up   1.00000  1.00000
 3    hdd  0.04880          osd.3      up   1.00000  1.00000
-9         0.08789      host cn4                            
 5    hdd  0.03909          osd.5      up   1.00000  1.00000
 7    hdd  0.04880          osd.7      up   1.00000  1.00000

[vagrant@cn1 ceph]$ sudo ceph orch ps
NAME     HOST  STATUS        REFRESHED  AGE  VERSION  IMAGE NAME               IMAGE ID      CONTAINER ID  
mgr.cn1  cn1   running (2m)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  d78985f2032c  
mgr.cn2  cn2   running (4h)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  5e58edab7675  
mgr.cn3  cn3   running (4h)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  d9ed52328267  
mon.cn1  cn1   running (2m)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  4378311bf0b5  
mon.cn2  cn2   running (4h)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  9d3ea16eb844  
mon.cn3  cn3   running (4h)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  c684ce6ee3a8  
osd.0    cn1   running (2m)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  442c4c066321  
osd.1    cn2   running (4h)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  7151b9932f25  
osd.2    cn3   running (4h)  2m ago     4h   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  9d2855b02a97  
osd.3    cn3   running (9m)  2m ago     9m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  1331e87f0f3a  
osd.4    cn2   running (9m)  2m ago     9m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  44b6c825e91a  
osd.5    cn4   running (9m)  2m ago     9m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  73d8899f1971  
osd.6    cn1   running (2m)  2m ago     9m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  eb151c982896  
osd.7    cn4   running (9m)  2m ago     9m   15.2.13  docker.io/ceph/ceph:v15  2cf504fded39  3a34e31ab037  
[vagrant@cn1 ceph]$ sudo ceph -s
  cluster:
    id:     dbb88f95-1cd6-482a-a735-23c832ee7795
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn2,cn3 (age 2m)
    mgr: cn1(active, since 3m), standbys: cn2, cn3
    osd: 8 osds: 8 up (since 2m), 8 in (since 10m)
 
  data:
    pools:   2 pools, 65 pgs
    objects: 24 objects, 14 MiB
    usage:   8.3 GiB used, 352 GiB / 360 GiB avail
    pgs:     65 active+clean
# remarque : il y a bien les 8 osd.
# remarque : pourquoi cela n'affiche que 4/4 osd dans la commande ceph orch ls, alors qu'il y a bien 8 osd...
```
