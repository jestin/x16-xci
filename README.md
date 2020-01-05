# XCI: eXtremely Compact Interpreter
An adventure game engine for the Commander X16

## Overview
XCI is an adventure game engine designed to be as compact as
possible while still creating a rich experience, featuring a
point-and-click interface, animated sprites, and detailed backgrounds.
This is being designed specifically for the Commander X16
retrocomputer, which has just over 37kB of base RAM, 512kB of
extended RAM split into 8kB banks, and 128kB of VRAM. It uses a
65C02 CPU, standard PS/2 mouse and keyboard, and an FPGA-based
video adapter (VERA) that can output 256 colors at a 640x480 resolution.
Because of the memory restrictions, XCI will be using 16 colors
per element at a 320x240 resolution. The X16 allows elements
to have a palette offset to choose which "row" of 16 colors from
the 256-color palette will be used.

The elements of the game consist of three independent layers:
* 320x200 Bitmap backgound, placed 8 pixels from the top of the screen, then end 32 pixels from the bottom. The entire bitmap
will comprise of only 16 colors, taken from a palette offset.
* 40x30 map of 8x8 tiles. Each tile can have 16 colors, but also
its own palette offset. The tiles provide a menu at the top of the
screen, text display and controls at the bottom. Tiles may also
act as static overlays on top of the background. Up to 720 unique
tiles can be defined.
* 128 individual 16x16 sprites, including a dynamic mouse cursor.
Each sprite can have 16 colors, from its own palette offset. Up to 512 unique sprite frames can be defined. All sprites except for the mouse cursor are confined to the bitmap area, and support collision
with other sprites and tiles. The mouse cursor is context-sensitive,
changing its frame based on its position and game state to indicate
the type of action that will happen when the mouse button is clicked.

## Memory Map
Main RAM ($0801-$9EFF): XCI engine code, top-level game data

Banked RAM ($A000-$BFFF): 6 banks per level, up to 10 levels per currently loaded zone.
* Bank 0: Kernal Use
* Bank 1: Zone level 0 configuration data
* Banks 2-5: Zone level 0 background bitmap
* Bank 6: Zone level 0 music and sound effects
* Bank 7: Zone level 1 configuration data

...
* Bank 60: Zone level 9 music and sound effects

When transit from a level leads to a different zone, the new
zone is loaded to banked RAM from the file system. Up to 256
different zones can be defined.

VRAM:
* Bank 0: Bitmap ($0000-$95FF), Tile Map ($9600-$A5FF), Tiles ($A600-$FFFF)
* Bank 1: Sprites ($0000-$FFFF)

## Data Format
All game data is described in text files, starting with a main file. This file defines the top-level game data, providing references to all other source files. Its filename is the only input to the build utility (**xci.exe**). It is simply a set of key-value pairs. There are certain mandatory keys that are required for the game to be successfully built. Unrecognized keys are ignored by the build utility, as are comments, which begin with a hash (```#```) symbol. Keys have no spaces and are not case-sensitive, but values may be case-sensitive and consist of all text after the first whitespace after the key up to the end of the line or the start of a comment. New lines can be part of a value by using the escape code ```\n```. A value can contain a hash character by escaping it with a backslash (i.e. ```\#```).

### Main File
The following is an example of a main file, showing all required keys.

```
# This is a comment
title My Game
author John Doe
palette mygame_pal.hex
tiles mygame_tiles.hex
sprites mygame_sprites.hex
menu mygame_menu.xci
title_screen mygame_start.xci
init_cursor 0 # sprite frame index
zone mygame_zone0.xci # the first zone defined will be loaded first,
                      # with its first level the start of the game
zone mygame_zone1.xci
zone mygame_zone2.xci
```

Let's say this file is named **mygame.xci**. The game can be built by passing it to the build utility, as follows:

```
$ /path/to/xci.exe mygame.xci
```

If it finishes without errors, the game is built! Let's assume that zone 0 has 2 levels, zone 1 has 3 levels, and zone 2 a full 10 levels. The following binaries will be generated:

