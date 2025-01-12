# Part1 - Bind9 Local DNS Installation Guide IPv4

Do not use this guide to setup a public DNS server!!

This will get a single installation of bind9 up and running for a local lab.

This was done with a Nonexistent DNS name and can’t use commercial encryption certificates. But the easiest way to fix that is register a domain name and use a subdomain name `internal` such that `internal.example.com` would be used for internal dns routing. But in this example we are using a nonexistent domain.

This guide uses raspberry pi 3, but should work on other environments which have bind9 and are debian based.

This guide is only for setting up a internal dns for internal purposes.

Simply copying, pasting, and running the commands will not work. You will need to changed subnets, ips, and other values to make this work with your environment.



## Background Info

Nonexistent DNS names can’t use commercial encryption certificates, such as those ending in .local, .lab, etc. 

Register a domain name for your DNS service through a domain registrar

Zone - a bind zone is a segment of the DNS namespace that is managed by a specific DNS server. This server is responsible for providing authoritative answers for the domain names within that zone. Zones can be defined as either primary or secondary.

Primary Zone - The primary bind zone is the primary copy of the zone file. Changes to the DNS records are made here, and these changes are then propagated to secondary zones through zone transfers.

Secondary Zone - A secondary bind zone receives a copy of the zone data from the primary zone. It acts as a backup and can also provide authoritative answers for the zone, helping to distribute the load and improve reliability.

forward lookup

reverse lookup


## Prereqs
* Raspberry Pi or Ubuntu Linux (the new dns server)
* A local router with ipv4 dhcp 
* A reserved IPV4 ip on router and that ip is static on the dns server

## Install bind9

sudo apt install bind9 bind9utils bind9-doc

## Open ports from firewall on linux instance

## Configure Bind9
First you will want to start with just ipv4 and get that working. So we will force bind to use ipv4 and then later change that.
### 1. IPV4

#### 1.1 Default Server Settings
If using a debian or ubuntu based linux system you will need to turn off resolveconf
Setting `RESOLVCONF=no` in the `/etc/default/bind9` file can prevent resolvconf from interfering with BIND9’s operation. 
This configuration ensures that BIND9 does not attempt to modify /etc/resolv.conf through resolvconf, which leads to conflicts or issues 
if resolvconf is not properly configured.

In addition, we want to set the bind service to only allow ipv4 and exclude ipv6.

In terminal run to create the file.

    sudo tee /etc/default/bind9 > /dev/null << EOF
    # Prevent resolvconf from interfering with BIND9’s operation
    RESOLVCONF=no
    # Allow bind to start with ipv4 and exclude ipv6
    OPTIONS="-u bind -4"
    EOF


#### 1.2 Configuring Global Options
The `named.conf.options` file is where you set the configuration options for the overall dns server.
Typically you will define acl(s) and then apply options to the acl(s)
In this example, this will provide dns to a local lab and the network ip is 198.168.4.0.

    sudo tee named.conf.options > /dev/null << EOF
    acl localclients {
    192.168.4.0/24;
    localhost;
    localnets;
    };
    options {
    listen-on port 53 { 127.0.0.1; 192.168.4.141; };
    listen-on-v6 port 53 { "none"; };
    directory "/var/cache/bind";

    // If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
    // If you are building a RECURSIVE (caching) DNS server, you need to enable recursion.
    // If your recursive DNS server has a public IP address, you MUST enable access
    // control to limit queries to your legitimate users. Failing to do so will
    // cause your server to become part of large scale DNS amplification
    // attacks. Implementing BCP38 within your network would greatly
    // reduce such attack surface
    recursion no;
    
    allow-query { localclients; };
    
    // auth-nxdomain no is the default, ensuring the server behaves strictly according 
    // to whether it is authoritative.
    auth-nxdomain no;
    
    // If there is a firewall between you and nameservers you want
    // to talk to, you may need to fix the firewall to allow multiple
    // ports to talk. See http://www.kb.cert.org/vuls/id/800113
    // If your ISP provided one or more IP addresses for stable
    // nameservers, you probably want to use them as forwarders.
    // Uncomment the following block, and insert the addresses replacing
    // the all-0's placeholder.
    forwarders {
    8.8.8.8; // Google
    8.8.4.4; // Google
    };
    
    // Need to configure dynamic keys to use dnssec.
    //dnssec-enable yes;
    //========================================================================
    // If BIND logs error messages about the root key being expired,
    // you will need to update your keys. See https://www.isc.org/bind-keys
    //========================================================================
    dnssec-validation yes;

    };
    EOF

