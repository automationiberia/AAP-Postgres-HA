# AAP-Postgres-HA

## 1. Install AAP normally

Install AAP following any of the official installation paths. They are all available at the section `Installation and Upgrade` in https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4

## 2. Deploy extra DB nodes

The easiest way to deploy these extra DB nodes, if VMware or equivalent is used, is to clone the DB node deployed by the AAP installer and change it's hostname and IP address. Any other way requires to install the same version of the O.S. and PostgreSQL as it is installed in the DB node deployed by the AAP installer.

If cloning is not the selected method, the following considerations must be taken:

* AAP 2.4 requires all the implied nodes to be RHEL 8.5+.
* The default version for PostgreSQL in RHEL 8 is 10.23, but 12+ is required by AAP. The recommendation is to install the same version that is deployed byt the AAP installer, so the following steps must be executed before installing PostgreSQL:
  ```console
  dnf module switch-to postgresql:13
  dnf install postgresql-server-13.11 postgresql-contrib-13.11
  ```

Once the initial installation is completed, the firewall must be configured to let the DB be reachable from the other nodes:

```console
$ firewall-cmd --add-service postgres --permanent
$ firewall-cmd --add-service postgres
```

## 3. Configure DB replicas with `repmgr`

The first step is to download the latest version of `repmgr`, and manually install the RPMs, avoiding its dependencies, as these RPMs wants to install it's own version of PostgreSQL:

```console
$ URLS=(
    https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/repmgr_13-5.4.1-1PGDG.rhel8.x86_64.rpm 
    https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/repmgr_13-devel-5.4.1-1PGDG.rhel8.x86_64.rpm
    https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-8-x86_64/repmgr_13-llvmjit-5.4.1-1PGDG.rhel8.x86_64.rpm
)
$ type wget || dnf install -y wget
$ for url in ${URLS[@]}; do
    wget "${url}"
done
$ rpm -ivh --nodeps "repmgr_13-*"
```

The user `postgres` must have the SSH configured to connect to every database node without the need to enter a password, so the switchover operations can run from any node.

> [!NOTE]
> To be executed at every database node

```console
su - postgres
ssh-keygen -b 2048 -t rsa -q -N ""
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

## 4. Deploy HAProxy nodes

At each HAProxy server, the configuration file `/etc/haproxy/haproxy.cfg` is fullfilled with the following contents:

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
> If Selinux is enabled, it should be configured to allow HAProxy to bind to any needed port
>
> ```console
> setsebool -P haproxy_connect_any=1
> ```

## 5. Configure AAP to use the HAProxy endpoint
## 6. Test failover and recovery