* **MAIN.BIN** - This is the compiled version of **mygame.xci**. This will be loaded into base RAM by the XCI program (**XCI.PRG**) at run time. This and all binaries generated by the build utility (in addition to **XCI.PRG**) need to be loaded into the X16 file system to run the game. It will contain information from **mygame.xci** and other files specified by the main file keys.
* **PAL.BIN** - This is the binary version of **mygame_pal.hex**, which contains the initial palette for the game, as specified by the **palette** key. It will be loaded by the XCI program into VRAM (F:1000) prior to displaying the title screen. Note that the palette beyond index 15 will be modified as different parts of the game are loaded, including the title screen.
* **TILES.BIN** - This is the binary version of **mygame_tiles.hex**, which contains the tile set for the game, as specified by the **tiles** key. It will be loaded by the XCI program into VRAM (0:A600) prior to displaying the title screen.
* **SPRITES.BIN** - This is the binary version of **mygame_sprites.hex**, which contains the sprite frames for the game, as specified by the **sprites** key. It will be loaded by the XCI program into VRAM (1:0000) prior to displaying the title screen.
* **TITLE.BIN** - This is the background bitmap for the title screen, specified by the contents of **mygame_start.xci**, which will be explained later. It will be loaded into VRAM (0:0000) once the palette, tiles and sprites are all loaded. Unlike the background bitmaps of the game levels, this bitmap can take up the full screen (320x240). This will remain in VRAM and on screen until the user starts a new game or loads a saved game. It is never stored in base or banked RAM.
* **Z0.L0.1.BIN** - This is the configuration data for level 0 of zone 0. This is the first level that will be loaded after starting a new game. It is defined by the contents of **mygame_zone0.xci**, as specified by the first instance of the **zone** key. It will be loaded into bank 1 of banked RAM whenever the game enters level 0 (as it does when starting a new game, but a saved game may have also left off in level 0).
* **Z0.L0.2.BIN** - This is the background bitmap for level 0 of zone 0, specified by the contents of **mygame_zone0.xci**, which will be explained later. It will be loaded into banks 2-5 of banked RAM whenever the game enters zone 0, and into VRAM when the game enters level 0. Because of the in-game screen layout, this bitmap will start 8 pixel9s from the top, and therefore starting at VRAM address 0:0500. From 0:0000 to 0:04FF will be zero-filled, as it lies behind the menu bar. From 0:7D00 to 0:95FF will be zero-filled, as it lies behind the text field/toolbar.
* **Z0.L0.6.BIN** - The music and sound effects for level 0 of zone 0. The length of the music and addresses for each sound effect are defined in the level configuration data. This file will be loaded into bank 6 when the game enters zone 0. They data will be played directly from banked RAM when the game enters level 0.
* **Z0.L1.7.BIN** - This is the configuration data for level 1 of zone 0. It will be loaded to bank 7 when the game enters zone 0.
* **Z0.L1.8.BIN** - This file contains the background bitmap for level 1 of zone 0. It will be loaded into banks 8-11 when the game enters zone 0, and into VRAM starting at 0:0500 when the game enters level 1.
* **Z0.L1.12.BIN** - The music and sound effects for level 1 of zone 0. It will be loaded to bank 12 when the game enters zone 0. It is also the last file to be loaded into banked RAM for zone 0, as it only contains two levels. When starting a new game, or loading a game that left off in zone 0, the rest of the files below will not be loaded from the file system at the beginning of game play.
* **Z1.L0.1.BIN, Z1.L0.2.BIN, Z1.L0.6.BIN** - All data for level 0 of zone 1. These will be loaded into banks 1-6 when the game enters zone 1.
* **Z1.L1.7.BIN, Z1.L1.8.BIN, Z1.L1.12.BIN** - All data for level 1 of zone 1. These will be loaded into banks 7-12 when the game enters zone 1.
* **Z1.L2.13.BIN, Z1.L2.14.BIN, Z1.L2.18.BIN** - All data for level 2 of zone 1. These will be loaded into banks 13-18 when the game enters zone 1. They are the last files to be loaded into banked RAM for zone 1.
* **Z2.L0.1.BIN, Z2.L0.2.BIN, Z2.L0.6.BIN** - Zone 2, level 0
* **Z2.L1.7.BIN, Z2.L1.8.BIN, Z2.L1.12.BIN** - Zone 2, level 1
* **Z2.L2.13.BIN, Z2.L2.14.BIN, Z2.L2.18.BIN** - Zone 2, level 2
* **Z2.L3.19.BIN, Z2.L3.20.BIN, Z2.L3.24.BIN** - Zone 2, level 3
* **Z2.L4.25.BIN, Z2.L4.26.BIN, Z2.L4.30.BIN** - Zone 2, level 4
* **Z2.L5.31.BIN, Z2.L5.32.BIN, Z2.L5.36.BIN** - Zone 2, level 5
* **Z2.L6.37.BIN, Z2.L6.38.BIN, Z2.L6.42.BIN** - Zone 2, level 6
* **Z2.L7.43.BIN, Z2.L7.44.BIN, Z2.L7.48.BIN** - Zone 2, level 7
* **Z2.L8.49.BIN, Z2.L8.50.BIN, Z2.L8.54.BIN** - Zone 2, level 8
* **Z2.L9.55.BIN, Z2.L9.56.BIN, Z2.L9.60.BIN** - Zone 2, level 9. These are loaded into banks 55-60, the highest RAM banks that can be populated by an XCI game, when the game enters zone 2.

