---
title: "Puppet Fault Tolerance"
description: "Setup Puppet in a fault tolerant manner to handle failures gracefully."
category: devops
keywords: "devops, puppet, fault tolerance, highly available, redundant, infrastructure, configuration management"
date: 2012-05-01
draft: false
---

Puppet is an infrastructure and configuration management tool-set that no only unifies the management of multiple machines across multiple platforms. It allows for knowledge sharing, elimination of repetition, increase portability of configurations because it is platform agnostic. This is done by overlaying a domain-specific language (DSL) on top that can be shared across multiple systems, platforms, and architectures. By doing this you turn things like `yum install` and `apt-get install` into

```
package { sudo:
  ensure => latest,
}
```

OK, not that you are excited about Puppet we can move on!

## Initial Setup

In this setup we will be running the following solutions to support our puppet infrastructure in a fault tolerant manner. Once the virtual machines are deployed and reachable on the network you will want to do a full system patch to ensure you are running the latest software on your virtual machines.

```
sudo apt-get update
sudo apt-get dist-upgrade
sudo reboot
```

### IP Addressing

There will be two Puppet servers in this setup, one will be active at any given time. Puppet can scale out more than this, but initially this will be fine for our implementation. If you find that you need more hosts, you would just add them to the proxy setup and disable the certificate authority on the new Puppet servers. Below are the IP addresses used in our environment.

 * **IP:** 192.168.1.10 **Hostname:** puppet (Virtual IP)
 * **IP:** 192.168.1.11 **Hostname:** puppet-1
 * **IP:** 192.168.1.12 **Hostname:** puppet-2

### Topology
Below is an example of the topology of the setup. It is a simple two node active/passive setup. This can grow as your number of nodes increases.

