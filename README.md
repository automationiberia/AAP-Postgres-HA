# AAP-Postgres-HA

> [!CAUTION]
> Take a look at <https://access.redhat.com/articles/4010491> and <https://access.redhat.com/solutions/3682951> to learn about the supportability of this solution.

## 1. Install AAP normally

Install AAP following any of the official installation paths. They are all available at the section `Installation and Upgrade` in <https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4>

## 2. Deploy extra DB nodes

The easiest way to deploy these extra DB nodes, if VMware or equivalent is used, is to clone the DB node deployed by the AAP installer and change it's hostname and IP address. Any other way requires to install the same version of the O.S. and PostgreSQL as it is installed in the DB node deployed by the AAP installer.

If cloning is not the selected method, the following considerations must be taken:

* AAP 2.4 requires all the implied nodes to be RHEL 8.5+.
* The default version for PostgreSQL in RHEL 8 is 10.23, but 12+ is required by AAP. The recommendation is to install the same version that is deployed by the AAP installer, so the following steps must be executed before installing PostgreSQL:

  ```console
  dnf module switch-to postgresql:13
  dnf install postgresql-server-13.13 postgresql-contrib-13.13
  ```

Once the initial installation is completed, the firewall must be configured to let the DB be reachable from the other nodes:

```console
firewall-cmd --add-service postgresql --permanent
firewall-cmd --add-service postgresql
```

## 3. Configure DB replicas with `repmgr`

The first step is to download the latest version of `repmgr`, and manually install the RPMs, avoiding its dependencies, as these RPMs wants to install it's own version of PostgreSQL:

```console
URLS=(
    https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/repmgr_13-5.4.1-1PGDG.rhel8.x86_64.rpm 
    https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/repmgr_13-devel-5.4.1-1PGDG.rhel8.x86_64.rpm
    https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/repmgr_13-llvmjit-5.4.1-1PGDG.rhel8.x86_64.rpm
)

type wget || dnf install -y wget

for url in ${URLS[@]}; do
    wget "${url}"
done

rpm -ivh --nodeps "repmgr_13-*"
```

If these RPM packages are installed by the normal way, a custom version of the PostgreSQL packages is already installed. This is not the desired scenario, as the AAP installation is using the PostgreSQL packages provided by Red Hat instead. Because of this differences in the PostgreSQL installation process, the following steps must be done to let the PostgreSQL server provided by Red Hat to work properly along with the `repmgr` packages that have been recently installed:

```console
cd /usr/bin/
ln -sf /usr/pgsql-13/bin/repmgr .
ln -sf /usr/pgsql-13/bin/repmgrd .

cd /usr/lib64/pgsql
ln -sf /usr/pgsql-13/lib/repmgr.so .
ln -sf /usr/pgsql-13/lib/bitcode/ .

cd /usr/share/pgsql/extension
find /usr/pgsql-13/share/extension/ -type f | while read file; do ln -sf ${file} .; done
```

The user `postgres` must have the SSH configured to connect to every database node without the need to enter a password, so the switchover operations can run from any node.

> [!NOTE]
> To be executed at every database node

```console
su - postgres
ssh-keygen -b 2048 -t rsa -q -N "" -f .ssh/id_rsa
```

The easiest way to copy the keys is to execute the following steps between all the database servers with each other:

1. In server A:
   1. `cat ~/.ssh/id_rsa.pub`
   2. Copy the contents of the previous command
2. In server B: Paste the contents copied from the previous command, to the end of the `~/.ssh/authorized_keys` file.

> [!NOTE]
> If any of the above paths needs to be manually created, it's important to remember that it's ownership and access permissions are critical for the SSH keys to work properly, so ensure that the following commands are executed afterwards:
>
> ```console
> mkdir ~/.ssh && chown ${USER}: ~/.ssh && chmod 0700 ~/.ssh
>
> touch ~/.ssh/authorized_keys && chown ${USER}: ~/.ssh/authorized_keys && chmod 0600 ~/.ssh/authorized_keys
> ```

Let's configure the database to be able to run the `repmgr` plugin. **Run the following commands at the database server installed by the AAP installer** (this will be the initial `primary` database replica):

