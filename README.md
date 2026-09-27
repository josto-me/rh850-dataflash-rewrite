# RH850 Data Flash Rewrite

[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.23002808-blue.svg)](https://doi.org/10.5281/zenodo.23002808) [![Docs: CC BY 4.0](https://img.shields.io/badge/docs-CC%20BY%204.0-lightgrey.svg)](LICENSE) [![Cite](https://img.shields.io/badge/cite-CITATION.cff-green.svg)](CITATION.cff)

*Data-Flash feingranular beschreiben am Renesas RH850/F1L (R7F7010).*

A short technical note on how to write the **data flash** of a Renesas
RH850/F1L (R7F7010) at the granularity the hardware actually supports —
individual 4-byte words, with unwritten words left in the erased state —
using the on-chip self-programming libraries instead of a full-image
programmer.

## The problem

A general-purpose flash programming tool (such as the Renesas Flash
Programmer) programs the device from a memory image and writes every byte in
the targeted region. That is fine for putting a complete image on the part,
but it cannot express "write these four bytes and leave the rest of the block
blank": whatever the image covers gets written as full bytes.

The RH850/F1L data flash, however, natively supports:

- **erase** in units of one 64-byte block, and
- **write** in units of one 4-byte word,

so words that are never written after an erase simply stay blank. Reaching
that granularity requires the on-chip **Data Flash Library (FDL)** /
**EEPROM Emulation Library (EEL)** rather than an external image programmer.

## Contents

```
README.md                         this file
docs/dataflash-write.md     the technical note (geometry, granularity,
                                  FDL call sequence, gotchas, references)
LICENSE                           CC BY 4.0 (documentation)
.gitattributes / .gitignore
```

## Summary of the technique

- Data flash = 64-byte blocks; erase per block, write per 4-byte word.
- Use the FDL/EEL self-programming libraries to erase a block once and then
  program its words independently; unwritten words remain in the erased
  (blank) state.
- Write requests address memory in **word units** (4 bytes), with a word
  count as length; the source buffer and library work area live in RAM.
- Erase/write/blank-check are asynchronous — start them and poll the handler
  until the operation reports done.

See [`docs/dataflash-write.md`](docs/dataflash-write.md) for the
details and for the Renesas references (community thread and the FDL/FCL
user's manuals).

## License

- Everything in this repository (documentation): **CC BY 4.0**, see [`LICENSE`](LICENSE).

You may use, change and share everything, also commercially. When you pass it on or
publish something based on it, credit it as:

> Johannes Stockhammer, "RH850 Data Flash Rewrite", version 1.0.1, Zenodo, https://doi.org/10.5281/zenodo.23002808

GitHub shows the same citation under "Cite this repository" (from [`CITATION.cff`](CITATION.cff)).

## Trademarks

Renesas and RH850 are trademarks of Renesas Electronics Corporation. This note is independent and not endorsed by Renesas.

## Author

Johannes Stockhammer

Method, technical content and text by Johannes Stockhammer. Translation and text were refined with the help of AI tools and reviewed by the author.
