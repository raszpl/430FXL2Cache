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

Ugly, but it works. Now you have secont segment called \_f000. Press G to "Jump to Address" and paste \_F000:fff0. Press C to interprete data under cursor as Code and voila, you are looking at x86 legacy BIOS entry point.

My next step was finding segment responsible for Post code C1. Easiest way is locating instruction that outputs actual POST code to port 80h. Here it is:

    _F000:E4E7 POST_C1:
    _F000:E4E7                 mov     al, 0C1h
    _F000:E4E9                 mov     dx, 80h
    _F000:E4EC                 out     dx, al          ; manufacture's diagnostic checkpoint
    _F000:E4ED                 mov     sp, 0E4F3h
    _F000:E4F0                 jmp     ram_cache

First part of C1 is responsible for detecting ram, lets skip that. Next we switch back to real mode.

    _F000:E1DD torealmode:
    _F000:E1DD                 mov     eax, cr0
    _F000:E1E0                 and     al, 0FEh
    _F000:E1E2                 mov     cr0, eax
    _F000:E1E5                 jmp     far ptr loc_FE1EA
    _F000:E1EA loc_FE1EA:

That jump at the end is very important to properly switch CPU mode. Next fragment seems to disable/enable L1 cache depending on a variable stored in CMOS under address 3Dh (0BDh but actual address is only 7 bit).

    _F000:E1EA                 mov     al, 0FFh
    _F000:E1EC                 mov     sp, 0E1F2h
    _F000:E1EF                 jmp     CMOS_L1cache
    
    _F000:F4FC CMOS_L1cache    proc near
    _F000:F4FC                 mov     ah, al
    _F000:F4FE                 mov     al, 0BDh
    _F000:F500                 out     70h, al         ; CMOS Memory:
    _F000:F500                                         ;
    _F000:F502                 out     0E1h, al
    _F000:F504                 in      al, 71h         ; CMOS Memory
    _F000:F506                 cmp     al, 0FFh
    _F000:F508                 jz      short locret_FF534
    _F000:F50A                 test    al, 80h
    _F000:F50C                 jz      short locret_FF534
    _F000:F50E                 and     al, 7Eh
    _F000:F510                 nop
    _F000:F511                 nop
    _F000:F512                 or      ah, ah
    _F000:F514                 jnz     short L1cache_enable
    _F000:F516                 mov     eax, cr0
    _F000:F519                 or      eax, 60000000h
    _F000:F51F                 mov     cr0, eax
    _F000:F522                 wbinvd
    _F000:F524                 jmp     short locret_FF534
    _F000:F526 ; ---------------------------------------------------------------------------
    _F000:F526
    _F000:F526 L1cache_enable:
    _F000:F526                 mov     eax, cr0
    _F000:F529                 and     eax, 9FFFFFFFh
    _F000:F52F                 mov     cr0, eax
    _F000:F532                 wbinvd
    _F000:F534
    _F000:F534 locret_FF534:
    _F000:F534                 retn
    _F000:F534 CMOS_L1cache    endp
    
After that is off to the races:

