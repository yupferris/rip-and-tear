- General stuff being displayed:
  - Plasma in char area (40 chars x 25 lines)
  - Star as sprite zoomer
  - Black border around screen

- General plasma routine:
  - The plasma is displayed using FPP by using the repeating char lines trick and changing $d018 every line for g-accesses, where one of 8 different 144x1 lines of graphics is chosen and displayed for each of the 200 lines in the visible screen.
  - The left/right borders of the plasma are simply displayed using hardcoded chars; there are no sprites visible over the plasma (except for the zoomer, where a separate FPP routine is used and what appears to be a constant number of sprites, which I don't plan on digging into).
  - The base 144x8 plasma picture that's stretched using FPP changes each frame; this is done by updating the 38 bytes of screen mem (char indexes). It doesn't look like any updates are done on a per-pixel level.
  - The screen mem used (only the first line is used) starts at 0x3c00. The hardware reads this line, along with all of the color data, at the top of the screen. Before the FPP starts doing its thing, the bank is switched. The FPP repeats the last line of each character, and the character mem is changed per-line. Since the bank is switched between reading screen/color mem and reading char mem, these regions don't actually overlap. This also means that the screen/color mem can't change throughout the screen, but it's sufficient to simply change char graphics. This FPP technique is explained here: http://codebase64.org/doku.php?id=base:fpp-last-line
  - TODO: Check screen alignment/reg notes below to double-check this theory. If possible, try to recreate this with custom data to be even more sure (even though I feel like I understand this pretty well now :) ).

