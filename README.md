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

### Looking for the code

I started by dissasembling bios file using IDA. You begin by loading bios file and selecting "Intel 80x86 processors:metapc", hitting OK, then clicking 16bit. Next you need to relocate code to the address actual BIOS is mapped at in a real computer. Open "ICD Command..." (Shift+F2) and paste:

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

Ugly, but it works. Now you have secont segment called \_f000. Press G to "Jump to Address" and paste \_F000:fff0. Press C to interpret data under cursor as Code and voila, you are looking at x86 legacy BIOS entry point.

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
    
After that its finally off to the races.
### Looking at the code
Actual L2 Cache detection procedure lies before us. First order of business seems to be setting maximum potential cache size supported (512KB), Cache type compatible with all possible choices - Async, and mode of operation "Disabled; tag invalidate on reads" resetting TAG sram Valid bits on access.

    _F000:E1F4 cache_detect:
    _F000:E1F4                 mov     cx, 52h
    _F000:E1F7                 mov     al, 0A2h        ; 512KB Async Disabled; tag invalidate on reads
    _F000:E1F9                 mov     sp, 0E1FFh
    _F000:E1FC                 jmp     pci_write_dev
    _F000:E1FC ; ---------------------------------------------------------------------------
    _F000:E1FF                 dw offset cache_invalidate
    _F000:E201 ; ---------------------------------------------------------------------------
    _F000:E201

we scan ram loading all possible cached range in order to ensure whole TAG ram is invalidated.

    _F000:E201 cache_invalidate:
    _F000:E201                 cld
    _F000:E202                 mov     dx, 8000h
    _F000:E205
    _F000:E205 loc_FE205:
    _F000:E205                 mov     ds, dx
    _F000:E207                 assume ds:nothing
    _F000:E207                 xor     si, si
    _F000:E209                 mov     cx, 4000h
    _F000:E20C                 rep lodsd
    _F000:E20F                 sub     dx, 1000h
    _F000:E213                 jnz     short loc_FE205
    
    
Eagle eyed among you might notice unusual x86 instruction combination REP LODS, often taught as useless and never used or even non existing at all! yet here it is doing the heavy lifting :) First to try is the most common L2 256KB Async setup.
    
    _F000:E215                 mov     cx, 52h
    _F000:E218                 mov     al, 61h         ; 256KB Async Normal L2 cache operation (dependent on SGS)
    _F000:E21A                 mov     sp, 0E220h
    _F000:E21D                 jmp     pci_write_dev
    _F000:E21D ; ---------------------------------------------------------------------------
    _F000:E220                 dw offset loc_FE222
    _F000:E222 ; ---------------------------------------------------------------------------
    _F000:E222
    _F000:E222 loc_FE222:
    _F000:E222                 mov     sp, 0E228h
    _F000:E225                 jmp     cache_test
    
