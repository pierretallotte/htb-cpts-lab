In this section, we will set up an [OpenSSH](https://www.openssh.com/) server and experiment the following settings:
- [PasswordAuthentication](#passwordauthentication)
- [PermitEmptyPasswords](#permitemptypasswords)
- [PermitRootLogin](#permitrootlogin)

## Default configuration
We will use a Docker image based on [Debian](https://hub.docker.com/_/debian) and install `openssh-server` on it. We will create a user `john` with the password `pass` and set the root password to `root`. This is the command to run the server:
```
docker build -t ssh_default_configuration $(pwd)/default_configuration && docker run -d ssh_default_configuration
```
We can check an SSH server is running using `nmap`:
```
$nmap -sV -p22 172.17.0.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-04-06 14:07 CEST
Nmap scan report for 172.17.0.2
Host is up (0.00011s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.32 seconds
```
Then, we can use SSH to log as `john` and use `su` to get a root shell:
```
$ssh john@172.17.0.2
The authenticity of host '172.17.0.2 (172.17.0.2)' can't be established.
ED25519 key fingerprint is SHA256:EG74MDHg2w15uJISVlbWwLu1YHApFLaWio65TlTL7Nk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.2' (ED25519) to the list of known hosts.
john@172.17.0.2's password: 
Linux dfeef8ff94d0 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
$ whoami
john
$ su root
Password: 
root@dfeef8ff94d0:/home/john# 
```
We can look at the default configuration:
```
root@dfeef8ff94d0:/home/john# cat /etc/ssh/sshd_config | grep -v "#" | sed -r '/^\s*$/d'
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
```
By default, there are no configuration files in `/etc/ssh/sshd_config.d/*.conf`:
```
root@dfeef8ff94d0:/home/john# ls /etc/ssh/sshd_config.d/*.conf
ls: cannot access '/etc/ssh/sshd_config.d/*.conf': No such file or directory
```
## PasswordAuthentication
By default, this setting is set to `no`. However, we can see password authentication is enabled using the default configuration:
```
$ssh john@172.17.0.2
john@172.17.0.2's password: 
Permission denied, please try again.
john@172.17.0.2's password: 
Permission denied, please try again.
john@172.17.0.2's password: 
john@172.17.0.2: Permission denied (publickey,password).
```
This is because `UsePAM` is set to `yes`. If we want to disable password authentication, we need to disable PAM authentication. Here is the command to run a server with password authentication disabled:
```
docker run -d -v $(pwd)/PasswordAuthentication/sshd_config:/etc/ssh/sshd_config ssh_default_configuration
```
We can see that we cannot log as `john` with a password anymore:
```
$ssh john@172.17.0.3
john@172.17.0.3: Permission denied (publickey).
```
With this configuration, you cannot log in to the server because no public keys have been added in the `authorized_keys` file. Use this command to add a public key for the user `john`:
```
ssh-keygen -t rsa -b 3072 -f /tmp/john_keys -N "" && docker run -d -v /tmp/john_keys.pub:/home/john/.ssh/authorized_keys -v $(pwd)/PasswordAuthentication/sshd_config:/etc/ssh/sshd_config ssh_default_configuration
```
This command generates a private-public key pair and stores it into the `authorized_keys` file of the user `john`. You can now log in as the `john` user using this key pair:
```
$ssh -i /tmp/john_keys john@172.17.0.3
$ whoami
john
```
## PermitEmptyPasswords
According to the [documentation](https://man.freebsd.org/cgi/man.cgi?sshd_config(5)), this setting allows login to accounts with empty password stings. This can only be enabled if password authentication is enabled. If this option is not enabled, you cannot log with a user that has no password. For instance, with the default configuration, we cannot log in with the user `mary`:
```
$ssh mary@172.17.0.2
mary@172.17.0.2's password: 
Permission denied, please try again.
mary@172.17.0.2's password: 
Permission denied, please try again.
mary@172.17.0.2's password: 
mary@172.17.0.2: Permission denied (publickey,password).
```

To start a server with `PermitEmptyPasswords` set to `yes`, we can use the following command:
```
docker run -d -v $(pwd)/PermitEmptyPasswords/sshd_config:/etc/ssh/sshd_config ssh_default_configuration
```

We can now log in as `mary` with no password:
```
$ssh mary@172.17.0.4
Linux e94b14fda961 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
$ whoami
mary
```
When this option is set, we can try to bruteforce usernames. Here is an example with `hydra`:
```
$ echo -e "john\nmary\nroot" > /tmp/passlist
$hydra -L /tmp/passlist -e n 172.17.0.4 ssh
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-04-06 18:31:09
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 3 tasks per 1 server, overall 3 tasks, 3 login tries (l:3/p:1), ~1 try per task
[DATA] attacking ssh://172.17.0.4:22/
[22][ssh] host: 172.17.0.4   login: mary
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-04-06 18:31:16
```
We can see that `mary` has been detected as a user we want log with an empty password. Just note that bruteforcing an SSH server can be very long.
## PermitRootLogin
Without this setting set, it is not possible to log as the `root` user. We can check that with the default configuration:
```
$ssh root@172.17.0.2
root@172.17.0.2's password: root
Permission denied, please try again.
```
This setting can takes four values: `yes`, `prohibit-password`, `forced-commands-only` and `no`. I tried to set up a server with the option value `forced-commands-only`, but I had a mysterious error. See my question on [Server Fault](https://serverfault.com/questions/1157491/client-loop-send-disconnect-broken-pipe-error-when-trying-to-log-as-root-wit).

This command sets up the server to allow any authentication type for the `root` user:
```
docker run -d -v $(pwd)/PermitRootLogin/sshd_config:/etc/ssh/sshd_config ssh_default_configuration
```

We are now able to log as the root user:
```
$ssh root@172.17.0.5
root@172.17.0.5's password: 
Linux c57fd1192b50 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@c57fd1192b50:~#
```

Let's try to enable public key authentication for the `root` user. To do that, we need to generate a key pair and send it to the server:
```
$ssh-keygen -t rsa -b 3072 -f /tmp/root_keys -N "" 
Generating public/private rsa key pair.
Your identification has been saved in /tmp/root_keys
Your public key has been saved in /tmp/root_keys.pub
The key fingerprint is:
SHA256:lxELS4jadckXCd9MGOC8iU9/f3aojIkjA3/KSI1ktXI pierre@parrot
The key's randomart image is:
+---[RSA 3072]----+
|     . o=++=.    |
|    . o++++*     |
|   o ...+.+ o    |
|  . .. o o o     |
|    + E S o      |
|   o.= o o       |
|    oo. . . .  . |
|   . o+ o. = .. +|
|    . o=..o o..o.|
+----[SHA256]-----+
$ssh-copy-id -i /tmp/root_keys root@172.17.0.5
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/tmp/root_keys.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@172.17.0.5's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@172.17.0.5'"
and check to make sure that only the key(s) you wanted were added.
```
We can now log in without password:
```
$ssh -i /tmp/root_keys root@172.17.0.5
Linux c57fd1192b50 6.5.0-13parrot1-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.5.13-1parrot1 (2023-12-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Apr  6 16:17:18 2024 from 172.17.0.1
root@c57fd1192b50:~#
```
## References
- [SSH Academy](https://www.ssh.com/academy/ssh)
- [sshd_config man page](https://man.freebsd.org/cgi/man.cgi?sshd_config(5))