![FT Puppet Topology](http://farm8.staticflickr.com/7074/7269133476_d24f0609a6_d.jpg)

### Time Server (NTP)

Now that your system is patched, you should ensure that you have a time server setup. This is imperative for accurate logs as well as certain functionality to work properly in this setup.

Install NTP by running the following command:

```
sudo apt-get install ntp
```


The default configuration in Ubuntu will work just fine if your VM can reach the internet. If not you will need to update the `server` field to an internal NTP server.

## Required Packages

Lets run through the full install of packages that we will need for this setup. This will ensure all the necessary directories and commands are available for the entire setup. I will also put the select install commands in each section for complete-ness. They will return an "already installed" error. I have broken them up into categories with explanations.

Install some useful utilities that will aid in troubleshooting.

```
sudo apt-get install tmux ltrace strace tcpdump lsof htop git vim-nox wget vim-puppet
```

Install Ruby and supporting libraries, these are used for various features for Puppet:

```
sudo apt-get install libffi-ruby libffi6 libffi-dev \
libxml2 libxml2-dev zlib1g zlib1g-dev libssl-dev \
libyaml-dev libyaml-0-2 libreadline6 libreadline6-dev \
ruby ruby-dev ri libruby rubygems rrdtool librrd-ruby \
librrd-dev librrd-ruby1.8 librrd4
```

Now we want fault tolerance without a huge infrastructure. The solution to host configuration management should not be impossible to maintain. The packages installed here are solid solutions that are battle tested. Below is a quick description of what each package is used for in the solution.

- **ucarp** - vIP and Fail-over between Puppet master servers.
- **haproxy** - Handles the balancing between Puppet master servers if/when we bring more online.
- **hatop** - Used to monitor the HAProxy setup.
- **drbd** - SSL file replication between Puppet CA servers.

```
sudo apt-get install drbd-utils ucarp haproxy hatop
```

### Puppet Installation

First we need to add the Puppet Labs apt repository. Run the following to add the repo to apt and add the apt key.


Install the Apt keys:

```
gpg --recv-key 4BD6EC30
gpg -a --export 4BD6EC30 | sudo apt-key add -
```

Update the apt package cache:

```
sudo apt-get update
```

Finally, we can now install the puppet package. This will be installed from the puppet labs apt repository.

```
sudo apt-get install puppet
```

## Data Replication

DRBD will be used to ensure the SSL files are mirrored automatically, this will be more robust then using rsync or similar to ship config files between the servers. First we need to add a block device to the VM, this is the backing device to the drbd setup that we will replicate between the virtual machines. Once the drive has been added to the VM we need to add a partition to the drive.

Run the following to create a partition:

```
sudo fdisk /dev/sdb
```

The output should look similar to the following:

```
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0x0bcb7a4c.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

Command (m for help): n
Partition type:
  p   primary (0 primary, 0 extended, 4 free)
  e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-4194303, default 2048): 2048
Last sector, +sectors or +size{K,M,G} (2048-4194303, default 4194303): 4194303

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

Now we can install drbd and enable the kernel module, run the two commands below to complete this.

```
sudo apt-get install drbd-utils`
sudo modprobe drbd`
```

*If this fails, you may have installed the linux minimal virtual machine install. To fix this you can install the default linux-server kernel, or build the kernel module from source.*

Once installed, we can now create the resource file that will manage the replication for the SSL files for puppet. The default configurations should be fine for our implementation.

### Resource Configuration
We need to create a resource for DRBD to replication. In order to do this you need to have a few things:

- **IP addresses** - We only have one choice for this, for critical replication or larger amounts of data you can dedicated a NIC for this replication.
- **Block Device** - The VMDK that we added to the VM. This is used by DRBD, after we created the partition on this device, we never need to touch it again.
- **DRBD Device** - This is what we will format with a filesystem instead of /dev/sdb1. We will use drbd0, the first available drbd device.

Create a new file on each node with the content below: `sudo vim /etc/drbd.d/r0.res`

```
resource r0 {
   device /dev/drbd0;
   disk /dev/sdb1;
   meta-disk internal;
   on puppet-1 {
      address 192.168.1.11:7789;
   }
   on puppet-2 {
      address 192.168.1.12:7789;
   }
}
```

### Resource Setup

Now that DRBD is configured and ready to go, we need to setup replication and format the newly available replication device.

#### Enabling your resource for the first time

Run the following commands on both nodes.

**Create Device Metadata:**

```
sudo drbdadm create-md r0
```

**Attach to Backing Device:**

```
sudo drbdadm up r0
```

# Observe Proc Filesystem

```
cat /proc/drbd
  version: 8.3.11 (api:88/proto:86-96)
  srcversion: 71955441799F513ACA6DA60
   0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:2096028
    ```

*The Inconsistent/Inconsistent disk state is expected at this point.*

#### The initial device synchronization

**On puppet-1**

Specify the primary device for DRBD, since there is no data we will choose the resource on puppet-1:

```
sudo drbdadm -- --overwrite-data-of-peer primary r0
```

**Verify Replication**

Once we enter the command above, replication will start. We can observe this by running:

```
sudo drbd-overview
```

At this point `/dev/drbd0` is usable with degraded performance.
Wait until `ds:UpToDate/Inconsistent` changes status to `ds:UpToDate/UpToDate`

### Create Filesystem on DRBD Device

We will format the `/dev/drbd0` device instead of the `/dev/sdb1` device. This will format the replicated device instead of the backing device.

Now that we have a block device with replication between our two nodes we can format the block device with a filesystem to make it usable. Run the following to create an ext4 filesystem: `sudo mkfs.ext4 /dev/drbd0`

### SSL Directory Creation

We need to create a folder to mount the shared device to. Run the commands below to create an ssl directory within our Puppet environment.

```
sudo su -
cd /var/lib/puppet
mkdir ssl
exit
```

Now we want to update the `/etc/fstab` file to tell it that `/var/lib/puppet/ssl` is something it should know about. Add the following line to the bottom of your `/etc/fstab`.

```
/dev/drbd0     /var/lib/puppet/ssl     ext4     noauto     0     0
```

Now run the following command to mount the new filesystem on the primary node only, if you do it on the secondary it will try to mount read-only and fail.

```
sudo mount /var/lib/puppet/ssl
```

Verify the mount was successful, run the following commands and compare the last line of each commands output, it should look similar to what is shown below.

```
mount
/dev/drbd0 on /var/lib/puppet/ssl type ext4 (rw)
```

```
sudo df -kh
/dev/drbd0                  2.0G   64M  1.9G   4% /var/lib/puppet/ssl
```

## IP Failover

By providing failover of a shared IP address, we can be certain that HAProxy and Puppet are available on a node. We can also run commands when a failover happens to bring services online at that time. This is very useful when doing an active/passive clustered setup.

Install UCARP which is a linux user-land implementation of Common Address Resolution Protocol (CARP). CARP is much more robust and optimized than the bloated Virtual Router Redundancy Protocol (VRRP).

Install UCARP by running the following command: `sudo apt-get install ucarp`

### Configuration

Failover using UCARP is integrated very well into Ubuntu, all the configuration happens right in the interfaces file. There is a section below for each node in the cluster, this will enable failover to/from each host.

#### puppet-1:

On the first puppet node you want to update the network configuration file to support and enable UCARP: `sudo vim /etc/network/interfaces`

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 192.168.1.11
netmask 255.255.255.0
up route add default gw 192.168.1.1
dns-nameservers 192.168.1.1
dns-search domain.tld
ucarp-vid 50
ucarp-vip 192.168.1.10
ucarp-password s3cur3p@ssw0rd
ucarp-master yes

iface eth0:ucarp inet static
address 192.168.1.10
netmask 255.255.255.0
```


Enable configuration by restarting the networking on the host. Run the following if you are logged in via SSH: `sudo /etc/init.d/networking restart`

#### puppet-2:

Now we will repeat the same task on the 2nd puppet node. The only difference would be the IP address of the host and the `ucarp-master` definition. Run the following to open the network interfaces file and make the necessary changes: `sudo vim /etc/network/interfaces (puppet-2)`

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 192.168.1.12
netmask 255.255.255.0
up route add default gw 192.168.1.1
dns-nameservers 192.168.1.1
dns-search domain.tld
ucarp-vid 50
ucarp-vip 192.168.1.10
ucarp-password s3cur3p@ssw0rd
ucarp-master no

iface eth0:ucarp inet static
address 192.168.1.10
netmask 255.255.255.0
```

Enable configuration by restarting the networking on the host. Run the following if you are logged in via SSH: `sudo /etc/init.d/networking restart`

At this point, the `eth0:ucarp` interface should show up when you run `ifconfig -a` on the puppet-1 primary node. If it does not, check that your `ucarp-*` statements are the same on both nodes except the `ucarp-master` which should be yes on the primary and no on the backup.

**Backup the default vip-up script:** `sudo cp /usr/share/ucarp/vip-up /usr/share/ucarp/vip-up.bak`

**Update the vip-up script:** `sudo vim /usr/share/ucarp/vip-up`

```
#!/bin/sh

# Set the current node as primary
drbdadm primary all

# Mount the HA SSL directory
mount /var/lib/puppet/ssl

# Bring up the ucarp interface
/sbin/ifup $1:ucarp

# Enable HAProxy
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg

# Turn on Puppet CA
/etc/init.d/puppetmaster start

# Turn on Front-end Nginx server
/etc/init.d/nginx start

# Anything Else? Email Admins?
```

**Backup the default vip-down script:** `sudo cp /usr/share/ucarp/vip-down /usr/share/ucarp/vip-down.bak`

**Update the vip-down script:** `sudo vim /usr/share/ucarp/vip-down`

```
#!/bin/sh

# Turn off Nginx Server
/etc/init.d/nginx stop

# Turn off Puppet CA
/etc/init.d/puppetmaster stop

# Unmount the high availability partition
umount /var/lib/puppet/ssl

# Then set the node into secondary state
drbdadm secondary all

# First Kill HAProxy
/usr/bin/pkill -9 haproxy

# Now Kill The UCARP Interface
/sbin/ifdown $1:ucarp
```

## HA Proxy
Now that we have IP failover working we can now setup HAProxy. HAProxy will be used to load balance between each node, both nodes will be running an instance of the HAProxy setup, this will ensure that the loadbalancer itself is redundant as well.

### Configuration

Now we need to tell haproxy where our puppet servers will listen on: `sudo vim /etc/haproxy/haproxy.conf`

```
global
  log                     127.0.0.1 syslog notice
  pidfile                 /var/run/haproxy.pid
  maxconn                 4096
  user                    haproxy
  group                   haproxy
  stats                   socket /var/run/haproxy_stats.sock
  daemon

defaults
  log                     global
  option                  redispatch
  retries                 3
  timeout http-request    10s
  timeout queue           1m
  timeout connect         10s
  timeout client          1m
  timeout server          1m
  timeout http-keep-alive 10s
  timeout check           10s
  maxconn                 2048

frontend puppet 192.168.1.10:8140
  mode                    tcp
  default_backend         puppet0

backend puppet0
  mode                    tcp
  option                  ssl-hello-chk
  balance                 roundrobin
  server                  server1 192.168.1.11:8140 check
  server                  server2 192.168.1.12:8140 check backup
```

## Puppet

Now it is time to setup the Puppet installation, for this we will use Puppet master hosted by Unicorn to UNIX sockets. Then we will load balance the connections to these sockets using Nginx.

### Configuration

We need to update our puppet configuration file to tell it to use the DNS record for the shared IP and not the hostname. `sudo vim /etc/puppet/puppet.conf`. I have also added some reporting configuration options in there, changes those to suit your environment.

```
[main]
  logdir=/var/log/puppet
  vardir=/var/lib/puppet
  ssldir=/var/lib/puppet/ssl
  rundir=/var/run/puppet
  factpath=$vardir/lib/facter
  templatedir=$confdir/templates
  server = puppet.domain.tld
  certname = puppet.domain.tld
  config = /etc/puppet/puppet.conf
  reports = store,log,tagmail,rrdgraph
  bindaddress = 192.168.1.10 # UCARP vIP
```

Update **tagmail.conf** configuration to email on error: `sudo vim /etc/puppet/tagmail.conf`

```
err: admins@domain.tld
```

Puppet will be hosted via a Rack application server, for this we need a **config.ru** file to tell Rack about the application: `sudo vim /etc/puppet/config.ru`

```
# config.ru, for use with every rack-compatible web server.
# SSL needs to be handled outside this, though.

# if puppet is not in your RUBYLIB:
# $LOAD_PATH.unshift('/opt/puppet/lib')

$0 = "master"

# if you want debugging:
# ARGV \<\< "--debug"

ARGV \<\< "--rack"

# Rack applications typically don't start as root.  Set --confdir to prevent
# reading configuration from ~/.puppet/puppet.conf
ARGV \<\< "--confdir" \<\< "/etc/puppet"

# NOTE: it's unfortunate that we have to use the "CommandLine" class
#  here to launch the app, but it contains some initialization logic
#  (such as triggering the parsing of the config file) that is very
#  important.  We should do something less nasty here when we've
#  gotten our API and settings initialization logic cleaned up.
#
# Also note that the "$0 = master" line up near the top here is
#  the magic that allows the CommandLine class to know that it's
#  supposed to be running master.
#
# --cprice 2012-05-22

require 'puppet/util/command_line'
# we're usually running inside a Rack::Builder.new {} block,
# therefore we need to call run *here*.
run Puppet::Util::CommandLine.new.execute
```

Now setup the **unicorn.conf** file to tell unicorn how many workers to spawn etc: `sudo vim /etc/puppet/unicorn.conf`

```
worker_processes 8
timeout 120
working_directory "/etc/puppet"
listen '/etc/puppet/unicorn.sock', :backlog => 512
pid "/etc/puppet/unicorn.pid"

stderr_path "/var/log/puppet/unicorn.stderr.log"
stdout_path "/var/log/puppet/unicorn.stdout.log"

preload_app true
if GC.respond_to?(:copy_on_write_friendly=)
  GC.copy_on_write_friendly = true
end

before_fork do |server, worker|
  old_pid = "#{server.config[:pid]}.oldbin"
  if File.exists?(old_pid) && server.pid != old_pid
  begin
    Process.kill("QUIT", File.read(old_pid).to_i)
  rescue Errno::ENOENT, Errno::ESRCH
    # someone else did our job for us
  end
  end
end
```

Update our **.gemrc** to ensure we don't install documentation on our servers.

```
gem: --no-ri --no-rdoc"
```

Now lets install unicorn: `sudo gem install unicorn`

Finally write the startup script for the puppetmaster environment: `sudo vim /etc/init.d/puppetmaster`

```
#!/bin/sh
set -u
set -e

# Feel free to change any of the following variables for your app:
APP_ROOT=/etc/puppet
PID=/etc/puppet/unicorn.pid
CMD="/usr/local/bin/unicorn -c /etc/puppet/unicorn.conf -D"

old_pid="$PID.oldbin"

cd $APP_ROOT || exit 1

sig () {
  test -s "$PID" && kill -$1 `cat $PID`
}

oldsig () {
  test -s $old_pid && kill -$1 `cat $old_pid`
}

workersig () {
  workerpid="/etc/puppet/unicorn.$2.pid"
  test -s "$workerpid" && kill -$1 `cat $workerpid`
}

case $1 in
start)
  sig 0 && echo >&2 "Already running" && exit 0
  $CMD
  ;;
