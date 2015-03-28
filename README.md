# AVEx

### Compilation ###
    # To compile Visual Boy Advance use this in its directory
    CXXFLAGS=-fpermissive ./configure
    make
    
    # It produces messy results with all compiled objects inside src directories. Install with
    sudo make install

### Process VM Interpreter ###
Uses a decode-and-dispatch approach to emulating the 16.78 MHz ARM7TDMI processor (32bit GameBoy Advance) and the 8 or 4 MHz Z80 coprocessor (8bit GameBoy). Does not rely on data structures for execution, but rather on local variables and calls to functions that implement each opcode functionality.
The "big switch" is located inside src/arm-new.h.

### Memory Architecture ###
The GameBoy Advance has a 32 kilobyte internal DRAM + 96 kilobyte VRAM (internal to the CPU) and 256 kilobyte DRAM (outside the CPU). These RAMs are implemented in the form of arrays allocated in the emulator's heap space.

In the following table we attempt to describe all arrays in the emulator.

 Name        | Size       | VBA array name      | Description
-------------|------------|---------------------|------------
System Rom   | 16kb       | bios                | BIOS memory. Executable functions to enable faster development.
DRAM         | 256kb      | workRAM             | External work RAM. Can be used for code and data.
Internal RAM | 32kb       | internalRAM         | This memory is embedded in the CPU and it's the fastest memory section. The 32bit bus means that ARM instructions can be loaded at once. This is available for code and data.    
IO RAM       | 1kb        | ioMem               | Memory-mapped IO registers. This section is used to control graphics, sound, buttons and other features.             
PAL RAM      | 1kb        | paletteRAM          | Memory for two palettes containing 256 entries of 15-bit colors each. The first is for backgrounds, the second for sprites.         
VRAM         | 96kb       | vram                | Video RAM. This is where the data used for backgrounds and sprites are stored. The interpretation of this data depends on a number of things, including video mode and background and sprite settings.
OAM          | 1kb        | oam                 | Object Attribute Memory. This is sprites are controlled.
PAK ROM      | up to 32mb | rom                 | This is the memory of the game cartridge inserted into the console.
Cart RAM     | variable   | handled directly (1)| This is where saved data is stored. Cart RAM can be in the form of SRAM, Flash ROM or EEPROM. It's present inside the cartridge.

VBA array name | Allocation Location (line)  | Notes
---------------|-----------------------------|---------------------------------------------
BIOS           | src/GBA.cpp (1327)          | Initially, VBA had to load a bios file dumped from the original console directly into this array. Eventually, the bios was reimplemented by functions present in bios.cpp
workRAM        | src/GBA.cpp (1278)          |
internalRAM    | src/GBA.cpp (1334)          |
ioMem          | src/GBA.cpp (1369)          |
paletteRAM     | src/GBA.cpp (1341)          |
vram           | src/GBA.cpp (1348)          | This allocation asks for more (128kb) than originally needed (92kb)
oam            | src/GBA.cpp (1355)          |
rom            | src/GBA.cpp (1272)          | Loading is done by reading a file in (src/GBA.cpp - 1290) and writing to the array through a for loop (1322)

(1) Handled directly through functions implemented in VBA. Save files are loaded to memory, written and saved to disk.

#### Memory Access Functions ####
Due to GameBoy Advanced's shared address space, switch cases implementing read/write opcodes have to translate (2) given addresses to match the appropriated allocated array in the emulator. Therefore, several auxiliar functions were implemented: 

##### Memory read #####

CPUReadMemory(address) :-> 32-bit value
GBAinline.h (41)

CPUReadHalfWord(address) :-> 32-bit value
GBAinline.h (159)

CPUReadHalfWordSigned(address) :-> 16-bit value
GBAinline.h (261)

##### Memory write #####

void CPUWriteMemory(address, 32-bit value)
GBAinline.h (344)

void CPUWriteHalfWord(address, 16-bit value)
GBA.cpp (2714)

(2) - Translation is done through 24-bit shift-rights, due to GBA's conveniently laid out address space, to identify one of the memories below:

Memory            | start      | end
------------------|------------|-----
System ROM        | 0000:0000h | 0000:3FFFh
External Work RAM | 0200:0000h | 0203:FFFFh
External Work RAM | 0300:0000h | 0300:7FFFh
IO RAM            | 0400:0000h | 0400:03FFh
PAL RAM           | 0500:0000h | 0500:03FFh
VRAM              | 0600:0000h | 0601:7FFFh
OAM               | 0700:0000h | 0700:03FFh
PAK ROM           | 0800:0000h | variable
Cart RAM          | 0E00:0000h | variable
