RapidPro Controller
===================

A set of scripts to handle the management of the RapidPro components within the Master/Slave Cluster at CIMS.

* `rapidpro_controller`
* [`github: rgaudin/rapidpro-controller`](https://github.com/rgaudin/rapidpro-controller)
* [`pypi: rapidpro_controller`](https://pypi.python.org/pypi/rapidpro_controller)

# Installation

``` sh
pip install -U rapidpro_controller
```

## Configuration

* Config file: `/etc/rapidproctl.conf`
* __Attention__: Config file is a `JSON` file. It must be valid json.
 * `true` and `false` (lowercase).
 * last element of list has no `,` at the end.
 * last variable in config is not followed by `,`.
 * only double quotes are allowed: `"xx"`
* Config file __can/should__ be identical in the two servers.
* Sample configuration with all variables: `/usr/lib/python2.7/site-packages/rapidpro_controller/rapidproctl.conf.sample`
* If `rapidproctl.conf` is not readable or for all variables not in it, scripts uses default values.
* Default values are OK for the standard CIMS/SUSE installation/

### Important variables:

__Cluster__

 * `CRM_IP_RESOURCE`: name of the `ha-proxy` resource which handles the virtual IP. It is usually an `IPAddr2` resource and can be found in Hawk or via `crm status`
 * `SERVERA`: hostname of one of the two servers.
 * `SERVERB`: hostname of one of the other server.


__SMS sending__

* `SEND_SMS_ALERTS`: whether SMS are to be sent when calling `rapidpro-alert`.
* `SMS_ALERTS_TO`: list of phone numbers to send SMS to when using `rapidpro-alert` and `rapidpro-test-sms`.
* `SEND_SMS_URL`: URL of the SMS Gateway. It will pass variables `to`, `from` and `text` to this URL using a `POST` request.
* `SEND_SMS_FROM`: the name or phone number to use as `from` parameter to the SMS Gateway.

__Email sending__

* `SEND_EMAIL_ALERTS`: whether emails are to be sent when calling `rapidpro-alert`.
* `EMAIL_ALERTS_TO`: list of email addresses to send message when calling `rapidpro-alert`

# Roles of each script

* __`rapidpro-alert`__: send an alert by email/sms to people in config.
* __`rapidpro-backup-cleanup`__: removes obsolete backups
* __`rapidpro-change-role`__: change configuration based on role (`--force`)
* __`rapidpro-change-status`__: change configuration for new status (`--force`)
* __`rapidpro-cluster-available`__: make specified (or current) node available to the virtual IP
* __`rapidpro-cluster-check`__: check which node has the virtual IP and calls `rapidpro-record-ipmaster` or `rapidpro-record-ipslave` __on change__.
* __`rapidpro-cluster-ip`__: gives the name of the node having the virtual IP
* __`rapidpro-cluster-requestip`__: request virtual IP for current node
* __`rapidpro-cluster-setipmaster`__: set the virtual IP to the specified node
* __`rapidpro-cluster-status`__: display `ha-proxy` cluster status
* __`rapidpro-cluster-unavailable`__: make specified (or current) node unavailable to the virtual IP
* __`rapidpro-config`__: display the parsed config file. helps debug config file issue.
* __`rapidpro-configure-master`__: reconfigure some components (redis,postgres) for master role.
* __`rapidpro-configure-slave`__: reconfigure some components (redis,postgres) for slave role.
* __`rapidpro-disable`__: disable all components with `systemctl`
* __`rapidpro-dumpdb`__: executes db dump logic. might trigger a dump or not depending on config.
* __`rapidpro-enable`__: enable all components with `systemctl`
* __`rapidpro-html-state`__: displays an HTML version of `rapidpro-state` and `rapidpro-peer-state`
* __`rapidpro-monit-reconfigure`__: enable/disable monit for components.
* __`rapidpro-notify-peer`__: calls the `rapidpro-receive-notif` on other node to inform this node is in failure.
* __`rapidpro-peer-state`__: display state (role, status) of other node
* __`rapidpro-pgdump-to`__: executes a db dump to the specified location
* __`rapidpro-postgres-status`__: shortcut to dispay postgres process status for checking WAL operations
* __`rapidpro-receive-notif`__: triggers tilt to master role if current node is a ready slave
* __`rapidpro-record-failure`__: set this node in failure and triggers actions (stop, notify,alert, etc)
* __`rapidpro-record-ipmaster`__: triggers role change as node just received the virtual IP
* __`rapidpro-record-ipslave`__: triggers role change as node just lost the virtual IP
* __`rapidpro-restart`__: restarts all components with `systemctl`
* __`rapidpro-set-clustermode`__: set specified attribute for the specified (or current) node on `ha-proxy`
* __`rapidpro-set-role`__: write role file with specified role
* __`rapidpro-set-status`__: write status file with specified status
* __`rapidpro-start`__: starts all components with `systemctl`
* __`rapidpro-state`__: display state (role, status) of current node
* __`rapidpro-status`__: display all components status with `systemctl`
* __`rapidpro-stop`__: stops all components with `systemctl`
* __`rapidpro-test-sms`__: 

# External configuration

## Cron tasks

`crontab -l` to list current configuration (as `root`).

### `rapidpro-dumpdb`

Launched every 5 minutes, it allows up-to 5mn-old database dumps to be created (depending on the config).

If a backup should be created, it launches `rapidpro-pgdump-to`, otherwise, it returns quickly.

### `rapidpro-backup-cleanup`

Launched every 2 hours, it looks at the database dump directory and removes those that it considers obsolete.

Obsolescence is based on config.

### `rapidpro-cluster-check`

Launched every minutes, it checks which node has the virtual IP and compares it to the previous state (`/etc/rapidpro.clusterip`)

If it changed, it triggers another script `-record-ipmaster` or `-record-ipslave`.

## Monitoring (`monit`)

* [monit sms1](http://10.172.20.201:2812/)
* [monit sms2](http://10.172.20.201:2812/)
* Credentials: `admin`/`monit`

`monit` watches all RapidPro components for failure.
It is configured to check them all _every 30s_.

* Should a component fail (not running, not available), monit tries to restart it first.
* If 2 restarts happen within a 90s timeframe, monit will stop watching it and call `rapidpro-record-failure` as well as send an email.

### Configuration

monit is supposed to __run all the time__, whether the node is in failure or not and wether the node is a master or a slave.

Depending on the role of the server (master or slave), monit configuration changes: in slave mode, only PostgreSQL and redis are supposed to run.

monit configuration is stored in `/etc/monit/conf-available/`.

Switch between `master` and `slave` configuration is done through `symlinks` to the `/etc/monit/conf-enabled/` folder and a call to `monit reload`.

`rapidpro-monit-reconfigure [enable|disable]` is used to handle this change.

When `disable`, only system-related files are kept symlinked.

When `enable`, all components are symlinked. If the node is in slave mode and a `component.slave` exists, then it is symlinked instead of the component one.

__System-related configurations__ (always present):

* `monit`: general monit configuration (httpd, email)
* `system`: system checks: cpu, load, mem, free space
* `cluster`: checks that virtual IP is attached to a node
* `network`: network checks: `eth0` and external internet connectivity
* `smsgateway`: access to the SMS Gateway server

# Database Backups

Database dumps feature can be turned on/off from the config.

``` json
	"ENABLE_BACKUP": true,
	"BACKUP_DIR": "/data/rapidpro-backups",
```

--

In a `master`/`slave` strategy, database dumps are usually done onto the slave node so that it doesn't consume resources on the master.

Because roles of each node might change, traditional backup tools are not suitable.

`rapidpro-dumpdb` solves this by applying a different strategy depending on the state of the node.

A node can be in three different state (backup-wise):

* `WORKING_SLAVE`: a slave which is working. best scenario for backup.
* `SUPPORTED_MASTER`: a master with a working slave (opposite of previous)
* `SINGLE_MASTER`: a master without a working slave (other node in failure)

Those 3 different states call for different backup strategies. Periodicity of backups for each is defined in the config:

``` json
	"WORKING_SLAVE_BACKUP_ON": "every-5-minutes",
	"SUPPORTED_MASTER_BACKUP_ON": ["0600", "2300"],
	"SINGLE_MASTER_BACKUP_ON": "every-30-minutes",
```

Default configuration creates a backup every 5mn on a working slave, at 6am and 11pm on a supported master and every 30 mn on a single master. A compromise between the safety of frequent dumps and the extra resources they require.

--
All backup files are time-coded in their file names: `rapidpro-20171215_1105.sql`

The date is present __`20171215`__ as well as the hour code __`1105`__. This fie reference the backup of December 15th 2017 at 11:05am.

All Backups are tied to a 5mn time code. If a backup runs at 11:03, it will get the `1100` time code.

--

In addition, `rapidpro-backup-cleanup` is launched periodically to remove obsolete backups from the backup directory.

It considers four different kind of backups:

* `SUB_HOURLY`: those with a timecode between `XX05` and `XX55`.
* `HOURLY`: those with a timecode of `XX00`
* `DAILY`: those with a timecode of `0000`
* `MONTHLY`: those with a date of `XXXXXX01` and a timecode of `0000`.

Those different kind of backups have configurable obsolescence rules.

``` json
	"SUB_HOURLY_BACKUPS_TO_KEEP": 10,
	"HOURLY_BACKUPS_TO_KEEP": 10,
	"DAILY_BACKUPS_TO_KEEP": 10,
	"MONTHLY_BACKUPS_TO_KEEP": 3,
```

Default configuration keeps the latest 10 of each type except for the monthly ones (3 are kept) resulting in a total of 33 backup files kept.

At the moment, DB dumps are ~120MB large so it represents less than 4GB of storage.

# Usage

## `rapidpro-alert`

``` sh
rapidpro-alert [message]
```

`rapidpro-alert` is used to send an alert to the people specified in the configuration.

``` json
	"SEND_EMAIL_ALERTS": true,
	"SEND_SMS_ALERTS": true,
	"EMAIL_ALERTS_TO": [],
	"SMS_ALERTS_TO": [],
```

Email alerts uses the `mail` command on the node while SMS alerts uses the SMS Gateway.

--

if the message is not specified, an automatic one is created including the hostname of the host and the datetime.

All recipients receives all alerts.