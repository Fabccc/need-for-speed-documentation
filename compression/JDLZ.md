# Compression type JDLZ

JDLZ is an implementation of dictionnary compression, derived from LZ77. JD
might be initials of engineer(s) working on the project.

It works with 2 flags that acts as state machine. Flag1 is for dicting wherever
the next byte is literal data Flag2 is for dicting how to handle the data.

## JDLZ Header Structure (16 bytes, Little-Endian)

| Offset |         Type         | Field Name           | Description / Expected Value                                      |
| :----: | :------------------: | :------------------- | :---------------------------------------------------------------- |
| `0x00` | `char[4]` / `uint32` | `magic`              | Magic signature `"JDLZ"` (`0x5A4C444A` in LE)                     |
| `0x04` |       `uint8`        | `version`            | Codec version (In need for speed carbon, its `0x02`)              |
| `0x05` |       `uint8`        | `header_size`        | Size of this header in bytes, always `0x10` (16 bytes)            |
| `0x06` |       `uint16`       | `reserved` / `flags` | Padding/flags, typically `0x0000`                                 |
| `0x08` |       `uint32`       | `decompressed_size`  | Size of the uncompressed data in bytes                            |
| `0x0C` |       `uint32`       | `compressed_size`    | Total size of the compressed block (including the 16-byte header) |

## JDLZ v0x02

### Decompression

Pseudo code decompression algorithm

```c
uint8* decompress(const uint8* in, usize in_len, usize expected_decompressed_size) {
    if (expected_decompressed_size == 0) {
        return allocate_memory(0);
    }
    // 1. Memory Allocation
    // Allocate the destination buffer upfront based on the header's uncompressed size
    uint8* out = (uint8*) allocate_memory(expected_decompressed_size);
    usize in_pos = 0;
    usize in_end = in_len;
    usize out_pos = 0;
    // 2. Control Registers State (9-bit shift registers with sentinel bit 0x100)
    uint32 flags1 = 1;
    uint32 flags2 = 1;
    // 3. Main Decompression Loop
    while (in_pos < in_end && out_pos < expected_decompressed_size) {
        
        // Reload Flags 1 (Literal vs Match decision) when all 8 bits have been consumed
        if (flags1 == 1) {
            if (in_pos >= in_end) break;
            flags1 = ((uint32) in[in_pos++]) | 0x100;
        }
        // Reload Flags 2 (Near vs Far match mode) when all 8 bits have been consumed
        if (flags2 == 1) {
            if (in_pos >= in_end) break;
            flags2 = ((uint32) in[in_pos++]) | 0x100;
        }
        // --- BRANCH A: BACK-REFERENCE MATCH (LZ77) ---
        if ((flags1 & 1) != 0) {
            // Require 2 bytes payload [b0, b1] to decode distance and length
            if (in_pos + 1 >= in_end) {
                error("Invalid JDLZ stream: truncated reference command");
            }
            uint32 b0 = (uint32) in[in_pos++];
            uint32 b1 = (uint32) in[in_pos++];
            usize length;
            usize dist;
            if ((flags2 & 1) != 0) {
                // Mode 1: Near / RLE Match
                // - Distance: 4 bits in b0 [3:0] (Range: 1 .. 16)
                // - Length:   12 bits split across b0 [7:4] and b1 [7:0] (Range: 3 .. 4098)
                length = (b1 | ((b0 & 0xF0) << 4)) + 3;
                dist   = (b0 & 0x0F) + 1;
            } else {
                // Mode 2: Far / Standard Match
                // - Distance: 11 bits split across b0 [7:5] and b1 [7:0] + 17 (Range: 17 .. 2064)
                // - Length:   5 bits in b0 [4:0] (Range: 3 .. 34)
                dist   = (b1 | ((b0 & 0xE0) << 3)) + 17;
                length = (b0 & 0x1F) + 3;
            }
            // Boundary Check: Ensure back-reference doesn't look back before start of buffer
            if (out_pos < dist) {
                error("Invalid JDLZ stream: negative back-reference distance");
            }
            // Copy loop: must copy byte-by-byte to allow RLE overlapping patterns (when length > dist)
            usize copy_limit = min(length, expected_decompressed_size - out_pos);
            for (usize i = 0; i < copy_limit; i++) {
                out[out_pos] = out[out_pos - dist];
                out_pos++;
            }
            // Consume one bit of Flags 2 (only consumed when a match occurs)
            flags2 >>= 1;
        } 
        // --- BRANCH B: LITERAL BYTE ---
        else {
            // Copy raw uncompressed byte directly from input to output stream
            if (out_pos < expected_decompressed_size) {
                if (in_pos >= in_end) {
                    error("Invalid JDLZ stream: truncated literal byte");
                }
                out[out_pos++] = in[in_pos++];
            }
        }
        // Consume one bit of Flags 1 (always consumed at each iteration)
        flags1 >>= 1;
    }
    // 4. Sanity Check
    if (out_pos != expected_decompressed_size) {
        error("Invalid JDLZ stream: decompressed size mismatch");
    }
    return out;
}
```

### Compression

