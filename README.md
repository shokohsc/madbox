
# DNS SPOOF: This is a test and absolutely  wrong to do that !

# Pre-requisites

### Server
- root access
- ports 53/udp, 1194/udp, 80/tcp ,443/tcp available
- [Docker](https://docs.docker.com/get-docker/)

[EC2-AWS](https://serverfault.com/questions/836198/how-to-install-docker-on-aws-ec2-instance-with-ami-ce-ee-update)

- [Docker-compose](https://docs.docker.com/compose/install/)

### Client
- [Openvpn client](https://openvpn.net/)

## VPN setup [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn)

#### Choose if accessed from LAN or WAN then get the server ip/hostname:

Lookout for your NIC, Network Interface Controller, here eth0 is an example, maybe yours will be something else, find it with 'ifconfig'

Local Area Network IP

    $> export VPN_HOST=$(ifconfig eth0 | sed -En 's/127.0.0.1//;s/.*inet (addr:)?(([0-9]*\.){3}[0-9]*).*/\2/p')

Local Area Network Hostname

    $> export VPN_HOST=$(hostname)

Wide Area Network IP

    $> export VPN_HOST=$(curl -4 ifconfig.co)

Wide Area Network Hostname

    $> export VPN_HOST=$(dnsdomainname)

#### Choose spoof dns ip (which should be the same as $VPN_HOST ip):

Lookout for your NIC, Network Interface Controller, here eth0 is an example, maybe yours will be something else, find it with 'ifconfig' command

Local Area Network IP

    $> export SPOOF_DNS_IP=$(ifconfig eth0 | sed -En 's/127.0.0.1//;s/.*inet (addr:)?(([0-9]*\.){3}[0-9]*).*/\2/p')

Wide Area Network IP

    $> export SPOOF_DNS_IP=$(curl -4 ifconfig.co)

#### Choose a strong encryption

    $> export CIPHER="AES-256-CBC"

#### Create an openvpn configuration (1.1.1.1 is a secondary, backup dns, you could skip it)

    $> docker-compose run --rm openvpn ovpn_genconfig -u udp://$VPN_HOST:1194 -C $CIPHER -n $SPOOF_DNS_IP -n 1.1.1.1 -e 'push "redirect-gateway def1 bypass-dhcp"' -e 'push "comp-lzo no"'

### Next step requires to remember input passphrases

    $> docker-compose run --rm openvpn ovpn_initpki

#### Fix host permissions

    $> sudo chown -R $(whoami): ./config/openvpn

#### Start vpn

    $> docker-compose up -d openvpn

#### Create user configuration file without password, remove 'nopass' to add a password at connection

    $> export CLIENTNAME="your_client_name"
    $> docker-compose run --rm openvpn easyrsa build-client-full $CLIENTNAME nopass

#### Retrieve ovpn client file to import to your openvpn client

    $> docker-compose run --rm openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn

[link](https://github.com/kylemanna/docker-openvpn/issues/496) if cannot create client file

## Configure dns spoofing

Edit two files to replace SPOOFED_IP
- config/dnsmasq/dnsmasq.conf
- config/dnsmasq/spoof.hosts

## Start

    $> docker-compose up -d

- Connect to your vpn from your device by importing the client ovpn file.
- Browse to http://dnsmasq.madbox to watch domain requests
- Browse to http://traefik.madbox to watch traefik dashboard
- Browse to http://google.com to end up on nginx default index page

## Tear down

    $> docker-compose rm -sfv
    $> docker network rm madbox_default

# How long did it take ?

Done in one evening (4-5 hours) and a morning (3 hours)

# Why was it done like this ?
It basically goes like editing your /etc/hosts file on unix !

DNS Servers translate domain to ips, edit the dictionary of the dns server and you can redirect google.com to 127.0.0.1 exactly like adding '127.0.0.1 google.com' to your /etc/hosts file

OpenVPN has a configuration which forces client that connects to it to use a configured dns server and force all ipv4 traffic to pass through it, namely:

- "dhcp-option DNS SOME_DNS_IP"
- "redirect-gateway def1 bypass-dhcp"

Now we can make openvpn clients use the spoofed dns.

I used docker because it is fast, easily configurable & exportable anywhere

# How to test ?

Follow the readme and you're golden

# What went grong ?

I wasted time with an openvpn configuration useless in this case setting up One time password for openvpn clients, besides that, I already figured out what to do.

# What can we do now ?

So many things
- Monitor openvpn server, [grafana dashboard]([https://grafana.com/grafana/dashboards/10562](https://grafana.com/grafana/dashboards/10562))
- Monitor/Alerts Nginx incoming request for the spoof domain, [grafana dashboard]([https://grafana.com/grafana/dashboards/5063](https://grafana.com/grafana/dashboards/5063))
- Monitor Traefik metrics, [grafana dashboard]([https://grafana.com/grafana/dashboards/10479](https://grafana.com/grafana/dashboards/10479))
- Setup fake google.com web site and serve results from another search engine, i.e [qwant]([https://www.qwant.com](https://www.qwant.com))
- [Ansible]([https://github.com/ansible/ansible](https://github.com/ansible/ansible)) playbook to skip all manual steps
- [Docker swarm]([https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/)) for container & volumes High Availability
- Store container logs to elasticsearch via filebeat

# Comments

- It was fun !
- Learned about addn-hosts dnsmasq directive.