#### 1.2 Validate Global Options file
Bind provides a command which verifies server configuration for syntax errors but does not check the semantics or correctness of the configuration’s content.

    sudo named-checkconf

#### 1.3 Configuring Local Options - Part 1
The `/etc/bind/named.conf.local` file contains the local DNS server configuration, including zone declarations and other local settings. It is where you declare the zones associated with your domain and define their properties.

Because in our example this will be the main and only dns server, we will set the local options up to have a single primary zone. In addition, this is being organized such that a zone will manage a subnet of `192/168.0.0/16` but in reality we are only going to be adding records for the acl we defined above as `192.168.4.0/24`. 

    sudo tee /etc/bind/named.conf.local > /dev/null << EOF
    // Primary server for pitchblack408.lab
    zone "pitchblack408.lab" {
    type primary;
    file "/etc/bind/zones/pitchblack408.lab"; # zone file path
    // allow-transfer ; # ns2 private IP address - secondary
    };
    // Provides reverse mapping zone
    zone "168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/192.168"; # 192/168.0.0/16 subnet
    // allow-transfer ; # ns2 private IP address - secondary
    };
    EOF


#### 1.4 Configuring Local Options - Part 2
In the the step above we referenced where we are going to place the zone config files. Now we need to create that folder and the files.

First create the folder

    sudo mkdir /etc/bind/zones/

Create the dns config for the forward lookup

    sudo tee /etc/bind/zones/pitchblack408.lab > /dev/null << EOF
    ;
    ; BIND data file for pitchblack408.lab forward lookup
    ;
    \$TTL 604800
    @   IN  SOA r2d2.pitchblack408.lab. admin.pitchblack408.lab. (
        9         ; Serial
        604800    ; Refresh
        86400     ; Retry
        2419200   ; Expire
        604800 )  ; Negative Cache TTL
    ;
    ; Name servers - NS records
        IN  NS  r2d2.pitchblack408.lab.
    ;
    ; Name servers - A records
    r2d2 IN  A   192.168.4.141  ; r2d2 (nameserver)
    EOF

Validate the added zone by running:

    sudo named-checkzone pitchblack408.lab /etc/bind/zones/pitchblack408.lab

The results if the zone is fine:

    zone pitchblack408.lab/IN: loaded serial 9
    OK


Create the dns config for the reverse lookup. Remember that we are creating record as the 

    sudo tee /etc/bind/zones/192.168 > /dev/null << EOF
    ;
    ; BIND data file for pitchblack408.lab reverse lookup
    ;
    \$TTL 604800
    @ IN SOA r2d2.pitchblack408.lab. admin.pitchblack408.lab. (
    3 ; Serial
    604800 ; Refresh
    86400 ; Retry
    2419200 ; Expire
    604800 ) ; Negative Cache TTL
    ;
    ; name servers - NS records
    @ IN NS r2d2.pitchblack408.lab.

    ; PTR Records
    4.141 IN  PTR  r2d2.pitchblack408.lab.
    EOF

Reload server

    sudo systemctl restart bind9

Check status

    sudo systemctl status bind9


# Verify Results
On the dns server run dig and nslookup to verify results.

Verify reverse lookup

    dig @127.0.0.1 -x 192.168.4.141

Verify forward lookup

    nslookup r2d2.pitchblack408.lab 127.0.0.1


From a different PC on your network

    nslookup r2d2.pitchblack408.lab 192.168.4.141

If you tried from a windows machines it returns `Server:  UnKnown` but if it returns the ip address you are fine.

# Add dns to Devices
The dns server is working, but the other computers will not know about it automatically.
You will have to add the dns server(s) to ipv4 settings per each device. 
    
# Default logging
 By default, BIND9 has minimal logging enabled to avoid performance overhead, but it allows you to configure more detailed logging if needed. In modern systems, journalctl is the tool used to query and manage logs created by systemd-journald, the logging component of systemd. In legacy systems, BIND may use syslog to log messages unless otherwise configured. These logs are typically stored in system log files like /var/log/syslog, /var/log/messages, or /var/log/daemon.log.

    sudo journalctl -u named --follow

If you want to keep using systemd-journal, you might want to make sure the logs are persistent. By default the auto setting is used. In addition, by default systemd-journal log rotation is set.

    vi /etc/systemd/journald.conf

Then change the Storage to persistent.

    Storage=persistent

Restart service

    sudo systemctl restart systemd-journald

