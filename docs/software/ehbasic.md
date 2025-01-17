# Durango-specific EhBASIC commands

[**EhBASIC** from the late Lee Davidson](http://forum.6502.org/viewforum.php?f=5) is intended as a generic, _console-type_ BASIC interpreter for 6502 systems.
In order to support _Durango-X_'s **graphic, colour and sound** capabilities, new commands have been added.

## CONIO interface

The standalone EhBASIC interpreter for Durango-X is supplied with a suitable minimal _firmware_, mostly around the `CONIO` function as included in
[the minimOS Operating System](https://github.com/zuiko21/minimOS/tree/master/OS). There is similar CONIO support in
[DurangoLib](https://github.com/durangoretro/DurangoLib/blob/main/asm/conio.s) C library.

`CONIO` acts as a **terminal emulator**; for easier parsing, most _control codes_ are **single-byte**, thus the [standard CONIO features](conio.md) are accessible
thru a mixture of **traditional ASCII, `bash` shortcut keys and ZX-Spectrum codes**. _All of them are accessible via standard `PRINT CHR$()` BASIC sentences_, but
a few of them have beed granted new BASIC commands for convenience:

### `CLS`

Clears the screen. _If an alternative screen is on display, will set that as the current output screen_.

Equivalent to `PRINT CHR$(12);`

### `LOCATE c,r`

Moves the printing position to column `c` and row `r`; acceptable values are 0...15 in colour mode, and 0...31 in HIRES.

Equivalent to `PRINT CHR$(23);CHR$(32+r);CHR$(32+c);`

### `INK i`

Sets foreground ("ink") colour to `i`; values go thru `MOD 16` to become **0...15**, as per the [standard Durango-X palette](../hardware/palette.md). _In HIRES mode, colours are internally set
but NOT displayed in any way until back into colour mode_.

Equivalent to `PRINT CHR$(18);CHR$(32+i);`

### `PAPER p`

Sets background ("paper") colour to `p`; same restrictions as above apply.

Equivalent to `PRINT CHR$(20);CHR$(32+i);`

### `CURSOR n`

Enables (when `n` is 1) or disables (0) the cursor.

Equivalent to `PRINT CHR$(17);` (enable) and `PRINT CHR$(19);` (disable)

!!! note

	Any _odd_ value acts as `1`, while any _even_ value acts as `0`.

## Video modes

### `MODE n`

Sets video mode according to the following table:

* `0`: colour mode, standard video
* `1`: colour mode, inverse video
* `2`: HIRES mode, standard video (default mode in EhBASIC)
* `3`: HIRES mode, inverse video
* `4`: greyscale mode, standard video
* `5`: greyscale mode, inverse video

!!! bug

	Values `4` and `5` (_greyscale_ mode) are **NOT yet accepted**.

Equivalent to `POKE $DF80, (PEEK($DF80) AND %00110000) OR ((n AND 3)<<6) OR (-8*(n<4))` _(...phew!)_

### `SCREEN n`

Selects one out of four available screens in Durango-X (default=**3** at `$6000`) for _display_. In order to actually _redirect text output_ to the selected screen, a `CLS` or equivalent command must be issued.

!!! warning

	* In order to _write_ on any screen besides the standard 3, `Memory size` MUST be properly set **at boot time**, otherwise EhBASIC may crash! Suitable values are:
		- `16384` (or `$4000`) allows use of screens **2 and 3**
		- `8192` (or `$2000`) allows use of screens **1, 2 and 3**
		- `4096` (or `$1000`) allows use of **any** screen, including **0**.  _Since zeropage, stack and system variables live in this area, only the **bottom half** of the screen will be used;_ `CONIO` automatically accounts for that.
	* _Graphic commands_ do NOT take into account the `CONIO` pointers, thus will draw into the screen _on display_ at once. **They won't respect the _upper half_ of `SCREEN 0`**, either.

Equivalent to `POKE $DF80, (PEEK($DF80) AND %11000000) OR (n<<4) OR 8`

## Sound and other time-related commands

### `BEEP d,p`

Plays a note of pitch `p` (integer value 0...42, chromatic scale from F3 to B6) for `d`/50 seconds.

In case a similar sentence is found on a _ZX-Spectrum_ programme, it must be rewritten in Durango-X EhBASIC as `BEEP d*50,p+7`

### `PAUSE n`

Waits for `n` _IRQ system interrupts_ to happen, or until any key is pressed. _Current version is limited to 255, thus a little more than a second wait_.
If `n` is zero, it waits for any key to be pressed, without any time limit.

In case a similar sentence is found on a _ZX-Spectrum_ programme, it must be rewritten in Durango-X EhBASIC as `PAUSE n*5`

## Graphic commands

These commands are inspired by those on the _ZX-Spectrum_, but instead of the set foreground colour, they all rely on an extra parameter for colour.
_There is no "attribute area" in Durango-X_, thus any pixel is **completely independent** of its neighbours.

Unlike the ZX-Spectrum, though, the _origin of coordinates_ is at **top left** of the screen.

### `PLOT x,y,c`

Draws a **pixel** of colour `c` (from the [standard Durango-X palette](../hardware/palette.md)) on screen coordinates `x,y`; acceptable values are 0...127 in colour mode,
and 0...255 in HIRES (this mode will also do `c MOD 2`, thus only _odd_ colour values will display the pixel, otherwise is _cleared_).

### `LINE x1,y1,x2,y2,c`

Draws a **line** of colour `c` from screen coordinates `x1,y1` to `x2,y2`; the usual restrictions apply.

### `CIRCLE x,y,r,c`

Draws a **circle** of colour `c` and radius `r` centered at screen coordinates `x,y`; besides the usual restrictions,
the whole circle **must** fit inside the screen, otherwise a `Function call Error` is generated.

### `RECT x1,y1,x2,y2,c`

!!! bug

	_This is **NOT working** on the current version_. Intended to draw a filled rectangle, the command will be accepted (and its parameters checked!) but nothing will be drawn on screen.

## LOADing and SAVing programs

!!! note

	In its standard form, Durango-X _has no mass-storage device_. But several options are available, including an **SD card reader** on the _Development Cartridge_ (plus a similar _sidecar_ SD reader, under development) and, of course, the _host filesystem_ when running under the **Perdita** emulator thru the **Virtual Serial Port**.
	A **plug-in system** has been developed to include a [**suitable _driver_**]() when assembling the EhBASIC binary, thru the option `-DAUXIO=path-to-driver.s`, but there might be differences in use depending on the chosen device.
	
### `LOAD`

After issuing this command, if the device is available it will propmt asking for a `Filename`. If a _matching file_ is found, it will load;
otherwise, a `Function call Error` will be issued. **Partial filenames** are thus supported. _No parameters are used_.

If `$` is stated as _filename_, then a **Directory listing** is displayed (with _executable binaries_ preceded by a `*`) and will return
to the BASIC prompt _without any LOAD attempt_.

!!! warning

	If `LOAD` succeeds finding a file, **the program previously stored in RAM will be DELETED**. If a _Directory listing_ is requested via `$`
	or no matching file is found, the previous program will stay in RAM.

### `SAVE`

Syntax and operation is the same as `LOAD`: use `$` for **Directory listing** or a **_complete_ filename** for saving, when prompted. _No parameters_.

!!! tip

	Both `LOAD` and `SAVE` use **ASCII format** for easy interoperation with modern computers.

!!! note

	Loading or Saving in _Perdita_ thru **VSP** (default) does NOT support `$` _directory listings_, nor _partial filenames_.
