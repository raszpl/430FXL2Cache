Late Award BIOS (4-5x PnP) post code list:

|  Code   |  Meaning   |
| --- | --- |
|  C0   |  1. Turn off OEM specific cache, shadow<br>2. Initialize standard devices with default values:<br>- DMA controller (8237)<br>- Programmable Interrupt Controller (8259)<br>- Programmable Interval Timer (8254)<br>- RTC chip  |
|  C1  |  Auto detection of onboard DRAM & Cache  |
|  C3  |  1. Test system BIOS checksum<br>2. Test the first 256K DRAM<br>3. Expand the compressed codes into temporary DRAM area including the compressed System BIOS & Option ROMs  |
|  C5  |  Copy the BIOS from ROM into E0000-FFFFF shadow RAM so that POST will go faster  |
|  01-02   |  Reserved  |
|  03  |  Initialize EISA registers (EISA BIOS only)  |
|  04  |  Reserved  |
|  05  |  1. Keyboard Controller Self Test<br>2. Enable Keyboard Interface  |
|  06  |  Reserved  |
|  07  |  Verifies CMOS's basic R/W functionality  |
|  BE   |  Program defaults values into chipset according to the MODBINable Chipset Default Table  |
|  09  |  1. Program configuration register of Cyrix CPU according to the MODBINable Cyrix Register Table<br>2. OEM specific cache initialization (if needed)  |
|  0A  |  1. Initialize the first 32 interrupt vectors with corresponding interrupt handlers; Initialize INT No from 33-120 with Dummy (Spurious) interrupt handler<br>2. Issue CPUID instruction to identify CPU type<br>3. Early Power Management initialization (OEM specific)  |
|  0B  |  1. Verify the RTC time is valid or not<br>2. Detect bad battery<br>3. Read CMOS data into BIOS stack area<br>4. PnP initializations including (PnP BIOS only)<br>- Assign CSN to PnP ISA card<br>- Create resource map from ESCD<br>5. Assign IO & Memory for PCI devices (PCI BIOS only)  |
|  0C  |  Initialization of the BIOS data area (40:00-40:FF)  |
|  0D  |  1. Program some of the chipset's value according to setup. (Early setup value program)<br>2. Measure CPU speed for display & decide the system clock speed<br>3. Video initialization including Monochrome, CGA, EGA/VGA. If no display device found, the speaker will beep.  |
|  0E  |  1. Initialize the APIC (Multi-Processor BIOS only)<br>2. Test video RAM (If Monochrome display device found)<br>3. Show message including:<br>- Award logo<br>- Copyright string<br>- BIOS date code & Part No.<br>- OEM specific sign on messages<br>- Energy Star logo (Green BIOS only)<br>- CPU brand, type & speed  |
|  0F  |  DMA channel 0 test  |
|  10   |  DMA channel 1 test  |
|  11  |  DMA page registers test  |
|  12-13  |  Reserved  |
|  14  |  Test 8254 timer 0 counter 2  |
|  15  |  Test 8259 interrupt mask bits for channel 1  |
|  16  |  Test 8259 interrupt mask bits for channel 2  |
|  17  |  Reserved  |
|  19  |  Test 8259 functionality  |
|  1A-1D  |  Reserved  |
|  1E  |  If EISA NVM checksum is good, execute EISA initialization (EISA BIOS only)  |
|  1F-29  |  Reserved  |
|  30   |  Detect base memory & extended memory size  |
|  31  |  1. Test base memory from 256K to 640K<br>2. Test extended memory from 1M to the top of memory  |
|  32  |  1. Display the Award Plug & Play BIOS extension message(PnP BIOS only)<br>2. Program all onboard super I/O chips(if any) including COM ports, LPT ports, FDD port according to setup value  |
|  33-3B  |  Reserved  |
|  3C  |  Set flag to allow users to enter CMOS setup utility  |
|  3D  |  1. Initialise keyboard<br>2. Install PS2 mouse  |
|  3E  |  Try to turn on level 2 cache<br>Note: Some chipset may need to turn on the L2 cache in this stage. But usually, the cache is turn on later in Post 61h  |
|  3F-40  |  Reserved  |
|  BF  |  1. Program the rest of the chipset's value according to setup (later setup value program)<br>2. If auto configuration is enabled, programmed the chipset with predefined values in the MODBINable AutoTable  |
|  41   |  Initialize floppy disk drive controller  |
|  42  |  Initialize hard drive controller  |
|  43  |  If it is a PnP BIOS, initialize serial & parallel ports  |
|  44  |  Reserved  |
|  45  |  Initialize math coprocessor  |
|  46-4D  |  Reserved  |
|  4E  |  If there is any error detected (such as video, KB....), show all the error messages on the screen & wait for user to press <F1> key  |
|  4F  |  1. If password is needed, ask for password<br>2. Clear the Energy Star logo (Green BIOS only)  |
|  50   |  Write all the CMOS values currently in the BIOS stack are back into the CMOS  |
|  51  |  Reserved  |
|  52  |  1. Initialize all ISA ROMs<br>2.Later PCI initializations(PCI BIOS only)<br>- assign IRQ to PCI devices<br>- initialize all PCI ROMs<br>3. PnP initializations (PnP BIOS only)<br>- assign IO, Memory, IRQ & DMA to PnP ISA devices<br>- initialize all PnP ISA ROMs<br>4. Program shadow RAM according to setup settings<br>5. Program parity according to setup setting<br>6. Power Management initialization<br>- Enable/Disable global PM<br>- APM interface initialization  |
|  53  |  1. If it is not a PnP BIOS, initialize serial & parallel ports<br>2. Initialize time value in BIOS data area by translate the RTC time value into a timer tick value  |
|  54-5F  |  Reserved  |
|  60   |  Setup virus protection (boot sector protection) functionality according to setup setting  |
|  61  |  1. Try to turn on level 2 cache<br>Note: if L2 cache is already turned on in POST 3D, this part will be skipped<br>2. Set the boot up speed according to setup setting<br>3. Last chance for chipset initialization<br>4. Last chance for Power Management initialization (Green BIOS only)<br>5. Show the system configuration table  |
|  62  |  1. Setup daylight saving according to setup values<br>2. Program the NUM lock, typematic rate & typematic speed according to setup setting  |
|  63  |  1. If there is any changes in the hardware configuration, update the ESCD information (PnP BIOS only)<br>2. Clear memory that have been used<br>3. Boot system via INT 19h  |
|  FF   |  Boot  |
