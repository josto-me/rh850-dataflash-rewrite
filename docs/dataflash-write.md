# Fine-grained data-flash writes on the Renesas RH850/F1L (R7F7010)

This note explains how to write the **data flash** of a Renesas RH850/F1L
(R7F7010) at the granularity the device actually supports — individual
4-byte words, leaving neighbouring words in the erased state — instead of
being forced to re-write a whole byte image.

It is a plain MCU-programming note: it is only about placing *your own* data
into data flash at the correct granularity. Nothing here is about defeating
any protection mechanism.

## Background: two ways to program the flash

An RH850/F1L device has two independent non-volatile arrays:

- **Code flash** – holds the program.
- **Data flash** – a smaller array intended for parameters, calibration and
  EEPROM-style storage that the application updates at run time.

There are two fundamentally different ways to get bytes into these arrays:

1. **External programming tool (Renesas Flash Programmer, RFP).**
   RFP talks to the on-chip boot firmware over the serial programming
   interface and programs from a memory *image* (`.hex` / `.mot` / `.srec`).
   It is aimed at production programming of a complete, contiguous region:
   you hand it an image, it erases and programs that whole region. In this
   workflow every byte position in the programmed range is written; it is not
   the right tool for "write these four bytes here and leave the rest of the
   block blank". If the image describes a full byte for an address, that byte
   is programmed.

2. **On-chip self-programming libraries.**
   Renesas ships firmware libraries that run *on the RH850 itself* and drive
   the flash sequencer directly. These expose the array's native erase and
   write granularity to the application:
   - **Code Flash Library (FCL / Type T01)** for the code flash.
   - **Data Flash Library (FDL)** and the **EEPROM Emulation Library (EEL)**
     built on top of it, for the data flash.

   The self-programming route is what lets you write a single 4-byte word and
   leave the other words of the block untouched (erased).

## Data-flash geometry and write granularity

For the RH850/F1L data flash:

- The data flash is divided into **blocks of 64 bytes**.
- **Erase** works on a **whole block** (64 bytes) at a time — you cannot erase
  a single word.
- **Write** works in **word units of 4 bytes** (one 32-bit word). This is the
  smallest amount you can program in one write operation.

Because erase is per-block and write is per-word, you can erase a block once
and then program its sixteen 4-byte words independently. Words you never write
after the erase simply stay in the **erased (blank)** state; a word only leaves
the blank state when you explicitly write it.

This is the key difference from the image-programming workflow: with the
library you decide, word by word, what gets written and what is left blank.
With a full-image tool the whole programmed range ends up written.

## Reading back the erased state

An erased data-flash word is *blank*. Do not rely on a particular read
value for a blank word and compare against it in application code; instead use
the library's **blank-check** service to ask the flash sequencer whether a
region is still erased. The blank-check and the write/erase operations return a
status (busy / OK / error) that the caller polls or waits on.

## Typical FDL call sequence

The exact symbol names depend on the library version, but the shape is always
the same. A minimal "erase one block, then write one 4-byte word" sequence
looks like:

```
1.  FDL_Init(&fdl_descriptor)      // one-time init: clock, config, RAM area
2.  FDL_Execute(&erase_request)    // erase the target 64-byte block
    poll FDL_Handler() until status != BUSY   // e.g. FDL_OK
3.  FDL_Execute(&write_request)    // write request: address = word offset,
                                   // length = number of 4-byte words,
                                   // pointer = source data in RAM
    poll FDL_Handler() until status != BUSY   // e.g. FDL_OK
4.  (optional) FDL_Execute(&blankcheck_request) / verify
```

Notes that matter in practice:

- The **address/offset in a write request is expressed in words**, and the
  **length is a word count**, not a byte count. Passing byte values where the
  library expects word units is the classic first mistake.
- The write source buffer must live in **RAM** and stay valid until the
  operation completes; the flash library code and its work area typically must
  also run from RAM, not from the flash being programmed.
- Erase and write are **asynchronous**: kick them off with `FDL_Execute`, then
  drive `FDL_Handler` (or the equivalent) until the request reports done.
  Interrupts that touch the flash controller must be handled per the library
  manual during the operation.
- Only whole 4-byte words can be written; to change a single byte you write the
  4-byte word that contains it (after the block has been erased).

## When you need this

You need the self-programming / FDL route (rather than an image programmer)
whenever you want to:

- update **individual parameters/words** in data flash without re-flashing the
  whole array, or
- deliberately leave some words of a block in the **erased/blank** state while
  writing others (for example to keep spare words free for later updates or
  for a simple wear-levelling scheme in your own application).

## References

- Renesas Engineering Community — *"How to execute a 4-byte write on data
  flash of a R7F7010 RH850/F1L?"*
  <https://community.renesas.com/mcu/rh850/f/rh850-forum/51314/how-to-execute-a-4-byte-write-on-data-flash-of-a-r7f7010-rh850-f1l>
- Renesas — *RH850 Family Data Flash Libraries, User's Manual* (document
  R01US0079ED). Describes FDL/EEL, block/word granularity and the request/
  handler API.
- Renesas — *RH850 Family Code Flash Libraries (Type T01), User's Manual*
  (document R01US0078ED). The code-flash counterpart (FCL).
- Renesas RH850/F1L device documentation (hardware user's manual and data
  sheet) for the exact data-flash size and block map of the R7F7010.

Symbol names, exact request-structure fields and status codes vary between
library versions — always follow the user's manual for the FDL/FCL version you
link against.
