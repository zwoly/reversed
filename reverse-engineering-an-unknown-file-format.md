
# Reverse Engineering an Unknown File Format
 
> Disclaimer: the material presented in this text is purely for educational purposes.

I want to explain the steps I took to understand the structure of a file I am interested in. Here's the background: there's an online video game that uses a *custom file format* for its resources. I don't want to expose the game or the extension to not hurt that game in any way (without excluding the fact that those who are interested in reversing that particular game might realize what's the game), therefore I am going to just call it "game" and the extension of the file we're trying to reverse is "ext". Also, I'll assume that you've never done this before.

Let's start with basics. Trying to understand the file format mainly involves analyzing the file to find out its structure.

A file is a pointer to a section on your drive that is a sequence of bytes that define the file's metadata and its contents. The file *itself* is a bunch of bytes that define its contents. The metadata includes its name, type, size, date created/modified, path, etc.

And its contents can be represented in two forms: one is **encrypted**, another - **unencrypted**. An unecrypted file is a file that can be read by a human, for example, any text file with an extension *'.txt'*. An encrypted file, on the other hand, can either be opened by a specific software that can read it (decrypt it first, then show it), or if you open it using a text editor, it will just show some gibberish.

Now, there is a specific software that one can use to analyze it - a **hex editor**. It doesn't really matter which hex editor you're going to use, in our case we don't need anything fancy or feature-rich, even though it can be handy. In my case, I used a hex editor that happened to be feature-rich - [ImHex by WerWolv](https://github.com/werwolv/imhex), which is also free.

> Note: in no way I am associated with or promoting software that I mention in this article.

## Opening the file

Our first step is to open the file that we want to inspect in the hex editor of our choice. In this article, just for the sake of avoiding pictures, I am using my own `hexdump` program. The contents of my file that I see are as such:

```
-------------------------------------------------------------------------------
                              Hex dump of game.ext
-------------------------------------------------------------------------------
0000000000 | F2 1B 42 EB 01 25 01 01 1E 66 6F 6E 74 73 2F 73 | .←B..%..▲fonts/s
0000000016 | 66 70 72 6F 74 65 78 74 2D 62 6F 6C 64 69 74 61 | fprotext-boldita
0000000032 | 6C 69 63 2E 74 74 66 76 1B 7A ED 04 90 3F 04 00 | lic.ttfv←z...?..
0000000048 | 04 9A 6B 03 00 90 FF 10 F4 27 04 53 7D 9E 70 B6 | ..k......'.S}.p.
0000000064 | 79 8A 80 53 78 9E 70 F4 28 D8 C5 18 96 EA 51 B6 | y..Sx.p.(..↑..Q.
0000000080 | 6D B5 50 53 7C 9E B0 F0 2F DF CD D3 FD D7 0A B6 | m.PS|.../.......
0000000096 | 6D B5 34 53 7C 9E 6C F1 2D CE C6 58 5F 8F 50 B6 | m.4S|.l.-..X_.P.
0000000112 | 6A 86 00 53 7C 9E 5A F1 39 C4 D3 68 4A AE 33 B6 | j..S|.Z.9..hJ.3.
0000000128 | 6A A8 38 53 7D 84 8A F1 3A DE C2 DD 8E 75 73 B6 | j.8S}...:....us.
0000000144 | 6A 86 2C 53 7C 88 7A F9 3A A4 B2 25 38 01 4A B6 | j.,S|.z.:..%8.J.
0000000160 | 69 8A 08 53 7C 9E 10 D5 04 EA F0 76 FB 65 D8 B6 | i..S|......v.e..
0000000176 | 69 92 6C 53 7C 98 12 D1 08 F8 F0 AC 83 9E 73 B6 | i.lS|.........s.
0000000192 | 6A 86 F8 53 7C 9E 78 D1 05 F2 E6 64 D8 9D 3E B6 | j..S|.x....d..>.
```

The view here is made of three parts:
- **Left**: Offset (line numbering - each line has 16 hexadecimal values, so the offset increments by 16)
- **Center**: Data in hexadecimal format (instead of binary, just for the sake of readability)
- **Right**: ASCII representation of those hexadecimal values (the encoding can be changed - for example, to display Unicode characters)

If you don't have any programming skills, then it's going to be a bit hard to understand what's going on in here, so since this is not a programming tutorial, you should be assumed to have some basic skills, i.e. to know about data types (`char`, `int`, `float`, etc.).

Here, it's not particularly about those data types you see in languages like C or C++. But the concept of them applies here. How? Each hexadecimal value, such as the very first one in the code block above - `F2` - represents a single `byte`. A byte is a *"pack of bits"*: depending on the machine's architecture, it can represent the `N` number of bits. In our modern typical machines, one byte represents `8 bits`. The hexadecimal `F2` is a binary `11110010` (notice how there are 8 digits, each representing 1 bit).

However, in the view of this file it can get a bit tricky, since the **value** of our interest can be represented by *more than one byte*. And depending on how many bytes the value consists of, we use data types to specify their size. For example:

- `int8` for a single byte (Example: `F2`)
- `int16` for two bytes (Example: `F2 1B`)
- `int24` for three bytes (Example: `F2 1B 42`)
- `int32` for four bytes (Example: `F2 1B 42 EB`)
- `int48` for six bytes (Example: `F2 1B 42 EB 01 25`)
- `int64` for eight bytes (Example: `F2 1B 42 EB 01 25 01 01`)

And, by the way, if you opened the file in `ImHex`, you're going to be able to select those hexadecimal values. Clicking on one, you'll see clues on a side panel called *"Data Inspector"*, which represents that byte in all various ways listed in that panel, so it could be **only one** of those things. If you select the first byte `F2`, you'll see its 1-byte value under `uint8_t` or `int8_t`. This raises two questions: what's the difference between those two types; and what about those others - how can I see an `int16` value of a single byte.

- `int` stands for `integer` and is usually of a `signed` type. That means its value can have a sign `-`, so its value can range from a negative number to positive.
- `uint` stands for `unsigned integer`, which means it's a number that does not have a sign, so it ranges from 0 to positive.

When selecting one byte (one hexadecimal value, such as `F2`) and, for example, seeing `uint16_t` (which represents 2 bytes), it's going to show a value of the *current* selected byte and its `subsequent` byte. So, in this case, if we select the first byte `F2`, we're going to see the next:

- `uint8_t` - if `unsigned`, decimal value is `242` | hexadecimal value: `F2`
- `int8_t` - if `signed`, decimal value is `-14` | hexadecimal value: `F2`
- `uint16_t` - if `unsigned`, decimal value is `7154` | hexadecimal value: `F2 1B`
- `int16_t` - if `signed`, decimal value is `7154` | hexadecimal value: `F2 1B`
- `uint24_t` - if `unsigned`, decimal value is `4332530` | hexadecimal value: `F2 1B 42`
- `int24_t` - if `signed`, decimal value is `4332530` | hexadecimal value: `F2 1B 42`

As you can see, the `int8` shows the value for just `F2`, whereas `int16` for `F2 1B`; `int24` for `F2 1B 42`; `int32` for `F2 1B 42 EB`; and so on.

There are some "naming conventions" of values in different numeral systems: hexadecimal values (aka "hex values") usually start with `0x`, binary values start with `0b`, and decimal values... are just decimal values. So, `F2` in this case is also `0xF2`, and `F2 1B 42 EB` would be `0xF21B42EB`. **But!** There's one more thing - *endianness*.

The hex values inside of a file can be either in `big endian` or `little endian` format. Instead of explaining what it exactly means (since we assume you're somewhat familiar with programming), we will just put it simply: we pick a random decimal value for an example - `123456789`, whose hex value is `07 5B CD 15`:

- In `big endian` format, a value such as `07 5B CD 15` is going to be read as `0x075BCD15`
- In `little endian` format, a value such as `07 5B CD 15` is going to be read as `0x15CD5B07`

Notice the difference between `0x075BCD15` and `0x15CD5B07` - the bytes are just re-arranged: one is *"left-to-right"*, another - *"right-to-left"*.

These are the very basics of how to read the hex values. And now that we have the file open in our editor, we can inspect it to see *if we can make sense of these bytes*. But where to look for the clues? Within the right part of the view which shows ASCII representation of those hex values.

But, really, it's not just about that. The reason why I know where, what, and how, is because I know about file formats in general, so *I know what I can expect to see* when inspecting files. And some of that stuff I am going to lay down.

## Finding our first clues

The first clue happens to be just in front of us.

```
0000000000 | F2 1B 42 EB 01 25 01 01 1E 66 6F 6E 74 73 2F 73 | .←B..%..▲fonts/s
0000000016 | 66 70 72 6F 74 65 78 74 2D 62 6F 6C 64 69 74 61 | fprotext-boldita
0000000032 | 6C 69 63 2E 74 74 66 76 1B 7A ED 04 90 3F 04 00 | lic.ttfv←z...?..
```

If you pay attention to these first three lines, you will notice the next sequence of characters that reads like a string:

    `fonts/sfprotext-bolditalic.ttf`
    
Here's what I understand from this - it's a path to a specific file that is called `sfprotext-bolditalic.ttf` that resides in the `fonts` directory. A `ttf` extension is used for font files, and it's pretty much obvious since it's within a folder called `fonts`.

Our first clue hints us that there can be readable strings within this gibberish, so what we can do - we can try to identify all other strings within this file. It's possible to do it by guessing, i.e. if I have my hexdump, I can search for other occurrences of the word "font" or extension ".ttf". Or...  even better, we can utilize a nifty feature of `ImHex` that lets you find strings. For that, you need to click on the "Find" tab on the right panel (and if it's not there, you can toggle it through `View` -> `Find` on the top bar). Within the `Find` view, you want to make sure that in "Range" the "Entire Data" is selected, so we're scanning the entire file, instead of a particular selection or a region; then, we select the "Strings" tab where by default we have the minimum length set to `5`; by default the type is also ASCII; so, all we need to do is just hit "Search".

The output is a table with columns `Offset`, `Size`, and `Value`. Right now, we care about the value. And the first value it found was our string of size `31 bytes` (i.e. 31 hex values). If we select it, the hex and ASCII values in the hex editor will be highlighted. On the top of that table, there's a placeholder which allows us to filter the output. For example, we can type "font" and in the output it will give us 12 occurrences of a string that starts with the word "font". Cool! We can now be sure that the file we're dealing with is an **archive file**. Simply put, it's a file that contains many other files that you can unpack (like `RAR` or `Zip`). In our case, there are many font files, but there are also other files as well. We can find it out simply by looking for readable strings in the output table.

Tinkering with the string output, I found out that there are `37` files within this package:
- `3` *shader* files (`.hlsl`)
- `12` *font* files (`.ttf`)
- `22` *texture* files (`.dds`)

```
sprites/signin_hover.dds
sprites/checkbox_bg.dds
sprites/progress_fill.dds
--- Skipping 19 files ---
fonts/sfprotext-bolditalic.ttf
fonts/sfprotext-bold.ttf
fonts/sfprotext-heavy.ttf
--- Skipping 9 files ---
shaders/clip_region.hlsl
shaders/gaussian_blur.hlsl
shaders/text_glow.hlsl
```