Wow, that's a lot of files! But most of them are 32kB or smaller. Each zone would easily fit on a single double-density 3.5" floppy, to put it in perspective. In this case, the whole game maxes out at 870kB (and that's assuming very complicated title screen and levels, and all available sprite frames and tiles defined), which would fit on a single high-density floppy, 2 double-density 3.5" floppies, or 3 double-density 5.25" floppies. The biggest XCI game possible would be 125MB, or 87 high-density 3.5" floppies. A CD-ROM could hold at least 5 XCI games. A 2GB SD Card could hold at least 16. Of course, it is highly unlikely that even the most prolific storyteller could come up with 2550 levels for a single game, so most games should clock in at around 5MB, which means that an X16 user could play hundreds of XCI games without changing their SD card. Because of filename conflicts, each game would need to be in a separate directory.

#### Required Keys

The following are the required keys for the main file. Any keys outside of these will be ignored. If any of these are missing from the main file or have values that are formatted incorrectly, the game will fail to build. Please note that your development platform may have a case-sensitive file system (e.g. Linux, Mac), so the capitalization of filename values matters. Meanwhile all keys are not case-sensitive, but they must be spelled correctly and be immediately followed only by whitespace and then the value. This is the case for all XCI configuration files.

* **title** - Title of the game, will be printed to console when graphics data is loading. The maximum length is 255 characters, and it will be placed at the very beginning of **MAIN.BIN**.
* **author** - Author of the game, which will also be printed to the console.  The maximum length is 255 characters, and it will be placed 256 bytes into **MAIN.BIN**, after the space reserved for the title. The title and author will also be the main meta data to discriminate between different instances of **MAIN.BIN**.
* **palette** - Filename of the hex file containing the initial palette. The format of this file will be explained in the [Palette Hex File](#palette-hex-file) section. This file must be in the same directory as the main file, or the value must contain the path to the file. This applies to all filename values in XCI configuration source files.
* **tiles** - Filename of the hex file containing the tile definitions. The format of this file will be explained in the [Tiles Hex File](#tiles-hex-file) section.
* **sprites** - Filename of the hex file containing the sprite definitions. The format of this file will be explained in the [Sprites Hex File](#sprites-hex-file) section.
* **menu** - Filename of the menu file, another XCI configuration file like the main file.  The format of this file will be explained in the [Menu File](#menu-file) section.
* **title_screen** - Filename of the title screen file.  The format of this file will be explained in the [Title Screen File](#title-screen-file) section.
* **init_cursor** - Sprite frame used as the initial mouse cursor.  The sprite frames start with index zero, which is the first 16x16 bitmap defined in **SPRITES.BIN**. This value must be between 0 and the highest sprite frame index, which could be as high as 511, depending on the size of **SPRITES.BIN**.
* **zone** - This is the only key which can have more than one instance in the main file. The first instance will be for filename of the zone file that will be used for zone 0. The remaining instances will determine the index order for all zone files, from 1 up to 255. Any **zone** instances past the 256th will be ignored. The format for these files will be explained in the [Zone Files](#zone-files) section.

### Hex Files
Hex files are a human-friendly way of specifying binary data. Every four bits of data are represented by a single hexadecimal digit, represented as an ASCII character, ```0``` through ```9``` and ```A``` through ````F````, representing the binary values of 0000 through 1111, or 0 through 15 in decimal. The format of the binary data specified by hex files is determined by the X16's VERA video adapter, which is documented in the [VERA Programmer's Reference](https://github.com/commanderx16/x16-docs/blob/master/VERA%20Programmer's%20Reference.md). The binary files generated from the hex files will be loaded directly into VRAM and the contents are not verified in any way.

Like XCI configuration files, hex files can contain comments starting with ```#```, which may be at the very beginning of a line, or after a line of hex characters.  If any non-hex or non-whitespace characters appear on a line before a ```#```, the game will fail to build.  Hex files are not case sensitive at all, so upper and lowercase letters are interchangeable for ```A``` through ```F```.

Whitespace can be used however the developer sees fit, separating each byte with a space, or just having a whole block of hex characters uninterrupted. The only limitations are that each line may be no more than 1000 characters wide, and have an even number of hex characters (bytes may not be split across lines). The following is an example of a valid hex file.

```
# Header comment
12 34 56 78       # Bytes
9ABC DEF0         # Words
13579BDF 02468ACE # double words
```

To be loadable by the X16, binary files require a two-byte header, which is effectively ingored, so the binaries build from hex files will always start with two null, or zero bytes. The following table shows byte-by-byte what will be written to a binary file that would be build from the above hex file. Note that the hex files have no concept of "endianness" based on the whitespace; each 4-bit nibble will be written out in binary in the order they are read from hex characters.

| Hex | Binary | Decimal | Notes     |
|-----|--------|---------|-----------|
|   00|00000000|        0|Ignored, but required by X16|
|   00|00000000|        0|Ignored    |
|   12|00010010|       18|From line 1, this is the first actual byte that will be loaded to VRAM|
|   34|00110100|       52|           |
|   56|01010110|       86|           |
|   78|01111000|      120|           |
|   9A|10011010|      154|From line 2|
|   BC|10111100|      188|           |
|   DE|11011110|      222|           |
|   F0|11110000|      240|           |
|   13|00010011|       19|From line 3|
|   57|01010111|       87|           |
|   9B|10011011|      155|           |
|   DF|11011111|      223|           |
|   02|00000010|        2|           |
|   46|01000110|       70|           |
|   8A|10001010|      138|           |
|   CE|11001110|      206|This is the last of 16 bytes that will be loaded to VRAM|

Now, we will see what each of the hex files should contain.

#### Palette Hex File
The initial palette hex file may contain all 256 colors, but for practical purposes it only needs to define the first 16 colors of the palette.  This is in the format specified by the VERA documentation: two bytes for each color, with 4 bits for each of red, green and blue and four unused bits.  This means that a single hex character represents each color component from 0 to F (15 decimal). In binary, the two-byte field is formatted as ```bbbbgggg 0000rrrr```. So, pure white would be ```11111111 00001111``` in binary, or ```FF 0F``` in hex. So, 16 colors would be simply 64 hex characters.

The following example shows how a hex file would specify the first 16 colors of the default X16 palette.

```
# Default X16 palette, offset 0
0000 FF0F 0008 FE0A 4C0C C500 0A00 E70E 850D 4006 770F 3303 7707 F60A 8F00 BB0B
```

When VERA elements are using 16 colors (as all elements in XCI game do), the color is specified by a four-bit value, or a single hex digit, representing 0 through 15. Despite how the palette may be configured, color 0 of every palette offset is transparent, so it might as well be black, as shown above. As we can see, the next color value is white, which is color index 1. So, a bitmap where each byte is hex ```01``` would have alternating transparent and white pixels. Looking down the palette, we can see that color index 2 is a medium red, as the blue and green values are set to zero, but the red value is set to 8, which is the middle intensity between 1 and 15. Making each byte of a bitmap hex ```12``` would have alternating white and red pixels, with no transparency. We will see more concrete examples of hex-encoded bitmaps in the next two sections, which will assume the default palette.

#### Tiles Hex File
The tiles for XCI games are 8x8 pixels, with 4 bits per pixel. So, for the hex file, each hex character will represent a pixel of a tile, and 64 characters (32 bytes) are required to define one tile. XCI allows for up to 720 different tiles to be defined in this file, but at least 177 need to be defined. This is because tile indices 32 (space) through 176 (tilde) will be an ASCII character font. Included with the XCI development kit is an example tile set that defines a default character font and some other basic tiles that are needed for the menu and toolbar. But, you are completely free to redefine all of them, just remember the usage of those ASCII character tiles for the menu and text field is fixed.

Also note that while tiles being used for the menu, text field and tool will be using palette offset zero, as defined by the palette hex file, within the levels, any tile may be used with any of the palette offsets available in the current zone. This means that tiles (and sprites!) can be used as overlays on the level bitmap to add additional color depth. Tiles can also be flipped both horizontally and vertically.

The following is an example of the beginning of a tile set. Comments describe what each tile is supposed to be.

```
# Tile 0: Solid white square
11111111
11111111
11111111
11111111
11111111
11111111
11111111
11111111

# Tile 1: Small red circle with white outline, transparent background
00000000
00011000
00122100
01222210
01222210
00122100
00011000
00000000

# Tile 2: White square with rounded corner. Can be flipped to round different
#         different corners of a white box
11100000
11111000
11111100
11111110
11111110
11111111
11111111
11111111

# Tile 3: Commander X16 "Butterfly"
44000044
0EE00EE0
00333300
00055000
00777700
08800880
22000022
00000000
```

Using these tiles, we can build a little tile map that defines a rounded white box with the red circle and butterfly inside. The first tile we need is the upper-left corner. Tile 2 has the corner in the upper-right corner, so it will need to be flipped horizontally. Then, we need two white squares (tile 0), and then the upper-right corner, where we can use tile 2 without any flipping. That's the entire first row, so
onto the second row, where we start with a white square for the left side, then the circle (tile 2), the butterfly (tile 3) and another white square for the right side. Finally, the last row starts with tile 2 flipped both horizontally and vertically to make the lower-left corner, two more white squares, and then finishing with tile 2 flipped vertically to make the lower-right corner.

And here's what you get: ![tile map example](example/tilebox.png)

Later on, we'll see how tiles are used to define the menu and toolbar, and how they can be added to game levels.

#### Sprites Hex File
All sprites for XCI games are 16x16 pixels, but since there are up to 128 sprites available for rendering at any time, larger sprites can be accomplished by synchronizing the movements of adjacent sprites. Each sprite can have 16 colors, and use any palette offset at any time.  Like tiles, sprites can add to the color depth of an image, with the added capability of being able to be placed at any position on screen.  The only required sprite is the mouse cursor, which must be sprite index 0. However, any sprite frame may be used for the mouse cursor, but its initial frame is determined my the **init_cursor** key in the main file.  Generally this should be a simple pointer cursor, but during game play the cursor can be context-sensitive, changing its frame based on its position and game state. Many games have a player avatar sprite, but that is not required for XCI. Sprite movements may be pre-programmed to happen once or loop through a level, or they could respond to mouse actions. Each frame of each sprite's potential movement needs to be defined, but like tiles, sprite frames can change their appearance through changing the palette offset or flipping horizontally, vertically or both.

Each sprite frame has an index, which is used to specify which frame any given sprite should be displaying according to the game configuration. Any sprite can use any of the frames, from index 0 through 511, or whatever the highest defined sprite index is in the hex file. The example below shows a very basic example of a sprite frame set. You may recognize the player avatar from [Chase Vault](https://github.com/SlithyMatt/x16-chasevault).

```
# frame 0: initial mouse cursor
f000000000000000
f100000000000000
f110000000000000
f111000000000000
f111100000000000
f111110000000000
f111111000000000
fff1100000000000
000f110000000000
000f100000000000
0000000000000000
0000000000000000
0000000000000000
0000000000000000
0000000000000000
0000000000000000
# frame 1: player avatar standing, side
0000000977000000
0000007797700000
0000079779977000
0000077997700000
0000999776a00000
0000777aaaaa0000
000070aaaaa00000
0000000aaa000000
0000009997700000
0000095775500000
0000995775500000
00009959a5500000
0000999555000000
0000000990000000
0000000990000000
0000000bbb000000
# frame 2: player avatar walking 1, side
0000000977000000
0000007797700000
0000079779977000
0000077997700000
0000999776a00000
0000777aaaaa0000
000070aaaaa00000
0000000aaa000000
0000009997700000
0000095777500000
0000995577500000
000099595a500000
0000999555000000
0000000999000000
00000b99099b0000
000000bb0bb00000
# frame 3: player avatar walking 2, side
0000000977000000
0000007797700000
0000079779977000
0000077997700000
0000999776a00000
0000777aaaaa0000
000070aaaaa00000
0000000aaa000000
0000009997700000
00000957777a0000
0000995597700000
0000995955500000
0000999555000000
00000b99999b0000
00000b90009b0000
000000b000b00000
# frame 4: player avatar walking 3, side
0000000977000000
0000007797700000
0000079779977000
0000077997700000
0000999776a00000
0000777aaaaa0000
000070aaaaa00000
0000000aaa000000
0000009997700000
0000097775500000
0000997795500000
0000995a55500000
0000999555000000
0000000999000000
00000b99099b0000
000000bb0bb00000
# frame 5: player avatar walking 7, side
0000000977000000
0000007797700000
0000079779977000
0000077997700000
0000999776a00000
0000777aaaaa0000
000070aaaaa00000
0000000aaa000000
0000009997700000
0000077795500000
0000a77795500000
00009959a5500000
0000999555000000
00000b99999b0000
00000b90009b0000
000000b000b00000
```

These 6 sprite frames are enough to have a mouse cursor and a player avatar that can walk from side to side. Frames 1-5 have the avatar facing right, but they each can be flipped horizontally to walk to the left.  

With a [simple BASIC program](example/sprite.bas), we can test both the mouse sprite and do a little animation with the avatar: ![sprite demo GIF](example/sprite.gif)

Notice that the mouse pointer has its point in the upper left corner of the frame. This is because that corner pixel is the actual mouse position. This goes for all sprites, where the position specified will be its upper-left corner, so make sure that there is room below and to the right of the position to display the portion of the sprite you want visible.  We'll see how to do this later when configuring game levels.

### Menu File
The menu file is a key-value configuration text file, just like the main file.  This one defines the appearance and behavior of the menu bar, which occupies the top 8 pixel rows of the screen, as well as the toolbar, which can appear at the bottom of the screen. These elements are all rendered completely with tiles and using only palette offset 0 to maintain consistency throughout the game. The example menu file and tiles provide the basis for any game, but you can customize them however you want for your own game. The example menu file is shown below.

```
(TODO)
```

### Title Screen File
(TODO)

#### Raw Image Files
(TODO)

### Zone Files
(TODO)

#### Level Files
(TODO)

#### VGM Files
(TODO)

#### Sound Effects Files
(TODO)

## Building XCI Toolchain from Source
(TODO)

## Building XCI Game Binaries
(TODO)

## Deploying XCI Game
(TODO)

## Licensing
(TODO)

## Notes
