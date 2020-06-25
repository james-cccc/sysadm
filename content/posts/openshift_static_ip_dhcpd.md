---
date: 2020-06-22T18:00:00
description: "Modify CoreOS ignition files to set static network configuration (ESXi On Packet pt2)"
#featured_image: "/images/keycloak_logo.png"
tags: ["Openshift","VMware"]
# categories: "POC"
title: "Modifying Openshifts CoreOS ignition files"
comment : false
---

This article was put together to help assist performing Openshift 4 deployments on VMWare ESXi where static IPs are required but where DHCP is available and is allowed to give out temporary addresses. Doing it this way as (opposed to my last article where DHCP was not used) allows us to use the ova templates and will not require the creation of any separate boot isos. This is possible as enabling DHCP will allow the machines that have been created with the just the OVAs to boot first using a random DHCP given IP. However, as we want a static addresses we can set what the network configuration will be by injecting this into the ignition files and once the machine has booted for the first time using DHCP it will then use the network configuration to what is defined in the ignition file going forward.

Like my last article, I performed and validated the steps below on ESXi hosted by Packet. I did not include all the steps (like my last article did); only the pertinent parts this time. If you follow my previous article up the the point of generating ignition files you should be able to follow the rest of the steps listed here.

### High Level Overview

1. Install and setup DHCP server
2. Modify Ignition files
3. Create VMs

### Networking

The network setup is the same as it was in the previous article. So I will not go into details here this time as the previous article can be referred to if necessary. For the DHCP element, we only care about addresses in our private range. If you don't want to flick back or read the last article, here's the network summary for our private addresses:

| ADDRESS                | NETWORK                   | GATEWAY               | TYPE         |
|------------------------|---------------------------|-----------------------|--------------|
| 10.80.158.2            | 10.80.158.0/28            | 10.80.158.1           | Private IPv4 |

For our private network we have a CIDR of /28, so its netmask is 255.255.255.240.

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


##  Setup DHCP

First, create a backup of the default file (in case you make a mess) and then go ahead and amend the default file ```/etc/dhcp/dhcpd.conf``` with something like the below example. I crafted the below example actually using the shipped example file ```/usr/share/doc/dhcp-server/dhcpd.conf.example```. It's fairly straight forward, just set the relevant network configuration and range of IP addresses you want to use, in my case I set that to be the four IP adddresses 10.80.158.11 - 10.80.158.14 (this is a very small range but good enough for what we're doing to test this process).

```text
sudo yum -y install dhcp-server
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.vanilla
sudo cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf
sudo vim /etc/dhcp/dhcpd.conf
```

#### /etc/dhcp/dhcpd.conf
```
# dhcpd.conf

option domain-name "jftest.com";
option domain-name-servers 10.80.158.5;

default-lease-time 600;
max-lease-time 7200;

ddns-update-style interim;

authoritative;

subnet 10.80.158.0 netmask 255.255.255.240 {
  range 10.80.158.11 10.80.158.14;
  option routers 10.80.158.1;
  option routers 10.5.5.1;
  option broadcast-address 10.80.158.15;
}
```

### Start DHCP

Add the relevant firewall rules then start the DHCP service. Bonus points if it starts straight away with no errors! If you do get errors, go back and check what you've done.
```text
sudo firewall-cmd --add-service=dhcp --permanent
sudo firewall-cmd --reload
sudo systemctl enable dhcpd --now
```

Great, at this stage if you were to boot a VM with the ova it would pick up an IP address from DHCP and run with it. That might actually be enough for you if this is just a temporary environment where you don't care about static addresses, but if want to set dedicated IPs as mentioned we will have to put them into our ignition files.

# Ignition

My previous article covered creating the ignition files, so again I will not cover this here. From now the prerequisites and creation of the ignition files should be done. The below file is what my master.ign file that was generated by the Openshift install tool looks like:

