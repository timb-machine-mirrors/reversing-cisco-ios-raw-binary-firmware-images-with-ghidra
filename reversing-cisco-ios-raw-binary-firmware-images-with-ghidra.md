I was recently interested in reversing some older Cisco IOS images.  Those images come in the form of a single binary blob, without any sort of ELF, Mach-o, or PE header to describe the binary.

While I am using Cisco IOS Images in this example, the same process should apply to other Raw Binary Firmware Images.

That makes importing this type of firmware file difficult, as Ghidra doesn't have any idea what type of ISA it needs to disassemble and decompile for.

The following are a few things I learned while trying to get the Cisco IOS image in a reversible state within Ghidra.

First I had to extract the image.  The first 112 bytes of the firmware I received from the vendor are some sort of Cisco proprietary header that is not useful for our purpose.  We need to extract the bzip2 archive that we are interested in.  The easist way to do that is binwalk:
```
binwalk -eM c3750-ipservicesk9-mz.122-50.SE3.bin
```
This will create a `_c3750-ipservicesk9-mz.122-55.SE.bin.extracted` directory which will have a file named `70` inside it. 

Now we need to figure out the CPU ISA.  For this we use binwalk again:

```
binwalk -m /usr/local/lib/python2.7/dist-packages/binwalk/magic/binarch _c3750-ipservicesk9-mz.122-55.SE.bin.extracted/70
```

This will output a lot of things, so lets take a look at the output:

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
24            0x18            PowerPC big endian instructions, function prologue
1360          0x550           PowerPC big endian instructions, function epilogue
1364          0x554           PowerPC big endian instructions, function epilogue
1372          0x55C           PowerPC big endian instructions, function epilogue
1380          0x564           PowerPC big endian instructions, function epilogue
1388          0x56C           PowerPC big endian instructions, function prologue
1612          0x64C           PowerPC big endian instructions, function epilogue
1648          0x670           PowerPC big endian instructions, function epilogue
1656          0x678           PowerPC big endian instructions, function prologue
3224          0xC98           PowerPC big endian instructions, function epilogue
3232          0xCA0           PowerPC big endian instructions, function prologue
6772          0x1A74          PowerPC big endian instructions, function epilogue
6780          0x1A7C          PowerPC big endian instructions, function prologue
[...]
```

We can see that the binary has Big Endian PowerPC function prologues followed by epilogues.  This is a good indicator that the firmware image ISA is PowerPC Big Endian.

Now that we know the ISA, we need to know the text-base offset and the data-base offset within the firmware image. The best way to figure this out is to load the firmware on an actual device and boot up the device.

To retrieve the base address (fileOffset), run the `show version` command on the Cisco Switch:
```
Switchy>
Switchy>enable
Password:
Switchy#show version
Cisco IOS Software, C3750 Software (C3750-IPSERVICESK9-M), Version 12.2(55)SE, RELEASE SOFTWARE (fc2)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2010 by Cisco Systems, Inc.
Compiled Sat 07-Aug-10 22:45 by prod_rel_team
Image text-base: 0x01000000, data-base: 0x02F00000

ROM: Bootstrap program is C3750 boot loader
BOOTLDR: C3750 Boot Loader (C3750-HBOOT-M) Version 12.2(44)SE5, RELEASE SOFTWARE (fc1)
[...]
```

This dumps out a line: `Image text-base: 0x01000000, data-base: 0x02F00000`.  
Both of those addresses are important, so note them and save them for later.

Now open the `70` binary blob in ghidra.  Again, since there is no standardized binary format for the binary, you will have to import the file as `Raw Binary`, and then set the Code Architecture to `PowerPC Big Endian 4xx`.  Also, click the options button and set the image offset base to the value we retrieve from the `show version` command: 0x01000000.  Then import.

Ghidra will then churn on the binary for a while, and when it is done the strings should be resolved to labels within the decompiler. This is because the label regions in memory are marked as Read/Write within ghidra. We want to resolve those labels to strings for ease of use.

Open up `Window->Memory Map` and click the `Split` button up in the right hand corner of that screen.  You will need to split at the data-base address `0x02f00000`, and then mark the data-base memory region as Read Only.  

After you have completed these steps, the labels should resolve to strings and you should be able to start reversing the image quite easily.