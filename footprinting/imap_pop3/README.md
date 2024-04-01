In this section, we will set up a [dovecot](https://www.dovecot.org/) server and experiment the following settings:
- auth_debug
- auth_debug_passwords
- auth_verbose
- auth_verbose_passwords
- auth_anonymous_username

We will use the Docker image provided by [dovecot](https://hub.docker.com/r/dovecot/dovecot) for this usage. The configuration files that we will use are slight variations of the one [provided by dovecot](https://github.com/dovecot/docker/blob/6b6020b7f8e81155bdd03fb04dd3318226e4fecb/2.3.21/dovecot.conf).

## Default configuration
This command runs a dovecot server with the default configuration:
```
docker run dovecot/dovecot:2.3.21
```
We can see the running services using `nmap`:
```
$nmap -p- 172.17.0.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-04-01 12:17 CEST
Nmap scan report for 172.17.0.2
Host is up (0.00011s latency).
Not shown: 65528 closed tcp ports (conn-refused)
PORT     STATE SERVICE
24/tcp   open  priv-mail
110/tcp  open  pop3
143/tcp  open  imap
587/tcp  open  submission
993/tcp  open  imaps
995/tcp  open  pop3s
4190/tcp open  sieve

Nmap done: 1 IP address (1 host up) scanned in 1.51 seconds
```
POP3 and IMAP are running on ports 110 and 143, and their SSL versions are running on ports 995 and 993. There are also an LMTP server is running on port 24. We will focus only on these services here.
### Send an email
To be able to use POP3 and IMAP services, we need at least one email to work with. We will use the LMTP service for this purpose. The commands used are very similar to SMTP commands. We will use `telnet` to connect to the LMTP server:
```
telnet 172.17.0.2 24
```
As authentication is not required, we can send an email from any user to any user:
```
C: MAIL FROM:<sender>
S: 250 2.1.0 OK
C: RCPT TO:<john>
S: 250 2.1.5 OK
C: DATA
S: 354 OK
C: This is the content of the email.
C: .
S: 250 2.0.0 <john> 1FVlDeKLCmYeAAAAvwCTPA Saved
```
Each line are prefixed by `C: ` if the data are sent by the client, and by `S: ` if they are received from the server.
### Get the email using IMAP
We are now able to use IMAP to get this email. With the default configuration of the server, it is not possible to log into the IMAP service with the non-secure version. We will need to use IMAPS:
```
openssl s_client -connect 172.17.0.2:imaps
```
We can log as `john` with the password `pass` (which is the same for all the users by default), and retrieve the email:
```
C: 1 LOGIN john pass
S: 1 OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS BINARY MOVE SNIPPET=FUZZY PREVIEW=FUZZY PREVIEW STATUS=SIZE SAVEDATE LITERAL+ NOTIFY] Logged in
C: 1 SELECT INBOX
S: * FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
S: * OK [PERMANENTFLAGS (\Answered \Flagged \Deleted \Seen \Draft \*)] Flags permitted.
S: * 1 EXISTS
S: * 1 RECENT
S: * OK [UNSEEN 1] First unseen.
S: * OK [UIDVALIDITY 1711967285] UIDs valid
S: * OK [UIDNEXT 2] Predicted next UID
S: 1 OK [READ-WRITE] Select completed (0.003 + 0.000 + 0.002 secs).
C: 1 FETCH 1 RFC822
S: * 1 FETCH (FLAGS (\Seen \Recent) RFC822 {234}
S: Return-Path: <>
S: Delivered-To: john
S: Received: from unknown ([172.17.0.1])
S: 	by ce9672af7b36 with LMTP
S: 	id 1FVlDeKLCmYeAAAAvwCTPA
S: 	(envelope-from <>)
S: 	for <john>; Mon, 01 Apr 2024 10:27:44 +0000
S: This is the content of the email.
S: )
S: 1 OK Fetch completed (0.003 + 0.000 + 0.002 secs).
```
### Get the email using POP3
The process to get the email using POP3 is pretty much the same as using IMAP. We also need to use SSL to be able to log:
```
openssl s_client -connect 172.17.0.2:pop3s
```
The only differences are the commands used:
```
C: USER john
S: +OK
C: PASS pass
S: +OK Logged in.
C: LIST
S: +OK 1 messages:
S: 1 234
S: .
C: retr 1
S: +OK 234 octets
S: Return-Path: <>
S: Delivered-To: john
S: Received: from unknown ([172.17.0.1])
S: 	by ce9672af7b36 with LMTP
S: 	id 1FVlDeKLCmYeAAAAvwCTPA
S: 	(envelope-from <>)
S: 	for <john>; Mon, 01 Apr 2024 10:27:44 +0000
S: This is the content of the email.
S: .
```
Just a note: `retr` is written lowercase because, if not, the client tries to renegotiate with the server:
```
C: RETR 1
S: RENEGOTIATING
```
## auth_debug setting
According to the documentation, [`auth_debug`](https://doc.dovecot.org/admin_manual/debugging/debugging_authentication/) "makes Dovecot log a debug line for just about anything related to authentication." This can be an issue in a production server if it logs sensitive information, and an attacker can access to these logs.
To look the impact of this setting on the server logs, we will add `auth_debug=yes` in the `dovecot.conf` file and try to log with a user.
The server can be run with this configuration file using this command:
```
docker run -v $(pwd)/auth_debug/dovecot.conf:/etc/dovecot/dovecot.conf dovecot/dovecot:2.3.21
```
We can already see that `auth_debug` is enabled in the setup logs of the server:
```
Apr 01 11:41:45 auth: Debug: Loading modules from directory: /usr/lib/dovecot/modules/auth
Apr 01 11:41:45 auth: Debug: Module loaded: /usr/lib/dovecot/modules/auth/lib20_auth_var_expand_crypt.so
Apr 01 11:41:45 auth: Debug: Module loaded: /usr/lib/dovecot/modules/auth/libdriver_mysql.so
Apr 01 11:41:45 auth: Debug: Module loaded: /usr/lib/dovecot/modules/auth/libdriver_pgsql.so
Apr 01 11:41:45 auth: Debug: Module loaded: /usr/lib/dovecot/modules/auth/libdriver_sqlite.so
Apr 01 11:41:45 auth: Debug: Wrote new auth token secret to /run/dovecot/auth-token-secret.dat
Apr 01 11:41:45 auth: Debug: auth client connected (pid=11)
Apr 01 11:41:45 auth: Debug: auth client connected (pid=10)
Apr 01 11:41:45 auth: Debug: auth client connected (pid=12)
Apr 01 11:41:45 auth: Debug: auth client connected (pid=9)
```
We will now try to log on the IMAP service:
```
$openssl s_client -connect 172.17.0.2:imaps -quiet
S: Can't use SSL_get_servername
S: depth=0 CN = localhost
S: verify error:num=18:self-signed certificate
S: verify return:1
S: depth=0 CN = localhost
S: verify return:1
S: * OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE LITERAL+ AUTH=PLAIN] Dovecot ready.
C: 1 LOGIN john pass
S: 1 OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS BINARY MOVE SNIPPET=FUZZY PREVIEW=FUZZY PREVIEW STATUS=SIZE SAVEDATE LITERAL+ NOTIFY] Logged in
```
There are no differences from a user point of view, but we can get the following log lines on the server side:
```
Apr 01 11:43:56 auth: Debug: client in: AUTH	2	PLAIN	service=imap	secured=tls	session=c3czgQcVdMKsEQAB	lip=172.17.0.2	rip=172.17.0.1	lport=993	rport=49780	resp=<hidden>
Apr 01 11:43:56 auth: Debug: static(john,172.17.0.1,<c3czgQcVdMKsEQAB>): Performing passdb lookup
Apr 01 11:43:56 auth: Debug: static(john,172.17.0.1,<c3czgQcVdMKsEQAB>): lookup
Apr 01 11:43:56 auth: Debug: static(john,172.17.0.1,<c3czgQcVdMKsEQAB>): Finished passdb lookup
Apr 01 11:43:56 auth: Debug: auth(john,172.17.0.1,<c3czgQcVdMKsEQAB>): Auth request finished
Apr 01 11:43:56 auth: Debug: client passdb out: OK	2	user=john
Apr 01 11:43:56 auth: Debug: master in: REQUEST	1370488833	12	2	f9eda57c7f93d413bdc08c9d13e9396d	session_pid=20	request_auth_token
Apr 01 11:43:56 auth: Debug: static(john,172.17.0.1,<c3czgQcVdMKsEQAB>): Performing userdb lookup
Apr 01 11:43:56 auth: Debug: static(john,172.17.0.1,<c3czgQcVdMKsEQAB>): Finished userdb lookup
Apr 01 11:43:56 auth: Debug: master userdb out: USER	1370488833	john		auth_mech=PLAIN	auth_token=1eb97cd7b3c289ea8a6d011b32581e0e6ada7432
Apr 01 11:43:56 imap-login: Info: Login: user=<john>, method=PLAIN, rip=172.17.0.1, lip=172.17.0.2, mpid=20, TLS, session=<c3czgQcVdMKsEQAB>
```
We can see that `john` successfully logged into the service, and no sensitive information is displayed. Let's try the case where `john` provides a wrong password:
```
C: 1 LOGIN john john
S: 1 NO [AUTHENTICATIONFAILED] Authentication failed.
```
The logs are:
```
Apr 01 11:46:53 auth: Debug: auth client connected (pid=12)
Apr 01 11:46:59 auth: Debug: client in: AUTH	3	PLAIN	service=imap	secured=tls	session=x30TjAcVnLCsEQAB	lip=172.17.0.2	rip=172.17.0.1	lport=993	rport=45212	resp=<hidden>
Apr 01 11:46:59 auth: Debug: static(john,172.17.0.1,<x30TjAcVnLCsEQAB>): Performing passdb lookup
Apr 01 11:46:59 auth: Debug: static(john,172.17.0.1,<x30TjAcVnLCsEQAB>): lookup
Apr 01 11:46:59 auth: Info: static(john,172.17.0.1,<x30TjAcVnLCsEQAB>): Password mismatch
Apr 01 11:46:59 auth: Debug: static(john,172.17.0.1,<x30TjAcVnLCsEQAB>): Finished passdb lookup
Apr 01 11:46:59 auth: Debug: auth(john,172.17.0.1,<x30TjAcVnLCsEQAB>): Auth request finished
Apr 01 11:47:01 auth: Debug: client passdb out: FAIL	3	user=john
```
We can see why the login failed ("Password mismatch"). However, we cannot see any passwords in the logs.
## auth_debug_passwords setting
This time, we will try the `auth_debug_passwords` setting which "will log [passwords] in plaintext". Let's see that by running a new server with this configuration:
```
docker run -v $(pwd)/auth_debug_passwords/dovecot.conf:/etc/dovecot/dovecot.conf dovecot/dovecot:2.3.21
```
After a successful login attempt, we get these log lines:
```
Apr 01 11:51:43 auth: Debug: client in: AUTH	1	PLAIN	service=imap	secured=tls	session=QesJnQcVbL6sEQAB	lip=172.17.0.2	rip=172.17.0.1	lport=993	rport=48748	resp=AGpvaG4AcGFzcw== (previous base64 data may contain sensitive data)
Apr 01 11:51:43 auth: Debug: static(john,172.17.0.1,<QesJnQcVbL6sEQAB>): Performing passdb lookup
Apr 01 11:51:43 auth: Debug: static(john,172.17.0.1,<QesJnQcVbL6sEQAB>): lookup
Apr 01 11:51:43 auth: Debug: static(john,172.17.0.1,<QesJnQcVbL6sEQAB>): Finished passdb lookup
Apr 01 11:51:43 auth: Debug: auth(john,172.17.0.1,<QesJnQcVbL6sEQAB>): Auth request finished
Apr 01 11:51:43 auth: Debug: client passdb out: OK	1	user=john
Apr 01 11:51:43 auth: Debug: master in: REQUEST	2790129665	12	1	6aa4ac7e3d75e702f1043a7723fa10c3	session_pid=19	request_auth_token
Apr 01 11:51:43 auth: Debug: static(john,172.17.0.1,<QesJnQcVbL6sEQAB>): Performing userdb lookup
Apr 01 11:51:43 auth: Debug: static(john,172.17.0.1,<QesJnQcVbL6sEQAB>): Finished userdb lookup
Apr 01 11:51:43 auth: Debug: master userdb out: USER	2790129665	john		auth_mech=PLAIN	auth_token=a4a0b48145190440b312dd92445afe33d040de9c
```
We can see a base64 encoded data: "AGpvaG4AcGFzcw==". If we decode it, we can find the login and the password:
```
$echo "AGpvaG4AcGFzcw==" | base64 --decode | xxd
00000000: 006a 6f68 6e00 7061 7373                 .john.pass
```
In the case of a failing attempt, we have the following log lines:
```
Apr 01 11:55:44 auth: Debug: client in: AUTH	2	PLAIN	service=imap	secured=tls	session=+elfqwcVNJisEQAB	lip=172.17.0.2	rip=172.17.0.1	lport=993	rport=38964	resp=AGpvaG4Ad3JvbmdfcGFzc3dvcmQ= (previous base64 data may contain sensitive data)
Apr 01 11:55:44 auth: Debug: static(john,172.17.0.1,<+elfqwcVNJisEQAB>): Performing passdb lookup
Apr 01 11:55:44 auth: Debug: static(john,172.17.0.1,<+elfqwcVNJisEQAB>): lookup
Apr 01 11:55:44 auth: Info: static(john,172.17.0.1,<+elfqwcVNJisEQAB>): Password mismatch
Apr 01 11:55:44 auth: Debug: static(john,172.17.0.1,<+elfqwcVNJisEQAB>): PLAIN(wrong_password) != 'pass'
Apr 01 11:55:44 auth: Debug: static(john,172.17.0.1,<+elfqwcVNJisEQAB>): Finished passdb lookup
Apr 01 11:55:44 auth: Debug: auth(john,172.17.0.1,<+elfqwcVNJisEQAB>): Auth request finished
Apr 01 11:55:46 auth: Debug: client passdb out: FAIL	2	user=john
```
This time, we can see in clear text the password of the user and the given password: `PLAIN(wrong_password) != 'pass'`. In some cases, the given password just have a typo but, in some other cases, it can be the password for another account of the user.
## auth_verbose setting
The `auth_verbose` setting "enables logging all failed authentication attempts." The dovecot server with this configuration can be run with this command:
```
docker run -v $(pwd)/auth_verbose/dovecot.conf:/etc/dovecot/dovecot.conf dovecot/dovecot:2.3.21
```
After a successful login attempt, we do not have more information than without this setting:
```
Apr 01 12:03:54 imap-login: Info: Login: user=<john>, method=PLAIN, rip=172.17.0.1, lip=172.17.0.2, mpid=19, TLS, session=<TjOQyAcVau2sEQAB>
```
However, in case of a failed attempt, we have this line:
```
Apr 01 12:04:21 auth: Info: static(john,172.17.0.1,<BaEtygcV6sGsEQAB>): Password mismatch
```
No sensitive information are displayed in the log file. It can also be interesting to detect a potential brute-force attack.
## auth_verbose_passwords setting
The `auth_verbose_passwords` setting can take three values: "no", "plain" or "sha1". If this setting is not set to "no", when a failing login attempt occurs, the password will be displayed in the log. It will be displayed in plaintext if the setting is set to "plain", in SHA1 hash format if "sha1" is set. The "sha1" option value can be useful to check if a user tries the same password multiple times or to detect a potential brute-force attack. With the following command, the server is configured to log the password in plaintext:
```
docker run -v $(pwd)/auth_verbose_passwords/dovecot.conf:/etc/dovecot/dovecot.conf dovecot/dovecot:2.3.21
```
With this setting, the given password appears in clear in the log:
```
Apr 01 12:12:40 auth: Info: static(john,172.17.0.1,<1xnw5wcV/I2sEQAB>): Password mismatch (given password: wrong_password)
```
## auth_anonymous_username setting
The `auth_anonymous_username` setting "specifies the username to be used for users logging in with the ANONYMOUS SASL mechanism." It is useful when the anonymous authentication mechanism is on. This can be done with `auth_mechanisms` set to `anonymous`. For instance, `auth_mechanisms=plain anonymous` will enable both PLAIN and ANONYMOUS authentication mechanisms. We can use the following command to run a server with this configuration:
```
docker run -v $(pwd)/auth_anonymous_username/dovecot.conf:/etc/dovecot/dovecot.conf dovecot/dovecot:2.3.21
```
When we start an IMAP connection, we can see that ANONYMOUS login mechanism is enabled in the list of the capabilities of the server:
```
$openssl s_client -connect 172.17.0.2:imaps -quiet
Can't use SSL_get_servername
depth=0 CN = localhost
verify error:num=18:self-signed certificate
verify return:1
depth=0 CN = localhost
verify return:1
* OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE LITERAL+ AUTH=PLAIN AUTH=ANONYMOUS] Dovecot ready.
```
We can now use this capability to log as `john`:
```
C: 1 AUTHENTICATE ANONYMOUS
S: + 
C: 
S: 1 OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS BINARY MOVE SNIPPET=FUZZY PREVIEW=FUZZY PREVIEW STATUS=SIZE SAVEDATE LITERAL+ NOTIFY] Logged in
```
We can see in the server log that `john` has been logged:
```
Apr 01 12:32:26 imap-login: Info: Login: user=<john>, method=ANONYMOUS, rip=172.17.0.1, lip=172.17.0.2, mpid=19, TLS, session=<vWuaLggVBLGsEQAB>
```
## References
- [RFC 3501](https://datatracker.ietf.org/doc/html/rfc3501) (INTERNET MESSAGE ACCESS PROTOCOL - VERSION 4rev1)
- [RFC 1939](https://www.ietf.org/rfc/rfc1939.txt) (Post Office Protocol - Version 3)
- [RFC 2245](https://datatracker.ietf.org/doc/html/rfc2245) (Anonymous SASL Mechanism)
- [Dovecot manual](https://doc.dovecot.org/)