# SASPPU

## Introduction

_"Super Analogue Stick Picture Processing Unit"_ (SASPPU) is a custom graphics engine for the EMF Tildagon that can entirely bypass the default graphics api [ctx](./ctx.md), all the way down to flipping to the display. It is inspired by how the [SNES](https://en.wikipedia.org/wiki/Super_Nintendo_Entertainment_System) and [GBA](https://en.wikipedia.org/wiki/Game_Boy_Advance) "PPU"s draw to the screen, and is written in a mixture of C and _inline esp32s3 SIMD assembly (!)_. It supports two tiled backgrounds, up to 256 sprites, HDMA (scanline tricks, like [Amiga Copper](https://en.wikipedia.org/wiki/Amiga_Original_Chip_Set#Copper)), and full per pixel "color math" and "window" effects.

### A feature list:
* Full 240x240 resolution with RGB555 (15bit) colour.
* Two 512x512 pixel backgrounds, made up from up to 1024 8x8 tiles.
* Ability to flip background tiles in the x and y axis.
* Infinite per pixel background scrolling.
* Up to 256 sprites, of any size (as long as they are a multiple of 8 wide), of which you can get 16 on any one scanline.
* A dedicated 256x256 pixel area for sprite graphics.
* Ability to flip each sprite individually in the x and y axis.
* Ability to double the size of each sprite individually (in both axes at once), with nearest-neighbour upscale.
* Position sprites anywhere on screen, including offscreen, and either in front or behind the higher background.
* Well defined sprite layering in relation to each other.
* Disable the background and sprite layers if they are not needed.
* Colour 0 is transparent and allows backgrounds to have holes and sprites to not be squares.
* Both a main screen and a sub screen, which can be combined into one as a final step using per pixel mathematical operations.
* Changeable solid background colour for both main and sub screen.
* Ability to send each background and individual sprite to either the main screen or the sub screen, or both.
* Two "windows", which allow you to mask off parts of the screen using boolean operations.
* Per background and individual sprite window settings.
* Dedicated fade to/from black feature.
* HDMA: Per scanline adjustment of all registers (including all sprites), so you can perform fancy wavy effects and copper bars.
* Easily fits into flash alongside the firmware (~70kb or so), and doesn't take up that much ram (surprisingly), with the large structures residing in PSRAM.
* Significantly faster than ctx: easily hits 30fps, and on simple scenes can reach 60fps (!), even with the entire rest of the OS running.
* Uses inline esp32s3 SIMD assembly to render 8 pixels at once.
* Co-exists with ctx. SASPPU and ctx are chosen on a per frame basis, and switching in and out of a SASPPU app is seamless.
* Comes with some helper functions for copying data around the graphics memory, including some simple graphics decompression, and routines to draw text to the backgrounds.
* Deterministic. HDMA notwithstanding, if you call "SASPPU_render" twice you should get the same frame twice.
* Renders directly into the Tildagon framebuffer, with no double buffering - but also no tearing :)

### Caveats:
* Much harder to use than ctx. You can draw a line with ctx with a couple lines of code, but SASPPU requires much more effort to get graphics on screen.
* Due to the statically allocated memory buffers that SASPPU uses, only one app can safely use it at once - including minimised apps. If another app tries to start the scheduler refuses to load it.
* An app has to pick either ctx or SASPPU. It cannot currently switch between, and either way it is not possible to use them in the same frame.
* Due to the two caveats above, notifications and other overlay layers are not currently supported.
* ctx specialises in vector graphics, and SASPPU specialises in bitmap graphics. Trying to do the opposite in each is not easy/performant/both.
* No palette, and therefore no oldschool palette effects. Memory bandwidth isn't what it used to be, sorry.
* No arbitrary rotation or scaling (or SNES mode 7) for the same reason as above.
* No anti-aliasing. But then again, that's at least slightly an aesthetic decision :)

## Overview

