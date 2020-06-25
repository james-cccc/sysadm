---
date: 2020-06-14T18:00:00
description: "Perform a disconnected Openshift 4 UPI install on ESXi hosted on Packet using static IPs"
#featured_image: "/images/keycloak_logo.png"
tags: ["Openshift","VMware"]
# categories: "POC"
title: "Installing Openshift 4 on ESXi using Packet"
comment : false
---

This article aims to present the steps required to create the minimum necessary infrastructure to install and gain familiarity with Openshift 4.4 on VMware ESXi (hosted on packet.com) using static IP addresses in a disconnected environment... so a bit more than the standard run of the mill install!

So why did I choose this setup? Well, I chose to use Packet as they provide easily provisionable bare metal in the cloud that is reasonably priced and I'm hoping that if someone wanted to replicate this setup they could easily do so following the below steps. This activity may potentially be a decent exercise to become somewhat familiar and comfortable performing a UPI (that's User Provisioned Infrastructure) installation of Openshift. I additionally chose to do a disconnected installation using static IPs to add to the fun and give you the reader a wider view on the installation process (plus I have a gig coming up where I'm about to do just that... so there is also that reason)!

For a bit of background, I'm very much new to Openshift myself. Previous to starting this I did a couple of very boring AWS IPI (installer provisioned infrastructure) installations... which were very much 'click button' then make a brew whilst you wait for it to come up. That was nice, but I didn't really learn a lot. I then performed a UPI install on Openstack. So really, this is my second proper UPI install. Hopefully, the below ramblings will be useful to others in the same position as me.

Before continuing, I suggest you quickly read the Packet guide on using ESXi on Packet.

https://www.packet.com/resources/guides/esxi/

### High Level Overview

This article will cover the below steps in order to build an Openshift cluster from nothing:

1. Provision ESXi
2. Create a Bastion VM
3. Setup a container registry
4. Setup an Apache webserver
5. Setup DNS
6. Setup HAProxy
7. Create ignition files
8. Create a boot iso
9. Create bootstrap and cluster VMs
10. Check bootstrap process and approve CSRs.

# Packet Preamble

In total it took me 4 days to get this process together and get it working. I was provisioning a new server each day starting fresh and then throwing it away at the end of the day. I estimate that with the below information someone could run through this process in about a day. The total cost I incurred was $19.93, and I actually found a voucher 'nixos' for $25 Packet credit, so effectively it didn't cost me anything! I initially used the 'c1.small.x86' instances at $0.40 per hour to understand how packet and Esxi work, I then used the 'c2.medium.x86' instances at $1.00 per hour to perform the actual Openshift installation. You may think that the 'c2.medium.x86' instance is a bit lacking for an Openshift install with its measly 64GB of RAM.. technically it is, however you can get away overcommitting the VM guest machines memory, at least for the installation process anyway. Here was my usage for the week:

![Shiny new server](/images/finished14.png#center)

### Provision ESXi

Although not mandatory, I suggest adding your ssh public key to your packet account before provisioning any server. Choose a suitable instance such as the c2.medium.x86. Set it's OS to be VMware ESXi 6.5. In the optional settings select Configure IPs and select the /28 subnet in the Private IPv4 section. The provisioning process takes 5 minutes or so, once it has completed you can click the instance name in your list of servers to see its details.

![Shiny new server](/images/ocp_finished10.png#center)

### Networking

Post provisiong your server on the overview page you will see a table of its network configuration that will look like mine below.

| ADDRESS                | NETWORK                   | GATEWAY               | TYPE         |
|------------------------|---------------------------|-----------------------|--------------|
| 139.178.72.18          | 139.178.72.16/29          | 139.178.72.17         | Public IPv4  |
| 2604:1380:2001:2300::1 | 2604:1380:2001:2300::/127 | 2604:1380:2001:2300:: | Public IPv6  |
| 10.80.158.2            | 10.80.158.0/28            | 10.80.158.1           | Private IPv4 |

Using this table we can define our free IP addresses. Before continuing I recommend making a quick table of your network configuration. This online tool here can help you work our your IPs if you are unsure: http://magic-cookie.co.uk/iplist.html

#### Public

For our public network we have a CIDR of /29, so its netmask is 255.255.255.248. Using the network address and CIDR we can work out the available IPs that we have. Therefore, with 139.178.72.16/29 we have 8 addresses in the range of: 139.178.72.16-139.178.72.23 (again as mentioned in the Packet ESXi guide 4 of these addresses are used). We will use the first available IP 139.178.72.19 to be our bastion VM.

| ADDRESS       | USEAGE    |
|---------------|-----------|
| 139.178.72.16 | Network   |
| 139.178.72.17 | Gateway   |
| 139.178.72.18 | ESXi      |
| 139.178.72.19 | Bastion   |
| 139.178.72.20 | Available |
| 139.178.72.21 | Available |
| 139.178.72.22 | Available |
| 139.178.72.23 | Broadcast |

#### Private

For our private network we have a CIDR of /28, so its netmask is 255.255.255.240. Using the network address and CIDR we can work out the available IPs that we have. Therefore, with 10.80.158.0/28 we have 16 addresses in the range of: 10.80.158.0-10.80.158.15 (as mentioned in the Packet ESXi guide 4 of these addresses are used). I've defined the IP addresses here I'm planning to use for each node.

| ADDRESS       | USEAGE                |
|---------------|-----------------------|
| 10.80.158.0   | Network               |
| 10.80.158.1   | Gateway               |
| 10.80.158.2   | ESXi                  |
| 10.80.158.3   | Available             |
| 10.80.158.4   | Available             |
| 10.80.158.5   | packetfeathersbastion |
| 10.80.158.6   | bootstrap.ocp4        |
| 10.80.158.7   | master0.ocp4          |
| 10.80.158.8   | master1.ocp4          |
| 10.80.158.9   | master2.ocp4          |
| 10.80.158.10  | worker0.ocp4          |
| 10.80.158.11  | worker1.ocp4          |
| 10.80.158.12  | worker2.ocp4          |
| 10.80.158.13  | Available             |
| 10.80.158.14  | Available             |
| 10.80.158.15  | Broadcast             |








# Setup Bastion/Installer server

### Stage bastion installation media

Grab the URL of the latest CentOS iso, ssh to your ESXi server then pull the iso down into the below directory. If you had previously added your ssh key to your packet account you'll get straight in, if not you'll have to use the randomly generated root password Packet sets which you can find by clicking on your server in the Packet console.

```text
ssh root@139.178.72.18
mkdir /vmfs/volumes/datastore1/iso
cd /vmfs/volumes/datastore1/iso
wget http://mirror.as29550.net/mirror.centos.org/8.1.1911/isos/x86_64/CentOS-8.1.1911-x86_64-dvd1.iso
```

### Create the bastion VM

Log into your ESXi web console which will be something like: https://139.178.72.18/ui/#/login

Create a VM with 2 NICs, 4GB RAM, 40GB HDD, 2 VPCUs and mount the Centos8 iso to the DVD drive. Power the VM on and progress with the installation process using the console. When you get to the configuration screen the only important section is the Network one. Configure your hostname and domain to be something like 'packetfeathersbastion.jftest.com'.

Configure the first NIC manually to be your public network:

| FUNCTION      | ADDRESS                |
|---------------|------------------------|
| IP            | 139.178.72.19          |
| Netmask       | 255.255.255.248        |
| Gateway       | 139.178.72.17          |
| DNS           | 139.178.72.19, 8.8.8.8 |

Configure the second NIC manually to be your private network:

| FUNCTION      | ADDRESS                |
|---------------|------------------------|
| IP            | 10.80.158.5            |
| Netmask       | 255.255.255.240        |
| Gateway       | 10.80.158.1            |
| DNS           | 10.80.158.5            |

#### Base Configuration

When the VM is up we can now ssh to its public address. I would recommend to copy your ssh public key to the server and disable password authentication (not necessary but good practice).
```text
ssh-copy-id james@139.178.72.19
ssh james@139.178.72.19
vim /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

Check that the network configuration is okay and you can reach both public and private addresses. I found that the second device didn't start by default so I had to enable that.
```text
nmcli device status
sudo nmcli device connect ens34
nmcli connection show
sudo nmcli device modify ens34 autoconnect yes
sudo nmcli con mod ens34 connection.autoconnect yes
ip addr show
ping 10.80.158.2
ping 139.178.72.18
cat /etc/sysconfig/network-scripts/*
```

Install the necessary packages.
```text
sudo yum install epel-release -y
sudo yum install httpd httpd-tools podman-docker.noarch podman bind haproxy jq -y
```

#### Stage Openshift installation media

On the bastion VM stage the below media and add the CLI tools into a directory in your PATH environment variable for ease of use.
```text
cd ~
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/latest/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/latest/rhcos-4.4.3-x86_64-installer.x86_64.iso
tar -zxvf openshift-install-linux.tar.gz
tar -zxvf openshift-client-linux.tar.gz
sudo mv oc kubectl openshift-install /usr/local/sbin/
sudo chmod 775 /usr/local/sbin/*
```




# Setup Container Registry

The steps below on setting up the container registry were pretty much just taken from the official documentation. If you get stuck at all, that's probably the best bet. We'll setup the container registry on our Bastion/Installer VM.

https://docs.openshift.com/container-platform/4.4/installing/install_config/installing-restricted-networks-preparations.html

Create the directories to host the relevant content, create an SSL certificate (make sure to set the common name properly e.g. **packetfeathersbastion.jftest.com** ) and create a flat file with a username e.g. *james* and a  password e.g. *fugazi77*.
```text
sudo mkdir -p /opt/registry/{auth,certs,data}
sudo chown -R $USER /opt/registry
cd /opt/registry/certs
hostnamectl
openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt
htpasswd -bBc /opt/registry/auth/htpasswd james fugazi77
```

You should now be able to start the container registry with the below command.
```text
podman run -d --name mirror-registry \
   -p 5000:5000 --restart=always \
   -v /opt/registry/data:/var/lib/registry:z \
   -v /opt/registry/auth:/auth:z \
   -e "REGISTRY_AUTH=htpasswd" \
   -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
   -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
   -v /opt/registry/certs:/certs:z \
   -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
   -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
   docker.io/library/registry:2
```

Test the connection to the registry.
```text
curl -u james:fugazi77 -k https://packetfeathersbastion.jftest.com:5000/v2/_catalog
```

Add firewall exceptions and add the self-signed certificate to your list of trusted certificates:
```text
sudo firewall-cmd --add-port=5000/tcp --zone=internal --permanent
sudo firewall-cmd --add-port=5000/tcp --zone=public   --permanent
sudo firewall-cmd --reload
sudo cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
sudo update-ca-trust
```

Test that you can pull down an image (e.g. the ubi image), tag it and then push it to your new registry.
```text
podman pull ubi7/ubi:7.7
podman run ubi7/ubi:7.7 cat /etc/os-release
podman login -u james -p fugazi77 packetfeathersbastion.jftest.com:5000
podman tag registry.access.redhat.com/ubi7/ubi:7.7 packetfeathersbastion.jftest.com:5000/ubi7/ubi:7.7
podman push packetfeathersbastion.jftest.com:5000/ubi7/ubi:7.7
ls /opt/registry/data/docker/registry/v2/repositories
curl -u james:fugazi77 https://packetfeathersbastion.jftest.com:5000/v2/_catalog
```

### Prepare Pull Secret

Obtain your pull secret from https://cloud.redhat.com/openshift/ and then place it here: ```/tmp/pull-secret.text```. Then you can use jq to tidy it up so you can do the next step of adding in a new section about your mirror registry.
```text
vim /tmp/pull-secret.text
cat  /tmp/pull-secret.text | jq .  > /tmp/pull-secret.json
```

Generate the base64-encoded user name and password or token for your mirror registry e.g.
```text
echo -n 'james:fugazi77' | base64 -w0
```

Add a section like the below into the auths section of ```/tmp/pull-secret.json``` but replace your auth key with the one that was the output of the previous command.
```json
"packetfeathersbastion.jftest.com:5000": {
"auth": "amFtZXM6ZnVnYXppNzc="
},
```
The full auth file can also be generated with the below command, but be careful and don't go full-on copypasta as you don't need all the contents of the file.
```text
podman login -u james -p fugazi77 --authfile /tmp/pullsecret_config.json packetfeathersbastion.jftest.com:5000
```

Now you have a new auth file, make an 'ugly' copy for later.
```text
cat /tmp/pull-secret.json | jq -c . /tmp/pull-secret.json >> /tmp/pull-secret-ugly.json
```

### Mirror the OCP Registry

Now that we have a container registry up and running and a pull secret containing authentication for both the official and your registry you should now be ready to mirror the registry for the release that you want.

Depending on your release you may wish to edit these variable values:
```text
export OCP_RELEASE='4.4.6-x86_64'
export LOCAL_REGISTRY='packetfeathersbastion.jftest.com:5000'
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON='/tmp/pull-secret.json'
export RELEASE_NAME="ocp-release"
```

You can view release information with the below command.
```text
oc adm release info -a /tmp/pull-secret.json "quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}" | head -n 18
```

You can now mirror all the images required for the release you are intending to install.
```text
oc adm -a ${LOCAL_SECRET_JSON} release mirror \
    --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
    --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
    --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}
```

Once the above command has completed make a note of the output it gives. Particularly the end section regarding your registry. We will use this later.

```yaml
imageContentSources:
- mirrors:
  - packetfeathersbastion.jftest.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - packetfeathersbastion.jftest.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```







# Setup Openshift Prerequisites

Before even thinking about starting the Openshift installation you have to nail the infrastructure prereqs. DNS and a load balancer are a must, and for our installation, as we're partly performing the bare-metal installation process as well as the VMware installation process we'll need a web server as well.

## Setup an Apache Webserver

The Openshift install procedure here requires hosting the ignition files and the compressed metal RAW image. Set the server to listen on just the internal IP address on port 8080 and then start the service.
```text
sudo semanage port -a -t http_port_t -p tcp 8080
sudo sed -i 's/Listen 80/Listen 10.80.158.5:8080/' /etc/httpd/conf/httpd.conf
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo systemctl enable --now httpd
```

Copy the CoreOS compressed metal RAW image into place on the webserver and set the relevant permissions.
```text
cp ~/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz /var/www/html/
chmod 644 /var/www/html/*
restorecon -R -v /var/www/html
```

The only other files that the webserver will host is the ignition files which we will generate later.

##  Setup DNS

First, create a backup of the default file (in case you make a mess) and then go ahead and amend the default file ```/etc/named.conf``` with something like the below example. Then create two additional files in the ```/var/named``` directory, one for the A and SRV records and the other for the reverse records, again something like the below examples ```/var/named/jftest.com``` and ```/var/named/10.80.158.db```

```text
sudo cp -p /etc/named.conf /etc/named.conf.vanilla
sudo vim /etc/named.conf
sudo vim /var/named/jftest.com
sudo vim /var/named/10.80.158.db
```

#### /etc/named.conf
```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for jftest named configuration files.
//

options {

#	listen-on port 53 { any; };
#	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	secroots-file	"/var/named/data/named.secroots";
	recursing-file	"/var/named/data/named.recursing";
	allow-query     { any; };


	/*
	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
	 - If you are building a RECURSIVE (caching) DNS server, you need to enable
	   recursion.
	 - If your recursive DNS server has a public IP address, you MUST enable access
	   control to limit queries to your legitimate users. Failing to do so will
	   cause your server to become part of large scale DNS amplification
	   attacks. Implementing BCP38 within your network would greatly
	   reduce such attack surface
	*/
	recursion yes;

	/* Fowarders */
	forward only;
	forwarders { 8.8.8.8; 8.8.4.4; };

	dnssec-enable yes;
	dnssec-validation yes;

	managed-keys-directory "/var/named/dynamic";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";

	/* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
	include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
	type hint;
	file "named.ca";
};



zone "jftest.com" IN {
	type master;
	file	"jftest.com";
};

zone "158.80.10.in-addr.arpa" IN {
	type	master;
	file	"10.80.158.db";
};


include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```


#### /var/named/jftest.com
```
$TTL 1W
@	IN	SOA	ns1.jftest.com.	root (
			2019052300	; serial
			3H		; refresh (3 hours)
			30M		; retry (30 minutes)
			2W		; expiry (2 weeks)
			1W )		; minimum (1 week)
	IN	NS	ns1.jftest.com.
	IN	MX 10	smtp.jftest.com.
;
;
ns1	IN	A	10.80.158.5
smtp	IN	A       10.80.158.5
smtp	IN	A       10.80.158.5
esxi	IN	A	10.80.158.2
packetfeathersbastion   IN      A       10.80.158.5

;
; The api points to the IP of your load balancer
api.ocp4	IN	A       10.80.158.5
api-int.ocp4	IN	A       10.80.158.5
;
; The wildcard also points to the load balancer
*.apps.ocp4	IN	A       10.80.158.5
;
; Create entry for the bootstrap host
bootstrap.ocp4	IN	A       10.80.158.6
;
; Create entries for the master hosts
master0.ocp4	IN	A	10.80.158.7
master1.ocp4	IN	A       10.80.158.8
master2.ocp4	IN	A       10.80.158.9
;
; Create entries for the worker hosts
worker0.ocp4	IN	A       10.80.158.10
worker1.ocp4	IN	A       10.80.158.11
worker2.ocp4	IN	A       10.80.158.12
;
; The ETCd cluster lives on the masters...so point these to the IP of the masters
etcd-0.ocp4	IN	A       10.80.158.7
etcd-1.ocp4	IN	A       10.80.158.8
etcd-2.ocp4	IN	A	10.80.158.9
;
; The SRV records are IMPORTANT....make sure you get these right...note the trailing dot at the end...
_etcd-server-ssl._tcp.ocp4	IN	SRV	0 10 2380 etcd-0.ocp4.jftest.com.
_etcd-server-ssl._tcp.ocp4	IN	SRV	0 10 2380 etcd-1.ocp4.jftest.com.
_etcd-server-ssl._tcp.ocp4	IN	SRV	0 10 2380 etcd-2.ocp4.jftest.com.
;
;EOF
```

#### /var/named/10.80.158.db
```
$TTL 1W
@	IN	SOA	ns1.jftest.com.	root (
			2019052300	; serial
			3H		; refresh (3 hours)
			30M		; retry (30 minutes)
			2W		; expiry (2 weeks)
			1W )		; minimum (1 week)
	IN	NS	ns1.jftest.com.
;
; syntax is "last octet" and the host must have fqdn with trailing dot
7	IN	PTR	master0.ocp4.jftest.com.
8	IN	PTR	master1.ocp4.jftest.com.
9	IN	PTR	master2.ocp4.jftest.com.
;
6	IN	PTR	bootstrap.ocp4.jftest.com.
;
5	IN	PTR	api.ocp4.jftest.com.
5	IN	PTR	api-int.ocp4.jftest.com.
;
10	IN	PTR	worker0.ocp4.jftest.com.
11	IN	PTR	worker1.ocp4.jftest.com.
12	IN	PTR	worker2.ocp4.jftest.com.
;
;
;EOF
```

### Start DNS

Add the relevant firewall rules, make selinux happy and then start the DNS service. Bonus points if it starts straight away with no errors! If you do get errors, go back and check what you've done.
```text
sudo firewall-cmd --permanent --zone=public --add-service=dns
sudo firewall-cmd --permanent --add-port=53/tcp --add-port=53/udp
sudo firewall-cmd --add-port=53/tcp --add-port=53/udp
sudo firewall-cmd --reload
sudo restorecon -vR /var/named
sudo chown named:named /var/named/jftest.com /var/named/10.80.158.db
sudo systemctl enable --now named
```

### Check DNS records

Now that DNS is up and running we need to verify if it is actually working. This can easily be verified by checking a couple of A records, SRV records and performing some nslookup queries. If they all come back with what you expect congrats DNS is functional!
```text
dig api-int.ocp4.jftest.com +short
dig api.ocp4.jftest.com +short
dig _etcd-server-ssl._tcp.ocp4.jftest.com SRV +short

nslookup api.ocp4.jftest.com
nslookup api-int.ocp4.jftest.com
nslookup master0.ocp4
```

## HA Proxy

First, create a backup of the default file (in case you make a mess) if you haven't guessed yet this is a personal habit of mine! Then modify the default file to look something like the below example.

```text
cp -p /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.vanilla
vim /etc/haproxy/haproxy.cfg
```

#### /etc/haproxy/haproxy.cfg
```
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /
    monitor-uri /healthz


frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server boot 10.80.158.6:6443 check
    server master-0 10.80.158.7:6443 check
    server master-1 10.80.158.8:6443 check
    server master-2 10.80.158.9:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server boot 10.80.158.6:22623 check
    server master-0 10.80.158.7:22623 check
    server master-1 10.80.158.8:22623 check
    server master-2 10.80.158.9:22623 check

frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server worker-0 10.80.158.10:80 check
    server worker-1 10.80.158.11:80 check
    server worker-2 10.80.158.12:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server worker-0 10.80.158.10:443 check
    server worker-1 10.80.158.11:443 check
    server worker-2 10.80.158.12:443 check
```

### Start HAProxy

Add the relevant firewall rules, make selinux happy and then start the service.
```text
sudo firewall-cmd --permanent --add-port=9000/tcp \
--add-port=443/tcp \
--add-port=80/tcp \
--add-port=6443/tcp \
--add-port=6443/udp \
--add-port=22623/tcp \
--add-port=22623/udp
sudo firewall-cmd --reload
sudo setsebool -P haproxy_connect_any on
sudo systemctl enable haproxy --now
systemctl status haproxy
```

You can view the status of the load balancer on port 9000 e.g. http://139.178.72.19:9000










# Ignition

We're at the stage now when all the pre-req has been done and we can actually prepare to start the installation process for Openshift. Following the installation documentation prepare an ssh key and directory.
```text
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/vsphere-ocp4
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/vsphere-ocp4
cd ~
mkdir ocp4
cd ocp4
touch install-config.yaml
```

Output the contents of the following files as you'll need all this detail for the ```install-config.yaml``` file.
```text
cat /tmp/pull-secret-ugly.json
cat ~/.ssh/vsphere-ocp4.pub
cat /opt/registry/certs/domain.crt
vim install-config.yaml
```

Now create the ```install-config.yaml``` file using the below template but amending it per your values. I won't bother going over what all the parameters are as this can easily be found in the actual Openshift installation documents. FYI for ESXi provisioned on Packet the datacenter value is always 'ha-datacenter'.
```yaml
apiVersion: v1
baseDomain: jftest.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
platform:
  vsphere:
    vcenter: 139.178.72.18
    username: root
    password: 7Y;tzq-4U6
    datacenter: ha-datacenter
    defaultDatastore: datastore1
fips: false
pullSecret: '************************************************************'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC/DPorZQ5VIbQKW7A2G/0nz5uFOKBkkQq7oMum4YlfFJVl2W721LNywT5cd3CJYmYUL7UWdwIp5uQexb4/VSlpdNEgwHhnt4c805EaLkZRNaxBpGE2Q3DNiN9X2nqHH+AHIRivA7uv3smIn+5Gd9WjB1vTXkdYSlwUPIJI916XYLV7BeVjTgQKyJ1OscAUVFwJZG6n3QVQWHK75w2yGd1yNn0OpJz2mL39fxeaseUDUicO1rtNWgmaLXLi1dtWQ0xYmW+Hf4Sh6zKYm/b6jPLj4Q/A64C8LS7RB+sbpWJ1Tiqk/O5WLWXByRrGhH8xsZnRms6lsu+GQCgXbgu4EuDFFpk9Jq+vyI7GFNaAB+XhOq4UpUiLP8cegRe2Y4IlXkgp6rmJWK1DOI51+P9lu0EI3KSFfHBHVjRXhXtvhG4dlELDRGYZuG9ZGR8Q4KIO4vLIQyuzia5IC3PVS4AEWmpn53BDsiGX1waegClapG/kKHgdNIwRZkCnjo/LUB9e6Ls6Vd1YR9yBt6B9C3hPe/+3HO040bV16hQi+1JnK5Iw6GtIZLEGPdTNTjdVxLEsLHDCid90JclCMuay5mR2KdWvXJfxOnGhimUbsVCmspYJgb0Ee0JnF0RRMNtm2xUS4ybRLC5bCv9SXcpXt7lYQYSUPcm2SJceAyM1T6cpyLEvBQ== james@packetfeathersbastion.jftest.com'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIFuzCCA6OgAwIBAgIUBvNZ/tsER/n9yqaUZiX0OkRCMjQwDQYJKoZIhvcNAQEL
  BQAwbTELMAkGA1UEBhMCWFgxFTATBgNVBAcMDERlZmF1bHQgQ2l0eTEcMBoGA1UE
  CgwTRGVmYXVsdCBDb21wYW55IEx0ZDEpMCcGA1UEAwwgcGFja2V0ZmVhdGhlcnNi
  YXN0aW9uLmpmdGVzdC5jb20wHhcNMjAwNjExMTA1NzAzWhcNMjEwNjExMTA1NzAz
  WjBtMQswCQYDVQQGEwJYWDEVMBMGA1UEBwwMRGVmYXVsdCBDaXR5MRwwGgYDVQQK
  DBNEZWZhdWx0IENvbXBhbnkgTHRkMSkwJwYDVQQDDCBwYWNrZXRmZWF0aGVyc2Jh
  c3Rpb24uamZ0ZXN0LmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIB
  ANQz5cndKH4QCg22xmorF44vYkSM0cu8qZAooJR56fedBOvwYSaqvKoWWwboQEAl
  0LdH4/y0wv/6Ia8v3UP4EDts+boMkuJT1XsOYNXEzKRuD+3oepBPP/VVMw3glGXu
  QJpEOcvhC2pT0c5ZTLmjvEQee5Wlc087Mu0KiQFWl1W/yUe5ubz50EJjClIaG8o0
  j4ZqSEJMFnYhB3f5PkF/HtdS5Ozcor4W5+uu0zU4BO+ZmBoTHjYXDGDaKZ/EmWPm
  haW5NGD7G359HKa7Eh+ZORFjOZfj7R4YGlieVw5yN6IXsRp1iR7jhLGBYZK6O0++
  dCnkBXqFIjOL/eDOYwfKIebsi+tsBzcmaFF59jpWwW3EQPDTGPeqmMPOVQ0RjXub
  y2UQLgLTGQCxOd3p0EjCSwC/kVu7CgycNuXQIgVt87VyiKCg8tL54sbs4tyJb51h
  8oQFMLbla+1ZjBaA5VIH+3njTSncgXYANnFmC7YdEN4q76fZWbIH4huwpzsl1tBt
  LSyvSRwnFrRml2KMPCv80bRXbK79SWGHJSBlvcx08vKvnXD+R3mFn1pBwI+4RGDC
  yuaa2mX0+nUEBaF0WksXzEUUezYHF+YL3FCbMHvw02r7KpunJuVcudhz1SLQMW+N
  zF15OGASYJtPopST7tZUvQtk/28DZsFYyZ2lJZxdiCflAgMBAAGjUzBRMB0GA1Ud
  DgQWBBS4M7H5i+EfQYC2UMlgszpCgDH1+jAfBgNVHSMEGDAWgBS4M7H5i+EfQYC2
  UMlgszpCgDH1+jAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4ICAQC4
  MsauQR02hRd2qYeYqPXobc8BbG+sxOa9M1SGweqzq627z4Ws4BxMZ60uxo5sox+6
  ea/6lp0EZ6RINZmysHhPhq2uPhDWKxUEriM1HdicZjGHFz/beEBIKLi3U1hlAR7N
  MDqAFeCH01X9/cgNN7EvZcmguhq3Is+OAIZjPPBAR5UHBDUdjbufeee0yHrwT/kP
  2f5D5VKRvUCor6VaJR+jwEUPEeOsJcxu6Ns1rYbO3ZXJyna3g1ZsHVd1FboH0Al4
  I4E0hpKnGNgMmZ/LC0TA9cn7bQkkpkptZ5vzdmBpIb7GU4PMX6MyRhBjlgnFAKB6
  KwdptYAI/Gq9ffaQEZa22II00hS66++u+TUgScECCxr65vPKvCDGJZBm4C3I8AnM
  qZARTqxnCvhf7UYEBAFcPXI+WodynapNZeagQGJrJJLMCuoy2etYrdRMwkEgCLJj
  aDqhs8yrJ5WXy4nHjoHFfadKDo72oXlANj8sSa7QYp9JXp6M8NmUbm7vrjSd8ajc
  dmK9pYo8lRNZzwBVwPqICWxvaZiudtLhx4lMqKkKbsg3SohGgwIeQ/9ybZ/f31be
  V/yPRCfOVKhL551c9C5FdYBAEwIi7+dJt4ET8fSw3/5k9Vwg5ev0WDImHNadiUBK
  2PCbKkvyfqrBQ/lsvN3upI0sDI372JAchVXyX5H8ow==
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - packetfeathersbastion.jftest.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - packetfeathersbastion.jftest.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - packetfeathersbastion.jftest.com:5000/ocp4/openshift4
  source: registry.svc.ci.openshift.org/ocp/release
```

> Note: I added that bottom one as that source existed on the installation document example. I am not sure if it is necessary or not.

Now you should have a hopefully killer ```install-config.yaml``` file first create a backup of it (as the installer eats it) then follow the rest of the commands to create the ignition files and then copy them to the webserver.
```text
cp ~/ocp4/install-config.yaml ~
cd ~/ocp4
openshift-install create manifests
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' manifests/cluster-scheduler-02-config.yml
openshift-install create ignition-configs
sudo cp -pr *.ign /var/www/html/
sudo chmod 644 /var/www/html/*
restorecon -R -v /var/www/html
vi ~/ocp4/append-bootstrap.ign
```

#### Probably not necessary

I ended up creating another ignition file (which is basically an ignition file to pull another ignition file) then creating base64 encoded copies of all the ignition files and although I technically don't need them for the process I'm performing I think it's still worth noting for example if you were doing an install where you had DHCP.
```json
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "http://10.80.158.5:8080/bootstrap.ign",
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "2.1.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
```

Commands to create the base64 encoded versions of the ignition files.
```text
base64 -w0 ~/ocp4/master.ign > ~/ocp4/master.64
base64 -w0 ~/ocp4/worker.ign > ~/ocp4/worker.64
base64 -w0 ~/ocp4/append-bootstrap.ign > ~/ocp4/append-bootstrap.64
```






# Create a multi optioned boot iso

At this stage we are more or less ready, in fact, if this setup had DHCP we would be ready... however it does not and that's not what we're doing... so as we are using static IPs we need some way to set them. We'll do this by preparing an iso that we will first boot the VMs with which will set the network configuration and then perform the installation by pulling down the metal image and ignition files from our web server.

Mount the install iso and sync the files to a new directory. We can now modify the 'isolinux.cfg' file then package it all up in shiny new iso.
```text
cd ~
sudo mkdir /mnt/rhcos_iso
sudo mount rhcos-4.4.3-x86_64-installer.x86_64.iso /mnt/rhcos_iso/
rsync -a /mnt/* /tmp/
cd /tmp/rhcos_iso
vi isolinux/isolinux.cfg
```

For the purpose of this activity, I opted to create one multi boot iso with multiple options burnt in. When you boot with this iso you will be able to select which machine you want to create. This is okay, but not exactly a great process. I think I'll do another post around automation of this process, I believe there are some out there already, I just need to have a look, so watch this space!

A single boot entry will look like this (however everything after append should be on the same line - I've displayed it this way for readability and hopefully make it easier to understand what information we're giving to the boot process):
```text
label linux
  menu label ^Install RHEL CoreOS bootstrap0
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1
  coreos.inst=yes ip=10.80.158.6::10.80.158.1:255.255.255.240:bootstrap0.ocp4.jftest.com:ens192:none nameserver=10.80.158.5
  coreos.inst.install_dev=sda
  coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz
  coreos.inst.ignition_url=http://10.80.158.5:8080/bootstrap.ign
```

So when you've understood and digested what the above is doing (the actual Openshift documentation for a bare metal install explains this well so I won't bother) add the multiple boot option entries into the 'isolinux.cfg' file towards the bottom (there will be one existing one so where to put them should be self-explanatory). Quoting the classic line "Here's one I made earlier":
```text
label linux
  menu label ^Install RHEL CoreOS bootstrap0
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes ip=10.80.158.6::10.80.158.1:255.255.255.240:bootstrap0.ocp4.jftest.com:ens192:none nameserver=10.80.158.5 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://10.80.158.5:8080/bootstrap.ign

label linux
  menu label ^Install RHEL CoreOS master0
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes ip=10.80.158.7::10.80.158.1:255.255.255.240:master0.ocp4.jftest.com:ens192:none nameserver=10.80.158.5 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://10.80.158.5:8080/master.ign

label linux
  menu label ^Install RHEL CoreOS master1
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes ip=10.80.158.8::10.80.158.1:255.255.255.240:master1.ocp4.jftest.com:ens192:none nameserver=10.80.158.5 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://10.80.158.5:8080/master.ign

label linux
  menu label ^Install RHEL CoreOS master2
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes ip=10.80.158.9::10.80.158.1:255.255.255.240:master2.ocp4.jftest.com:ens192:none nameserver=10.80.158.5 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://10.80.158.5:8080/worker.ign

label linux
  menu label ^Install RHEL CoreOS worker0
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes ip=10.80.158.10::10.80.158.1:255.255.255.240:worker0.ocp4.jftest.com:ens192:none nameserver=10.80.158.5 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://10.80.158.5:8080/worker.ign

label linux
  menu label ^Install RHEL CoreOS worker1
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes ip=10.80.158.11::10.80.158.1:255.255.255.240:worker1.ocp4.jftest.com:ens192:none nameserver=10.80.158.5 coreos.inst.install_dev=sda coreos.inst.image_url=http://10.80.158.5:8080/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://10.80.158.5:8080/worker.ign
```

Next job is to package everything up into a shiny new iso. This long one-liner will create a nice iso which you will then use to boot the VMs up with and allow you to select one of the options set above. Special thanks to Shanna for this process! (see references at the bottom)
```text
sudo mkisofs -U -A "RHCOS-x86_64" -V "RHCOS-x86_64" -volset "RHCOS-x86_64" -J -joliet-long -r -v -T -x ./lost+found -o /tmp/rhcos_install.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e images/efiboot.img -no-emul-boot .
```

## Upload modded iso to ESXi datastore

To easily upload the modified iso to ESXi datastore I used govc (a vSphere CLI) as I'm somewhat familiar with it. You could probably also just scp/sftp it to the correct directory on the ESXi server. If you wish to do it using govc this is what to do:
```text
wget https://github.com/vmware/govmomi/releases/download/v0.22.1/govc_linux_amd64.gz
gunzip govc_linux_amd64.gz
sudo mv govc_linux_amd64 /usr/local/sbin/govc
sudo chmod 777 /usr/local/sbin/govc
export GOVC_URL='139.178.72.18	'
export GOVC_USERNAME='root'
export GOVC_PASSWORD='7Y;tzq-4U6'
export GOVC_NETWORK='VM Network'
export GOVC_DATASTORE='datastore1'
export GOVC_INSECURE=1
govc datastore.upload --ds=datastore1 --dc=ha-datacenter /tmp/rhcos_install.iso iso/rhcos_install.iso
```






# Openshift Install Process

## Create VMs

Create VMs using the ova template and then when prompted import the base64 ignition file for its relevant use case... I'm saying this as I did this, however, I do not believe this is required at all as the ignition files will be pulled down when we run the installer from the iso. So if you want to try without, feel free to. I'll be revisiting this process for improvements soon so an update is on the way.

1. Edit boot settings so that the VM boots from cd first
2. Run the VM and select what element you will be installing e.g. bootstrap first
3. Remove the DVD drive and then reboot the guest VM

I ran step 1 and 2 on all the guest VMs first e.g. bootstrap, x3 masters and x2 workers. Then I performed step 3 and booted the bootstrap and masters in order. I later booted the workers when the masters were up.

![Finished, here's the console](/images/ocp_bootmenu.png#center)


## Check bootstrap process

When the VMs are coming up it's really just a waiting game... the below commands might be useful for troubleshooting (they were very useful for me).
```text
ssh -i ~/.ssh/vsphere-ocp4 core@10.80.158.6
ip addr show
sudo cat /etc/containers/registries.conf
curl -u james:fugazi77 -k https://packetfeathersbastion.jftest.com:5000/v2/_catalog
journalctl -u release-image.service
journalctl -b -f -u bootkube.service
```

Monitor the bootstrap process:
```text
openshift-install wait-for bootstrap-complete --log-level=info
```

If everything is going according to plan the monitoring process will progress to the waiting for the bootstrap complete step. Keep this process running in a separate session.
```text
  INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp4.jftest.com:6443...
  INFO API v1.17.1+f63db30 up
  INFO Waiting up to 40m0s for bootstrapping to complete...
```

As before with the bootstrap node, you can ssh to the master nodes and check that they are coming up okay.
```text
ssh -i ~/.ssh/vsphere-ocp4 core@10.80.158.7
ssh -i ~/.ssh/vsphere-ocp4 core@10.80.158.8
ssh -i ~/.ssh/vsphere-ocp4 core@10.80.158.9
```

You can also check the load balancer: http://139.178.72.19:9000/

Once the bootstrap process is complete (like with the example below) you can export your KUBECONFIG directory and you can view your nodes.

```text
  INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp4.jftest.com:6443...
  INFO API v1.17.1+f63db30 up
  INFO Waiting up to 40m0s for bootstrapping to complete...
  INFO It is now safe to remove the bootstrap resources
```

```text
export KUBECONFIG=/home/james/ocp4/auth/kubeconfig
oc whoami
oc get nodes
```

Awesome! The masters have come up okay!
```text
NAME                      STATUS   ROLES    AGE   VERSION
master0.ocp4.jftest.com   Ready    master   15m   v1.17.1
master1.ocp4.jftest.com   Ready    master   15m   v1.17.1
master2.ocp4.jftest.com   Ready    master   15m   v1.17.1
```


## Approve certs
Check pending certificate signing requests and approve the ones for your worker nodes.
```text
oc get csr
oc adm certificate approve <csr_name>
oc get co
```


## Completing the installation:

A colleague gave me this handy command to run. You want to see no output except for those stars.
```text
while true; do oc get pods --all-namespaces | grep -v Completed | grep "0/"; sleep 3; echo "*****************"; done
```

Check all the are up okay
```text
oc get nodes
```
```text
NAME                      STATUS   ROLES    AGE   VERSION
master0.ocp4.jftest.com   Ready    master   45m   v1.17.1
master1.ocp4.jftest.com   Ready    master   44m   v1.17.1
master2.ocp4.jftest.com   Ready    master   45m   v1.17.1
worker0.ocp4.jftest.com   Ready    worker   27m   v1.17.1
worker1.ocp4.jftest.com   Ready    worker   26m   v1.17.1
```

Monitor for cluster completion:
```text
openshift-install wait-for install-complete
```
```text
  INFO Waiting up to 30m0s for the cluster at https://api.ocp4.jftest.com:6443 to initialize...
  INFO Waiting up to 10m0s for the openshift-console route to be created...
  INFO Install complete!
  INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/jforce/ocp4/auth/kubeconfig'
  INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.jftest.com
  INFO Login to the console with user: kubeadmin, password: jxgag-qtQ27-bCCsW-TauyP
```

You should now be able to login to the web console. I'm doing so from my bastion host in the ESXi console.

![Finished, here's the console](/images/ocp_vmware_finished.png#center)

![Finished, here's the console](/images/ocp_vmware_finished3.png#center)

Congrats, if you made it this far you've completed the installation! ...or just read my article... equally a good accomplishment!


# Summary, Thoughts and Troubleshooting

The installation process isn't as bad as I thought. It's relatively straight forward once you have the re-reqs such as correct DNS records, a properly configured load balancer and a web server with proper permissions serving the correct content it's fairly straight forward after that.

One thing that I did find odd was that I had to create a VM using the ova as I couldn't get it to work myself when creating a VM. Whenever I tried with a non ova created VM it would hang in the dracut process. Shouldn’t booting from the iso and it pulling the necessary file be enough. I'm not quite sure why the ova created VM worked and a non ova created VM didn't... I need to investigate this some more (there is probably some reason I've missed).

As I mentioned I think I will write another article soon about putting some automation around this process.

# References

The below documentation and articles helped me put all this together.

1. https://docs.openshift.com/container-platform/4.4/installing/installing_vsphere/installing-restricted-networks-vsphere.html
2. https://docs.openshift.com/container-platform/4.4/installing/install_config/installing-restricted-networks-preparations.html
3. https://docs.openshift.com/container-platform/4.4/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html
4. https://shanna-chan.blog/2020/03/16/openshift4-3-retest-static-ip-configuration-on-vsphere/
5. https://github.com/christianh814/openshift-toolbox/blob/master/ocp4_upi/docs/0.prereqs.md#dns
