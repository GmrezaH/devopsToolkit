# Pre-install

1. Check [Zabbix official requirements](https://www.zabbix.com/documentation/current/en/manual/installation/requirements)

1. Disable SELinux (or set to permissive) for simplicity in production, or configure SELinux policies for Zabbix and MariaDB if required by security policies

   ```bash
   sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
   sudo setenforce 0
   ```

1. Update the system

   ```bash
   sudo dnf update -y
   sudo dnf install -y bash-completion vim tmux git net-tools sysstat tree telnet epel-release
   ```

1. To avoid any issues with timestamps in logs and databases, set the system locale and timezone

   ```bash
   sudo timedatectl set-timezone Asia/Tehran
   sudo dnf install glibc-langpack-en
   sudo localectl set-locale LANG=en_US.UTF-8
   ```

1. Configure DNS or /etc/hosts for name resolution

   ```bash
   sudo cat<<EOF >> /etc/hosts
   192.168.7.101 zabbix-1.example.com zabbix-1
   192.168.7.102 zabbix-2.example.com zabbix-2
   192.168.7.103 zabbix-vip.example.com zabbix-vip
   EOF
   ```

1. Ensure firewall ports are open:

   - Zabbix server: 10051/TCP
   - Zabbix agent: 10050/TCP
   - MariaDB: 3306/TCP
   - Maxscale: 3307/TCP
   - Nginx (or httpd): 80/TCP, 443/TCP

# Install MariaDB

1. Add the [MariaDB Repository](https://mariadb.org/download/?t=repo-config) in **/etc/yum.repos.d/mariadb.repo**

   ```ini
   # https://mariadb.org/download/
   [mariadb]
   name = MariaDB
   baseurl = https://rpm.mariadb.org/11.4/rhel/$releasever/$basearch
   gpgkey = https://rpm.mariadb.org/RPM-GPG-KEY-MariaDB
   gpgcheck = 1
   ```

1. Install MariaDB server

   ```bash
   sudo dnf install -y mariadb-server mariadb
   sudo systemctl enable --now mariadb
   ```

1. Run the secure installation script

   ```bash
   sudo mariadb-secure-installation
   ```

   Follow the prompts to secure the database (set a strong root password, remove test users, disable remote root login, etc.).

1. Start and verify service

   ```bash
   sudo systemctl status mariadb
   sudo systemctl start mariadb
   mariadb -u root -p
   ```

## Configure MariaDB Replication (GTID-based)

### Master Configuration

1. Edit /etc/my.cnf.d/server.cnf

   ```ini
   [mariadb]
   server_id=1
   log-bin
   log_bin=mariadb-bin
   log-basename=master1
   binlog-format=mixed
   bind-address=192.168.7.101 # Master VM IP
   innodb_buffer_pool_size=2G # 70-80% of available RAM
   log_slave_updates=1
   ```

1. Restart Mariadb

   ```bash
   sudo systemctl restart mariadb
   ```

1. Check if `binary logging` and `log_slave_updates` is enabled

   ```sql
   SHOW GLOBAL VARIABLES LIKE 'log_bin';
   SHOW GLOBAL VARIABLES LIKE 'log_slave_updates';
   ```

1. Create replication user

   ```sql
   CREATE USER 'replication_user'@'%' IDENTIFIED BY 'bigs3cret';
   GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
   FLUSH PRIVILEGES;
   ```

1. Getting the Master's Binary Log Co-ordinates

   ```sql
   FLUSH TABLES WITH READ LOCK;
   SHOW GLOBAL VARIABLES LIKE 'gtid_binlog_pos';
   -- or
   SHOW MASTER STATUS;
   SELECT BINLOG_GTID_POS("master1-bin.000001", 600); -- values obtained from previous query
   ```

   Keep this session running - exiting it will release the lock.

1. Backup all databases and copy it to the Second VM (slave)

   Now, with the lock still in place, copy the data from the master to the slave

   ```bash
   mariadb-dump -uroot -p --all-databases > mariadb-master.sql
   scp mariadb-master.sql rocky@zabbix-2:~
   ```

### Slave Configuration

1. Edit /etc/my.cnf.d/server.cnf

   ```ini
   [mariadb]
   server_id=2
   log-bin
   log_bin=mariadb-bin
   log-basename=slave1
   binlog-format=mixed
   bind-address=192.168.7.102 # Slave VM IP
   innodb_buffer_pool_size=2G # 70-80% of available RAM
   log_slave_updates=1
   read_only=ON
   ```

1. Restart Mariadb

   ```bash
   sudo systemctl restart mariadb
   ```

1. Import MariaDB backup from First VM (master)

   ```bash
   mariadb -uroot -p < ~/mariadb-master.sql
   ```

1. Once the data has been copied, you can release the lock on the `master` with `UNLOCK TABLES;`

1. Configure GTID based replication

   ```sql
   SET GLOBAL gtid_slave_pos = "0-1-2";

   CHANGE MASTER TO
     MASTER_USE_GTID=slave_pos,
     MASTER_HOST='192.168.7.101',
     MASTER_USER='replication_user',
     MASTER_PASSWORD='bigs3cret',
     MASTER_PORT=3306,
     MASTER_CONNECT_RETRY=10;
   ```

1. Set enable read_only

   ```sql
   SET GLOBAL read_only=1;
   ```

1. Start the slave

   ```sql
   START SLAVE;
   ```

1. Check that the replication is working

   ```sql
   SHOW SLAVE STATUS \G
   ```

   If replication is working correctly, both the values of `Slave_IO_Running` and `Slave_SQL_Running` should be `Yes`:

   ```
   Slave_IO_Running: Yes
   Slave_SQL_Running: Yes
   ```

# Install Maxscale

1. Download and Install [Maxscale package](https://dlm.mariadb.com/browse/mariadbmaxscale/)

   ```bash
   sudo dnf install -y https://dlm.mariadb.com/4296983/MaxScale/24.02.6/packages/rocky/9/x86_64/maxscale-24.02.6-1.rhel.9.x86_64.rpm
   ```

1. Create the maxscale_monitor user on your backend MariaDB servers with appropriate privileges

   ```sql
   CREATE USER 'maxscale_monitor'@'%' IDENTIFIED BY 'monitor_password';
   GRANT
     BINLOG ADMIN, BINLOG MONITOR,
     CONNECTION ADMIN, READ_ONLY ADMIN,
     REPLICATION SLAVE ADMIN, SLAVE MONITOR,
     RELOAD, PROCESS, SUPER, EVENT, SET USER,
     SHOW DATABASES
     ON *.*
     TO `maxscale_monitor`@`%`;
   GRANT SELECT ON mysql.global_priv TO 'maxscale_monitor'@'%';
   GRANT SELECT ON mysql.global_priv TO 'maxscale_monitor'@'%';
   ```

1. Create the maxscale_user on your backend MariaDB servers with the following privileges

   ```sql
   CREATE USER 'maxscale_user'@'%' IDENTIFIED BY 'maxscale_password';
   GRANT SELECT ON mysql.user TO 'maxscale_user'@'%';
   GRANT SELECT ON mysql.db TO 'maxscale_user'@'%';
   GRANT SELECT ON mysql.tables_priv TO 'maxscale_user'@'%';
   GRANT SELECT ON mysql.columns_priv TO 'maxscale_user'@'%';
   GRANT SELECT ON mysql.procs_priv TO 'maxscale_user'@'%';
   GRANT SELECT ON mysql.proxies_priv TO 'maxscale_user'@'%';
   GRANT SELECT ON mysql.roles_mapping TO 'maxscale_user'@'%';
   GRANT SHOW DATABASES ON *.* TO 'maxscale_user'@'%';
   ```

1. Configure maxscale in `/etc/maxscale.cnf`

   ```ini
   [maxscale]
   threads=auto

   [server1]
   type=server
   address=192.168.7.101
   port=3306

   [server2]
   type=server
   address=192.168.7.102
   port=3306

   [MariaDB-Cluster]
   type=monitor
   module=mariadbmon
   servers=server1,server2
   user=maxscale_monitor
   password=monitor_password
   monitor_interval=5s
   auto_failover=true
   auto_rejoin=true
   replication_user=replication_user
   replication_password=bigs3cret

   [Read-Write-Service]
   type=service
   router=readwritesplit
   cluster=MariaDB-Cluster
   user=maxscale_user
   password=maxscale_password

   [Read-Write-Listener]
   type=listener
   service=Read-Write-Service
   port=3307
   ```

1. Restart Mariadb

   ```bash
   sudo systemctl restart mariadb
   ```

1. list servers and monitors and ensure the Maxscale is working correctly:

   ```bash
   maxctrl list servers
   maxctrl list monitors
   ```

> [!NOTE]
> Switchover is for cases when you explicitly want to move the primary role from one server to another

```bash
maxctrl call command mariadbmon switchover $MONITOR_NAME $SLAVE_NAME $MASTER_NAME
maxctrl call command mariadbmon switchover MariaDB-Cluster server1 server2
```

> [!NOTE]
> If auto_failover is **not enabled**, the failover mechanism must be invoked manually:

```bash
maxctrl call command mariadbmon failover $MONITOR_NAME
maxctrl call command mariadbmon failover MariaDB-Cluster
```

# Install Zabbix

1. Install Zabbix repository

   Disable Zabbix packages provided by EPEL, if you have it installed. Edit file **/etc/yum.repos.d/epel.repo** and add the following statement.

   ```ini
   [epel]
   ...
   excludepkgs=zabbix*
   ```

   Proceed with installing zabbix repository.

   ```bash
   rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/rocky/9/noarch/zabbix-release-latest-7.4.el9.noarch.rpm
   dnf clean all
   ```

1. Install Zabbix server, frontend, agent2

   ```bash
   dnf install -y zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2
   ```

1. Install Zabbix agent 2 plugins

   ```bash
   dnf install -y zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
   ```

1. Create initial database

   Run the following on your database host (Master)

   ```sql
   CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
   CREATE USER zabbix@'%' IDENTIFIED BY 'password';
   GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@'%';
   SET GLOBAL log_bin_trust_function_creators = 1;
   ```

   On Zabbix server host import initial schema and data. You will be prompted to enter your newly created password.

   ```bash
   zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mariadb --default-character-set=utf8mb4 -uzabbix -p zabbix
   ```

   Disable log_bin_trust_function_creators option after importing database schema

   ```sql
   SET GLOBAL log_bin_trust_function_creators = 0;
   ```

1. Configure the database and HA for Zabbix server

   Edit file **/etc/zabbix/zabbix_server.conf**

   ```
   LogFileSize=256
   DBHost=zabbix-2.example.com   # Maxscale host
   DBPort=3307                   # Maxscale Port
   DBName=zabbix
   DBUser=zabbix
   DBPassword=password
   ```

1. Start Zabbix server and agent processes

   Start Zabbix server and agent processes and make it start at system boot

   ```bash
   systemctl restart zabbix-server zabbix-agent2 nginx php-fpm
   systemctl enable zabbix-server zabbix-agent2 nginx php-fpm
   ```

1. Open Zabbix UI web page

   The URL for Zabbix UI when using Nginx depends on the configuration changes you should have made.

   Enter the user name `Admin` with password `zabbix` to log in as a Zabbix superuser. Access to all menu sections will be granted.

   For security reasons, it is strongly recommended to change the default password for the Admin account immediately after the first login.

## Enabling high availability

1. Start Zabbix server as cluster node

   Two parameters are required in the server configuration to start a Zabbix server as cluster node:

   - `HANodeName` parameter must be specified for each Zabbix server that will be an HA cluster node.

     This is a unique node identifier (e.g. zabbix-node-01) that the server will be referred to in agent and proxy configurations. If you do not specify HANodeName, then the server will be started in standalone mode.

   - `NodeAddress` parameter must be specified for each node.

     The NodeAddress parameter (address:port) will be used by Zabbix frontend to connect to the active server node. NodeAddress must match the IP or FQDN name of the respective Zabbix server.

1. Restart all Zabbix servers

   Restart all Zabbix servers after making changes to the configuration files

   ```bash
   systemctl restart zabbix-server
   systemctl enable zabbix-server
   ```

   The new status of the servers can be seen in `Reports → System information` and by running:

   ```bash
   zabbix_server -R ha_status
   ```

1. Preparing frontend

   Make sure that `Zabbix server address:port` is **not defined** in the frontend configuration (found in **/etc/zabbix/web/zabbix.conf.php**)

1. Agent configuration

   To enable passive checks, the node names must be listed in the Server parameter, separated by a comma.

   ```
   Server=zabbix-node-01,zabbix-node-02
   ```

   To enable active checks, the node names must be listed in the ServerActive parameter. Note that for active checks the nodes must be separated by a comma from any other servers, while the nodes themselves must be separated by a semicolon, e.g.:

   ```
   ServerActive=zabbix-node-01;zabbix-node-02
   ```

# Install Keepalived

# Install Zabbix Proxy
