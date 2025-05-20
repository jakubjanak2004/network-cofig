# Network configurations cheat sheet

This document provides instructions for setting up and managing a computer network, including configuration methods for Debian-based Linux systems.

## Firewall

Firewalling in iptables:

Install the iptabls:
```bash
sudo apt install iptables
```

Chains:
1. `INPUT`: Incomming to your system.
2. `OUTPUT`: Outgoing from my system
3. `FORWARD`: Through the system.
4. `PREROUTING`: Before the routing(NAT). 
5. `POSTROUTING`: After the routing.

Default policies:
1. ACCEPT: Let it pass.
2. DROP: Silently discard the packet.

Set defualt policy for chain:
```bash
sudo iptables -P {CHAIN} {POLICY}
```

Show the firewall configurations:
```bash
sudo iptables -L -v -n
```

Show the NAT configurations:
```bash
sudo iptables -t nat -L -v -n
```

Show the configurations with the line number:
```bash
sudo iptables -L -v -n --line-numbers
```

Delte the line at number:
```bash
sudo iptables -D {CHAIN} {LINE NUMBER}
```

Actions:
1. ACCEPT: Allows the packet to pass through.
2. DROP: Silently discards the packet (no reply sent).
3. REJECT: Drops the packet and sends an ICMP or TCP reset response.
4. LOG: Logs the packet details to syslog (does not affect packet flow).
...

Allow related and established connections:
```bash
sudo iptables -A {CHAIN} -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Add rule to allow protocol on input with certain destination port.
```bash
sudo iptables -A {CHAIN} -p {PROTOCOL} --dport {DESTINATION PORT} -j {ACTION}
```

More specifications for the command:
```bash
sudo iptables -t {TABLE} -A {CHAIN} -p {PROTOCOL} -s {SOURCE_IP} --sport {SOURCE_PORT} -d {DEST_IP} --dport {DEST_PORT} -j {ACTION}
```

Configure NAT:
```bash
sudo iptables -t nat -A POSTROUTING -o {INTERFACE} -j MASQUERADE
```
> 📝 **Note:** Replace `{INTERFACE}` with the name of your **external network interface**, such as `enp0s8`, `enp0s9`, `enp0s10`, etc.  

## Certificates

Here I will be focusing on how the certificates work and how to create them. I will be using `easy-rsa`.

### Initialize the `easy-rsa`
Install `easy-rsa`.
```bash
sudo apt install easy-rsa
```

Create and initialize new PKI directory
```bash
make-cadir ~/easy-rsa
cd ~/easy-rsa
./easyrsa init-pki
```

### Create CA
```bash
./easyrsa build-ca
```

### Create Certifcate
Create certificate request:
```bash
./easyrsa gen-req {CERTIFICATE NAME} nopass
```

Sign the certificate:
```bash
./easyrsa sign-req server {CERTIFICATE NAME}
```

### Output Files:
* `pki/ca.crt` - your Certificate Authority certificate.
* `pki/private/ca.key` — The private key of your Certificate Authority (keep this secret).
---
* `pki/issued/{CERTIFICATE NAME}.crt` - the signed certificate.
* `pki/private/{CERTIFICATE NAME}.key` — the certificate private key.
* `pki/reqs/{CERTIFICATE NAME}.req` — The certificate signing request (CSR) that was signed by the CA.
* `pki/index.txt` — A database tracking issued certificates.

### HTTPS Certificate creation
```bash
./easyrsa gen-req {CERTIFICATE NAME} nopass
openssl x509 -req   -in pki/reqs/{CERTIFICATE_NAME}.req   -CA pki/ca.crt   -CAkey pki/private/ca.key   -CAcreateserial   -out pki/issued/{CERTIFICATE_NAME}.crt   -days 825   -sha256   -extfile config.cnf   -extensions web_server
```

The config:
```cnf
[web_server]
subjectAltName = @alt_names