```json
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "https://api-int.ocp4.jftest.com:22623/config/master",
          "verification": {}
        }
      ]
    },
    "security": {
      "tls": {
        "certificateAuthorities": [
          {
            "source": "data:text/plain;charset=utf-8;base64,LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURFRENDQWZpZ0F3SUJBZ0lJVmFMNjVXRGsvdEl3RFFZSktvWklodmNOQVFFTEJRQXdKakVTTUJBR0ExVUUKQ3hNSmIzQmxibk5vYVdaME1SQXdEZ1lEVlFRREV3ZHliMjkwTFdOaE1CNFhEVEl3TURZeE56RTBNVEkwTkZvWApEVE13TURZeE5URTBNVEkwTkZvd0pqRVNNQkFHQTFVRUN4TUpiM0JsYm5Ob2FXWjBNUkF3RGdZRFZRUURFd2R5CmIyOTBMV05oTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF2a1J4UjlNcm0wRlMKWjdDbnNkdmswOFllaTJoOGlMMWRyejBldzhMNUp0b0F2MzEzM2J4T1ZUcW96aWtMQlUvVUM3b1I4ejVncGFBYgpEOThCdGY5S2FKQ2kyRHRxYjJmQU9KV042L1NYaC9EUjR5T0pyVjY1V1NIKzZyMlJwOVo1b0FlY2IrUXdybEFCCnh3SHVEbVovdDA0Rm5zVktzK3VMQzZ5SitiTU81Y2c2ek55cmlCblkvY2pZd2NEekZNNkowSnJ2VG54VlpCT0YKM0RiOE1XQU02bEIvUjYrTzBKcktuWU1xZXdzYTBwMVpoSVNZMVBCN3RvQTMxS2NaREh2QTZMVW9jTGduQUYwago1Ylk4cVJITHQrcnNrdkJ0dHhMdzlDbVRLTVl6eDZaVlFtTDVjRjVlSCtkVXpNSVhuaHNpdVZnRmJPRXpDbWlRCjBnc29FQXlVU1FJREFRQUJvMEl3UURBT0JnTlZIUThCQWY4RUJBTUNBcVF3RHdZRFZSMFRBUUgvQkFVd0F3RUIKL3pBZEJnTlXIUTRFRmdRVVY3L0tEK3o2TmRQZCs2azNNUVRnWEF1dnVTUXdEUVlKS29aSWh2Y05BUUVMQlFBRApnZ0VCQUs0a3BRTHlleGdWOHBNVAJZZ04xVlpnY25Mb2tOZFhDdG8vTGZ4UldqRElDc1lLWkt3azVZVUg3eTVaCm4wV2svTG9wcFJlNkdCUnpEVStCS3daUEZpZWM0V1FqZnowcWdUZ0tOOVBiNzVVY1hZSjA5RVJQR3N1Ymo5aWoKVlE2VGxPb3lVYnhiWjFsamo4MW1aVmpPc3BQbFhySEF4amwvNjRNckpDRWYzbFRrSnJnZ3p5d2RvTmVmOGo0ZQpmOFZSaFlUQklLQVlLSlorMnZzZkRmejRRT1NRTXFhVUtIRHB5UUthc01zQ3dCSFBjRmZYWUgzTUFVenFTdE90CnYwRHFyMjdkVmRsOU1QZlJXRHkrK3g2Nk80Y2llUE9rbzY1OVo2U3J1TWZrQXkwMGZOcjZYK09JMEpaTnI3QzUKSTUyUHBFRTdhMmVEdGxDa2NsUkhYT3ZaR3ZRPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==",
            "verification": {}
          }
        ]
      }
    },
    "timeouts": {},
    "version": "2.2.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
```

#### FYI

To encode and decode a string you can use the ```base64``` command, e.g.

```text
echo 'james' | base64 -w0
echo 'amFtZXMK' | base64 -d
```
This will come in handy.

### Creating new modified Ignition files

