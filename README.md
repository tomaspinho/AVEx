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

#### Memory Write Functions ####
Are declared within src/GBA.cpp, beginning on line 2714, after register accessing functions.
