In this section, we will set up an SNMP agent, enabling different versions of the protocol, and looking at the impact of some settings.

## Default configuration

The installation of `snmpd` package comes from a default configuration file, which is almost the same as the file `snmpd.conf` in this folder. The only difference is that the line `agentaddress  127.0.0.1,[::1]` has been commented out in order to make the agent available from the outside. Also, in the given Dockerfile, it is necessary to manually create a user to be able to use the version 3 of the protocol. Otherwise, no user will be able to make any query, even so a user is specified in the `rouser` line of the configuration file.

To run the agent, we will run the following command:
```
docker build -t snmp_default_configuration $(pwd) && docker run -v $(pwd)/snmpd.conf:/etc/snmp/snmpd.conf snmp_default_configuration
```

The agent is now running. We can check that using `nmap`:
```
$ sudo nmap -p161 -sU -sC -sV 192.168.0.1
Starting Nmap 7.93 ( https://nmap.org ) at 2024-04-10 16:24 UTC
Nmap scan report for 192.168.0.1
Host is up (0.00011s latency).

PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
| snmp-info:
|   enterprise: net-snmp
|   engineIDFormat: unknown
|   engineIDData: ebe96c24f0bc166600000000
|   snmpEngineBoots: 1
|_  snmpEngineTime: 1m18s
| snmp-sysdescr: Linux 492e4f48219b 5.10.0-28-amd64 #1 SMP Debian 5.10.209-2 (2024-01-31) x86_64
|_  System uptime: 1m17.89s (7789 timeticks)
MAC Address: 02:42:C0:A8:00:01 (Unknown)
Service Info: Host: 492e4f48219b

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.77 seconds
```

We can see that two versions of SNMP are running (v1 and v3). This is because the default configuration file contains `rocommunity` lines which are dedicated for v1 and v2c, and a line `rouser` dedicated for v3.

### SNMPv1

In the configuration file, one view is defined:
```
view   systemonly  included   .1.3.6.1.2.1.1
view   systemonly  included   .1.3.6.1.2.1.25.1
```
In the view `systemonly` are included the nodes `system` and `hrSystem`.
This view has been made accessible in read-only to the `public` community:
```
rocommunity  public default -V systemonly
```

A query can be done using `snmpwalk` to get the information in these nodes:
```
$ snmpwalk -v1 -c public 192.168.0.1
Created directory: /var/lib/snmp/cert_indexes
iso.3.6.1.2.1.1.1.0 = STRING: "Linux 492e4f48219b 5.10.0-28-amd64 #1 SMP Debian 5.10.209-2 (2024-01-31) x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (39815) 0:06:38.15
iso.3.6.1.2.1.1.4.0 = STRING: "Me <me@example.org>"
iso.3.6.1.2.1.1.5.0 = STRING: "492e4f48219b"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (12304542) 1 day, 10:10:45.42
iso.3.6.1.2.1.25.1.2.0 = Hex-STRING: 07 E8 04 0A 10 1D 32 00 2B 00 00 
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/vmlinuz-5.10.0-28-amd64 root=/dev/mapper/gnb--ptallott--l.vg.00-root ro quiet"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 0
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 1
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
End of MIB
```

To bruteforce the community string, it is possible to use `onesixtyone`:
```
$onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt 172.17.0.2
Scanning 1 hosts, 120 communities
172.17.0.2 [public] Linux a218134b9c0c 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64
172.17.0.2 [public] Linux a218134b9c0c 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64
```
The `public` community string has been found.

### SNMPv3

SNMPv3 is more secure than SNMPv1 because it adds authentication and encrypted communication. In the default configuration file, the user `authPrivUser` can read the view `systemonly` if the communication is authenticated and encrypted:
```
rouser authPrivUser authpriv -V systemonly
```

We can make a query using `snmpwalk`:
```
$ snmpwalk -v3 -u authPrivUser -A password -X password -a MD5 -l authpriv 192.168.0.1
iso.3.6.1.2.1.1.1.0 = STRING: "Linux 492e4f48219b 5.10.0-28-amd64 #1 SMP Debian 5.10.209-2 (2024-01-31) x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (88381) 0:14:43.81
iso.3.6.1.2.1.1.4.0 = STRING: "Me <me@example.org>"
iso.3.6.1.2.1.1.5.0 = STRING: "492e4f48219b"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (12353108) 1 day, 10:18:51.08
iso.3.6.1.2.1.25.1.2.0 = Hex-STRING: 07 E8 04 0A 10 25 37 00 2B 00 00
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/vmlinuz-5.10.0-28-amd64 root=/dev/mapper/gnb--ptallott--l.vg.00-root ro quiet"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 0
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 1
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
iso.3.6.1.2.1.25.1.7.0 = No more variables left in this MIB View (It is past the end of the MIB tree)
```

