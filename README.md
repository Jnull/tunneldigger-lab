# tunneldigger-lab
experiments on digging tunnels 

Why tunneldigger? See https://wlan-si.net/en/blog/2012/10/29/tunneldigger-the-new-vpn-solution/ .

tested on ubuntu 16.04 LTS

# prerequisites

```
sudo apt update
sudo apt install cmake libnl-3-dev libnl-genl-3-dev build-essential pkg-config
sudo apt install linux-image-extra-$(uname -r)
```

# install
## kernel modules
You have to load some kernel modules l2tp_*

```
sudo modprobe l2tp_netlink
sudo modprobe l2tp_eth
sudo modprobe l2tp_core
```

Verify that the modules where loaded by running ```sudo lsmod | grep l2tp```, result should be something like:

```
$ sudo lsmod | grep l2tp
l2tp_eth               16384  0
l2tp_ppp               24576  0
l2tp_netlink           20480  2 l2tp_eth,l2tp_ppp
l2tp_core              32768  3 l2tp_eth,l2tp_ppp,l2tp_netlink
ip6_udp_tunnel         16384  1 l2tp_core
udp_tunnel             16384  1 l2tp_core
pppox                  16384  2 l2tp_ppp,pppoe
```

If you'd like to automatically load the kernel modules on reboot, ```The system should be configured to load these modules at boot which is usually done by listing the modules in /etc/modules.``` For more information see http://tunneldigger.readthedocs.io/en/latest/server.html .

## clone
First clone and build the tunneldigger client

```
git clone https://github.com/wlanslovenija/tunneldigger.git
```

