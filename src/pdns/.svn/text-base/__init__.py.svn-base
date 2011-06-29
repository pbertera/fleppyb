#!/usr/bin/env python

import logging
import sys
import traceback

logger = logging.getLogger('pdns')

class DNSQuery(object):
    def __init__(self, qname, qclass, qtype, id, remote_ip, local_ip=None):
        self.qname = qname
        #rqname ip address, only for reverse query
        if qtype in ['PTR', 'ANY'] and qname.endswith('.in-addr.arpa'):
            ptr = qname.split('.')[:-2][::-1]
            self.rqname = '.'.join(''.join(ptr[x:x+1]) for x in xrange(0, len(ptr), 1))
        else:
            self.rqname = ''
        self.qclass = qclass
        self.qtype = qtype
        self.id = id
        self.remote_ip = remote_ip
        self.local_ip = local_ip

    def __str__(self):
        return "Q\t%s\t%s\t%s\t%s\t%s\t%s" % (self.qname, self.qclass, self.qtype, self.id, self.remote_ip, self.local_ip or '')

class DNSAnswer(object):
    def __init__(self, qname, qclass, qtype, ttl, id, content):
        self.qname = qname
        self.qclass = qclass
        self.qtype = qtype
        self.ttl = ttl
        self.id = id
        self.content = content

    def __str__(self):
        return "DATA\t%s\t%s\t%s\t%d\t%s\t%s" % (self.qname, self.qclass, self.qtype, self.ttl, self.id, self.content)

class PowerDNSBackend(object):
    def __init__(self, backend, logger=logger):
        self.backend = backend
        self.logger = logger

    def run(self):
        greeted = False
        while True:
            line = self.read()
            if line is None:
                break

            if not greeted:
                if not line.startswith('HELO'):
                    self.logger.error("Didn't receive HELO as expected.")
                    self.write('FAIL')
                    self.read()
                    break
                self.write('OK')
                greeted = True
            else:
                query = line.split('\t')
                _type = query[0]
                response = None
                try:
                    if _type == "PING":
                        self.logger.info("PING")
                    elif _type == 'Q':
                        self.logger.debug("DNS query")
                        q = DNSQuery(*query[1:])
                        self.logger.debug(str(q))
                        response = self.backend.query(q)
                    elif _type == 'AXFR':
                        self.logger.debug("AXFR query")
                        response = self.backend.axfr(*query) or []
                    if response:
                        self.write("\n".join(str(x) for x in response))
                    self.write("END")
                except Exception, exc:
                    self.logger.error("Failed query: %r\n%s" % (query, traceback.format_exc()))
                    self.write("FAIL")

    def read(self):
        line = sys.stdin.readline()
        if not line:
            self.logger.debug("Lost connection")
            return None
        line = line.strip()
        self.logger.debug("RECV: %s" % line)
        return line

    def write(self, line):
        self.logger.debug("SEND: %s" % line)
        sys.stdout.write(line + "\n")
        sys.stdout.flush()

