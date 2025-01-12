# Part3 - Bind9 Local DNS Installation Guide - DNSSEC

Do not use this guide to setup a public DNS server!!

This will get a single installation of bind9 up and running for a local lab.

This was done with a Nonexistent DNS name and can’t use commercial encryption certificates. But the easiest way to fix that is register a domain name and use a subdomain name `internal` such that `internal.example.com` would be used for internal dns routing. But in this example we are using a nonexistent domain.

This was done on raspberry pi 3, but should work on other environments which have bind9 and are debian based.


## Prereqs
* Raspberry Pi or Ubuntu Linux Box
* Completed "Part1 Bind9 Local DNS Installation Guide IPv4"
* IPV6 is turn on local router ( without this set on the router, the box you are using will set its own ipv6  )
* Statically set IPV6 ips on router


## Background
Red Hat's BIND (Berkeley Internet Name Domain) package includes support for DNSSEC (Domain Name System Security Extensions). DNSSEC is a suite of extensions that adds security to the DNS by enabling domain name servers to sign their data cryptographically, which helps ensure the authenticity and integrity of DNS responses.

Key Features of DNSSEC with BIND:

* Key Management - BIND supports generating, managing, and rotating DNSSEC keys, such as KSK (Key Signing Key) and ZSK (Zone Signing Key).

* Zone Signing - You can sign DNS zones using tools provided in the BIND suite, such as dnssec-signzone.

* Validation - BIND supports validating DNSSEC-signed responses when acting as a resolver.

* Automatic Key and Zone Management: With tools like rndc and configuration options in named.conf, BIND can manage DNSSEC operations automatically.

* RFC Compliance - Red Hat BIND is regularly updated to comply with DNSSEC-related RFCs and standards.

## Setting Up DNSSEC

create a keys directory

    sudo mkdir /etc/bind/keys


Enable DNSSEC validation in your configuration (/etc/named.conf):

    options {
        dnssec-enable yes;
        dnssec-validation yes;
    };

## Generate Keys:
Use dnssec-keygen to generate the required keys.

    dnssec-keygen -a RSASHA256 -b 2048 -n ZONE example.com

Sign the Zone:

Use dnssec-signzone to sign your zone file.

    dnssec-signzone -A -3 randomstring -o example.com example.com.db

## Update Zone Configuration:
Update the named.conf file to load the signed zone.

## Testing:
Use tools like dig to verify DNSSEC functionality.

## Verifying Installation
To confirm that BIND was built with DNSSEC support, run:

named -V

The output should include --enable-dnssec.

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
