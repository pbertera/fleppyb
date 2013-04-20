Fleppyb (Flexible PowerDNS Python Backend)
==========================================

Fleppyb is a [PowerDNS pipe backend][1]. In brief Fleppyb process DNS query from the PowerDNS master process and according with matching rules can resolve the query.
Parameters checked during the matching rule process are these provided by the **PipeBackend v.2** protocol:

**Query Name:** the name whereby the query asks, for example the query performed by

    dig @192.168.2.20 www.google.it

the Query Name will be www.google.it

**Query Type:** the record type requested by the query, for example:

    dig @192.168.2.20 -t MX google.it

the Query Type will be **MX**

**Remote IP:** the IP address that performed the query:

    dig -b 192.168.3.11 @192.168.2.20 -t MX google.it

the Remote IP will be **192.168.3.11**

**Local IP:** the local IP address that received the query:

    dig @192.168.2.20 www.google.it

the Local IP will be **192.168.2.20**

With this Fleppyb you can implement a *split brain* configuration or some other match against the query parameters.

Features
========

At the moment of writing Fleppyb implements 2 backend:
* **static**: a static backend that allows you to configure your dns answer in a config file, you can also add a dealy in some answers.
* **ldap**: an ldap backend that allow you to be totally independent from your ldap schema, you can use more than one ldap server and configure differents LDAP query for different DNS requests

Configuration
=============

There is 3 steps:

1. Configuring Fleppyb parameters
2. Configuring Flappyb matching rules
3. Configuring PowerDNS in order to use Fleppyb as a backend

Configuring Fleppyb parameters
----------------------

You must edit the main file fleppyb.py configuring these variables: 

* **DEBUG:** enable/disable debugging messages, valid values: **True** or **False**
* **CONFIG_FILE:** path of config file, the config file contains matching rules. Eg. **/etc/powerdns/fleppyb/fleppyb.conf**
* **PARSE_CONFIG_ONCE:** parse the config file once (at startup) or at every query. This parameter can slow down performance in query resolutions. Using *True* you need to restart PowerDNS ad every Fleppyb config file changes. Valid values: **True** or **False**
* **LOGFILE:** where Fleppyb must write logging messages. You can use a file system path or a void value (""). With a void value logging messages will be printed on stdout (useful for testing or debugging). **NB: This file path must exists and must be writable by PowerDNS user**.

Configuring query matching
--------------------------

The matching rules file (defined bu variable `CONFIG_FILE`) uses [file ini syntax][2]: you must define sections and section variables.

You can define a section with name **DEFAULT**: In this section you can define variables that will be inherited by others sections.
Variable defined in **DEFAULT** section can be overwritten in non-default section.

The name used in sections define the query matching criteria.
Section name must respect this syntax:

**[PREC;QUERY_NAME_REGEX:QUERY_TYPE:REMOTE_IP:LOCAL_IP]**

Where:

* **PREC:** is the rule precedence: matching rules is an ordered list, first rule that match with query parameters will be used for the query resolution.
* **QUERY_NAME_REGEX:** it's a regular expression that will be evalued against the *Query Name*. Here you can use the [Python regular expression syntax][3].
* **QUERY_TYPE:** this value will be matched against the *Query Type*, (A, TXT, MX, etc..). You can use the special char * in order to match every query type. 
* **REMOTE_IP:** this value will be evaluated against the client IP address requesting the query, can be an IP address or a subnet in CIDR format (Eg.: 192.168.2.0/24)
* **LOCAL_IP:** this value will be evaluated against the destination IP address. Alsto this one can be an IP address or a subnet.

Examples
--------

**[1:.*.in-addr\.arpa$:*:0.0.0.0/0:0.0.0.0/0]** With this section name you define:

*   **Precedence:** *1*
*   **Query name regex:** \*.\*\.in-addr\.arpa$ (every name that ends with *in-addr.arpa*)
*   **Query Type:** * (every query type)
*   **Remote IP:** 0.0.0.0/0 (every client IP address)
*   **Local IP:** 0.0.0.0/0 (every server IP address)

This section will be evalued as a first rule and will match against every reverse query (every names anding with in-addr.arpa) Ex.:

    dig @192.168.2.20 -x 192.168.3.10
    
    ;; QUESTION SECTION:
    ;20.2.168.192.in-addr.arpa.	IN	PTR
    
    ;; ANSWER SECTION:
    20.2.168.192.in-addr.arpa. 2400	IN	PTR	host.example.com.

Query resolution
================

**Query resolution:**</p> 

Inside sections you can define your resolution rules, you can define differents parameters depending on backend used.
In every sections you must define a **backend** paramether that define the backend. Unltil now only **ldap** and **static** backend are implemented

Static Backend
==============
Using the **static** backend you can statically configure answers for query matching the section. You can define 2 variables:

* **answer**: this is a three fields value separed by a semicolon (:), these fields must be: TYPE:TTL:VALUE, eg. A:300:192.168.10.100. you can define multiple response separing ansewrs with a comma (,). Eg.: 

