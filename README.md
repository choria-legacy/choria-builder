Choria Collective Builder
=========================

## What

This configures NATS and a number of MCollective instances running
in your home directory as a normal user with a set of plugins of
your choice.

This is primarily meant as an aid in developing plugins, you can run
1 or 10 instances on your machine and interact with them over the
middleware.

Be aware that this will run as non root typically and of course if you
do things that can only be done one at a time such as installing packages
this will not really work too well.

## Prerequisites

You should put the NATS daemon in your PATH, grab the binary for your platform
from their page https://github.com/nats-io/gnatsd/releases

You should install the *nats-pure* gem into your shell

## Getting Started

First you have to clone this repo and provide the Choria plugins:

```
$ cd ~/temp
$ git clone https://github.com/choria-io/choria-builder.git
$ git clone https://github.com/choria-io/mcollective-choria.git
$ cd choria-builder
$ mkdir plugins
$ cp -R ../mcollective-choria/lib/mcollective plugins
```

You can now drop your own Plugins into that directory as well.

### Starting NATS

You should now start a NATS instance, this starts one listening on
*localhost:4222*:

```
$ rake nats_server
Starting a NATS instance on localhost:4222 press ^c to terminate

    TLS Certificate: /home/rip/temp/choria-builder/collective/ssl/certs/localhost.pem
            TLS Key: /home/rip/temp/choria-builder/collective/ssl/private_keys/localhost.pem
     CA Certificate: /home/rip/temp/choria-builder/collective/ssl/certs/ca.pem
           PID File: /home/rip/temp/choria-builder/collective/pid/gnats.pid

Creating SSL certificate for localhost
[11659] 2017/02/26 19:31:47.770148 [INF] Starting nats-server version 0.9.6
```

This NATS instance runs in the foreground with debug and trace enabled
so you can see the messages that travel over the wire. As it runs in
the foreground you probably want to do this in a dedicated terminal.

### Starting MCollective

You can now create your instances of MCollective:

First it will ask you some questions, this lets you adjust which version
of MCollective to run, how many instances and their names:

```
$ rake create
MCollective GIT Repository (git://github.com/puppetlabs/marionette-collective.git):
MCollective Branch Name (2.9):
MCollective Version (2.9):
Instances To Create (10):
Instance Count Start (0):
Instance Name Prefix (dev1):

  .......


Created a collective with 10 members:

To recreate this collective use this command:

  MC_SOURCE=git://github.com/puppetlabs/marionette-collective.git \
  MC_SOURCE_BRANCH=2.9 \
  MC_VERSION=2.9 \
  CHORIA_COUNT=10 \
  CHORIA_COUNT_START=0 \
  CHORIA_PREFIX=dev1 \
  rake create

The collective instances and client are stored in collective/*

Use rake start to start the collective, rake -T to see commands available to start,
stop and update it.
```

It will then do the various git checkouts and sets up a bunch of instances.
As you can see it also gives you an option to copy a command that will recreate
this collective without all the questions being asked.

```
$ rake status
Collective Status:

NATS: running pid 11659

dev1-0.choria: stopped
dev1-1.choria: stopped
dev1-2.choria: stopped
dev1-3.choria: stopped
dev1-4.choria: stopped
dev1-5.choria: stopped
dev1-6.choria: stopped
dev1-7.choria: stopped
dev1-8.choria: stopped
dev1-9.choria: stopped
```

Lets go ahead and start the collective, you'll have a *logs* directory
with a log file for each instance in debug level:

```
$ rake start
Starting collective member dev1-0.choria
Starting collective member dev1-1.choria
Starting collective member dev1-2.choria
Starting collective member dev1-3.choria
Starting collective member dev1-4.choria
Starting collective member dev1-5.choria
Starting collective member dev1-6.choria
Starting collective member dev1-7.choria
Starting collective member dev1-8.choria
Starting collective member dev1-9.choria

$ rake status
Collective Status:

NATS: running pid 11659

dev1-0.choria: running pid 11827
dev1-1.choria: running pid 11870
dev1-2.choria: running pid 11913
dev1-3.choria: running pid 11956
dev1-4.choria: running pid 12000
dev1-5.choria: running pid 12043
dev1-6.choria: running pid 12086
dev1-7.choria: running pid 12129
dev1-8.choria: running pid 12172
dev1-9.choria: running pid 12215
```

## Accessing MCollective

You can set up a shell that will communicate with this collective:

```
$ rake shell

Running /bin/zsh to start a subshell with MCOLLECTIVE_EXTRA_OPTS and RUBYLIB set

Please run the following once started:

    PATH=/home/rip/work/github/choria-builder/collective/client.choria/bin:$PATH

To return to your normal shell and collective just type exit

$ PATH=/home/rip/work/github/choria-builder/collective/client.choria/bin:$PATH
$ mco ping
dev1-0.choria                            time=45.87 ms
dev1-4.choria                            time=46.31 ms
dev1-2.choria                            time=47.74 ms
dev1-1.choria                            time=48.21 ms
dev1-8.choria                            time=52.66 ms
dev1-3.choria                            time=53.80 ms
dev1-5.choria                            time=54.21 ms
dev1-9.choria                            time=54.55 ms
dev1-7.choria                            time=54.91 ms
dev1-6.choria                            time=55.20 ms


---- ping statistics ----
10 replies max: 55.20 min: 45.87 avg: 51.35
```

You can do most things here, RPC commands etc, any plugins you put into the
plugins directory should be loaded etc.

## Updating MCollective

Should you want to change your plugins while developing you can update
your code in *plugins*, you can also update the various config file templates
or put in new *facts* or *classes.txt* in the *templates* directory.

Once you've done that, you can update everything, copy new plugins, recreate
config files and restart everything.  If you want to update the mcollective
code you will have to recreate this collective:

NOTE: Make sure you are not inside *rake shell* when running this:

```
$ rake update
```

## Cleaning up

Everything can shut down, deleted and cleaned up easily:

```
$ rake clean
Stopping collective member dev1-0.choria
Stopping collective member dev1-1.choria
Stopping collective member dev1-2.choria
Stopping collective member dev1-3.choria
Stopping collective member dev1-4.choria
Stopping collective member dev1-5.choria
Stopping collective member dev1-6.choria
Stopping collective member dev1-7.choria
Stopping collective member dev1-8.choria
Stopping collective member dev1-9.choria
Stopping NATS on pid 11659
```

Everything will be deleted but the logs and plugins will remain