The version that is used in [firmware](https://github.com/sudomesh/sudowrt-firmware) can be found at https://github.com/sudomesh/nodewatcher-firmware-packages/blob/sudomesh/net/tunneldigger/Makefile . At time of writing https://github.com/sudomesh/tunneldigger was used, a fork of https://github.com/wlanslovenija/tunneldigger . The sudomesh fork does not run on ubuntu because of some library depedencies. 

## compile
```
cd tunneldigger/client
cmake .
```
cmake may provide an output like:
```
-- Checking for module 'libasyncns'
--   No package 'libasyncns' found
-- Configuring done
-- Generating done
-- Build files have been written to: /home/user/tunneldigger/client
```
do not worry about the missing package, the libasyncns source is included in the tunneldigger repository, so it does not need to be installed globally.  
now you can run make, 
```
make 
```
which should produce and output like:
```
Scanning dependencies of target tunneldigger
[ 33%] Building C object CMakeFiles/tunneldigger.dir/l2tp_client.c.o
[ 66%] Building C object CMakeFiles/tunneldigger.dir/libasyncns/asyncns.c.o
[100%] Linking C executable tunneldigger
[100%] Built target tunneldigger
```

and the file [tunneldigger-lib]/tunneldigger/client/tunneldigger should exist.

# digging a tunnel
Before digging a tunnel, check interfaces using ```ip addr```, there should be no l2tp interface yet. Check udp ports using ```netstat -u```, this should be empty. Check syslog using ```cat /var/log/syslog | grep td-client```, this should not contain any recent entries. 

First, generate a uuid using ```uuidgen``` on the commandline: the output should be a valid [uuid](https://en.wikipedia.org/wiki/Universally_unique_identifier) .

Now run 
```sudo $PWD/tunneldigger/client/tunneldigger -b exit.sudomesh.org:8942 -u [uuid] -i l2tp0 -s $PWD/tunnel_hook.sh```

where:

1. exit.sudomesh.org:8942 is the end of the tunnel you are attempting to dig also known as the "broker"
2. [uuid] is the uuid you just generated with ```uuidgen```
3. l2tp0 is the interface that will be created for the tunnel
4. tunnel_hook.sh is the shell script (aka "hook") that is called by the tunnel digger on creating/destroying a session.

Now, open another terminal and check the status of the tunnel by:

1. inspecting the tunnel_hook.sh.log for recent entries of new sessions. Expected entries are like
```
Mon Dec 18 21:29:28 PST 2017 [td-hook] session.up l2tp0
Mon Dec 18 21:30:10 PST 2017 [td-hook] session.down l2tp0
```
2. run ```ip addr``` and verify that an interface ```l2tp0``` now exists. 
3. also, open udp ports ```netstat -u``` and verify you see something like this:
```
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
udp        0      0 xxxx:42862         unassigned.psychz.:8942 ESTABLISHED
```
4. verify syslog entries using ```cat /var/log/syslog | grep td-client``` - expecting something like:
```
Dec 17 13:24:06 xx td-client: Performing broker selection...
Dec 17 13:24:08 xx td-client: Broker usage of exit.sudomesh.org:8942: 1471
Dec 17 13:24:08 xx td-client: Selected exit.sudomesh.org:8942 as the best broker.
Dec 17 13:24:12 xx td-client: Tunnel successfully established.
Dec 17 13:24:21 xx td-client: Setting MTU to 1446
```
5. the tunnel can be closed using CRTL-C in the original, or can be run in the background like any shell command.

## setting up a broker 

It is also possible to set up your own broker within the client machine or on a hosted server (such as on digitalocean). You can follow the instructions below on how to do this, adapted from http://tunneldigger.readthedocs.io/en/latest/server.html . Note that these instructions use the latest tunneldigger broker. See below for the (older) broker version that currently runs on sudomesh exist node.  
```
sudo apt update
sudo apt install iproute bridge-utils libnetfilter-conntrack-dev libnfnetlink-dev libffi-dev python-dev libevent-dev ebtables python-virtualenv
mkdir /srv/tunneldigger
cd /srv/tunneldigger
virtualenv env_tunneldigger
git clone https://github.com/wlanslovenija/tunneldigger.git
source env_tunneldigger/bin/activate
cd tunneldigger/broker
python setup.py install
cp l2tp_broker.cfg.example l2tp_broker.cfg
```

If you are running your broker on an external server, you will need to edit the `l2tp_broker.cfg` file. Change the file by replacing the loopback address and interface with the IP and name of the public interface. It should look like so,   

```
[broker]
; IP address the broker will listen and accept tunnels on
address=<public_ip_of_broker>
; Ports where the broker will listen on
port=53,123,8942
; Interface with that IP address
interface=eth0
```

On starting the broker with default configuration, you should see something like:  

```
$ sudo /srv/tunneldigger/env_tunneldigger/bin/python -m tunneldigger_broker.main /srv/tunneldigger/tunneldigger/broker/l2tp_broker.cfg
[INFO/tunneldigger.broker] Initializing the tunneldigger broker.
[INFO/tunneldigger.broker] Maximum number of tunnels is 1024.
[INFO/tunneldigger.broker] Tunnel identifier base is 100.
[INFO/tunneldigger.broker] Tunnel port base is 20000.
[INFO/tunneldigger.broker] Namespace is default.
[INFO/tunneldigger.broker] Listening on 127.0.0.1:53.
[INFO/tunneldigger.broker] Listening on 127.0.0.1:123.
[INFO/tunneldigger.broker] Listening on 127.0.0.1:8942.
[INFO/tunneldigger.broker] Broker initialized.
```

Now, repeat [digging a tunnel](#digging-a-tunnel) using broker config localhost:8942 .   

Or if you are testing an external broker, point your client computer at <public_ip_of_broker>:8942.  

Now, you should see the following in the broker log:  

```
[INFO/tunneldigger.broker] Creating tunnel (07105c7f-681f-4476-b5aa-5146c6e579de) with id 100.
[INFO/tunneldigger.tunnel] Set tunnel 100 MTU to 1446.
```

on closing the client, the following is logged:

```
[INFO/tunneldigger.tunnel] Closing tunnel 100 after 42 seconds
```

## digging a tunnel to the current production broker
The broker running on the sudomesh exit node (5 Feb 2018) is reportedly github.com/sudomesh/tunneldigger/blob/f05d9adc170929f883600c3637b66b9c60705630/ with install instructions at https://github.com/sudomesh/exitnode/blob/master/provision.sh#L72 .

Here's some steps on how to get it running on ubuntu 16.04:

```
pip install netfilter
pip install virtualenv

git clone git@github.com:sudomesh/tunneldigger [tunneldigger_dir]
cd [tunneldigger_dir]

# copy some default config
cp broker/l2tp_broker.cfg.example broker/l2tp_broker.cfg

virtualenv broker/env_tunneldigger

# install dependencies
broker/env_tunneldigger/bin/pip install -r broker/requirements.txt

sudo broker/env_tunneldigger/bin/python -m broker.main broker/l2tp_broker.cfg
```

with resulting output something like:

```

$sudo broker/env_tunneldigger/bin/python -m broker.main broker/l2tp_broker.cfg
[INFO/tunneldigger.broker] Initializing the tunneldigger broker.
[INFO/tunneldigger.broker] Maximum number of tunnels is 1024.
[INFO/tunneldigger.broker] Tunnel identifier base is 100.
[INFO/tunneldigger.broker] Tunnel port base is 20000.
[INFO/tunneldigger.broker] Namespace is default.
[INFO/tunneldigger.broker] Listening on 127.0.0.1:53.
[INFO/tunneldigger.broker] Listening on 127.0.0.1:123.
[INFO/tunneldigger.broker] Listening on 127.0.0.1:8942.
[INFO/tunneldigger.broker] Broker initialized.
```



sudo broker/env_tunneldigger/bin/python -m broker.main broker/l2tp_broker.cfg


