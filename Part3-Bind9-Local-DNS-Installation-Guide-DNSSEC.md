# Part3 - Bind9 Local DNS Installation Guide - DNSSEC

Do not use this guide to setup a public DNS server!!

This is a fallow up on "Part2-Bind9-Local-DNS-Installation-Guide-IPv4-IPv6"

This will get a single installation of bind9 up and running for a local lab.

This was done with a Nonexistent DNS name and can’t use commercial encryption certificates. But the easiest way to fix that is register a domain name and use a subdomain name `internal` such that `internal.example.com` would be used for internal dns routing. But in this example we are using a nonexistent domain.

This guide uses raspberry pi 3, but should work on other environments which have bind9 and are debian based.

This guide is only for setting up a internal dns for internal purposes.

Simply copying, pasting, and running the commands will not work. You will need to changed subnets, ips, and other values to make this work with your environment.


## Prereqs
* Raspberry Pi or Ubuntu Linux Box
* IPV6 is turn on local router
* Completed "Part2-Bind9-Local-DNS-Installation-Guide-IPv4-IPv6"

## Background
DNSSEC is a set of extensions that enhance the security of the Domain Name System (DNS) by ensuring that DNS responses have not been altered. Organizations can implement DNSSEC to strengthen their DNS security.

DNS technology was not originally designed with security in mind. A common attack on DNS infrastructure is DNS spoofing, where an attacker manipulates a DNS resolver’s cache. This causes users to be redirected to a malicious site, displaying an incorrect IP address instead of the intended website.

BIND (Berkeley Internet Name Domain) package includes support for DNSSEC (Domain Name System Security Extensions). DNSSEC is a suite of extensions that adds security to the DNS by enabling domain name servers to sign their data cryptographically, which helps ensure the authenticity and integrity of DNS responses.

Key Features of DNSSEC with BIND:

* Key Management - BIND supports generating, managing, and rotating DNSSEC keys, such as KSK (Key Signing Key) and ZSK (Zone Signing Key).

* Zone Signing - You can sign DNS zones using tools provided in the BIND suite, such as dnssec-signzone.

* Validation - BIND supports validating DNSSEC-signed responses when acting as a resolver.

* Automatic Key and Zone Management: With tools like rndc and configuration options in named.conf, BIND can manage DNSSEC operations automatically.

* RFC Compliance - Red Hat BIND is regularly updated to comply with DNSSEC-related RFCs and standards.

For more information on 

## Setting Up DNSSEC on Bind
### 1.1 Update options configuration 
Set `dnssec-enable` and  `dnssec-validation` both to yes in your configuration options file.

Note: you will need to change ipv4 and ipv6 values to match your environment

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
    
    //========================================================================
    // If BIND logs error messages about the root key being expired,
    // you will need to update your keys. See https://www.isc.org/bind-keys
    //========================================================================
    dnssec-enable yes;
    dnssec-validation yes;

    };
    EOF
 
## Generate Keys:
Use dnssec-keygen to generate the required keys.

    dnssec-keygen -a RSASHA256 -b 4096 -n ZONE pichblack408.lab

Sign the Zone:

Use dnssec-signzone to sign your zone file.

    dnssec-signzone -A -3 randomstring -o pichblack408.lab pichblack408.lab.db

## Update Zone Configuration:
Update the named.conf file to load the signed zone.

## Testing:
Use tools like dig to verify DNSSEC functionality.

## Verifying Installation
To confirm that BIND was built with DNSSEC support, run:

named -V

The output should include --enable-dnssec.


# Important
Again this guide is to get a internal DNS created, but if you wanted to allow external DNS server to verify the zone you would need to have that external DNS configured. 
    
    "To enable external DNS servers to verify the authenticity of a zone, you must add the public key of the zone to the parent zone. Contact your domain provider or registry for further details on how to accomplish this."


# Conclusion



# Sources
* https://www.talkdns.com/articles/a-beginners-guide-to-dnssec-with-bind-9/
* https://www.talkdns.com/articles/
* https://docs.redhat.com/fr/documentation/red_hat_enterprise_linux/7/html/networking_guide/sec-BIND#sec-bind-features-dnssec
* https://wiki.debian.org/Bind9
* https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services#proc_configuring-bind-as-a-caching-dns-server_assembly_setting-up-and-configuring-a-bind-dns-server
* https://bind9.readthedocs.io/en/v9.18.1/security.html
* https://bind9.readthedocs.io/en/v9.18.14/chapter3.html
* https://kb.isc.org/docs/aa-01526
* https://wiki.debian.org/Bind9
* https://www.doscher.com/work-local-dns/
* https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services#proc_updating-a-bind-zone-file_assembly_configuring-zones-on-a-bind-dns-server
* https://www.digitalocean.com/community/tutorials/how-to-setup-dnssec-on-an-authoritative-bind-dns-server-2

create a keys directory

    sudo mkdir /etc/bind/keys
