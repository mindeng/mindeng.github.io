---
layout: post
title:  "Linux Security Configuration"
date:   2015-10-19
tags:   [linux, admin]
---

ssh
=======

Modify `/etc/ssh/sshd_config`:

```
# Disable root login
PermitRootLogin no

# Change the default port
Port 12345

# Enable login with key
RSAAuthentication yes
PubkeyAuthentication yes

# Disable login with password
UsePAM no
PasswordAuthentication no
```

After that, remember to restart sshd: `sudo /etc/init.d/ssh restart` 
(for Debian/Ubuntu) or `sudo service sshd restart` (for CentOS).

iptables
==========

*Refer to https://wiki.debian.org/iptables*

Create the file `/etc/iptables.test.rules`, and enter rules:

```
*filter

# Allows all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT

# Accepts all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allows all outbound traffic
# You could modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

# Allows HTTP and HTTPS connections from anywhere (the normal ports for websites)
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

# Allows SSH connections 
# The --dport number is the same as in /etc/ssh/sshd_config
-A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

# Now you should read up on iptables rules and consider whether ssh access 
# for everyone is really desired. Most likely you will only allow access from certain IPs.

# Allow ping
#  note that blocking other types of icmp packets is considered a bad idea by some
#  remove -m icmp --icmp-type 8 from this line to allow all kinds of icmp:
#  https://security.stackexchange.com/questions/22711
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

# log iptables denied calls (access via 'dmesg' command)
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

# Reject all other inbound - default deny unless explicitly allowed policy:
-A INPUT -j REJECT
-A FORWARD -j REJECT

COMMIT
```

Apply these rules:

```
sudo iptables-restore < /etc/iptables.test.rules
```

And see the difference:

```
iptables -L
```

Save the rules:

```
iptables-save > /etc/iptables.up.rules
```

Create file `/etc/network/if-pre-up.d/iptables`, and add these lines to it:

```bash
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.up.rules
```

Add executable permision for the file: 

`chmod +x /etc/network/if-pre-up.d/iptables`

To check who is listening on TCP port 12345:

```lsof -n -i4TCP:12345 | grep LISTEN```

nginx
=======

Modify the file nginx.conf:

```
http {
# Hide nginx version information
    server_tokens off;

# Catch all requests with wrong host and return 444 status
    server {
        listen 80 default_server;
        listen [::]:80 default_server;

        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;

        return 444;
    }
}
```
