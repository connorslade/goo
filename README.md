# `goo` [![Build](https://github.com/connorslade/goo/actions/workflows/build.yml/badge.svg)](https://github.com/connorslade/goo/actions/workflows/build.yml) [![Latest Version](https://img.shields.io/crates/v/goo)](https://crates.io/crates/goo) [![Downloads](https://img.shields.io/crates/d/goo?label=Downloads)](https://crates.io/crates/goo)

> [!IMPORTANT]  
> This crate has been deprecated in favor of my new [msla_format](https://crates.io/crates/msla_format) crate, which supports additional formats (.ctb and .nanodlp), and is auto generated from the [mslicer](https://github.com/connorslade/mslicer) source tree so it can be more easily updated.
> This crate will no longer be updated.

Library for encoding and decoding Elegoo's `.goo` file format.
This is a stand alone version of this crate taken from my [mslicer](https://github.com/connorslade/mslicer) project, its an open source slicer for masked stereolithography printers.

If you want to learn more about the goo format, make sure you read the [official format spec](https://github.com/elegooofficial/GOO).
Some things aren't mentioned in the spec like how everything is big-endian, the checksum is just the negated sum of all bytes in the payload, and the image data encoding specification is hard to follow, so look at [my implementation](https://github.com/connorslade/goo/blob/main/src/encoded_layer.rs).
Also, if you use [ImHex](https://imhex.werwolv.net), I have created a pattern file ([goo.hexpat](https://github.com/connorslade/goo/blob/main/goo.hexpat)) that may be helpful.

## Examples

For some real world examples, check out the following links to my mslicer project source code:

- [Slicing a triangular mesh into a layer](https://github.com/connorslade/mslicer/blob/15d69c7c9f6bd921d8517d81047aa29b18ba4f92/slicer/src/slicer.rs#L72)
- [Decoding a sliced file for a layer preview](https://github.com/connorslade/mslicer/blob/15d69c7c9f6bd921d8517d81047aa29b18ba4f92/mslicer/src/windows/slice_operation.rs#L142)

### Encoding an Image

Because thousands of very high resolution images take many tens of gigabytes to store, goo files use a run length encoding mechanism as a simple form of compression.
In order to encode an image into a goo layer, you therefore must separate it into runs first.
In this example ill just add alternating rows of white and black.

It is important to note that you must define a value for every pixel.
This is because on my printer at least the buffer that each layer is decoded into is uninitialized memory.
So if the last run doesn't fill the buffer, the printer will just print whatever was in the buffer before which just makes a huge mess.

```rust
use goo::{LayerEncoder, LayerDecoder, Run};

let size = (11520, 5102);
let mut out = LayerEncoder::new();

for y in 0..size.1 {
    let color = if y % 2 == 0 { 255 } else { 0 };
    out.add_run(size.0, color);
}

let (bytes, checksum) = out.finish();

let decoder = LayerDecoder::new(&bytes);
assert_eq!(checksum, decoder.checksum());

for run in decoder {
    println!("{:?}", run);
}
```

### Decoding to Images

This example is a simplified version of the included example to inspect sliced goo files.
Note that setting each pixel individually is not very fast, but it was enough for a debugging tool.

```rust
use std::fs;

use goo::{GooFile, LayerDecoder, Run};
use image::RgbImage;

let raw_goo = fs::read("input.goo")?;
let goo = GooFile::deserialize(&raw_goo)?;
println!("{:#?}", goo.header);

for (i, layer) in goo.layers.iter().enumerate() {
    let decoder = LayerDecoder::new(&layer.data);
    let mut pixel = 0;

    if layer.checksum != decoder.checksum() {
        eprintln!("WARN: Checksum mismatch for layer {}", i);
    }

    let mut image = RgbImage::new(
        goo.header.x_resolution as u32,
        goo.header.y_resolution as u32,
    );

    for Run { length, value } in decoder {
        let color = image::Rgb([value, value, value]);
        for _ in 0..length {
            let x = pixel % goo.header.x_resolution as u32;
            let y = pixel / goo.header.x_resolution as u32;

            image.put_pixel(x, y, color);
            pixel += 1;
        }
    }

    image.save(format!("layer_{:03}.png", i))?;
}
```
