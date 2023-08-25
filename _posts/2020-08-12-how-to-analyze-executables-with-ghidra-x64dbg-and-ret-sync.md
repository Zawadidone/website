---
name: How to analyze executables with Ghidra, x64dbg and ret-sync
date: 2020-08-12
header: 
  teaser: /assets/img/Screenshot 2020-08-12 at 16.50.43.png
---

In this blog post I would like to explain how to analyze files using Ghidra and x64dbg while using the [ret-sync](https://github.com/bootleg/ret-sync) plugin. While analyzing malware it is important to no lose track of the instructions you're analyzing. I like to do static analysis and dynamic analysis at the same time. This way I have Ghidra on the left of my screen and x64dbg on the right. In Ghidra I rename variables and functions an determine what the executable does. And to check my static analysis I let x64dbg execute the instructions and functions anaylyzed in Ghidra. For me [ret-sync](https://github.com/bootleg/ret-sync) is the perfect plugin that keeps track of where I am in both programs.

### Installation

Required software packages
* [Ghidra](https://ghidra-sre.org/InstallationGuide.html)
* [x64dbg](https://github.com/x64dbg/x64dbg/releases)
	* I also use [xAnalyzer](https://github.com/ThunderCls/xAnalyzer), but this is not required

**Ghidra**

```bash
git clone https://github.com/bootleg/ret-sync
cp ext_ghidra/dist/ghidra_9.1.2_PUBLIC_20200428_retsync.zip ~/ghidra_scripts/
``` 

1. From Ghidra projects manager: ``File`` -> ``Install Extensions...``, click on the
   `+` sign and select the `~/ghidra_scripts/ghidra_9.1.2_PUBLIC_20200428_retsync.zip` and click OK.
  
2. Restart Ghidra.
3. Configure the plugin.
4. Open an executable in the Ghidra CodeBrowser. The console should show this:
	```bash
[*] retsync init
[>] programOpened: test.exe
    imageBase: 0x400000
	```
5. Enable the synchronization `Alt+s`. The console should show this:
	```bash
[>] ret-sync enable
[>] server listening 
[>] server started
	```

**x64dbg**

1. Download the x64dbg plugins from the latest [Azure pipeline](https://dev.azure.com/bootlegdev/ret-sync-release/_build/results?buildId=101&view=artifacts&type=publishedArtifacts) (`ret-sync-release-x64dbg-*`). 
2. For both 32-bit and 64-bit unzip the file and copy the DLL `ext64dbg/Release/x64dbg_sync.dp3*` to the x64dbg program folder `C:\Program Files (x86)\x64dbg\x*\plugins`. `*` stands for 32 or 63.
5. Start x64dbg and open the same executable opened in Ghidra.
6. Click on Plugins->SyncPlugin->Enable sync
7. If enabled the console should show this:	 `[sync] sync is now enabled with host 127.0.0.1`.

Now both programs have the ret-sync plugins installed. Based on the documentation this can also work over a network, but I didn't get it working on Mac OS with VMware Fusion.

### Analysis
The instruction that will be executed is marked with yellow. This way you can rename variables and functions in Ghidra and execute instructions using x64dbg. The executable is a hello world example.

**x64dbg**
![image-20200805174024690]({{site.url}}/assets/img/Screenshot 2020-08-12 at 16.51.08.png)

<br><br>
**Ghidra Decompile window**
![image-20200805174024690]({{site.url}}/assets/img/Screenshot 2020-08-12 at 16.49.36.png)

<br><br>
**Ghidra Listing**
![image-20200805174024690]({{site.url}}/assets/img/Screenshot 2020-08-12 at 16.50.43.png)