What we will do here is create a separate ignition file for each node, so for all teh master nodes create a copy of the masters ignition and the same for the workers, for the bootstrap you wil have to create the one ammend-botstrap ignition file. For us to add the network configuration, we can break it down into 2 additional sections that we need to add.

1. **Network**

The first section is in the ```networkd``` part, we're adding the IP address, Gateway, DNS and Domain like so:

```json
  "networkd": {
    "units": [
      {
        "name": "00-ens192.network",
        "contents": "[Match]\nName=ens192\n\n[Network]\nAddress=.10.80.158.7/28\nGateway=10.80.158.1\nDNS=10.80.158.5\nDomains=jftest.com\n"
      }
    ]
  },
```

2. **Storage**

The second section is in the ```storage``` part where we are defineing two files to be created. The first file is the ```/etc/hostname``` file, in which we are setting it's contents to ```master0.ocp4.jftest.com```. The second file is the Interface configuration (ifcfg) file where we are adding network configuration.

```json
  "storage": {
    "files": [
      {
        "path": "/etc/hostname",
        "mode": 420,
        "contents": {
          "source": "data:text/plain;charset=utf-8;base64,bWFzdGVyMC5vY3A0LmpmdGVzdC5jb20=",
          "verification": {}
        },
        "filesystem": "root"
      },
      {
        "path": "/etc/sysconfig/network-scripts/ifcfg-ens192",
        "mode": 420,
        "contents": {
          "source": "data:text/plain;charset=utf-8;base64,IyBHZW5lcmF0ZWQgYnkgY3JlYXRlaWducyAKVFlQRT1FdGhlcm5ldApOQU1FPSJlbnMxOTIiCkRFVklDRT0iZW5zMTkyIgpPTkJPT1Q9eWVzCk5FVEJPT1Q9eWVzCkJPT1RQUk9UTz1ub25lCklQQUREUj0iMTAuODAuMTU4LjciCk5FVE1BU0s9IjI1NS4yNTUuMjU1LjI0MCIKR0FURVdBWT0iMTAuODAuMTU4LjEiCkROUzE9IjEwLjgwLjE1OC41Ig==Cg==",
          "verification": {}
        },
        "filesystem": "root"
      }
    ]
  },

```

The below is the ifcfg file I created which as mentioned above was converted into a base64 string and added into the ignition file:
```
# Generated by createigns
TYPE=Ethernet
NAME="ens192"
DEVICE="ens192"
ONBOOT=yes
NETBOOT=yes
BOOTPROTO=none
IPADDR="10.80.158.7"
NETMASK="255.255.255.240"
GATEWAY="10.80.158.1"
DNS1="10.80.158.5"
```

The reason that we are base64 encoding these values are that there a limit on the size of the config which may be provided to a machine. Therefore, base64 encoding them works around this. We could probably get away with having the hostname in cleartext like shown in the CoreOS documentation like the example below... but to be on the safe side, probably better to encode it.

```json
"storage": {
   "files": [
  {
     "filesystem": "root",
     "path": "/etc/hostname",
     "mode": 420,
     "contents": { "source": "data:,master0" }
   }
 ]
 }
```


#### Post Modification

Postmodification each file will look similar to the example below:

