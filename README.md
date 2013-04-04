Fleppyb (Flexible PowerDNS Python Backend)
==========================================

Fleppyb è [un pipe backend per Powerdns][1]. In breve si occupa di ricevere la query dal processo master di PowerDNS e, in base a dei criteri di matching effettuati sui parametri della query, di risolvere la query. 
I parametri della query su cui viene effettuato il match sono quelli offerti dal protocollo di **PipeBackend** di PowebDNS e sono:

**Query Name:** il nome per qui si sta facendola query, ad esempio la query:

    dig @192.168.2.20 www.google.it

il Query Name sarà www.google.it

**Query Type:** il tipo di query effettuata, ovvero il record che si sta chiedendo, ad esempio:

    dig @192.168.2.20 -t MX google.it

il Query Type sarà **MX**

**Remote IP:** l'indirizzo ip del richiedente della query, ad esempio:

    dig -b 192.168.3.11 @192.168.2.20 -t MX google.it

il Remote IP sarà **192.168.3.11**

**Local IP:** l'indirizzo del server a cui viene richiesta la query, ad esempio:

    dig @192.168.2.20 www.google.it

il Local IP sarà **192.168.2.20**

Tramite il backend fleppyb è possibile implementare le viste su PowerDNS oppure qualsiasi altra configurazione nella risoluzione delle query

Configurazione
==============

Ci sono 2 tipi di configurazioni di effettuare:

1. Configurare Fleppyb
2. Configurare i Matching
3. Configurare PowerDNS

Configurare fleppyb.py
----------------------

Per la prima occorre editare il file fleppyb.py e modificare le seguenti variabili:

* **DEBUG:** valori possibili: True o False
* **CONFIG_FILE:** il path al file di configurazione in cui vengono specificati i criteri di match
* **PARSE_CONFIG_ONCE:** rileggi il file sopra ad ogni query (valori possibili: True o False) oppure solo allo start. L'impostazione a True potrebbe influire sulle performance, se a False ad ogni modifica del file di match occorre riavviare PowerDNS.
* **LOGFILE:** il file di log di fleppy: NB: powerdns deve poter scrivere questo file

Configurare i query matching
----------------------------

Il file di match (definito dalla variabile `CONFIG_FILE`) ha la sintassi di un file .ini: definisce delle sezioni e per ogni sezione vengono definite delle variabili.

E' possibile definire una sezione particolare con nome **DEFAULT** in cui vengono definite delle variabili che verranno ereditate dalle altre sezioni. Se una sezione definisce una variabile questa sovrascrive la variabile nella sezione **DEFAULT**.
Il nome delle sezioni è particolarmente importante: tramite il nome della sezione viene definito il criterio di match della query.
La sezione **DEFAULT** non paretecipa all'algoritmo di matching.

Il nome della sezione deve rispettare la seguente sintassi:

**[PREC;QUERY_NAME_REGEX:QUERY_TYPE:REMOTE_IP:LOCAL_IP]**

in cui:

* **PREC:** rappresenta la precedenza della sezione: la prima sezione che matcha con i parametri della query sarà utilizzata per risolvere la query, deve essere un numero intero, piu' è alto il numero e piu' è bassa la priorità
* **QUERY_NAME_REGEX:** è una regular expression che viene controntata con il parametro Query Name della query, la sintassi è la sintassi delle regex
* **QUERY_TYPE:** questo valore viene confrontato con il tipo di query, puo' assumere valori A, TXT, MX, ecc.. oppure * per essere sempre verificata
* **REMOTE_IP:** questo valore viene confrontato con l'ip del richiedente, puo' essere un indirizzo ip oppure una subnet CIDR (es: 192.168.2.0/24)
* **LOCAL_IP:** viene confrontato con l'ip del server a cui viene fatta la richiesta, il formato è come il valore precedente

Esempi
-----

**[1:.*.in-addr\.arpa$:*:0.0.0.0/0:0.0.0.0/0]** Una sezione con questo nome ha:

*   **Precedenza:** *1*
*   **Query name regex:** *.*\.in-addr\.arpa$* (qualsiasi nome che finisce con *in-addr.arpa*)
*   **Query Type:** *** (qualsiasi tipo di query)
*   **Remote IP:** 0.0.0.0/0 (qualsiasi ip client)
*   **Local IP:** 0.0.0.0/0 (qualsiasi ip server)

Questa sezione verrà valutata per prima e verrà utilizzata per la risoluzione per tutte le query inverse (tutti nomi che finiscono per in-addr.arpa) es:

    dig @192.168.2.20 -x 192.168.3.10
    
    ;; QUESTION SECTION:
    ;20.2.168.192.in-addr.arpa.	IN	PTR
    
    ;; ANSWER SECTION:
    20.2.168.192.in-addr.arpa. 2400	IN	PTR	host.example.com.

Query resolution
================

**Query resolution:**</p> 
All'interno della sezioni vanno definiti parametri per effettuare la risoluzione, i parametri possono essere:

**backend:** attualmente puo' essere solo **ldap** o **delay**, specifica quale backend fleppyb deve utilizzare per la risoluzione della query