## Demystifying bytes

We have found strings as our first clue about what is this file about. And what I've learned is that it contains resources, such as fonts, shaders and textures. And I figured (even before analyzing it) that it belongs to the game's launcher (more on this later).

Well, we found the obvious things. It seemed easy. But everything else is gibberish, so is there a way to understand what are all other bytes - what do those other hex values stand for? Sure. And we haven't just found the strings, we actually found two more things. But before I say what are those, let me list things a typical archive file can have as part of its structure:

- **File signature** - a file format usually starts with a `magic number`. It's a unique sequence of bytes that identify the file's format. For example, a `.exe` file starts with the "magic bytes" `4D 5A` (`MZ` in ASCII). This is handy when a program that reads the file tries to verify it before processing.;
- **Number of files** within the archive - the total number of entries (could be files; could be other data, such as configurations) in the package;
- **Filename** - the name of the entry/file, which could also include directories/subdirectories, i.e. a path;
- **Original size** - the original (uncompressed) size of the file;
- **Compressed size** - the compressed size of the file;
- **Last modification time**
- **Last modification date**
- **Cyclic redundancy check** - a CRC value that could be of any format (i.e. CRC-16, CRC-32, etc.) - a unique hash value that checks whether the data has not been changed.
- **Offset** - offset to where the actual data starts
- **Data** - the data/file itself (which, other than being compressed, can also be encrypted)

Now we (approximately) know what we can expect to find in the hex values. Sometimes I don't know where to start, or whether I should do things sequentially (i.e. start with the first byte). I am just clicking on every byte and look at their values. For example, I'll start guessing that the magic bytes would probably be the first two to four bytes; or that the number of files should be one byte (so I will look at the `int8` value); or that the original size should be three to four bytes (so I will look at either `int16` or `int24` value). For now, we identified the `filename`/`path`.

Previously, we found out that there are `37` strings within this package, so it's 37 files. Let's see if there's such a hex value that represents the decimal `37`. Or, instead of clicking on each byte, we can simply convert decimal `37` to hexadecimal, which is `0x25`. Looking at the first line we can indeed find such a value, and it's so specific, the chances it's what we think it is are pretty high:

```
F2 1B 42 EB 01 25 01 01 1E 66 6F 6E 74 73 2F 73
               ^
```

I took a screenshot of the hexdump and outlined the bytes I think I "demystified". There are some `0x01` bytes before and after `0x25`, which could be flags or dummy values or something else. But let's try to figure out the magic bytes. How would I go about doing that? Well, there are actually more than one of the files with the "ext" extension. So I am going to open several of those files in the hex editor and see which first bytes are matching and when they start to differ. Those that match are going to be the "magic bytes".

```
resource_file_01.ext | F2 1B 42 EB 01 25 01 01 | This file is located in the root directory with the game's executable
resource_file_02.ext | F2 1B 42 EB 02 34 12 01 |
resource_file_03.ext | F2 1B 42 EB 02 AB 03 01 | These files are located in the subdirectory `package`
resource_file_04.ext | F2 1B 42 EB 02 F3 01 01 |
resource_file_05.ext | F2 1B 42 EB 01 03 01 01 | This file is located in game's "My Documents" folder
```

What I see here is that the first four bytes are similar in all those files `F2 1B 42 EB`, so I am going to assume that this is the magic number I am looking for. Now, let's get to the "number of files" byte - `0x25`. Comparing it with other files (they are on the same column) - `0x34` `0xAB` `0xF3` `0x03`, which in decimal are `52`, `171`, `243`, `3` - and doing the same string scanning operation we did earlier, it did not confirm that all of the values match with the number of entries. We will come back to this a bit **later**.

What about the byte in between the magic number and the number of entries? It's either `01` or `02`. It looks more like a flag. First, my reasoning was that it's related to whether the file is in the root directory (`01`) or in a subdirectory (`02`), but then the `resource_file_05.ext` is neither of them; and it contains 3 entries, which are not files, but rather configurations that are related to the game's launcher. And I also mentioned that `resource_file_01.ext` is in the root directory with the game's executable, where's also the launcher. So it could be that `01` means it's related to the game's launcher, and `02` means it's game-related resources (item textures, sounds, fonts, etc.). But this is only a guessing. For now, I'll just move on to the next byte.

