# Cài đặt Galera Cluster Keepalived với 2 node

## IP Plan

| hostname          | ip           | VIP          |
|-------------------|--------------|--------------|
| msp-galera-node01 | 172.16.20.68 | 172.16.20.70 |
| msp-galera-node02 | 172.16.20.68 | 172.16.20.70 |

## Cài đặt các thành phần
### MariaDB Package Repository Setup Script

`curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash`

`sudo apt update`

### Install common package

`sudo apt-get install mariadb-server galera-4 mariadb-client libmariadb3 mariadb-backup mariadb-common`

### Cài đặt arbitrator (garbd) để hỗ trợ 2-node cluster
`sudo apt install galera-arbitrator-4`

### Cài đặt KeepAlived

`sudo apt install keepalived`

## Thiết lập hostname

trên node1: `hostnamectl set-hostname msp-galera-node01`
trên node2: `hostnamectl set-hostname msp-galera-node02`

## Cấu hình Galera và Mariadb
### Cấu hình trên node 1

`vim /etc/mysql/conf.d/galera.cnf`

```bash
# Galera settings
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="galera_test_cluster"
wsrep_cluster_address="gcomm://172.16.20.68,172.16.20.69"

# Galera Node Configuration
wsrep_node_name="msp-galera-node01"
wsrep_node_address="172.16.20.68"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# MySQL settings
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
```

### Cấu hình trên node 2

`vim /etc/mysql/conf.d/galera.cnf`

```bash
# Galera settings
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="galera_test_cluster"
wsrep_cluster_address="gcomm://172.16.20.68,172.16.20.69"

# Galera Node Configuration
wsrep_node_name="msp-galera-node02"
wsrep_node_address="172.16.20.69"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# MySQL settings
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
```

### Cấu hình MariaDB trên cả 2 node

`vim /etc/mysql/mariadb.conf.d/50-server.cnf`

tìm `bind-address` và sửa lại như sau:

```bash
bind-address = 0.0.0.0
```

### Khởi chạy cluster trên node 1:

`sudo galera_new_cluster`

# Cấu hình arbitrator

**Thực hiện trên cả 2 node**

`vi /etc/default/garb`

```bash
GALERA_NODES="172.16.20.68:4567,172.16.20.69:4567"
GALERA_GROUP="galera_test_cluster"
GALERA_OPTIONS="pc.recovery=TRUE;gmcast.listen_addr=tcp://0.0.0.0:4444"
```

## Kiểm tra node
`mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"`
`mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"`

# Cấu hình Keepalived

`tạo script check mariadb running`

```bash
#!/bin/bash
nc -zv 127.0.0.1 3306
if [ $? -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

## Trên Node Master thực hiện cấu hình


```bash
global_defs {
    script_user root
    script_security 1
}
vrrp_script chk_mysql {
    script "/etc/keepalived/check_mariadb.sh"
    interval 2
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass mypassw
    }

    virtual_ipaddress {
        172.16.20.70
    }

    track_script {
        chk_mysql
    }
}
```

## trên node backup thực hiện cấu hình

```bash
global_defs {
    script_user root
    script_security 1
}
vrrp_script chk_mysql {
    script "/etc/keepalived/check_mariadb.sh"
    interval 2
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass mypassw
    }

    virtual_ipaddress {
        172.16.20.70
    }

    track_script {
        chk_mysql
    }
}
```
