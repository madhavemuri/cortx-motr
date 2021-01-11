Manual Setup of LR R2 cluster
=============================

Super Micro Setup,

- smc65-m14.colo.seagate.com
- smc66-m14.colo.seagate.com
- smc67-m14.colo.seagate.com

Dell Setup,

- dell05-m02.colo.seagate.com
- dell06-m02.colo.seagate.com
- dell07-m02.colo.seagate.com


1) Supported OS and kernel version,
==================================

# cat /etc/redhat-release
 CentOS Linux release 7.8.2003 (Core)
 
# uname -r
 3.10.0-1127.el7.x86_64
 
2) Passwordless login setup between nodes
=========================================

# ssh-keygen

# ssh-copy-id root@sm66-m14.colo.seagate.com

# ssh smc66-m14.pun.seagate.com

# ssh-copy-id root@smc67-m14.colo.seagate.com

# ssh smc67-m14.colo.seagate.com
 
 Check passwordless login between all the nodes.

3) Create volumes in 5u84
==========================

   Refer https://github.com/Seagate/cortx-prvsnr/wiki/Seagate-Gallium-Storage-Controller-Configuration
   
   # wget https://github.com/Seagate/cortx-prvsnr/blob/pre-cortx-1.0/srv/components/controller/files/scripts/controller-cli.sh
   
  - Check whether 5u84 ip's 10.0.0.2 or 10.0.0.3 are reachable or not
       If it's not working then check status of scsi-network-relay
	  # systemctl status scsi-network-relay
	  
	  # systemctl start scsi-network-relay
	  
  - To see the current volumes setup in the 5u84
     # ./controller-cli.sh host -h 10.0.0.2 -u manage -p '!Manage20' prov -s
      OR
      
     # ssh -l manage 10.0.0.2
     
     > show-volumes
      
  Then cleanup old volumes and create them as per LR R2 requirement,
  
  # ./controller-cli.sh host -h 10.0.0.2 -u manage -p '!Manage20' prov -c -a
  
4) Multipath setup
==================

   # yum install device-mapper-multipath
   
   Update /etc/multipath.conf
   
   # systemctl start multipathd.service
   
   # Validate with "multipath -ll"
   
5) Check storage throughput with fio
=====================================

   # yum install fio
   
   # fio write-mpath.fio
   
   # fio read-mpath.fio

6) Validate Networking,
=======================
  #  show_gids
         
  Node-1,
  #  ibv_rc_pingpong -d mlx5_3 -g 3
  
  Node-2,
  #  ibv_rc_pingpong -d mlx5_3 -g 3 192.168.80.

  Validate all the network interfaces (IB devices : mlx5_0, mlx5_1, mlx5_2, mlx5_3)

 Check iperf thouthput,
 
 # yum install iperf
  
  Node-1,
  #  iperf -s -B 192.168.49.15
  
  Node-2,
  #  iperf -s -B 192.168.49.12 -c 192.168.49.15 -P 8
  
 Libfabrics Throughput,
 
  # yum install libfabrics fabtests
  
  Node-1,
  # fi_msg_bw -p sockets|verbs -s 192.168.49.15 -I 100000 -S all
  
  Node-2,
  # fi_msg_bw -p sockets|verbs  192.168.49.15 -I 100000 -S all
  
7) Addition of Cortx Repo
=========================

    # yum-config-manager --add-repo=http://cortx-storage.colo.seagate.com/releases/cortx_builds/centos-7.8.2003/538/3rd_party/
    
    #  yum-config-manager --add-repo=http://cortx-storage.colo.seagate.com/releases/cortx_builds/centos-7.8.2003/538/cortx_iso/
    
    #  yum install cortx-motr --nogpgcheck
    
    #  yum install cortx-hare --nogpgcheck
    
    #  yum install cortx-s3server --nogpgcheck
    
8) Motr setup
==============

   Lnet setup,
   
   # yum install http://cortx-storage.colo.seagate.com/releases/cortx_builds/centos-7.8.2003/538/3rd_party/lustre/custom/o2ib/kmod-lustre-client-2.12.4.2_171_g9356888-1.el7.x86_64.rpm
    
   # yum install http://cortx-storage.colo.seagate.com/releases/cortx_builds/centos-7.8.2003/538/3rd_party/lustre/custom/o2ib/lustre-client-2.12.4.2_171_g9356888-1.el7.x86_64.rpm
  
   update Lnet interfaces as per interface being used enp175s0f1 and tcp,
   
   # vi /etc/modprobe.d/lnet.conf
   
   # systemctl restart lnet
   
   # lctl list_nids
   
   Check with lctl ping <nid>, if ping is not working then do the following steps. 
    - wget https://raw.githubusercontent.com/Mellanox/gpu_direct_rdma_access/master/arp_announce_conf.sh
   
    - sh arp_announce_conf.sh
   
    - sysctl -w net.ipv4.conf.all.rp_filter=0
   
    - sysctl -w net.ipv4.conf.p1p1.rp_filter=0
   
    - sysctl -w net.ipv4.conf.p1p2.rp_filter=0
    
    - sysctl -w net.ipv4.conf.p2p1.rp_filter=0
   
    - sysctl -w net.ipv4.conf.p2p2.rp_filter=0
   
    - ip neigh flush all
   
    -  systemctl restart lnet
 