There are two `01` bytes, and they could be `int8` (so each byte is separate; and if it was 0x0101, it'd be `int16`). They could be flags, they could be a dummy value. But. This is what I'd assume if I was just looking into the first file. Seeing the beginning bytes in multiple files makes me see a better picture:

```
resource_file_01.ext | F2 1B 42 EB 01 25 01 01 1E 66 |
                                         *  |
resource_file_02.ext | F2 1B 42 EB 02 34 12 01 01 20 |
                                            |  *
resource_file_03.ext | F2 1B 42 EB 02 AB 03 01 01 1B |
                                            |  *
resource_file_04.ext | F2 1B 42 EB 02 F3 01 01 01 23 |
                                         *  |
resource_file_05.ext | F2 1B 42 EB 01 03 01 01 04 73 |
                                         ^^^^^^^^
```

In the first, fourth, and fifth files the two bytes have a value `0x0101`, while in second and third they differ - `0x0112` and `0x0103`. Again, those could be individual bytes, i.e. `int8` and `int8`, instead of one `int16`, so let's look into that. If it's an `uint16`, then the decimal values for each file would be: `257`, `274`, `259`, `257`, `257`. These values look weird to me. `66` is where the string starts, it's the first letter of the word `fonts` (`f` = `66`). Therefore, it's either 1, 2, or 3 bytes that we're dealing with here. What I also noticed is that `01` is consistent within all the files. Is it an individual byte or is related to its neighboring bytes?

Mostly, the hints are in numbers. Such as, when you count the number of entries, or the number of bytes the data itself is. What about the number of characters in the string? Knowing that sometimes a string can have a byte in front of it that indicates its length, I started counting the bytes. In the first file, the length of the string (the number of bytes I counted) is `30`. Therefore, there must be a byte that holds the decimal value `30` in a hexadecimal format. `30` in hexadecimal is `1E`. Perfect! There is a such a value and it goes right before where the string starts. What about other files? Let's see: in the second file I noticed one thing - the first byte of the filename string is *not* `0x20`, it's `0x69`. So let's extend our illustration with one more byte:

```
resource_file_01.ext | F2 1B 42 EB 01 25 01 01 1E 66 6F |
                                                  ^
resource_file_02.ext | F2 1B 42 EB 02 34 12 01 01 20 69 |
                                                     ^
resource_file_03.ext | F2 1B 42 EB 02 AB 03 01 01 1B 65 |
                                                     ^
resource_file_04.ext | F2 1B 42 EB 02 F3 01 01 01 23 6D |
                                                     ^
resource_file_05.ext | F2 1B 42 EB 01 03 01 01 04 73 61 |
                                                  ^
```

The arrows indicate where the string starts. And counting the length of the string I confirmed that the previous byte in each entry refers to the length of the string (`0x1E`, `0x20`, `0x1B`, `0x23`, `0x73`). Hmm. `resource_file_01.ext` is kind of similar to `resource_file_05.ext`. They both "miss" one extra byte.

```
                       ___________ -- __ ?? ?? ~~ __ __
resource_file_01.ext | F2 1B 42 EB 01 25 01 01 1E 66 6F

                       ___________ -- __ ?? ?? ?? ~~ __
resource_file_02.ext | F2 1B 42 EB 02 34 12 01 01 20 69 |
```

Here, we can clearly see that there's `01` value that is indeed consistent, even though this is just my assumptions, but we can - just in case - offset the first and last file by one byte.

```
resource_file_01.ext | F2 1B 42 EB 01 25    01 01 1E 66 6F |
                                      ***** ^^^^^ --------->
resource_file_02.ext | F2 1B 42 EB 02 34 12 01 01 20 69 |
                                      ***** ^^^^^ --------->
resource_file_03.ext | F2 1B 42 EB 02 AB 03 01 01 1B 65 |
                                      ***** ^^^^^ --------->
resource_file_04.ext | F2 1B 42 EB 02 F3 01 01 01 23 6D |
                                      ***** ^^^^^ --------->
resource_file_05.ext | F2 1B 42 EB 01 03    01 01 04 73 61 |
                                      ***** ^^^^^ --------->
```

This made me realize that the `0x01` and `0x02` could be telling the following number of bytes.

## Tricky bytes

[Flashback]: *"We will come back to this a bit later."* This is later. What if the `0x25` byte is `int8`, but `34 12` is `int16`? Let's see the `int16` value of `0x1234` - it's `4660` in decimal. If we think it is what it is, that means there are 4660 entries. The number looks really big. But. The *size of the file* (which is `~177 MB`) also hints at the possibility that there are tons of files. To confirm that number, we sadly have to find all of the strings, copy paste them into a text editor that shows lines, then scroll to the very bottom to read the last line number. At least, that's how I did it. And here are my exact steps:

- I opened the `resource_file_02.ext` file in `ImHex`, then in the `Find` tab I specified the `Minimum length` to be `10`, and hit the `Search` button;
- It found `797114` entries, which I am not going to read one-by-one. I am going to scroll to see what kind of file extensions I might notice;
- For example, I saw a string that ends with `.png`. So I type in the filter `.png`, and now I see all of the strings that end with the extension `.png`;
- There's a small button on the right from the filter placeholder which has an arrow icon. Click it to save the filtered output to a file.
- Scrolling through the "unfiltered" output, I also find other extensions: `.seq`, `.dds`, `.ttf`, `.jpg`, `.scn`.
- We can also search by the name of the root directory, such as "gui", "image", etc.
- After we think we found all the strings, we combine them together and read the number of the last line.

```
8128 .png files
 523 .dds files
 179 .seq files
   2 .ttf files
  40 .jpg files
 141 .scn files
---------------
Total:     9013
```

It doesn't look anything like 4660. Let's look at the `34 12 01`, i.e. `0x011234` (little endian), and... it's too big - `70196`. Let's make it easier for us and convert `9013` to hexadecimal (assuming I didn't miss any files). It's `0x2335`, i.e. we want to find two bytes `35 23`. I found it, but it's not in the beginning of the file. I tried to do the same on another random file that is much smaller and has less strings - `resource_file_22.ext`, where the two bytes `2B 03` match the number of entries.

```
___________ -- xx xx -- -- __ __
F2 1B 42 EB 02 2B 03 01 01 2D 6D
               ^^^^^
               813 entries
```

Does this mean that `4660` is actually a correct number? That'd be really weird since we've found 9013 entries! Well, if I feel like I reached a dead end here, I simply continue with other bytes. But this one really bugs me, so what I tried to do is *sort the entries* in my text editor. I sorted them alphabetically and started scrolling to see if there's anything weird. And this is what I saw:

```
image/costume/24_male_hand_2.png
image/costume/24_male_hand_2.png
image/costume/24_male_leg_2.png
image/costume/24_male_leg_2.png
image/costume/25_female_body.png
image/costume/25_female_body.png
image/costume/25_female_body_1.png
```

Duplicate data! That now explains why it doesn't match the value we're looking for - `4660`. We could divide `9013` by `2` and it would result in `4,506.5`. This is closer to the truth, as I most likely missed some strings, and if we assume that each entry has a duplicate. Could it be that something went wrong when packing this file? If so, then we just found a bugged file that "thinks" it has the correct number of entries.

## The wasteland

When I open a file for inspection and I can find some readable sequence of ASCII characters, I can try to work around it to try to understand what it's about. I could "easily" (well, it did take some time) understand the sequence of bytes that went before the string, which was the first clue. What goes after - could be anything, and arranged in any way. That part I call "wasteland". There's nothing much else to understand, nothing is readable, and it's mostly gibberish, and we're left guessing whether the next value is an individual byte or a seuqnce of bytes. But, well, things aren't that complicated with this one, so let's continue digging.

The next possible bytes could be *file size* or *compressed size* (since it's an *archive file*, but we're just assuming). How to know whether the value that we're reading is the size of the file? Well, this particular case makes things very easy for us: what I did is - while I can't download the assets, I can find the font file on the web! So, I found the `sfprotext-bolditalic.ttf`, and found out its size. You can do it by opening the Properties, where you can look at the `Size` information (not `Size on disk`), where it should be `278,416 bytes` (note that we want to know the bytes, not KB or MB). However, just for this, I wrote a small simple character counter program `cc` which can count either ASCII characters or bytes:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define FILE_MODE_READ_ASCII  "r"
#define FILE_MODE_READ_BINARY "rb"

static inline
unsigned int ret_number_of_characters(char* filename, char* filemode)
{
    FILE* file = NULL;

    file = fopen(filename, filemode);

    if (file == NULL)
    {
        fprintf(stderr, "\n[ ERROR ]: Could not open '%s'.\n"
                "Check if it exists.\n", filename);
        exit(EXIT_FAILURE);
    }
    
    unsigned int number_of_characters = 0;

    int cc = 0;

    while(cc != EOF)
    {
        cc = fgetc(file);

        number_of_characters += 1;
    }

    fclose(file);

    if (file) file = NULL;

    return number_of_characters;
}

int main(int argc, char* argv[])
{
    if (argc != 3)
    {
    USAGE_INFO:
        puts(
            "\nUsage: cc [reading mode] [filename]\n"
            "\nReading modes:\n"
            "    -t - text\n"
            "    -b - binary\n"
            "\nExample: cc -t my_file.txt");
        exit(EXIT_SUCCESS);
    }

    unsigned int number_of_characters = 0;

    if (!strcmp(argv[1], "-t"))
    {
        number_of_characters = ret_number_of_characters(
            argv[2], FILE_MODE_READ_ASCII);
    }
    else if (!strcmp(argv[1], "-b"))
    {
        number_of_characters = ret_number_of_characters(
            argv[2], FILE_MODE_READ_BINARY);
    }
    else
    {
        fprintf(stderr, "\n[ ERROR ]: No such argument is supported.\n");
        goto USAGE_INFO;
    }
    
    printf("\nNumber of characters in '%s': %d\n",
           argv[2], number_of_characters);

    return 0;
}
```

Now we're going to look for a value `278416`. We can convert it to a hex value - `43F90` or `0x043F90`. In `ImHex` we can use `Ctrl+F` to open the search dialogue, make sure the `Hex` tab is selected, and type `903F04` (remember, it's little endian, so `04 3F 90` changes to `90 3F 04`).

And, we found it:

```
F2 1B 42 EB 01 25 01 01  1E 66 6F 6E 74 73 2F 73
66 70 72 6F 74 65 78 74  2D 62 6F 6C 64 69 74 61
6C 69 63 2E 74 74 66 76  1B 7A ED 04 90 3F 04 00
                                     ^^ ^^ ^^
04 9A 6B 03 00 90 FF 10  F4 27 04 53 7D 9E 70 B6
```

This is a perfect match, but there are bytes in between the `file size` and `filename`. And the following byte is `0x00`. Looks like some sort of a `NUL` terminator. Or maybe it's a flag. Maybe it's just a separator. But other values don't have such a separator, so it would be weird to have it here. Let's compare it against other entries within the same file; and I will ignore the previously uncovered bytes, so I'll just include only those that come right after the string:

```
               file size
               __ __ __
76 1B 7A ED 04 90 3F 04 00 04 9A 6B 03 00 90 FF 10 F4 27 04
            |           |  |        *  |           |
C4 C5 54 CC 04 DC F8 06 00 04 86 9F 04 00 DC F1 1B F4 5E 01
            |           |  |        *  |           |
4E B9 BC 76 04 70 F9 06 00 04 98 DD 04 00 F0 F2 1B F4 3D 08
            |           |  |        *  |           |
3D E7 C1 74 04 70 36 04 00 04 C0 8D 03 00 F0 EC 10 F4 27 04
```

There are 5 bytes that come before `file size`. And one thing that I noticed is that there are matching bytes `04`, `00`, `04`, `00`, `F4`; but also one column has either `03` or `04`. How to understand whether the files are compressed? We can simply click on nearby bytes to see whether there's a value that is slightly smaller than the original file size. But this is not perfect, because the compression can be high and the value can be twice small, or even much smaller than that. But we're also not sure whether all of the files within the package are compressed as well. Even though it could be quiet obvious, we want to be *explicit* here.

We know the package's (archive file's) size, which is `5,5 MB`. That means that all the files combined should be less than `5,5 MB`. We can count it approximately, i.e. assuming that since all fonts are of the similar family, they probably are *slightly* different in size. But, let's say even if all of them are around `250 KB`, and we have `12` font files, then 12 * 250 = `3 MB`, plus `22` texutre files that - when combined - are expected to be heavier. One of the `.dds` textures called `bg.dds` is `7 MB`. And that exceeds the package's size. Therefore, the files within it are compressed. Of course, we're also not excluding the possibility that someone can decide to put a flag that indicates whether the data is compressed or not. But we will find that out as well, shortly after we find the compressed value.

Why am I so sure that there can be a compressed value at all? Well, I am not sure. Or, I wasn't sure when I was analyzing the file the first time. But then it makes sense because you can use the compressed file's size in order to find out where its data ends within the archive file.

## Pattern recognition

The wasteland can be confusing, but one core aspect of analyzing binary files is being able to see the patterns. It's possible by comparing files or data within the file you're inspecting. Let's take the previous block of bytes, where each line represents different entries within the same file, and the first byte is the byte right after the filename:

```
.. 76 1B 7A ED 04 90 3F 04 00 04 9A 6B 03 00 90 FF 10 F4 27 04 .. [bolditalic]
.. C4 C5 54 CC 04 DC F8 06 00 04 86 9F 04 00 DC F1 1B F4 5E 01 .. [bold]
.. 4E B9 BC 76 04 70 F9 06 00 04 98 DD 04 00 F0 F2 1B F4 3D 08 .. [heavy]
.. 3D E7 C1 74 04 70 36 04 00 04 C0 8D 03 00 F0 EC 10 F4 27 04 .. [mediumitalic]
               ~~ ^^ ^^ ^^ -- ~~          --       xx xx
```

Notice how there's a mysterious `0x04` byte before our guessed `file size`, and after it there's `00`, and then another `0x04`, and after 3 bytes it's again `00`. That just makes me think "there gotta be something in those 3 bytes" (`9A 6B 03`). What if it's what we're looking for - the compressed file size? Let's see the value `0x036B9A`; and it's `224154` - slightly less than the original `278416`. Looks legit, but we want to be sure. I did the same with the `.hlsl` entries from the same file:

```
                           _____    _____             probably where the data starts
filename -> D4 31 71 CA 02 27 03 02 D5 02 A7 06 F0 4C 29 09 C4 EF D5 52
                        |        |              |  |
filename -> 8C F4 97 5D 02 16 09 02 F6 06 96 12 F0 4C FB 7D 80 24 1D 17
                        |        |              |  |
filename -> 1E 95 8B 39 02 62 06 02 A2 05 E2 0C F0 4C 27 62 DC 06 1E 33
```

Hmm. Here, the sizes are separated with `0x02` instead of `0x04`. This made me instantly realize that it's most likely about the following value being 2 or 4 bytes long. Plus, I know that the value of a type `.hlsl` file should be 2 bytes or less. That's because `.hlsl` files are shader files that are used in games programmed with Microsoft's DirectX graphics API. Having worked with them myslef I know that the files are usually less than `10 KB` in size. And these happened to be a no exception (~1.5 KB). The compressed values are as well slightly less than the original which makes me further believe those bytes represent the compressed file size.

## The data

There are still 4 bytes before the original file size and bytes after the compressed file size:

```
               original       compressed        sus portion    probably where the data starts
               ___________    ___________       ___________
76 1B 7A ED 04 90 3F 04 00 04 9A 6B 03 00 90 FF 10 F4 27 04 -> 53 7D 9E 70 B6 79 8A 80 53 78   [bolditalic]
            |  ^^^^^^^^^^^ |                    *  |  *  *
            |  size        |                    *  |  *  *
            |              |                    *  |  *  *
C4 C5 54 CC 04 DC F8 06 00 04 86 9F 04 00 DC F1 1B F4 5E 01 -> 37 7B 94 52 7D A5 BD 1A 37 7E   [bold]
            |  ^^^^^^^^^^^ |                    *  |  *  *
            |  size        |                    *  |  *  *
            |              |                    *  |  *  *
4E B9 BC 76 04 70 F9 06 00 04 98 DD 04 00 F0 F2 1B F4 3D 08 -> 7B 11 35 41 27 13 36 6A 7B 14   [heavy]
            |  ^^^^^^^^^^^ |              |     *  |  *  *
            |  size        |              |     *  |  *  *
            |              |              |     *  |  *  *
3D E7 C1 74 04 70 36 04 00 04 C0 8D 03 00 F0 EC 10 F4 27 04 -> C7 70 3A 73 9C 19 62 39 C7 75   [mediumitalic]
```

I outlined the bytes that look suspicious. Can we make sense out of them? There are also two bytes in between the `compressed` and `sus portion`. What if it's file's creation date or what else could it be? This is where I decided to make another small program `ov.exe` which will help me find out where the actual data starts. But... how? By trying to figure out where the data ends! There are few things that we already - allegedly - know:

- Compressed file size
- We know approximately where the next entry starts (we can scroll until we see the next entry, which is at least the start of the filename).

And this is what the program does:

```
   0     1     2     3     4     5     6     7     8     9     10
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+------+
|  03 |  01 |  04 |  73 |  61 |  1B |  DF |  04 |  12 |  3A |  02  |
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+------+
            -------------------           -------------------
FBEGIN       bytes_before_data   Size: 2   bytes_after_data     FEND

offset_start      = FBEGIN + offset
bytes_before_data = offset_start - 10
bytes_after_data  =  offset_start + data_size
bytes_after_data  += 10
```

Basically, it skips the `N` amount of bytes and outputs several bytes that go before and after. What we're skipping is the assumed data, i.e. the compressed file. But before we proceed with this, we want to inspect the bytes of other entries in the same file, exactly the bytes that come before the filename:

```
_________________        @  f  o  n  t  s
F2 1B 42 EB 01 25 01 01 1E 66 6F 6E 74 73
                  |  |  *
                  |  |  *  f  o  n  t  s
00 00 00 01 00 00 01 01 18 66 6F 6E 74 73
                  |  |  *
                  |  |  *  f  o  n  t  s
63 E9 FF AE 0D 03 01 01 19 66 6F 6E 74 73
                  |  |  *
                  |  |  *  f  o  n  t  s
0C 1D 1D 1B 19 36 01 01 20 66 6F 6E 74 73
```

All of the entries have the `01 01` bytes before the `string length` byte. Maybe those simply indicate whether the data ended. But maybe it's part of the data (which would be weird that different files end exactly in same bytes). So I checked other entries and they also end with `01 01`. The same `01 01` can also be found at the beginning of the file, after magic bytes and number of files. That makes me think it could be that it indicates the *start of the entry*. If so, I am going to assume that the end of the file is the byte before the first `01` byte.

And here's the program:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

static inline
void view_offset(char* filename, unsigned int size_of_data, unsigned int offset_start)
{
    FILE* file = NULL;

    file = fopen(filename, "rb");

    if (file == NULL)
    {
        fprintf(stderr, "\n[ ERROR ]: Could not open '%s'.\n"
                "Check if it exists.\n", filename);
        exit(EXIT_FAILURE);
    }
    rewind(file);

    unsigned int file_size = 0;

    int cc = 0;

    while(cc != EOF)
    {
        cc = fgetc(file);

        file_size += 1;
    }

    printf("\n         Skipping: %d bytes\n", size_of_data);

    /* ==========================================================
     *
     * 00 00 00 00 00 00 00 00 - [data] - 00 00 00 00 00 00 00 00
     *            ^                ^                 ^
     *    bytes_before_data    size_of_data     bytes_after_data
     *                        (bytes to skip)
     *
     * ==========================================================
     */

    char bytes_before_data[32];
    memset(bytes_before_data, 0, sizeof(bytes_before_data));

    char bytes_after_data[32];
    memset(bytes_after_data, 0, sizeof(bytes_after_data));
    
    char* file_buffer = (char*)calloc(file_size, sizeof(char));

    if (file_buffer == NULL)
    {
        fprintf(stderr, "\n[ ERROR ]: Failed to allocate memory.\n");
        exit(EXIT_FAILURE);
    }

    fseek(file, 0, SEEK_SET);

    fread(file_buffer, 1, file_size, file);;
    
    for(size_t i = ((offset_start - 10) & 0x80000000) ? 0 : (offset_start - 10); i < offset_start; ++i)
    {
        char temp[4];
        memset(temp, 0, sizeof(char));
        sprintf(temp, "%02hhX ", (unsigned char)(file_buffer[i] & 0xff));
        strcat(bytes_before_data, temp);
    }
    printf("\nBytes before data: %s\n", bytes_before_data);

    for(size_t i = (offset_start + size_of_data); i < (offset_start + size_of_data) + 10; ++i)
    {
        char temp[4];
        memset(temp, 0, sizeof(char));
        sprintf(temp, "%02hhX ", (unsigned char)(file_buffer[i] & 0xff));
        strcat(bytes_after_data, temp);
    }
    printf(" Bytes after data: %s\n", bytes_after_data);

    printf(
        "\nPrettier view:"
        "\n\n    %s- [data] - %s\n", bytes_before_data, bytes_after_data);

    if (file_buffer)
    {
        free(file_buffer);
        file_buffer = NULL;
    }

    fclose(file);

    if (file) file = NULL;
}

unsigned char is_positive_integer(char* str)
{
    while (*str)
    {
        if (!isdigit(*str))
        {
            return 0;
        }
        str++;
    }

    return 1;
}

int main(int argc, char* argv[])
{
    if (argc != 4)
    {
        USAGE_INFO:
        puts("\nUsage: ov [filename] [bytes to skip] [offset]"
            "\n"
            "\n[filename]      - filename with extension, e.g. 'test.txt'"
            "\n[bytes to skip] - (uint) size of the data in bytes"
            "\n[offset]        - (uint) offset from the beginning of the file");
        exit(EXIT_SUCCESS);
    }

    if (is_positive_integer(argv[2]) == 0 || is_positive_integer(argv[3]) == 0)
    {
        puts("\nError: only positive integer values are allowed.");
        goto USAGE_INFO;
    }

    /* Arg_1: filename
     * Arg_2: size of the data in bytes
     * Arg_3: offset from the beginning of the file
     */
    view_offset(argv[1], atoi(argv[2]), atoi(argv[3]));

    return 0;
}
```

This one takes 3 arguments - the filename of the file that we want to inspect; the size of the data (we are going to specify the `compressed size`); and the offset from the beginning of the file. We're going to play around with the offset here - whether we want to start it from what we assume where the data starts or from somewhere else.

```
--------------------------------------------------------------------------
                            Offset Viewer
--------------------------------------------------------------------------

         Skipping: 224154 bytes

Bytes before data: 04 90 3F 04 00 04 9A 6B 03 00
 Bytes after data: 01 01 18 66 6F 6E 74 73 2F 73

Prettier view:

    04 90 3F 04 00 04 9A 6B 03 00 - [data] - 01 01 18 66 6F 6E 74 73 2F 73

--------------------------------------------------------------------------
```

Let's look at the right side and the left side of the skipped data. On the left side we see the `01 01` bytes that presumably indicate the start of another entry. What about the right side - the data starts after the `00` byte. And before that byte is the... value for the compressed size. It seems like that the data doesn't start two bytes after, and the things we assumed here:

```
               original       compressed        sus portion    probably where the data starts
               ___________    ___________       ___________
76 1B 7A ED 04 90 3F 04 00 04 9A 6B 03 00 90 FF 10 F4 27 04 -> 53 7D 9E 70 B6 79 8A 80 53 78   [bolditalic]
```

are just a wrong guess, and that `sus portion` is part of the compressed data (the `.ttf` file in this case). Also, after using the "ov" program to skip through the data of the last entry, it ended exactly at the end of the file. That means there's no additional data.

## The final bytes

It seems like we are left with 4 bytes that go right after the `filename`:

```
.. 76 1B 7A ED | 04 90 3F 04 00 04 9A 6B 03 00 90 FF 10 F4 27 04 .. [bolditalic]
.. C4 C5 54 CC | 04 DC F8 06 00 04 86 9F 04 00 DC F1 1B F4 5E 01 .. [bold]
.. 4E B9 BC 76 | 04 70 F9 06 00 04 98 DD 04 00 F0 F2 1B F4 3D 08 .. [heavy]
.. 3D E7 C1 74 | 04 70 36 04 00 04 C0 8D 03 00 F0 EC 10 F4 27 04 .. [mediumitalic]
   ^^ ^^ ^^ ^^
```

They all look unique and I can't make sense of individual bytes. Therefore, it made me instantly assume that this could be a CRC value, particularly CRC-32 since there are 4 bytes and it's a 4-byte version of it. And, guess what? There's a way to check it. And this is how I did it: I copied the bytes of the compressed data and pasted in "https://crccalc.com/", chose 'HEX' as input and hit "CRC-32", which on the table below showed the value '0xED7A1B76' under the "Result" column. And it exactly matches the 4-byte value of the entry - '76 1B 7A ED'. Later I learned that I could calculate the CRC using `ImHex`...

## The complete structure of the file

![file-format](https://github.com/zwoly/reversed/assets/164810717/5363c2cb-c6ba-40b6-bde6-9c34d4f79533)



## Doing the same, but documenting in real-time

What I did previously is write down the way I reversed the file format weeks before I actually decided to write this article. I decided to write this part in "real-time", i.e. I am documenting my thoughts while analyzing the same file format, which seems like has a slightly different structure for configuration files (as opposed to resource files).

### The part where I'm confused

Recall the `resource_file_05.ext`, which contains 3 entries only. It seems to be a configuration file that is used by the game's launcher. The 3 entries there are: `save`, `exclusive_fullscreen`, `hwid`. Now, it makes me think it's related to whether the `save` and `exclusive_fullscreen` checkboxes are selected; and the `hwid` could store my PC's unique hardware id string and might be checking whether I'm banned or not. These are just my thoughts, not what it actually is. Since those are not files, they don't have the `original size` and `compressed size` attributes. But do they still comply to the structure? Let's see:

```
___________ __ __ _____  __ ___________ ?? ?? ??
F2 1B 42 EB 01 03 01 01  04 73 61 76 65 1B DF 05 | .........save...
A5 01 01 01 03 01 00 01  01 01 14 65 78 63 6C 75 | ...........exclu
73 69 76 65 5F 66 75 6C  6C 73 63 72 65 65 6E 1B | sive_fullscreen.
DF 05 A5 01 01 01 03 01  00 01 01 01 04 68 77 69 | .............hwi
...
```

The beginning bytes seem to match with other files, however I'm not sure what comes after the string, as it contains many `01` values. I noticed that bytes after `save` and `exclusive_fullscreen` strings look similar, so I did some byte re-arrangement:

```
    F2 1B 42 EB 01 03 01 01 04 73 61 76 65 | .........save
+-> 1B DF 05 A5 01 01 01 03 01 00 01       | ...........
|
|   01 01 14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E | ..exclusive_fullscreen
+-> 1B DF 05 A5 01 01 01 03 01 00 01                                     | ...........
```

But the bytes after `hwid` look completely different:

```
               __    __    __
1B DF 05 A5 01 01 01 03 01 00 01
            |  *  |  *  *  *  *
1B DF 05 A5 01 01 01 03 01 00 01                        0x20 = 32 = size of the following value
            |  *  |  *  *  _______________________________________________________________________________________________
7B 74 4C 04 01 20 01 22 20 ..                                                                                              7D
```

If that last `0x20` means the length of the following string (which leaves out one extra byte at the end), then maybe the `01` on top could also mean the amount of bytes. But what about `0x22`? What about another `0x20`? Now here's my reasoning of what these bytes could be: since they are related to checkboxes, then there must be boolean values for whether they are checked (`01`) or not (`00`).

```
s     v    s     v    s     v    ?
01 -> 01 | 01 -> 03 | 01 -> 00 | 01 ..

s     v    s     v    s?
01 -> 20 | 01 -> 22 | 22 ..

s  - size
v  - value
?  - what the heck
s? - size?
```

There's only one way to find out whether my assumptions are correct - play around with the launcher. I moved away the file `resource_file_05.ext`, which we now will call `config_file_01.ext`, then ran the launcher. Before the window opened, I saw the `config_file_01.ext` being automatically created in the same folder. However, upon inspection, I only saw the `hwid` entry, without `save` or `exclusive_fullscreen` entries. This is weird, because it's not like those entries are included and some value is just simply set to `0x00`. No, they are not there *at all*.

Both checkboxes are unchecked. I decided to first check the `Exclusive Fullscreen` and then inspect the `config_file_01.ext` file. Now it has `exclusive_fullscreen` and `hwid` entries, without the `save` entry. But checking `Save` (which is for saving credentials such as Email+Password) writes all 3 entries. The bytes remain unchanged. This really makes me to give up, but cases like this - what makes me want to do it even more.

Oh, wait. I just noticed something:

```
[earlier version]
F2 1B 42 EB 01 03 01 01  04 73 61 76 65 1B DF 05 | .........save...
A5 01 01 01 03 01 00 01  01 01 14 65 78 63 6C 75 | ...........exclu
73 69 76 65 5F 66 75 6C  6C 73 63 72 65 65 6E 1B | sive_fullscreen.
DF 05 A5 01 01 01 03 01  00 01 01 01 04 68 77 69 | .............hwi
...

[recent version]
F2 1B 42 EB 01 03 01 01  04 73 61 76 65 00 00 00 | .........save...
00 01 00 01 01 00 01 01  14 65 78 63 6C 75 73 69 | .........exclusi
76 65 5F 66 75 6C 6C 73  63 72 65 65 6E 1B DF 05 | ve_fullscreen...
A5 01 01 01 03 01 00 01  01 01 04 68 77 69 64 7B | ...........hwid.
...
```

If you pay attention, you might notice how the `exclu` part moved left by two bytes. Two bytes missing?

```
                            (where the string starts)
[earlier version]           @
F2 1B 42 EB 01 03 01 01  04 73 61 76 65 1B DF 05 | .........save...
   _____ _____ ?? ?? ??  xx xx __ @     ^^ ^^ ^^
A5 01 01 01 03 01 00 01  01 01 14 65 78 63 6C 75 | ...........exclu
^^
73 69 76 65 5F 66 75 6C  6C 73 63 72 65 65 6E 1B | sive_fullscreen.
DF 05 A5 01 01 01 03 01  00 01 01 01 04 68 77 69 | .............hwi
...

[recent version]            @
F2 1B 42 EB 01 03 01 01  04 73 61 76 65 00 00 00 | .........save...
   _____ _____ ?? xx xx  __ @           ^^ ^^ ^^
00 01 00 01 01 00 01 01  14 65 78 63 6C 75 73 69 | .........exclusi
^^
76 65 5F 66 75 6C 6C 73  63 72 65 65 6E 1B DF 05 | ve_fullscreen...
A5 01 01 01 03 01 00 01  01 01 04 68 77 69 64 7B | ...........hwid.
...

@  - where the string starts
_  - paired values (sizeof_next_value -> value)
xx - beginning of this entry
?? - unknown byte
```

OK, there are quiet few differences in `earlier version` vs `recent version`:
- `1B DF 05 A5` vs `00 00 00 00`
- `01 01` `01 03` vs `01 00` `01 01`
- `01 00 01` vs `00`

Everything else is the same. The first gives CRC vibes, but what value does it hash? Then, `0x01` has changed to `0x00`; and `0x03` has changed to `0x01`. Unless... it didn't. We're assuming that the missing bytes are *at the end* (`01 00 01`). But what if missing bytes are `01 03`? We can find that out if we re-align the bytes:

```
F2 1B 42 EB 01 03 01 01 04 73 61 76 65 1B DF 05 A5 01 01 01 00 01 01 01 14 65 78 63 6C 75  | Removed '01 03'
                           @           |  |  |  |  #     #     ?? |  |     @
F2 1B 42 EB 01 03 01 01 04 73 61 76 65 00 00 00 00 01 00 01 01 00 01 01 14 65 78 63 6C 75
```

And this one looks weird because of the "extra" byte `0x01`/`0x00`. All of the values *change* and the bytes that we assume are the "size of the next value" remain *unchaged*. That at least tells us something. But, still, we need to dig deeper.

### Digging deeper

I am going to play around with the launcher again:

```
[config_file_01.ext - all boxes unchecked] - all 3 entries are presented
F2 1B 42 EB 01 03 01 01 04 73 61 76 65 00 00 00 00 01 00 01 01 00
                  01 01 14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E 1B DF 05 A5 01 01 01 03 01 00 01

[config_file_01.ext - only 'exclusive_fullscreen' checked] - only "exclusive_fullscreen" and "hwid" are presented
F2 1B 42 EB 01 02 01 01 14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E 1B DF 05 A5 01 01 01 03 01 00 01

[config_file_01.ext - 'exclusive_fullscreen' + 'save' checked] - all 3 entries are presented
F2 1B 42 EB 01 03 01 01 04 73 61 76 65 1B DF 05 A5 01 01 01 03 01 00 01
                  01 01 14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E 1B DF 05 A5 01 01 01 03 01 00 01

[config_file_01.ext - only 'save' checked] - only "save" and "hwid" are presented
F2 1B 42 EB 01 02 01 01 04 73 61 76 65 1B DF 05 A5 01 01 01 03 01 00 01
```

After combining lines:

```
                           s  a  v  e                            __ __ __ __
F2 1B 42 EB 01 03 01 01 04 73 61 76 65                         | 00 00 00 00 | 01 00 01    01 00 
F2 1B 42 EB 01 03 01 01 04 73 61 76 65                         | 1B DF 05 A5 | 01 01 01 03 01 00 01
F2 1B 42 EB 01 02 01 01 04 73 61 76 65                         | 1B DF 05 A5 | 01 01 01 03 01 00 01
                                                                               ** **

   e  x  c  l  u  s  i  v  e  _  f  u  l  l  s  c  r  e  e  n    __ __ __ __
14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E | 1B DF 05 A5 | 01 01 01 03 01 00 01
14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E | 1B DF 05 A5 | 01 01 01 03 01 00 01
                                                                 ^^ ^^ ^^ ^^   ** **
```

On the first line, I tried to align `01 00` to match the pattern. But then it looks weird that it has `0x01` that is not followed with any byte. The bytes marked with `*` - if I am not wrong - indicate whether the checkbox is checked. This leaves us with the mysterious bytes, which can be arranged in one of the following ways:

```
Version A                  Version B
+-------+-------+----+     +----+----+----------+
| 01 03 | 01 00 | 01 |     | 01 | 03 | 01 00 01 |
+-------+-------+----+     +----+----+----------+
u16     u16     u8         u8   u8   u24 (decimal value = 65537)

             Pattern A                  Pattern B                  Pattern C                  Pattern D
Old File     01 00 01 01 00             01 00 01    01 00          01 00       01 01 00             01 00 01 01 00      
New File     01 01 01 03 01 00 01       01 01 01 03 01 00 01       01 01 01 03 01 00 01       01 01 01 03 01 00 01
```

- Pattern A: `0x01` from `Old File` changed to `0x03` in `New File`; `0x00` and `0x01` are missing in `Old File`
- Pattern B: `0x03` and `0x01` are missing in `Old File`
- Pattern C: `0x01` and `0x03` are missing in `Old File`; `0x01` from `Old File` changed to `0x00` in `New File`
- Pattern D: `0x01` and `0x01` are missing in `Old File`

Also, the launcher remembers whether the checkbox was checked before, so it's possible that one of the bytes could indicate whether the checkbox should be remembered as checked. But those are not the first `01 00`/`01 01` bytes. For example, changing the `01 01` bytes to `01 00` does not do anything. This makes me wonder whether they are actually related to the state of the checkbox.

When I changed `01 03` to `00 03`, the launcher didn't start, but changing it back made the launcher start. Changing `01 03` to `01 00` also made the launcher not start. It also doesn't update the file once all three entries are present. I feel like I am messing with the structure of the file. With `Version B`, I can assume that `03` could be interpreted as `size of the next value`, but then we have to make sense of `0x010001` that goes after, and `0x010101` the goes before.

Well, playing around with `0x010001` gives such results:

- Changing `01 00 01` to `00 00 00` caused the checkbox to be *unchecked*;
- Changing the same `00 00 00` to `00 00 01` still leaves the checkbox *unchecked*;
- Changing the same value to `01 00 00` leaves the checkbox *unchecked*;
- Changing the same value to `00 01 01` leaves the checkbox *unchecked*;
- Changing the same value to `00 01 00` leaves the checkbox *unchecked*;
- Changing the same value to `01 01 00` makes the checkbox to be *checked*;
- Changing the same value to `01 01 01` makes the checkbox to be *checked*;
- Changing it back to `01 00 01` makes the checkbox to be *checked*;

As we can see here, there are many ways the checkbox can be either checked or unchecked. But what's the logic here? Here's my reasoning: every value that results in the checkbox being unchecked starts with either the `0x00` byte, or ends with `0x00 0x00` bytes. So, it's going to be `true` if the first value starts with the `0x01` byte, and either of the two (or both) following bytes are `0x01` as well.

Oh, genius. Something clicked in my brain and I decided to check whether the `1B DF 05 A5` is a CRC-32 value for the byte `01` (and it is). That's because I was sure it looked too random and unique to mean anything else. Plus, it only appeared to be generated only when the checkboxes are checked, otherwise there are four zero bytes. But now I have to guess what `hwid` is hashing in its CRC-32 value.

```
01 01       - start of the entry
04          - length of the string
73 61 76 65 - string "save"
1B DF 05 A5 - CRC-32 of `01`
01
01
01
03          - size of the next value
01 00 01    - state of the checkbox (checked)
```
 
Making sense of the three `0x01` bytes is a little bit challenging, since we're dealing with a checkbox that can be either checked or unchecked, and we already know that `01 00 01` is for toggling its state. Could they be just dummy values? If so, why changing one of the values to `00` prevents the launcher from starting? It starts only when the values are either `01 01 01` or `01 00 01`. I need more information, and I am going to get it by running the game.

### Byte structuring

After filling the email and password fields and running the game, I got new entries in the `config_file_01.ext` file. Those are `username` and `password`. I am using a test account that has a username of length `12` and a password of length `15`. I'm counting the characters just in case they could be used as a clue.

```
___________ __ __
F2 1B 42 EB 01 05

_____ __ s  a  v  e  ___________          __ ________
01 01 04 73 61 76 65 1B DF 05 A5 01 00 01 03 01 01 01

_____ __ u  s  e  r  n  a  m  e  ___________          __ ______________________________________________________________________________________
01 01 08 75 73 65 72 6E 61 6D 65 CD 12 0B 37 01 1B 01 1D 1B .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. 00

_____ __ e  x  c  l  u  s  i  v  e  _  f  u  l  l  s  c  r  e  e  n  ___________          __ ________
01 01 14 65 78 63 6C 75 73 69 76 65 5F 66 75 6C 6C 73 63 72 65 65 6E 1B DF 05 A5 01 00 01 03 01 00 01

_____ __ p  a  s  s  w  o  r  d  ___________          __ ________________________________________________________________________________________________________
01 01 08 70 61 73 73 77 6F 72 64 1C 26 4E 1F 01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
```

As you can see, I am using the same pattern of skipping the three mysterious bytes that go after the CRC value, and consider the next byte to be "the size of the following value". And `0x1D` in decimal is `29` - exactly the amount of bytes that go after the `1D` byte; `0x23` in decimal is `35` - also confirms the number of bytes. I will move forward with the `password` entry, even though I must also remind you that our goal here is to not decrypt the data, but to understand the structure of the file.

One weird thing that I've noticed:

```
1B DF 05 A5 01 01 01 03 01 00 01
|  |  |  |  |  *  |  |  *
1B DF 05 A5 01 01 01 03 01 00 01
*  *  *  *  |  *  |     *
CD 12 0B 37 01 1B 01 1D 1B .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. .. 00
*  *  *  *  |  *  |     *
1C 26 4E 1F 01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
               ^^-------^^
```

There's this weird pattern of the second byte after the CRC matching the first byte of the *allegedly* identified value. So it goes as `01 XX 01 YY XX`, where `YY` is known. I'll come back to this later. Another thing that I noticed after clicking on bytes is how some bytes consistently repeat. Here's for the `password` (I excluded the CRC value):

```
01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
                              ^^ -------------------- ^^ -------------------- ^^ -------------------- ^^
01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
                                    ^^ -------------------- ^^ -------------------- ^^ -------------------- ^^
01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
                                          ^^ -------------------- ^^ -------------------- ^^ -------------------- ^^
01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
                                                ^^ -------------------- ^^ -------------------- ^^
```

There are more bytes than there are characters in the `password` (the length is `15`); so, can we structure the bytes the way we can find out the string length?

```
1C 26 4E 1F             - CRC-32 value

01 21 01                - ???

23                      - number of the following bytes?

21                      - number of the following bytes, except the NUL terminator?

80 56 A6

93 73                   - ..15?

35 1A                   - 14..
   1A 1E                - 13
      54 EA             - 12
         9C 46          - 11
35 38                   - 10
   1A 3C                - 9
      54 F0             - 8
         9C 04          - 7
35 1A                   - 6
   1A 11                - 5
      54 E3             - 4
         9C 08          - 3
35 31                   - 2
   1A 1A                - 1
      00                - NUL terminator?
```

I am assuming that the last byte could be a `NUL` terminator, as both `username` and `password` end with `0x00`. What if `0x21` is the length of the following data and `80 56 A6` is part of the data's header? I am going to start the game using a different account, which has the same password length:

```
___________ ?? ?? ?? __ __ ?? ?? ??
1C 26 4E 1F 01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
*  *  *  *  |  |  |  |  |  |  |  |  |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |
4D 0E 26 00 01 21 01 23 21 80 56 A6 93 65 35 2D 1A 29 54 E5 9C 43 35 1B 1A 73 54 C9 9C 76 35 27 1A 08 54 95 9C 74 35 0C 1A 7E 00
            ^^^^^^^^^^^^^^^^^^^^^^^^^^
```

The bytes in the outlined part are identical. Then, there are other bytes that look identical that follow the same pattern. Let's remove the identical bytes to see what's remaining:

```
1C 26 4E 1F .. .. .. .. .. .. .. .. .. 73 .. 1A .. 1E .. EA .. 46 .. 38 .. 3C .. F0 .. 04 .. 1A .. 11 .. E3 .. 08 .. 31 .. 1A ..
4D 0E 26 00 .. .. .. .. .. .. .. .. .. 65 .. 2D .. 29 .. E5 .. 43 .. 1B .. 73 .. C9 .. 76 .. 27 .. 08 .. 95 .. 74 .. 0C .. 7E ..
```

We have 15 bytes. I am going to play around with "visible" bytes to find out the CRC value. OK, I failed. I am going to log in using yet another test account (thanks to my friend Sveir for providing it), this time the password is going to be `18` characters in length.

```
01 21 01 23 21 80 56 A6 93 73 35 1A 1A 1E 54 EA 9C 46 35 38 1A 3C 54 F0 9C 04 35 1A 1A 11 54 E3 9C 08 35 31 1A 1A 00
|  |  |  |  |  |  |  |  |     |     |     |     |     |     |     |     |     |     |     |     |     |     |     |
01 21 01 23 21 80 56 A6 93 65 35 2D 1A 29 54 E5 9C 43 35 1B 1A 73 54 C9 9C 76 35 27 1A 08 54 95 9C 74 35 0C 1A 7E 00
|     |           |  |        |     |     |     |     |     |     |     |     |     |     |     |     |     |     |
01 27 01 29 27 98 56 A6 8E 70 35 03 1A 19 54 9F 9C 57 35 10 1A 1C 54 95 9C 7C 35 21 1A 13 54 C5 9C 1D 35 02 1A 05 00 44 00 66 00 54 00
```

Here's what I see:

- The same `01 xx 01 yy xx` at the beginning
- Two values (`0x80` vs `0x98` and `0x93` vs `0x8E`) that are different... because the password is longer?
- Similar `0x56` and `0xA6` bytes
- The same byte pattern starting with `0x35`
- `0x23` (decimal `35`) and `0x29` (decimal 41) confirm that it could mean the "number of the following bytes"
- Many `0x00` bytes at the end, which makes me believe the `0x00` that I thought is a `NUL terminator` is actually something different

Let's also hide the matching bytes:

```
1C 26 4E 1F .. .. .. .. .. .. .. .. .. 73 .. 1A .. 1E .. EA .. 46 .. 38 .. 3C .. F0 .. 04 .. 1A .. 11 .. E3 .. 08 .. 31 .. 1A ..
4D 0E 26 00 .. .. .. .. .. .. .. .. .. 65 .. 2D .. 29 .. E5 .. 43 .. 1B .. 73 .. C9 .. 76 .. 27 .. 08 .. 95 .. 74 .. 0C .. 7E ..
B5 4C E9 6E .. 27 .. 29 27 98 .. .. 8E 70 .. 03 .. 19 .. 9F .. 57 .. 10 .. 1C .. 95 .. 7C .. 21 .. 13 .. C5 .. 1D .. 02 .. 05 .. 44 .. 66 .. 54 ..
```

Again, the password is 18 characters long, and there are 18 visible bytes if we start counting from the `0x70` byte. Since the password is encrypted, we can assume that there could also be data used to decrypt the password. The bytes in between the CRC and `0x70` are still unknown. But now I want to look at the bytes we decided to hide:

```
01 27 01 | 29 | 27 98 56 A6 8E | .. 35 .. 1A .. 54 .. 9C .. 35 .. 1A .. 54 .. 9C .. 35 .. 1A .. 54 .. 9C .. 35 .. 1A .. 00 .. 00 .. 00 .. 00
```

It goes `35 1A 54 9C`, `35 1A 54 9C`, `35 1A 54 9C`, `35 1A 00 00`, `00 00`. Why the sudden change of the pattern - why does it continue with `00`, instead of `54 9C ...`? The `00` bytes start after 14 (visible) bytes or 28 bytes if we count their hidden counterparts. I want to understand whether the `35 1A 54 9C` bytes play any role in the encryption/decryption process, because they don't seem to be part of the original password, since the same bytes repeat regardless of the password being encrypted. Even though my goal is to not decrypt the data, in order to understand the structure of the file, I gotta know what each byte means, but for that I probably need to decrypt it. Some sort of a paradox.

Here's one thing about the byte `0x27`: when analyzing other entries, they all seem to be about "the number of following bytes, except the last byte".

```
                       ________ |                __ ??
    save | 01 01 01 03 01 00 01 | 01 01 01 03 01 00 01
                    ^^          |             ^^

                       ________ |                __ ??
username | 01 1B 01 1D 1B .. 00 | 01 1B 01 1D 1B .. 00
                    ^^          |             ^^

                       ________ |                __ ??
password | 01 21 01 23 21 .. 00 | 01 21 01 23 21 .. 00
                    ^^          |             ^^
```

### The more data - the better

I want to combine all entries. Previously, I thought I can exclude the `username` and just work with the `password`, because I was using my game's main account and used my alt account later as well (and the last one was my friend's alt account). But, here's the thing. The more data - the better: you might see things that you otherwise can't without sufficient data. In this case, after combining `username` and `password` entries to compare them against each other, I found out these interesting details:

```
                                       ___________________________________________________________________________________________
                                      /  ______________________________________________________________________________________
             --- CRC ---             /  /      -----       probably where the data starts (encrypted password)
username A | CD 12 0B 37 - 01 1B 01 1D 1B 68   37 00 AB -> BA B5 xx A6 xx 35 xx A7 xx B5 xx A6 xx 35 xx A7 xx B5 6A A6 FD 00 6F 00
                           |     |             *  *           *     *     *     *     *     *     *     *     *     *     *     *
password A | 1C 26 4E 1F - 01 21 01 23 21 80   56 A6 93 -> 73 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 00
                           |     |             *  *           *     *     *     *     *     *     *     *     *     *     *     *     *     *     *
username B | 00 BA 31 2F - 01 25 01 25 25 4C   37 00 B6 -> B9 B5 xx A6 xx 35 xx A7 xx B5 xx A6 xx 35 xx A7 xx 05 xx 2C xx A7 xx B5 6A A6 F7 00 6F 00 6D 00
                           |     |             *  *           *     *     *     *     *     *     *     *     *     *     *     *     *     *     *     *
password B | 4D 0E 26 00 - 01 21 01 23 21 80   56 A6 93 -> 65 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 00
                           |     |             *  *           *     *     *     *     *     *     *     *     *     *     *     *     *     *     *
username C | 6C D2 F0 5B - 01 27 01 29 27 98   37 00 B5 -> B5 B5 xx A6 xx 35 xx A7 xx B5 xx A6 xx 35 xx A7 xx B5 xx A6 xx 35 xx A7 xx B5 28 A6 BA 00 63 00 6F 00 6D 00
                           |     |     |  |    *  *           *     *     *     *     *     *     *     *     *     *     *     *     *     *     *     *     *     *
password C | B5 4C E9 6E - 01 27 01 29 27 98   56 A6 8E -> 70 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 00 xx 00 xx 00 xx 00
                              ^^-------^^

username A: 12 characters / 12 bytes
password A: 15 characters / 15 bytes

username B: 17 characters / 16 bytes - weird! (lacks the byte `0x35`?)
password B: 15 characters / 15 bytes

username C: 18 characters / 18 bytes
password C: 18 characters / 18 bytes
```

Particularly:

- The first and second `0x01` bytes are always the same. Could probably indicate on the size of the next value.
- Second byte is same as the fifth byte (.. `0x1B` .. .. `0x1B`; .. `0x21` .. .. `0x21`; etc.).
- Fourth byte matches the number of bytes until the next entry: `0x1D` is `29 bytes`; `0x23` is `35 bytes`; etc.
- Fifth and sixth bytes of the last entries `username C` and `password C` match (`27 98`); `password A` and `password B` also have the same bytes matching (`21 80`).
- Seventh and eighth bytes of `usernames` and `passwords` match.
- Ninth byte of `password A` matches the ninth byte of `password B` (`0x93`).
- `username` pattern: `37 00 xx yy B5 A6 35 A7 ... 00`.
- `password` pattern: `56 A6 xx yy 35 1A 54 9C ... 00`.

First of all, it's interesting how `username` and `password` have a different byte pattern. They first start with the mysterious `37 00` / `56 A6` bytes. Is this related to the encryption or does it just define the type of the entry? And then, if we believe that the data starts with the tenth byte (`0xBA` / `0x73` / `0xB9` / etc.), we're left with one unknown byte on the left. Maybe it's part of the two previous bytes, but then it's unique for almost each entry. Also, the fifth and sixth bytes... it's too much. What I mean - I'm getting tired. Why am I even doing this? It might look like it took me hour(s) to write this, but it actually took/takes days, since I am doing this on my free time. Now I'm ranting. This is no good, no good. I'm not even sure if I should look into the similarity of the last bytes of all `username` entries:

```
username A | ... B5 6A A6 FD 00 6F 00
username B | ... B5 6A A6 F7 00 6F 00 6D 00
username C | ... B5 28 A6 BA 00 63 00 6F 00 6D 00
```

I think I could add some padding:

```
username A | ... [B5 6A] [A6 FD]         [00 6F]         [00]
username B | ... [B5 6A] [A6 F7]         [00 6F] [00 6D] [00]
username C | ... [B5 28] [A6 BA] [00 63] [00 6F] [00 6D] [00]
```

Not like this helped me understand anything. But this is how I'm trying to make sense of everything. And that looks absolutely weird to me. I want to give up. I want this to be the end of my file format scooby-doobery. The goal here is to learn to read bytes and not decrypt the password. But it seems like I really have to decrypt the password to understand which bytes play role in encryption in order to label them appropriately. This is not what I want to do right now, but I still need to at least make sense of what I've found so far.

### Hardest part - dealing with encrypted data

One thing to not forget about: *Don't forget the previous findings*. I have focusing issues, so this is a good thing to keep in mind. We are dealing with an entry that stores encrypted username and password.

`password A` and `password B` have bytes `21 80 56`. They have the same length - 15 characters / 15 bytes. `0x80` is `128`. `password C` has bytes `27 98 56`, but `username C` also has the bytes `27 98`, and they are both of the same length - 18 characters / 18 bytes. `0x98` is `152`. Entries with those bytes matching have a similar length. Should I interpret it as... if there's `0x27`, then the following byte is `0x98`; and if there's `0x21` then the following byte is `0x80`. Question: why? Is that byte related to the previous or the next byte?

```
01 1B 01 1D 1B  68 37 00 AB  -> BA ...   |   ... 1D 1B 68 37 00 AB ...   |   12 bytes
|  *  |  *  *      *  *                  |                               |
01 21 01 23 21 [80 56 A6 93] -> 73 ...   |   ... 23 21 80 56 A6 93 ...   |   15 bytes
|  *  |  *  *      *  *                  |                               |
01 25 01 25 25  4C 37 00 B6  -> B9 ...   |   ... 25 25 4C 37 00 B6 ...   |   17 characters / 16 bytes
|  *  |  *  *      *  *                  |                               |
01 21 01 23 21 [80 56 A6 93] -> 65 ...   |   ... 23 21 80 56 A6 93 ...   |   15 bytes
|  *  |  *  *      *  *                  |                               |
01 27 01 29 27  98 37 00 B5  -> B5 ...   |   ... 29 27 98 37 00 B5 ...   |   18 bytes
|  *  |  *  *   |  *  *                  |                               |
01 27 01 29 27  98 56 A6 8E  -> 70 ...   |   ... 29 27 98 56 A6 8E ...   |   18 bytes

0x1B = 27      0x21 = 33      0x25 = 37     0x27 = 39      0x37 = 55     0xA6 = 166
0x68 = 104     0x80 = 128     0x4C = 76     0x98 = 152     0x56 = 86     0xA656 = 42582

0xAB = 171     0x93 = 147     0xB6 = 182    0xB5 = 181     0x8E = 142

01 - size of the following value
   1B - value A
      01 - size of the following value
         1D - value B, number of bytes til the next entry
            1B - value C (2 bytes less than the previous value), number of bytes excluding the last byte
               68 - value D, encryption related (?)
                  0037 - value E, entry type / encryption related (?)
                     AB - value F, encryption related (?)
                        BA ... - value G, data (encrypted username)
```

This is how I see the data is structured, which is not sequential the same way as the previous file format, I mean the resource files, it's the same format, but the structure is different:

```
                                  +------------------------------------------------------------------------------------------------------------------------------------------+
                                  |      +------------------------------------------------------------------------------------------------------------------------------+    |
+-------------+----+----+----+----| +----| +----+-------+----+----------------------------------------------------------------------------------------------------------|----|
| B5 4C E9 6E | 01 | 27 | 01 | 29 | | 27 | | 98 | 56 A6 | 8E | 70 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 00 xx 00 xx 00 xx | 00 |
+-------------+----+----+----+----| +----| +----+-------+----+----------------------------------------------------------------------------------------------------------|----|
                                  |      +------------------------------------------------------------------------------------------------------------------------------+    |
                                  +------------------------------------------------------------------------------------------------------------------------------------------+
```

In this one, it seems to be a bit complicated because we're dealing with (allegedly) encrypted credentials, where they are twice the original string length, and on top of that there are 4-5 more mysterious bytes. I don't know what exactly the `0x98`, `0xA656` and `0x8E` bytes represent, but I feel like the first and last ones are `uint8`, and the second one is `uint16`. But there's also the possibility that `0x8E` could be part of the data: maybe it's possible because the last `0x00` byte means "the ending of the entry", so if we count backwards from the last byte, we're left with the `8E 70` pair. And it's `password C` which has 18 characters / 18 bytes, therefore it could be that `0x8E` could be part of the data. I just assumed it's not part of the data because: 1) I counted from the last `0x00` byte; 2) it's random and doesn't match the `35 .. 1A .. 54 .. 9C ..` pattern, which I think is weird and there's gotta be something to it.

```
                                  +----------------------------------------------------------------------------------------------------------------------------------------+
                                  |      +----------------------------------------------------------------------------------------------------------------------------+    |
+-------------+----+----+----+----| +----| +----+-------+-------------------------------------------------------------------------------------------------------------|----|
| B5 4C E9 6E | 01 | 27 | 01 | 29 | | 27 | | 98 | 56 A6 | 8E 70 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 54 xx 9C xx 35 xx 1A xx 00 xx 00 xx 00 xx | 00 |
+-------------+----+----+----+----| +----| +----+-------+-------------------------------------------------------------------------------------------------------------|----|
                                  |      +----------------------------------------------------------------------------------------------------------------------------+    |
                                  +----------------------------------------------------------------------------------------------------------------------------------------+
```

But maybe my assumptions are wrong. *Sigh*. I need to *prove* that this is the correct structure. I changed the password of the `password C` entry to make it 8 characters long and see if we still get the `0x00` byte at the end, now we have `password D`:

```
             --- CRC ---   -- -- -- -- -- --  -----    probably where the data starts
password C | B5 4C E9 6E - 01 27 01 29 27 98  56 A6 -> 8E 70 35 03 1A 19 54 9F 9C 57 35 10 1A 1C 54 95 9C 7C 35 21 1A 13 54 C5 9C 1D 35 02 1A 05 00 44 00 66 00 54 00
password D | 05 EE B4 64 - 01 13 01 15 13 48  56 A6 -> 94 66 35 31 1A 2C 54 C3 9C 44 35 3B 1A 78 00 36 00
```

Well, it does end with `0x00`. Well. These 00's look very suspicious. I am going to compare it against the original password:

```
Original password:  56 78 67 64 74 72 33 36
Encrypted password: 94 66 35 31 1A 2C 54 C3 9C 44 35 3B 1A 78 00 36 00
```

Need to align the bytes of the original password with the ones that are random in the encrypted password:

```
   56    78    67    64    74    72    33    36
94 66 35 31 1A 2C 54 C3 9C 44 35 3B 1A 78 00 36 00
```

Hm. That `0x36` is same in both `original` and `encrypted`. Okay, I need a longer password, I need more data. I am going to use original and encrypted `password C`:

```
   40    4A    52    38    67    59    57    32    4C    68    58    62    2D    4B    4E    44    66    54
8E 70 35 03 1A 19 54 9F 9C 57 35 10 1A 1C 54 95 9C 7C 35 21 1A 13 54 C5 9C 1D 35 02 1A 05 00 44 00 66 00 54 00
```

Interesting. The last bytes here also match, so it's not a coincidence. This clearly shows that those `0x00` bytes basically say "the next byte is not going to be encrypted". And, if my understanding is correct, the repeating bytes `0x35` / `0x1A` / `0x54` / `0x9C` are used to encrypt the following byte. But I still don't know if I should read starting from the last `0x00` byte until the `0x70` byte or from the `0x54` byte to the `0x8E` byte. Actually, if it was to be read like this:

```
4E    44    66    54
05 00 44 00 66 00 54 00
----- ----- ----- -----
```

That means that `0x05` should be the original value `0x4E`, because "00 doesn't encrypt the value". So, probably, that `0x05` is encrypted with the neighboring `0x1A` byte. If so, the last `0x00` is not part of the password:

```
   4E    44    66    54
1A 05 00 44 00 66 00 54 00
----- ----- ----- -----
```

This means we need to do something with the bytes `0x1A` and `0x4E` to get `0x05`. Or not. It's really weird that if some encryption algorithm is used, then it's expected that the entire data is going to be encrypted, but there are cases where a portion (usually the ending) of data is not encrypted. I tried to change the password one more time, this time to "aabcaabcaabcaabcaabcaabc". Here's the reasoning behind that: I want to see how each of the repeating bytes `35 1A 54 9C` are going to encrypt the "aabc". So let's look at the new `password E` (starting from its CRC value):

```
C6 C7 00 24 01 33 01 14 33 28 56 A6 84 51 35 28 1A 29 54 C4 9C 92 08 00 08 00 63 00
```

This is... disappointing. If you didn't notice - the "encrypted" data is *smaller* than the original password. Original password has *24 characters*, whereas there are only 9 bytes. This is probably because there's some sort of compression involved. Does that mean it just compresses the data and does not encrypt; or it compresses first, then encrypts?

```
--- CRC --- __ __ __ __    ?? _____ ----- ----- ----- ----- ----- ----- ----- -- __
C6 C7 00 24 01 33 01 14 33 28 56 A6 84 51 35 28 1A 29 54 C4 9C 92 08 00 08 00 63 00
                        ^^
```

Look at the `0x33` byte - it's `51` in decimal. Judging by the previous findings, it means "the number of following bytes, except the last `0x00`". But there are no 51 bytes. There are only 18 bytes. And `0x14` (a byte before `0x33`) - `20` in decimal - also means "the number of following bytes", but including the last `0x00` as well. And this one seems correct. Also, the last `0x63` doesn't have a counterpart? One byte missing?

```
                         _______ _______ _______ _______ _______ -------- _____ __
Encrypted password | ... [84] 51 [35] 28 [1A] 29 [54] C4 [9C] 92 08 00 08 00 63 00
```

If we don't count the mysterious `08 00 08`, then there are 6 bytes in the encrypted data. I assumed compression then encryption of the data. Remember the `username B`, which had 17 characters, but only 16 bytes -  probably because of compression.

```
             --- CRC ---   __ __ __ __       _____                                                                                        "o"   "m"      (".com")
username B | 00 BA 31 2F - 01 25 01 25 25 4C 37 00 B6 -> B9 B5 xx A6 xx 35 xx A7 xx B5 xx A6 xx 35 xx A7 xx 05 xx 2C xx A7 xx B5 6A A6 F7 00 6F 00 6D 00
```

But why did `password E` and `username B` get compressed, but not other entries? Maybe compression doesn't always "work". I don't know. This makes things even more complicated, as we would need to inspect game's executables through disassembly to find the function that reads the file. I feel like this is where I should wrap this up.

### "Final" thoughts

I am interested in digging even deeper, but for now I just want to mention some things I would've thought about if I knew that the password is encrypted without compression. For example, continuing with the `password C` example, probably the `0x98` and `0x56 0xA6` bytes also play a role in encryption. Remember - those are the "password's header data".

```
0x98 = 152   |   0x56 = 86   |   0xA6 = 166   |   0xA656 = 42582
```

Even though `0x56` and `0xA6` "go together" (they are unique to password entries), doesn't mean they can't be individual (`uint8`) bytes.

```
+----------------+----------------+----------------+-------------------------+------------------+
|   Original     |   Encrypted    |   Counterpart  |   Transition            |   Comparison     |
+----------------+----------------+----------------+-------------------------+------------------+
|   0x40 = 64    |   0x70 = 112   |   0x8E = 142   |   64  -> [142] -> 112   |   Low  -> High   |
|   0x4A = 74    |   0x03 = 3     |   0x35 = 53    |   74  -> [53]  -> 3     |   High -> Low    |
|   0x52 = 82    |   0x19 = 25    |   0x1A = 26    |   82  -> [26]  -> 25    |   High -> Low    |
|   0x38 = 56    |   0x9F = 159   |   0x54 = 84    |   56  -> [84]  -> 159   |   Low  -> High   |
|   0x67 = 103   |   0x57 = 87    |   0x9C = 156   |   103 -> [156] -> 87    |   High -> Low    |
|   0x59 = 89    |   0x10 = 16    |   0x8E = 142   |   89  -> [142] -> 16    |   High -> Low    |
|   0x57 = 87    |   0x1C = 28    |   0x35 = 53    |   87  -> [53]  -> 28    |   High -> Low    |
|   0x32 = 50    |   0x95 = 149   |   0x1A = 26    |   50  -> [26]  -> 149   |   Low  -> High   |
|   0x4C = 76    |   0x7C = 124   |   0x54 = 84    |   76  -> [84]  -> 124   |   Low  -> High   |
|   0x68 = 104   |   0x21 = 33    |   0x9C = 156   |   104 -> [156] -> 33    |   High -> Low    |
|   0x58 = 88    |   0x13 = 19    |   0x8E = 142   |   88  -> [142] -> 19    |   High -> Low    |
|   0x62 = 98    |   0xC5 = 197   |   0x35 = 53    |   98  -> [53]  -> 197   |   Low  -> High   |
|   0x2D = 45    |   0x1D = 29    |   0x1A = 26    |   45  -> [26]  -> 29    |   High -> Low    |
|   0x4B = 75    |   0x02 = 2     |   0x54 = 84    |   75  -> [84]  -> 2     |   High -> Low    |
|   0x4E = 78    |   0x05 = 5     |   0x9C = 156   |   78  -> [156] -> 5     |   High -> Low    |
+----------------+----------------+----------------+-------------------------+------------------+
```

I neatly put the data into this table to see their decimal values, and also see the encryption behavior. After comparing the original and encrypted bytes, we can see that it doesn't always go from the high value to the low. This means we don't do some subtraction to get a small value, because the original value can be small and the encrypted can be big. Now I need to figure out what to do with this piece of information. But also look at this:

```
(142) (64)    (112) (+48)
0x8E: 0x40 -> 0x70  (+0x30)          10001110: 01000000 -> 01110000

(156) (103)   (87)  (-16)
0x9C: 0x67 -> 0x57  (-0x10)          10011100: 01100111 -> 01010111

(156) (76)    (124) (+48)
0x9C: 0x4C -> 0x7C  (+0x30)          10011100: 01001100 -> 01111100

(156) (45)    (29)  (-16)
0x9C: 0x2D -> 0x1D  (-0x10)          10011100: 00101101 -> 00011101


(53)  (74)    (3)   (-71)
0x35: 0x4A -> 0x03  (-0x47)          00110101: 01001010 -> 00000011

(53)  (89)    (16)  (-73)
0x35: 0x59 -> 0x10  (-0x49)          00110101: 01011001 -> 00010000

(53)  (104)   (33)  (-71)
0x35: 0x68 -> 0x21  (-0x47)          00110101: 01101000 -> 00100001

(53)  (75)    (2)   (-73)
0x35: 0x4B -> 0x02  (-0x49)          00110101: 01001011 -> 00000010


(26)  (82)    (25)  (-57)
0x1A: 0x52 -> 0x19  (-0x39)          00011010: 01010010 -> 00011001

(26)  (87)    (28)  (-59)
0x1A: 0x57 -> 0x1C  (-0x3B)          00011010: 01010111 -> 00011100

(26)  (88)    (19)  (-69)
0x1A: 0x58 -> 0x13  (-0x45)          00011010: 01011000 -> 00010011

(26)  (78)    (5)   (-73)
0x1A: 0x4E -> 0x05  (-0x49)          00011010: 01001110 -> 00000101


(84)  (56)    (159) (+103)
0x54: 0x38 -> 0x9F  (+0x67)          01010100: 00111000 -> 10011111

(84)  (50)    (149) (+99)
0x54: 0x32 -> 0x95  (+0x63)          01010100: 00110010 -> 10010101

(84)  (98)    (197) (+99)
0x54: 0x62 -> 0xC5  (+0x63)          01010100: 01100010 -> 11000101
```

Weirdly enough, the beginning of the data has a pattern of adding `48 bytes`, then the next byte subtracts `16 bytes`, then for the next bytes it adds `48` and subtracts `16` again. This is just an interesting observation of the encryption behavior. But this is *only if* we assume that the "counterpart byte" plays any role in the encryption process. This weird pattern just says "it could". Then, I would go from bytes to bits. Because encryption is all about shifting and masking bits.

```
+---------------------+---------------------+---------------------+----------------------------------------+
|   Original          |   Encrypted         |   Counterpart       |   Transition                           |
+---------------------+---------------------+---------------------+----------------------------------------+
|   0x40 = 01000000   |   0x70 = 01110000   |   0x8E = 10001110   |   01000000 -> [01110000] -> 10001110   |
|   0x4A = 01001010   |   0x03 = 00000011   |   0x35 = 00110101   |   01001010 -> [00000011] -> 00110101   |
|   0x52 = 01010010   |   0x19 = 00011001   |   0x1A = 00011010   |   01010010 -> [00011001] -> 00011010   |
|   0x38 = 00111000   |   0x9F = 10011111   |   0x54 = 01010100   |   00111000 -> [10011111] -> 01010100   |
|   0x67 = 01100111   |   0x57 = 01010111   |   0x9C = 10011100   |   01100111 -> [01010111] -> 10011100   |
|   0x59 = 01011001   |   0x10 = 00010000   |   0x8E = 10001110   |   01011001 -> [00010000] -> 10001110   |
|   0x57 = 01010111   |   0x1C = 00011100   |   0x35 = 00110101   |   01010111 -> [00011100] -> 00110101   |
|   0x32 = 00110010   |   0x95 = 10010101   |   0x1A = 00011010   |   00110010 -> [10010101] -> 00011010   |
|   0x4C = 01001100   |   0x7C = 01111100   |   0x54 = 01010100   |   01001100 -> [01111100] -> 01010100   |
|   0x68 = 01101000   |   0x21 = 00100001   |   0x9C = 10011100   |   01101000 -> [00100001] -> 10011100   |
|   0x58 = 01011000   |   0x13 = 00010011   |   0x8E = 10001110   |   01011000 -> [00010011] -> 10001110   |
|   0x62 = 01100010   |   0xC5 = 11000101   |   0x35 = 00110101   |   01100010 -> [11000101] -> 00110101   |
|   0x2D = 00101101   |   0x1D = 00011101   |   0x1A = 00011010   |   00101101 -> [00011101] -> 00011010   |
|   0x4B = 01001011   |   0x02 = 00000010   |   0x54 = 01010100   |   01001011 -> [00000010] -> 01010100   |
|   0x4E = 01001110   |   0x05 = 00000101   |   0x9C = 10011100   |   01001110 -> [00000101] -> 10011100   |
+---------------------+---------------------+---------------------+----------------------------------------+
```

```
+---+---+---+---+---+---+---+---+
| 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 |   Value A (original: 0x40)
+---+---+---+---+---+---+---+---+
  |   |   |   |   |   |   |   |
+---+---+---+---+---+---+---+---+
| 1 | 0 | 0 | 0 | 1 | 1 | 1 | 0 |   Value B (counterpart: 0x8E)
+---+---+---+---+---+---+---+---+
  |   |   |   |   |   |   |   |
  v   v   v   v   v   v   v   v
+---+---+---+---+---+---+---+---+
| 0 | 1 | 1 | 1 | 0 | 0 | 0 | 0 |   Result (encrypted: 0x70)
+---+---+---+---+---+---+---+---+
         ^^^                 ^^^
```

The third and the last bits differ even though the inputs are both `0`. That means we're dealing with more values than just the `Value B`, and this is a "serious" encryption. It would take a *lot* of combinations to try to decrypt the password. We could go that way, or at least try to ease our process by guessing the encryption method. We could look for algorithms that turn the plaintext into twice its size. But there's one thing that bugs me - sometimes, a part of the data doesn't get encrypted (like those at the end with `0x00`). Is it because a custom encryption algorithm is being used?

If not that weird fact, I would like to look into *symmetric-key algorithms*. Particularly, RC4 - a stream cipher. Is there any reason to believe that this is what is being used here? I don't know. I don't even know if there's a secret key, or why does each byte have a repeating counterpart. All I know:

- We have three mysterious bytes (`98 56 A6`) that could play role in encrypting the password;
- If there's an encryption key, it probably has a length of 128 bytes, and is probably generated through KSA (Key Scheduling Algorithm) and PRGA (Pseudo Random Generation Algorithm), if we believe the RC4 is being used;
- Then, there's a keystream that XORs with the plaintext (original password) in order to generate the encrypted password.

Look. All of those "counterpart bytes" are consistently same no matter what password is being encrypted, as well as `56 A6`, and one byte after those two bytes is different in all entries, but somehow it's (allegedly) part of the encrypted password. Let's end this here, and probably leave this for the future material where I am going to discuss about basics of reverse engineering executables.

### Unfinished structure of the same file format used for storing configuration data

![file-format-config](https://github.com/zwoly/reversed/assets/164810717/f0d18db5-77cd-4b49-bbe4-e10d2c6a7430)








































