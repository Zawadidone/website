---
name: Let's analyze Ragnar Locker
date: 2020-06-01
header: 
  teaser: /assets/img/image-20200529212911106.png
---

After reading this [article](https://www.bleepingcomputer.com/news/security/ransomware-encrypts-from-virtual-machines-to-evade-antivirus/) about the Ragnar Locker ransomware running in a Windows XP VM to prevent it from being detected. I thought why not just analyze it to see what it does compare to other ransomware families. This blog post will further explain the ransomware using the programs Ghidra and x64dbg.

### Analysis

Using the function `GetLocalInfoW` the ransomware checks if the language on the computers is one of the following languages. And if true the process terminates itself.

```
Azerbaijani, Armenian, Belorussian, Kazakh, Kyrgyz, Moldavian, Tajik, Russian, Turkmen, Uzbek, Ukrainian, Georgian
```


![image-20200529171413730]({{site.url}}/assets/img/image-20200529171413730.png)

To make sure that there is only one process running it creates an event with the use of `CreateEventW`. If the event already exists the function `GetLastError` will return `0xb7 (ERROR_ALREADY_EXISTS)`, and after looping 32768 times the process will terminate itself. But if the event is created successfully the function `GetLastError` returns `0 (ERROR_SUCCESS)` and ransomware continues running. It does this to ensure that only one copy of the ransomware is running at a time.

![image-20200529222814164]({{site.url}}/assets/img/image-20200529222814164.png)

To ensure that large amounts of data can be encrypted. The ransomware checks whether the following services are running and if running it stops the services. This is also stated on the [malware wiki](https://malware.wikia.org/wiki/Ragnar_Locker).

```
vss,sql,memtas,mepocs,sophos,veeam,backup,pulseway,logme,logmein,connectwise,splashtop,kaseya
```

![image-20200531232343276]({{site.url}}/assets/img/image-20200531232343276.png)

After that, the function at memory address `00402150` renamed to `decrypt_string` is called to decrypt an RSA public key and a ransomware note in memory.

![image-20200531190214847]({{site.url}}/assets/img/image-20200531190214847.png)

**Public key**

```
 -----BEGIN PUBLIC KEY-----\n
 MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3rt9EPkNBSGeoCGzU50f\nOaEgC3EdDSXvMT26aRlzsUcng/EZUlTKwYDYwHXdIuWvshUymKexyi/BLR1fGs5Y\n044BnrBqFPSgrjwarZw37wLTYqAKGR/5pTKxjwVuJ4ArC2A1XbYOlmhv2pbnVq4l\nq0juc6W2MNoK31Bfds3/lrLAqlu3KMMg43PCvI2IMooguRRm7NEvqSeuu5ZmuC/A\nv2/aNxSQoXfr2yS6JoZP7EFx/I00bkWWrHr4qhHppJrRVcJH8jGh9DDSuz7XzoW7\ntLAPQZKR8V29x5z0Yscgm64Bd60uj3Fp9N7xqRDWZUKZQ+om9yTRhpsi8gORGrVp\nMQIDAQAB\n
 -----END PUBLIC KEY-----\n
```

**Ransomware note**

![image-20200531190802429]({{site.url}}/assets/img/image-20200531190802429.png)

To use CryptoAPI function of Windows the function `CryptAcquireContextW` is called. The encryption method used is [RSA](https://docs.microsoft.com/nl-be/windows/win32/seccrypto/prov-rsa-full) public key alogrithm, because the `dwProvType` parameter is [0x1](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gpnap/e58d0d81-6cb4-4e07-bbc3-1e27978c1e72)  which stands for Cryptographic Provider Type [PROV_RSA_FULL](https://docs.microsoft.com/nl-be/windows/win32/seccrypto/prov-rsa-full).

![image-20200531194111538]({{site.url}}/assets/img/image-20200531194111538.png)

The public key gets imported with the use of `CryptImportPublicKeyInfo`.

![image-20200531234017475]({{site.url}}/assets/img/image-20200531234017475.png)

The imported RSA public key is then used to encrypt the later used Salsa20 key and nonce.

![image-20200531235759021]({{site.url}}/assets/img/image-20200531234853376.png)

Before encrypting files the ransomware writes the ransomware note to the path `C:\Users\Public\Documents\RGNR_04BFF775.txt`. And then adds a `generated_secret` to file. I didn't further analyzed the way the `generated_secret` is created. I think this is for the malware authors to identify the malware sample used in a ransomware attack, but I don't know for sure.

![image-20200531200232105]({{site.url}}/assets/img/image-20200531200232105.png)

![image-20200531200311609]({{site.url}}/assets/img/image-20200531200311609.png)

**Encryption**

The encryption method skips the following folders, files and extensions. This is also stated on the [malware wiki](https://malware.wikia.org/wiki/Ragnar_Locker).

```
kernel32.dll, Windows, Windows.old, Tor browser, Internet Explorer, Google, Opera, Opera Software, Mozilla, Mozilla Firefox, $Recycle.Bin, ProgramData, All Users, autorun.inf, boot.ini, bootfont.bin, bootsect.bak, bootmgr, bootmgr.efi, bootmgfw.efi, desktop.ini, iconcache.db, ntldr, ntuser.dat, ntuser.dat.log
, ntuser.ini, thumbs.db, .sys, .dll, .lnk, .msi, .drv, .exe
```

To open a file the function `CreateFileW` is called. The fifth parameter `dwCreationDisposition = 3` which stands for `OPEN_EXISTING`.

![image-20200601203144499]({{site.url}}/assets/img/image-20200601203144499.png)


Then the function `ReadFile` is called to read a number of bytes of the file, that gets written to `buffer`. This buffer is then encrypted by the function at address `00402380` renamed to `encrypt_buffer`. The ransomware uses an encryption algorithm based on Salsa20 stream cipher.

![image-20200601211421004]({{site.url}}/assets/img/image-20200601211421004.png)

**Buffer before encryption** - the buffer contains the local file "C:\Program Files\die_win32_portable\base\db\Binary\bzip.1.sg" (not associated with the malware)

![image-20200601204806960]({{site.url}}/assets/img/image-20200601204806960.png)

**Buffer after encryption**

![image-20200601205054698]({{site.url}}/assets/img/image-20200601205054698.png)

After written the encrypted buffer to the file, the used key, nonce and the string `_RAGNAR_` are written to the end of the file.

![image-20200601210824215]({{site.url}}/assets/img/image-20200601210824215.png)

At the end, the ransomware opens the ransomware note with Notepad.

![image-20200531203519707]({{site.url}}/assets/img/image-20200531203519707.png)

![image-20200529212911106]({{site.url}}/assets/img/image-20200529212911106.png)

**IOC**

SHA256 - EC35C76AD2C8192F09C02ECA1F263B406163470CA8438D054DB7ADCF5BFC0597

### References

[https://www.bleepingcomputer.com/news/security/ransomware-encrypts-from-virtual-machines-to-evade-antivirus/](https://www.bleepingcomputer.com/news/security/ransomware-encrypts-from-virtual-machines-to-evade-antivirus/)

[https://news.sophos.com/en-us/2020/05/21/ragnar-locker-ransomware-deploys-virtual-machine-to-dodge-security/](https://news.sophos.com/en-us/2020/05/21/ragnar-locker-ransomware-deploys-virtual-machine-to-dodge-security/)

[https://malware.wikia.org/wiki/Ragnar_Locker](https://malware.wikia.org/wiki/Ragnar_Locker)
