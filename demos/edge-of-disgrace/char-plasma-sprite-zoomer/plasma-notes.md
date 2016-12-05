- Note that top border ends at line $32 (50), bottom border begins at line ($fa)
  - TODO: Does this apply to both 24 and 25 row screens? I'm assuming it does otherwise opening the borders would shift the screen pos..

- General stuff being displayed:
  - Plasma in char area (38 chars x 25 lines)
  - Star as sprite zoomer
  - Black border around screen

- Initial frame raster IRQ: $1000 on line 0
  - IO regs:
    - $dd00: $4b (VIC bank 0 ($0000-$3fff))
    - $d011: $11 (y scroll: 1, 24 rows, screen visible, text mode, extended bg off)
      - Note 24 rows when effect is 25 rows high
        - It also seems like the top or bottom border takes up two of these rows, and the other border makes the screen 2 pixels taller (each border is 2px high)
    - $d016: $d8
    - $d018: $ff
    - Interesting that these don't change even though the plasma appears to be doing FPP
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
    - TODO: Find out what this does
  - Sets up another raster IRQ at $1040 on line $2f (47)
  - ACK interrupts
    - asl $d019
    - lda $dc0d (Interrupt control + status reg), does nothing with this value (?)
  - Restores Y, A, X from $e0-$e2

- Raster IRQ at $1040-$27ea on line $2f (47)
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

- Semi-stable IRQ at $1062 on line $30 (48) (occurs at cycle 9 or 10; one cycle jitter)
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
      - Write occurs on line 49 cycle 1/2 (one cycle jitter from before)
    - lda with immediate value that's updated each frame (values seen: $44, $45, $46, $47)
    - Add immediate value to value above (clc, adc #$01, then some cmp/bcc stuff to limit it below $b0)
    - tay
      - Y is some value from $00-$af that changes each frame
  - Stores $ef in $dd02 (port A data direction)
    - TODO: No idea what this is for :) might be shutting down the loader
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
  - Each iteration of this loop consists of two parts; one unrolled part, and one subroutine that's called each iteration. In total each iteration is 63 cycles exactly, which I find a bit odd since I would've expected the VIC to steal cycles when doing FPP like this (and displaying sprites for the border, for that matter)..
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
      sta $d011 (always stores $19 from other part; TODO: Which cycle(s) does the write occur at? 4 cycles)
      lda $3de7 (low byte is the part of the addr written to by unrolled part, 4 cycles)
      sta $d018 (changing screen location for FPP surely; TODO: inspect data at $3d00-$3dff to be sure, 4 cycles)
      cmp $00   (timing instruction surely, 3-cycles)
      nop       (2 cycles)
      nop       (2 cycles)
      clc       (clear carry for next iteration, 2 cycles)
      rts       (6 cycles)
```

    - One of the quirks of this routine, however, is the sprite zoomer on top. As the sprites affect raster timing, a different routine is required for lines where the sprite zoomer is visible. To handle this, the unrolled code is modified and a `jmp $4xxx` is inserted over the usual `nop`'s (as both instruction streams are 3 bytes in length) for the line where the switch needs to take place. This appears to jump into the middle of another unrolled part with sprite timing. TODO: Check to see if this other code is modified as well to jump back into the first piece of speedcode. If this is the case, it should be possible to get rid of the sprite zoomer and isolate the plasma.

```
      jmp $4e13
      lda $344b, y
      adc $3700, x
      sta $0ea7
      jsr $0e9e
```

  - Continue to ??? at $1dbd :)

- ??? at $1dbd

- Not sure what's happening outside of the raster IRQ's