```c
const HEADER_SIZE = 16;
const HASH_SIZE = 8192;
const MAX_SEARCH_DEPTH = 16;
const MIN_MATCH_LENGTH = 3;

uint8* compress(const uint8* in, usize in_len, usize* out_len) {
    if (in_len == 0) {
        uint8* empty_out = (uint8*) allocate_memory(HEADER_SIZE);
        write_header(empty_out, 0, HEADER_SIZE);
        *out_len = HEADER_SIZE;
        return empty_out;
    }

    // Allocate hash table & chained hash positions
    int32 hash_pos[HASH_SIZE];
    for (usize i = 0; i < HASH_SIZE; i++) hash_pos[i] = -1;

    int32* hash_chain = (int32*) allocate_memory(in_len * sizeof(int32));
    for (usize i = 0; i < in_len; i++) hash_chain[i] = -1;

    // Allocate conservative output buffer
    usize out_capacity = in_len * 2 + HEADER_SIZE + 64;
    uint8* out = (uint8*) allocate_memory(out_capacity);
    usize out_pos = 0;
    usize in_pos = 0;
    usize input_bytes_left = in_len;

    // 1. Write JDLZ 16-byte Header
    write_header(out, in_len, 0); // Placeholder 0 for compressed size
    out_pos += HEADER_SIZE;

    // 2. Control Flags State (Sentinels & Backpatching offsets)
    usize flags1_pos = out_pos++;
    usize flags2_pos = out_pos++;
    uint32 flags1_bit = 1;
    uint32 flags2_bit = 1;
    uint8  flags1 = 0;
    uint8  flags2 = 0;

    // First byte is always emitted as an uncompressed literal
    flags1_bit <<= 1;
    out[out_pos++] = in[in_pos++];
    input_bytes_left--;

    // 3. Main LZ Search & Encoding Loop
    while (input_bytes_left > 0) {
        usize best_match_len = MIN_MATCH_LENGTH - 1;
        usize best_match_dist = 0;

        // Search for matches in sliding window using the hash chain
        if (input_bytes_left >= MIN_MATCH_LENGTH) {
            uint8 b0 = in[in_pos];
            uint8 b1 = in[in_pos + 1];
            uint8 b2 = in[in_pos + 2];
            uint32 hash = (-0x1A1 * (b0 ^ ((b1 ^ (b2 << 4)) << 4))) & (HASH_SIZE - 1);

            int32 match_pos = hash_pos[hash];
            hash_pos[hash] = (int32) in_pos;
            hash_chain[in_pos] = match_pos;
            int32 prev_match_pos = (int32) in_pos;

            for (usize depth = 0; depth < MAX_SEARCH_DEPTH; depth++) {
                if (match_pos < 0 || match_pos >= prev_match_pos) break;

                usize match_dist = in_pos - (usize) match_pos;
                if (match_dist > 2064) break;

                usize match_limit = (match_dist <= 16) ? 4098 : 34;
                usize max_match_len = min(input_bytes_left, match_limit);
                if (best_match_len >= max_match_len) break;

                // Compare byte sequences
                usize match_len = 0;
                while (match_len < max_match_len && in[in_pos + match_len] == in[match_pos + match_len]) {
                    match_len++;
                }

                if (match_len > best_match_len) {
                    best_match_len = match_len;
                    best_match_dist = match_dist;
                }

                prev_match_pos = match_pos;
                match_pos = hash_chain[match_pos];
            }
        }

        // 4. Emit Token (Match vs Literal)
        if (best_match_len >= MIN_MATCH_LENGTH) {
            // Flag1 = 1 indicates a Back-Reference
            flags1 |= flags1_bit;
            in_pos += best_match_len;
            input_bytes_left -= best_match_len;
            best_match_len -= MIN_MATCH_LENGTH;

            if (best_match_dist <= 16) {
                // Near match: Flag2 = 1 (Dist: [1..16], Len: [3..4098])
                flags2 |= flags2_bit;
                out[out_pos++] = (uint8) (((best_match_dist - 1) & 0x0F) | ((best_match_len >> 4) & 0xF0));
                out[out_pos++] = (uint8) (best_match_len & 0xFF);
            } else {
                // Far match: Flag2 = 0 (Dist: [17..2064], Len: [3..34])
                best_match_dist -= 17;
                out[out_pos++] = (uint8) ((best_match_len & 0x1F) | ((best_match_dist >> 3) & 0xE0));
                out[out_pos++] = (uint8) (best_match_dist & 0xFF);
            }
            flags2_bit <<= 1;
        } else {
            // Literal Byte (Flag1 = 0)
            out[out_pos++] = in[in_pos++];
            input_bytes_left--;
        }

        flags1_bit <<= 1;

        // 5. Flush completed flag bytes (8 bits accumulated)
        if ((flags1_bit & 0x100) != 0) {
            out[flags1_pos] = flags1;
            flags1 = 0;
            flags1_pos = out_pos++;
            flags1_bit = 1;
        }
        if ((flags2_bit & 0x100) != 0) {
            out[flags2_pos] = flags2;
            flags2 = 0;
            flags2_pos = out_pos++;
            flags2_bit = 1;
        }
    }

    // 6. Flush trailing partial flag bytes
    if (flags2_bit > 1) {
        out[flags2_pos] = flags2;
    } else if (flags2_pos == out_pos - 1) {
        out_pos--; // Reclaim unused trailing byte
    }

    if (flags1_bit > 1) {
        out[flags1_pos] = flags1;
    } else if (flags1_pos == out_pos - 1) {
        out_pos--; // Reclaim unused trailing byte
    }

    // 7. Backpatch final compressed size into Header at offset 0x0C
    write_uint32_le(&out[12], (uint32) out_pos);

    free_memory(hash_chain);
    *out_len = out_pos;
    return out;
}
```