# RapidPro Cluster Recovery Manual

## Periodic monitoring

Although a software monitoring is provided through `monit` and service should continue to function, it is **very important** to periodically check the status of the solution, especially during the first months of operations:

* An unforeseen incident could fail the system without monitoring noticing.
* A failure could occur outside the monitored parts (SMS ?)
* The software monitoring could malfunction.
* The master/slave tilt could malfunction.
* The alter system could malfunction.

---

To check the status of the servers, three tools:

* Hawk Interface, to check the `ha-cluster` status. (`hacluster`/`linux`).
* `monit` Web Interface, to check all monitored services. (`admin`/`monit`).
* RapidPro Web interface itself, to ensure the actual product is working.

**Links**:

* virtual IP: [Hawk](https://10.172.20.200:7630/), [monit](http://10.172.20.200:2812), [RapidPro UI](https://10.172.20.200/)
* sms1: [Hawk](https://10.172.20.201:7630/), [monit](http://10.172.20.201:2812)
* sms2: [Hawk](https://10.172.20.202:7630/), [monit](http://10.172.20.202:2812)

### Using monit

The following *services* are checked by monit.

Two types of reactions to a *monit failure*:

* an alert to the admins for to serve as a warning for failure that are not critical or for which we have no control over.
* a call to the `rapidpro-record-failure` script which itself triggers an alert but also starts the tilt mechanism.

While all non-components checks (system, server, etc) are present at all times, rapidpro components are either not present or different for slave mode where only a working PostgreSQL and redis replication is required.

* System (CPU, RAM, Load average). Alert-only
* `rootfs` & `varfs` (`/` and `/var` where postgres and redis resides). Alert-only
* `nginx` (*master only*). Tests a `GET` request on `/` for both HTTP and HTTPS.
* `postgresql-*`. Tests an actual `psql` connection (*master*) or a `TCP` connection to postgresql port (*slave*).
* `redis-server`. Tests a `TCP` connection to redis port for both master and slave.
* `rapidpro` (*master-only*). Tests a `TCP` connection to the `uwsgi` port serving temba.
* `celery-*` & `celerybeat`. Tests that the corresponding processes are running.
* `rapidpro-status`: displays output of command. Fails only is status is `failure` or unknown. Alert-only.
* `cluster-ip`: displays output of command. Fails only if IP is not assigned to any node. Alert-only
* `network`: Tests that `eth0` is working. Alert-only.
* `google`: Tests that Google.com can be reached via HTTP and HTTPS. Alert-only.
* `sms-gateway`: Tests that the SMS gateway responds to HTTP on port `90`.

#### On a master node

![](monit-sms2-master.png)

#### On a slave node

![](monit-sms1-slave.png)

---

__Note__: Program checks (`cluster-ip` and `rapidpro-state`) are slow and might not be completely up-to-date ; could be 1mn old. Use command line version for live results.

### Using the command line

``` sh
# list ha-cluster's status (rapidproctl's shortcut)
rapidpro-cluster-status
# list ha-cluster's status
crm_mon -1

# display which node owns the virtual IP
rapidpro-cluster-ip

# display the node's role and status
rapidpro-state

# display the other node's role and status
rapidpro-peer-state

# display postgres processes
rapidpro-postgres-status

# check the running postgres mode (wal sender or wal receiver)
rapidpro-postgres-status |grep wal

# output of rapidpro-cluster-status and systemctl status for all components
rapidpro-status

# verify that rapidproctl config is working (if actions/alerts not triggered)
rapidpro-config

# sends an SMS to all numbers configured
rapidpro-test-sms

# sends an email to all addresses in config + SMS to all numbers in config
rapidpro-alert
```

## Checking SMS capability

Because all interactions with users happen over SMS, it is important to periodically ensure that SMS are working, both `inbound` and `outbound`, for all  operators.

Checks :

1. Create a Flow with a custom trigger (`test`?) that would simply reply. Sending this SMS would allow to quickly make sure both inbound and outbound SMS are working.
* Make sure a log or other trace capability exists on the SMS gateway to help identify whether a non-working SMS issue is related to rapidpro, the gateway or the operator.
* Make sure you have a test number registered to the system and follow-up with each expected message. This would help identify rapidpro-related messaging issues (celery queue mostly)

## What to do in case of failure/alert?

1. Is this *just an alert* (ie. a warning) or a failure?
* What is a the current state of both servers?
* Did the tilt mechanism function?
* What triggered the failure?
* Find out the root cause
* Fix it
* Put failed server into working slave mode

## Check state of both servers ; ensure tilt worked

Using monitoring tools, check what triggered the alert:

* A warning (disk space, CPU) ? Important but not urgent.
* A rapidpro issue (component not working) ? Should have triggered the tilt.
* A non-rapidpro issue (SMS Gateway, Internet connectivity) ? Won't trigger tilt as it would probably affect both servers. Find and fix the issue.

Get a sense of **both servers state**, using command-line:

* Do we have a *working master*? `rapidpro-state` `rapidpro-peer-state`
 * no: is one in `transient` status? Give it a couple minutes.
* Is this *working master* **really working** ? `rapidpro-status` **Verify** with RapidPro UI.
* Which server has IP? `rapidpro-cluster-ip` Is this the working master?
 * no: change it (`rapidpro-cluster-setipmaster <node>`)
* Is failed server in `failure` status ? Are his services stopped?

## Triggering the tilt manually

When to trigger the tilt mechanism manually?

* RapidPro is not working yet the tools did not trigger the mechanism
* `rapidpro-record-failure` was called but it did not work/complete.

**Before** triggering it manually, **check** that the servers are not in `transient` state (ie. they are in the process of changing status/role).

If a server stays in `transient` state for too long (several minutes), it might be hung. consider it failed.


``` sh
# a quick backup just in case
rapidpro-pgdump-to /tmp/before-intervention.sql
```

### Automatically

``` sh
# on the failed server
rapidpro-record-failure

# on the slave server (if the failed one is not working/communicating)
rapidpro-receive-notif
```

Run it on the failed server first as this is what the monitoring should have done. Launching it manually will let you know if the tilt works (wasn't called) or if it fails.

**If it fails**, please report it to rapidpro_controller developer with the maximum possible information (output of command at least) so this use case can be taken care of.

### Manually

If the automatic method did not work, you can trigger all its actions individually ; both on the (failed) master and the slave.

---
**Always** check the states of both servers after using any of those commands to ensure that it worked as expected.

#### On the slave

Assuming the slave has to become the master, use the following.

**Note**: rapidproctl never tries to automatically reconfigure a non-working slave as a master. It is assumed that it if is in `failure` state, it needs to be fixed first and if it is in `transient` state, it has to finish its transition.

``` sh
# make sure we're either in working or disabled state
rapidpro-state

# change role to master
# check rapidpro_controller docs to break it down if this script fails
rapidpro-change-role master

# make sure we have the IP
rapidpro-cluster-ip

# get the IP if we don't have it
rapidpro-cluster-requestip

# check state and monitoring
rapidpro-state
monit summary
```

At this point, `rapidpro-state` should be `master: working`.

#### On the master

Use the following commands to make sure the former master is out of order and users won't accidentally use it.

``` sh
# what's the current state of the server
rapidpro-state

# mark server in failure (to prevent fail-over to give it master role back)
rapidpro-set-status failure

# which server has the IP?
rapidpro-cluster-ip

# get off the IP (turn slave into master first)
rapidpro-cluster-unavailable

# what is still running?
rapidpro-status
monit summary

# disable monit if its still configure for master (if it's watching celery or nginx for instance)
rapidpro-monit-reconfigure disable

# disable services (so they don't start upon reboot)
rapidpro-disable

# stop the services if they are running
rapidpro-stop
```

At this point, `rapidpro-state` should be `master: failure`.

## Identifying an issue

Calls to `rapidpro-record-failure` trigger an alert but the alert does not describe what triggered it.

After following previous steps, your have a working master (previously the slave) and your master is in failure mode.

* stop monit completely so it does not disrupt your investigation `systemctl stop monit`
* check logs for clues
* start services individually

Remember the following dependency order for rapidpro components:

1. `postgresql` depends on nothing
* `redis-server` depends on nothing
* `rapidpro` depends on both `postgresql` and `redis-server`
* `celerybeat` and `celery` depends on `postgresql`, `redis-server` and working `rapidpro` setup (does not require it to be running though).
* `nginx` depends on nothing but displays output of `rapidpro` so returns a 500 error if it crashes or a HTTP 502 (bad gateway) in case it is not running.


## Post-issue recovery

At this point, `rapidpro-state` should be `master: failure` but you would have fixed the original issue and know that this node is OK.

Use the following commands to come back as a working slave.

``` sh
# verify our state
rapidpro-state

# verify peer state: should be master: working
verify-peer-state

# verify peer node has the IP
rapidpro-cluster-ip

# change status to disabled (mandatory)
rapidpro-change-status disabled

# verify new state ; we should be master: disabled
rapidpro-state

# change role to slave
# check rapidpro_controller docs to break it down if this script fails
rapidpro-change-role slave

# verify new state ; we should be slave: working
rapidpro-state

# verify monitoring
monit summary
```

## Use cases

### Two masters

Two working masters is a dangerous scenario even if only one has the virtual IP and thus is the only one to receive requests because:

* *mCessation* sends outgoing messages at specified times, not only as replies to incoming messages.
* SMS gateway accepts request from any of the two servers.

When in two masters situation:

``` sh
rapidpro-change-status disabled
```

Outgoing SMS are sent by the `celery` service so it is safe to restart other services for debugging.

### Debugging PostgreSQL

In case PostgreSQL is not able to start, you can try

* turn off monit (to make sure it's not starting processes): `rapidpro-monit-reconfigure disable` and `systemctl stop monit`.
* check that no zombie process exists `ps aux |grep postgres`
* check that there is no PID files `ls -al /var/run/postgres`
* Start `pg_ctl` manually and look at the logs

``` sh
su - postgres
pg_ctl -D /var/lib/pgsql/data -l /tmp/postgres-debug.log start
```

### Debugging temba

Temba is configured to return HTTP 500 errors on unexpected errors.

To debug a particular issue :

1. Login as `rapidpro` user.
* Make a backup of the temba `settings.py` file
* Edit `~/settings.py` and enable `DEBUG`
* Start the debug django server (on port 8000)
* Reproduce your issue and observe the error message (debug trace) on screen. Copy it so it can be shared with developers.
* Restore config file.

``` python
DEBUG = True
IS_PROD = False
```

``` sh
su - rapidpro

cp ~/settings.py ~/settings.orig.py

vim ~/settings.py

./manage.py runserver 0.0.0.0:8000

mv ~/settings.orig.py ~/settings.py
```

**Note**: temba needs to be reloaded for any change in `settings.py` to apply. `runserver` is a different process, so it loads the edited version of the config but the running production version (through nginx+uwsgi) is not affected.

**Very important**: restore previous config after your debug **or** restart `rapidpro-restart` all services if you want to make permanent changes.

Otherwise, your changes to the configuration will only be applied at the next restart.


## What happens during the tilt?

The tilt is a change of state for the two servers ; in such behavior might seem unexpected. Here's what happen in case a failure happens on the master:

1. everything is working (master: working, slave: working)
* A service crashes (say `postgresql`).
 * outgoing messages are not sent.
 * incoming messages fail (__lost__).
* monit notices it after `30s` (at most) and restarts it.
* monit verify its status every `30s`.
 * if restart worked, it comes back to normal (outgoing messages are sent, *new* incoming messages are processed).
 * If it failed to restart `2 times` within `90s`, monit sends an email and calls `rapidpro-record-failure`.
* `rapidpro-record-failure` executes:
 1. `rapidpro-set-status failure`
 * `rapidpro-monit-reconfigure disable`
 * `rapidpro-disable`
 * `rapidpro-stop`
 * `rapidpro-cluster-unavailable`
 * At this point `ha-cluster` changes the virtual IP to the slave.
 * `rapidpro-notify-peer` which executes `rapidpro-receive-notif` on the slave.
 * `rapidpro-alert` which sends email+SMS to admins.
* Slave if notified either from `rapidpro-record-ipmaster` (cron task every minute) which itself calls `rapidpro-receive-notif` or the direct call by `rapidpro-notify-peer`.
* `rapidpro-receive-notif` calls:
 1. `rapidpro-change-role master --force` 

That's it for this specific scenario. Please check [rapidpro_controller documentation](RapidProController.md) to learn more about how those scripts react based on different states.

**Note**: During a service outage,

* Incoming SMS would **probably be lost** when contacting the failing server, until the slave is reconfigured and received the IP (**max 3mn**).
* Outgoing SMS *might* be sent twice in rare circumstance (see two masters scenario above).