cache_test is at the bottom. TLDR is
    ok - clc, take next jnb
    bad - stc, ignore next jnb
    
    _F000:E225 ; ---------------------------------------------------------------------------
    _F000:E228                 dw offset loc_FE22A
    _F000:E22A ; ---------------------------------------------------------------------------
    _F000:E22A
    _F000:E22A loc_FE22A:
    _F000:E22A                 jnb     short loc_FE26A
    _F000:E22C                 mov     cx, 52h
    _F000:E22F                 mov     al, 0B1h        ; 512KB PB Normal L2 cache operation (dependent on SGS)
    _F000:E231                 mov     sp, 0E237h
    _F000:E234                 jmp     pci_write_dev
    _F000:E234 ; ---------------------------------------------------------------------------
    _F000:E237                 dw offset loc_FE239
    _F000:E239 ; ---------------------------------------------------------------------------
    _F000:E239
    _F000:E239 loc_FE239:
    _F000:E239                 mov     sp, 0E23Fh
    _F000:E23C                 jmp     cache_test
    _F000:E23C ; ---------------------------------------------------------------------------
    _F000:E23F                 dw offset loc_FE241
    _F000:E241 ; ---------------------------------------------------------------------------
    _F000:E241
    _F000:E241 loc_FE241:
    _F000:E241                 jnb     short loc_FE296
    _F000:E243                 mov     cx, 52h
    _F000:E246                 mov     al, 51h         ; 256KB Burst Normal L2 cache operation (dependent on SGS)
    _F000:E248                 mov     sp, 0E24Eh
    _F000:E24B                 jmp     pci_write_dev
    _F000:E24B ; ---------------------------------------------------------------------------
    _F000:E24E                 dw offset loc_FE250
    _F000:E250 ; ---------------------------------------------------------------------------
    _F000:E250
    _F000:E250 loc_FE250:
    _F000:E250                 mov     sp, 0E256h
    _F000:E253                 jmp     cache_test
    _F000:E253 ; ---------------------------------------------------------------------------
    _F000:E256                 dw offset loc_FE258
    _F000:E258 ; ---------------------------------------------------------------------------
    _F000:E258
    _F000:E258 loc_FE258:
    _F000:E258                 jnb     short loc_FE296
    _F000:E25A
    _F000:E25A loc_FE25A:
    _F000:E25A                 mov     cx, 52h
    _F000:E25D                 mov     al, 22h         ; 0KB Async Disabled; tag invalidate on reads
    _F000:E25F                 mov     sp, 0E265h
    _F000:E262                 jmp     pci_write_dev
    _F000:E262 ; ---------------------------------------------------------------------------
    _F000:E265                 dw offset loc_FE267
    _F000:E267 ; ---------------------------------------------------------------------------
    _F000:E267
    _F000:E267 loc_FE267:
    _F000:E267                 jmp     loc_FE30E
    _F000:E26A ; ---------------------------------------------------------------------------
    _F000:E26A
    _F000:E26A loc_FE26A:
    _F000:E26A                 mov     cx, 52h
    _F000:E26D                 mov     al, 41h         ; 256KB PB Normal L2 cache operation (dependent on SGS)
    _F000:E26F                 mov     sp, 0E275h
    _F000:E272                 jmp     pci_write_dev
    _F000:E272 ; ---------------------------------------------------------------------------
    _F000:E275                 dw offset loc_FE277
    _F000:E277 ; ---------------------------------------------------------------------------
    _F000:E277
    _F000:E277 loc_FE277:
    _F000:E277                 mov     sp, 0E27Dh
    _F000:E27A                 jmp     cache_test
    _F000:E27A ; ---------------------------------------------------------------------------
    _F000:E27D                 dw offset loc_FE27F
    _F000:E27F ; ---------------------------------------------------------------------------
    _F000:E27F
    _F000:E27F loc_FE27F:
    _F000:E27F                 jnb     short loc_FE25A
    _F000:E281                 mov     cx, 52h
    _F000:E284                 mov     al, 61h         ; 256KB Async Normal L2 cache operation (dependent on SGS)
    _F000:E286                 mov     sp, 0E28Ch
    _F000:E289                 jmp     pci_write_dev
    _F000:E289 ; ---------------------------------------------------------------------------
    _F000:E28C                 dw offset loc_FE28E
    _F000:E28E ; ---------------------------------------------------------------------------
    _F000:E28E
    _F000:E28E loc_FE28E:
    _F000:E28E                 mov     sp, 0E294h
    _F000:E291                 jmp     cache_test
    _F000:E291 ; ---------------------------------------------------------------------------
    _F000:E294                 dw offset loc_FE296
    _F000:E296 ; ---------------------------------------------------------------------------
    _F000:E296
    _F000:E296 loc_FE296:
    _F000:E296                 mov     cx, 52h
    _F000:E299                 mov     sp, 0E29Fh
    _F000:E29C                 jmp     pci_read_dev
    _F000:E29C ; ---------------------------------------------------------------------------
    _F000:E29F                 dw offset loc_FE2A1
    _F000:E2A1 ; ---------------------------------------------------------------------------
    _F000:E2A1
    _F000:E2A1 loc_FE2A1:
    _F000:E2A1                 and     al, 3Fh
    _F000:E2A3                 or      al, 80h         ; set 512KB, leave type and mode as is
    _F000:E2A5                 mov     sp, 0E2ABh
    _F000:E2A8                 jmp     pci_write_dev
    _F000:E2A8 ; ---------------------------------------------------------------------------
    _F000:E2AB                 dw offset loc_FE2AD
    _F000:E2AD ; ---------------------------------------------------------------------------
    _F000:E2AD
    _F000:E2AD loc_FE2AD:
    _F000:E2AD                 xor     si, si
    _F000:E2AF                 xor     ax, ax
    _F000:E2B1                 mov     ds, ax
    _F000:E2B3                 assume ds:nothing
    _F000:E2B3                 mov     eax, [si]
    _F000:E2B6                 mov     dword ptr [si], 0A55A55AAh
    _F000:E2BD                 mov     dword ptr [si+4], 5AA5AA55h
    _F000:E2C5                 wbinvd
    _F000:E2C7                 mov     ax, 4000h
    _F000:E2CA                 mov     ds, ax
    _F000:E2CC                 assume ds:nothing
    _F000:E2CC                 mov     eax, [si]
    _F000:E2CF                 mov     dword ptr [si], 5AA5AA55h
    _F000:E2D6                 mov     dword ptr [si+4], 0A55A55AAh
    _F000:E2DE                 wbinvd
    _F000:E2E0                 xor     ax, ax
    _F000:E2E2                 mov     ds, ax
    _F000:E2E4                 assume ds:nothing
    _F000:E2E4                 cmp     dword ptr [si], 0A55A55AAh
    _F000:E2EB                 jnz     short loc_FE2F7
    _F000:E2ED                 cmp     dword ptr [si+4], 5AA5AA55h
    _F000:E2F5                 jz      short loc_FE30E
    _F000:E2F7
    _F000:E2F7 loc_FE2F7:
    _F000:E2F7                 mov     cx, 52h
    _F000:E2FA                 mov     sp, 0E300h
    _F000:E2FD                 jmp     pci_read_dev
    _F000:E2FD ; ---------------------------------------------------------------------------
    _F000:E300                 dw offset loc_FE302
    _F000:E302 ; ---------------------------------------------------------------------------
    _F000:E302
    _F000:E302 loc_FE302:
    _F000:E302                 and     al, 3Fh
    _F000:E304                 or      al, 40h         ; set 256KB, leave type and mode as is
    _F000:E306                 mov     sp, 0E30Ch
    _F000:E309                 jmp     pci_write_dev
    _F000:E309 ; ---------------------------------------------------------------------------
    _F000:E30C                 dw offset loc_FE30E
    _F000:E30E ; ---------------------------------------------------------------------------
    _F000:E30E
    _F000:E30E loc_FE30E:
    _F000:E30E                 mov     cx, 52h
    _F000:E311                 mov     sp, 0E317h
    _F000:E314                 jmp     pci_read_dev
    _F000:E314 ; ---------------------------------------------------------------------------
    _F000:E317                 dw offset loc_FE319
    _F000:E319 ; ---------------------------------------------------------------------------
    _F000:E319
    _F000:E319 loc_FE319:
    _F000:E319                 mov     ah, al
    _F000:E31B                 and     ah, 0F0h
    _F000:E31E                 cmp     ah, 80h         ; test if 512KB was selected?
    _F000:E321                 jnz     short loc_FE325
    _F000:E323                 or      al, 30h         ; if 512KB then set PB
    _F000:E325
    _F000:E325 loc_FE325:
    _F000:E325                 or      al, 3           ; set Enabled; miss forced on reads/writes
    _F000:E327                 mov     sp, 0E32Dh
    _F000:E32A                 jmp     pci_write_dev
    _F000:E32A ; ---------------------------------------------------------------------------
    _F000:E32D                 dw offset cache_prefill
    _F000:E32F ; ---------------------------------------------------------------------------
    _F000:E32F
    _F000:E32F cache_prefill:
    _F000:E32F                 mov     dx, 8000h
    _F000:E332
    _F000:E332 loc_FE332:
    _F000:E332                 mov     ds, dx
    _F000:E334                 assume ds:nothing
    _F000:E334                 mov     cx, 4000h
    _F000:E337                 xor     si, si
    _F000:E339                 rep lodsd
    _F000:E33C                 sub     dx, 1000h
    _F000:E340                 jnz     short loc_FE332
    _F000:E342                 mov     al, 0
    _F000:E344                 mov     sp, 0E34Ah
    _F000:E347                 jmp     CMOS_L1cache
    _F000:E347 ; ---------------------------------------------------------------------------
    _F000:E34A                 dw offset loc_FE34C
    _F000:E34C ; ---------------------------------------------------------------------------
    _F000:E34C
    _F000:E34C loc_FE34C:
    _F000:E34C                 shr     esp, 10h
    _F000:E350                 clc
    _F000:E351                 retn
    _F000:E351 ram_cache       endp
    _F000:E351
    _F000:E352
    _F000:E352 ; =============== S U B R O U T I N E =======================================
    _F000:E352
    _F000:E352
    _F000:E352 cache_test      proc near
    _F000:E352                 cld
    _F000:E353                 mov     ax, 1000h
    _F000:E356                 mov     ds, ax
    _F000:E358                 assume ds:nothing
    _F000:E358                 xor     si, si
    _F000:E35A                 mov     cx, 1000h
    _F000:E35D                 rep lodsd
    _F000:E360                 mov     ax, 1000h
    _F000:E363                 mov     es, ax
    _F000:E365                 assume es:nothing
    _F000:E365                 xor     di, di
    _F000:E367                 mov     eax, 0A55A55AAh
    _F000:E36D                 mov     edx, 3C3C33CCh
    _F000:E373                 mov     cx, 100h
    _F000:E376
    _F000:E376 loc_FE376:
    _F000:E376                 mov     es:[di], eax
    _F000:E37A                 add     di, 4
    _F000:E37D                 mov     es:[di], edx
    _F000:E381                 add     di, 4
    _F000:E384                 not     eax
    _F000:E387                 not     edx
    _F000:E38A                 loop    loc_FE376
    _F000:E38C                 wbinvd
    _F000:E38E                 mov     cx, 100h
    _F000:E391                 xor     di, di
    _F000:E393                 mov     eax, 0A55A55AAh
    _F000:E399                 mov     edx, 3C3C33CCh
    _F000:E39F
    _F000:E39F loc_FE39F:
    _F000:E39F                 cmp     es:[di], eax
    _F000:E3A3                 jnz     short loc_FE3BB
    _F000:E3A5                 add     di, 4
    _F000:E3A8                 cmp     es:[di], edx
    _F000:E3AC                 jnz     short loc_FE3BB
    _F000:E3AE                 add     di, 4
    _F000:E3B1                 not     eax
    _F000:E3B4                 not     edx
    _F000:E3B7                 loop    loc_FE39F
    _F000:E3B9                 clc
    _F000:E3BA                 retn
    _F000:E3BB ; ---------------------------------------------------------------------------
    _F000:E3BB
    _F000:E3BB loc_FE3BB:
    _F000:E3BB                 stc
    _F000:E3BC                 retn
    _F000:E3BC cache_test      endp