stop)
  sig QUIT && exit 0
  echo >&2 "Not running"
  ;;
force-stop)
  sig TERM && exit 0
  echo >&2 "Not running"
  ;;
restart|reload)
  sig HUP && echo reloaded OK && exit 0
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  $CMD
  ;;
upgrade)
  sig USR2 && exit 0
  echo >&2 "Couldn't upgrade, starting '$CMD' instead"
  $CMD
  ;;
kill_worker)
  workersig QUIT $2 && exit 0
  echo >&2 "Worker not running"
  ;;
rotate)
  sig USR1 && echo rotated logs OK && exit 0
  echo >&2 "Couldn't rotate logs" && exit 1
  ;;
*)
  echo >&2 "Usage: $0 \<start|stop|restart|upgrade|rotate|force-stop>"
  exit 1
  ;;
esac
```

Ensure that it is executable: `sudo chmod +x /etc/init.d/puppetmaster`

### Bootstrap Puppet SSL Files
Now that we have a shared SSL directory, a virtual IP that can float between machines, and puppet configured on both servers, we can write our SSL config files and start up the puppet server in the foreground. Note: the puppet server will not be listening on a reachable interface, it uses sockets. That will be fixed in the next step by installing nginx. You should see output similar to what is below:

```
sudo puppet master --no-daemonize --verbose
info: Creating a new SSL key for ca
info: Creating a new SSL certificate request for ca
info: Certificate Request fingerprint (md5): 11:38:9F:F8:96:AC:37:85:A6:A5:3A:D5:E7:A0:8B:33
notice: Signed certificate request for ca
notice: Rebuilding inventory file
info: Creating a new certificate revocation list
info: Creating a new SSL key for puppet.domain.tld
info: Creating a new SSL certificate request for puppet.domain.tld
info: Certificate Request fingerprint (md5): 9C:EA:47:50:54:4D:C8:17:AD:7E:2F:02:BE:A4:52:16
notice: puppet.domain.tld has a waiting certificate request
notice: Signed certificate request for puppet.domain.tld
notice: Removing file Puppet::SSL::CertificateRequest puppet.domain.tld at '/var/lib/puppet/ssl/ca/requests/puppet.domain.tld.pem'
notice: Removing file Puppet::SSL::CertificateRequest puppet.domain.tld at '/var/lib/puppet/ssl/certificate_requests/puppet.domain.tld.pem'
notice: Starting Puppet master version 2.7.14
```

You can now close the server by hitting `Ctrl-C`, now we will try and start it using Unicorn: `sudo /etc/init.d/puppetmaster start`

There should be no output from this comment, you can verify that it is running by running: `ps -ef | grep unicorn`

```
puppet    7828     1  1 12:25 ?        00:00:00 unicorn master -c /etc/puppet/unicorn.conf -D
puppet    7849  7828  0 12:25 ?        00:00:00 unicorn worker[0] -c /etc/puppet/unicorn.conf -D
puppet    7850  7828  0 12:25 ?        00:00:00 unicorn worker[1] -c /etc/puppet/unicorn.conf -D
puppet    7851  7828  0 12:25 ?        00:00:00 unicorn worker[2] -c /etc/puppet/unicorn.conf -D
puppet    7852  7828  0 12:25 ?        00:00:00 unicorn worker[3] -c /etc/puppet/unicorn.conf -D
puppet    7853  7828  0 12:25 ?        00:00:00 unicorn worker[4] -c /etc/puppet/unicorn.conf -D
puppet    7854  7828  0 12:25 ?        00:00:00 unicorn worker[5] -c /etc/puppet/unicorn.conf -D
puppet    7855  7828  0 12:25 ?        00:00:00 unicorn worker[6] -c /etc/puppet/unicorn.conf -D
puppet    7856  7828  0 12:25 ?        00:00:00 unicorn worker[7] -c /etc/puppet/unicorn.conf -D
```

Now it is time to move onto nginx.

## Nginx
Nginx is a very optimized and secure web server. We will use this to load balance connections coming in to each of the 8 Unicorn processes. To install nginx run the following: `sudo apt-get install nginx-full`

### Configuration
Now update the configuration to point at puppet. There will be two files we need to update in order for this to happen.

Write a new nginx.conf to optimize it for load balancing to puppet. `sudo vim /etc/nginx/nginx.conf`

```
user www-data;
worker_processes  4;