```json
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "https://api-int.ocp4.jftest.com:22623/config/master",
          "verification": {}
        }
      ]
    },
    "security": {
      "tls": {
        "certificateAuthorities": [
          {
            "source": "data:text/plain;charset=utf-8;base64,LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURFRENDQWZpZ0F3SUJBZ0lJVmFMNjVXRGsvdEl3RFFZSktvWklodmNOQVFFTEJRQXdKakVTTUJBR0ExVUUKQ3hNSmIzQmxibk5vYVdaME1SQXdEZ1lEVlFRREV3ZHliMjkwTFdOaE1CNFhEVEl3TURZeE56RTBNVEkwTkZvWApEVE13TURZeE5URTBNVEkwTkZvd0pqRVNNQkFHQTFVRUN4TUpiM0JsYm5Ob2FXWjBNUkF3RGdZRFZRUURFd2R5CmIyOTBMV05oTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF2a1J4UjlNcm0wRlMKWjdDbnNkdmswOFllaTJoOGlMMWRyejBldzhMNUp0b0F2MzEzM2J4T1ZUcW96aWtMQlUvVUM3b1I4ejVncGFBYgpEOThCdGY5S2FKQ2kyRHRxYjJmQU9KV042L1NYaC9EUjR5T0pyVjY1V1NIKzZyMlJwOVo1b0FlY2IrUXdybEFCCnh3SHVEbVovdDA0Rm5zVktzK3VMQzZ5SitiTU81Y2c2ek55cmlCblkvY2pZd2NEekZNNkowSnJ2VG54VlpCT0YKM0RiOE1XQU02bEIvUjYrTzBKcktuWU1xZXdzYTBwMVpoSVNZMVBCN3RvQTMxS2NaREh2QTZMVW9jTGduQUYwago1Ylk4cVJITHQrcnNrdkJ0dHhMdzlDbVRLTVl6eDZaVlFtTDVjRjVlSCtkVXpNSVhuaHNpdVZnRmJPRXpDbWlRCjBnc29FQXlVU1FJREFRQUJvMEl3UURBT0JnTlZIUThCQWY4RUJBTUNBcVF3RHdZRFZSMFRBUUgvQkFVd0F3RUIKL3pBZEJnTlXIUTRFRmdRVVY3L0tEK3o2TmRQZCs2azNNUVRnWEF1dnVTUXdEUVlKS29aSWh2Y05BUUVMQlFBRApnZ0VCQUs0a3BRTHlleGdWOHBNVAJZZ04xVlpnY25Mb2tOZFhDdG8vTGZ4UldqRElDc1lLWkt3azVZVUg3eTVaCm4wV2svTG9wcFJlNkdCUnpEVStCS3daUEZpZWM0V1FqZnowcWdUZ0tOOVBiNzVVY1hZSjA5RVJQR3N1Ymo5aWoKVlE2VGxPb3lVYnhiWjFsamo4MW1aVmpPc3BQbFhySEF4amwvNjRNckpDRWYzbFRrSnJnZ3p5d2RvTmVmOGo0ZQpmOFZSaFlUQklLQVlLSlorMnZzZkRmejRRT1NRTXFhVUtIRHB5UUthc01zQ3dCSFBjRmZYWUgzTUFVenFTdE90CnYwRHFyMjdkVmRsOU1QZlJXRHkrK3g2Nk80Y2llUE9rbzY1OVo2U3J1TWZrQXkwMGZOcjZYK09JMEpaTnI3QzUKSTUyUHBFRTdhMmVEdGxDa2NsUkhYT3ZaR3ZRPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==",
            "verification": {}
          }
        ]
      }
    },
    "timeouts": {},
    "version": "2.2.0"
  },
  "networkd": {
    "units": [
      {
        "name": "00-ens192.network",
        "contents": "[Match]\nName=ens192\n\n[Network]\nAddress=.10.80.158.7/28\nGateway=10.80.158.1\nDNS=10.80.158.5\nDomains=jftest.com\n"
      }
    ]
  },
  "passwd": {},
  "storage": {
    "files": [
      {
        "path": "/etc/hostname",
        "mode": 420,
        "contents": {
          "source": "data:text/plain;charset=utf-8;base64,bWFzdGVyMC5vY3A0LmpmdGVzdC5jb20=",
          "verification": {}
        },
        "filesystem": "root"
      },
      {
        "path": "/etc/sysconfig/network-scripts/ifcfg-ens192",
        "mode": 420,
        "contents": {
          "source": "data:text/plain;charset=utf-8;base64,IyBHZW5lcmF0ZWQgYnkgY3JlYXRlaWducyAKVFlQRT1FdGhlcm5ldApOQU1FPSJlbnMxOTIiCkRFVklDRT0iZW5zMTkyIgpPTkJPT1Q9eWVzCk5FVEJPT1Q9eWVzCkJPT1RQUk9UTz1ub25lCklQQUREUj0iMTAuODAuMTU4LjciCk5FVE1BU0s9IjI1NS4yNTUuMjU1LjI0MCIKR0FURVdBWT0iMTAuODAuMTU4LjEiCkROUzE9IjEwLjgwLjE1OC41Ig==",
          "verification": {}
        },
        "filesystem": "root"
      }
    ]
  },
  "systemd": {}
}
```

