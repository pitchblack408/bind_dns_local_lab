# Part1 - Bind9 Local DNS Installation Guide IPv4 & IPv6

Do not use this guide to setup a public DNS server!!

This is a fallow up on "Part1 Bind9 Local DNS Installation Guide IPv4"

This guide will attempt to add ipv6 to the previous set up that was made in the "Part1 Bind9 Local DNS Installation Guide IPv4" document.

This guide is only for setting up a internal dns for internal purposes.

## Prereqs
* Raspberry Pi or Ubuntu Linux (the new dns server)
* A local router with ipv4 dhcp 
* Completed "Part1 Bind9 Local DNS Installation Guide IPv4"
* IPV6 is turn on local router
* Reserve a IPV6 ip on router using the server's mac address


## Determine subnet from ipv6 
Find the interface

    nmcli device show

List the ips associated to interface

    nmcli device show <interface>


* Global unique addresses (GUA): Start with 2 or 3 in the first hexadecimal digit.
    * These addresses are globally routable on the internet. The prefix for these addresses typically starts with 2000::/3 (i.e., addresses starting with 2 or 3).

* Link local addresses (LLA): Start with fe80 in the first four hexadecimal digits.
    * These addresses are only valid within the local link (i.e., the local network segment). They cannot be routed across networks.

* Unique local addresses (ULA): Start with fc00 or fd00 in the first four hexadecimal digits.
    * These addresses are similar to private IPv4 addresses and are used for local networks that do not require global routing.
    * ULAs are now the recommended version of private addressing for IPv6 and are used for communication within a private network.


Your router typically provides a global unicast address (GUA) subnet from your ISP or assigns a unique local address (ULA) for private use.
The router will advertise these subnets to devices on your network using protocols like SLAAC or DHCPv6. Since the DNS lookups in this guide are limited to a single local network segment, you can rely on the ULA IPv6 address. GUA is for traffic requiring internet access or external communication.

In addition, the global ip functions exactly as named, if anyone obtains your global ip they will be able to make attempts to access that ip unless firewall restrictions are in place. You can use external websites to ping your global ip and even port scan your global ip.

In this guide we will be not be using the GUAs.

## Configure Bind9
Note that these sections are not a manner of just copy, paste, and run. You will need to changed ipv4 subnets, ips, and other values to work with your environment. The following was setup in in a template manner, but you will need to modify it to match your environment.

### 1.1 Default Server Settings
In the last guide the ipv6 was not allow and that was set in the server's default settings, 
which now need to be changed to allow both ipv4 and ipv6.

In terminal run:

    sudo tee /etc/default/bind9 > /dev/null << EOF
    # Prevent resolvconf from interfering with BIND9’s operation
    RESOLVCONF=no
    EOF


#### 1.2 Configuring Global Options
The `named.conf.options` file is where you set the configuration options for the overall dns server.
Typically you will define acl(s) and then apply options to the acl(s)


    sudo tee named.conf.options > /dev/null << EOF
    acl localclients {
    192.168.4.0/24;
    localhost;
    localnets;
    // 2001:db8::/64;    # IPv6 subnet; DON'T add dns server's global ipv6 address range unless you plan to expose dns to the internet
    fc00::/7;            # Unique Local address range (ULA)
    };
    options {
    listen-on port 53 { 127.0.0.1; 192.168.4.141; };
    listen-on-v6 port 53 { 
        ::1; 
        # DON'T add dns server's global ipv6 address unless you plan to expose dns to the internet
        # DON'T add your dns server's Link-local address here.
        # Unique Link Address here
    };
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
    2001:4860:4860::8888; // Google
    2001:4860:4860::8844; // Google
    };
    // As a fall-back behavior, BIND resolves queries recursively if the 
    // forwarder servers do not respond. Disabling this behavior
    forward only;
    
    // Need to configure dynamic keys to use dnssec.
    //dnssec-enable yes;
    //========================================================================
    // If BIND logs error messages about the root key being expired,
    // you will need to update your keys. See https://www.isc.org/bind-keys
    //========================================================================
    dnssec-validation yes;

    };
    EOF

#### 1.3 Validate Global Options file
Bind provides a command which verifies server configuration for syntax errors but does not check the semantics or correctness of the configuration’s content.

    sudo named-checkconf


#### 1.4 Configuring Local Options - Routes Part 1
The `/etc/bind/named.conf.local` file contains the local DNS server configuration, including zone declarations and other local settings. It is where you declare the zones associated with your domain and define their properties.

