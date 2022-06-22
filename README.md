# EAC-Kernel-Packet-Fucker
Not my code. Only for saving
https://www.unknowncheats.me/forum/anti-cheat-bypass/503052-easy-anti-cheat-kernel-packet-fucker.html


This is the Easy Anti-Cheat Kernel Packet Fucker (for short, EACKPFucker).
What is this? Basically, packets via their kernel mode driver are not going to be sent to them, which means your pasta pasta 2023 kdmapper FUD bypasses can be used without any trouble.

Okay, you got me out of my pants. How the fuck does this work?
By simply changing one address. Now, let's dive deep into how EAC actually works.

From the beginning, Easy Anti-Cheat has to actually get your data in order to ban you. These packets are sent over their Hydra channel and are cryptographically secure. That is all you need to know for this bypass, I will not go into more detail about this.

Let's take a look at how this works inside their kernel driver, with a random violation:

![image](https://user-images.githubusercontent.com/13917777/173832205-99154956-9186-4edd-a9a0-654384856299.png)

Doesn't this look vulnerable to you? Because it sure does to me.
Let's take a look at our first function: kalloc_rt
![image](https://user-images.githubusercontent.com/13917777/173832254-ecdf64c9-2524-4248-9810-305dba0573a3.png)
Hmm, okay. Let's jump into alloc_pool_with_tag
![image](https://user-images.githubusercontent.com/13917777/173832337-8fb57475-8e45-478b-8390-f31d1b51868e.png)
It dynamically imports ExAllocatePoolWithTag. Hmmmm... I wonder what would happen if someone were to modify that qword to their modified malloc function... (yeah, it works -- and since you're modifying a writable section, EAC is none the wiser)

Okay, now we have control over memory allocation. Cool! What can we do with this?

I'm glad you asked! Here's the thing: All packets from kernel mode are the size of 33096i64.. aaand previously, we saw that if the memory doesn't get allocated, EAC just.. ignores the violation.

Okay, say someone was to simply.. do this:

![image](https://user-images.githubusercontent.com/13917777/173832960-7e26cb65-5752-4a6c-9e04-a31075f9382d.png)

![image](https://user-images.githubusercontent.com/13917777/173832390-4483329e-0a5a-45a6-90d7-49b203f4677b.png)

![image](https://user-images.githubusercontent.com/13917777/173832413-ec97bc29-0dc2-40dd-b3e6-c4d9f3480bdd.png)

---------------------------------------------------------------------------------------------------------------------------------------
```C++
// report encryption looks like this (some parts may vary for each report, i believe they use key1, key2, key3 to use only 1 function for decryption)
static report_t* encrypt(uint8_t* data, uint64_t size) {
 
	report_t* packet = (report_t*)malloc(sizeof(report_t));
	if (!packet) return nullptr;
 
	uint32_t seed = 0x80BE5ED5 * ((uint64_t)&data >> 2);
 
	memset(&packet->key1, 0, 0x8200);
	packet->raw_size = 0;
	packet->key1 = 0x66259F86; // gives key4?
	packet->key2 = 0x21EBA81; // gives key5?
	packet->key3 = 0xACE987AF; // gives key6?
	packet->key4 = 0x50BFC583; // gives seed
	packet->key5 = 0x3C61A927; // gives dynamic_key (from end)
	packet->key6 = 0x70881859; // gives actual payload size
 
	uint8_t* raw_data = packet->raw;
	uint64_t raw_size = 24;
	uint8_t* payload_data = packet->payload;
	uint64_t payload_size = 0;
 
	uint32_t dynamic_key = seed ^ 0x6957FDB6;
	while (payload_size < size && payload_size < 0x8000) {
 
		uint32_t a = (dynamic_key << 0xD) ^ dynamic_key;
		uint32_t b = (a >> 0x11) ^ a;
		uint32_t c = (b << 0x5) ^ b;
		uint32_t d = _rotr(c, 2);
 
		uint8_t shift = 8 * (payload_size & 3);
		payload_data[payload_size] = data[payload_size] ^ (d >> shift);
		dynamic_key = data[payload_size] ^ d;
 
		payload_size++;
		raw_size++;
	}
 
	uint64_t aligned_size = (raw_size + 0xFF) & ~0xFF; // align up by 0x100
	while (raw_size < aligned_size) {
		dynamic_key *= 0x80BE5ED5;
		raw_data[raw_size++] = dynamic_key;
	}
 
	packet->key4 ^= seed;
	packet->key5 ^= dynamic_key;
	packet->key6 ^= payload_size;
	packet->raw_size = raw_size;
	return packet;
}

// thus my decryption looks like this
static void decrypt(report_t* packet) {
 
	uint32_t seed = packet->key4 ^ 0x50BFC583;
	uint32_t dynamic_key = seed ^ 0x6957FDB6;
	//uint32_t dynamic_key = packet->key5 ^ 0x3C61A927;
 
	uint8_t* payload_data = packet->payload;
	uint32_t payload_size = packet->key6 ^ 0x70881859;
 
 
	for (uint32_t i = 0; i < payload_size; i++) {
		uint32_t a = (dynamic_key << 0xD) ^ dynamic_key;
		uint32_t b = (a >> 0x11) ^ a;
		uint32_t c = (b << 0x5) ^ b;
		uint32_t d = _rotr(c, 2);
 
		uint8_t shift = 8 * (i & 3);
		payload_data[i] ^= (d >> shift);
		dynamic_key = payload_data[i] ^ d;
	}
}

```
after dumping some reports and decrypting:

REPORT ID: 0x107531A1 (text)
```
[Ob] N: MsMpEng.exe T: E37B095A C: 000280DE E: 0000BE90
[Lib] N: ntdll.dll B: 00007FFD4D310000 S: 1F7000 P: ntdll.pdb
[Lib] N: kernel32.dll B: 00007FFD4CFF0000 S: BD000 P: kernel32.pdb
[Lib] N: KernelBase.dll B: 00007FFD4AF90000 S: 2CD000 P: kernelbase.pdb
[Lib] N: advapi32.dll B: 00007FFD4D220000 S: AE000 P: advapi32.pdb
[Ob] N: ProcessHacker. T: 86B67A1B C: 002A2F41 E: 0019D960
[Ob] N: Discord.exe T: 6255D9BD C: 0708B371 E: 030AAD10
[Lib] N: msvcrt.dll B: 00007FFD4CAE0000 S: 9E000 P: msvcrt.pdb
[Lib] N: X3DAudio1_7.dll B: 00000000620E0000 S: 9000 P: X3DAudio1_7.pdb
[Lib] N: xinput1_3.dll B: 0000000000400000 S: 1E000 P: XInput1_3.pdb
[Lib] N: XAPOFX1_5.dll B: 00007FFD2FC30000 S: 15000 P: XAPOFX1_5.pdb
[Ob] N: EasyAntiCheat. T: 6273A229 C: 0011DA03 E: 0006D465
[Lib] N: msvcrt.dll B: 000002AAA3540000 S: 9E000 P: msvcrt.pdb
[Lib] N: msvcrt.dll B: 000002AAA3730000 S: 9E000 P: msvcrt.pdb
[Lib] N: setupapi.dll B: 00007FFD4CB80000 S: 46F000 P: setupapi.pdb
[Lib] N: dxgi.dll B: 00007FFD49330000 S: F3000 P: dxgi.pdb
[Lib] N: sechost.dll B: 00007FFD4B3A0000 S: 9C000 P: sechost.pdb
[Lib] N: ole32.dll B: 00007FFD4C950000 S: 12A000 P: ole32.pdb
[Lib] N: rpcrt4.dll B: 00007FFD4BA00000 S: 125000 P: rpcrt4.pdb
[Lib] N: ucrtbase.dll B: 00007FFD4AAD0000 S: 100000 P: ucrtbase.pdb
[Lib] N: win32u.dll B: 00007FFD4ABD0000 S: 22000 P: win32u.pdb
[Lib] N: user32.dll B: 00007FFD4BB30000 S: 1A0000 P: user32.pdb
[Lib] N: d3d9.dll B: 00007FFD38750000 S: 1CF000 P: d3d9.pdb
[Lib] N: combase.dll B: 00007FFD4B440000 S: 354000 P: combase.pdb
[Lib] N: cfgmgr32.dll B: 00007FFD4B260000 S: 4E000 P: cfgmgr32.pdb
[Lib] N: bcrypt.dll B: 00007FFD4B340000 S: 27000 P: bcrypt.pdb
[Lib] N: gdi32.dll B: 00007FFD4BFD0000 S: 2A000 P: gdi32.pdb
[Lib] N: gdi32.dll B: 000002AAA3570000 S: 2A000 P: gdi32.pdb
[Lib] N: d3d11.dll B: 00007FFD47110000 S: 263000 P: d3d11.pdb
[Lib] N: combase.dll B: 000002AAA39A0000 S: 354000 P: combase.pdb
[Lib] N: gdi32full.dll B: 00007FFD4AE80000 S: 10B000 P: gdi32full.pdb
[Lib] N: D3DCompiler_43.dll B: 00007FFD187E0000 S: 26F000 P: D3DCompiler_43.pdb
[Lib] N: msvcp_win.dll B: 00007FFD4AA30000 S: 9D000 P: msvcp_win.pdb
[Lib] N: kernel.appcore.dll B: 00007FFD49310000 S: 12000 P: Kernel.Appcore.pdb
[Lib] N: windows.storage.dll B: 00007FFD48B10000 S: 794000 P: Windows.Storage.pdb
[Lib] N: dwmapi.dll B: 00007FFD48200000 S: 2F000 P: dwmapi.pdb
[Lib] N: ws2_32.dll B: 00007FFD4C880000 S: 6B000 P: ws2_32.pdb
[Lib] N: crypt32.dll B: 00007FFD4AD20000 S: 156000 P: crypt32.pdb
[Lib] N: Wldap32.dll B: 00007FFD4CA80000 S: 56000 P: wldap32.pdb
[Lib] N: normaliz.dll B: 00007FFD4BCD0000 S: 8000 P: normaliz.pdb
[Lib] N: winmm.dll B: 00007FFD39FD0000 S: 27000 P: winmm.pdb
[Lib] N: shell32.dll B: 00007FFD4C120000 S: 744000 P: shell32.pdb
[Lib] N: wldp.dll B: 00007FFD4A490000 S: 2C000 P: WLDP.pdb
[Lib] N: oleaut32.dll B: 00007FFD4B930000 S: CD000 P: oleaut32.pdb
[Lib] N: IPHLPAPI.DLL B: 00007FFD49E90000 S: 3B000 P: iphlpapi.pdb
[Lib] N: imm32.dll B: 00007FFD4B370000 S: 30000 P: imm32.pdb
[Lib] N: UIAutomationCore.dll B: 00007FFD22240000 S: 2F5000 P: UIAutomationCore.pdb
[Lib] N: winhttp.dll B: 00007FFD44920000 S: 10A000 P: winhttp.pdb
[Lib] N: dsound.dll B: 00007FFD10F40000 S: 9C000 P: dsound.pdb
[Lib] N: vcruntime140.dll B: 00007FFD3F380000 S: 1B000 P: D:\a\_work\1\s\\binaries\amd64ret\bin\amd64\\vcruntime140.amd64.pdb
[Lib] N: msvcp140.dll B: 00007FFD3D5E0000 S: 8E000 P: D:\a\_work\1\s\\binaries\amd64ret\bin\amd64\\msvcp140.amd64.pdb
[Lib] N: vcruntime140_1.dll B: 00007FFD3F370000 S: C000 P: D:\a\_work\1\s\\binaries\amd64ret\bin\amd64\\vcruntime140_1.amd64.pdb
[Lib] N: powrprof.dll B: 00007FFD49FB0000 S: 4B000 P: powrprof.pdb
[Lib] N: powrprof.dll B: 000002AAA3570000 S: 4B000 P: powrprof.pdb
[Lib] N: winmmbase.dll B: 00007FFD3F210000 S: 26000 P: WINMMBASE.pdb
[Lib] N: propsys.dll B: 00007FFD46280000 S: F6000 P: propsys.pdb
[Lib] N: umpdc.dll B: 00007FFD49E70000 S: 12000 P: UMPDC.pdb
[Lib] N: nvapi64.dll B: 00007FFD40030000 S: 747000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\nvapi\gpu\_out\wddm2_amd64_release\nvapi64.pdb
[Lib] N: version.dll B: 00007FFD433F0000 S: A000 P: version.pdb
[Lib] N: shlwapi.dll B: 00007FFD4BCE0000 S: 55000 P: shlwapi.pdb
[Lib] N: msasn1.dll B: 00007FFD4A620000 S: 12000 P: msasn1.pdb
[Lib] N: cryptnet.dll B: 00007FFD424B0000 S: 31000 P: cryptnet.pdb
[Lib] N: drvstore.dll B: 00007FFD41800000 S: 148000 P: drvstore.pdb
[Lib] N: devobj.dll B: 00007FFD4A7E0000 S: 2C000 P: devobj.pdb
[Lib] N: cryptbase.dll B: 00007FFD4A400000 S: C000 P: cryptbase.pdb
[Lib] N: bcryptprimitives.dll B: 00007FFD4B2B0000 S: 82000 P: bcryptprimitives.pdb
[Lib] N: SHCore.dll B: 00007FFD4BF20000 S: AD000 P: shcore.pdb
[Lib] N: profapi.dll B: 00007FFD4A970000 S: 1F000 P: profapi.pdb
[Ob] N: explorer.exe T: F7B2A2B2 C: 004E72AA E: 000A1920
[Ob] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: sspicli.dll B: 00007FFD4A920000 S: 32000 P: sspicli.pdb
[Lib] N: dbghelp.dll B: 00007FFCFB450000 S: 1E3000 P: dbghelp.pdb
[Lib] N: nsi.dll B: 00007FFD4BD40000 S: 8000 P: nsi.pdb
[Lib] N: dhcpcsvc6.dll B: 00007FFD41620000 S: 17000 P: dhcpcsvc6.pdb
[Lib] N: dhcpcsvc.dll B: 00007FFD424F0000 S: 1D000 P: dhcpcsvc.pdb
[Lib] N: mswsock.dll B: 00007FFD4A1F0000 S: 6A000 P: mswsock.pdb
[Lib] N: wintrust.dll B: 00007FFD4ACB0000 S: 67000 P: wintrust.pdb
[Lib] N: secur32.dll B: 00007FFD3F6C0000 S: C000 P: secur32.pdb
[Lib] N: version.dll B: 00007FFD433F0000 S: A000 P: version.pdb
[Lib] N: cryptsp.dll B: 00007FFD4A3E0000 S: 18000 P: cryptsp.pdb
[Lib] N: rsaenh.dll B: 00007FFD49B00000 S: 34000 P: rsaenh.pdb
[Lib] N: imagehlp.dll B: 00007FFD4BF00000 S: 1D000 P: imagehlp.pdb
[Lib] N: gpapi.dll B: 00007FFD492C0000 S: 23000 P: gpapi.pdb
[Lib] N: opengl32.dll B: 00007FFD18A50000 S: 126000 P: opengl32.pdb
[Lib] N: glu32.dll B: 00007FFD26CA0000 S: 2C000 P: glu32.pdb
[Lib] N: ResourcePolicyClient.dll B: 00007FFD48100000 S: 14000 P: ResourcePolicyClient.pdb
[Ob] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: msctf.dll B: 00007FFD4C000000 S: 115000 P: msctf.pdb
[Lib] N: nvldumdx.dll B: 00007FFD38C80000 S: 117000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\nvldumd\_out\wddm2_amd64_release\nvldumdx.pdb
[Lib] N: nvwgf2umx.dll B: 00007FFD13260000 S: 29FB000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\wgf2um\_out\wddm2_amd64_release\nvwgf2umx.pdb
[Lib] N: NvCameraAllowlisting64.dll B: 00007FFCFB2F0000 S: AE000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCameraAllowlisting64.pdb
[Lib] N: nvspcap64.dll B: 00007FFD10940000 S: 2C7000 P: C:\dvs\p4\build\sw\rel\gfclient\rel_03_25_1\shadowplay2\proxy\win7_amd64_release\nvspcap64.pdb
[Lib] N: ntmarta.dll B: 00007FFD49730000 S: 33000 P: ntmarta.pdb
[Lib] N: NvCamera64.dll B: 00007FFCFAB50000 S: 84F000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCamera64.pdb
[Lib] N: dinput8.dll B: 00007FFD2FB90000 S: 45000 P: dinput8.pdb
[Lib] N: XInput9_1_0.dll B: 00007FFD2FBE0000 S: 7000 P: XInput9_1_0.pdb
[Lib] N: WindowsCodecs.dll B: 00007FFD44AA0000 S: 1B4000 P: WindowsCodecs.pdb
[Lib] N: hid.dll B: 00007FFD492B0000 S: D000 P: hid.pdb
[Lib] N: InputHost.dll B: 00007FFD2C350000 S: 152000 P: InputHost.pdb
[Lib] N: CoreUIComponents.dll B: 00007FFD478F0000 S: 35E000 P: CoreUIComponents.pdb
[Lib] N: CoreMessaging.dll B: 00007FFD47C50000 S: F2000 P: CoreMessaging.pdb
[Lib] N: WinTypes.dll B: 00007FFD46590000 S: 154000 P: WinTypes.pdb
[Lib] N: DXCore.dll B: 00007FFD44CF0000 S: 3B000 P: DXCore.pdb
[Lib] N: d3dcompiler_47_64.dll B: 00007FFCFA740000 S: 404000 P: D3DCompiler_47.pdb
[Lib] N: nvapi64.dll B: 00007FFD40030000 S: 747000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\nvapi\gpu\_out\wddm2_amd64_release\nvapi64.pdb
[Lib] N: nvcuda64.dll B: 00007FFCF8920000 S: 1515000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\gpgpu\_out\wddm2_amd64_release\nvcuda.pdb
[Ob] N: nvcontainer.ex T: 621DBDA6 C: 000FF234 E: 00036BD0
[Lib] N: NapiNSP.dll B: 00007FFD23680000 S: 17000 P: NapiNSP.pdb
[Lib] N: pnrpnsp.dll B: 00007FFD22CE0000 S: 1B000 P: pnrpnsp.pdb
[Lib] N: wshbth.dll B: 00007FFD28020000 S: 15000 P: wshbth.pdb
[Lib] N: nlaapi.dll B: 00007FFD45B40000 S: 1D000 P: nlaapi.pdb
[Lib] N: dnsapi.dll B: 00007FFD49EE0000 S: CB000 P: dnsapi.pdb
[Lib] N: winrnr.dll B: 00007FFD22CC0000 S: 12000 P: winrnr.pdb
[Lib] N: uxtheme.dll B: 00007FFD48010000 S: 9E000 P: UxTheme.pdb
[Ob] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: clbcatq.dll B: 00007FFD4B880000 S: A9000 P: CLBCatQ.pdb
[Lib] N: gameux.dll B: 00007FFD2FBE0000 S: B000 P: gameux.pdb
[Lib] N: PxFoundation_x64.dll B: 00007FFD2FB70000 S: 13000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\PxFoundation_x64.pdb
[Lib] N: PhysX3Common_x64.dll B: 00007FFCFB0E0000 S: 18C000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\PhysX3Common_x64.pdb
[Lib] N: PxPvdSDK_x64.dll B: 00007FFCFB0A0000 S: 32000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\PxPvdSDK_x64.pdb
[Lib] N: PhysX3_x64.dll B: 00007FFCFAE90000 S: 20B000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\PhysX3_x64.pdb
[Lib] N: ApexFramework_x64.dll B: 00007FFCFAD80000 S: 108000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\ApexFramework_x64.pdb
[Lib] N: APEX_Legacy_x64.dll B: 00007FFCFAA10000 S: 367000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\APEX_Legacy_x64.pdb
[Lib] N: APEX_Clothing_x64.dll B: 00007FFCFA8D0000 S: 137000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\APEX_Clothing_x64.pdb
[Lib] N: PhysX3Cooking_x64.dll B: 00007FFCFA890000 S: 40000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\PhysX3Cooking_x64.pdb
[Lib] N: PhysXUpdateLoader64.dll B: 00007FFCFA860000 S: 2A000 P: N/A
[Lib] N: PhysXUpdateLoader64.dll B: 00007FFCFA860000 S: 2A000 P: N/A
[Ob] N: ctfmon.exe T: 60C3FE88 C: 000118E7 E: 000011C0
[Lib] N: TextInputFramework.dll B: 00007FFD2C7E0000 S: F9000 P: TextInputFramework.pdb
[Lib] N: ExplorerFrame.dll B: 00007FFD27170000 S: 220000 P: ExplorerFrame.pdb
[Lib] N: nvldumdx.dll B: 00007FFD38C80000 S: 117000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\nvldumd\_out\wddm2_amd64_release\nvldumdx.pdb
[Lib] N: nvwgf2umx.dll B: 00007FFD13260000 S: 29FB000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\wgf2um\_out\wddm2_amd64_release\nvwgf2umx.pdb
[Lib] N: NvCameraAllowlisting64.dll B: 00007FFCFA7E0000 S: AE000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCameraAllowlisting64.pdb
[Lib] N: NvCamera64.dll B: 00007FFCF2F20000 S: 84F000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCamera64.pdb
[Lib] N: XInput9_1_0.dll B: 00007FFD2FBE0000 S: 7000 P: XInput9_1_0.pdb
[Lib] N: WindowsCodecs.dll B: 00007FFD44AA0000 S: 1B4000 P: WindowsCodecs.pdb
[Lib] N: hid.dll B: 00007FFD492B0000 S: D000 P: hid.pdb
[Lib] N: d3dcompiler_47_64.dll B: 00007FFCFA480000 S: 404000 P: D3DCompiler_47.pdb
[Lib] N: d3d10warp.dll B: 00007FFD39400000 S: 6F6000 P: d3d10warp.pdb
[Proc] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: nvldumdx.dll B: 00007FFD38C80000 S: 117000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\nvldumd\_out\wddm2_amd64_release\nvldumdx.pdb
[Lib] N: nvwgf2umx.dll B: 00007FFD13260000 S: 29FB000 P: C:\dvs\p4\build\sw\rel\gpu_drv\r515\r516_10\drivers\wgf2um\_out\wddm2_amd64_release\nvwgf2umx.pdb
[Lib] N: NvCameraAllowlisting64.dll B: 00007FFCFA780000 S: AE000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCameraAllowlisting64.pdb
[Lib] N: NvCamera64.dll B: 00007FFCF2F20000 S: 84F000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCamera64.pdb
[Lib] N: WindowsCodecs.dll B: 00007FFD44AA0000 S: 1B4000 P: WindowsCodecs.pdb
[Lib] N: hid.dll B: 00007FFD492B0000 S: D000 P: hid.pdb
[Lib] N: XInput9_1_0.dll B: 00007FFD2FBE0000 S: 7000 P: XInput9_1_0.pdb
[Lib] N: d3dcompiler_47_64.dll B: 00007FFCFA420000 S: 404000 P: D3DCompiler_47.pdb
[Ob] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: DataExchange.dll B: 00007FFD27390000 S: 3E000 P: DataExchange.pdb
[Lib] N: dcomp.dll B: 00007FFD47380000 S: 1E4000 P: dcomp.pdb
[Lib] N: twinapi.appcore.dll B: 00007FFD44D30000 S: 200000 P: twinapi.appcore.pdb
[Ob] N: GameBarFTServe T: 6274C5DE C: 00000000 E: 0005AA40
[Lib] N: NvCameraAllowlisting64.dll B: 00007FFCFA370000 S: AE000 P: C:\BuildAgent\work\e4cd6e8028b37277\bin\Release\NvCameraAllowlisting64.pdb
[Ob] N: EpicGamesLaunc T: 629FB8BA C: 01F2B27C E: 017C9C84
[Ob] N: WmiPrvSE.exe T: 5DA7AB91 C: 000860A3 E: 00012580
[Lib] N: apphelp.dll B: 00007FFD47F60000 S: 90000 P: apphelp.pdb
[Lib] N: MessageBus.dll B: 00007FFD3A000000 S: 737000 P: C:\dvs\p4\build\sw\gcomp\dev\src\NvContainer\_out\x86_64\release\bus\MessageBus.pdb
[Lib] N: NvCloth_x64.dll B: 00007FFCEC1C0000 S: 47000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\NvCloth_x64.pdb
[Lib] N: mf.dll B: 00007FFD10DB0000 S: 84000 P: mf.pdb
[Lib] N: mfplat.dll B: 00007FFD20800000 S: 1BC000 P: MFPLAT.pdb
[Lib] N: RTWorkQ.dll B: 00007FFD207C0000 S: 34000 P: rtworkq.pdb
[Lib] N: MFPlay.dll B: 00007FFCEC130000 S: 8C000 P: MFPlay.pdb
[Lib] N: mfcore.dll B: 00007FFCEBCA0000 S: 48E000 P: mfcore.pdb
[Lib] N: ksuser.dll B: 00007FFD2FB60000 S: 9000 P: ksuser.pdb
[Lib] N: mfmp4srcsnk.dll B: 00007FFCEBA90000 S: 204000 P: mfmp4srcsnk.pdb
[Lib] N: MMDevAPI.dll B: 00007FFD43BF0000 S: 85000 P: MMDevAPI.pdb
[Lib] N: AudioSes.dll B: 00007FFD43F70000 S: 182000 P: audioses.pdb
[Lib] N: APEX_Destructible_x64.dll B: 00007FFCEB890000 S: 1FF000 P: D:\Build\++Fortnite\Sync\Engine\Binaries\ThirdParty\PhysX3\Win64\VS2015\APEX_Destructible_x64.pdb
[Lib] N: ResourcePolicyClient.dll B: 00007FFD48100000 S: 14000 P: ResourcePolicyClient.pdb
[Lib] N: PhysXUpdateLoader64.dll B: 00007FFCFA310000 S: 2A000 P: N/A
[Ob] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: CompPkgSup.dll B: 00007FFD3E9B0000 S: 37000 P: CompPkgSup.pdb
[Lib] N: MSAudDecMFT.dll B: 00007FFCEB820000 S: 6F000 P: MSAudDecMFT.pdb
[Proc] N: svchost.exe T: 1F37EB46 C: 0001A0FA E: 00005040
[Lib] N: mfperfhelper.dll B: 00007FFD42640000 S: 12D000 P: mfperfhelper.pdb
[Lib] N: Windows.Media.dll B: 00007FFD127B0000 S: 727000 P: Windows.Media.pdb
[Lib] N: RESAMPLEDMO.DLL B: 00007FFCEB7E0000 S: 3F000 P: RESAMPLEDMO.pdb
[Lib] N: msdmo.dll B: 00007FFD27140000 S: B000 P: msdmo.pdb
[Lib] N: AkSilenceGenerator.dll B: 00007FFCFA320000 S: 1D000 P: D:\Jenkins\ws\wwise_v2021.1\Wwise\SDK\x64_vc160\Release\bin\AkSilenceGenerator.pdb
[Lib] N: Windows.ApplicationModel.dll B: 00007FFD36210000 S: E6000 P: Windows.ApplicationModel.pdb
[Lib] N: COLORCNV.DLL B: 00007FFCEB7A0000 S: 3C000 P: COLORCNV.pdb
[Lib] N: AkParametricEQ.dll B: 00007FFCEB770000 S: 25000 P: D:\Jenkins\ws\wwise_v2021.1\Wwise\SDK\x64_vc160\Release\bin\AkParametricEQ.pdb
[Lib] N: AppXDeploymentClient.dll B: 00007FFD42A10000 S: F7000 P: AppXDeploymentClient.pdb
[Lib] N: msmpeg2vdec.dll B: 00007FFD07CF0000 S: 27A000 P: msmpeg2vdec.pdb
[Lib] N: AkDelay.dll B: 00007FFCEB750000 S: 20000 P: D:\Jenkins\ws\wwise_v2021.1\Wwise\SDK\x64_vc160\Release\bin\AkDelay.pdb
[Ob] N: audiodg.exe T: C40FE88F C: 0009A59F E: 0001CFD0
[Lib] N: Windows.UI.dll B: 00007FFD2C8E0000 S: 141000 P: Windows.UI.pdb
[Lib] N: WindowManagementAPI.dll B: 00007FFD45160000 S: A1000 P: WindowManagementAPI.pdb
[Lib] N: AkCompressor.dll B: 00007FFCEB730000 S: 20000 P: D:\Jenkins\ws\wwise_v2021.1\Wwise\SDK\x64_vc160\Release\bin\AkCompressor.pdb
[Lib] N: avrt.dll B: 00007FFD45990000 S: A000 P: avrt.pdb
[Lib] N: AkPeakLimiter.dll B: 00007FFD19840000 S: 20000 P: D:\Jenkins\ws\wwise_v2021.1\Wwise\SDK\x64_vc160\Release\bin\AkPeakLimiter.pdb
[Lib] N: AkRoomVerb.dll B: 00007FFCEB6F0000 S: 3B000 P: D:\Jenkins\ws\wwise_v2021.1\Wwise\SDK\x64_vc160\Release\bin\AkRoomVerb.pdb
```
REPORT ID: 0x107531A1 (raw bytes)
```
0 FF FF FF FF 0 0
```
REPORT ID: 0x6829432C (raw bytes, cpuid_1 + HvTimingAttack)
```
struct CPU_INFO_REPORT {
uint32_t cpuid_1_eax; // 0x0
uint32_t cpuid_1_ebx; // 0x4
uint32_t cpuid_1_ecx; // 0x8
uint32_t cpuid_1_edx; // 0xC
uint64_t quotient; // 0x10 // cpuid_iet / nop_iet (if > 15 = hv present)
uint64_t cpuid_iet; // 0x18 // 100 iterations
uint64_t nop_iet; // 0x20 // 100 iterations
};

TEXTUAL REPRESENTATION: EAX[A50F00] EBX[40C0800] ECX[7ED8320B] EDX[178BFBFF] QUOTIENT[4] CPUID_IET[397E] NOP_IET[E2E]
```
REPORT ID: 0x2511F5D9 (text)
![图片](https://user-images.githubusercontent.com/13917777/174939607-94a4aa89-2795-4dfc-b970-aa0dbbfbaadf.png)

REPORT ID: 0x55AD0F18 (text)
```
\SystemRoot\System32\drivers\CLASSPNP.SYS
```
i redacted some of the reports, because they store confidential data
here's all reports sent while game is open
```
14.46319485 [!] CreateThread: 0
14.46319675 [!] Entering waiting loop...
26.55835915 [!] Found!
27.44778633 [!] <ALLOC> (BUFFER: FFFFB5036AC7D000) (CALLER: 00000000007372D5) (PV: 0) (START: FFFFF807550BF9E0)
27.45028877 [!] <FREE> (BUFFER: FFFFB5036AC7D000) (IDX: 0) (IQRL: 0)
27.46345901 [!] <ALLOC> (BUFFER: FFFFB5036AC7D000) (CALLER: 0000000000101514) (PV: 0) (START: FFFFF8075559CFA0)
27.49540138 [!] <ALLOC> (BUFFER: FFFFB5036AD34000) (CALLER: 0000000000B0DC9B) (PV: 0) (START: FFFFF807550BF9E0)
27.49777222 [!] <FREE> (BUFFER: FFFFB5036AD34000) (IDX: 1) (IQRL: 0)
27.49875259 [!] <ALLOC> (BUFFER: FFFFB5036AD34000) (CALLER: 0000000000B0DC9B) (PV: 0) (START: FFFFF807550BF9E0)
27.50095367 [!] <FREE> (BUFFER: FFFFB5036AD34000) (IDX: 2) (IQRL: 0)
27.51844215 [!] <ALLOC> (BUFFER: FFFFB5036AD0B000) (CALLER: 00000000005804C1) (PV: 0) (START: FFFFF807550BF9E0)
27.52266502 [!] <FREE> (BUFFER: FFFFB5036AD0B000) (IDX: 3) (IQRL: 0)
27.62048340 [!] <FREE> (BUFFER: FFFFB5036AC7D000) (IDX: 4) (IQRL: 0)
27.84068871 [!] <ALLOC> (BUFFER: FFFFB5036AC7D000) (CALLER: 0000000000A0DAB2) (PV: 0) (START: FFFFF8075559CFA0)
27.94095421 [!] <FREE> (BUFFER: FFFFB5036AC7D000) (IDX: 5) (IQRL: 0)
27.94399452 [!] <ALLOC> (BUFFER: FFFFB5036AC7D000) (CALLER: 0000000000040E9C) (PV: 0) (START: FFFFF8075559CFA0)
27.94753265 [!] <FREE> (BUFFER: FFFFB5036AC7D000) (IDX: 6) (IQRL: 0)
27.95534134 [!] <ALLOC> (BUFFER: FFFFB5036AC7D000) (CALLER: 00000000004E598A) (PV: 0) (START: FFFFF8075559CFA0)
27.95627785 [!] <FREE> (BUFFER: FFFFB5036AC7D000) (IDX: 7) (IQRL: 0)
27.96207619 [!] <ALLOC> (BUFFER: FFFFB5036AD21000) (CALLER: 00000000003B8C95) (PV: 0) (START: FFFFF807550BF9E0)
27.96610260 [!] <FREE> (BUFFER: FFFFB5036AD21000) (IDX: 8) (IQRL: 0)
28.33023071 [!] <ALLOC> (BUFFER: FFFFB5036AD3C000) (CALLER: 00000000001C9E77) (PV: 0) (START: FFFFF8075559CFA0)
28.33943367 [!] <FREE> (BUFFER: FFFFB5036AD3C000) (IDX: 9) (IQRL: 0)
28.46156883 [!] <ALLOC> (BUFFER: FFFFB5036AD3C000) (CALLER: 000000000070BF5C) (PV: 0) (START: FFFFF8075559CFA0)
28.46789360 [!] <ALLOC> (BUFFER: FFFFB5036AD9E000) (CALLER: 000000000005E48B) (PV: 0) (START: FFFFF807550BF9E0)
28.47018242 [!] <FREE> (BUFFER: FFFFB5036AD9E000) (IDX: 10) (IQRL: 0)
28.49411774 [!] <FREE> (BUFFER: FFFFB5036AD3C000) (IDX: 11) (IQRL: 0)
28.75633049 [!] <ALLOC> (BUFFER: FFFFB5036AD2C000) (CALLER: 00000000009EB074) (PV: 0) (START: FFFFF807550BF9E0)
28.75914001 [!] <FREE> (BUFFER: FFFFB5036AD2C000) (IDX: 12) (IQRL: 0)
61.45036697 [!] <ALLOC> (BUFFER: FFFFB5036B165000) (CALLER: 00000000009331AA) (PV: 0) (START: FFFFF80754E207C0)
61.48706055 [!] <FREE> (BUFFER: FFFFB5036B165000) (IDX: 13) (IQRL: 0)
67.27204132 [!] <ALLOC> (BUFFER: FFFFB5036B165000) (CALLER: 0000000000048520) (PV: 0) (START: FFFFF8075559CFA0)
67.29238129 [!] <FREE> (BUFFER: FFFFB5036B165000) (IDX: 14) (IQRL: 0)
67.29380798 [!] <ALLOC> (BUFFER: FFFFB5036B165000) (CALLER: 0000000000751132) (PV: 0) (START: FFFFF8075559CFA0)
67.29904175 [!] <FREE> (BUFFER: FFFFB5036B165000) (IDX: 15) (IQRL: 0)
68.36171722 [!] <ALLOC> (BUFFER: FFFFB5036B165000) (CALLER: 00000000007F6556) (PV: 0) (START: FFFFF8075559CFA0)
68.45364380 [!] <FREE> (BUFFER: FFFFB5036B165000) (IDX: 16) (IQRL: 0)
70.06682587 [!] <ALLOC> (BUFFER: FFFFB5036AF2A000) (CALLER: 00000000008E428A) (PV: 0) (START: FFFFF8075559CFA0)
70.07866669 [!] <FREE> (BUFFER: FFFFB5036AF2A000) (IDX: 17) (IQRL: 0)
70.91152191 [!] <ALLOC> (BUFFER: FFFFB5036AF2A000) (CALLER: 0000000000AA53C8) (PV: 0) (START: FFFFF8075559CFA0)
70.91969299 [!] <FREE> (BUFFER: FFFFB5036AF2A000) (IDX: 18) (IQRL: 0)
78.18015289 [!] <ALLOC> (BUFFER: FFFFB5036AF2A000) (CALLER: 00000000005C0923) (PV: 0) (START: FFFFF8075559CFA0)
78.28571320 [!] <FREE> (BUFFER: FFFFB5036AF2A000) (IDX: 19) (IQRL: 0)
79.56472778 [!] <ALLOC> (BUFFER: FFFFB5036AF2A000) (CALLER: 000000000042AF7C) (PV: 0) (START: FFFFF8075559CFA0)
79.62374878 [!] <FREE> (BUFFER: FFFFB5036AF2A000) (IDX: 20) (IQRL: 0)
79.63484192 [!] <ALLOC> (BUFFER: FFFFB5036AF2A000) (CALLER: 000000000005CB69) (PV: 0) (START: FFFFF8075559CFA0)
79.65575409 [!] <FREE> (BUFFER: FFFFB5036AF2A000) (IDX: 21) (IQRL: 0)
91.38448334 [!] <ALLOC> (BUFFER: FFFFB5036AF2A000) (CALLER: 00000000000529FE) (PV: 0) (START: FFFFF8075559CFA0)
91.40061188 [!] <FREE> (BUFFER: FFFFB5036AF2A000) (IDX: 22) (IQRL: 0)
327.62762451 [!] <ALLOC> (BUFFER: FFFFB5036B18E000) (CALLER: 0000000000101514) (PV: 0) (START: FFFFF8075559CFA0)
327.69091797 [!] <FREE> (BUFFER: FFFFB5036B18E000) (IDX: 23) (IQRL: 0)
635.37713623 [!] <ALLOC> (BUFFER: FFFFB5036C633000) (CALLER: 00000000006CD8EB) (PV: 0) (START: FFFFF807555D2B00)
635.45568848 [!] <FREE> (BUFFER: FFFFB5036C633000) (IDX: 24) (IQRL: 0)
660.46936035 [!] <ALLOC> (BUFFER: FFFFB5036C633000) (CALLER: 00000000001B48F7) (PV: 0) (START: FFFFF8075559CFA0)
660.47485352 [!] <FREE> (BUFFER: FFFFB5036C633000) (IDX: 25) (IQRL: 0)
660.73034668 [!] <ALLOC> (BUFFER: FFFFB5036C633000) (CALLER: 0000000000B3BE52) (PV: 0) (START: FFFFF8075559CFA0)
660.75280762 [!] <FREE> (BUFFER: FFFFB5036C633000) (IDX: 26) (IQRL: 0)
927.69372559 [!] <ALLOC> (BUFFER: FFFFB5036B2C7000) (CALLER: 0000000000101514) (PV: 0) (START: FFFFF8075559CFA0)
927.72619629 [!] <FREE> (BUFFER: FFFFB5036B2C7000) (IDX: 27) (IQRL: 0)
939.71856689 [!] <ALLOC> (BUFFER: FFFFB5036C633000) (CALLER: 000000000050E6AF) (PV: 0) (START: FFFFF8075559CFA0)
939.71948242 [!] <FREE> (BUFFER: FFFFB5036C633000) (IDX: 28) (IQRL: 0)
958.45611572 [!] <ALLOC> (BUFFER: FFFFB5036B2C7000) (CALLER: 000000000012AF7C) (PV: 0) (START: FFFFF8075559CFA0)
958.47155762 [!] <FREE> (BUFFER: FFFFB5036B2C7000) (IDX: 29) (IQRL: 1)
1060.63464355 [!] <ALLOC> (BUFFER: FFFFB5036C87F000) (CALLER: 0000000000A7CAD4) (PV: 0) (START: FFFFF8075559CFA0)
1060.64123535 [!] <FREE> (BUFFER: FFFFB5036C87F000) (IDX: 30) (IQRL: 0)
1064.00781250 [!] <ALLOC> (BUFFER: FFFFB5036C87F000) (CALLER: 0000000000A7CAD4) (PV: 0) (START: FFFFF8075559CFA0)
1064.01342773 [!] <FREE> (BUFFER: FFFFB5036C87F000) (IDX: 31) (IQRL: 0)
1741.98950195 [!] <ALLOC> (BUFFER: FFFFB5036CAA7000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
1742.00671387 [!] <FREE> (BUFFER: FFFFB5036CAA7000) (IDX: 32) (IQRL: 0)
1977.99316406 [!] <ALLOC> (BUFFER: FFFFB5036AAAB000) (CALLER: 0000000000A7CAD4) (PV: 0) (START: FFFFF8075559CFA0)
1977.99853516 [!] <FREE> (BUFFER: FFFFB5036AAAB000) (IDX: 33) (IQRL: 0)
2127.74389648 [!] <ALLOC> (BUFFER: FFFFB5036AAAB000) (CALLER: 0000000000101514) (PV: 0) (START: FFFFF8075559CFA0)
2127.76245117 [!] <FREE> (BUFFER: FFFFB5036AAAB000) (IDX: 34) (IQRL: 0)
2228.88964844 [!] <ALLOC> (BUFFER: FFFFB5036E7F2000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
2228.91699219 [!] <FREE> (BUFFER: FFFFB5036E7F2000) (IDX: 35) (IQRL: 0)
2230.83740234 [!] <ALLOC> (BUFFER: FFFFB5036EA29000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
2230.86010742 [!] <FREE> (BUFFER: FFFFB5036EA29000) (IDX: 36) (IQRL: 0)
2275.82666016 [!] <ALLOC> (BUFFER: FFFFB5036E9EE000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
2275.84399414 [!] <FREE> (BUFFER: FFFFB5036E9EE000) (IDX: 37) (IQRL: 0)
2280.82226563 [!] <ALLOC> (BUFFER: FFFFB5036E9EE000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
2280.84570313 [!] <FREE> (BUFFER: FFFFB5036E9EE000) (IDX: 38) (IQRL: 0)
2290.88281250 [!] <ALLOC> (BUFFER: FFFFB5036E9EE000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
2290.90014648 [!] <FREE> (BUFFER: FFFFB5036E9EE000) (IDX: 39) (IQRL: 0)
2585.81738281 [!] <ALLOC> (BUFFER: FFFFB5036E9EE000) (CALLER: 0000000000A7CAD4) (PV: 0) (START: FFFFF8075559CFA0)
2585.82812500 [!] <FREE> (BUFFER: FFFFB5036E9EE000) (IDX: 40) (IQRL: 0)
4527.77343750 [!] <ALLOC> (BUFFER: FFFFB5036E9EE000) (CALLER: 0000000000101514) (PV: 0) (START: FFFFF8075559CFA0)
4527.90234375 [!] <FREE> (BUFFER: FFFFB5036E9EE000) (IDX: 41) (IQRL: 0)
4799.58203125 [!] <ALLOC> (BUFFER: FFFFB50370A8A000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
4799.60449219 [!] <FREE> (BUFFER: FFFFB50370A8A000) (IDX: 42) (IQRL: 0)
4851.77734375 [!] <ALLOC> (BUFFER: FFFFB50370B17000) (CALLER: 00000000005EF7BC) (PV: 0) (START: FFFFF8075559CFA0)
4851.79687500 [!] <FREE> (BUFFER: FFFFB50370B17000) (IDX: 43) (IQRL: 0)
4885.99707031 [!] <ALLOC> (BUFFER: FFFFB50374648000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
4886.02099609 [!] <FREE> (BUFFER: FFFFB50374648000) (IDX: 44) (IQRL: 0)
4901.03710938 [!] <ALLOC> (BUFFER: FFFFB50374648000) (CALLER: 00000000005EF7BC) (PV: 0) (START: FFFFF8075559CFA0)
4901.05664063 [!] <FREE> (BUFFER: FFFFB50374648000) (IDX: 45) (IQRL: 0)
5085.52636719 [!] <ALLOC> (BUFFER: FFFFB50374617000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
5085.54785156 [!] <FREE> (BUFFER: FFFFB50374617000) (IDX: 46) (IQRL: 0)
5100.55175781 [!] <ALLOC> (BUFFER: FFFFB50374617000) (CALLER: 00000000005EF7BC) (PV: 0) (START: FFFFF8075559CFA0)
5100.56933594 [!] <FREE> (BUFFER: FFFFB50374617000) (IDX: 47) (IQRL: 0)
5343.82910156 [!] <ALLOC> (BUFFER: FFFFB50374617000) (CALLER: 00000000005EF7BC) (PV: 0) (START: FFFFF8075559CFA0)
5343.84667969 [!] <FREE> (BUFFER: FFFFB50374617000) (IDX: 48) (IQRL: 0)
5348.29443359 [!] <ALLOC> (BUFFER: FFFFB50374617000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
5348.31982422 [!] <FREE> (BUFFER: FFFFB50374617000) (IDX: 49) (IQRL: 0)
9327.91210938 [!] <ALLOC> (BUFFER: FFFFB503727E7000) (CALLER: 0000000000101514) (PV: 0) (START: FFFFF8075559CFA0)
9327.99609375 [!] <FREE> (BUFFER: FFFFB503727E7000) (IDX: 50) (IQRL: 0)
12538.43066406 [!] <ALLOC> (BUFFER: FFFFB503727E7000) (CALLER: 00000000005EF7BC) (PV: 1) (START: 00007FFD4D362630)
12538.45605469 [!] <FREE> (BUFFER: FFFFB503727E7000) (IDX: 51) (IQRL: 0)
12553.46386719 [!] <ALLOC> (BUFFER: FFFFB503727E7000) (CALLER: 00000000005EF7BC) (PV: 0) (START: FFFFF8075559CFA0)
12553.48339844 [!] <FREE> (BUFFER: FFFFB503727E7000) (IDX: 52) (IQRL: 0)
13447.64062500 [!] <ALLOC> (BUFFER: FFFFB503727E7000) (CALLER: 00000000005EF7BC) (PV: 0) (START: FFFFF8075559CFA0)
13447.65917969 [!] <FREE> (BUFFER: FFFFB503727E7000) (IDX: 53) (IQRL: 0)
```
---------------------------------------------------------------------------------------------------------------------------------------

PATCHED:https://www.unknowncheats.me/forum/3457323-post50.html


Looks like EAC changed their dynamic import code to break the method after seeing this thread.

They now resolve ExAllocatePoolWithTag once on start-up with the old find-by-hash method, but it's stored crypted.

![image](https://user-images.githubusercontent.com/13917777/174869973-c5d67744-8c3b-4b87-86e4-b70dd5dadd24.png)

Then when they want to call it they call the crypt function again but with the third param set to 0 instead of 1 to decrypt instead of encrypt.

![image](https://user-images.githubusercontent.com/13917777/174870014-975f7dbf-90c3-4af3-80fa-8f61e72e15ec.png)

For reference this is how they used to do it, just call the resolve function inline the at the first call attempt and then cache the plain pointer for future calls.

![image](https://user-images.githubusercontent.com/13917777/174870062-2f0fa0a1-5dc4-4b5d-87ff-3f57906f9a2b.png)

Knowing EAC you could probably just call import_cipher with your hook address and set last param to 1 to blindly bypass this.