[alt_names]
DNS.1 = {DOMAIN NAME}
```

## DNS

The bind 9 configurations.

At first downlaod the bind9 server:
```bash
sudo apt install bind9 bind9utils bind9-doc dnsutils
```

Important configurations files:
* `/etc/bind/named.conf` - Main config file
* `/etc/bind/named.conf.options` - Set forwarders, recursion, etc.
* `/etc/bind/named.conf.local` - Zone definitions

The `/etc/bind/named.conf.local`, when adding the views you need to comment out the including of the default zones:
```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
//include "/etc/bind/named.conf.default-zones";
```

The `/etc/bind/named.conf.options`, here the important is the forwarding to the `8.8.8.8` DNS server. 
If the record is not found locally it is going to be forwarded.
```
options {
	directory "/var/cache/bind";
	recursion yes;
    	allow-query { any; };

    	forwarders {
        	8.8.8.8;
        	8.8.4.4;
    	};

    	dnssec-validation auto;

    	auth-nxdomain no;    # conform to RFC1035
    	listen-on { any; };
	
	listen-on-v6 { any; };
};
```

The `/etc/bind/named.conf.local` shows a configuration of views. Meaning returnig different responses for requests from different ip networks.  
The `zone` configuration shows where the file with configurations for particular zone is.
```
view "from-100" {
    match-clients { 192.168.100.0/24; };
    recursion yes;
    zone "test" {
        type master;
        file "/etc/bind/db.test.100";
    };

    zone "test2" {
        type master;
        file "/etc/bind/db.test2";
    };

    zone "0.8.10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.10.8.0";
    };
};

view "from-10" {
    match-clients { 192.168.10.0/24; };
    recursion yes;
    zone "test" {
        type master;
        file "/etc/bind/db.test.10";
    };

    zone "test2" {
        type master;
        file "/etc/bind/db.test2";
    };

    zone "10.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.192.168.10";
    };
};
```

The records:
* `A` - Maps domain name to the IP address.
* `MX` - Tells where to deliver mail.
* `PTR` - Maps IP address to the domain name(reverse to the A record).

The `/etc/bind/db.test2` as an example:
```
$TTL    604800
@       IN      SOA     ns1.test2. admin.test2. (
                        1 ; Serial
                        604800     ; Refresh
                        86400      ; Retry
                        2419200    ; Expire
                        604800 )   ; Negative Cache TTL

        IN      NS      ns1.test2.
ns1     IN      A       192.168.10.12
@       IN      A       192.168.10.12
@       IN      MX 10   192.168.10.12
sub     IN      A       192.168.10.13
```

And for the `PTR` records configuration:
```
$TTL    604800
@       IN      SOA     ns1.test2. admin.test2. (
                          1         ; Serial
                          604800     ; Refresh
                          86400      ; Retry
                          2419200    ; Expire
                          604800 )   ; Negative Cache TTL

        IN      NS      ns1.test2.

11      IN      PTR     test.
12      IN      PTR     test2.
```

## VPN

### The server configuration:
```conf
port 1194
proto udp
dev tun

ca ca.crt
cert server.crt
key server.key
dh dh.pem

server 10.8.0.0 255.255.255.0
topology subnet

push "redirect-gateway"
keepalive 10 60

cipher AES-256-CBC
auth SHA256

persist-key
persist-tun

user nobody
group nogroup

status openvpn-status.log
log-append /var/log/openvpn.log
verb 3
```

There needs to be `ca.crt`, `server.crt`, `server.key` and `dh.pem`.

### The client configuration:
```conf
client
dev tun
proto udp
remote 192.168.100.11 1194  # Replace with your server's IP or domain
resolv-retry infinite
nobind

cipher AES-256-CBC
verb 3

user nobody
group nogroup

persist-key
persist-tun

# Crypto and certs
ca ca.crt
cert client1.crt
key client1.key
ns-cert-type server
```

There needs to be ...

## Mailserver

## Monitoring system
