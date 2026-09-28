# Compression

From my knowledge, Need for speed games use 5 types of compression (I'm not counting "RAW" as a compression type, so technicaly it's 6):

- [HUFF](./HUFF.md)
- [JDLZ](./JDLZ.md)
- [REF](./REF.md)
- [BTREE](./BTREE.md)
- [COMP](./COMP.md)
- [RAWW](./RAWW.md)

All those compression have specific id's, all of them are a 4 byte ASCII string :

```
enum CompressionType {
    HUFF = 0x48554646,
    JDLZ = 0x4A444C5A,
    REF = 0x5246504B,
    BTREE = 0x42545245,
    COMP = 0x434F4D50,
    RAWW = 0x52415757
}
```

Usually, they use those compression when packing assets for the final game, and they compress part of binary files, often meshes.

> Personnaly, I think they just run all of these algorithms, and based on the size of their output, pick the one which performs the best.
> Those codec were made for low complexity in time and in space for hardware with as little as 32Mb of RAM (for exemple the PS2 had 32Mb of RAM and 4Mb of VRAM as described [here](https://en.wikipedia.org/wiki/PlayStation_2#Hardware))

[Someone exceptionnal](https://github.com/RayneDuarte/EAC/tree/main) made a github repo containing all of the original compression AND decompression algorithm, probably found in all the released [Electronic Arts](https://github.com/ElectronicArts) repo. Also, the EA organization contains amazing repositories.