```console
su - postgres

sed -i "s,^#include_dir = 'conf.d',include_dir = 'conf.d'," data/postgresql.conf

mkdir data/conf.d

cat > data/conf.d/repmgr.conf <<EOF
shared_preload_libraries = 'repmgr'
max_wal_senders = 10
max_replication_slots = 10
wal_level = 'logical'
hot_standby = on
archive_mode = on
archive_command = '/bin/true'
EOF

logout
```

Now, let's configure the `repmgr` tool at all of the database servers that will be replicas for the database installed by the AAP installer. **Run the following commands at all of the database servers, as `root` user**:

```console
cp /etc/repmgr/13/repmgr.conf{,.orig}

cat > /etc/repmgr/13/repmgr.conf <<EOF
node_id=1                                                                                   # A unique integer greater than zero
node_name='aap-ha-db-1'                                                                     # An arbitrary (but unique) string; we recommend
conninfo='postgresql://repmgr:repmgr@aap-ha-db-1/repmgr'                                    # Database connection information as a conninfo string.
data_directory='/var/lib/pgsql/data'                                                        # The node's data directory. This is needed by repmgr
log_file='/var/log/repmgr.log'                                                              # STDERR can be redirected to an arbitrary file
pg_bindir='/usr/bin/'                                                                       # Path to PostgreSQL binary directory (location
ssh_options='-l postgres -q -o ConnectTimeout=10'                                           # Options to append to "ssh"
failover='automatic'                                                                        # one of 'automatic', 'manual'.
connection_check_type=connection                                                            # How to check availability of the upstream node; valid options:
reconnect_attempts=6                                                                        # Number of attempts which will be made to reconnect to an unreachable
reconnect_interval=10                                                                       # Interval between attempts to reconnect to an unreachable
promote_command='repmgr standby promote -f /etc/repmgr/13/repmgr.conf'                      # command repmgrd executes when promoting a new primary; use something like:
follow_command='repmgr standby follow -f /etc/repmgr/13/repmgr.conf -upstream-node-id=%n'   # command repmgrd executes when instructing a standby to follow a new primary;
primary_notification_timeout=10                                                             # Interval (in seconds) which repmgrd on a standby
service_start_command = 'sudo systemctl start postgresql'
service_stop_command = 'sudo systemctl stop postgresql'
service_restart_command = 'sudo systemctl restart postgresql'
service_reload_command = 'sudo systemctl reload postgresql'
repmgrd_service_start_command = 'sudo systemctl start repmgr-13'
repmgrd_service_stop_command = 'sudo systemctl stop repmgr-13'
EOF

touch /var/log/repmgr.log
chown postgres: /var/log/repmgr.log
```

Create the database superuser `repmgr` and the database `repmgr`:

```console
su - postgres

psql <<EOF
CREATE USER repmgr WITH PASSWORD 'repmgr';
CREATE DATABASE repmgr WITH OWNER repmgr;
ALTER USER repmgr SUPERUSER;
ALTER USER repmgr REPLICATION;
ALTER USER repmgr BYPASSRLS;
EOF

logout
```

Let the newly created user to connect from any database server. **Run the following commands at all the database servers servers as `root` user**:

```console
su - postgres

cat >> data/pg_hba.conf <<EOF
local   replication      repmgr                             trust
host    replication      repmgr        127.0.0.1/32         trust
host    replication      repmgr        192.168.122.47/32    trust
host    replication      repmgr        192.168.122.88/32    trust
local   repmgr           repmgr                             trust
host    repmgr           repmgr        127.0.0.1/32         trust
host    repmgr           repmgr        192.168.122.47/32    trust
host    repmgr           repmgr        192.168.122.88/32    trust
EOF

logout

systemctl restart postgresql
```

Enable and start the `repmgr-13` service to control the replicas. **Run the following commands at all of the database servers, as `root` user**:

```console
systemctl enable --now repmgr-13
```

> [!tip]
> Replace the IP addresses in the previous command for the IPs of the database servers. Add more lines if more replicas are to be configured.

The `postgres` user must have permissions to run some commands using sudo and without any password, for the switchover operations to work as expected. **Run the following commands at all of the database servers**:

