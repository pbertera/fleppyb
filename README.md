Fleppyb (Flexible PowerDNS Python Backend)
==========================================

Fleppyb è [un pipe backend per Powerdns][1]. In breve si occupa di ricevere la query dal processo master di PowerDNS e, in base a dei criteri di matching effettuati sui parametri della query, di risolvere la query.</p> 
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

**backend:** attualmente puo' essere solo **ldap**, specifica quale backend fleppyb deve utilizzare per la risoluzione della query

**ldap_uri:** URI del server ldap es: **ldap://localhost**

**base:** ldap base per la query es: **dc=example,dc=com**  
**bind_dn: **dn con cui effettuare il bind (se necessario) es: **cn=admin,dc=example,dc=com**  
**bind_password:** password per il bind (se necessario)  
***_attribute:** mapping degli attributi ldap, es **A_attribute= aRecord** definisce un mapping tra la entry dns **A** e l'attributo ldap **aRecord**. Esempi:  
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



 [1]: http://doc.powerdns.com/backends-detail.html#pipebackend