This section define a rule matching every query for the name **host.example.com** with 2 A records with dittents TTL:

    [10:host\.example\.com:*:0.0.0.0/0:0.0.0.0/0]
    backend=static
    answer=A:300:192.168.10.20,A:900:192.168.10.22

Qwering the server:

    dig @172.16.18.5 host.example.com

    ;; QUESTION SECTION:
    ;host.example.com.		IN	A

    ;; ANSWER SECTION:
    host.example.com.	900	IN	A	192.168.10.22
    host.example.com.	300	IN	A	192.168.10.20

* **delay**: A delay in query response. This is a value in seconds, you can express also decimal values. You can use also random values using rand:MIN:MAX (eg. rand:3:10 will add arandom delay between 3 and 10 sec.)
Eg.:

Adding a 5 seconds delay for every query to host.example.com

    [10:host\.example\.com:*:0.0.0.0/0:0.0.0.0/0]
    delay=5
    backend=static
    answer=A:300:192.168.10.20,A:900:192.168.10.2

LDAP Backend
============
With this backend you can implement your DNS server using your own LDAP schema.
With LDAP Backend you can define for every query matching rule the ldap server, base, credentials, attributes and query filters using these parameters: 

**ldap_uri:** LDAP server URI Eg.: **ldap://localhost**

**base:** LDAP base used in query Eg.: **dc=example,dc=com**  
**bind_dn:** LDAP bind DN used during the query (if needed). Eg.: **cn=admin.example.com**
**bind_password:** LDAP bind password (if needed)  
**XXX_attribute:** Attribute mapping rule, Eg.: **A_attribute=aRecord** define a mapping between the *A* attribute and the ldap attribute **aRecord** thanks to this field you can use every LDAP schema for your DNS server.

**Examples:**
    MX_attribute = MxRecord  
    NS_attribute = nSRecord  
    TTL_attribute = dNSTTL  
    SOA_attribute = sOARecord  
    CNAME_attribute = cNAMERecord  
    TXT_attribute = tXTRecord  