It is interesting to note that `snmpwalk` does not return the same error when access is denied and when a user does not exist:
```
$ snmpwalk -v3 -u authPrivUser -l noauth 192.168.0.1
Error in packet.
Reason: authorizationError (access denied to that object)

$ snmpwalk -v3 -u unknownUser -l noauth 192.168.0.1
snmpwalk: Unknown user name
```

This means that we could be able to enumerate the users of the agent by looking at the error returned by `snmpwalk`. For instance, this command could give us the list of the users:
```
for user in `cat userlist` ; do snmpwalk -v3 -u $user -l noauth 192.168.0.1 2>&1 | grep -v "Unknown user name" > /dev/null && echo $user ; done
```
## Read-write settings
In `modified_snmpd.conf`, several lines have been modified:
```
view   location  included   .1.3.6.1.2.1.1.6 
view   contact   included   .1.3.6.1.2.1.1.4
view   services  included   .1.3.6.1.2.1.1.7
view   hrSystem  included   .1.3.6.1.2.1.25.1
```
Four views have been created. These views will be accessible only by some users or community strings:
```
rocommunity public    default -V location
rwcommunity private   default -V hrSystem
```
It will be possible to modify `hrSystem` but not the location properties using SNMPv1. Moreover, the `hrSystem` will use the `private` community string.
```
rouser authPrivUser noauth   -V contact
rouser authPrivUser auth     -V location
rwuser authPrivUser priv     -V hrSystem
```
Using SNMPv3, the user `authPrivUser` will be able to access to all the views, but the security levels will be different. Moreover, the user will be able to modify the values of `hrSystem`.
### SNMPv1
Again, we can use `onesixtyone` to bruteforce the community strings:
```
$onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt 172.17.0.2 
Scanning 1 hosts, 120 communities
172.17.0.2 [public] Host responded with error NO SUCH NAME
172.17.0.2 [private] Host responded with error NO SUCH NAME
172.17.0.2 [private] Host responded with error NO SUCH NAME
172.17.0.2 [public] Host responded with error NO SUCH NAME
```
I do not understand the given error, but we can see that `public` and `private` community strings have been found. We can use them to get the corresponding values:
```
$snmpwalk -v1 -c public 172.17.0.2
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
End of MIB

$snmpwalk -v1 -c private 172.17.0.2
iso.3.6.1.2.1.1.1.0 = STRING: "Linux ed0a72ae2aa8 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (1284) 0:00:12.84
iso.3.6.1.2.1.1.4.0 = STRING: "Me <me@example.org>"
iso.3.6.1.2.1.1.5.0 = STRING: "ed0a72ae2aa8"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (1) 0:00:00.01
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (0) 0:00:00.00
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (1) 0:00:00.01
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (1) 0:00:00.01
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (1) 0:00:00.01
End of MIB
```

To be able to modify a value, we need to have the MIB installed on the system:
```
sudo apt install snmp-mibs-downloader
```

Then, `snmpset` is used to update a value if the user has the permission to do it:
```
$snmpset -v1 -c private -m ALL 172.17.0.2 iso.3.6.1.2.1.1.5.0 s "Another string"
SNMPv2-MIB::sysName.0 = STRING: Another string
```
We can check this works as expected using `snmpget`:
```
$snmpget -v1 -c private 172.17.0.2 iso.3.6.1.2.1.1.5.0 
iso.3.6.1.2.1.1.5.0 = STRING: "Another string"
```
### SNMPv3
To access to the data, we can use the different security levels:
```
$snmpwalk -v3 -u authPrivUser  -l noauth 172.17.0.2 
iso.3.6.1.2.1.1.4.0 = STRING: "Me <me@example.org>"
iso.3.6.1.2.1.1.4.0 = No more variables left in this MIB View (It is past the end of the MIB tree)

$snmpwalk -v3 -u authPrivUser -A password -l auth 172.17.0.2 
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.6.0 = No more variables left in this MIB View (It is past the end of the MIB tree)

$snmpwalk -v3 -u authPrivUser -A password -X password -a MD5 -l priv 172.17.0.2 
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (855832) 2:22:38.32
iso.3.6.1.2.1.25.1.2.0 = Hex-STRING: 07 E8 04 0C 0A 2C 25 00 2B 00 00 
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/@/boot/vmlinuz-6.5.0-13parrot1-amd64 root=UUID=8aa862fe-5469-4da5-8079-e9f0732fade0 ro rootflags=subvol=@ quiet spla"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 0
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 1
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
iso.3.6.1.2.1.25.1.7.0 = No more variables left in this MIB View (It is past the end of the MIB tree)
```

`snmpset` can be used the same way to modify some data:
```
$snmpset -v3 -u authPrivUser -A password -X password -a MD5 -l priv 172.17.0.2 iso.3.6.1.2.1.25.1.7.0 i 2
Error in packet.
Reason: notWritable (That object does not support modification)
Failed object: iso.3.6.1.2.1.25.1.7.0
```
However, some data cannot be modified even for a `rwuser`.
## References
- [snmpd.conf man page](http://www.net-snmp.org/docs/man/snmpd.conf.html)