If you want to change bind logging configuration for logs going to systemd-journald, then it is best practice to create on config file for logging and add the include in the named.conf file.

Add the include reference to `/etc/bind/named.conf.logging`

    sudo tee /etc/bind/named.conf > /dev/null << EOF
    // This is the primary configuration file for the BIND DNS server named.
    //
    // Please read /usr/share/doc/bind9/README.Debian for information on the
    // structure of BIND configuration files in Debian, *BEFORE* you customize
    // this configuration file.
    //
    // If you are just adding zones, please do that in /etc/bind/named.conf.local

    include "/etc/bind/named.conf.options";
    include "/etc/bind/named.conf.local";
    include "/etc/bind/named.conf.default-zones";
    include "/etc/bind/named.conf.logging";
    EOF


Create the `/etc/bind/named.conf.logging` file

    sudo tee /etc/bind/named.conf.logging > /dev/null << EOF
    logging {
        channel default_syslog {
            print-time no;
            print-category yes;
            print-severity yes;
            syslog daemon;
            severity info;
        };

        category default { default_syslog; };
        category general { default_syslog; };
    };
    EOF


Restart service

    sudo systemctl restart systemd-journald

Documentation with configuration examples of logging to files for logical separation of output.

* https://kb.isc.org/docs/aa-01526
* https://wiki.debian.org/Bind9


# Securing DNS
The goal of this document was to create an internal dns for a lab. But even so, you might want to create some additional acls and block from certain sources. In addition, create a registered domain and used that name instead of the Nonexistent DNS name.

Adding blocking acls

    sudo tee named.conf.options > /dev/null << EOF
    acl bogusnets {
        0.0.0.0/8;  192.0.2.0/24; 224.0.0.0/3;
        10.0.0.0/8; 172.16.0.0/12;
    };
    acl localclients {
    192.168.4.0/24;
    localhost;
    localnets;
    };
    options {
    listen-on port 53 { 127.0.0.1; 192.168.4.141; };
    listen-on-v6 port 53 { "none"; };
    directory "/var/cache/bind";

    // If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
    // If you are building a RECURSIVE (caching) DNS server, you need to enable recursion.
    // If your recursive DNS server has a public IP address, you MUST enable access
    // control to limit queries to your legitimate users. Failing to do so will
    // cause your server to become part of large scale DNS amplification
    // attacks. Implementing BCP38 within your network would greatly
    // reduce such attack surface
    recursion no;

    blackhole { bogusnets; };
    allow-query { localclients; };

    // auth-nxdomain no is the default, ensuring the server behaves strictly according 
    // to whether it is authoritative.
    auth-nxdomain no;

    //========================================================================
    // If there is a firewall between you and nameservers you want
    // to talk to, you may need to fix the firewall to allow multiple ports to talk. 
    // See http://www.kb.cert.org/vuls/id/800113
    // If your ISP provided one or more IP addresses for stable nameservers, 
    // you probably want to use them as forwarders.
    //========================================================================
    forwarders {
    8.8.8.8; // Google
    8.8.4.4; // Google
    };

    // Need to configure dynamic keys to use dnssec.
    dnssec-enable yes;
    //========================================================================
    // If BIND logs error messages about the root key being expired,
    // you will need to update your keys. See https://www.isc.org/bind-keys
    //========================================================================
    dnssec-validation yes;

    };
    EOF


If this was a public dns you would have to secure the server from dns poisoning by implementing dnssec.

* https://bind9.readthedocs.io/en/v9.18.14/chapter5.html

Also, if you were going to make this a public dns, you would have risk of ddos attacks and need implement preventive measures, high availability, and disaster recovery.


# Conclusion
This is a short exercise will help one get started with a local install of dns, but there is much more to read and work with in dns. In the future, I plan on making a part 2 for this lab which is adding configuration for ipv6.


# Sources
* https://wiki.debian.org/Bind9
* https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services#proc_configuring-bind-as-a-caching-dns-server_assembly_setting-up-and-configuring-a-bind-dns-server
* https://bind9.readthedocs.io/en/v9.18.1/security.html
* https://bind9.readthedocs.io/en/v9.18.14/chapter3.html
* https://kb.isc.org/docs/aa-01526
* https://wiki.debian.org/Bind9
* https://www.doscher.com/work-local-dns/
* https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services#proc_updating-a-bind-zone-file_assembly_configuring-zones-on-a-bind-dns-server
* https://www.digitalocean.com/community/tutorials/how-to-setup-dnssec-on-an-authoritative-bind-dns-server-2