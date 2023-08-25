---
name: Let's analyze Wasted Locker
date: 2020-08-07
header: 
  teaser: /assets/img/image-20200807001824526.png
---

This blog post will explain the workings of the ransomware named Wasted Locker based on 3 different samples. Last weeks I read a lot of articles about companies getting attack by Wasted Locker. Wasted Locker is ransomware used to attack large companies to make ransom demands. These amounts sometimes go up to $ 10 million.

> Employees later shared with BleepingComputer that the ransom demand was $10 million. - [https://www.bleepingcomputer.com/news/security/confirmed-garmin-received-decryptor-for-wastedlocker-ransomware/](https://www.bleepingcomputer.com/news/security/confirmed-garmin-received-decryptor-for-wastedlocker-ransomware)

### Static analysis

##### **5CD04805F9753CA08B82E88C27BF5426D1D356BB26B281885573051048911367**

* The program Detect It Easy detected the linker `Polink(2.50*)[EXE32,signed]`
* Compiler stamp `Thu Jun 11 06:10:48 2020 UTC`
* 18 imports 
* File extension - `rhl_wastedinfo`

##### **905EA119AD8D3E54CD228C458A1B5681ABC1F35DF782977A23812EC4EFA0288A**

* The program Detect It Easy detected the linker `Polink(2.50*)[EXE32,signed]`
* Compiler stamp `Wed Jul 22 20:43:17 2020 UTC`
* 64 imports
* File extension - `garminwasted`

#####  **BCDAC1A2B67E2B47F8129814DCA3BCF7D55404757EB09F1C3103F57DA3153EC8**

* The program Detect It Easy detected the linker `Microsoft Visual C/C++(2005)[-]`.
* Compiler stamp `Tue May 26 19:46:34 2020 UTC`
* 128 imports
* File extension - `bbawasted_info`

### Crypter

The first 2 samples are protected with a custom crypter, referred to as CryptOne by Fox-IT. To further analyse the unpacked executable I used x64dbg and Scylla to dump the memory to an executable.
> *WastedLocker* is protected with a custom crypter, referred to as ***CryptOne\*** by Fox-IT InTELL. On examination, the code turned out to be very basic and used also by other malware families such as: *Netwalker*, *Gozi ISFB v3*, *ZLoader* and *Smokeloader*. - [https://research.nccgroup.com/2020/06/23/wastedlocker-a-new-ransomware-variant-developed-by-the-evil-corp-group/](https://research.nccgroup.com/2020/06/23/wastedlocker-a-new-ransomware-variant-developed-by-the-evil-corp-group/)

Packed executables SHA-256 hashes

* 5CD04805F9753CA08B82E88C27BF5426D1D356BB26B281885573051048911367
* 905EA119AD8D3E54CD228C458A1B5681ABC1F35DF782977A23812EC4EFA0288A

The `main` function executes uses `LoadLibrary` to load `kernel32.dll` in memory and `GetProcAddress`  to retrieve the address of the function `VirtualAlloc`. `VirtualAlloc` is then used to create a memory with the memory protection constant `0x40 = PAGE_EXECUTE_READWRITE` ([https://docs.microsoft.com/en-us/windows/win32/memory/memory-protection-constants](https://docs.microsoft.com/en-us/windows/win32/memory/memory-protection-constants)).

![image-20200805174024690]({{site.url}}/assets/img/image-20200805174024690.png)

After that, another function writes shellcode to the new `memory_region`. At the end of the main function, it takes a jump to the created `memory_region_offset`.

![image-20200805174414526]({{site.url}}/assets/img/image-20200805174414526.png)

![image-20200805174601699]({{site.url}}/assets/img/image-20200805174601699.png)

To further analyse the instructions at new the `memory_region` I dumped to memory region to a file and opened it Ghidra at the specified offset. The crypter writes a "new" executable to a memory region, then uses a XOR-based algorithm to decrypt the code. And then writes the new memory region that contains a new executable to the address `0x00400000` of the original executable in memory.

`local_3a0` = new memory_region
`local_38c` = original memory address of executable in memory

![image-20200806233438155]({{site.url}}/assets/img/image-20200806233438155.png)

![image-20200806233925424]({{site.url}}/assets/img/image-20200806233925424.png)

After the crypter is done it jumps back to the entry point of the decrypted executable.

![image-20200805182713894]({{site.url}}/assets/img/image-20200805182713894.png)

Using x64dbg I dumped the memory to an executable to further analysis the ransomware.

### Ransomware Analysis

This part of the analysis is based on the sample with the following SHA256 hash. This file was already unpacked but works almost the same as the other samples.

* BCDAC1A2B67E2B47F8129814DCA3BCF7D55404757EB09F1C3103F57DA3153EC8

The ransomware starts with determining if it's get executed with elevated privileges. Using `GetTokenInformation` the `TokenElevation` is checked. The `TOKEN_ELEVATION` structure indicates whether a token has elevated privileges. If the process runs with elevated privileges `0x1` is written to the buffer `elevated_flag = TokenInformation`, otherwise it writes `0x0` to `elevated_flag`.

![image-20200806022101680]({{site.url}}/assets/img/image-20200806022101680.png)

If the major version number of the Windows version (https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-osversioninfoexa) is bigger then 5, so Windows Vista or later and if the process is not running with elevated privileges (`elevated_flag = 0`) it will execute the function `elevate_priviliges`.

![image-20200806172027825]({{site.url}}/assets/img/image-20200806172027825.png)

The function `elevate_priviliges` copies a "random" DLL in the directory `C:\WINDOWS\system32\` to the folder `C:\Users\<username>\AppData\Roaming\<random>` using `CopyFileW`.

![image-20200806185257942]({{site.url}}/assets/img/image-20200806185257942.png)

It then sets the file attribute of to new file location to `0x2 = FILE_ATTRIBUTE_HIDDEN`.

![image-20200806185618885]({{site.url}}/assets/img/image-20200806185618885.png)

**Alternate Data Streams** ([https://blog.malwarebytes.com/101/2015/07/introduction-to-alternate-data-streams/ ](https://blog.malwarebytes.com/101/2015/07/introduction-to-alternate-data-streams/))

Instead of just writing the executable to copied DLL, the ransomware uses the file attribute ADS. This a way of hiding the ransomware in a not suspicious file. The function `ReadFile` reads the file that started the process, then `CreateFileW` is used to create the ADS in the `<random-DLL>:bin` and `WriteFile` writes the buffer that contains the executable to the ADS of the `<random-DLL>:bin`.

![image-20200806190725263]({{site.url}}/assets/img/image-20200806190725263.png)

The buffer shows the following in the dump.

![image-20200806190627400]({{site.url}}/assets/img/image-20200806190627400.png)

After that, a new directory is created in the temporary folder (`%temp%\<random>\`), for example, `C:\Users\re\AppData\Local\Temp\D8BB`. Then another folder is created with the name `"C:\\Windows " `. Note the extra space! 

![image-20200806191915949]({{site.url}}/assets/img/image-20200806191915949.png)

Before copying files `NTcreate` is called with the flag `IO_REPARSE_TAG_MOUNT_POINT` to `%temp%\<random>\` and the directory   `%temp%\<random>\system32` is created. Using `CopyFileW` `winsat.exe` and the DDL `winmm.dll` is copied to `%temp%\<random>\system32\`  and then the entry point of `winmm.dll` is overwritten with new shellcode shown below.

![image-20200806215603764]({{site.url}}/assets/img/image-20200806215603764.png)

To launch `winsat.exe` the function `ShellExecuteExW` is called which starts the following processes.  `winsat.exe` is executed with elevated privileges and loads the DLL `winmm.dll` and `winmm.dll` is patched with shellcode that executes the ransomware hidden in the ADS `C:\Users\re\AppData\Roaming\<random>:bin`. The ransomware uses a vulnerability called DLL hijacking ([https://www.bleepingcomputer.com/news/security/bypassing-windows-10-uac-with-mock-folders-and-dll-hijacking/](https://www.bleepingcomputer.com/news/security/bypassing-windows-10-uac-with-mock-folders-and-dll-hijacking/)).

![image-20200807001824526]({{site.url}}/assets/img/image-20200807001824526.png)

### Encryption

To use CryptoAPI functions of Windows the function `CryptAcquireContextW` is called. The encryption method used is [RSA](https://docs.microsoft.com/nl-be/windows/win32/seccrypto/prov-rsa-full) public key alogrithm, because the `dwProvType` parameter is [0x1](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gpnap/e58d0d81-6cb4-4e07-bbc3-1e27978c1e72)  which stands for Cryptographic Provider Type [PROV_RSA_FULL](https://docs.microsoft.com/nl-be/windows/win32/seccrypto/prov-rsa-full).

![image-20200805203626391]({{site.url}}/assets/img/image-20200805203626391.png)

Then `CryptGenRandom` is called which fills the buffer `random_bytes` with cryptographically random bytes.

![image-20200806225321100]({{site.url}}/assets/img/image-20200806225321100.png)

This buffer is used by other functions that execute the encryption routine.

Instead of copying the file that will be encrypted `MoveFileW` is used to move the file to `<file-name>.bbawasted`. The extension is not the same in all samples sometimes the name of the company being attacked is used here.

At this moment I was stuck because I didn't understand how the files were being encrypted. Luckily Sophos wrote a blog post ([https://news.sophos.com/en-us/2020/08/04/wastedlocker-techniques-point-to-a-familiar-heritage/](https://news.sophos.com/en-us/2020/08/04/wastedlocker-techniques-point-to-a-familiar-heritage/)) about the technique used.

> Each file is encrypted using the AES algorithm with a newly generated AES key and IV (256-bit in CBC mode) for each file. The AES key and IV are encrypted with an embedded public RSA key (4096 bits). The RSA encrypted output of key material is converted to base64 and then stored into the ransom note. - [https://research.nccgroup.com/2020/06/23/wastedlocker-a-new-ransomware-variant-developed-by-the-evil-corp-group/](https://research.nccgroup.com/2020/06/23/wastedlocker-a-new-ransomware-variant-developed-by-the-evil-corp-group/)

After encrypting the file the ransomware note is written to `<file-name>.bbawasted_info` with the key used to encrypt the file. The file `update.py` is not associated with the ransomware, it is just a random file in my home directory.

![image-20200806225445548]({{site.url}}/assets/img/image-20200806225445548.png)

I hope I was able to explain in a technical way how Wasted Locker works. I did not write about the arguments that can be given to the ransomware which executed different functions based on that, but it is already covered in another blog posts in the references. Next time I want to better analyze the encryption method used by the ransomware.

### IOC's

* 5CD04805F9753CA08B82E88C27BF5426D1D356BB26B281885573051048911367
* 905EA119AD8D3E54CD228C458A1B5681ABC1F35DF782977A23812EC4EFA0288A
* BCDAC1A2B67E2B47F8129814DCA3BCF7D55404757EB09F1C3103F57DA3153EC8

### Triage reports

* [https://tria.ge/200625-a7f1nq69h6/behavioral2](https://tria.ge/200625-a7f1nq69h6/behavioral2)
* [https://tria.ge/200726-jgqm86w26x/behavioral2](https://tria.ge/200726-jgqm86w26x/behavioral2)
* [https://tria.ge/200625-ykwrj3nmz6/behavioral2](https://tria.ge/200625-ykwrj3nmz6/behavioral2)

### References

* [https://research.nccgroup.com/2020/06/23/wastedlocker-a-new-ransomware-variant-developed-by-the-evil-corp-group/](https://research.nccgroup.com/2020/06/23/wastedlocker-a-new-ransomware-variant-developed-by-the-evil-corp-group/)
* [https://news.sophos.com/en-us/2020/08/04/wastedlocker-techniques-point-to-a-familiar-heritage/](https://news.sophos.com/en-us/2020/08/04/wastedlocker-techniques-point-to-a-familiar-heritage/)
* [https://securelist.com/wastedlocker-technical-analysis/97944/](https://securelist.com/wastedlocker-technical-analysis/97944/)