error_log  /var/log/nginx/error.log;
pid        /var/run/nginx.pid;

events {
  worker_connections  4096;
  multi_accept on;
}

http {
  include       /etc/nginx/mime.types;
  access_log     /var/log/nginx/access.log;

  sendfile        on;
  #tcp_nopush     on;

  #keepalive_timeout  0;
  keepalive_timeout  65;
  tcp_nodelay        on;

  gzip  on;
  gzip_disable "MSIE [1-6]\.(?!.*SV1)";

  include /etc/nginx/conf.d/*.conf;
  include /etc/nginx/sites-enabled/*;
}
```

#### puppet-1
Now create the puppet configuration to point at each UNIX socket: `sudo vim /etc/nginx/conf.d/puppet.conf`

```
upstream puppetmaster {
  server unix:/etc/puppet/unicorn.sock fail_timeout=0;
}

server {
  listen 192.168.1.11:8140;

  ssl on;
  ssl_session_timeout 5m;
  ssl_certificate /var/lib/puppet/ssl/certs/puppet.domain.tld.pem;
  ssl_certificate_key /var/lib/puppet/ssl/private_keys/puppet.domain.tld.pem;
  ssl_client_certificate /var/lib/puppet/ssl/ca/ca_crt.pem;
  ssl_ciphers ALL;
  ssl_protocols SSLv3 TLSv1;
  ssl_verify_client optional;

  root /usr/share/empty;

  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Client-Verify $ssl_client_verify;
  proxy_set_header X-Client-DN $ssl_client_s_dn;
  proxy_set_header X-SSL-Issuer $ssl_client_i_dn;
  proxy_read_timeout 120;

  location / {
    proxy_pass http://puppetmaster;
    proxy_redirect off;
  }
}
```

#### puppet-2
Now create the puppet configuration to point at each UNIX socket: `sudo vim /etc/nginx/conf.d/puppet.conf`

```
upstream puppetmaster {
  server unix:/etc/puppet/unicorn.sock fail_timeout=0;
}

server {
  listen 192.168.1.12:8140;

  ssl on;
  ssl_session_timeout 5m;
  ssl_certificate /var/lib/puppet/ssl/certs/puppet.domain.tld.pem;
  ssl_certificate_key /var/lib/puppet/ssl/private_keys/puppet.domain.tld.pem;
  ssl_client_certificate /var/lib/puppet/ssl/ca/ca_crt.pem;
  ssl_ciphers ALL;
  ssl_protocols SSLv3 TLSv1;
  ssl_verify_client optional;

  root /usr/share/empty;

  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Client-Verify $ssl_client_verify;
  proxy_set_header X-Client-DN $ssl_client_s_dn;
  proxy_set_header X-SSL-Issuer $ssl_client_i_dn;
  proxy_read_timeout 120;

  location / {
  proxy_pass http://puppetmaster;
  proxy_redirect off;
  }
}
```

### Testing
Now we can test the nginx frontend for puppet: `sudo /etc/init.d/nginx start`

Verify that Nginx is online: `ps -ef | grep nginx`

```
root      8256     1  0 12:33 ?        00:00:00 nginx: master process /usr/sbinnginx
www-data  8259  8256  0 12:33 ?        00:00:00 nginx: worker process
www-data  8260  8256  0 12:33 ?        00:00:00 nginx: worker process
www-data  8261  8256  0 12:33 ?        00:00:00 nginx: worker process
www-data  8262  8256  0 12:33 ?        00:00:00 nginx: worker process
```

Check that we are listening on the right port: `sudo netstat -plant | grep 8140`

```
tcp        0      0 192.168.1.11:8140      0.0.0.0:*               LISTEN      8256/nginx
```

## Test Failover
Testing failover is actually quite simple in this setup. If you restart the networking on the primary node it will fail the services over to the secondard node.

*On puppet-1:*

```
sudo /etc/init.d/networking restart
ifconfig -a
```

At this point there should be no `eth0:ucarp` interface listed.
Now run the following on the secondary node and we should be good to go.

*On puppet-2 run:* `ifconfig -a`

```
eth0:ucarp Link encap:Ethernet  HWaddr 00:50:56:b3:6c:7b
    inet addr:192.168.1.10  Bcast:192.168.1.255  Mask:255.255.255.0
    UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
```

*On puppet-2 run:* `ps -ef | grep {haproxy,nginx,unicorn}`

```
haproxy   8325     1  0 12:36 ?        00:00:00 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg
```

Lastly we want to ensure that we still have a consistent drbd0 block device: `sudo drbd-overview`

```
0:r0  Connected Primary/Secondary UpToDate/UpToDate C r----- /var/lib/puppet/ssl ext4 2.0G 64M 1.9G 4%
```

## Update /etc/services
Now that we have all the services online we should update our `/etc/services` file to keep track of the new ports on the sevrer.

```
# Local services
puppetmaster  8140/tcp
puppet        8139/tcp
```

## Puppet Manifest Deployment
For manifest deployment we will use the Rakefile on the development machines to push the code to both nodes. Each server will have a read only git connection to the repo that they will use to get the latest versions of the manifests and modules.

We need the administrator to be in the same group as the puppet group. This should be done on both puppet servers: `sudo usermod -G puppet,sudo administrator`

### SSH Key Setup
Now that we have two nodes, and that either one may answer on the `puppet` hostname, we can share the SSH keys that the service uses to eliminate any confusion/errors later on.

```
ssh administrator@puppet-1
sudo scp /etc/ssh/ssh_host* administrator@puppet-2:
exit
```

```
ssh administrator@puppet-2
sudo mv ssh_host* /etc/ssh/
sudo service ssh restart
exit
```