If you know how the SNES, NES or GBA render frames it should be at least somewhat familiar already.

As opposed to the immediate mode command based style of ctx, SASPPU is entirely retained. The average rendering loop within SASPPU is as follows:

* Set up all the "registers" before the frame starts
* Render the entire frame in one go, as an opaque operation
* Put that framebuffer on the screen.

As an app developer you will only ever need to worry about the first step. The other two steps will be handled by the operating system. Therefore, most of rendering a frame with SASPPU is writing values into variables.

### Additional Reading
Although not entirely applicable to SASPPU, a good primer on how the SNES renders to the screen is available [here](https://www.youtube.com/playlist?list=PLHQ0utQyFw5KCcj1ljIhExH_lvGwfn6GV).

The following videos are most useful:
* [Objects](https://www.youtube.com/watch?v=sheOZ-Dlleo)
* [Backgrounds and Rendering](https://www.youtube.com/watch?v=uRjf8ZP6rs8)
* [Color Math](https://www.youtube.com/watch?v=zcoU6-9_fDM)
* [DMA & HDMA](https://youtu.be/K7gWmdgXPgk?feature=shared&t=368) (only the part on HDMA)

## Render order

The exact steps SASPPU follows is as follows:

* At the start of each scanline, execute the next line of any enabled HDMA tables.
* Clear the main screen and sub screen to black.
* Fill the main screen with the main screen background colour, masked by the main screen portion of the background colour window settings
* Fill the sub screen with the sub screen background colour, masked by the sub screen portion of the background colour window settings
* If background 0 is enabled:
  * Retrieve the tilemap values for background 0.
  * Read in the pixel data that the map points to.
  * If the tilemap says to flip a tile, do so.
  * Scroll the background.
  * If the background 0 cmath flag is set, force the pixel's cmath flag on.
  * Draw the background over the main and sub screen, as dictated by the background 0 window settings.
* If sprites 0 are enabled:
  * For the first 16 sprites with low priority that are enabled:
    * Calculate the bounding box of the sprite
    * Read in the sprite tiles
    * If the sprite is flipped or doubled, transform the tiles as such.
    * Scroll the sprite.
    * If the sprite's cmath flag is set, force the pixel's cmath flag on.
    * Draw the sprite over the main and sub screen, as dictated by the sprite's window settings.
* If background 1 is enabled:
  * As above, but for background 1
* If sprites 1 are enabled:
  * For the first 16 sprites with high priority that are enabled:
    * As above, but for sprites 1
* If color math is enabled:
  * For all pixels in the main screen that have the color math bit set:
    * If main screen doubling is enabled, the main screen RGB values are doubled and clipped to pure white.
    * If main screen halving is enabled, the main screen RGB values are halved and clipped to pure black.
    * If sub screen doubling is enabled, the sub screen RGB values are doubled and clipped to pure white.
    * If sub screen halving is enabled, the sub screen RGB values are halved and clipped to pure black.
    * If add sub screen is enabled, the sub screen RGB values are added onto the main screen RGB values and clipped to pure white.
    * If sub sub screen is enabled, the sub screen RGB values are subtracted from the main screen RGB values and clipped to pure black.
* If fade is enabled:
  * All pixels in the main screen are darkened based on the fade value.
* The main screen is copied into the output framebuffer.

## Individual Features

### Colours

SASPPU uses 16 bits to encode a colour, which are laid out as follows:

```
15             0
CRRRRRGGGGGBBBBB
```
where:
 * C - [color math](#color-math) enable bit
 * R - red component, 0-31
 * G - green component, 0-31
 * B - blue component, 0-31

SASPPU therefore can display 2^15 colours, or 32768, with 32 shades of grey.

The colour 0x00 is special, and is considered transparent. If this colour appears in a sprite or background it will let the colour behind show through. The colour behind all layers is true black.

You may construct these colours yourself, but additionally there are helper macros (functions in micropython) that can construct them for you.

#### Colours Reference

### Backgrounds

Backgrounds are massive screen sized images that can be scrolled around. SASPPU provides two backgrounds, each 512x512 pixels in size.

Backgrounds are composed of many 8x8 pixel tiles, which are stored in the background [graphics memory](#memories). A tilemap dictates which tiles are used and where.

Unlike sprites, a background has no bounds and can be considered infinite. If a background is scrolled past its borders, it seamlessly wraps around. That is to say, if you were to scroll a background 512 pixels in any direction you would end up with the same image.
As the background is infinite if you wish to see what is behind the background tiles should use the transparent colour to poke holes.

Each of the two backgrounds can be entirely disabled in the MainState flags.

#### Tilemap

The tilemap is a 32x32 character array which dictates what tiles are used in a background and where. When drawing a background, SASPPU will first read the tilemap, and then use the pointers there to read the actual background pixel data.

Each tile map entry is 16 bits, and is laid out as follows:

```
15             0
YYYYYYYYXXXXX0TL
```
where:
 * Y - Y index into background memory of the pixel data
 * X - X index into background memory of the pixel data
 * T - flip tile top to bottom
 * L - flip tile left to right

The X index into the background memory is in steps of 8 - that is, incrementing the X index by one moves you 8 pixels to the right. Therefore all background pixel data must sit on an 8 pixel horizontal boundary.

Setting the flip bits will make the tile appear flipped vertically or horizontally, or both. This is independent of any other tile.

Access to the background tilemaps is performed directly, in both C and micropython.

#### Background Reference

| C     | Micropython   | Summary |
| ----- | ------------- | ------- |
| Background | sasppu.Background |  Struct/class containing all background state |
| SASPPU_bg0_state | Background.bind(0) | The state of background 0 |
| SASPPU_bg1_state | Background.bind(1) | The state of background 1 |
| SASPPU_bg0 | sasppu.bg0 | The tilemap for background 0 |
| SASPPU_bg1 | sasppu.bg1 | The tilemap for background 1 |
| SASPPU_background | N/A | The pixel data for both backgrounds |
| Background.x | Background.x | The horizontal scroll position of the background |
| Background.y | Background.y | The vertical scroll position of the background |
| Background.windows | Background.windows | The windowing state for the background |
| Background.flags | Background.flags | Additional boolean values for the background |
| BG_C_MATH | Background.C_MATH | Flag to force color math on for all pixels of the background |
| BG_WIDTH | Background.WIDTH | Width of a background (512) |
| BG_HEIGHT | Background.HEIGHT | Height of a background (512) |
| BG_WIDTH_POWER | Background.WIDTH_POWER | Width of a background log2 (9) |
| BG_HEIGHT_POWER | Background.HEIGHT_POWER | Height of a background log2 (9) |

### Windows

SASPPU provides two windows, which are infinitely tall rectangular sections of the screen. These windows can be used to mask away parts of the background and sprites.

Each windows is defined by a left and right x position. All pixels that are greater or equal to the left position and less than or equal to the right position are considered "inside" the window, and all other pixels are not.

Each background and sprite defines a byte of data which dictates how the two windows are used to mask them out. It is laid out as follows:

```
7      0
YXBAyxba
```
where:
 * A - Inside window A but not window B
 * B - Inside window B but not window A
 * X - Inside both window A and window B
 * Y - Outside both windows
The capital letters apply to the sub screen, and the lowercase to the main screen.

When a pixel is to be merged with the main and sub screen, the windowing rules take place. It will be merged with each of the screens if its x position fulfills any of the conditions in the set bits of the window settings.

For example, if a pixel is at x position 10, and both windows start later than x position 20, then it will be considered outside both windows. In this case, it will only appear on screen if the "Outside both windows" bit is set in the window settings. If it is set for the main screen but not the sub screen then that pixel will only appear on the main screen.

This can also be imagined as a Venn diagram. If you were to draw a Venn diagram using the windows of SASPPU, then each section can be assigned one of the bits of the window settings. Assuming the left circle is window A, then a sprite which is set to only appear in window A will only be visible in the part of the left circle which the right circle does not touch. If a sprite is set to appear in both "window A" and "window A and B", then it will appear for the whole of the left circle.

If you wish to ignore the window feature setting them to 0xF (1111) will cause a pixel to appear no matter what. Conversely, setting them to 0x0 (0000) will cause a pixel to never appear. By default windows are set to 0xF (1111).

Windows are by default rectangles, but by using HDMA you can change the shape of them each scanline. With this you can make circular windows, for example.

#### Windows Reference

### MainState

The MainState struct/class contains various global settings that apply to the entire image.

#### MainState reference

### Color Math

#### Color Math Reference

### HDMA

#### HDMA Reference

### Memories

SASPPU contains two graphics memories used for storing raw pixel data. There is one dedicated to backgrounds, and one dedicated to sprites. All backgrounds must share the background memory, and all sprites must share the sprite memory.

Each of these memories is made up from one massive 256x256 pixel image. Indexing into these memories is performed using an X and Y position, from 0 to 255 for each.

Each pixel of graphics memory is one SASPPU colour, or 16 bits.

To access a graphics memory, you can either access it directly as an array or use the helper functions for blitting pixels. If you are in micropython, you may only use the helper functions and cannot access the graphics memories directly.

As all backgrounds/sprites must share the memory, care must be taken to lay out graphics in such a way that images do not overlap and that they all have their own portion of the 256x256 memory.

Consider making a background repetitive, only storing one tile in graphics memory, and then using the tilemap to repeat it across. Tilemap can be also used to flip tiles, which can add variation If a graphic is symmetrical, consider only storing half of it and using the tilemap to create the other half.

#### Memories Reference



### Micropython "Binding"

In C, all registers are available static variables. This presents a challenge for microypython.

Instead, micropython uses a binding model. Any instance of a state, background or sprite may become synchronised with the underlying static version by calling `.bind`. By default, this will copy the current values into your version - if you wish to store the values you have you may call `.bind(True)`.

Once bound, all reads and writes to the class will be performed on the static versions of the class. Once finished, the class may be `unbound`, where afterwards its data is separate from the underlying static values.

You may also call `get_bind_point`, which returns whether a class is bound, and if applicable where.

Multiple instances of a class may be bound to the same place at the same time. In this case, a write to one will appear when reading the other.

Additionally, Backgrounds and Sprites need to choose where to bind. For Backgrounds this is either 0 or 1, and for Sprites this is between 0 and 255. 

#### Binding Reference

| C     | Micropython   | Summary |
| ----- | ------------- | ------- |
| N/A | MainState.bind() | Bind this MainState to the static one |
| N/A | MainState.unbind() | Unbind this MainState from the static one |
| N/A | MainState.get_bind_point() | Returns true if this MainState is bound |
| N/A | CMathState.bind() | Bind this CMathState to the static one |
| N/A | CMathState.unbind() | Unbind this CMathState from the static one |
| N/A | CMathState.get_bind_point() | Returns true if this CMathState is bound |
| N/A | Background.bind(0) | Bind this Background to the static background 0 |
| N/A | Background.bind(1) | Bind this Background to the static background 1 |
| N/A | Background.unbind() | Unbind this Background from the static one |
| N/A | Background.get_bind_point() | Returns either 0 or 1 if bound, or None otherwise |
| N/A | Sprite.bind(0) | Bind this Sprite to the static sprite at OAM index 0 |
| N/A | Sprite.bind(1) | Bind this Sprite to the static sprite at OAM index 1 |
| N/A | Sprite.bind(255) | Bind this Sprite to the static sprite at OAM index 255 |
| N/A | Sprite.unbind() | Unbind this Sprite from the static one |
| N/A | Sprite.get_bind_point() | Returns a number between 0 and 255 if bound, or None otherwise |