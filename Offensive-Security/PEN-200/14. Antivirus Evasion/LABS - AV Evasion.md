
```
2. **Capstone Lab**: In this exercise, you'll be facing off against _COMODO_ antivirus engine running on Module Exercise VM #1. Use another popular 32-bit application, like _PuTTY_, to replicate the steps learned so far in order to inject malicious code in the binary with Shellter. The victim machine runs an anonymous FTP server with open read/write permissions. Every few seconds, the victim user will double-click on any existing **.exe** file(s) in the FTP root directory. If the antivirus flags the script as malicious, the script will be quarantined and then deleted. Otherwise, the script will execute and hopefully, grant you a reverse shell. NOTE: set the FTP session as _active_ and enable _binary_ encoding while transferring the file.
```


```
3. **Capstone Lab**: Similar to the previous exercise, you'll be facing off against _COMODO_antivirus engine v12.2.2.8012 on Module Exercise VM #2. Although the PowerShell AV bypass we covered in this Module is substantial, it has an inherent limitation. The malicious script cannot be _double-clicked_ by the user for an immediate execution. Instead, it would open in _notepad.exe_ or another default text editor. The tradecraft of manually weaponizing PowerShell scripts is beyond the scope of this module, but we can rely on another open-source framework to help us automate this process. Research how to install and use the [_Veil_](https://github.com/Veil-Framework/Veil) framework to help you with this exercise.
```