9) Hare + Motr Setup
=====================
 
    # yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
    
    # yum -y install consul-1.7.8
    
    #  hctl bootstrap --mkfs cluster-mp-R1.yaml
    
    #  /opt/seagate/cortx/hare/libexec/m0crate-io-conf > crate.yaml
    
    #  dd if=/dev/urandom of=/tmp/128M bs=1M count=128
    
    #  m0crate -S crate.yaml
    
    #  hctl shutdown
	
10) S3 setup and do the IO
==========================
 
  - Setup HAproxy,
   
    # yum install haproxy
   
    # cp /opt/seagate/cortx/s3/install/haproxy/haproxy_osver7.cfg /etc/haproxy/haproxy.cfg
   
    # mkdir /etc/haproxy/errors/
   
    # cp /opt/seagate/cortx/s3/install/haproxy/503.http /etc/haproxy/errors/
   
    # Update haproxy config for number s3server instances, /etc/haproxy/haproxy.cfg
   
    # systemctl restart haproxy
   
  - Setup Ldap,
   
    # cd /opt/seagate/cortx/s3/install/ldap/
   
    # ./setup_ldap.sh --defaultpasswd --skipssl --forceclean
   
    # systemctl restart slapd
   
  
    #  vim /etc/hosts
      127.0.0.1 - s3.seagate.com sts.seagate.com iam.seagate.com sts.cloud.seagate.com
  
    #  systemctl restart s3authserver
   
    #  yum install -y http://cortx-storage.colo.seagate.com/releases/cortx/github/integration-custom-ci/release/centos-7.8.2003/custom-build-338/cortx_iso/cortx-s3iamcli-1.0.0-2301_gitfd8fbcd2.noarch.rpm
   
  - Setup S3 client,
   
    #  easy_install pip
   
    #  pip install awscli
   
    #  pip install awscli-plugin-endpoint
   
    #  aws configure
   
    #  aws configure set plugins.endpoint awscli_plugin_endpoint
   
    #  aws configure set s3.endpoint_url http://s3.seagate.com
   
    #  aws configure set s3api.endpoint_url http://s3.seagate.com
  
    #  s3iamcli CreateAccount -n motr -e motr@seagate.com --ldapuser sgiamadmin --ldappasswd ldapadmin
	
    #  aws s3 ls
   
    #  aws s3 mb s3://testbucket1
   
    #  aws s3 cp /tmp/128M s3://testbucket1
   
  - Run s3bench,
   
    #  yum install go
   
    #  go get github.com/igneous-systems/s3bench
   
    # /root/go/bin/s3bench -accessKey AKIAwk0geOx8SnCfiveCzzV0Uw -accessSecret rD2FQIeMf0ZBsiEaKfAEafeNoP0K/B9p0Bm+ox3+ -bucket firstbucket7 -endpoint http://s3.seagate.com -numClients 512 -numSamples 4096 -objectSize 134217728b  -verbose
  
References:
===========

- https://github.com/Seagate/cortx/blob/main/doc/scaleout/README.rst
- https://github.com/Seagate/cortx/blob/main/doc/Cluster_Setup.md
- https://github.com/Seagate/cortx-s3server/blob/main/docs/R2-setup/3%20Node%20Manual%20Hare%20Cluster%20Setup.md
- https://seagatetechnology.sharepoint.com/:w:/r/sites/gteamdrv1/tdrive1224/_layouts/15/guestaccess.aspx?e=D9OLR8&share=EfvqJjlha8pNkOeIfCgDywUBn0YNIYGT6-tWMREx9iGxpw
- https://github.com/Seagate/cortx-hare/blob/dev/README_developers.md 
- https://seagatetechnology.sharepoint.com/:w:/r/sites/gteamdrv1/tdrive1224/_layouts/15/Doc.aspx?sourcedoc=%7B8c2c808f-3b39-4331-9a93-c67009ae7fdd%7D&action=edit&wdPreviousSession=27cb5705-c553-4852-bdc1-6e01de59f8ef&cid=3e7b558b-8aa7-4fea-a50e-91816f8be2b9
	  
	  
Multipath configuration
=======================
	  
[root@smc65-m14 ~]# cat /etc/multipath.conf
 
defaults {
        polling_interval 10
        max_fds 8192
        user_friendly_names yes
        find_multipaths yes
}

devices {
        device {
                vendor "SEAGATE"
                product "*"
                path_grouping_policy group_by_prio
                uid_attribute "ID_SERIAL"
                prio alua
                path_selector "service-time 0"
                path_checker tur
                hardware_handler "1 alua"
                failback immediate
                rr_weight uniform
                rr_min_io_rq 1
                no_path_retry 18
        }
}

Cluster Description File
=========================

