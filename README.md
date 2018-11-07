# colorblind-writeup
Writeup to UMDCTF's colorblind challenge

ColorBlind is a challenge that requires players to understand a little about PNG headers and chunks. The challenge is as follows:

# Colorblind
Help! I can't seem to see the flag...I'm colorblind. Are you?

* `Author` -- Towel
* File: https://github.com/0xTowel/UMDCTF-2018-Challenges/raw/master/Steganography/ColorBlind/colorblind.png

After downloading the file, I first tried to open it, but noticed that the file seemed corrupted in some way.

Let's first check if the file is actually a PNG by running the `file` command:

```
file colorblind.png
colorblind/colorblind.png: PNG image data, 1920 x 834, 4-bit colormap, non-interlaced
```

We notice that this is a PNG. Let's check that the headers and chunks are correct with `pngcheck`:

```
pngcheck colorblind.png
OK: colorblind/colorblind.png (1920x834, 4-bit palette, non-interlaced, -88.6%).
```

Looks good! So maybe the chunks aren't corrupted. Let's explore the first few chunks. Each chunk has the length of the chunk followed by the name of the chunk. The first chunk in PNG files, after the first 7 mandatory hardcoded bytes, is the IHDR chunk, which has the following fields:

```
Width:              4 bytes - 0x00000780  (1920)
Height:             4 bytes - 0x00000342  (834)
Bit depth:          1 byte  - 0x04
Color type:         1 byte  - 0x03
Compression method: 1 byte  - 0x00
Filter method:      1 byte  - 0x00
Interlace method:   1 byte  - 0x00
```
These line up with what we got from the `file` command. The next block is the PLTE chunk. Values in the PLTE chunk are RGB values, and are therefore 3 bytes each. For example, the first palette value is FF 80 00.

Let's assume that the palette is correct, but the fact that it is a palette image is not correct. There are several possibilities for what the color type could be:
```
   Color    Allowed    Interpretation
   Type    Bit Depths
   
   0       1,2,4,8,16  Each pixel is a grayscale sample.
   
   2       8,16        Each pixel is an R,G,B triple.
   
   3       1,2,4,8     Each pixel is a palette index;
                       a PLTE chunk must appear.
   
   4       8,16        Each pixel is a grayscale sample,
                       followed by an alpha sample.
   
   6       8,16        Each pixel is an R,G,B triple,
                       followed by an alpha sample.
```
Currently, the color type is set to 3. What if we set it to different values? I tried each, but let's assume I was smart and guessed 6 first. (As an aside, the PLTE header is not allowed for types 0 and 2, is required for type 3, and is optional for types 4 and 6). 

First, we overwrite the value for the color type to 0x06. However we also need to set the allowed bit depth, since 4 - the old value - is not permitted. Let's change it to 0x08, a permitted value. I used a simple hex editor to make these changes.

I then opened the picture, but got the following error: `Fatal error reading PNG image file: PLTE: CRC error`. Each block also requires the CRC (Cyclic Redundancy Check) to be correct. For each chunk, the CRC is found with the following formula: 

```
CRC32(Chunk Type || data)
```
So in this case, we need to compute the CRC for the following:
```
CRC32(49 48 44 52 00 00 07 80 00 00 03 42 04 03 00 00 00)
```
We can compute this with the `crc32` command, or use an online CRC32 calculator, which I did. Remember that PNG requires numbers to be in Big Endian, which gives `0x87cdd16a`. Then, we overwrite the old checksum with the new one. Let's make sure we di everything correctly with `pngcheck`:
```
pngcheck colorblind.png
OK: colocopy.png (1920x834, 32-bit RGB+alpha, non-interlaced, 76.4%).
```

Awesome! Let's open it up. We get the following:

![colocopy](https://user-images.githubusercontent.com/14065974/48111049-8f3f0100-e21d-11e8-9efc-e68b662b24ab.png)

Looks like we're not quite done yet. But hey, that thing on the bottom looks like a QR code, no? Let's extend and crop it:

![cropped](https://user-images.githubusercontent.com/14065974/48111097-d927e700-e21d-11e8-95f9-a786d0339781.png)

These colors appear to be incorrect - look up any QR code, and it looks like the colors have been inverted. We can fix this using Gimp. Open up the image in Gimp, go to the Colors Menu, and click "Invert". This gives:

![inverted](https://user-images.githubusercontent.com/14065974/48111163-3d4aab00-e21e-11e8-8048-6e2f728090cf.png)

We're close, but it isn't correct. The black squares should be at the bottom left, top left, and top right of the picture. Let's try rotating it 90 degrees to the left (I used paint):

![rotated](https://user-images.githubusercontent.com/14065974/48111215-88fd5480-e21e-11e8-8efc-09eff2dd4125.png)

Now we use a QR code reader and get:

`UMDCTF-Wh0_N3eded_Th4t_Color_StuFF_ANyW4y`

Yay!
