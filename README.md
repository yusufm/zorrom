Zorro's (mask) ROM

Utilities to convert between physical and logical representation of chip's memory layout.
For example, a photograph of a boot ROM that has been converted into a 2D array of bits (.txt) can be converted into a machine readable binary (.bin).
This .bin can then emulated, disassembled, etc as you'd do with any normal firmware file.

You can generate a .txt file to input via:
* typing by hand
* rompar: https://github.com/AdamLaurie/rompar
* bitract: https://github.com/SiliconAnalysis/bitract/
* django-monkeys: https://github.com/andrew-gardner/django-monkeys/blob/master/tools/db2txt.py

Main programs:
  * txt2bin.py: convert bits in .txt file into a machine readable binary
  * bin2txt.py: convert a machine readable binary into a .txt file
  * txtmunge.py: modify a raw .txt file by flipping, rotating, inverting, etc

Advanced programs:
  * vizlayout.py: visually show the memory layout of given architecture
  * bindiff.py: compute statistics between two ROMs
  * imgdiff.py: visually compare two ROMs
  * test_misc.py: test suite
  * randbin.py: generate a test binary for given architecture


# Auto solver tutorial

About half of the ROMs we encounter:
  * Linearly lay out bits
  * Don't interleave (ie all of the least significant bits are together)
  * Have some known pattern, such as a common first instruction

If this is the case the solver may be able to automatically decode your ROM.

Example: Game Boy boot ROM

A Z80 like architecture may start by setting up stack, something like "ld sp,$XXXX". This means that we expect the first byte to be 0x31 for the "ld, sp" instruction. Lets add this as a constraint and run the analyzer:

```
$ wget https://www.neviksti.com/DMG/DMG_ROM.txt
$ ./solver.py --bytes 0x31 DMG_ROM.txt DMG_ROM
...
Exact matches: 2
  Writing out/r-90_flipx-1_invert-0_cols-lr-r.bin
  Writing out/r-90_flipx-1_invert-0_cols-ud-r.bin
```

Analayzing candidates:

```
$ unidasm -arch lr35902 DMG_ROM/r-90_flipx-1_invert-0_cols-ud-r.bin |head -n 1
00: 31 fe ff  ld   sp,$FFFE
$ unidasm -arch lr35902 DMG_ROM/r-90_flipx-1_invert-0_cols-lr-r.bin |head -n 1
00: 31 11 47  ld   sp,$4711
```

Only the first seems reasonable to setup the stack pointer. Additional analysis of the first binary and/or comparing to existing data shows its the correct binary.

Alternatively if we knew ahead of time the stack pointer value we could be more explicit:

```
$ ./solver.py --bytes 0x31,0xfe,0xff DMG_ROM.txt DMG_ROM
...
Exact matches: 1
  Writing DMG_ROM/r-90_flipx-1_invert-0_cols-ud-r.bin
```

Constraints can be put in any of these forms:
* value: assumes an auto incrementing address, starting from 0. All bits are matched
* address:value: explicit address. All bits are matched
* address:value:mask: explicit address and only bits in the given bitmask are checked

For example, if we thought the stack pointer would be in the upper address space but not sure where we could do this:

```
$ ./solver.py --bytes 0x00:0x31,0x02:0x80:0x80 DMG_ROM.txt DMG_ROM
...
Exact matches: 1
  Writing DMG_ROM/r-90_flipx-1_invert-0_cols-ud-r.bin
```

Which does:
  * 0x00:0x31: match byte at address 0x00 to value 0x31
  * 0x02:0x80:0x80: match byte at address 0x02 to value 0x80, but only check bit 0x80

## Algorithm

The parameters output are:
* r: clockwise rotation
* flipx: whether the ROM is mirrored
* invert: whether bits are inverted (ie swap 0 and 1)
* cols-lr-l: bits are in contiguous columns. They start upper left, move right, and then wrap around next row left
* cols-lr-r: bits are in contiguous columns. They start upper right, move left, and then wrap around next row right
* cols-ud-l: bits are in contiguous columns. They start upper left, move down, and then wrap around next column top
* cols-ud-r: bits are in contiguous columns. They start upper right, move down, and then wrap around next column top


# Developer guide

## Tutorial 1: Hello, World!

Say a character ROM was photographed and converted (say by hand) to produce a bit pattern like this:

```
1111111111111111
0000011000001111
1000000000000111
1111111010111111
0100001101011111
1000001001001111
1111011000111111
1011011001110111
```

