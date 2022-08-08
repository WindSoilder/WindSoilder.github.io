## Make nushell read binary

One month ago [jt](https://github.com/jntrnr) shared an idea about reading png data, in short, if we can implement something like this, it'd be helpful for users to parse binary data.

```
open myfile.png | find IHDR | read bits 32 32
```

I think that's a good idea, start my research about it.

### TL;DR
Please check [nu_plugin_bin_reader](https://github.com/WindSoilder/nu_plugin_bin_reader) to see what [nushell](https://www.nushell.sh/) can do for binary format.

### Research journey
The first thing come to mind is that I can use awesome rust [image](https://docs.rs/image/latest/image/) crate to achieve this, we can read metadata from `xxx.png`, and use the crate to parse the binary file, finally fetch the information we want.

But what if we want to read other binary file format?  Like `ttf`, I need to find other `ttf` parsing lib, and make it into `nushell` somehow.

And `IHDR` is only valid for `png` file, not valid in `ttf` file, different file have different header and body definition.

The solution is not scale.

Then I start google to check if there are something exists for parsing binary data in general...I get something like [deku](https://github.com/sharksforarms/deku), [binrw](https://github.com/jam1garner/binrw), [nom](https://github.com/Geal/nom), all of them seems interesting, and it seems that I can write parser thought one of these lib, then we can make nushell achieve it.

But it feels somehow I'm just reimplement binary parsing logic, and these binary format are well defined and unlikely to be changed, that's not really good.

Finally I find [kaitai struct](https://kaitai.io/), here is the introduction:

> Reading and writing binary formats is hard, especially if it’s an interchange format that should work across a multitude of platforms and languages.  Have you ever found yourself writing repetitive, error-prone and hard-to-debug code that reads binary data structures from files or network streams and somehow represents them in memory for easier access?  Kaitai Struct tries to make this job easier — you only have to describe the binary format once and then everybody can use it from their programming languages — cross-language, cross-platform.

Wow, it seems so promising, and it supports so much format in their [gallery](https://formats.kaitai.io/), that's what I want.

### Implementation
Firstly I want to implement it as nushll's builtin command, but sadly found that `rust` supports for `kaitai` is not good.  But it supports `python` well, how can I implement the parsing logic in python and use it inside nushell directly?

[Nushell plugin system](https://www.nushell.sh/book/plugins.html) comes into play!  After reading the awesome documentation and [python plugin example](https://github.com/nushell/nushell/blob/main/crates/nu_plugin_python/plugin.py), it can be implemented as plugin.

Finally here comes the repo: [nu_plugin_bin_reader](https://github.com/WindSoilder/nu_plugin_bin_reader)

It empowers nushell to read binary data, and it supports the following feature:
1. supports 100+ binary formats, if there are more formats available in the gallery, this plugin can auto support it without changing source code
2. provides some nushell commands to download parsing lib dynamically, it's easy to support new binary format.