- Known (rough) memory map:
  - $0060       - sprite zoomer enable (0/1 for off/on)
  - $0066-$0067 - SMC temp storage for the absolute addr for the instruction restored at the addr at $006a-$006b in the sprite zoomer uninit routine (little-endian)
  - $0068-$0069 - SMC address in first $d011 bashing routine (where the nop's will be replaced by a jmp to switch raster routines mid-frame for the sprite zoomer)
  - $006a-$006b - SMC address in second $d011 bashing routine (where lda $zzzz, y will be replaced by a jmp to switch routines mid-frame back to first routine)
  - $1000-$103f - Initial frame raster IRQ
  - $1040-$1061 - Semi-stable raster IRQ and screen setup 1/2 (semi-stable because it actually ends up with 0-1 cycle jitter)
  - $1062-$1096 - Semi-stable raster IRQ and screen setup 2/2
  - $1097-$1dbc - First $d011-bashing raster routine (no sprite zoomer)
  - $1e0e-$???? - plasma x-update routine (this is what I'm after :) )
  - $27eb-$???? - sprite zoomer frame init routine
  - $29df-$2a05 - sprite zoomer frame uninit routine
  - $2a06-$2a08 - effect sequencing values (TODO: Figure out what each one is and if there are more of them)
  - $2a09-$???? - frame counting/effect sequencing code
  - $3c00-$3c28 - Screen memory (only this first row is used)
  - $9000-$???? - music update routine
  - $c000-$ffff - Character data

- Initial frame raster IRQ: $1000-$103f on line 0
  - Initial IO regs:
    - $dd00: $4b (VIC bank 0 ($0000-$3fff))
    - $d011: $11 (y scroll: 1, 24 rows, screen visible, text mode, extended bg off)
      - Note 24 rows when effect is 25 rows high
        - It also seems like the top or bottom border takes up two of these rows, and the other border makes the screen 2 pixels taller (each border is 2px high)
    - $d016: $d8
    - $d018: $ff
  - Stores Y, A, X in $e0-$e2
  - Stores $1d (29) in $d001 and $d003 (sprite 0 and 1 y coords)
    - Note that 29 + 21 = 50, where the screen starts (these probably display some of the border)
    - TODO: Check data pointers for sprites 0 and 1 to see what's there
  - Stores $05 (green) in $d020, $d021, $d027, $d028 (bg color, screen color, sprite 0 color, sprite 1 color)
    - Note that colors change as plasma changes color
  - Stores $0e (light blue) in $d022 (extra bg color 1)
  - Stores $04 (purple) in $d023 (extra bg color 2)
  - Jumps to subroutine at $27eb
    - Looks like this is code to start the sprite zoomer stuff and do the jump injection stuff in the first $d011 bashing section. This code also appears to be located right after the raster IRQ at $1040 (yes it's huge).
    - Fun fact: we can overwrite this JSR with 3 `nop`'s and the sprite zoomer goes away. This indicates that 1. yes, it's the sprite zoomer init, and 2., there's probably a separate routine later that will undo the SMC to switch raster routines (otherwise the screen would go bonkers if we removed this call on subsequent frames).
    - TODO: Find out what this does
  - Sets up another raster IRQ at $1040 on line $2f (47)
  - ACK interrupts
    - asl $d019
    - lda $dc0d (Interrupt control + status reg), does nothing with this value (?)
  - Restores Y, A, X from $e0-$e2

- Raster IRQ at $1040-$1061 on line $2f (47)
  - Pushes A, X, Y onto stack
  - Sets up nested raster IRQ at $1062 on next line ($30, 48)
  - ACK interrupts
    - asl $d019
    - lda $dc0d (Interrupt control + status reg), does nothing with this value (?)
      - This seems to be a consistent pattern. Come to think of it I remember reading somewhere that technically you're supposed to touch this register when ACK'ing interrupts or something, but I'm not entirely sure. Need to look into this.
  - Stores SP in X
  - Clear interrupt flag so nested IRQ can fire
  - Bunch of nop's to stabilize when nested IRQ fires
    - Looks like there's more than there needs to be; my tests only ever hit 3 or 4 of these before the nested IRQ fires
    - This actually ends with `jmp *` at the end, wonder if that was some safety net thing for testing. If this is meant to stabilize the raster to 0 or 1 cycles off, it would be bad if we actually hit this jmp, so I don't think it ever gets that far.

- Semi-stable IRQ at $1062-$1096 on line $30 (48) (occurs at cycle 9 or 10; one cycle jitter)
  - Restore SP from X
    - Easy way to get around HW IRQ stack manipulation
  - Wait loop (waits until cycle 27/28 on same line)
  - Stores $f8 (248) in $d001 and $d003 (sprite 0 and 1 y coords)
    - Yeah these are def related to the borders
  - Set up X and Y for plasma motion in y-direction (I think :) )
    - Here we have a pattern `clc, lda imm1, adc imm2` where imm1 and imm2 determine the plasma's offset in the x/y directions. Limiting these to small values makes movement appear "chunky"; writing them to specific values appears to offset the plasma. I'm not sure yet which var is which off set, though.
      - imm1 is initialized to some value and imm2 is added to it each frame. In both of the following cases the value is limited further and then written back to imm1.
    - lda with immediate value that's updated each frame (values seen: $32, $30, $2e)
    - Add immediate value to value above (clc, adc #$fe, and #$7f, sta)
    - tax
      - X is some value from $00-$7f that changes each frame
    - Stores $79 in $d011
      - y scroll: 1, 25 rows, screen visible, bitmap mode (why does this change from earlier on the screen?), extended bg on
        - I think the point here is to set an illegal mode, which causes the VIC to continue to scan chars normally, but output all black. This creates the solid black line on the top of the plasma, and the left/right edges of the line are covered by sprites so it doesn't go outside of the borders on the side that are made with chars later.
      - Write occurs on line 49 cycle 1/2 (one cycle jitter from before)
    - lda with immediate value that's updated each frame (values seen: $44, $45, $46, $47)
    - Add immediate value to value above (clc, adc #$01, then some cmp/bcc stuff to limit it below $b0)
    - tay
      - Y is some value from $00-$af that changes each frame
  - Stores $3f in $dd02 (port A data direction)
    - This magically changes $dd00 from $4b to $48. I'm not entirely sure how, but this is critical to the display routine; this means the VIC bank is bank 3 at this point ($c000-$ffff), _not_ bank 0!
    - TODO: Find out more details about this port and why this does what it does :)
  - Clear carry
    - Preparation for the next bit which contains lots of ADC's and assumes a clear carry for each iteration
  - Continue in $d011-bashing loop :)

- $d011-bashing loop 1 at $1097-$1dbc (no sprite zoomer on top)
  - Judging by the size of this block of code and that each iteration is 17 bytes long, we get 198 max iterations, which works out with the size of our screen minus one of the two-pixel borders at the top/bottom of the screen.
  - TODO: Calculate loop iteration length in cycles
  - TODO: Inspect data these things are pointing to
  - $d018 values written are always one of the following:
    - $d0 (screen mem: $3400-$37ff, bitmap mem: $0000)
    - $d2 (screen mem: $3400-$37ff, bitmap mem: $0800)
    - $d4 (screen mem: $3400-$37ff, bitmap mem: $1000)
    - $d6 (screen mem: $3400-$37ff, bitmap mem: $1800)
    - $d8 (screen mem: $3400-$37ff, bitmap mem: $2000)
    - $da (screen mem: $3400-$37ff, bitmap mem: $2800)
    - $dc (screen mem: $3400-$37ff, bitmap mem: $3000)
    - $de (screen mem: $3400-$37ff, bitmap mem: $3800)
    - Table arranged as 256 bytes where each 32 bytes repeats $d0 x 4, $d2 x 4, etc.
    - I've tried replacing the table with all the same value ($d0 for example) and the result is that each scanline on the screen becomes identical. This indicates that this is what's switching the graphics out per-line, but it seems strange that there are only 8 possibile lines.
    - Looking further into this, I tried replacing the whole table with `$d0 $d2 $d4 $d6 $d8 $da $dc $de` sequences instead of `$d0 $d0 $d0 $d0 $d1 ...` and the plasma "scales", showing many more lines that still fit together.
  - Each iteration of this loop consists of two parts; one unrolled part, and one subroutine that's called each iteration. In total each iteration is 63 cycles exactly, which I find a bit odd since I would've expected the VIC to steal cycles when doing FPP like this (and displaying sprites for the border, for that matter).. but indeed, this _does_ happen every line.
    - Unrolled part (17 bytes each, 26 cycles):

```
      nop          (2 cycles)
      nop          (2 cycles)
      nop          (2 cycles)
      lda $340z, y (where z appears to increase by 1 each iteration, 4* cycles)
      adc $373z, x (where z appears to increase by 1 each iteration, 4* cycles)
      sta $0ea7    (this addr is part of an address in the subroutine part; same for each iteration, 4 cycles)
      lda #$19     (same for each iteration, 2 cycles)
      jsr $0e9e    (subroutine addr, 6 cycles)
```

    - Subroutine (at $0e9e, 37 cycles):

```
      nop       (2 cycles)
      nop       (2 cycles)
      nop       (2 cycles)
      nop       (2 cycles)
      nop       (2 cycles)
      sta $d011 (always stores $19 from other part on cycle 56 or 57 each line, 4 cycles)
      lda $3de7 (low byte is the part of the addr written to by unrolled part, 4 cycles)
      sta $d018 (changing screen location for FPP surely; TODO: inspect data at $3d00-$3dff to be sure, 4 cycles)
      cmp $00   (timing instruction surely, 3-cycles)
      nop       (2 cycles)
      nop       (2 cycles)
      clc       (clear carry for next iteration, 2 cycles)
      rts       (6 cycles)
```

    - One of the quirks of this routine, however, is the sprite zoomer on top. As the sprites affect raster timing, a different routine is required for lines where the sprite zoomer is visible. To handle this, the unrolled code is modified and a `jmp $4xxx` is inserted over the usual `nop`'s (as both instruction streams are 3 bytes in length) for the line where the switch needs to take place. This appears to jump into the middle of another unrolled part with sprite timing. TODO: Figure out where this second routine starts/stops and what it looks like

```
      jmp $4e13
      lda $344b, y
      adc $3700, x
      sta $0ea7
      jsr $0e9e
```

  - Continue to ??? at $1dbd :)

- ??? at $1dbd-$1e0d
  - Set up X and Y for plasma x-update
    - Here we have a pattern `clc, lda imm1, adc imm2` where imm1 and imm2 determine how the plasma's x stuff will update. Overwriting the code to make these both 0 results in a plasma that doesn't update each frame (minus the stretchy y-movement ofc).
      - imm1 is initialized to some value and imm2 is added to it each frame. In both of the following cases the value is limited further and then written back to imm1.
    - lda with immediate value that's updated each frame
    - Add immediate value to value above (clc, lda #$c8, adc #$fe, sta)
    - Store value in $1dbf (this is read later into X after some subroutines are called)
    - lda with immediate value that's updated each frame
    - Add immediate value to value above (clc, lda #$1c, adc #$01, then some cmp/bcc stuff to limit it below $ba)
    - Store value in $1dc7 (this is read later into Y after some subroutines are called)
  - 4 nop's (timing delay most likely)
  - Store $71 into $d011
    - TODO: Write out bit meanings and figure out which line/cycle(s) this occurs on
  - Store $ff into $d018 (back to what it was at the beginning of the frame)
    - TODO: Write out bit meanings
  - Store $3c into $dd02
    - See note about "Stores $3f in $dd02 (port A data direction)" earlier
  - Small timing delay (lda #$12, sbc #$01, bne * - 3)
  - Store $11 into $d011 (back to what it was at the beginning of the frame)
  - Reset original frame IRQ to $1000 on line $00
  - ACK interrupts
    - asl $d019
    - lda $dc0d
  - Clear interrupt flag
    - This is interesting; are we expecting the interrupt to fire before we're done here?
  - Jumps to subroutine at $29df (sprite zoomer uninit routine)
    - Returns if value at $60 is zero
      - This appears to be 0 or 1 basically and indicates whether or not the sprite zoomer should show
      - TODO: Look for other references to this value (particularly in the sprite zoomer setup routine at $27eb)
    - Sets $d015 to $03 (disables all sprites except 0 and 1, the border sprites)
    - Writes $ea (nop) to 3 bytes starting at the address in $0068-$0069
      - Undoes first raster routine switch SMC in the setup routine
    - Writes $b9 (lda absolute,y) followed by address stored in $0066-$0067 (little-endian) to 3 bytes starting at the address in $006a-$006b
      - Undoes second raster routine switch SMC in the setup routine
  - Jumps to subroutine at $9000 (music update routine)
    - A bit further down ($92b1 onwards) it looks like it's touching SID registers ($d400-$d406 at least), so this is probably the music routine. It's also in a place that's pretty out-of-the-way in memory, which makes that even more likely.
    - Replacing this jsr with nop's freezes the music
  - Jumps to subroutine at $2a09
    - TODO: Figure out what this is, lots of confusing branching going on
    - Haha, replacing this jsr with nop's makes the effect stop changing :D
      - Suddenly the confusing branches make sense: this is frame counting/sequencing code :)
      - Good to know for testing other stuff so I don't have to reload save states as much :)
      - I'd be willing to bet this is a good place to look for effect param/frame counting addresses
  - Continue to ??? at $1e0e :)

- ??? at $1e0e-$25a9
  - This appears to be the first part of the plasma x-update
  - load value at $27e5 into a
    - TODO: Find out where this points to and if whether or not address changes via SMC each frame
  - load X and Y from values updated in previous section ($1dbd-$1dd2)
  - TODO: Tear this bit apart :)

- ??? at $25aa-$27ea
  - This appears to be the second part of the plasma x-update
  - TODO: Initial setup bits (just clc I think)
  - Unrolled loop, 38 iterations
    - Note this is the same as the effect is wide in chars minus the left/rightmost chars, which are used for the left/right borders. Perhaps this single char row is updated each frame?
    - TODO: Tear this bit apart :)
  - TODO: Last bits before/including RTI

- Note that top border ends at line $32 (50), bottom border begins at line ($fa)
  - TODO: Does this apply to both 24 and 25 row screens? I'm assuming it does otherwise opening the borders would shift the screen pos..

- Not sure what's happening outside of the raster IRQ's

- Latest knowns:
  - There are only 8 different charsets for the different plasma lines displayed per frame. These are located at $c000, $c800, $d000, ..., $f800.
  - No sprites are visible in the FPP portion of the screen (except for when the sprite zoomer is showing ofc). The left/right borders are just characters. This means no sprites stealing cycles.
  - Screen mem is at $3c00; only the first row (40 bytes) is used. It would seem the screen and color data for this row is read at the top of the frame, and then the video bank is switched and the FPP stretching starts.
  - The base picture update routine appears to ultimately just modify 38/40 screen mem bytes each frame; I don't think any per-pixel stuff is going on at this point (but I could be wrong).