```console
cat > /etc/sudoers.d/95-postgres <<EOF
postgres ALL=(root) NOPASSWD: /usr/bin/systemctl start postgresql, \
                              /usr/bin/systemctl stop postgresql, \
                              /usr/bin/systemctl restart postgresql, \
                              /usr/bin/systemctl reload postgresql, \
                              /usr/bin/systemctl start repmgr-13, \
                              /usr/bin/systemctl stop repmgr-13
EOF

chmod 0440 /etc/sudoers.d/95-postgres

visudo -c
```

Register the primary server. **Run the following commands at the primary database server**:

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf primary register
INFO: connecting to primary database...
NOTICE: attempting to install extension "repmgr"
NOTICE: "repmgr" extension successfully installed
NOTICE: primary node record (ID: 1) registered

$ logout
```

It should be registered successfully. It can be checked with the following command:

```console
$ repmgr -f /etc/repmgr/13/repmgr.conf cluster show
 ID | Name        | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string                            
----+-------------+---------+-----------+----------+----------+----------+----------+-----------------------------------------------
 1  | aap-ha-db-1 | primary | * running |          | default  | 100      | 1        | postgresql://repmgr:repmgr@aap-ha-db-1/repmgr
```

Clone the primary database to the secondary node (repeat these steps if more than one replica is to be added). **Run the following steps at the secondary database servers**:

```console
$ su - postgres

