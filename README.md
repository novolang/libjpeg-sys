# libjpeg-sys

JPEG is a lossy compression method for photographs, specified in
ITU-T T.81. libjpeg-turbo is the reference implementation most
programs use, and it publishes two C APIs: the libjpeg API, which is
the original one from the Independent JPEG Group, and the
[TurboJPEG API](https://libjpeg-turbo.org/Documentation/Documentation),
which compresses and decompresses a whole image in one call. This
package declares twenty of the TurboJPEG entry points to novo-lang,
one declaration each.

Every function here is a declaration of a function in libturbojpeg.
The package contains no logic of its own, and it does nothing without
the C library installed. The twenty calls cover the compressor, the
decompressor, the planar YUV conversions, the buffer allocator and the
error reporting. The libjpeg API is not among them: every one of its
entry points works on a state structure the caller allocates, and the
first call takes the structure's size as an argument and refuses any
number but the library's own. That size is a build-time fact only the
C header knows, and a program that gets it wrong is ended by the
library rather than told. The section "What is not included" gives the
measurement.

## What it is

An **image** here is a rectangle of pixels in memory. Its **pixel
format** says how many bytes a pixel takes and in what order the
components sit: RGB is three bytes, RGBA is four, grayscale is one.
Its **pitch** is the number of bytes from the start of one row to the
start of the next, which is larger than the row when the rows are
padded.

A **JPEG image** is the compressed form: a series of markers and
entropy-coded blocks, ending at an end-of-image marker. Its header
records the size, the colour space and the subsampling, and it can be
read without decompressing anything.

**Chrominance subsampling** is the first thing JPEG throws away. The
eye sees detail in brightness better than in colour, so the colour
components are stored at a lower resolution than the brightness.
4:4:4 keeps them all, 4:2:0 keeps one colour sample for each two-by-two
block of pixels, and grayscale keeps none.

**Quality** is a number between 1 and 100 that scales the quantization
tables. It is not a percentage of anything, and the size it produces
depends on the picture.

A **planar YUV image** is the halfway house: the colours have been
converted and subsampled, and nothing has been entropy-coded yet. It
is three planes one after another — one of brightness and two of
colour — each row padded out to a multiple of a chosen number of
bytes. A program that has to do the two halves separately, or that
gets its frames from a video decoder in that form already, works in
it.

A **handle** is TurboJPEG's instance. It holds the working memory, and
a program that compresses many images creates one and reuses it.

## Install

```
novo pkg add libjpeg-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package
`libturbojpeg0-dev`:

```
sudo apt install libturbojpeg0-dev
```

On macOS the Homebrew formula is `jpeg-turbo`. On other systems the
library builds from the libjpeg-turbo source with CMake.

A system may carry libjpeg-turbo's libjpeg API and not its TurboJPEG
one, which is a separate shared library in a separate package. This
package needs `libturbojpeg.so`, and `libjpeg.so` alone is not enough.

## Example

A picture compressed and read back:

```novo ignore
use libjpeg

fn main() [io, ffi]
    let comp = libjpeg.tj_init_compress()
    if comp == 0
        println("no compressor")
        return

    // A grey rectangle, 32 by 16 pixels, three bytes each. The three
    // bytes beyond it are the reach of the last `ptr.write_i32`.
    let src = ptr.alloc(32 * 16 * 3 + 3)
    for i in 0..32 * 16 * 3
        ptr.write_i32(src + i, 128)

    // A buffer of the bound never overflows, whatever the picture.
    let bound = libjpeg.tj_buf_size(32, 16, 2)
    let out = libjpeg.tj_alloc(bound)
    let buf_slot = ptr.alloc_word()
    ptr.write_word(buf_slot, out)
    let size_slot = ptr.alloc_word()
    ptr.write_word(size_slot, bound)

    // Quality 90, 4:2:0, and the flag 1024 keeps the buffer above.
    if libjpeg.tj_compress2(comp, src, 32, 0, 16, 0, buf_slot, size_slot, 2, 90, 1024) != 0
        println(ptr.read_str(libjpeg.tj_get_error_str2(comp)))
        return
    println("${ptr.read_word(size_slot)} byte JPEG image")

    // The header says how big the picture is, without decompressing.
    let decomp = libjpeg.tj_init_decompress()
    let w = ptr.alloc_word()
    let h = ptr.alloc_word()
    let samp = ptr.alloc_word()
    let cs = ptr.alloc_word()
    let _ = libjpeg.tj_decompress_header3(decomp, out, ptr.read_word(size_slot), w, h,
                                          samp, cs)
    println("${ptr.read_word(w)} by ${ptr.read_word(h)}")

    // A buffer the library reserved is released by the library.
    libjpeg.tj_free(out)
    ptr.free(src)
    let _ = libjpeg.tj_destroy(decomp)
    let _ = libjpeg.tj_destroy(comp)
```

The fence reads `novo ignore`, so `novo doc` lists the example and does
not compile it. A compiled block is linked against libturbojpeg, and
the link fails on a machine where that library is not installed. The
same calls are in `tests/libjpeg_tests.nv`, where the round trip is
asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libjpeg` | Every entry point, in five groups: the compressor, the decompressor, the planar helpers, the buffers, and the instance and the errors. |

The five groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The compressor | 6 | Compresses pixels or planes into a JPEG image, and sizes the output. |
| The decompressor | 6 | Reads a header, decompresses to pixels or planes, and lists the scaling factors. |
| The planar helpers | 3 | Answers the width, height and size of one plane of a planar YUV image. |
| The buffers | 2 | Reserves and releases a buffer the library may reallocate. |
| The instance and the errors | 3 | Releases a handle, and names and rates its last failure. |

## How to choose an entry point

`tjCompress2` and `tjDecompress2` are the ordinary pair: pixels in,
JPEG image out, and back. Most programs need nothing else.

`tjEncodeYUV3` and `tjCompressFromYUV` split the compression in two,
and `tjDecompressToYUV2` and `tjDecodeYUV3` split the decompression.
Use them when the frames arrive as planes already, or when the two
halves run in different places.

`tjDecompressHeader3` is for sizing a buffer, and for deciding whether
to decompress at all. It reads a few bytes and decompresses nothing.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** The handle the C
   library returns arrives as the address it returned.
2. **A call that answers a number answers -1 on failure.** A C `int`
   arrives in 32 bits, so write `as i32` before comparing the answer
   with -1. `tjInitCompress`, `tjInitDecompress` and `tjAlloc` answer
   an address, and their failure is 0.
3. **A failure names itself on its handle.** `tjGetErrorStr2` answers
   the text of the last failure on that handle, and `tjGetErrorCode`
   answers 0 when the call only warned and 1 when it failed outright.
   A -1 with a code of 0 means the call produced output anyway, which
   is what a corrupt but readable image gives.
4. **The output buffer is the library's unless you say otherwise.**
   `tjCompress2` without the flag 1024 reallocates the buffer it is
   given. Reserve it with `tjAlloc` and release it with `tjFree`, never
   with `ptr.free`. With the flag 1024 the library keeps the buffer and
   fails if it is too small.
5. **`tjBufSize` is the size that always suffices.** Whatever the
   picture and whatever the quality, a buffer of that size holds the
   result.
6. **The two slots of a compression are eight bytes each.** The first
   holds the address of the output buffer, and the second holds its
   capacity going in and the length coming out.
7. **The four out-parameters of `tjDecompressHeader3` are four bytes
   each.** `ptr.alloc_word` reserves eight with every byte zero, so on
   a little-endian machine `ptr.read_word` reads the value back with no
   other byte in the way.
8. **A pitch of 0 means the rows are packed**, which is width times the
   bytes a pixel takes.
9. **A width or height of 0 in `tjDecompress2` means that side does not
   limit the size.** The decompressor scales the header's size by the
   largest factor from `tjGetScalingFactors` whose result fits within
   both sides, so 0 for both gives the header's size. The picture
   written is that scaled size, which can be smaller than the size
   asked for, and the call fails when no factor is small enough to fit.
10. **A scaling factor is two four-byte integers**, a numerator and
    then a denominator, eight bytes to the pair, in an array the
    library owns. The list holds 1/1, and the factors this
    implementation supports run from 2/1 down to 1/8.
11. **The pixel formats are numbers**, because the C header spells
    them as an enumeration.

    | Format | Number | Bytes per pixel |
    | --- | --- | --- |
    | `TJPF_RGB` | 0 | 3 |
    | `TJPF_BGR` | 1 | 3 |
    | `TJPF_RGBX` | 2 | 4 |
    | `TJPF_BGRX` | 3 | 4 |
    | `TJPF_XBGR` | 4 | 4 |
    | `TJPF_XRGB` | 5 | 4 |
    | `TJPF_GRAY` | 6 | 1 |
    | `TJPF_RGBA` | 7 | 4 |
    | `TJPF_BGRA` | 8 | 4 |
    | `TJPF_ABGR` | 9 | 4 |
    | `TJPF_ARGB` | 10 | 4 |
    | `TJPF_CMYK` | 11 | 4 |

12. **The subsamplings are numbers too.**

    | Subsampling | Number | What it keeps |
    | --- | --- | --- |
    | `TJSAMP_444` | 0 | every colour sample |
    | `TJSAMP_422` | 1 | one colour sample per two pixels across |
    | `TJSAMP_420` | 2 | one per two-by-two block |
    | `TJSAMP_GRAY` | 3 | no colour at all |
    | `TJSAMP_440` | 4 | one per two pixels down |
    | `TJSAMP_411` | 5 | one per four pixels across |

13. **The colour spaces are numbers too**: 0 RGB, 1 YCbCr, 2
    grayscale, 3 CMYK, 4 YCCK. A colour JPEG image is stored as YCbCr.
14. **The flags are bits**, combined with a bitwise or: 2 reads the
    rows from the bottom up, 1024 keeps the caller's output buffer,
    2048 uses the fast discrete cosine transform, 4096 the accurate
    one, 8192 makes a warning a failure and 16384 writes a progressive
    JPEG image.
15. **A plane's padding must be a power of two.** `tjBufSizeYUV2`
    answers the size the three planes take at that padding, and the
    three `tjPlane` calls answer the parts of it.

## What is not included

- **The libjpeg API.** libjpeg-turbo's other C API —
  `jpeg_CreateCompress`, `jpeg_start_compress`, `jpeg_write_scanlines`,
  `jpeg_read_header` and the rest — works on a `jpeg_compress_struct`
  or a `jpeg_decompress_struct` that the caller allocates.
  `jpeg_CreateCompress` takes the library version and the structure's
  size as arguments and compares each with the library's own, and both
  numbers are build-time facts that only the C header records. On the
  machine this package was written on, passing 600 for the size
  answered `JPEG parameter struct mismatch: library thinks size is 584,
  caller expects 600` and ended the process, because a mismatch reaches
  the error manager's `error_exit`, whose default calls `exit`. The
  error manager cannot be replaced either: it is a structure of C
  function pointers the caller installs before the first call, and a
  novo-lang function is not one. Without the header a program cannot
  make the first call of that API, and so it can make none of them.
- **`tjTransform` and `tjInitTransform`.** The lossless rotation, flip
  and crop take an array of `tjtransform` structures, one field of
  which is a C function pointer for a filter. The field may be left
  null, so the call could be bound; it is left out because its
  structure's layout cannot be checked on a machine with no
  libturbojpeg on it.
- **The TurboJPEG 3 API.** `tj3Init`, `tj3Set`, `tj3Compress8`,
  `tj3Decompress8` and their relatives arrived in libjpeg-turbo 3.0.
  This package binds the older calls, which every version of the
  library has.
- **The file helpers.** `tj3LoadImage8` and `tj3SaveImage8` read and
  write files and belong to the TurboJPEG 3 API.
- **`tjGetErrorStr`.** The form without a handle reports the last
  failure anywhere in the process. `tjGetErrorStr2` takes the handle
  and is here.

## Related packages

`image-nv` is the image codec library written in novo-lang, with no C
library. This package is the reference implementation that port
measures its JPEG codec against. `image-nv` is planned and not
published yet.

Choose `image-nv` when the program must build for a microcontroller or
for WebAssembly, or when a C toolchain is not wanted. Choose this
package when the program needs libjpeg-turbo's speed, which is what its
hand-written vector code is for.

## Tests

`tests/libjpeg_tests.nv` holds eight tests over the twenty entry
points:

```
novo test tests/libjpeg_tests.nv
```

The suite links against libturbojpeg, so it needs that library
installed. Without it the link fails, naming `-lturbojpeg`.
`novo pkg build` type-checks the declarations and needs nothing
installed.

The machine this package was written on has no libturbojpeg, so the
suite has not linked there and no test in it has run there. The
sources compile, and the link is the step that fails.

Every test works in memory over a picture the suite draws itself, so
the suite reads and writes nothing and needs no privileges. The suite
asserts that a 32-by-16 picture compresses at quality 90 and 4:2:0 into
a buffer no larger than `tjBufSize`, that its header reports the size,
the subsampling and the YCbCr colour space it was written with, that
bytes which are not a JPEG image answer -1 with a named error and a
code of 1, that the list of scaling factors holds 1/1, that a picture
converts to planes and back, that planes compress to a JPEG image and
decompress to planes again, that a 4:2:0 chrominance plane is half as
wide and half as tall as its picture, that grayscale has no
chrominance plane at all, and that a grayscale picture compresses.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

libjpeg-turbo itself is distributed under the IJG licence, the
three-clause BSD licence and the zlib licence, and installing it is the
reader's own step.
