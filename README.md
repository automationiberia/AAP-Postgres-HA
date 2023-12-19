# AAP-Postgres-HA

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
> touch ~/.ssh/authorized_keys && chown ${USER}: ~/.ssh/authorized_keys && chmod 0600 ~/.ssh/authorized_keys
> ```

Let's configure the database to be able to run the `repmgr` plugin. Run the following commands at the database server installed by the AAP installer (this will be the initial `primary` database replica):

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
```

## 4. Deploy `HAProxy` nodes

At each `HAProxy` server, the configuration file `/etc/haproxy/haproxy.cfg` is fullfilled with the following contents:

```cfg file
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
  bind *:5433
  mode tcp
  default_backend pg_cluster

backend pg_cluster
  mode tcp
  #stick-table type integer size 1 expire 1d
  stick-table type integer size 1 expire 2m
  stick on int(1)
  option pgsql-check user repmgr
  server aap-ha-db-1 192.168.122.88:5432 check port 5432 inter 2s downinter 5s rise 2 fall 3 on-marked-down shutdown-sessions
  server aap-ha-db-2 192.168.122.47:5432 check port 5432 inter 2s downinter 5s rise 2 fall 3 backup on-marked-down shutdown-sessions
  # Add more replica servers as needed
```

After modifying the configuration file, the service must be restarted with the command `systemctl restart haproxy`.

> [!TIP]
> If Selinux is enabled, it should be configured to allow `HAProxy` to bind to any needed port
>
> ```console
> setsebool -P haproxy_connect_any=1
> ```

### 4.1 `HAProxy` with `Keepalived`

> [!TIP]:
> To let `Keepalived` to use Direct Routing, the following command should be executed to every `HAProxy` server to configure the firewall accordingly:
>
> ```console
> firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
> ```

`Keepalived` at primary `HAProxy` server:

```cfg file
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

virtual_server 192.168.122.100 5433 {
    delay_loop 10
    lb_algo rr
    lb_kind DR
    persistence_timeout 9600
    protocol TCP

    real_server 192.168.122.48 5433 {
        weight 1
        TCP_CHECK {
          connect_timeout 10
          connect_port    5433
        }
    }
    real_server 192.168.122.139 5433 {
        weight 1
        TCP_CHECK {
          connect_timeout 10
          connect_port    5433
        }
    }
}
```

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

virtual_server 192.168.122.100 5433 {
    delay_loop 10
    lb_algo rr
    lb_kind DR
    persistence_timeout 9600
    protocol TCP

    real_server 192.168.122.48 5433 {
        weight 1
        TCP_CHECK {
          connect_timeout 10
          connect_port    5433
        }
    }
    real_server 192.168.122.139 5433 {
        weight 1
        TCP_CHECK {
          connect_timeout 10
          connect_port    5433
        }
    }
}
```

> [!TIP]
> If Selinux is enabled, it should be configured to allow `Keepalived` to bind to any needed port
>
> ```console
> setsebool -P keepalived_connect_any=1
> ```

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
       'PORT': '5433',
       'OPTIONS': { 'sslmode': 'prefer',
                    'sslrootcert': '/etc/pki/tls/certs/ca-bundle.crt',
       },
   }
}
```

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
DATABASES = {'default': {'HOST': '192.168.122.48', 'ENGINE': 'django.db.backends.postgresql_psycopg2', 'NAME': 'automationhub', 'USER': 'automationhub', 'PASSWORD': 'redhat00', 'PORT': 5433, 'OPTIONS': {'sslmode': 'prefer', 'sslrootcert': '/etc/pki/tls/certs/ca-bundle.crt'}}}
DB_ENCRYPTION_KEY = '/etc/pulp/certs/database_fields.symmetric.key'
DEPLOY_ROOT = '/var/lib/pulp'
```

The line to be modified is the `DATABASES` one, and the fields to be customized are `HOST`, `USER`, `PASSWORD` and `PORT`.

Once the configuration has been adapted, the service can be started again:

```console
systemctl start automation-hub pulpcore pulpcore-content pulpcore-api nginx redis
```

## 6. Test failover and recovery

With this configurations already done, all the services have a redundant counterpart that will continue serving requests when the main ones are failing.
