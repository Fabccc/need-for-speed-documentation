# Compression type JDLZ

JDLZ is an implementation of dictionnary compression, derived from LZ77.
JD might be initials of engineer(s) working on the project.

It works with 2 flags that acts as state machine.
Flag1 is for dicting wherever the next byte is literal data
Flag2 is for dicting how to handle the data.

## Decompression

Pseudo code decompression algorithm

```c
uint8* decompress(uint8* in, usize len, usize expected_decompressed_size) {
    uint8* out = (uint8*) allocate_memory(expected_decompressed_size);
    usize inPos = 0;
    usize inEnd = inPos + len;
    usize outPos = 0;
    int32 flag1 = 1;
    int32 flag2 = 1;

    while(inPos < inEnd && pos < expected_decompressed_size){
        if(flags1 == 1){
            if(inPos >= inEnd){
                break;
            }
            flags1 = (in[inPos++] & 0xFF) | 0x100;
        }
        if(flags2 == 1){
            if(inPos >= inEnd){
                break;
            }
            flags2 = (in[inPos++] & 0xFF) | 0x100;
        }

        if ((flags1 & 1) != 0) {
            if (inPos + 1 >= inEnd) {
                error("Invalid JDLZ stream (truncated)");
            }
            uint8 b0 = in[inPos++] & 0xFF;
            uint8 b1 = in[inPos++] & 0xFF;
            usize length;
            usize dist;

            if ((flags2 & 1) != 0) {
                length = (b1 | ((b0 & 0xF0) << 4)) + 3; // 12 bits read (2^12 max val)
                dist = (b0 & 0x0F) + 1; // 4 bits read (2^4 max val)
            } else {
                dist = (b1 | ((b0 & 0xE0) << 3)) + 17;
                length = (b0 & 0x1F) + 3;
            }

            if (outPos - dist < 0) {
                error("Invalid JDLZ stream (broken back reference)");
            }

            usize copylimit = min(length, expected_decompressed_size - outPos);
            for (int i = 0; i < copylimit; i++) {
                out[outPos] = out[outPos - dist];
                outPos++;
            }
            flags2 >>>= 1;
        } else {
            if (outPos < expected_decompressed_size) {
                if (inPos >= inEnd) {
                    error("Invalid JDLZ stream (truncated)");
                }
                out[outPos++] = in[inPos++];
            }
        }
        flags1 >>>= 1;
    }
    return out;
}
```


## Compression
