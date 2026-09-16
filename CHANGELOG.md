# Changelog

All notable changes to libjpeg-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-16

The first release: twenty entry points of the TurboJPEG C API, one
`@ffi` declaration each, and no logic.

### Added

- `libjpeg` — the whole TurboJPEG surface, in five groups.
  - The compressor: `tjInitCompress`, `tjCompress2`, `tjBufSize`,
    `tjEncodeYUV3`, `tjBufSizeYUV2` and `tjCompressFromYUV`.
  - The decompressor: `tjInitDecompress`, `tjDecompressHeader3`,
    `tjDecompress2`, `tjDecompressToYUV2`, `tjDecodeYUV3` and
    `tjGetScalingFactors`.
  - The planar helpers: `tjPlaneSizeYUV`, `tjPlaneWidth` and
    `tjPlaneHeight`.
  - The buffers: `tjAlloc` and `tjFree`.
  - The instance and the errors: `tjDestroy`, `tjGetErrorStr2` and
    `tjGetErrorCode`.
- `tests/libjpeg_tests.nv` — eight tests over the signatures. They
  call the C library, so they need libturbojpeg installed. Every test
  works in memory over a picture the suite draws itself, so the suite
  reads and writes nothing.

### `wraps = "libturbojpeg"` and not `libjpeg`

libjpeg-turbo ships two interfaces in two shared libraries, and a
`sys` package wraps one. **The libjpeg API is the one this package
cannot bind, and finding that out is what the release decided.**

`jpeg_CreateCompress` and `jpeg_CreateDecompress` take a
`jpeg_compress_struct` or a `jpeg_decompress_struct` that the CALLER
allocates, and they take its `sizeof` as an argument to check it
against the library's own. The size is a build-time fact only the C
header knows, and it is not published anywhere a program can read it
at run time. A wrong number reaches `ERREXIT2`, which calls the error
manager's `error_exit`, whose default prints a line and calls `exit`.
Measured on the staging machine against `libjpeg.so.8`, passing 600
answered:

```
JPEG parameter struct mismatch: library thinks size is 584, caller expects 600
```

and the process ended there. The number 584 is a fact about that build
of that library on that architecture, so it cannot be written into a
binding either. The error manager is no way out: it is a structure of
C FUNCTION POINTERS the caller installs before the first call, and a
novo-lang function is not one, so the default `error_exit` that calls
`exit` cannot be replaced. **Without the header a novo-lang program
cannot make the first call of the libjpeg API, and nothing after it.**

Every TurboJPEG entry point takes an opaque handle, addresses and
scalars, reports a failure by answering -1, and keeps its error text on
the handle. That is the whole reason this package binds it.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Unverified

**The suite has never linked on the staging machine.** The machine
carries `libjpeg.so.8` and no libturbojpeg at all, so `novo test`
stops at `cannot find -lturbojpeg`. `novo pkg build` type-checks the
declarations and is green, and the test sources compile; the run is the
step nobody has seen. The `test` row of `novo pkg publish --dry-run` is
red for that reason and for no other, and the package reads shard 4/5.

### Named as missing

**The libjpeg API.** `jpeg_CreateCompress`, `jpeg_CreateDecompress`,
`jpeg_std_error`, `jpeg_start_compress`, `jpeg_write_scanlines`,
`jpeg_read_header`, `jpeg_read_scanlines` and the rest of
`libjpeg.so`, for the reason above.

**`tjTransform` and `tjInitTransform`.** The lossless transform — the
rotation, the flip and the crop that libjpeg-turbo is known for —
takes an array of `tjtransform` structures, and one field of that
structure is a C FUNCTION POINTER for a filter the caller supplies.
The field may be left null, so the call is not unbindable in
principle; it is left out because the structure's layout cannot be
verified on a machine that has no libturbojpeg to verify it against.

**The TurboJPEG 3 API.** `tj3Init`, `tj3Set`, `tj3Compress8`,
`tj3Decompress8` and their relatives arrived in libjpeg-turbo 3.0 and
replace the calls here, which that release keeps and marks
deprecated. This package binds the older set because it is in every
version of the library, including the ones the long-term distributions
ship.

**The file helpers.** `tj3LoadImage8` and `tj3SaveImage8` belong to
the TurboJPEG 3 API and read and write files, which a binding with no
logic of its own has no reason to do first.

**`tjGetErrorStr`.** The form without a handle reports the last
failure in the whole process, which is not safe to read from more than
one thread. `tjGetErrorStr2` takes the handle and is here.
