---
title: Hack the box Nest walkthrough 
date: 2020-06-02 08:46:00 +0800
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [TAG]     # TAG names should always be lowercase
---
# overview 
tools used: nmap,smbclient,smbget, dot net fiddle, dnspy

# service enumeration
first, I run Nmap to find open ports, services, and service versions.
```bash
kali@kali:~/htb/nest$ Nmap -T4 -A -p- 10.10.10.178 -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-27 15:43 EDT
Nmap scan report for 10.10.10.178
The host is up (0.057s latency).
Not shown: 65533 filtered ports
PORT STATE SERVICE VERSION
445/tcp open Microsoft-ds?
4386/tcp open unknown

| fingerprint-strings:
| DNSStatusRequestTCP, DNSVersionBindReqTCP, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, NULL, RPCCheck, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, X11Probe:
| Reporting Service V1.2
| FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, RTSPRequest, SIPOptions:
| Reporting Service V1.2
| Unrecognised command
| Help:
| Reporting Service V1.2
| This service allows users to run queries against databases using the legacy HQK format
| AVAILABLE COMMANDS ---
| LIST
| SETDIR <Directory_Name>
| RUNQUERY <Query_ID>
| DEBUG <Password>
|_ HELP <Command>

1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
Host script results:

|_clock-skew: 3m38s
| smb2-security-mode:
| 2.02:
|_ Message signing enabled but not required
| smb2-time:
| date: 2020-05-27T19:51:29
|_ start_date: 2020-05-27T19:41:37

Service detection performed. Please report any incorrect results at https://nmap.org/submit/.
Nmap did: 1 IP address (1 host up) scanned in 301.07 seconds
kali@kali:~/htb/nest$
```
the Nmap scan tells me that there are two ports open 445/smb and 4386/HQK reporting I SMB is what I am most familiar with so I will start with that.
# SMB share enumeration 
the following command lists the shares.
```bash
kali@kali:~/htb/nest$ smbclient -L \\\\10.10.10.178
Enter WORKGROUP\kali's password:

        Sharename      Type      Comment
        ---------      ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        Data            Disk     
        IPC$            IPC      Remote IPC
        Secure$        Disk     
        Users          Disk     
SMB1 disabled -- no workgroup available
```
the Data share seams to be accessible anonymously.

```bash
kali@kali:~/htb/nest$ smbclient  \\\\10.10.10.178\\Data
Enter WORKGROUP\kali's password:
Try "help" to get a list of possible commands.
smb: \> recurse on
smb: \> ls
  .                                  D        0  Wed Aug  7 18:53:46 2019
  ..                                  D        0  Wed Aug  7 18:53:46 2019
  IT                                  D        0  Wed Aug  7 18:58:07 2019
  Production                          D        0  Mon Aug  5 17:53:38 2019
  Reports                            D        0  Mon Aug  5 17:53:44 2019
  Shared                              D        0  Wed Aug  7 15:07:51 2019

\IT
NT_STATUS_ACCESS_DENIED listing \IT\*
\Production
NT_STATUS_ACCESS_DENIED listing \Production\*
\Reports
NT_STATUS_ACCESS_DENIED listing \Reports\*
\Shared
  .                                  D        0  Wed Aug  7 15:07:51 2019
  ..                                 D        0  Wed Aug  7 15:07:51 2019
  Maintenance                        D        0  Wed Aug  7 15:07:32 2019
  Templates                          D        0  Wed Aug  7 15:08:07 2019
\Shared\Maintenance
  .                                  D        0  Wed Aug  7 15:07:32 2019
  ..                                 D        0  Wed Aug  7 15:07:32 2019
  Maintenance Alerts.txt             A       48  Mon Aug  5 19:01:44 2019
\Shared\Templates
  .                                  D        0  Wed Aug  7 15:08:07 2019
  ..                                 D        0  Wed Aug  7 15:08:07 2019
  HR                                 D        0  Wed Aug  7 15:08:01 2019
  Marketing                          D        0  Wed Aug  7 15:08:06 2019
\Shared\Templates\HR
  .                                  D        0  Wed Aug  7 15:08:01 2019
  ..                                 D        0  Wed Aug  7 15:08:01 2019
  Welcome Email.txt                  A      425  Wed Aug  7 18:55:36 2019
\Shared\Templates\Marketing
  .                                  D        0  Wed Aug  7 15:08:06 2019
  ..                                 D        0  Wed Aug  7 15:08:06 2019
smb: \>
```

the `Maintenance Alerts.txt` and `Welcome Email.txt` look interesting
you can retrieve them using the following commands. 
```bash
kali@kali:~/htb/nest$ smbclient  \\\\10.10.10.178\\Data
Enter WORKGROUP\kali's password:
Try "help" to get a list of possible commands.
smb: \> cd \Shared\Maintenance
smb: \Shared\Maintenance\> mget "Maintenance Alerts.txt"
Get file Maintenance Alerts.txt? Yes
getting file \Shared\Maintenance\Maintenance Alerts.txt of size 48 as Maintenance Alerts.txt (0.2 KiloBytes/sec) (average 0.2 KiloBytes/sec)
smb: \Shared\Maintenance\> cd ../Templates/HR
smb: \Shared\Templates\HR\> mget "Welcome Email.txt"
Get file Welcome Email.txt? Yes
getting file \Shared\Templates\HR\Welcome Email.txt of size 425 as Welcome Email.txt (2.0 KiloBytes/sec) (average 1.1 KiloBytes/sec)
smb: \Shared\Templates\HR\>
```
the "Welcome Email.txt" file has smb login creds
```
kali@kali:~/htb/nest$ cat Welcome\ Email.txt
We would like to extend a warm welcome to our newest member of staff, <FIRSTNAME> <SURNAME>

You will find your home folder in the following location:
\\HTB-NEST\Users\<USERNAME>

If you have any issues accessing specific services or workstations, please inform the
IT department and use the credentials below until all systems have been set up for you.

Username: TempUser
Password: welcome2019


Thank you
HR
```
TempUser has access to  a visual basic project at Secure$\IT\Carl you can download all of carls files using the following command
```bash
smbget -rR smb://10.10.10.178/Secure$/IT/Carl/ -U TempUser
```
you can move the project files from /Secure$/IT/Carl//VB Projects/WIP/RU/RUScanner/  to [dotnetfiddle.net](https://dotnetfiddle.net/) this lets us use the code to decrypt the encrypted password in **RU_Config.xml**

still seting up the blog and writing the post sorry