DELAY Backend
=============
Usando il backend **delay** il backend aspetterà un delay impostato e poi risponderà con un not found.
**delay:** delay espresso in secondi prima di rispondere, si possono usare dei decimali, es.: 0.673 oppure generare un delay random: random:0.5:30 

LDAP Backend
============
**ldap_uri:** URI del server ldap es: **ldap://localhost**

**base:** ldap base per la query es: **dc=example,dc=com**  
**bind_dn: **dn con cui effettuare il bind (se necessario) es: **cn=admin,dc=example,dc=com**  
**bind_password:** password per il bind (se necessario)  
***_attribute:** mapping degli attributi ldap, es **A_attribute=aRecord** definisce un mapping tra la entry dns **A** e l'attributo ldap **aRecord**. 

**Esempi:**
    MX_attribute = MxRecord  
    NS_attribute = nSRecord  
    TTL_attribute = dNSTTL  
    SOA_attribute = sOARecord  
    CNAME_attribute = cNAMERecord  
    TXT_attribute = tXTRecord  

**TTL_default:** il TTL da restiture di default (se non è specificato nell'attributo definito da TTL_attribute)  
**query**:  il query filter con cui effettuare la query ldap es: **(&(objectClass=extensibleObject)(associatedDomain=%(qname)s))**  
**bind:** definisce se effettuare il bind oppure fare una query ldap anonima, puo' essere **False** o **True**  
 

**Query filter formatting:**

il query filter puo' contenere dei formatter specifier per comporre dinamicamente la query ldap:

**%(qname)s** : viene espanso con il campo Query Name  
**%(qtype)s** : viene espanso con il campo Query Type della query  
**%(remote_ip)s** : viene espanso con il campo Remote IP della query  
**%(local_ip)s** : viene espanso con il campo Local IP della query

**%(rqname)s** : viene espanso con l'indirizzo IP rappresentante il qname se si tratta di una query inversa

**Configurare PowerDNS**

Ocorre aggiungere al file di configurazione di PowerDNS (**/etc/powerdns/pdns.conf**) queste direttive:

   pipe-command=/etc/powerdns/fleppyb/fleppyb.py
   pipebackend-abi-version=2

**launch=pipe** definisce di utilizzare un pipe backend  
**pipe-command** definisce il path di fleppyb.py  
**pipebackend-abi-version=2** definisce la versione del protocollo pipebackend da utilizzare 

Esempi
======

Implementazione delle viste
---------------------------

Per le query proveninenti dalla subnet 192.168.2.0/24 viene effettuato il lookup ldap nel base **dc=internal,dc=example,dc=com**
il resto delle query vengono risolte cercando nel base **dc=external,dc=example,dc=com**
Di fondamentale importanza le priorità nel file di configurazione.

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

**Database ldap (base dc=external,dc=example,dc=com):**

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

**Dadabase ldap (base dc=internal,dc=example,dc=com)**

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

**Query da ip non nella subnet 192.168.2.0/24**

    dig @192.168.2.20 -b 192.168.3.20 www.example.com
    
    ;; QUESTION SECTION:
    ;www.example.com.               IN      A
    
    ;; ANSWER SECTION:
    www.example.com.        2400    IN      CNAME   fast.example.com.
    fast.example.com.       60      IN      A       83.102.11.32

**Query da ip nella subnet 192.168.2.0/24**

    dig @192.168.2.20 -b 192.168.2.50 www.example.com
    
    ;; QUESTION SECTION:
    ;www.example.com.               IN      A
    
    ;; ANSWER SECTION:
    www.example.com.        2400    IN      CNAME   fast.example.com.
    fast.example.com.       60      IN      A       192.168.101.3

Reverse zone
------------

se la query è di tipo ANY o PTR e richiede un nome che finisce in in-addr.arpa
il formatter specifier %(rqname)s viene risolto con l'ip dell'oggetto richiesto.

Es:

**dig -t ANY -x 192.168.2.20**

effettua una query di tipo ANY per il nome 20.2.168.192.in-addr.arpa 
**%(rqname)s viene risolto con 192.168.2.20**

Se la query non è di tipo ANY o PTR o la richiesta non è per un *in-addr.arpa rqname è vuoto

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

PowerDNS e caching
==================

Se si configura fleppyb in modo da rispondere in modi diversi per lo stesso query name (ad. es. configurazione con le viste) occorre disabilitare la cache delle query in PowerDNS configurando i parametri query-cache-ttl e negquery-cache-ttl a 0.
Il parametro cache-ttl puo' essere configurato a piacere: esegue lo store nella cache secondo tutti iparametri della query

Testing

è possibile testare a mano fleppy.py dandogli in pasto le query a mano:

    echo -ne "HELO\nQ\t20.2.168.192.in-addr.arpa\tIN\tANY\t-1\t192.168.3.13\t0.0.0.0" | /etc/powerdns/fleppyb/fleppyb.py


 [1]: http://doc.powerdns.com/backends-detail.html#pipebackend
