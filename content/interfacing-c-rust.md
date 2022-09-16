+++
title = "Using C library in Rust"
description = "In this blog post, I will describe a very simple way to call C functions from Rust"
draft = false
date = "2022-09-16"
template = "page.html"
menu = "sandesh"
+++

## Introduction: 

C functions can be directly called from Rust using some linker settings. In this post, we will take a look at how to do the same.

## Writing and compiling the code(s):

Our C function will be a trivial function that returns an integer for now: 

```c
int somefunction() {
    return 15;
}
```

The function does not take in anything but returns the value of "15". Let's compile this code and turn it into a C library. 

```bash 
gcc -c test.o test.c
ar -cvq libtest.a *.o
```

A library file "libtest.a" will be created in the current directory after doing this. 
Let's continue to the Rust code. But first, in our `Cargo.toml` file, let's include `libc` as a dependency.

```rust
extern crate libc;
use libc::int64_t;

#[link(name="test")]
extern {
    fn somefunction() -> int64_t;
}

fn main() {
    let value = unsafe { somefunction() };
    println!("Got a returned value {}.", value);
}
```

More information regarding the FFI can be found in the [official rust docs](https://www.cs.brandeis.edu/~cs146a/rust/doc-02-21-2015/book/ffi.html).
The `#[link]` attribute instructs the linker to link against a `libtest` file which we need to instruct the rustc linker to find.a

## Linking and running our code. 
There's two ways we can do the final link and run: 
**Using `RUSTFLAGS`:**
Inside our Rust Project, run the following command: 
```bash
RUSTFLAGS="-L <path_to_libtest.a>" cargo build
cargo run
```

**Using `build.rs`:**
Inside our Rust project, in Cargo.toml, in the project section, add the following line: 
```
[package]
name=".."
...
build = "build.rs"
...
```

Here, we added the "build.rs" file. This instructs the cargo compiler to use the build.rs file to customize our build. 
Now, inside the build.rs file, let's add the following code section: 

```rust
fn main(){
    println!("cargo:rustc-link-search=<path_to_libtest.a>");
}
```

After this, 

```bash
cargo build && cargo run
```

should work just fine.