[root@smc65-m14 ~]# cat cluster-R2.yaml
# Cluster Description File (CDF).
# See `cfgen --help-schema` for the format description.

nodes:
  - hostname: smc65-m14.colo.seagate.com # [user@]hostname
    data_iface: enp175s0f0     # name of data network interface
    data_iface_type: o2ib   # LNet type of network interface (optional);
                            # supported values: "tcp" (default), "tcp"
    m0_servers:
      - runs_confd: true
        io_disks:
          data: []
      - io_disks:
          data:
          - /dev/mapper/mpatha
          - /dev/mapper/mpathb
          - /dev/mapper/mpathc
          - /dev/mapper/mpathd
          - /dev/mapper/mpathe
          - /dev/mapper/mpathf
          - /dev/mapper/mpathg
          - /dev/mapper/mpathh
      - io_disks:
          data:
          - /dev/mapper/mpathi
          - /dev/mapper/mpathj
          - /dev/mapper/mpathk
          - /dev/mapper/mpathl
          - /dev/mapper/mpathm
          - /dev/mapper/mpathn
          - /dev/mapper/mpatho
          - /dev/mapper/mpathp
    m0_clients:
        s3: 11           # number of S3 servers to start
        other: 2        # max quantity of other Mero clients this node may have
  - hostname: smc66-m14.colo.seagate.com
    data_iface: enp175s0f0
    data_iface_type: o2ib
    m0_servers:
      - runs_confd: true
        io_disks:
          data: []
      - io_disks:
          data:
          - /dev/mapper/mpatha
          - /dev/mapper/mpathb
          - /dev/mapper/mpathc
          - /dev/mapper/mpathd
          - /dev/mapper/mpathe
          - /dev/mapper/mpathf
          - /dev/mapper/mpathg
          - /dev/mapper/mpathh
      - io_disks:
          data:
          - /dev/mapper/mpathi
          - /dev/mapper/mpathj
          - /dev/mapper/mpathk
          - /dev/mapper/mpathl
          - /dev/mapper/mpathm
          - /dev/mapper/mpathn
          - /dev/mapper/mpatho
          - /dev/mapper/mpathp
    m0_clients:
        s3: 0           # number of S3 servers to start
        other: 2
  - hostname: smc67-m14.colo.seagate.com # [user@]hostname
    data_iface: enp175s0f0     # name of data network interface
    data_iface_type: o2ib   # LNet type of network interface (optional);
                            # supported values: "tcp" (default), "tcp"
    m0_servers:
      - runs_confd: true
        io_disks:
          data: []
      - io_disks:
          data:
          - /dev/mapper/mpatha
          - /dev/mapper/mpathb
          - /dev/mapper/mpathc
          - /dev/mapper/mpathd
          - /dev/mapper/mpathe
          - /dev/mapper/mpathf
          - /dev/mapper/mpathg
          - /dev/mapper/mpathh
      - io_disks:
          data:
          - /dev/mapper/mpathi
          - /dev/mapper/mpathj
          - /dev/mapper/mpathk
          - /dev/mapper/mpathl
          - /dev/mapper/mpathm
          - /dev/mapper/mpathn
          - /dev/mapper/mpatho
          - /dev/mapper/mpathp
    m0_clients:
        s3: 0           # number of S3 servers to start
        other: 2        # max quantity of other Mero clients this node may have
pools:
  - name: the pool
    type: sns
    data_units: 4
    parity_units: 2
    # allowed_failures: { site: 0, rack: 0, encl: 0, ctrl: 0, disk: 0 }

Fio workload,
=============

# cat write.fio

[global]
direct=1
ioengine=libaio
iodepth=16
;invalidate=1
ramp_time=5
runtime=60
time_based
bs=1M
rw=write
numjobs=32


[job0]
filename=/dev/mapper/mpatha
[job1]
filename=/dev/mapper/mpathb
[job2]
filename=/dev/mapper/mpathc
[job3]
filename=/dev/mapper/mpathd
[job4]
filename=/dev/mapper/mpathe
[job5]
filename=/dev/mapper/mpathf
[job6]
filename=/dev/mapper/mpathg
[job7]
filename=/dev/mapper/mpathh
[job8]
filename=/dev/mapper/mpathi
[job9]
filename=/dev/mapper/mpathj
[job10]
filename=/dev/mapper/mpathk
[job11]
filename=/dev/mapper/mpathl
[job12]
filename=/dev/mapper/mpathm
[job13]
filename=/dev/mapper/mpathn
[job14]
filename=/dev/mapper/mpatho
[job15]
filename=/dev/mapper/mpathp

Lnet self test
==============

# cat lst.sh

export LST_SESSION=$$
lst new_session twonoderead
lst add_group server 192.168.49.49@o2ib
lst add_group client 192.168.49.3@o2ib

lst add_batch bulk_rw
lst add_test --concurrency 90 --batch bulk_rw --from client --to server brw write size=1M
lst run bulk_rw
lst stat client server & sleep 30; kill $!
lst stop bulk_read
lst end_session

Run following command on both the nodes,

# modprobe lnet_selftest

# sh lst.sh