Because in our example this will be the main and only dns server, we will set the local options up to have a single primary zone and comment out how the secondary dns. In addition, this is being organized such that a zone will manage a subnet of `192/168.0.0/16` but in reality we are only going to be adding records for the acl we defined above as `192.168.4.0/24`. 

In addition the ipv6 unique local range fd84:f006:1a18::/64 and or IPv6 local link range (fe80::/10) for reverse mapping are added.

    sudo tee /etc/bind/named.conf.local > /dev/null << EOF
    // Primary server for pitchblack408.lab
    zone "pitchblack408.lab" {
        type primary;
        file "/etc/bind/zones/pitchblack408.lab"; // zone file path
        // allow-transfer ; // ns2 private IP address - secondary
    };

    // Provides reverse mapping zone for IPv4
    zone "168.192.in-addr.arpa" {
        type primary;
        file "/etc/bind/zones/192.168"; // 192.168.0.0/16 subnet
        // allow-transfer ; // ns2 private IP address - secondary
    };

    // Provides reverse mapping zone for IPv6 unique local range (fd84:f006:1a18::/64)
    zone "0.6.f.0.6.f.d.ip6.arpa" {
        type primary;
        file "/etc/bind/zones/fd84f0061a18"; // zone file path for fd84:f006:1a18::/64
    };
    EOF


#### 1.5 Configuring Local Options - Routes Part 2
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
    r2d2 IN  AAAA ; add your ipv6 unique local address here
    ; DON'T add your ipv6 local link address here because there can be a duplicate
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


    sudo tee /etc/bind/zones/fd84f0061a18 > /dev/null << EOF
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
    ; IPv6 reverse mapping for fd84:f006:1a18::/64
    ; For the IPv6 address fd84:f006:1a18:1:e2e3:3f41:84a7:f0e4, the reverse address would be "e.f.0.4.a.8.4.1.f.3.f.2.e.1.1.a.6.f.0.6.f.4.d.8.f.0.0.0.0.0.8.f.d"
    e.f.0.4.a.8.4.1.f.3.f.2.e.1.1.a.6.f.0.6.f.4.d.8.f.0.0.0.0.0.8.f.d   IN PTR r2d2.pitchblack408.lab.
    EOF


Reload server

    sudo systemctl restart bind9

Check status

    sudo systemctl status bind9


# Verify Results
On the dns server run dig and nslookup to verify results.



Verify IPV4 reverse lookup

    dig @127.0.0.1 -x 192.168.4.141

Verify IPV4 forward lookup

Run

    nslookup r2d2.pitchblack408.lab 127.0.0.1

Run

    nslookup r2d2.pitchblack408.lab 192.168.4.141

Verify IPV6 reverse lookup

Run

    dig @::1 -x fd84:f006:1a18:1:e2e3:3f41:84a7:f0e4

Verify IPV6 forward lookup

Run

    nslookup r2d2.pitchblack408.lab ::1

Run

    nslookup r2d2.pitchblack408.lab fd84:f006:1a18:1:e2e3:3f41:84a7:f0e4


From a different PC on your network test
Verify IPV4

Run

    nslookup r2d2.pitchblack408.lab 192.168.4.141

Verify IPV6

Run

    nslookup r2d2.pitchblack408.lab fd84:f006:1a18:1:e2e3:3f41:84a7:f0e4

If you tried from a windows machines it returns `Server:  UnKnown` but if it returns the ip address you are fine.


# Conclusion
This is a short exercise will help one get started with a local install of dns with ipv4 and ipv6 using the Unique Link Address as the subnet, but there is much more to read and work with in dns. In the future, I plan on making a part 3 which main focus will be dnssec.


# Sources
* https://en.wikipedia.org/wiki/IPv6
* https://en.wikipedia.org/wiki/Unique_local_address
* https://www.networkworld.com/article/964600/ipv6-address-enterprises-how-to-easier-than-ipv4.html
* https://wiki.debian.org/Bind9
* https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services#proc_configuring-bind-as-a-caching-dns-server_assembly_setting-up-and-configuring-a-bind-dns-server
* https://bind9.readthedocs.io/en/v9.18.1/security.html
* https://bind9.readthedocs.io/en/v9.18.14/chapter3.html
* https://kb.isc.org/docs/aa-01526
* https://wiki.debian.org/Bind9
* https://www.doscher.com/work-local-dns/
* https://www.pitchblack408.com -->