This ".txt" gives a ROM physical layout of 16 rows by 8 columns. Conventions:
  * ROMs are usually oriented to be wider than high to fit on desktop monitors better
  * You may not know what bit is 0 vs 1 initially. Choose a convention arbitrarily and document it. That said, people usually choose the brighter bit as "0"

Now we need to come up with a theory how the bits are ordered. If the data is truly random / encrypted this is impossible to figure out. However, in most applications we have some idea of what the end data should look like and can make educated guesses.

Lets say that we suspect this contains ASCII data (as opposed to say machine instructions). Observations:
  * Often data fills in at the lowest address space and may be unused at the highest address. Since the right appears unused (low entropy), address is probably lower on the left and higher at right
  * ASCII data is often stored in 8 bit words (ie bytes). As the width (16) is a multiple of 8 as well as the height (8), its not immediately clear if addresses flow left/right or up/down
  * Typically the most significant bit of ASCII data is constant 0. We see this type of entropy pattern at the top, but its constant 1. Lets guess the most significant bit is at the top, but the data needs to be inverted

With these theories in mind lets create a decoder. First we need to define a few metadata functions:
  * desc: a brief description of this decoder
  * txtwh: how many columns and rows the input .txt file should have
    * In this case there are 16 columns and 8 rows, so return (16, 8)
    * Total expected size by default is calculated from the number of bits in the .txt file
  * invert: return True since we need to map 0 => 1 and 1 => 0

More advanced usage can define bits that don't map from the input .txt to the output .bin (ex: parity bits) or unusal word sizes. This will be covered in a later tutorial or see example decoders for more information.

In addition to metadata we need to define the transform from the .txt file to the output binary. There are a few ways to do this but the recommended way is to define the "oi2cr" function. This takes in the .bin file byte/word offset as well as a bit index ("oi") and must return the .txt file column and row ("cr", after removing whitespace).

Bit index meaning the corresponding bit mask is defined as 1 << index. For example, bit 7 is 1 << 7 => 0x80.

We're now ready to write our decoder function. Let's review what we suspect about the layout:
  * Lowest address at left (column 0), highest address at right (column 15)
  * Most significant bit at top (row 0), least significant bit at bottom (row 7)
  * Each column in our example is a byte

With that in mind our decode function might look like:

```
def oi2cr(self, offset, maski):
    return offset, 7 - maski
```

Where:
  * Column: the same as the byte offset
  * Row: bit 0 is in row 7 and bit 7 is in row 0. So this is complimented

Check out zorrom/tutorial1.py for the full implementation.

Now we just need to tie this into the architecture database to work. Edit zorrom/archs.py and add the import + matching table definition (already added for tutorial1.

Lets run the decoder:

```
python3 txt2bin.py --arch tutorial1 test/tutorial1.txt
hexdump -C test/tutorial1.bin
```

Which gives:

```
00000000  48 65 6c 6c 6f 2c 20 77  6f 72 6c 64 21 00 00 00  |Hello, world!...|
```

Please also add notes to the "Archs" section of this readme documenting information like your chip orientation and bit polarity convention.

Finally please add test data for the new architecture:

```
test_add.sh tutorial1
```

## Tutorial 2: machine example

TODO: do a small real world chip showing example interleaving and checking if the binary is valid. Maybe game boy boot ROM?


# Arch

Contains conventions to help you import specific architectures

## d8041ah (NEC)

Notes:
  * Technology: contact ROM
  * Polarity: contact as 1
  * Orientation: "NEC D8041AH" should be on the left such that address decoding circuitry is to the right and bottom

## m5l8042 (Mitsubishi)

Notes:
  * Technology: contact ROM
  * Polarity: FIXME
  * Orientation: FIXME


## mb8623x (Fujitsu)

Notes:
  * Technology: contact ROM
  * Polarity: FIXME
  * Orientation: FIXME

## pic1670 (Microchip/GI)

Notes:
  * Technology: implant NOR ROM
  * Polarity: etched bit 1 (appears darker)
  * Orientation: main decoding circuitry at left
  * First and last column row groups are not bits (appears as 10 bits but is 8)
  * Reference: https://siliconpr0n.org/map/apple/pic1670-adb-turbo/s1-3-dash-15_mit20x/

## lc5800 (Sayno)

Notes:
  * Technology: implant NOR ROM
  * Orientation: decoder at left
  * Polarity: FIXME
  * Reference: https://siliconpr0n.org/map/rsa/securid-3c58001e00/mz/