$ rm -rf data/*

$ repmgr -f /etc/repmgr/13/repmgr.conf -d 'postgresql://repmgr:repmgr@aap-ha-db-1.bcnconsulting.com/repmgr' standby clone
NOTICE: destination directory "/var/lib/pgsql/data" provided
INFO: connecting to source node
DETAIL: connection string is: user=repmgr password=repmgr dbname=repmgr host=aap-ha-db-1.bcnconsulting.com
DETAIL: current installation size is 69 MB
INFO: replication slot usage not requested;  no replication slot will be set up for this standby
NOTICE: checking for available walsenders on the source node (2 required)
NOTICE: checking replication connections can be made to the source server (2 required)
WARNING: data checksums are not enabled and "wal_log_hints" is "off"
DETAIL: pg_rewind requires "wal_log_hints" to be enabled
INFO: checking and correcting permissions on existing directory "/var/lib/pgsql/data"
NOTICE: starting backup (using pg_basebackup)...
HINT: this may take some time; consider using the -c/--fast-checkpoint option
INFO: executing:
  /usr/bin/pg_basebackup -l "repmgr base backup"  -D /var/lib/pgsql/data -d 'user=repmgr password=repmgr dbname=repmgr host=aap-ha-db-1.bcnconsulting.com' -X stream 
NOTICE: standby clone (using pg_basebackup) complete
NOTICE: you can now start your PostgreSQL server
HINT: for example: sudo systemctl start postgresql
HINT: after starting the server, you need to register this standby with "repmgr standby register"


$ sudo systemctl start postgresql

$ logout
```

Register the secondary database to `repmgr`. **Run the following commands at the secondary database servers**:

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf standby register
INFO: connecting to local node "aap-ha-db-2" (ID: 2)
INFO: connecting to primary database
WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID: 1)
INFO: standby registration complete
NOTICE: standby node "aap-ha-db-2" (ID: 2) successfully registered

$ logout
```

It should be registered successfully. It can be checked with the following command:

```console
$ repmgr -f /etc/repmgr/13/repmgr.conf cluster show
 ID | Name        | Role    | Status    | Upstream    | Location | Priority | Timeline | Connection string                            
----+-------------+---------+-----------+-------------+----------+----------+----------+-----------------------------------------------
 1  | aap-ha-db-1 | primary | * running |             | default  | 100      | 1        | postgresql://repmgr:repmgr@aap-ha-db-1/repmgr
 2  | aap-ha-db-2 | standby |   running | aap-ha-db-1 | default  | 100      | 1        | postgresql://repmgr:repmgr@aap-ha-db-2/repmgr
```

Let's test the failover functionality. Stop the primary server (run the following commands at the primary database server):

```console
$ su - postgres

$ sudo systemctl stop postgresql

$ repmgr -f /etc/repmgr/13/repmgr.conf cluster show
 ID | Name        | Role    | Status        | Upstream      | Location | Priority | Timeline | Connection string                            
----+-------------+---------+---------------+---------------+----------+----------+----------+-----------------------------------------------
 1  | aap-ha-db-1 | primary | ? unreachable | ?             | default  | 100      |          | postgresql://repmgr:repmgr@aap-ha-db-1/repmgr
 2  | aap-ha-db-2 | standby |   running     | ? aap-ha-db-1 | default  | 100      | 1        | postgresql://repmgr:repmgr@aap-ha-db-2/repmgr

WARNING: following issues were detected
  - unable to connect to node "aap-ha-db-1" (ID: 1)
  - node "aap-ha-db-1" (ID: 1) is registered as an active primary but is unreachable
  - unable to connect to node "aap-ha-db-2" (ID: 2)'s upstream node "aap-ha-db-1" (ID: 1)
  - unable to determine if node "aap-ha-db-2" (ID: 2) is attached to its upstream node "aap-ha-db-1" (ID: 1)

HINT: execute with --verbose option to see connection error messages

$ logout
```

And look at the cluster status after 50s:

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf cluster show
 ID | Name        | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string                            
----+-------------+---------+-----------+----------+----------+----------+----------+-----------------------------------------------
 1  | aap-ha-db-1 | primary | - failed  | ?        | default  | 100      |          | postgresql://repmgr:repmgr@aap-ha-db-1/repmgr
 2  | aap-ha-db-2 | primary | * running |          | default  | 100      | 2        | postgresql://repmgr:repmgr@aap-ha-db-2/repmgr

WARNING: following issues were detected
  - unable to connect to node "aap-ha-db-1" (ID: 1)

HINT: execute with --verbose option to see connection error messages

$ logout
```

Now, let's the primary server to be the primary server again (run the following commands at the primary database server):

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf -d 'postgresql://repmgr:repmgr@aap-ha-db-2.bcnconsulting.com/repmgr' node rejoin
NOTICE: rejoin target is node "aap-ha-db-2" (ID: 2)
INFO: local node 1 can attach to rejoin target node 2
DETAIL: local node's recovery point: 0/D000028; rejoin target node's fork point: 0/D0000A0
NOTICE: setting node 1's upstream to node 2
WARNING: unable to ping "postgresql://repmgr:repmgr@aap-ha-db-1/repmgr"
DETAIL: PQping() returned "PQPING_NO_RESPONSE"
NOTICE: starting server using "sudo systemctl start postgresql"
NOTICE: NODE REJOIN successful
DETAIL: node 1 is now attached to node 2

$ logout
```

And check that the primary node is now an standby one (run the following commands at the primary database server):

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf cluster show
 ID | Name        | Role    | Status    | Upstream    | Location | Priority | Timeline | Connection string                            
----+-------------+---------+-----------+-------------+----------+----------+----------+-----------------------------------------------
 1  | aap-ha-db-1 | standby |   running | aap-ha-db-2 | default  | 100      | 1        | postgresql://repmgr:repmgr@aap-ha-db-1/repmgr
 2  | aap-ha-db-2 | primary | * running |             | default  | 100      | 2        | postgresql://repmgr:repmgr@aap-ha-db-2/repmgr

$ logout
```

So, let's the primary node, to be primary again (run the following commands at the primary database server):

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf standby switchover
NOTICE: executing switchover on node "aap-ha-db-1" (ID: 1)
NOTICE: attempting to pause repmgrd on 2 nodes
NOTICE: local node "aap-ha-db-1" (ID: 1) will be promoted to primary; current primary "aap-ha-db-2" (ID: 2) will be demoted to standby
NOTICE: stopping current primary node "aap-ha-db-2" (ID: 2)
NOTICE: issuing CHECKPOINT on node "aap-ha-db-2" (ID: 2) 
DETAIL: executing server command "sudo systemctl stop postgresql"
INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
NOTICE: current primary has been cleanly shut down at location 0/E000028
NOTICE: promoting standby to primary
DETAIL: promoting server "aap-ha-db-1" (ID: 1) using pg_promote()
NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
NOTICE: STANDBY PROMOTE successful
DETAIL: server "aap-ha-db-1" (ID: 1) was successfully promoted to primary
NOTICE: node "aap-ha-db-1" (ID: 1) promoted to primary, node "aap-ha-db-2" (ID: 2) demoted to standby
NOTICE: switchover was successful
DETAIL: node "aap-ha-db-1" is now primary and node "aap-ha-db-2" is attached as standby
NOTICE: STANDBY SWITCHOVER has completed successfully

$ logout
```

And check again that the primary node is the primary again (run the following commands at the primary database server):

```console
$ su - postgres

$ repmgr -f /etc/repmgr/13/repmgr.conf cluster show
 ID | Name        | Role    | Status    | Upstream    | Location | Priority | Timeline | Connection string                            
----+-------------+---------+-----------+-------------+----------+----------+----------+-----------------------------------------------
 1  | aap-ha-db-1 | primary | * running |             | default  | 100      | 3        | postgresql://repmgr:repmgr@aap-ha-db-1/repmgr
 2  | aap-ha-db-2 | standby |   running | aap-ha-db-1 | default  | 100      | 2        | postgresql://repmgr:repmgr@aap-ha-db-2/repmgr

$ logout
```

## 4. Deploy `HAProxy` nodes

Install the `haproxy` package at every HAProxy server. Run the following commands as `root` user:

```console
dnf install -y haproxy
```

At each `HAProxy` server, the configuration file `/etc/haproxy/haproxy.cfg` is fullfilled with the following contents:

```console
cp /etc/haproxy/haproxy.cfg{,.orig}
cat > /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option log-health-checks
    option tcp-check
    timeout connect 10s
    timeout client 30s
    timeout server 30s

frontend pg_frontend
    bind *:5432
    mode tcp
    default_backend pg_cluster

backend pg_cluster
    mode tcp
    stick-table type integer size 1 expire 1d
    stick on int(1)
    option pgsql-check user repmgr
    server aap-ha-db-1 192.168.122.88:5432 check port 5432 inter 2s downinter 5s rise 2 fall 3 on-marked-down shutdown-sessions 
    server aap-ha-db-2 192.168.122.47:5432 check port 5432 inter 2s downinter 5s rise 2 fall 3 backup on-marked-down shutdown-sessions
    # Add more replica servers as needed

listen stats
    bind            *:1936
    mode            http
    log             global

    maxconn 10

    timeout client  100s
    timeout server  100s
    timeout connect 100s
    timeout queue   100s

    stats enable
    stats hide-version
    stats refresh 30s
    stats show-node
    stats auth admin:password
    stats uri  /haproxy?stats
EOF
```

> [!tip]
> Replace the IP addresses in the previous command for the IPs of the database servers. Add more lines if more replicas are to be configured.

After modifying the configuration file, the service must be restarted and the firewall configured to allow the incomming connections:

```console
systemctl restart haproxy
firewall-cmd --add-port 1936/tcp --add-port 5432/tcp --permanent
firewall-cmd --reload
```

> [!TIP]
> If Selinux is enabled, it should be configured to allow `HAProxy` to bind to any needed port
>
> ```console
> setsebool -P haproxy_connect_any=1
> ```

The HAProxy service can be checked by accessing to its stats webpage at <http://aap-ha-hk-1.bcnconsulting.com:1936/haproxy?stats>. Login information will be required (see the configuration file).

### 4.1 `HAProxy` with `Keepalived`

Install the `keepalived` package at every HAProxy server. Run the following commands as `root` user:

```console
dnf install -y keepalived
```

> [!TIP]:
> To let `Keepalived` to use Direct Routing, the following command should be executed to every `HAProxy` server to configure the firewall accordingly:
>
> ```console
> firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
> firewall-cmd --reload
> ```

Configure `Keepalived` at primary `HAProxy` server:

```console
cp /etc/keepalived/keepalived.conf{,.orig}
cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
}

vrrp_instance RH_1 {
    state MASTER
    interface enp1s0
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass passw123
    }
    virtual_ipaddress {
        192.168.122.100
    }
}
EOF
```

> [!tip]
> Replace the IP addresses in the previous command for the IPs of the database servers. Add more lines if more replicas are to be configured.

`Keepalived` at standby `HAProxy` server:

```cfg file
global_defs {
}

vrrp_instance RH_1 {
    state BACKUP
    interface enp1s0
    virtual_router_id 50
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass passw123
    }
    virtual_ipaddress {
        192.168.122.100
    }
}
```

> [!TIP]
> Replace the IP addresses in the previous command for the IPs of the database servers. Add more lines if more replicas are to be configured.
>
> Replace the network interface name if needed as well.
>
> If Selinux is enabled, it should be configured to allow `Keepalived` to bind to any needed port
>
> ```console
> setsebool -P keepalived_connect_any=1
> ```

Enable and start the `keepalived` service:

```console
systemctl enable --now keepalived
```

In order to allow the haproxy to bind to the VIP virtual address when it is not up (aka backup Keepalived nodes), the following configuration for `sysctl` is required. Run the following commands at the Keepalived nodes:

```console
cat > /etc/sysctl.d/90-keepalived.conf <<EOF
net.ipv4.ip_nonlocal_bind=1
# solution to ARP flux problem on both interfaces
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.eth0.arp_announce = 2
EOF

sysctl -p /etc/sysctl.d/90-keepalived.conf

systemctl restart haproxy
```

## 5. Configure AAP to use the `Keepalived` endpoint

All the components of the Ansible Automation Platform must be configured to use the `Keepalived` endpoint, which will help using one of the two `HAProxies` when the other one is not available. Each `HAProxies` ensures to send the SQL statements to the correct PostgreSQL database instance.

### 5.1. Controller configuration

The first step to change the database connection configuration is to stop the Controller daemons, so the current connections to the database are all closed. This is done by running the following command:

```console
systemctl stop automation-controller.service
```

The configuration file for the Ansible Automation Controller, is located at `/etc/tower/conf.d/postgres.py`, and the contents to be accomodated are the following ones:

```python
# Ansible Automation Platform controller database settings.

DATABASES = {
   'default': {
       'ATOMIC_REQUESTS': True,
       'ENGINE': 'awx.main.db.profiled_pg',
       'NAME': 'awx',
       'USER': 'awx',
       'PASSWORD': """redhat00""",
       'HOST': '192.168.122.100',
       'PORT': '5432',
       'OPTIONS': { 'sslmode': 'prefer',
                    'sslrootcert': '/etc/pki/tls/certs/ca-bundle.crt',
       },
   }
}
```

> [!tip]
> Replace the IP addresses in the previous command for the IPs of the database servers. Add more lines if more replicas are to be configured.

The lines to be customized are the following ones:

* USER
* PASSWORD
* HOST
* PORT

Once the configuration has been adapted, the service can be started again:

```console
systemctl start automation-controller.service
```

### 5.2. Private Automation Hub configuration

The first step to change the database connection configuration is to stop the Private Automation Hub daemons, so the current connections to the database are all closed. This is done by running the following command:

```console
systemctl stop automation-hub pulpcore pulpcore-content pulpcore-api nginx redis
```

> [!CAUTION]
> The systemctl unit file for the PAH management `automation-hub.service` is not stopping/starting all the implied services as it should. Is that the reason why the previous command shows all the implied components to be stopped (and later, to be started again).

The configuration file for the Ansible Private Automation Hub, is located at `/etc/tower/conf.d/postgres.py`, and the contents to be accomodated are the following ones:

```cfg file
# Do NOT edit this file. It may be subject of change when PULP is updated.
# Put new or overridden settings in /etc/pulp/settings.local.py instead.
DATABASES = {'default': {'HOST': '192.168.122.48', 'ENGINE': 'django.db.backends.postgresql_psycopg2', 'NAME': 'automationhub', 'USER': 'automationhub', 'PASSWORD': 'redhat00', 'PORT': 5432, 'OPTIONS': {'sslmode': 'prefer', 'sslrootcert': '/etc/pki/tls/certs/ca-bundle.crt'}}}
DB_ENCRYPTION_KEY = '/etc/pulp/certs/database_fields.symmetric.key'
DEPLOY_ROOT = '/var/lib/pulp'
```

> [!tip]
> Replace the IP addresses in the previous command for the IPs of the database servers. Add more lines if more replicas are to be configured.

The line to be modified is the `DATABASES` one, and the fields to be customized are `HOST`, `USER`, `PASSWORD` and `PORT`.

Once the configuration has been adapted, the service can be started again:

```console
systemctl start automation-hub pulpcore pulpcore-content pulpcore-api nginx redis
```

## 6. Test failover and recovery

With this configurations already done, all the services have a redundant counterpart that will continue serving requests when the main ones are failing.
