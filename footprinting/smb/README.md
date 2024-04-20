The purpose of this section is to explore the default configuration of a Linux Samba server and to test some of the settings that can be set on the different shares.

## Default configuration
The `samba` package comes with a default configuration that can be used directly. There is a slightly modified version (without the comments) in the folder `default_configuration`. There are two differences with the default `smb.conf` file:
- the setting `log file` is set to `/var/log/samba/log.txt` in order to have always the same log file independently of the name of the client machine
- the setting `log level` is set to `1` to get some log in the file mentioned above

In the `Dockerfile`, two Unix users will be created (`mary` and `john`), and one of them will have a Samba account (`mary`).

To start the server, run the following command:
```
docker build -t smb_default_configuration $(pwd) && docker run -v $(pwd)/smb.conf:/etc/samba/smb.conf smb_default_configuration
```

`nmap` can be used to check the server is running:
```
$nmap -p139,445 -sC -sV 172.17.0.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-04-15 19:21 CEST
Nmap scan report for 172.17.0.2
Host is up (0.00013s latency).

PORT    STATE SERVICE     VERSION
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2

Host script results:
| smb2-time: 
|   date: 2024-04-15T17:21:52
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.49 seconds
```

`smbclient` can be used to list the browsable drives:
```
$smbclient -N -L //172.17.0.2

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
	nobody          Disk      Home Directories
SMB1 disabled -- no workgroup available
```

The user `mary` can access to its home directory:
```
$smbclient -U mary //172.17.0.2/mary
Password for [WORKGROUP\mary]: smbpass
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Apr 15 19:08:26 2024
  ..                                  D        0  Mon Apr 15 19:08:26 2024
  .bash_logout                        H      220  Sun Apr 23 23:23:06 2023
  .bashrc                             H     3526  Sun Apr 23 23:23:06 2023
  .profile                            H      807  Sun Apr 23 23:23:06 2023

		124986368 blocks of size 1024. 99741940 blocks available
```

The home directories are only available to their owner:
```
$smbclient -U mary%smbpass //172.17.0.2/john
tree connect failed: NT_STATUS_ACCESS_DENIED
```

However, it is not possible to log with the user `john`:
```
$smbclient -U john%smbpass //172.17.0.2/john
tree connect failed: NT_STATUS_ACCESS_DENIED
```

In the log, there is a mention that `john` is considered as a guest:
```
[2024/04/15 17:35:30.130463,  1] ../../source3/smbd/smb2_service.c:355(create_connection_session_info)
  create_connection_session_info: guest user (from session setup) not permitted to access this share (mary)
[2024/04/15 17:35:30.130509,  1] ../../source3/smbd/smb2_service.c:545(make_connection_snum)
  create_connection_session_info failed: NT_STATUS_ACCESS_DENIED
```
Indeed, even if `john` is a Unix user, it is not a Samba user. Due to the option `map to guest = bad user`, when a username does not exist, it is considered as a guest.

`smbclient` returns a different error message when the share does not exist and when the user does not have access:
```
$smbclient -U mary%smbpass //172.17.0.2/john
tree connect failed: NT_STATUS_ACCESS_DENIED

$smbclient -U mary%smbpass //172.17.0.2/johny
tree connect failed: NT_STATUS_BAD_NETWORK_NAME
```
This can eventually be used to enumerate the non-browsable shares.

