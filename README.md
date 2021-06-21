<p align="center">
<img width="330" height="110" src=".github/logo.png" border="0" alt="kelindar/bitmap">
<br>
<img src="https://img.shields.io/github/go-mod/go-version/kelindar/bitmap" alt="Go Version">
<a href="https://pkg.go.dev/github.com/kelindar/bitmap"><img src="https://pkg.go.dev/badge/github.com/kelindar/bitmap" alt="PkgGoDev"></a>
<a href="https://goreportcard.com/report/github.com/kelindar/bitmap"><img src="https://goreportcard.com/badge/github.com/kelindar/bitmap" alt="Go Report Card"></a>
<a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License"></a>
<a href="https://coveralls.io/github/kelindar/bitmap"><img src="https://coveralls.io/repos/github/kelindar/bitmap/badge.svg" alt="Coverage"></a>
</p>


# Zero-Allocation, SIMD Bitmap Index (Bitset) in Go

This package contains a bitmap index which is backed by `uint64` slice, easily encodable to/from a `[]byte` without copying memory around so it can be present
in both disk and memory. As opposed to something as [roaring bitmaps](https://github.com/RoaringBitmap/roaring), this is a simple implementation designed to be used for small to medium dense collections.

I've used this package to build a columnar in-memory datastore, so if you want to see how it can be used for indexing, have a look at [kelindar/column](https://github.com/kelindar/column). I'd like to specifically point out the indexing part and how bitmaps can be used as a good alternative to B*Trees and Hash Maps.

## Features

 * **Zero-allocation** (see benchmarks below) on almost all of the important APIs.
 * **1-2 nanosecond** on single-bit operations (set/remove/contains).
 * **SIMD enhanced** `and`, `and not`, `or` and `xor` operations, specifically using `AVX2` (256-bit).
 * Support for `and`, `and not`, `or` and `xor` which allows you to compute intersect, difference, union and symmetric difference between 2 bitmaps.
 * Support for `min`, `max`, `count`, and `first-zero` which is very useful for building free-lists using a bitmap index.
 * Reusable and can be pooled, providing `clone` with a destination and `clear` operations.
 * Can be encoded to binary without copy as well as optional stream `WriteTo` method.
 * Support for iteration via `Range` method and filtering via `Filter` method.

## Example Usage

In its simplest form, you can use the bitmap as a bitset, set and remove bits. This is quite useful as an index (free/fill-list) for an array of data.

```go
import "github.com/kelindar/bitmap"
```

```go
bitmap := make(bitmap.Bitmap, 0, 8) // 8*64 = 512 elements pre-allocated
bitmap.Set(300)         // sets 300-th bit
bitmap.Set(400)         // sets 400-th bit
bitmap.Set(600)         // sets 600-th bit (auto-resized)
bitmap.Contains(300)    // returns true
bitmap.Contains(301)    // returns false
bitmap.Remove(400)      // clears 400-th bit

// Min, Max, Count
min, ok := bitmap.Min()  // returns 300
max, ok := bitmap.Max() // returns 600
count := bitmap.Count() // returns 2
```

The bits in the bitmap can also be iterated over using the `Range` method. It is a simple loop which iterates over and calls a callback. If the callback returns false, then the iteration is halted (similar to `sync.Map`).

```go
// Iterate over the bits in the bitmap
bitmap.Range(func(x uint32) bool {
    println(x)
    return true
})
```

Another way of iterating is using the `Filter` method. It iterates similarly to `Range` but the callback returns a boolean value, and if it returns `false` then the current bit will be cleared in the underlying bitmap. You could accomplish the same using `Range` and `Remove` but `Filter` is significantly faster.

```go
// Filter iterates over the bits and applies a callback
bitmap.Filter(func(x uint32) bool {
    return x % 2 == 0
})
```

Bitmaps are also extremely useful as they support boolean operations very efficiently. This library contains `And`, `AndNot`, `Or` and `Xor`.

```go
// And computes the intersection between two bitmaps and stores the result in the current bitmap
a := Bitmap{0b0011}
a.And(Bitmap{0b0101})

// AndNot computes the difference between two bitmaps and stores the result in the current bitmap
a := Bitmap{0b0011}
a.AndNot(Bitmap{0b0101})

// Or computes the union between two bitmaps and stores the result in the current bitmap
a := Bitmap{0b0011}
a.Or(Bitmap{0b0101})

// Xor computes the symmetric difference between two bitmaps and stores the result in the current bitmap
a := Bitmap{0b0011}
a.Xor(Bitmap{0b0101})
```

## Benchmarks
Benchmarks below were run on a large pre-allocated bitmap (slice of 100 pages, 6400 items).

```
cpu: Intel(R) Core(TM) i7-9700K CPU @ 3.60GHz
BenchmarkBitmap/set-8         	608127316     1.979 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/remove-8      	775627708     1.562 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/contains-8    	907577592     1.299 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/clear-8       	231583378     5.163 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/ones-8        	39476930      29.77 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/first-zero-8  	23612611      50.82 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/min-8         	415250632     2.916 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/max-8         	683142546     1.763 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/count-8       	33334074      34.88 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/clone-8       	100000000     11.46 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/simd-and-8    	74337927      15.47 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/simd-andnot-8 	80220294      14.92 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/simd-or-8     	81321524      14.81 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/simd-xor-8    	80181888      14.81 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/and-8         	29650201      41.68 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/andnot-8      	26496499      51.72 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/or-8          	20629934      50.83 ns/op     0 B/op    0 allocs/op
BenchmarkBitmap/xor-8         	23786632      51.46 ns/op     0 B/op    0 allocs/op
```
