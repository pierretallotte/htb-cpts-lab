FROM debian

RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y snmpd

RUN net-snmp-create-v3-user -A password authPrivUser

EXPOSE 161:161/udp

CMD ["/usr/sbin/snmpd", "-f"]
