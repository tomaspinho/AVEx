# AVEx

### Compilation ###
    # To compile Visual Boy Advance use this in its directory
    CXXFLAGS=-fpermissive ./configure
    make

#### Compilation for Profiling ####
    # To compile Visual Boy Advance with opcode profiling capabilities
    # This counts the number of dispatches per opcode and the maximum time
    # an opcode's dispatch has taken and produces opcodeextimes.csv and opcodetimes.csv
    # in the current directory
    CXXFLAGS="-fpermissive -DC_CORE -DPROFILING -DDEV_VERSION -DAVEXPROFILING" CFLAGS="-g" ./configure
    make
    
#### Installation
    # It produces messy results with all compiled objects inside src directories. Install with
    sudo make install

### Running
This emulator needs to be tested using a GBA ROM file, which can be found, for example, at [EmuParadise](http://www.emuparadise.me/Nintendo_Gameboy_Advance_ROMs/31).
In our tests, we used *Super Mario World: Super Mario Advance 2* and *Bomberman Max 2*.
    # Running from command line under Linux or Mac OS
    ./VisualBoyAdvance (ROM .gba file)

------
### Original Codebase ###
Can be found [here](http://sourceforge.net/projects/vba/files/VisualBoyAdvance/1.7.2/).

### Process VM Interpreter ###
Uses a **decode-and-dispatch** approach to emulating the **16.78 MHz ARM7TDMI processor** (32bit GameBoy Advance) and the **8 or 4 MHz Z80 coprocessor** (8bit GameBoy). Does not rely on data structures for execution, but rather on local variables and calls to functions that implement each opcode functionality.
The "big switch" is located inside `src/arm-new.h`.

#### Opcode Profiling ####
We have implemented opcode profiling routines in the emulator that count the number of times a specific opcode is dispatched and, upon the closing of the application, write these results to disk in a convenient CSV format. 
Analyzing the counts for each opcode's executions (see `opcodetimes.csv` and `mostfrequentopcodes.txt`) we come to the conclusion that arithmetic instructions are the most executed, followed by memory loading opcodes, branching instructions and memory storing opcodes. Therefore, instruction optimization priority should follow these results.

#### Possible optimizations ####
Version 1.7.2 of the Visual Boy Advance emulator implements the "big switch" of the decode-and-dispatch approach using copious amounts of local variables, function calls, macro expansions for direct array indexing and expression evaluation to prepare certain values for operations. Therefore, possible simple optimizations include cleaning redudant local variables while promoting them to globals, optimizing function implementations and evaluated expressions.

Possible complex optimizations include changing the **decode-and-dispatch** to an **indirect threaded interpretation** approach, merging all logic and called functions into a lookup table indexed by the opcode. However, recent GCC versions already decide on optimizing switch-case constructs into lookup tables with function pointers, and this may be pointless.

A **predecoding** approach may be a possible optimization and a predecoded opcode cache may improve execution times. The GameBoy Advance has atmost `32mb` of cartridge **ROM**, which means that, given a instruction of `32bit` size, there would be, atmost, 8.388.608 instructions in a **ROM** (`32mb = 256mbit / 32bit`). Given a `4byte` sized function pointer (standard in C x86) and up to 3 auxiliar `int` values (`4bytes` each), the `struct` held in cache would occupy `16bytes`. That would mean a `134217728bytes = 128mb` (8.388.608 * 16bytes) sized cache for all opcodes in a given ROM. We choose to, due to time restrictions, implement **predecoding** for the statistically ten most executed opcodes (see Opcode Profiling).

### Memory Architecture ###
The GameBoy Advance has a **32 kilobyte internal DRAM** + **96 kilobyte VRAM** (internal to the CPU) and **256 kilobyte DRAM** (outside the CPU). These RAMs are implemented in the form of arrays allocated in the emulator's heap space.

In the following table we attempt to describe all arrays in the emulator.

 Name        | Size       | VBA array name      | Description
-------------|------------|---------------------|------------
System Rom   | 16kb       | bios                | BIOS memory. Executable functions to enable faster development.
DRAM         | 256kb      | workRAM             | External work RAM. Can be used for code and data.
Internal RAM | 32kb       | internalRAM         | This memory is embedded in the CPU and it's the fastest memory section. The 32bit bus means that ARM instructions can be loaded at once. This is available for code and data.
IO RAM       | 1kb        | ioMem               | Memory-mapped IO registers. This section is used to control graphics, sound, buttons and other features.
PAL RAM      | 1kb        | paletteRAM          | Memory for two palettes containing 256 entries of 15-bit colors each. The first is for backgrounds, the second for sprites.
VRAM         | 96kb       | vram                | Video RAM. This is where the data used for backgrounds and sprites are stored. The interpretation of this data depends on a number of things, including video mode and background and sprite settings.
OAM          | 1kb        | oam                 | Object Attribute Memory. This is where sprites are controlled.
PAK ROM      | up to 32mb | rom                 | This is the memory of the game cartridge inserted into the console.
Cart RAM     | variable   | handled directly (1)| This is where saved data is stored. Cart RAM can be in the form of SRAM, Flash ROM or EEPROM. It's present inside the cartridge.

VBA array name | Allocation Location (line)  | Notes
---------------|-----------------------------|---------------------------------------------
BIOS           | src/GBA.cpp (1327)          | Initially, VBA had to load a bios file dumped from the original console directly into this array. Eventually, the bios was reimplemented by functions present in `bios.cpp`
workRAM        | src/GBA.cpp (1278)          |
internalRAM    | src/GBA.cpp (1334)          |
ioMem          | src/GBA.cpp (1369)          |
paletteRAM     | src/GBA.cpp (1341)          |
vram           | src/GBA.cpp (1348)          | This allocation asks for more (128kb) than originally needed (92kb)
oam            | src/GBA.cpp (1355)          |
rom            | src/GBA.cpp (1272)          | Loading is done by reading a file in (`src/GBA.cpp` - line 1290) and writing to the array through a for loop (line 1322)

(1) Handled directly through functions implemented in VBA. Save files are loaded to memory, written and saved to disk.

------
#### Memory Access Functions ####
Due to GameBoy Advanced's shared address space, switch cases implementing read/write opcodes have to translate (2) given addresses to match the appropriated allocated array in the emulator. Therefore, several auxiliar functions were implemented: 

##### Memory read #####

CPUReadMemory(address) :-> 32-bit value, GBAinline.h (41)

CPUReadHalfWord(address) :-> 16-bit value, GBAinline.h (159)

CPUReadHalfWordSigned(address) :-> 16-bit value, GBAinline.h (261)

##### Memory write #####

void CPUWriteMemory(address, 32-bit value), GBAinline.h (344)

void CPUWriteHalfWord(address, 16-bit value), GBA.cpp (2714)

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

##### Initial Results #####
We have implemented a predecoded instruction cache that contains a `struct` per opcode inside the ROM. Initially, we fill it for all opcodes with the pointer to a function that implements the default emulator behaviour: the "big-switch" dispatch.
Without profiling capabilities, the results are as follows.

Version           | Minimum (3)| Maximum (3)
------------------|------------|------------
Original          | 208%       | 694%
With cache        | 207%       | 671%

The emulator is slowed down by the cached version because there are no opcode caching optimizations in place. Only reading the function pointer from a given opcode's `struct` in the cache. As it is performing more memory accesses, the emulator is slightly slowed down.

(3) - Percentage of original hardware speed. Minimum is the default speed. Maximum is with maximum throttling (so the emulator performs at full CPU speed).

##### Optimizations #####
We optimized the following opcodes: 0x009 - MUL, 0x3cf - BIC, 0x080 - ADD, 0x250 - SUBS and 0xa00-0xaff - B.
We did this by caching results from operations inside the `switch cases` in the instruction case and replacing the initial `big switch` function pointer with an implementation of the same dispatch function that reads from the cache.