The share `printers` is not browsable but can be accessed by non-guest users:
```
$smbclient -U mary%smbpass //172.17.0.2/printers
Try "help" to get a list of possible commands.
smb: \> pwd
Current directory is \\172.17.0.2\printers\
```
### RPC
A Samba server exposes a lot of different procedures that can be call using RPC, even with no authentication. This can be a powerful way to get knowledge about a target:
```
$rpcclient -U "" -N 172.17.0.2
rpcclient $>
```
Several commands can be used to get information about the server:
```
rpcclient $> srvinfo
	1CCBAFF2EEC3   Wk Sv PrQ Unx NT SNT Samba 4.17.12-Debian
	platform_id     :	500
	os version      :	6.1
	server type     :	0x809a03

rpcclient $> enumdomains
name:[1CCBAFF2EEC3] idx:[0x0]
name:[Builtin] idx:[0x1]

rpcclient $> querydominfo 1CCBAFF2EEC3
Minimum password length:			5
Password uniqueness (remember x passwords):	0
password_properties: 0x00000000
password expire in:				4294942441 days, 4294967293 hours, 4294967282 minutes, 4294967288 seconds
Min password age (allow changing in x days):	Now

rpcclient $> querydominfo Builtin
Domain:		WORKGROUP
Server:		1CCBAFF2EEC3
Comment:	Samba 4.17.12-Debian
Total Users:	1
Total Groups:	0
Total Aliases:	0
Sequence No:	1713543886
Force Logoff:	-1
Domain Server State:	0x1
Server Role:	ROLE_DOMAIN_PDC
Unknown 3:	0x1
```
We can see that there is one user in the domain `WORKGROUP`. We can get more information about this user:
```
rpcclient $> enumdomusers
user:[mary] rid:[0x3e8]

rpcclient $> queryuser 0x3e8
	User Name   :	mary
	Full Name   :	
	Home Drive  :	\\1CCBAFF2EEC3\mary
	Dir Drive   :	
	Profile Path:	\\1CCBAFF2EEC3\mary\profile
	Logon Script:	
	Description :	
	Workstations:	
	Comment     :	
	Remote Dial :
	Logon Time               :	jeu., 01 janv. 1970 01:00:00 CET
	Logoff Time              :	mer., 06 févr. 2036 16:06:39 CET
	Kickoff Time             :	mer., 06 févr. 2036 16:06:39 CET
	Password last set Time   :	lun., 15 avril 2024 19:18:50 CEST
	Password can change Time :	lun., 15 avril 2024 19:18:50 CEST
	Password must change Time:	jeu., 14 sept. 30828 04:48:05 CEST
	unknown_2[0..31]...
	user_rid :	0x3e8
	group_rid:	0x201
	acb_info :	0x00000010
	fields_present:	0x00ffffff
	logon_divs:	168
	bad_password_count:	0x00000000
	logon_count:	0x00000000
	padding1[0..7]...
	logon_hrs[0..21]...

rpcclient $> querygroup 0x201
	Group Name:	None
	Description:	Ordinary Users
	Group Attribute:7
	Num Members:0
```
We can also get information about the shares available:
```
rpcclient $> netshareenumall
netname: print$
	remark:	Printer Drivers
	path:	C:\var\lib\samba\printers
	password:	
netname: IPC$
	remark:	IPC Service (Samba 4.17.12-Debian)
	path:	C:\tmp
	password:	
netname: nobody
	remark:	Home Directories
	path:	C:\nonexistent
	password:	
rpcclient $> netsharegetinfo print$
netname: print$
	remark:	Printer Drivers
	path:	C:\var\lib\samba\printers
	password:	
	type:	0x0
	perms:	0
	max_uses:	-1
	num_uses:	1
rpcclient $> netsharegetinfo IPC$
netname: IPC$
	remark:	IPC Service (Samba 4.17.12-Debian)
	path:	C:\tmp
	password:	
	type:	0x80000003
	perms:	0
	max_uses:	-1
	num_uses:	1
rpcclient $> netsharegetinfo nobody
netname: nobody
	remark:	Home Directories
	path:	C:\nonexistent
	password:	
	type:	0x0
	perms:	0
	max_uses:	-1
	num_uses:	1
```
We can even query hidden shares:
```
rpcclient $> netsharegetinfo mary
netname: mary
	remark:	Home Directories
	path:	C:\home\mary
	password:	
	type:	0x0
	perms:	0
	max_uses:	-1
	num_uses:	1
```
These queries can be automated using several tools.
#### `impacket-samrdump`
The purpose of this script is to dump the list of the users of the target system:
```
$impacket-samrdump 172.17.0.2
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Retrieving endpoint list from 172.17.0.2
Found domain(s):
 . 1CCBAFF2EEC3
 . Builtin
[*] Looking up users in domain 1CCBAFF2EEC3
Found user: mary, uid = 1000
mary (1000)/FullName: 
mary (1000)/UserComment: 
mary (1000)/PrimaryGroupId: 513
mary (1000)/BadPasswordCount: 0
mary (1000)/LogonCount: 0
mary (1000)/PasswordLastSet: 2024-04-15 19:18:50
mary (1000)/PasswordDoesNotExpire: False
mary (1000)/AccountIsDisabled: False
mary (1000)/ScriptPath: 
[*] Received one entry.
```
#### `SMBMap`
This tool can be used to enumerate the shares of a target:
```
$smbmap -u mary -p smbpass -H 172.17.0.2
[+] IP: 172.17.0.2:445	Name: 172.17.0.2                                        
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	READ ONLY	Printer Drivers
	IPC$                                              	NO ACCESS	IPC Service (Samba 4.17.12-Debian)
	mary                                              	READ ONLY	Home Directories
```
This tool has also other features to search or download files on the server.
#### `CrackMapExec`
This project is not maintained anymore, but we can still get it from the [GitHub project](https://github.com/byt3bl33d3r/CrackMapExec). It can be build using `docker` and run that way:
```
$docker run crackmapexec --help
```
With no options, it gives some simple information about the server:
```
$docker run crackmapexec smb 172.17.0.2
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Windows 6.1 Build 0 (name:1CCBAFF2EEC3) (domain:1CCBAFF2EEC3) (signing:False) (SMBv1:False)
```
It can be used to enumerate shares:
```
$docker run crackmapexec smb 172.17.0.2 -u '' -p '' --shares
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Windows 6.1 Build 0 (name:1CCBAFF2EEC3) (domain:1CCBAFF2EEC3) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    1CCBAFF2EEC3     [+] 1CCBAFF2EEC3\: 
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Enumerated shares
SMB         172.17.0.2      445    1CCBAFF2EEC3     Share           Permissions     Remark
SMB         172.17.0.2      445    1CCBAFF2EEC3     -----           -----------     ------
SMB         172.17.0.2      445    1CCBAFF2EEC3     print$                          Printer Drivers
SMB         172.17.0.2      445    1CCBAFF2EEC3     IPC$                            IPC Service (Samba 4.17.12-Debian)
SMB         172.17.0.2      445    1CCBAFF2EEC3     nobody                          Home Directories
```
It can also be used to enumerate users or disks, for instance:
```
$docker run crackmapexec smb 172.17.0.2 --disks
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Windows 6.1 Build 0 (name:1CCBAFF2EEC3) (domain:1CCBAFF2EEC3) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Enumerated disks
SMB         172.17.0.2      445    1CCBAFF2EEC3     C:

$docker run crackmapexec smb 172.17.0.2 --users
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Windows 6.1 Build 0 (name:1CCBAFF2EEC3) (domain:1CCBAFF2EEC3) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    1CCBAFF2EEC3     [*] Trying to dump local users with SAMRPC protocol
SMB         172.17.0.2      445    1CCBAFF2EEC3     [+] Enumerated domain user(s)
SMB         172.17.0.2      445    1CCBAFF2EEC3     1CCBAFF2EEC3\mary
```
#### enum4linux-ng
Finally, this tool is able to perform all the enumerations in one:
```
$./enum4linux-ng.py 172.17.0.2
ENUM4LINUX - next generation (v1.3.3)

 ==========================
|    Target Information    |
 ==========================
[*] Target ........... 172.17.0.2
[*] Username ......... ''
[*] Random Username .. 'ojdgmglf'
[*] Password ......... ''
[*] Timeout .......... 5 second(s)

[...]

 =======================================
|    RPC Session Check on 172.17.0.2    |
 =======================================
[*] Check for null session
[+] Server allows session using username '', password ''
[*] Check for random user
[+] Server allows session using username 'ojdgmglf', password ''
[H] Rerunning enumeration with user 'ojdgmglf' might give more results


[...]

 ===================================
|    Users via RPC on 172.17.0.2    |
 ===================================
[*] Enumerating users via 'querydispinfo'
[+] Found 1 user(s) via 'querydispinfo'
[*] Enumerating users via 'enumdomusers'
[+] Found 1 user(s) via 'enumdomusers'
[+] After merging user results we have 1 user(s) total:
'1000':
  username: mary
  name: ''
  acb: '0x00000010'
  description: ''

[...]

 ====================================
|    Shares via RPC on 172.17.0.2    |
 ====================================
[*] Enumerating shares
[+] Found 3 share(s):
IPC$:
  comment: IPC Service (Samba 4.17.12-Debian)
  type: IPC
nobody:
  comment: Home Directories
  type: Disk
print$:
  comment: Printer Drivers
  type: Disk
[*] Testing share IPC$
[+] Mapping: OK, Listing: NOT SUPPORTED
[*] Testing share nobody
[+] Mapping: DENIED, Listing: N/A
[*] Testing share print$
[+] Mapping: DENIED, Listing: N/A

[...]
```
## Modified configuration
In the modified configuration, the following share has been added:
```
[smbshare]
   comment = Share available to anyone
   path = /smbshare
   browseable = yes
   read only = no
   guest ok = yes
   create mask = 0777
   directory mask = 0777
   force user = root
```
With this configuration, any user and guest should be able to read and write files and directories in the share.

To run the server with this configuration:
```
docker build -t smb_default_configuration $(pwd) && docker run -v $(pwd)/modified.conf:/etc/samba/smb.conf smb_default_configuration
```

The share has the READ, WRITE permissions even for a guest user:
```
$smbmap -H 172.17.0.2
[+] IP: 172.17.0.2:445	Name: 172.17.0.2                                        
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	smbshare                                          	READ, WRITE	Share available to anyone
	IPC$                                              	NO ACCESS	IPC Service (Samba 4.17.12-Debian)
	nobody                                            	NO ACCESS	Home Directories
```
Note that the folder, on the server, should have the permissions of the `force user` which is `root` in our case.

Now, any user can create, modify and delete any files they want in the share:
```
$smbclient -U '' -N //172.17.0.2/smbshare
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sat Apr 20 18:55:01 2024
  ..                                  D        0  Sat Apr 20 18:54:56 2024

		124986368 blocks of size 1024. 98451208 blocks available
smb: \> put smb.conf
putting file smb.conf as \smb.conf (230,7 kb/s) (average 230,7 kb/s)
smb: \> mkdir test
smb: \> dir
  .                                   D        0  Sat Apr 20 18:57:08 2024
  ..                                  D        0  Sat Apr 20 18:54:56 2024
  smb.conf                            A      945  Sat Apr 20 18:57:03 2024
  test                                D        0  Sat Apr 20 18:57:08 2024

		124986368 blocks of size 1024. 98451232 blocks available
```
## References
- [smb.conf man page](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)