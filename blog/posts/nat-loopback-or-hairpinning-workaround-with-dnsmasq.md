---
title: "Nat Loopback or Hairpinning Workaround With Dnsmasq"
date: 2019-10-09T11:27:48+02:00
draft: true
---
https://raw.github.com/stephen-mw/raspberrypi/master/roles/dnsmasq_server

/lib/systemd/system/dnsmasq.service
```
[Unit]
Description=dnsmasq - A lightweight DHCP and caching DNS server
Requires=network.target
Wants=nss-lookup.target
Before=nss-lookup.target
After=network.target

[Service]
Type=forking
PIDFile=/run/dnsmasq/dnsmasq.pid

# Test the config file and refuse starting if it is not valid.
ExecStartPre=/usr/sbin/dnsmasq --test

# We run dnsmasq via the /etc/init.d/dnsmasq script which acts as a
# wrapper picking up extra configuration files and then execs dnsmasq
# itself, when called with the "systemd-exec" function.
ExecStart=/etc/init.d/dnsmasq systemd-exec

# The systemd-*-resolvconf functions configure (and deconfigure)
# resolvconf to work with the dnsmasq DNS server. They're called like
# this to get correct error handling (ie don't start-resolvconf if the 
# dnsmasq daemon fails to start.
ExecStartPost=/etc/init.d/dnsmasq systemd-start-resolvconf
ExecStop=/etc/init.d/dnsmasq systemd-stop-resolvconf


ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

/etc/dnsmasq.conf
```conf
# Dnsmasq.conf for raspberry pi    
# By Stephen Wood heystephenwood.com  
# Full examples found here:  
# http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq.conf.example  
 
# Set up your local domain here    
domain=raspberrypi.home    
resolv-file=/etc/resolv.dnsmasq  
min-port=4096   
server=8.8.8.8
server=8.8.4.4
      
# Max cache size dnsmasq can give us, and we want all of it!    
cache-size=10000    
      
# Below are settings for dhcp. Comment them out if you dont want    
# dnsmasq to serve up dhcpd requests.    
# dhcp-range=192.168.0.100,192.168.0.149,255.255.255.0,1440m    
# dhcp-option=3,192.168.0.1    
# dhcp-authoritative

address=/pbcompaan.tk/192.168.1.110

log-dhcp
log-queries
log-facility=/var/log/dnsmasq.log
```

install 
```bash
apt-get install -y dnsmasq
```