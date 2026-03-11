## Binary Format Codec Reference (EA Source Code)

> All struct definitions in this section are taken verbatim from the GPL v3 EA source code repositories:
> - [CnC_Remastered_Collection](https://github.com/electronicarts/CnC_Remastered_Collection) — primary source (REDALERT/ and TIBERIANDAWN/ directories)
> - [CnC_Red_Alert](https://github.com/electronicarts/CnC_Red_Alert) — VQA/VQ video format definitions (VQ/ and WINVQ/ directories)
>
> These are the authoritative definitions for `ra-formats` crate implementation. Field names, sizes, and types must match exactly for binary compatibility.

### MIX Archive Format (.mix)

**Source:** `REDALERT/MIXFILE.H`, `REDALERT/MIXFILE.CPP`, `REDALERT/CRC.H`, `REDALERT/CRC.CPP`

A MIX file is a flat archive. Files are identified by CRC hash of their filename — there is no filename table in the archive.

#### File Layout

```
[optional: 2-byte zero flag + 2-byte flags word]  // Extended format only
[FileHeader]                                       // 6 bytes
[SubBlock array]                                   // sorted by CRC for binary search
[file data]                                        // concatenated file bodies
```

#### Structures

```c
// Archive header (6 bytes)
typedef struct {
    short count;    // Number of files in the archive
    long  size;     // Total size of all file data (bytes)
} FileHeader;

// Per-file index entry (12 bytes)
struct SubBlock {
    long CRC;       // CRC hash of uppercase filename
    long Offset;    // Byte offset from start of data section
    long Size;      // File size in bytes
};
```

**Extended format detection:** If the first `short` read is 0, the next `short` is a flags word:
- Bit `0x0001` — archive contains SHA-1 digest
- Bit `0x0002` — archive header is encrypted (Blowfish)

When neither flag is set, the first `short` is the file count and the archive uses the basic format.

#### CRC Filename Hashing Algorithm

```c
// From CRC.H / CRC.CPP — CRCEngine
// Accumulates bytes in a 4-byte staging buffer, then:
//   CRC = _lrotl(CRC, 1) + *longptr;
// (rotate CRC left 1 bit, add next 4 bytes as a long)
//
// Filenames are converted to UPPERCASE before hashing.
// Partial final bytes (< 4) are accumulated into the staging buffer
// and the final partial long is added the same way.
```

The SubBlock array is sorted by CRC to enable binary search lookup at runtime.

#### Encrypted Header Handling (Extended Format)

**Source:** `REDALERT/MIXFILE.CPP` — `MixFileClass::Retrieve()` and key initialization

Some original RA `.mix` files ship with Blowfish-encrypted header indexes (flag `0x0002`). Notable examples: `local.mix`, `speech.mix`. Full resource compatibility (Invariant #8) requires decryption support.

**What is encrypted:** Only the `SubBlock` index array (the file directory). File data bodies are always plaintext. The Blowfish cipher operates in ECB mode on 8-byte blocks.

**Key source:** Westwood used a hardcoded 56-byte Blowfish key embedded in every game binary. This key is public knowledge — documented by XCC Utilities, OpenRA (`MixFile.cs`), and numerous community tools since the early 2000s. `cnc-formats` embeds this key as a constant. The key is not copyrightable (it is a number) and the decrypt algorithm is standard Blowfish — no EA-specific code is needed.

**Decryption steps:**
1. Read the 2-byte zero marker + 2-byte flags word (4 bytes total)
2. If flag `0x0002` is set, the next N bytes are the Blowfish-encrypted header
3. Decrypt using the hardcoded key in ECB mode (8-byte blocks)
4. The decrypted output contains the `FileHeader` (6 bytes) followed by the `SubBlock` array
5. Validate decrypted `FileHeader.count` and `FileHeader.size` against sane limits (V38 entry caps) before allocating the `SubBlock` array

**SHA-1 digest (flag `0x0001`):** When present, a 20-byte SHA-1 digest follows the file data section. It covers the unencrypted body data. Verification is optional but recommended for integrity checking of original archives.

**Implementation crate:** Use the `blowfish` crate from RustCrypto (MIT/Apache-2.0) for the Blowfish primitive. Do not reimplement the cipher.

**Security:** Blowfish decryption of untrusted input is a parsing step — see V38 in `security/vulns-infrastructure.md` for defensive parsing requirements (validate decrypted header values before allocation, reject malformed ciphertext cleanly).

---

### SHP Sprite Format (.shp)

**Source:** `REDALERT/WIN32LIB/SHAPE.H`, `REDALERT/2KEYFRAM.CPP`, `TIBERIANDAWN/KEYFRAME.CPP`

SHP files contain one or more palette-indexed sprite frames. Individual frames are typically LCW-compressed.

#### Shape Block (Multi-Frame Container)

```c
// From SHAPE.H — container for multiple shapes
typedef struct {
    unsigned short NumShapes;   // Number of shapes in block
    long           Offsets[];   // Variable-length array of offsets to each shape
} ShapeBlock_Type;
```

#### Single Shape Header

```c
// From SHAPE.H — header for one shape frame
typedef struct {
    unsigned short ShapeType;       // Shape type flags (see below)
    unsigned char  Height;          // Height in scan lines
    unsigned short Width;           // Width in bytes
    unsigned char  OriginalHeight;  // Original (unscaled) height
    unsigned short ShapeSize;       // Total size including header
    unsigned short DataLength;      // Size of uncompressed data
    unsigned char  Colortable[16];  // Color remap table (compact shapes only)
} Shape_Type;
```

#### Keyframe Animation Header (Multi-Frame SHP)

```c
// From 2KEYFRAM.CPP — header for keyframe animation files
typedef struct {
    unsigned short frames;              // Number of frames
    unsigned short x;                   // X offset
    unsigned short y;                   // Y offset
    unsigned short width;               // Frame width
    unsigned short height;              // Frame height
    unsigned short largest_frame_size;  // Largest single frame (for buffer allocation)
    unsigned short flags;               // Bit 0 = has embedded palette (768 bytes after offsets)
} KeyFrameHeaderType;
```

When `flags & 1`, a 768-byte palette (256 × RGB) follows immediately after the frame offset table. Retrieved via `Get_Build_Frame_Palette()`.

#### Shape Type Flags (MAKESHAPE)

| Value    | Name     | Meaning                            |
| -------- | -------- | ---------------------------------- |
| `0x0000` | NORMAL   | Standard shape                     |
| `0x0001` | COMPACT  | Uses 16-color palette (Colortable) |
| `0x0002` | NOCOMP   | Uncompressed pixel data            |
| `0x0004` | VARIABLE | Variable-length color table (<16)  |

#### Drawing Flags (Runtime)

| Value    | Name           | Effect                             |
| -------- | -------------- | ---------------------------------- |
| `0x0000` | SHAPE_NORMAL   | No transformation                  |
| `0x0001` | SHAPE_HORZ_REV | Horizontal flip                    |
| `0x0002` | SHAPE_VERT_REV | Vertical flip                      |
| `0x0004` | SHAPE_SCALING  | Apply scale factor                 |
| `0x0020` | SHAPE_CENTER   | Draw centered on coordinates       |
| `0x0100` | SHAPE_FADING   | Apply fade/remap table             |
| `0x0200` | SHAPE_PREDATOR | Predator-style cloaking distortion |
| `0x0400` | SHAPE_COMPACT  | Shape uses compact color table     |
| `0x1000` | SHAPE_GHOST    | Ghost/transparent rendering        |
| `0x2000` | SHAPE_SHADOW   | Shadow rendering mode              |

---

### LCW Compression

**Source:** `REDALERT/LCW.CPP`, `REDALERT/LCWUNCMP.CPP`, `REDALERT/WIN32LIB/IFF.H`

LCW (Lempel-Castle-Welch) is Westwood's primary data compression algorithm, used for SHP frame data, VQA video chunks, icon set data, and other compressed resources.

#### Compression Header Wrapper

```c
// From IFF.H — optional header wrapping compressed data
typedef struct {
    char  Method;   // Compression method (see CompressionType)
    char  pad;      // Padding byte
    long  Size;     // Decompressed size
    short Skip;     // Bytes to skip
} CompHeaderType;

typedef enum {
    NOCOMPRESS  = 0,
    LZW12       = 1,
    LZW14       = 2,
    HORIZONTAL  = 3,
    LCW         = 4
} CompressionType;
```

#### LCW Command Opcodes

LCW decompression processes a source stream and produces output by copying literals, referencing previous output (sliding window), or filling runs:

| Byte Pattern            | Name           | Operation                                                        |
| ----------------------- | -------------- | ---------------------------------------------------------------- |
| `0b0xxx_yyyy, yyyyyyyy` | Short copy     | Copy run of `x+3` bytes from `y` bytes back in output (relative) |
| `0b10xx_xxxx, n₁..nₓ₊₁` | Medium literal | Copy next `x+1` bytes verbatim from source to output             |
| `0b11xx_xxxx, w₁`       | Medium copy    | Copy `x+3` bytes from absolute output offset `w₁`                |
| `0xFF, w₁, w₂`          | Long copy      | Copy `w₁` bytes from absolute output offset `w₂`                 |
| `0xFE, w₁, b₁`          | Long run       | Fill `w₁` bytes with value `b₁`                                  |
| `0x80`                  | End marker     | End of compressed data                                           |

Where `w₁`, `w₂` are little-endian 16-bit words and `b₁` is a single byte.

**Key detail:** Short copies use *relative* backward references (from current output position), while medium and long copies use *absolute* offsets from the start of the output buffer. This dual addressing is a distinctive feature of LCW.

> **Security (V38):** All `ra-formats` decompressors (LCW, LZ4, ADPCM) must enforce decompression ratio caps (256:1), absolute output size limits, and loop iteration counters. Every format parser must have a `cargo-fuzz` target. Archive extraction (`.oramap` ZIP) must use `strict-path` `PathBoundary` to prevent Zip Slip. See `06-SECURITY.md` § Vulnerability 38.

#### IFF Chunk ID Macro

```c
// From IFF.H — used by MIX, icon set, and other IFF-based formats
#define MAKE_ID(a,b,c,d) ((long)((long)d << 24) | ((long)c << 16) | ((long)b << 8) | (long)(a))
```

---

### TMP Terrain Tile Format (.tmp / Icon Sets)

**Source:** `REDALERT/WIN32LIB/TILE.H`, `TIBERIANDAWN/WIN32LIB/TILE.H`, `*/WIN32LIB/ICONSET.CPP`, `*/WIN32LIB/STAMP.INC`, `REDALERT/COMPAT.H`

TMP files are **IFF-format icon sets** — collections of fixed-size tiles arranged in a grid. Each tile is a 24×24 pixel palette-indexed bitmap. The engine renders terrain by compositing these tiles onto the map.

#### On-Disk IFF Chunk Structure

TMP files use Westwood's IFF variant with these chunk identifiers:

| Chunk ID | FourCC                     | Purpose                                      |
| -------- | -------------------------- | -------------------------------------------- |
| `ICON`   | `MAKE_ID('I','C','O','N')` | Form identifier (file magic — must be first) |
| `SINF`   | `MAKE_ID('S','I','N','F')` | Set info: icon dimensions and format         |
| `SSET`   | `MAKE_ID('S','S','E','T')` | Icon pixel data (may be LCW-compressed)      |
| `TRNS`   | `MAKE_ID('T','R','N','S')` | Per-icon transparency flags                  |
| `MAP `   | `MAKE_ID('M','A','P',' ')` | Icon mapping table (logical → physical)      |
| `RPAL`   | `MAKE_ID('R','P','A','L')` | Icon palette                                 |
| `RTBL`   | `MAKE_ID('R','T','B','L')` | Remap table                                  |

#### SINF Chunk (Icon Dimensions)

```c
// Local struct in Load_Icon_Set() — read from SINF chunk
struct {
    char Width;      // Width of one icon in bytes (pixels = Width << 3)
    char Height;     // Height of one icon in bytes (pixels = Height << 3)
    char Format;     // Graphic mode
    char Bitplanes;  // Number of bitplanes per icon
} sinf;

// Standard RA value: Width=3, Height=3 → 24×24 pixels (3 << 3 = 24)
// Bytes per icon = ((Width<<3) * (Height<<3) * Bitplanes) >> 3
// For 24×24 8-bit: (24 * 24 * 8) >> 3 = 576 bytes per icon
```

#### In-Memory Control Structure

The IFF chunks are loaded into a contiguous memory block with `IControl_Type` as the header. **Two versions exist** — Tiberian Dawn and Red Alert differ:

```c
// Tiberian Dawn version (TIBERIANDAWN/WIN32LIB/TILE.H)
typedef struct {
    short           Width;      // Width of icons (pixels)
    short           Height;     // Height of icons (pixels)
    short           Count;      // Number of (logical) icons in this set
    short           Allocated;  // Was this iconset allocated? (runtime flag)
    long            Size;       // Size of entire iconset memory block
    unsigned char * Icons;      // Offset from buffer start to icon data
    long            Palettes;   // Offset from buffer start to palette data
    long            Remaps;     // Offset from buffer start to remap index data
    long            TransFlag;  // Offset for transparency flag table
    unsigned char * Map;        // Icon map offset (if present)
} IControl_Type;
// Note: Icons and Map are stored as raw pointers in TD

// Red Alert version (REDALERT/WIN32LIB/TILE.H, REDALERT/COMPAT.H)
typedef struct {
    short Width;      // Width of icons (pixels)
    short Height;     // Height of icons (pixels)
    short Count;      // Number of (logical) icons in this set
    short Allocated;  // Was this iconset allocated? (runtime flag)
    short MapWidth;   // Width of map (in icons) — RA-only field
    short MapHeight;  // Height of map (in icons) — RA-only field
    long  Size;       // Size of entire iconset memory block
    long  Icons;      // Offset from buffer start to icon data
    long  Palettes;   // Offset from buffer start to palette data
    long  Remaps;     // Offset from buffer start to remap index data
    long  TransFlag;  // Offset for transparency flag table
    long  ColorMap;   // Offset for color control value table — RA-only field
    long  Map;        // Icon map offset (if present)
} IControl_Type;
// Note: RA version uses long offsets (not pointers) and adds MapWidth, MapHeight, ColorMap
```

**Constraint:** "This structure MUST be a multiple of 16 bytes long" (per source comment in STAMP.INC and TILE.H).

#### How the Map Array Works

The `Map` array maps logical grid positions to physical icon indices. Each byte represents one cell in the template grid (`MapWidth × MapHeight` in RA, or `Width × Height` in TD). A value of `0xFF` (`-1` signed) means the cell is empty/transparent — no tile is drawn there.

```c
// From CDATA.CPP — reading the icon map
Mem_Copy(Get_Icon_Set_Map(Get_Image_Data()), map, Width * Height);
for (index = 0; index < Width * Height; index++) {
    if (map[index] != 0xFF) {
        // This cell has a visible tile — draw icon data at map[index]
    }
}
```

Icon pixel data is accessed as: `&Icons[map[index] * (24 * 24)]` — each icon is 576 bytes of palette-indexed pixels.

#### Color Control Map (RA only)

The `ColorMap` table provides per-icon land type information. Each byte maps to one of 16 terrain categories used by the game logic:

```c
// From CDATA.CPP — RA land type lookup
static LandType _land[16] = {
    LAND_CLEAR, LAND_CLEAR, LAND_CLEAR, LAND_CLEAR,  // 0-3
    LAND_CLEAR, LAND_CLEAR, LAND_BEACH, LAND_CLEAR,  // 4-7
    LAND_ROCK,  LAND_ROAD,  LAND_WATER, LAND_RIVER,  // 8-11
    LAND_CLEAR, LAND_CLEAR, LAND_ROUGH, LAND_CLEAR,  // 12-15
};
return _land[control_map[icon_index]];
```

#### IconsetClass (RA Only)

Red Alert wraps `IControl_Type` in a C++ class with accessor methods:

```c
// From COMPAT.H
class IconsetClass : protected IControl_Type {
public:
    int Map_Width()                  const { return MapWidth; }
    int Map_Height()                 const { return MapHeight; }
    int Icon_Count()                 const { return Count; }
    int Pixel_Width()                const { return Width; }
    int Pixel_Height()               const { return Height; }
    int Total_Size()                 const { return Size; }
    unsigned char const * Icon_Data()    const { return (unsigned char const *)this + Icons; }
    unsigned char const * Map_Data()     const { return (unsigned char const *)this + Map; }
    unsigned char const * Palette_Data() const { return (unsigned char const *)this + Palettes; }
    unsigned char const * Remap_Data()   const { return (unsigned char const *)this + Remaps; }
    unsigned char const * Trans_Data()   const { return (unsigned char const *)this + TransFlag; }
    unsigned char * Control_Map()        { return (unsigned char *)this + ColorMap; }
};
```

All offset fields are relative to the start of the `IControl_Type` structure itself — the data is a single contiguous allocation.

---

### PAL Palette Format (.pal)

**Source:** `REDALERT/WIN32LIB/PALETTE.H`, `TIBERIANDAWN/WIN32LIB/LOADPAL.CPP`, `REDALERT/WIN32LIB/DrawMisc.cpp`

PAL files are the simplest format — a raw dump of 256 RGB color values with no header.

#### File Layout

```
768 bytes total = 256 entries × 3 bytes (R, G, B)
```

No magic number, no header, no footer. Just 768 bytes of color data.

#### Constants

```c
// From PALETTE.H
#define RGB_BYTES      3
#define PALETTE_SIZE   256
#define PALETTE_BYTES  768   // PALETTE_SIZE * RGB_BYTES
```

#### Color Range: 6-bit VGA (0–63)

Each R, G, B component is in **6-bit VGA range (0–63)**, not 8-bit. This is because the original VGA hardware registers only accepted 6-bit color values.

```c
// From PALETTE.H
typedef struct {
    char red;
    char green;
    char blue;
} RGB;   // Each field: 0–63 (6-bit)
```

#### Loading and Conversion

```c
// From LOADPAL.CPP — loading is trivially simple
void Load_Palette(char *palette_file_name, void *palette_pointer) {
    Load_Data(palette_file_name, palette_pointer, 768);
}

// From DDRAW.CPP — converting 6-bit VGA to 8-bit for display
void Set_DD_Palette(void *palette) {
    for (int i = 0; i < 768; i++) {
        buffer[i] = palette[i] << 2;  // 6-bit (0–63) → 8-bit (0–252)
    }
}

// From WRITEPCX.CPP — PCX files use 8-bit, converted on read
// Reading PCX palette:  value >>= 2;  (8-bit → 6-bit)
// Writing PCX palette:  value <<= 2;  (6-bit → 8-bit)
```

**Implementation note for ra-formats:** When loading `.pal` files, expose both the raw 6-bit values and a convenience method that returns 8-bit values (left-shift by 2). The 6-bit values are the canonical form — all palette operations in the original game work in 6-bit space.

---

### AUD Audio Format (.aud)

**Source:** `REDALERT/WIN32LIB/AUDIO.H`, `REDALERT/ADPCM.CPP`, `REDALERT/ITABLE.CPP`, `REDALERT/DTABLE.CPP`, `REDALERT/WIN32LIB/SOSCOMP.H`

AUD files contain IMA ADPCM-compressed audio (Westwood's variant). The file has a simple header followed by compressed audio chunks.

#### File Header

```c
// From AUDIO.H
#pragma pack(push, 1)
typedef struct {
    unsigned short int Rate;        // Playback rate in Hz (e.g., 22050)
    long               Size;        // Size of compressed data (bytes)
    long               UncompSize;  // Size of uncompressed data (bytes)
    unsigned char      Flags;       // Bit flags (see below)
    unsigned char      Compression; // Compression algorithm ID
} AUDHeaderType;
#pragma pack(pop)
```

**Flags:**
| Bit    | Name              | Meaning                     |
| ------ | ----------------- | --------------------------- |
| `0x01` | `AUD_FLAG_STEREO` | Stereo audio (two channels) |
| `0x02` | `AUD_FLAG_16BIT`  | 16-bit samples (vs. 8-bit)  |

**Compression types** (from `SOUNDINT.H`):

| Value | Name             | Algorithm                                  |
| ----- | ---------------- | ------------------------------------------ |
| 0     | `SCOMP_NONE`     | No compression                             |
| 1     | `SCOMP_WESTWOOD` | Westwood ADPCM (the standard for RA audio) |
| 33    | `SCOMP_SONARC`   | Sonarc compression                         |
| 99    | `SCOMP_SOS`      | SOS ADPCM                                  |

#### ADPCM Codec Structure

```c
// From SOSCOMP.H — codec state for ADPCM decompression
typedef struct _tagCOMPRESS_INFO {
    char *          lpSource;         // Source data pointer
    char *          lpDest;           // Destination buffer pointer
    unsigned long   dwCompSize;       // Compressed data size
    unsigned long   dwUnCompSize;     // Uncompressed data size
    unsigned long   dwSampleIndex;    // Current sample index (channel 1)
    long            dwPredicted;      // Predicted sample value (channel 1)
    long            dwDifference;     // Difference value (channel 1)
    short           wCodeBuf;         // Code buffer (channel 1)
    short           wCode;            // Current code (channel 1)
    short           wStep;            // Step size (channel 1)
    short           wIndex;           // Index into step table (channel 1)
    // --- Stereo: second channel state ---
    unsigned long   dwSampleIndex2;
    long            dwPredicted2;
    long            dwDifference2;
    short           wCodeBuf2;
    short           wCode2;
    short           wStep2;
    short           wIndex2;
    // ---
    short           wBitSize;         // Bits per sample (8 or 16)
    short           wChannels;        // Number of channels (1=mono, 2=stereo)
} _SOS_COMPRESS_INFO;

// Chunk header for compressed audio blocks
typedef struct _tagCOMPRESS_HEADER {
    unsigned long dwType;             // Compression type identifier
    unsigned long dwCompressedSize;   // Size of compressed data
    unsigned long dwUnCompressedSize; // Size when decompressed
    unsigned long dwSourceBitSize;    // Original bit depth
    char          szName[16];         // Name string
} _SOS_COMPRESS_HEADER;
```

#### Westwood ADPCM Decompression Algorithm

The algorithm processes each byte as two 4-bit nibbles (low nibble first, then high nibble). It uses pre-computed `IndexTable` and `DiffTable` lookup tables for decoding.

```c
// From ADPCM.CPP — core decompression loop (simplified)
// 'code' is one byte of compressed data containing TWO samples
//
// For each byte:
//   1. Process low nibble  (code & 0x0F)
//   2. Process high nibble (code >> 4)
//
// Per nibble:
//   fastindex = (fastindex & 0xFF00) | token;   // token = 4-bit nibble
//   sample += DiffTable[fastindex];              // apply difference
//   sample = clamp(sample, -32768, 32767);       // clamp to 16-bit range
//   fastindex = IndexTable[fastindex];           // advance index
//   output = (unsigned short)sample;             // write sample

// The 'fastindex' combines the step index (high byte) and token (low byte)
// into a single 16-bit lookup key: index = (step_index << 4) | token
```

**Table structure:** Both tables are indexed by `[step_index * 16 + token]` where `step_index` is 0–88 and `token` is 0–15, giving 1424 entries each.

- `IndexTable[1424]` (`unsigned short`) — next step index after applying this token
- `DiffTable[1424]` (`long`) — signed difference to add to the current sample

The tables are pre-multiplied by 16 for performance (the index already includes the token offset). Full table values are in `ITABLE.CPP` and `DTABLE.CPP`.

---

### VQA Video Format (.vqa)

**Source:** `VQ/INCLUDE/VQA32/VQAFILE.H` (CnC_Red_Alert repo), `REDALERT/WIN32LIB/IFF.H`

VQA (Vector Quantized Animation) files store cutscene videos using vector quantization — a codebook of small pixel blocks that are referenced by index to reconstruct each frame.

#### VQA File Header

```c
// From VQAFILE.H
typedef struct _VQAHeader {
    unsigned short Version;         // Format version
    unsigned short Flags;           // Bit 0 = has audio, Bit 1 = has alt audio
    unsigned short Frames;          // Total number of video frames
    unsigned short ImageWidth;      // Image width in pixels
    unsigned short ImageHeight;     // Image height in pixels
    unsigned char  BlockWidth;      // Codebook block width (typically 4)
    unsigned char  BlockHeight;     // Codebook block height (typically 2 or 4)
    unsigned char  FPS;             // Frames per second (typically 15)
    unsigned char  Groupsize;       // VQ codebook group size
    unsigned short Num1Colors;      // Number of 1-color blocks(?)
    unsigned short CBentries;       // Number of codebook entries
    unsigned short Xpos;            // X display position
    unsigned short Ypos;            // Y display position
    unsigned short MaxFramesize;    // Largest frame size (for buffer allocation)
    // Audio fields
    unsigned short SampleRate;      // Audio sample rate (e.g., 22050)
    unsigned char  Channels;        // Audio channels (1=mono, 2=stereo)
    unsigned char  BitsPerSample;   // Audio bits per sample (8 or 16)
    // Alternate audio stream
    unsigned short AltSampleRate;
    unsigned char  AltChannels;
    unsigned char  AltBitsPerSample;
    // Reserved
    unsigned short FutureUse[5];
} VQAHeader;
```

#### VQA Chunk Types

VQA files use a chunk-based IFF-like structure. Each chunk has a 4-byte ASCII identifier and a big-endian 4-byte size.

**Top-level structure:**

| Chunk  | Purpose                                        |
| ------ | ---------------------------------------------- |
| `WVQA` | Form/container chunk (file magic)              |
| `VQHD` | VQA header (contains `VQAHeader` above)        |
| `FINF` | Frame info table — seek offsets for each frame |
| `VQFR` | Video frame (delta frame)                      |
| `VQFK` | Video keyframe                                 |

**Sub-chunks within frames:**

| Chunk           | Purpose                                                               |
| --------------- | --------------------------------------------------------------------- |
| `CBF0` / `CBFZ` | Full codebook, uncompressed / LCW-compressed                          |
| `CBP0` / `CBPZ` | Partial codebook (1/Groupsize of full), uncompressed / LCW-compressed |
| `VPT0` / `VPTZ` | Vector pointers (frame block indices), uncompressed / LCW-compressed  |
| `VPTK`          | Vector pointer keyframe                                               |
| `VPTD`          | Vector pointer delta (differences from previous frame)                |
| `VPTR` / `VPRZ` | Vector pointer + run-skip-dump encoding                               |
| `CPL0` / `CPLZ` | Palette (256 × RGB), uncompressed / LCW-compressed                    |
| `SND0`          | Audio — raw PCM                                                       |
| `SND1`          | Audio — Westwood "ZAP" ADPCM                                          |
| `SND2`          | Audio — IMA ADPCM (same codec as AUD files)                           |
| `SNDZ`          | Audio — LCW-compressed                                                |

**Naming convention:** Suffix `0` = uncompressed data. Suffix `Z` = LCW-compressed. Suffix `K` = keyframe. Suffix `D` = delta.

#### FINF (Frame Info) Table

The `FINF` chunk contains a table of 4 bytes per frame encoding seek position and flags:

```c
// Bits 31–28: Frame flags
//   Bit 31 (0x80000000): KEY   — keyframe (full codebook + vector pointers)
//   Bit 30 (0x40000000): PAL   — frame includes palette change
//   Bit 29 (0x20000000): SYNC  — audio sync point
// Bits 27–0: File offset in WORDs (multiply by 2 for byte offset)
```

#### VPC Codes (Vector Pointer Compression)

```c
// Run-skip-dump encoding opcodes for vector pointer data
#define VPC_ONE_SINGLE      0xF000  // Single block, one value
#define VPC_ONE_SEMITRANS   0xE000  // Semi-transparent block
#define VPC_SHORT_DUMP      0xD000  // Short literal dump
#define VPC_LONG_DUMP       0xC000  // Long literal dump
#define VPC_SHORT_RUN       0xB000  // Short run of same value
#define VPC_LONG_RUN        0xA000  // Long run of same value
```

---

### VQ Static Image Format (.vqa still frames)

**Source:** `WINVQ/INCLUDE/VQFILE.H`, `VQ/INCLUDE/VQ.H` (CnC_Red_Alert repo)

Separate from VQA movies, the VQ format handles single static vector-quantized images.

#### VQ Header (VQFILE.H variant)

```c
// From VQFILE.H
typedef struct _VQHeader {
    unsigned short Version;
    unsigned short Flags;
    unsigned short ImageWidth;
    unsigned short ImageHeight;
    unsigned char  BlockType;     // Block encoding type
    unsigned char  BlockWidth;
    unsigned char  BlockHeight;
    unsigned char  BlockDepth;    // Bits per pixel
    unsigned short CBEntries;     // Codebook entries
    unsigned char  VPtrType;      // Vector pointer encoding type
    unsigned char  PalStart;      // First palette index used
    unsigned short PalLength;     // Number of palette entries
    unsigned char  PalDepth;      // Palette bit depth
    unsigned char  ColorModel;    // Color model (see below)
} VQHeader;
```

#### VQ Header (VQ.H variant — 40 bytes, for VQ encoder)

```c
// From VQ.H
typedef struct _VQHeader {
    long           ImageSize;     // Total image size in bytes
    unsigned short ImageWidth;
    unsigned short ImageHeight;
    unsigned char  BlockWidth;
    unsigned char  BlockHeight;
    unsigned char  BlockType;     // Block encoding type
    unsigned char  PaletteRange;  // Palette range
    unsigned short Num1Color;     // Number of 1-color blocks
    unsigned short CodebookSize;  // Codebook entries
    unsigned char  CodingFlag;    // Coding method flag
    unsigned char  FrameDiffMethod; // Frame difference method
    unsigned char  ForcedPalette; // Forced palette flag
    unsigned char  F555Palette;   // Use 555 palette format
    unsigned short VQVersion;     // VQ codec version
} VQHeader;
```

#### VQ Chunk IDs

| Chunk  | Purpose                  |
| ------ | ------------------------ |
| `VQHR` | VQ header                |
| `VQCB` | VQ codebook data         |
| `VQCT` | VQ color table (palette) |
| `VQVP` | VQ vector pointers       |

#### Color Models

```c
#define VQCM_PALETTED  0   // Palette-indexed (standard RA/TD)
#define VQCM_RGBTRUE   1   // RGB true color
#define VQCM_YBRTRUE   2   // YBR (luminance-chrominance) true color
```

---

### WSA Animation Format (.wsa)

**Source:** `REDALERT/WSA.H`, `REDALERT/WSA.CPP`

WSA (Westwood Studios Animation) files contain LCW-compressed delta animations. Used for menu backgrounds, installation screens, campaign map animations, and some in-game effects in both TD and RA. Each frame is an XOR-delta against the previous frame.

#### File Header

```c
// From WSA.H
typedef struct {
    unsigned short NumFrames;     // Number of animation frames
    unsigned short Width;         // Frame width in pixels
    unsigned short Height;        // Frame height in pixels
    unsigned short Delta;         // Delta buffer size
    unsigned short Flags;         // (unused in RA)
    unsigned long  Offsets[];     // (NumFrames + 2) offsets to frame data
} WSA_Header;
```

**Frame data layout:**
- `Offsets[0]` through `Offsets[NumFrames-1]` point to each frame's LCW-compressed XOR-delta data
- `Offsets[NumFrames]` points to a loop-back delta (for seamless looping animations)
- `Offsets[NumFrames+1]` marks end of data (used to compute last frame's compressed size)
- If an offset is 0, that frame is identical to the previous frame (no delta)
- Palette data (768 bytes, 6-bit VGA) may follow the frame data if the WSA includes its own palette

**Decoding algorithm:**
1. Allocate a frame buffer (Width × Height bytes, palette-indexed)
2. For each frame: LCW-decompress the delta data, then XOR the delta onto the frame buffer
3. The frame buffer now contains the current frame's pixels
4. For looping: apply `Offsets[NumFrames]` delta to return to frame 0

**Security:** Same defensive parsing requirements as other LCW-consuming formats — decompression ratio cap (256:1), output size cap (max 4 MB per frame), iteration counter on LCW decode loop. See V38 in `security/vulns-infrastructure.md`.

---

### FNT Bitmap Font Format (.fnt)

**Source:** `REDALERT/WIN32LIB/FONT.H`, `REDALERT/WIN32LIB/FONT.CPP`

FNT files contain bitmap fonts used for in-game text rendering. Each file contains a fixed set of character glyphs as palette-indexed pixel bitmaps.

#### File Header

```c
// From FONT.H
typedef struct {
    unsigned short InfoBlock;     // Offset to font info block
    unsigned short OffsetBlock;   // Offset to character offset table
    unsigned short WidthBlock;    // Offset to character width table
    unsigned short DataBlock;     // Offset to glyph bitmap data
    unsigned short HeightBlock;   // Offset to per-character height info
} FontHeader_Type;
```

**Font info block** (at `InfoBlock` offset):
- `MaxHeight` (u8) — maximum character height in pixels
- `MaxWidth` (u8) — maximum character width in pixels
- `NumChars` (u16) — number of characters (typically 128 or 256)

**Per-character data:**
- **Offset table** (at `OffsetBlock`): `NumChars` × `u16` — byte offset from `DataBlock` to each character's bitmap
- **Width table** (at `WidthBlock`): `NumChars` × `u8` — pixel width of each character
- **Glyph data** (at `DataBlock`): raw palette-indexed pixels, 1 byte per pixel, row-major order. Each glyph is `width × height` bytes. Index 0 = transparent background.

**IC usage:** IC does not use `.fnt` for runtime text rendering (Bevy's font pipeline handles modern TTF/OTF fonts with CJK/RTL support). `.fnt` parsing is needed for: (1) displaying original game fonts faithfully in Classic render mode (D048), (2) Asset Studio (D040) font preview and export.

---

## Insights from EA's Original Source Code

Repository: https://github.com/electronicarts/CnC_Red_Alert (GPL v3, archived Feb 2025)

### Code Statistics
- 290 C++ header files, 296 implementation files, 14 x86 assembly files
- ~222,000 lines of C++ code
- 430+ `#ifdef WIN32` checks (no other platform implemented)
- Built with Watcom C/C++ v10.6 and Borland Turbo Assembler v4.0

### Keep: Event/Order Queue System

The original uses `OutList` (local player commands) and `DoList` (confirmed orders from all players), both containing `EventClass` objects:

```cpp
// From CONQUER.CPP
OutList.Add(EventClass(EventClass::IDLE, TargetClass(tech)));
```

Player actions → events → queue → deterministic processing each tick. This is the same pattern as our `PlayerOrder → TickOrders → Simulation::apply_tick()` pipeline. Westwood validated this in 1996.

### Keep: Integer Math for Determinism

The original uses integer math everywhere for game logic — positions, damage, timing. No floats in the simulation. This is why multiplayer worked. Our `FixedPoint` / `SimCoord` approach mirrors this.

### Keep: Data-Driven Rules (INI → MiniYAML → YAML)

Original reads unit stats and game rules from `.ini` files at runtime. This data-driven philosophy is what made C&C so moddable. The lineage: `INI → MiniYAML → YAML` — each step more expressive, same philosophy.

### Keep: MIX Archive Concept

Simple flat archive with hash-based lookup. No compression in the archive itself (individual files may be compressed). For `ra-formats`: read MIX as-is for compatibility; native format can modernize.

### Keep: Compression Flexibility

Original implements LCW, LZO, and LZW compression. LZO was settled on for save games:
```cpp
// From SAVELOAD.CPP
LZOPipe pipe(LZOPipe::COMPRESS, SAVE_BLOCK_SIZE);
// LZWPipe pipe(LZWPipe::COMPRESS, SAVE_BLOCK_SIZE);  // tried, abandoned
// LCWPipe pipe(LCWPipe::COMPRESS, SAVE_BLOCK_SIZE);   // tried, abandoned
```

### Leave Behind: Session Type Branching

Original code is riddled with network-type checks embedded in game logic:
```cpp
if (Session.Type == GAME_IPX || Session.Type == GAME_INTERNET) { ... }
```

This is the anti-pattern our `NetworkModel` trait eliminates. Separate code paths for IPX, Westwood Online, MPlayer, TEN, modem — all interleaved with `#ifdef`. The developer disliked the Westwood Online API enough to write a complete wrapper around it.

### Leave Behind: Platform-Specific Rendering

DirectDraw surface management with comments like "Aaaarrgghh!" when hardware allocation fails. Manual VGA mode detection. Custom command-line parsing. `wgpu` solves all of this.

### Leave Behind: Manual Memory Checking

The game allocates 13MB and checks if it succeeds. Checks that `sleep(1000)` actually advances the system clock. Checks free disk space. None of this translates to modern development.

### Interesting Historical Details

- Code path for 640x400 display mode with special VGA fallback
- `#ifdef FIXIT_CSII` for Aftermath expansion — comment explains they broke the ability to build vanilla Red Alert executables and had to fix it later
- Developer comments reference "Counterstrike" in VCS headers (`$Header: /CounterStrike/...`)
- MPEG movie playback code exists but is disabled
- Game refuses to start if launched from `f:\projects\c&c0` (the network share)

## Coordinate System Translation

For cross-engine compatibility, coordinate transforms must be explicit:

```rust
pub struct CoordTransform {
    pub our_scale: i32,       // our subdivisions per cell
    pub openra_scale: i32,    // 1024 for OpenRA (WDist/WPos)
    pub original_scale: i32,  // original game's lepton system
}

impl CoordTransform {
    pub fn to_wpos(&self, pos: &CellPos) -> (i32, i32, i32) {
        ((pos.x * self.openra_scale) / self.our_scale,
         (pos.y * self.openra_scale) / self.our_scale,
         (pos.z * self.openra_scale) / self.our_scale)
    }
    pub fn from_wpos(&self, x: i32, y: i32, z: i32) -> CellPos {
        CellPos {
            x: (x * self.our_scale) / self.openra_scale,
            y: (y * self.our_scale) / self.openra_scale,
            z: (z * self.our_scale) / self.openra_scale,
        }
    }
}
```