**TTL_default:** default TTL (if **TTL_attribute** isn't defined) 
**query**: LDAP query filter. Here you can use some variable extracted from the query (see *Query filter formatting* ) Ex: **(&(objectClass=extensibleObject)(associatedDomain=%(qname)s))**  
**bind:** if False the ldap bind will be not executed. Valid values: **True** or **False**   

**Query filter formatting:**

the **query** parameter can contains some formatters that permit to dinamically compose the ldap query:

**%(qname)s** : expanded with **Query Name**  
**%(qtype)s** : expandend with **Query Type** 
**%(remote_ip)s** : expanded with **Remote IP**  
**%(local_ip)s** : expanded with **Local IP**
**%(rqname)s** : expanded with the IP address representing the *Query Name* if we are resolving a reverse query.

**Configuring PowerDNS**

You neet to add to PowerDNS configuration file (**/etc/powerdns/pdns.conf**) these paramethers:

   launch=pipe
   pipe-command=/etc/powerdns/fleppyb/fleppyb.py
   pipebackend-abi-version=2

* **launch=pipe** definisce di utilizzare un pipe backend  
* **pipe-command** definisce il path di fleppyb.py  
* **pipebackend-abi-version=2** definisce la versione del protocollo pipebackend da utilizzare 

Examples
========

View (split brain) implementation
---------------------------------

Query coming from the subnet 192.168.2.0/24 will be used the LDAP tree **dc=internal,dc=example,dc=com**.
All athers queries will be resolved searching in **dc=external,dc=example,dc=com** ldap tree.

Very important is the right use of rule precendence!!.

**fleppyb.ini**

    [DEFAULT]
    ldap_uri=ldap://localhost
    A_attribute=aRecord
    MX_attribute = MxRecord
    NS_attribute = nSRecord
    TTL_attribute = dNSTTL
    TTL_default = 2400
    SOA_attribute = sOARecord
    CNAME_attribute = cNAMERecord
    base = dc=example,dc=com
    query = (&(objectClass=domainRelatedObject)(associatedDomain=%(qname)s))
    
    [2:.*:*:0.0.0.0/0:0.0.0.0/0]
    TTL_attribute = drink
    base = dc=external,dc=example,dc=com
    query = (dc=%(qname)s)
    backend=ldap
    bind=False
    
    [1:.*:*:192.168.2.50:0.0.0.0/0]
    TTL_attribute = drink
    base = dc=internal,dc=example,dc=com
    query = (dc=%(qname)s)
    backend=ldap
    bind=False

**Database ldap (tree dc=external,dc=example,dc=com):**

    dn: dc=external,dc=example,dc=com
    dc: external
    objectClass: top
    objectClass: dcObject
    objectClass: organization
    o: 2v
    
    dn: dc=example.com,dc=external,dc=example,dc=com
    sOARecord: ns1.example.com. dns-admin.example.com. 1446303 7200 1800 1209600 300
    dc: example.com
    objectClass: dNSDomain
    objectClass: domain
    objectClass: top
    nSRecord: ns1.example.com.
    nSRecord: ns2.example.com.
    
    dn: dc=ns1.example.com,dc=external,dc=example,dc=com
    dc: ns1.example.com
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    aRecord: 83.102.11.30
    
    dn: dc=ns2.example.com,dc=external,dc=example,dc=com
    dc: ns2.example.com
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    aRecord: 83.102.11.31
    
    dn: dc=www.example.com,dc=external,dc=example,dc=com
    dc: www.example.com
    cNAMERecord: fast.example.com
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    
    dn: dc=fast.example.com,dc=external,dc=example,dc=com
    dc: fast.example.com
    drink: 60
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    aRecord: 83.102.11.32

**Dadabase ldap (tree dc=internal,dc=example,dc=com)**

    dn: dc=internal,dc=example,dc=com
    dc: internal
    objectClass: top
    objectClass: dcObject
    objectClass: organization
    o: 2v
    
    dn: dc=example.com,dc=internal,dc=example,dc=com
    sOARecord: ns1.example.com. dns-admin.example.com. 1446303 7200 1800 1209600 300
    dc: example.com
    objectClass: dNSDomain
    objectClass: domain
    objectClass: top
    nSRecord: ns1.example.com.
    nSRecord: ns2.example.com.
    
    dn: dc=ns1.example.com,dc=internal,dc=example,dc=com
    dc: ns1.example.com
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    aRecord: 192.168.101.2
    
    dn: dc=ns2.example.com,dc=internal,dc=example,dc=com
    dc: ns2.example.com
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    aRecord: 192.168.101.1
    
    dn: dc=www.example.com,dc=internal,dc=example,dc=com
    dc: www.example.com
    cNAMERecord: fast.example.com
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    
    dn: dc=fast.example.com,dc=internal,dc=example,dc=com
    dc: fast.example.com
    drink: 60
    objectClass: dcObject
    objectClass: top
    objectClass: organization
    objectClass: extensibleObject
    o: example.com
    aRecord: 192.168.101.3

**Query coming from an IP not in 192.168.2.0/24 subnet**

    dig @192.168.2.20 -b 192.168.3.20 www.example.com
    
    ;; QUESTION SECTION:
    ;www.example.com.               IN      A
    
    ;; ANSWER SECTION:
    www.example.com.        2400    IN      CNAME   fast.example.com.
    fast.example.com.       60      IN      A       83.102.11.32

**Query coming from the interna subnet (192.168.2.0/24)**

    dig @192.168.2.20 -b 192.168.2.50 www.example.com
    
    ;; QUESTION SECTION:
    ;www.example.com.               IN      A
    
;; ANSWER SECTION:
    www.example.com.        2400    IN      CNAME   fast.example.com.
    fast.example.com.       60      IN      A       192.168.101.3

Reverse zone
------------

If a query with type **ANY** or **PTR** asks for a name that ends with in-addr.arpa the formatter **%(rqname)** will be resolved with the IP address.

Ex:

**dig -t ANY -x 192.168.2.20**

this command executes a query with type ANY and name 20.2.168.192.in-addr.arpa 
**%(rqname)s will be resolved with 192.168.2.20**

Example:

**fleppyb.ini**

    [DEFAULT]
    ldap_uri=ldap://localhost
    ;bind_dn=cn=admin,dc=example,dc=com
    ;bind_password=changeme
    A_attribute = aRecord
    MX_attribute = MxRecord
    NS_attribute = nSRecord
    TTL_attribute = dNSTTL
    ; the default value for TTL
    TTL_default = 2400
    SOA_attribute = sOARecord
    CNAME_attribute = cNAMERecord
    base = dc=example,dc=com
    query = (&(objectClass=extensibleObject)(associatedDomain=%(qname)s))
    
    [2:.*:*:0.0.0.0/0:0.0.0.0/0]
    base = dc=netconf,dc=example,dc=com
    query = (&(objectclass=extensibleObject)(associatedDomain=%(qname)s))
    backend=ldap
    bind=False
    autogen_ptr=True
    
    [1:.*.in-addr\.arpa:*:0.0.0.0/0:0.0.0.0/0]
    base = dc=netconf,dc=example,dc=com
    query = (&(objectclass=extensibleObject)(arecord=%(rqname)s))
    backend=ldap
    bind=False
    PTR_attribute=associatedDomain

**Reverse query:**   

    dig -x 192.168.2.20)
    
   ;; QUESTION SECTION:
    ;20.2.168.192.in-addr.arpa.     IN      PTR
    
    ;; ANSWER SECTION:
    20.2.168.192.in-addr.arpa. 2400 IN      PTR     www.example.com.

PowerDNS and caching
====================

If you configure Fleppyb in order to provide differents answers for the same name (Ex. split brain configuration) you need to disable query caching in PowerDNS configuring query-cache-ttl e negquery-cache-ttl at 0.
You can leave enable the cache-ttl.

Testing

You can manually testing fleppy simulating the PIPE backend protocol:

    echo -ne "HELO\nQ\t20.2.168.192.in-addr.arpa\tIN\tANY\t-1\t192.168.3.13\t0.0.0.0" | /etc/powerdns/fleppyb/fleppyb.py


 [1]: http://doc.powerdns.com/backends-detail.html#pipebackend
 [2]: http://docs.python.org/2/library/configparser.html
 [3]: http://docs.python.org/2/library/re.html