You can validate your ignition files created online here: https://coreos.com/validate/ (which is worth doing).

You can then create the base64 encoded version of the ignition files, like so:
```text
base64 -w0 ~/ocp4/master0.ocp4.jftest.com.ign > ~/ocp4/master0.ocp4.jftest.com.64
```

The base64 encoded strings can then be used when you create the VM in ESXi. It will first boot the VM with a temporary DHCP assigned IP whilst it comes up, then once it's up it will set the static IP configuration. You will see the IP address in the console when the VM is up.


# Openshift Install Process

## Create VMs

If you are creating the VMs using the ESXi web management console create the VMs using the ova template and then when prompted import the base64 ignition file for that specific VM. Nothing more should be required at this stage other than to power it on and wait. The bootstrap process will start and that's basically the start of the end of the install, mainly just waiting around at this stage, so feel free to make a brew or whatever.

If you are feeling like you'd like to provsion the VM's in a better more automated way, you every much can! I was able to use the below ansible playbook to do just that and it worked gloriously.

```yaml
---
- name: Create VMs
  hosts: localhost
  tasks:

  - name: Provision VM using OVA
    vmware_deploy_ovf:
      hostname: '139.178.72.66'
      username: 'root'
      password: 'n4{D2^U5Gg'
      datacenter: ha-datacenter
      datastore: datastore1
      validate_certs: no
      ovf: /home/james/rhcos-4.4.3-x86_64-vmware.x86_64.ova
      name: bootstrap.ocp4.jftest.com
      power_on: no
      inject_ovf_env: true
      properties:
        guestinfo.ignition.config.data.encoding: "base64"
        guestinfo.ignition.config.data: "ewogICJpZ25pdGlvbiI6IHsKICAgICJjb25maWciOiB7CiAgICAgICJhcHBlbmQiOiBbCiAgICAgICAgewogICAgICAgICAgInNvdXJjZSI6ICJodHRwOi8vMTAuODAuMTU4LjU6ODA4MC9ib290c3RyYXAuaWduIiwKICAgICAgICAgICJ2ZXJpZmljYXRpb24iOiB7fQogICAgICAgIH0KICAgICAgXQogICAgfSwKICAgICJ0aW1lb3V0cyI6IHt9LAogICAgInZlcnNpb24iOiAiMi4xLjAiCiAgfSwKICAibmV0d29ya2QiOiB7CiAgICAidW5pdHMiOiBbCiAgICAgIHsKICAgICAgICAibmFtZSI6ICIwMC1lbnMxOTIubmV0d29yayIsCiAgICAgICAgImNvbnRlbnRzIjogIltNYXRjaF1cbk5hbWU9ZW5zMTkyXG5cbltOZXR3b3JrXVxuQWRkcmVzcz0uMTAuODAuMTU4LjYvMjhcbkdhdGV3YXk9MTAuODAuMTU4LjFcbkROUz0xMC44MC4xNTguNVxuRG9tYWlucz1qZnRlc3QuY29tXG4iCiAgICAgIH0KICAgIF0KICB9LAogICJwYXNzd2QiOiB7fSwKICAic3RvcmFnZSI6IHsKICAgICJmaWxlcyI6IFsKICAgICAgewogIAAgICAgICJwYXRoIjogIi9ldGMvaG9zdG5hbWUiLAogICAgICAgICJtb2RlIjogNDIwLAogICAgICAgICJjb250ZW50cyI6IHsKICAgICAgICAgICJzb3VyY2UiOiAiZGF0YTp0ZXh0L3BsYWluO2NoYXJzZXQ9dXRmLTg7YmFzZTY0LFltOXZkSE4wY21Gd0xtOWpjRFF1YW1aMFpYTjBMbU52YlE9PSIsCiAgICAgICAgICAidmVyaWZpY2F0aW9uIjoge30KICAgICAgICB9LAogICAgICAgICJmaWxlc3lzdGVtIjogInJvb3QiCiAgICAgIH0sCiAgICAgIHsKICAgICAgICAicGF0aCI6ICIvZXRjL3N5c2NvbmZpZy9uZXR3b3JrLXNjcmlwdHMvaWZjZmctZW5zMTkyIiwKICAgICAgICAibW9kZSI6IDQyMCwKICAgICAgICAiY29udGVudHMiOiB7CiAgICAgICAgICAic291cmNlIjogImRhdGE6dGV4dC9wbGFpbjtjaGFyc2V0PXV0Zi04O2Jhc2U2NCxJeUJIWlc1bGNtRjBaV1FnWW5rZ1kzSmxZWFJsYVdkdWN5QUtWRmxRUlQxRmRHaGxjbTVsZEFwT1FVMUZQU0psYm5NeE9USWlDa1JGVmtsRFJUMGlaVzV6TVRreUlncFBUa0pQVDFROWVXVnpDazVGVkVKUFQxUTllV1Z6Q2tKUFQxUlFVazlVVHoxdWIyNWxDa2xRUVVSRVVqMGlNVEF1T0RBdU1UVTRMallpQ2s1RlZFMUJVMHM5SWpJMU5TNHlOVFV1TWpVMUxqSTBNQ0lLUjBGVVJWZEJXVDBpTVRBdU9EQXVNVFU0TGpFaUNrUk9VekU5SWpFd0xqZ3dMakUxT0M0MUlnPT0iLAogICAgICAgICAgInZlcmlmaWNhdGlvbiI6IHt9CiAgICAgICAgfSwKICAgICAgICAiZmlsZXN5c3RlbSI6ICJyb290IgogICAgICB9CiAgICBdCiAgfSwKICAic3lzdGVtZCI6IHt9Cn0="
    delegate_to: localhost

  - name: Reconfigure CPU, RAM and HDD size of VM
    vmware_guest:
      hostname: '139.178.72.66'
      username: 'root'
      password: 'n4{D2^U5Gg'
      datacenter: ha-datacenter
      datastore: datastore1
      name: bootstrap.ocp4.jftest.com
      state: "present"
      validate_certs: "false"
      hardware:
        memory_mb: "16384"
        num_cpus: "4"
      disk:
        - size_gb: 60
          type: thin
    delegate_to: "localhost"
```

I won't cover the last steps of bringing up the cluster and approving the workers certs etc as the previous article covers all that (the process is similar if not the same at this stage anyway).

# Summary

I like this process a bit more than the last one, however, a DHCPD instance is required and you have to be careful adding DHCP into an environment that doesn't have one. Special thanks to Alejandro for the inspiration for this process! (see references at the bottom).

I've put together a role to create a set of base64 encoded ignition files. This might be useful to quickly generate all necessary ignitions https://github.com/james-cccc/createigns - I do plan on working this further to not only generate the modified ignitions but also do the provisioning maybe using the ```vmware_deploy_ovf``` module.

# References

The below documentation and articles helped me put all this together.

1. https://www.openshift.com/blog/how-to-install-openshift-on-vmware-with-terraform-and-static-ip-addresses
2. https://coreos.com/ignition/docs/latest/examples.html
3. https://coreos.com/ignition/docs/latest/network-configuration.html
4. https://docs.ansible.com/ansible/latest/modules/vmware_deploy_ovf_module.html
5. https://docs.ansible.com/ansible/latest/modules/vmware_guest_module.html
