---
name: "Excorsist Ransomware analysis"
date: 2020-10-02
header: 
  teaser: /assets/img/xor-routine.png
---

Instead of attacking companies to deploy ransomware, the thread actors behind the Exorcist 2.0 ransomware are using a different way of attacking companies.

> The threat actors behind the Exorcist 2.0 ransomware are using malicious advertising to redirect victims to fake software crack sites that distribute their malware. - [https://www.bleepingcomputer.com/news/security/fake-software-crack-sites-used-to-push-exorcist-20-ransomware/](https://www.bleepingcomputer.com/news/security/fake-software-crack-sites-used-to-push-exorcist-20-ransomware/)

This blog post will try to explain how Excorsist 2.0 encrypts files using BCrypt API functions. This is the first time that I have focused more on analyzing the encryption method of ransomware. If there are mistakes in the blog post please send me a message. I am not a Crypto Expert!

### Anti-analysis

The ransomware resolves API calls dynamically by calling a function that accepts a first argument for example `0x69090810` based on this value an XOR routine is executed which results in the function name that will be called. Using `LoadLibrary` and `GetProcAddress` the function address is retrieved and called. This happens for every function the executable calls. This makes static analysis harder and requires dynamic analysis or a Ghidra script that can mimic the XOR routine. For example `IsDebuggerPresent`:

![image-20200930111837816]({{site.url}}/assets/img/debugger.png)


### Anti debug

To prevent the ransomware from being run by a debugger the following API calls are used to detect a debugger. This function is also called a few times before starting the encryption routine. I use x64dbg with [ScyllaHide](https://github.com/x64dbg/ScyllaHide) to prevent the process of exiting.

* IsDebuggerPresent
* CheckRemoteDebugger
* NtQueryInformationProcess - Checks the following PROCESSINFOCLASS (ProcessDebugPort)
* ProcessDebugObjectHandle - ProcessDebugFlags

![image-20200930112540949]({{site.url}}/assets/img/xor-routine.png)

### Preparation

To make sure only one copy of the ransomware is running the following mutex is created `6RvEPKk8vDyzP6PxmUJt2m8mVGh3r7SmtjfwGjfELqv2aC5`, if not the ransomware exits. After that to make sure system recovery is not possible the following commands are executed.

* Empty recycle bin - `SHEmptyRecycleBinW `
* Delete shadow copies - `vssadmin delete shadows /all /quiet`,  `wmic SHADOWCOPY /nointeractive`
* Disable recovery mode - `bcdedit /set {default} recoveryenabled no` , `bcdedit /set {default} bootstatuspolicy ignoreallfailures`
* Delete backps `wbadmin DELETE SYSTEMSTATEBACKUP -deleteOldest`, `wbadmin DELETE SYSTEMSTATEBACKUP`

Making sure most of the files are not opened by other processes the ransomware checks if a list of 53 process names is running. If true the processes are killed. Most of the process names are associated with data or are often used on Windows servers. Below are list some of the process names.

* sql
* msaccess
* wxServer
* wxServerView
* mssql

### Encryption

The ransomware starts with moving the file to the new name with extension using `MoveFileW(FileHandle, example.txt<extension>)`. Then it creates a `CreateIoCompletionPort` with a function renamed to `encrypt_write` as `CompletionKey`. By reading `0x10000` bytes from the FileHandle the function `encrypt_write` is triggered and encrypts the file. Each file encrypted using the AES algorithm with a newly generated AES key and IV for each file. The AES key is encrypted with an embedded public RSA key (4096 bits) and written to `example.txt<extention>key`. The encryption routine is described in the diagram below.

![image-20201002113750224]({{site.url}}/assets/img/encryption.png)

At the end, the ransomware executes the following commands and creates decrypt instructions in every folder (DECRYPT-[random]-decrypt.HTA).

* `cmd.exe /C wevtutil cl system`
* `cmd.exe /C timeout /T 15 /NOBREAK && del \"C:\\Users\\re\\Desktop\\samples\\excorsist.bin`

### IOC's

SHA256 - `dfecb46078038bcfa9d0b8db18bdc0646f33bad55ee7dd5ee46e61c6cf399620`

`51.83.171.35`

`http://51.83.171.35/fgate`

### References

[https://medium.com/@velasco.l.n/exorcist-ransomware-from-triaging-to-deep-dive-5b7da4263d81](https://medium.com/@velasco.l.n/exorcist-ransomware-from-triaging-to-deep-dive-5b7da4263d81)

[https://tria.ge/201001-qag4hak1ta/behavioral2](https://tria.ge/201001-qag4hak1ta/behavioral2)
