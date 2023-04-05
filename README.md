# Info.

An English translation of the PC-98 game **Aegean Kai No Shizuku** by Illusion Core.

# How to install.

The required downloads can be found on the [Releases](https://github.com/Svipur/AegeanKaiNoShizukuPC98-RUS/releases) page. The installation process is covered thereon as well.

# VC format specifications.

The game uses a custom format for its graphics. The same VC format is also used in other later games by Illusion: Kankin (which also mixes in their earlier format PG), Rankou Nyotaitsuri, Ura Mansion Hakkin, Hyouryuu.

The structure of the image is this:

|Segment|Description|
|---|---|
|**Header**|Fixed length, always 168h.|
|**1st colour channel**|Split into up to 9 blocks with a 00 byte appended at the end of each block.|
|**2nd colour channel**|Split into up to 9 blocks with a 00 byte appended at the end of each block.|
|**3rd colour channel**|Split into up to 9 blocks with a 00 byte appended at the end of each block.|
|**4th colour channel**|Split into up to 9 blocks with a 00 byte appended at the end of each block.|

1. If data in the four channels is identical, the image is treated as a 2-colour (1-byte) image. Otherwise it's treated as a 16-colour (4-byte) image.
2. The pixel data is stored in binary in 8-pixel columns, from top to bottom, each byte representing a line of 8 pixels.
3. The data employs 2 methods of compression - one is a custom RLE method, and the other is a rudimentary method of repeating existing chunks. Both are described below.

## Header.

All WORD-type numbers are big-endian.

|**Offset**|**Description**|
|---|---|
|18-19, 1a-1b, 1c-1d, 1e-1f|The number of blocks each channel is split into, respectively.|
|24-25|Horizontal size divided by 8.|
|26-27|Vertical size.|
|28-29|Uncompressed length (in lines/bytes) of the 1st data block. Usually equals 100e (3600 lines).|
|2a-2b|The length of the 1st block (in bytes) with RLE compression, but without the chunk compression.|
|2c-2d|The length of the 1st block (in bytes) with both compression methods.|
|2e-2f|The length of the control block after the first block, containing the lookup bytes for the chunk compression.|
|30-77|Same 4 WORD-type numbers for the remaining blocks in the 1st channel.|
|78-c7|Same blocks of 4 WORD-type numbers for the 2nd channel.|
|c8-117|Same blocks of 4 WORD-type numbers for the 3rd channel.|
|118-167|Same blocks of 4 WORD-type numbers for the 4th channel.|

## RLE compression.

When parsing the data block, if there are two identical bytes in a row, the third byte is treated as the control byte. It indicates the number of times the same byte should be repeated afterwards.

E.g. 11 11 03 refers to a run of 11 11 11 11 11.

**Note**: a run of 2 bytes is still treated as a run, i.e. '11 11' should be encoded as '11 11 00'.

## Chunk compression.

The compressed 'chunks' are pairs of bytes in the data block that refer to a previous data segment and say how many consecutive bytes should be copied from that segment.

The chunks are encoded as follows:

1. The first byte is split into nibbles.
2. The first nibble is the low byte of the offset backwards.
3. The second nibble says how many bytes to copy from that offset.
3. The second byte is the high byte of the offset.
4. The offset is counted using the length without the chunk compression (for each previous compressed chunk subtract 2nd nibble minus 2).

The chunk compression is optional.

The control block after each block is thus also optional. It indicates which data bytes should be treated as compressed chunks, rather than binary. It is separated from both the preceding and the following blocks with a 00 byte.

1. Each byte in the control block represents a group of 8 bytes in the data block (in binary).
2. A binary 1 in a control byte means that the given byte is the start of a compressed chunk.
3. Each consecutive compressed chunk is offset by 1 leftwards. E.g. an 82h at the start would refer to the  sequence 1000 0001 (81h) - where the 1st compressed chunk is at its natural position, and the 2nd one actually starts on the final byte. For decoding purposes, treat every binary 1 as a 10, or rotate the entire sequence rightwards every time a 1 is encountered.

## Channels and their respective colours.

0000: `#000000`
1000: `#880000`
0100: `#CC1122`
0010: `#6677CC`
0001: `#EEAA99`
1100: `#110077`
1010: `#cc8800`
1001: `#ffddcc`
0110: `#ffdd99`
0101: `#ee77bb`
0011: `#669966`
1110: `#bb7766`
1101: `#005522`
1011: `#bbbbdd`
0111: `#8888aa`
1111: `#ffffff`

# Changes to the EXE file.

|Offset|Original HEX|Changed to|Change in ASM|Description|
|---|---|---|---|---|
|1adf3|26803c**21**|26803c**40**|`CMP ES:Si,21 -> 40`|We change the EOF symbol from ! to @ to let us use '!'.|
|1ae70|2ec7064b010**2**00|2ec7064b010**1**00|`MOV CS:14b,02 -> 01`|Halves the distance the caret is moved after each char, getting us half-width chars.|
|1aec9|268**b**04|268**a**04|`Mov AX, Es:SI -> Mov AL, Es:SI`|Loads the char code as 1 byte instead of 2.|
|1aece|9ae7029014|eb0890xxxx|`CALL 1490 -> JMP 01d8`|Skips additional maths with the char code and jumps straight to char output.|
|1ae31|83c602|83c601|`ADD SI,02 -> 01`|Moves the pointer by 1 byte after reading char code instead of 2.|
|1aef8|83c602|83c601|`ADD SI,02 -> 01`|Moves the pointer by 1 byte after reading char code instead of 2.|
|1af24|83c602|83c601|`ADD SI,02 -> 01`|Moves the pointer by 1 byte after reading char code instead of 2.|
|1af43|83c602|83c601|`ADD SI,02 -> 01`|Moves the pointer by 1 byte after reading char code instead of 2.|
|1af66|81ed0020|90909090|`MOV BP,2000 -> NOP`|Optional. We're not using BP for storing the high byte of the char code, we can NOP this one out.|
|1af6a|8bc5|b000|`MOV AX,BP -> MOV AL,00`|Makes the high byte of the char code 00.|
