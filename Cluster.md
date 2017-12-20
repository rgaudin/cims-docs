# RapidPro Configuration for CIMS Cluster

CIMS Cluster is a two-server Cluster in a master/slave configuration.

Both servers are using [SUSE High Availability Extension](https://www.suse.com/products/highavailability) which manages a virtual IP address for the cluster (resource named `admin_addr`).

This document describes the required configuration of the two servers which [rapidpro_controller](https://github.com/rgaudin/cims-docs/blob/master/RapidProController.md) builds upon to handle the automatic fail-over tilt.

## System

`/etc/hosts`

``` sh
10.172.20.200	rapidpro
10.172.20.201	sms1
10.172.20.202	sms2
172.20.1.130    smsgateway
```

* `/etc/hostname`: `sms1`
* `/etc/rapidpro.role`: `master`
* `/etc/rapidpro.status`: `disabled`

## SSH

* sms1's key on sms2
* sms2's key on sms1
* initial connection from sms1 to sm2
* initial connection from sms2 to sms1

## RapidPro Controller

``` sh
pip install -U pip
pip install rapidpro_controller
cp /usr/lib/python2.7/site-packages/rapidpro_controller/rapidproctl.conf.sample /etc/rapidproctl.conf
```

* Edit `/etc/rapidproctl.conf`

``` json
{
	"LOG_FILE": "/var/log/rapidpro/rapidproctl.log",
	"LOG_LEVEL": 10,
	"SSH_USER": "root",
	"SERVERA": "sms1",
	"SERVERB": "sms2",
	"ROLE_PATH": "/etc/rapidpro.role",
	"STATUS_PATH": "/etc/rapidpro.status",
	"CLUSTERIP_PATH": "/etc/rapidpro.clusterip",
	"SERVICE_COMMAND": "/bin/systemctl",
	"REDIS_CONFIG_PATH": "/etc/redis/redis.conf",
	"POSTGRES_CONFIG_PATH": "/var/lib/pgsql/data/postgresql.conf",
	"POSTGRES_HBACONF_PATH": "/var/lib/pgsql/data/pg_hba.conf",
	"POSTGRES_DATA_DIR": "/var/lib/pgsql/data",
	"POSTGRES_DATABACKUP_DIR": "/var/lib/pgsql/data_backup",
	"POSTGRES_TRIGGER_PATH": "/tmp/postgresql.trigger.5432",
	"POSTGRES_USER": "postgres",
	"POSTGRES_GROUP": "postgres",
	"WORKING_SLAVE_BACKUP_ON": "every-5-minutes",
	"SUPPORTED_MASTER_BACKUP_ON": ["0600", "2300"],
	"SINGLE_MASTER_BACKUP_ON": "every-30-minutes",
	"BACKUP_DIR": "/data/rapidpro-backups",
	"SUB_HOURLY_BACKUPS_TO_KEEP": 10,
	"HOURLY_BACKUPS_TO_KEEP": 10,
	"DAILY_BACKUPS_TO_KEEP": 10,
	"MONTHLY_BACKUPS_TO_KEEP": 3,
	"ENABLE_BACKUP": true,
	"SEND_EMAIL_ALERTS": true,
	"SEND_SMS_ALERTS": true,
	"EMAIL_ALERTS_TO": [],
	"SMS_ALERTS_TO": [],
	"ALERT_COMMAND": "rapidpro-alert",
	"SEND_SMS_URL": "http://smsgateway:90/sendsmstt.php",
	"SENS_SMS_FROM": "85355",
	"HTTP_PROXY": null,
	"HTTPS_PROXY": null,
	"CRM_IP_RESOURCE": "admin_addr"
}
```

* Add periodic tasks to `crontab -e` (`root`)

``` sh
# gen static HTML status
# * * * * * /usr/bin/rapidpro-html-state > /home/rapidpro/public_html/rapidpro-state.html

# poke postres database backup script
0,5,10,15,20,25,30,35,40,45,50,55 * * * * /usr/bin/rapidpro-dumpdb > /dev/null

# cleanup obsolete backup files
0 */2 * * * /usr/bin/rapidpro-backup-cleanup > /dev/null

# check for changes in cluster IP
* * * * * /usr/bin/rapidpro-cluster-check > /dev/null
```

* Set default role and status

``` sh
rapidpro-set-role master
rapidpro-set-status disabled
rapidpro-state
```

## PostgreSQL

### Create user `replica`

``` sh
sudo su - postgres

echo -e "# hostname:port:database:username:password\n*:*:*:replica:replicapass" > ~/.pgpass
chmod 600 ~/.pgpass

psql
```

``` sql
\du
CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'replicapass';
\du
```

### Backup original data folder

``` sh
cp -r /var/lib/pgsql/data /var/lib/pgsql/data_orig
```

### Main configuration (master)

`/var/lib/pgsql/data/postgresql.conf`


``` ini
# either put in all servers addresses or * (file is copied from master to slave)
listen_addresses = '*'

# setup replication
wal_level = hot_standby
max_wal_senders = 3
wal_keep_segments = 10

### rapidpro-postgres-start ###
# enable archiving for master
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/data/archive/%f'

# enable hot_standby for slave
# hot_standby = on
### rapidpro-postgres-stop ###
```

`/var/lib/pgsql/data/pg_hba.conf`

``` ini
host    rapidpro_db     rapidpro_user    0.0.0.0/0                   md5
host    replication     replica          127.0.0.1/32                md5
host    replication     replica          10.172.20.201/32            md5
host    replication     replica          10.172.20.202/32            md5
host    replication     replica          10.172.20.200/32            md5
```

### Create archive folder

``` sh
mkdir -p /var/lib/pgsql/data/archive/
chmod 700 /var/lib/pgsql/data/archive/
chown -R postgres:postgres /var/lib/pgsql/data/archive/
```

### Slave-only setup

**SUSE**

``` sh
systemctl stop postgresql
su - postgres

mv /var/lib/pgsql/data /var/lib/pgsql/data_orig

mkdir -p /var/lib/pgsql/data
chmod 700 /var/lib/pgsql/data
# edit master IP bellow
pg_basebackup -h 10.172.20.202 -U replica -D /var/lib/pgsql/data -P --xlog
```

Use `replicapass` as password.

### Recovery (slave)

``` sh
touch /var/lib/pgsql/data/recovery.conf
chmod 600 /var/lib/pgsql/data/recovery.conf
```

`/var/lib/pgsql/data/recovery.conf`

``` ini
standby_mode = 'on'
primary_conninfo = 'host=sms1 port=5432 user=replica password=replicapass'
restore_command = 'cp /var/lib/pgsql/data/archive/%f %p'
trigger_file = '/tmp/postgresql.trigger.5432'
```

### Backup folder

``` sh
mkdir -p /data/rapidpro-backups
chown -R postgres:postgres /data/rapidpro-backups
chmod 755 /data/rapidpro-backups
```

### Restart & check

``` sh
systemctl restart postgresql
netstat -plntu
```

#### Ensure replication is working

``` sh
su - postgres
psql
```

__On the master__

``` sql
SELECT application_name, state, sync_priority, sync_state FROM pg_stat_replication;
SELECT * FROM pg_stat_replication;

CREATE TABLE replica_test (name varchar(100));
INSERT INTO replica_test VALUES ('this is from master');
INSERT INTO replica_test VALUES ('another row');
```

__On the slave__

``` sql
select * from replica_test;
INSERT INTO replica_test VALUES ('this is SLAVE');
```

## redis

* [Replication](https://redis.io/topics/replication)

`/etc/redis/default.conf`

``` ini
# customize for each server
bind 127.0.0.1 10.172.20.201

### rapidpro-redis-start ###

# slave-only config
slaveof masterIP 6379
### rapidpro-redis-stop ###
```

``` sh
systemctl restart redis-server
systemctl enable redis-server
```

## nginx

* Edit `/etc/nginx/sites-enabled/rapidpro`
Add to both servers:

``` nginx
    # shortcut for statically-generated rapidpro status
    location /server-state {
        alias /home/rapidpro/public_html;
        index rapidpro-state.html;
    }
```

* Reload nginx `nginx -s reload`
* Test at [http://10.172.20.202/server-state](http://10.172.20.202/server-state/)


## monit

### PostgreSQL

* Create user `root` and database `root`

``` sql
\du
CREATE USER root;
CREATE DATABASE root;
GRANT CREATE,CONNECT,TEMP ON DATABASE root to root;
\du
```

* Add to `/var/lib/pgsql/data/pg_hba.conf`

``` 
host    root            root             127.0.0.1/32                trust
```

* Create monit files

``` sh
mkdir -p /etc/monit/conf-{available,enabled}
```
* Edit `/etc/monitrc` and append

``` monit
  include /etc/monit/conf-enabled/*
```

`/etc/monit/conf-available/monit`

``` monit
# poll every 30s but start only after 4mn (boot)
set daemon 30 with start delay 240

set httpd port 2812 and
    address 0.0.0.0
    allow localhost
    allow 0.0.0.0/0.0.0.0
    allow guest:guest read-only
    allow admin:monit

set mailserver smtp.gmail.com port 587
    username "rapidpro.cims@gmail.com" password "<gmail password here>"
    using ssl

set alert <your email here>
```

* `/etc/monit/conf-available/system`

``` monit
check system $HOST
    if loadavg (5min) > 3 for 4 cycles then alert
    if loadavg (15min) > 2 for 4 cycles then alert
    if memory usage > 90% then alert
    if cpu usage (user) > 80% for 3 cycles then alert
    if cpu usage (system) > 80% for 3 cycles then alert

check filesystem rootfs with path /
    if space usage > 80% then alert

check filesystem varfs with path /var
    if space usage > 80% then alert
```

`/etc/monit/conf-available/nginx`

``` monit
check process nginx with pidfile /var/run/nginx.pid
    group rapidpro
    start program = "/usr/bin/systemctl start nginx"
    stop  program = "/usr/bin/systemctl stop nginx"
    restart  program = "/usr/bin/systemctl restart nginx"

    if failed port 80 protocol http request "/" then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"

check host rapidpro-web with address localhost
    if failed port 443 protocol https with ssl options {selfsigned: allow} then alert
```

``` sh
touch /etc/monit/conf-available/nginx.slave
```

`/etc/monit/conf-available/postgresql`

``` monit
check process postgresql-master with pidfile /var/lib/pgsql/data/postmaster.pid
    group rapidpro
    start program = "/usr/bin/systemctl start postgresql"
    stop  program = "/usr/bin/systemctl stop postgresql"
    restart  program = "/usr/bin/systemctl restart postgresql"

    if failed host localhost port 5432 protocol pgsql then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"
```

`/etc/monit/conf-available/postgresql.slave`

``` monit
check process postgresql-slave with pidfile /var/lib/pgsql/data/postmaster.pid
	group rapidpro
	start program = "/usr/bin/systemctl start postgresql"
	stop  program = "/usr/bin/systemctl stop postgresql"
	restart  program = "/usr/bin/systemctl restart postgresql"

	# if failed host localhost port 5432 type TCP then restart

	if 2 restarts within 3 cycles then unmonitor
	if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"
```

`/etc/monit/conf-available/redis-server`

``` monit
check process redis-server with pidfile /var/run/redis/default.pid
    group rapidpro
    start program = "/usr/bin/systemctl start redis-server"
    stop  program = "/usr/bin/systemctl stop redis-server"
    restart  program = "/usr/bin/systemctl restart redis-server"

    if failed host 127.0.0.1 port 6379 protocol redis then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"
```

`/etc/monit/conf-available/rapidpro`

``` monit
check process rapidpro-uwsgi with pidfile /var/run/rapidpro/uwsgi.pid
    group rapidpro
    start program = "/usr/bin/systemctl start rapidpro"
    stop  program = "/usr/bin/systemctl stop rapidpro"
    restart  program = "/usr/bin/systemctl restart rapidpro"

    if failed uid "rapidpro" then restart
    if failed gid "www-data" then restart

    if failed host 127.0.0.1 port 3030 then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"
```

``` sh
touch /etc/monit/conf-available/rapidpro.slave
```

`/etc/monit/conf-available/celerybeat`

``` monit
check process celerybeat with pidfile /var/run/rapidpro/celerybeat.pid
    group rapidpro
    start program = "/usr/bin/systemctl start celerybeat"
    stop  program = "/usr/bin/systemctl stop celerybeat"
    restart  program = "/usr/bin/systemctl restart celerybeat"

    if failed uid "rapidpro" then restart
    if failed gid "www-data" then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"
```
    
``` sh
touch /etc/monit/conf-available/celerybeat.slave
```

`/etc/monit/conf-available/celery`

``` monit
check process celery-default-node with pidfile /var/run/rapidpro/celery-default-node.pid
    group rapidpro
    start program = "/usr/bin/systemctl start celery"
    stop  program = "/usr/bin/systemctl stop celery"
    restart  program = "/usr/bin/systemctl restart celery"

    if failed uid "rapidpro" then restart
    if failed gid "www-data" then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"

check process celery-flow-node with pidfile /var/run/rapidpro/celery-flow-node.pid
    group rapidpro
    start program = "/usr/bin/systemctl start celery"
    stop  program = "/usr/bin/systemctl stop celery"
    restart  program = "/usr/bin/systemctl restart celery"

    if failed uid "rapidpro" then restart
    if failed gid "www-data" then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"


check process celery-handler-node with pidfile /var/run/rapidpro/celery-handler-node.pid
    group rapidpro
    start program = "/usr/bin/systemctl start celery"
    stop  program = "/usr/bin/systemctl stop celery"
    restart  program = "/usr/bin/systemctl restart celery"

    if failed uid "rapidpro" then restart
    if failed gid "www-data" then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"

check process celery-msg-node with pidfile /var/run/rapidpro/celery-msg-node.pid
    group rapidpro
    start program = "/usr/bin/systemctl start celery"
    stop  program = "/usr/bin/systemctl stop celery"
    restart  program = "/usr/bin/systemctl restart celery"

    if failed uid "rapidpro" then restart
    if failed gid "www-data" then restart

    if 2 restarts within 3 cycles then unmonitor
    if 2 restarts within 3 cycles then exec "/usr/bin/rapidpro-record-failure"
```

``` sh
touch /etc/monit/conf-available/celery.slave
```

`/etc/monit/conf-available/cluster`

``` monit
check file rapidpro-status with path /etc/rapidpro.status
    group rapidpro

    ignore content = "working"
    if content = "failure" then alert

check program cluster-ip with path "/usr/bin/rapidpro-cluster-ip --raw"
    group rapidpro

    if status != 0 for 2 cycles then alert
    if status != 0 for 2 cycles then exec "/usr/bin/rapidpro-alert 'cluster IP is not on attached to a node'"
```

`/etc/monit/conf-available/network`

``` monit
check network eth0 with interface eth0
   if failed link then alert

check host google with address google.com
    if failed ping for 3 cycles then exec "/usr/bin/rapidpro-alert 'internet connection lost (google ping)'"
    if failed port 443 protocol https for 3 cycles then exec "/usr/bin/rapidpro-alert 'internet connection lost (google https)'"
```

`/etc/monit/conf-available/smsgateway`

``` monit
check host smsgateway with address smsgateway
    group rapidpro

    if failed ping then alert
    if failed port 90 protocol http request "/" then alert
```

* Create symlinks for those which should always be present

``` sh
ln -sf /etc/monit/conf-available/{monit,system,network,cluster,smsgateway} /etc/monit/conf-enabled/
```