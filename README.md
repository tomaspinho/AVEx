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
The GameBoy Advance has a 32 kilobyte + 96 kilobyte VRAM (internal to the CPU) and 256 kilobyte DRAM (outside the CPU). These RAMs are implemented in the form of arrays allocated in the emulator's heap space. Their declarations are present in src/Globals.h, beginning on line 63, and their allocation is performed in .

#### Memory Write Functions ####
Are declared within src/GBA.cpp, beginning on line 2714, after register accessing functions.
