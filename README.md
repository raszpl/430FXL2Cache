Intel 430FX L2 Cache auto detection investigation based on code extracted from Asus PCI/I-P54TP4 bios T15I0302.AWD

### Documentation:

[Award BIOS (4-5x PnP) post code list](Award%20BIOS%20(4-5x%20PnP)%20post%20codes.md)

Bios sections and their associated Post codes relevant are:

|     |     |
| --- | --- |
|  C1  |  Auto detection of onboard DRAM & Cache  |
|  3E  |  Try to turn on level 2 cache<br>Note: Some chipset may need to turn on the L2 cache in this stage. But usually, the cache is turn on later in Post 61h  |

[430FX chipset Datasheet](290518-002_82430FX_82437FX_82438FX_PCIset_DATASHEET_199612.pdf)

![image](cache%20control%20register.png)

Intel Engineer wanted to be funny and the last table has FLCE and SCFMI bit order reversed, got me more than once. Correct bit order:

| SCFMI | FLCE | L2 Cache Result |
| --- | --- | --- |
|0|0|Disabled
|1|0|Disabled; tag invalidate on reads
|0|1|Normal L2 cache operation (dependent on SGS)
|1|1|Enabled; miss forced on reads/writes|

### Looking at the code

I started by dissasembling bios file using IDA. You start by loading bios file and selecting "Intel 80x86 processors:metapc", hitting OK, then picking 16bit. Next you need to relocate bios code to the address bios resides in a real computer. Open "ICD Command..." (Shift+F2) and paste:

    SegCreate(0x000f0000, 0x00100000, 0xF000, 0, 0, 0);   
    SegRename(0x000f0000, "_F000");       
                    
    auto src = [0,0x10000], dest = [0xF000, 0];   
    auto ea_src, ea_dest, hi_limit; 
    hi_limit = src + 0x10000;       
    ea_dest = dest; 
            
    for(ea_src = src; ea_src < hi_limit ; ea_src = ea_src + 4 )             
    {       
    PatchDword( ea_dest, Dword(ea_src));    
    ea_dest = ea_dest + 4;  
    }       

Ugly, but it works. Now you have secont segment called _f000. Press G to "jump to Address" and paste _F000:fff0. Press C


Next was finding segment responsible for Post code C1. 

_F000:E4E7 POST_C1:                                ; DATA XREF: _F000:E4E5o
_F000:E4E7                 mov     al, 0C1h ; '-'
_F000:E4E9                 mov     dx, 80h ; 'Ç'
_F000:E4EC                 out     dx, al          ; manufacture's diagnostic checkpoint
_F000:E4ED                 mov     sp, 0E4F3h
_F000:E4F0                 jmp     ram_cache
