# SRE Challenge:
Challenge:
Using a modern configuration management tool of your choice (i.e. Ansible, Chef, Puppet, SaltStack, etc.) configure a Linux host as a secure web server. The server should be configured to serve a page reading “SRE CHALLENGE” over HTTPS.
Deliverables:
Install and configure a web server on a Debian, Ubuntu, or CentOS host
Configure https (a self-signed certificate is acceptable for this challenge)
Configure http redirect to https
Configure the http/https homepage to display "SRE CHALLENGE"
Configure host-based firewall for ports 22, 80 and 443
Create an automated test to verify the web server is listening on port 443.
Place the project on GitHub using a generic name like “sre-challenge” (please do not include “Comcast” in your repo name or project)
Send the GitHub URL to the team for review

# Solution:
I'm going to use puppet to do this configuration automation with Docker in my local laptop. This solution created a new httpd module to handle the apache httpd installation and configuration for virtual host. It relied on a module firewall from puppetlabs https://forge.puppet.com/puppetlabs/firewall for the firewall configuration. I tested it in centos 7.
## Assumptions:
### Have Docker installed in local host. If not, please go to docker.com https://www.docker.com and download the docker engine and install it.

## Steps:
### Create puppetmaster as below:
```
docker pull rongyj/puppet-agent4-c7-systemd
cd ~/git/sre-challenge/
mkdir puppet-data

docker run  -d --name puppetmaster --cap-add SYS_ADMIN --security-opt seccomp:unconfined --hostname puppetmaster -p 8443:443 -p 8080:80 -v ~/git/sre-challenge/puppet-data/puppet/etc/agent/hosts:/etc/hosts -v ~/git/sre-challenge/puppet-data/puppet/etc/agent/puppet:/etc/puppet   -v ~/git/sre-challenge-new/creating-httpd-module:/opt/puppet -v ~/git/sre-challenge/puppet-data/varpuppet:/opt/varpuppet --cap-add=NET_ADMIN --cap-add=NET_RAW rongyj/puppet-agent4-c7-systemd bash -c "/usr/sbin/init"

docker run -d --name puppetmaster --cap-add SYS_ADMIN --security-opt seccomp:unconfined -h puppet -p 8140:8140 -v ~/git/sre-challenge/puppet-data/puppet:/opt/puppet -v ~/git/sre-challenge/puppet-data/varpuppet:/opt/varpuppet  --cap-add=NET_ADMIN --cap-add=NET_RAW rongyj/puppetserver4-c7-systemd /bin/bash -c "/usr/sbin/init"

docker exec -it puppetmaster /bin/bash
docker exec puppetmaster cp -Rf /etc/puppet /opt/

rpm -Uvh https://yum.puppetlabs.com/puppetlabs-release-pc1-el-6.noarch.rpm
yum -y install puppetserver
```
### Test it with puppetagent docker container
#### Git clone SRE-Challenge repo from GitHub
```
cd ~; git clone https://github.com/rongyj/sre-challenge.git
```
#### Pull docker image for puppet agent
```
docker pull rongyj/puppet-agent4-c7-systemd
```
#### Run puppetagent docker container with right mapping for the port and volumes
```
docker run  -d --name puppetagent --cap-add SYS_ADMIN --security-opt seccomp:unconfined --hostname puppetagent -p 8443:443 -p 8080:80 -v ~/sre-challenge-new/creating-httpd-module/agent/hosts:/etc/hosts -v ~/sre-challenge-new/creating-httpd-module:/etc/puppet --cap-add=NET_ADMIN --cap-add=NET_RAW rongyj/puppet-agent4-c7-systemd bash -c "/usr/sbin/init"
```
It will map the local port 8443 to the docker container port 443 and local port 8080 to container port 80.
####
[root@puppetagent tests]# pwd
/opt/puppet/tests
[root@puppetagent tests]# puppet apply --modulepath=/opt/puppet/modules --debug --verbose test.pp

/sbin/iptables -R INPUT 1 -t filter -p tcp -m multiport --dports 22,80,443 -m comment --comment "100 allow ssh, http and https access" -j ACCEPT
```

[root@puppetagent tests]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     icmp --  anywhere             anywhere             /* 000 accept all icmp */
ACCEPT     all  --  anywhere             anywhere             /* 001 accept all to lo interface */
ACCEPT     all  --  anywhere             anywhere             /* 003 accept related established rules */ state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere             multiport dports ssh,http,https /* 300 allow ssh, http and https access */ state NEW
ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     icmp --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            
ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:ssh
REJECT     all  --  anywhere             anywhere             reject-with icmp-host-prohibited
DROP       all  --  anywhere             anywhere             /* 999 drop all */

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
REJECT     all  --  anywhere             anywhere             reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     all  --  anywhere             anywhere             /* 004 accept related established rules */ state RELATED,ESTABLISHED
ACCEPT     udp  --  anywhere             anywhere             multiport dports domain /* 200 allow outgoing dns lookups */ state NEW
ACCEPT     icmp --  anywhere             anywhere             /* 200 allow outgoing icmp type 8 (ping) */ icmp echo-request

## Cento 6 Docker container issues
Error: Modifying the chain for existing rules is not supported.
Error: /Stage[main]/Main/Httpd::Vhost[puppetagent]/Firewall[100 allow http access]/chain: change from 80 to INPUT failed: Modifying the chain for existing rules is not supported.

FATAL: Could not load /lib/modules/4.9.13-moby/modules.dep: No such file or directory
