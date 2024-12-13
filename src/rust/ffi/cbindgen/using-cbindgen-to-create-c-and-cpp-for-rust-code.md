# Using cbindgen to Create C and C++ Bindings for Rust Code

<!-- toc -->

Rust's safety guarantees and performance make it an excellent choice for systems programming, but sometimes you need to interface with existing C or C++ codebases. This is where `cbindgen` comes in - a tool that automatically generates C/C++ bindings for your Rust code. In this post, I'll walk you through how to use cbindgen effectively.

## What is cbindgen?

`cbindgen` is a tool that parses your Rust crate and generates C or C++ header files, allowing you to call Rust functions from C or C++ code. It handles the translation of Rust types to their C/C++ equivalents and manages the FFI (Foreign Function Interface) boundary.

## Setting Up a Project

Let's start by setting up a simple project:

```bash
mkdir -p cbindgen_example
cd cbindgen_example
cargo init --lib
```

## Writing Rust Code with FFI Exports

First, let's create some Rust functions that we want to expose to C/C++:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(left: u64, right: u64) -> u64 {
    left + right
}

#[unsafe(no_mangle)]
pub extern "C" fn add_in_place(left: *mut u64, right: u64) {
    // check for nullity of `left`
    // and takes a mutable reference on it if it's non-null
    if let Some(left) = unsafe { left.as_mut() } {
        *left += right;
    }
}
```

The key elements here are:

- `#[no_mangle]` - Prevents Rust from mangling the function name
- `#[unsafe(no_mangle)]` - Calling from other language is unsafe, so we added `unsafe` here
- `extern "C"` - Specifies the C calling convention
- Using C-compatible types (like raw pointers)

### Handling Complex Types

Let's add a more complex example with a custom type:

```rust
pub struct Counter(u32);

impl Counter {
    fn new() -> Self {
        Counter(0)
    }

    fn get(&self) -> u32 {
        self.0
    }

    fn incr(&mut self) -> bool {
        if let Some(n) = self.0.checked_add(1) {
            self.0 = n;
            true
        } else {
            false
        }
    }
}

// C-compatible API
#[unsafe(no_mangle)]
pub extern "C" fn counter_new() -> *mut Counter {
    // Box allocates memory on the heap and Box::into_raw converts it to a raw pointer
    Box::into_raw(Box::new(Counter::new()))
}

#[unsafe(no_mangle)]
pub extern "C" fn counter_incr(counter: *mut Counter) -> std::os::raw::c_int {
    if let Some(counter) = unsafe { counter.as_mut() } {
        if counter.incr() { 0 } else { -1 }
    } else {
        -2
    }
}

#[unsafe(no_mangle)]
pub extern "C" fn counter_get(counter: *const Counter) -> u32 {
    if let Some(counter) = unsafe { counter.as_ref() } {
        return counter.get();
    }
    return 0;
}

#[unsafe(no_mangle)]
pub extern "C" fn counter_destory(counter: *mut Counter) -> std::os::raw::c_int {
    if !counter.is_null() {
        let _ = unsafe { Box::from_raw(counter) }; // get box and drop
        return 0;
    }
    return -1;
}
```

This example demonstrates how to expose a Rust struct to C/C++ by providing C-compatible functions that operate on raw pointers to the struct.

## Configuring Cargo.toml

To build your Rust code as a C-compatible library, you need to add `cbindgen` as a build dependency.

Run `cargo add cbindgen --build` to add `cbindgen` to dependency.

<details>
    <summary>Click to see output!</summary>

```bash
cargo add cbindgen --build
    Updating crates.io index
      Adding cbindgen v0.28.0 to build-dependencies
             Features:
             + clap
             - unstable_ir
    Updating crates.io index
     Locking 56 packages to latest Rust 1.85.1 compatible versions
      Adding anstream v0.6.18
      Adding anstyle v1.0.10
      Adding anstyle-parse v0.2.6
      Adding anstyle-query v1.1.2
      Adding anstyle-wincon v3.0.7
      Adding bitflags v2.9.0
      Adding cbindgen v0.28.0
      Adding cfg-if v1.0.0
      Adding clap v4.5.36
      Adding clap_builder v4.5.36
      Adding clap_lex v0.7.4
      Adding colorchoice v1.0.3
      Adding equivalent v1.0.2
      Adding errno v0.3.11
      Adding fastrand v2.3.0
      Adding getrandom v0.3.2
      Adding hashbrown v0.15.2
      Adding heck v0.4.1
      Adding indexmap v2.9.0
      Adding is_terminal_polyfill v1.70.1
      Adding itoa v1.0.15
      Adding libc v0.2.172
      Adding linux-raw-sys v0.9.4
      Adding log v0.4.27
      Adding memchr v2.7.4
      Adding once_cell v1.21.3
      Adding proc-macro2 v1.0.95
      Adding quote v1.0.40
      Adding r-efi v5.2.0
      Adding rustix v1.0.5
      Adding ryu v1.0.20
      Adding serde v1.0.219
      Adding serde_derive v1.0.219
      Adding serde_json v1.0.140
      Adding serde_spanned v0.6.8
      Adding strsim v0.11.1
      Adding syn v2.0.100
      Adding tempfile v3.19.1
      Adding toml v0.8.20
      Adding toml_datetime v0.6.8
      Adding toml_edit v0.22.24
      Adding unicode-ident v1.0.18
      Adding utf8parse v0.2.2
      Adding wasi v0.14.2+wasi-0.2.4
      Adding windows-sys v0.59.0
      Adding windows-targets v0.52.6
      Adding windows_aarch64_gnullvm v0.52.6
      Adding windows_aarch64_msvc v0.52.6
      Adding windows_i686_gnu v0.52.6
      Adding windows_i686_gnullvm v0.52.6
      Adding windows_i686_msvc v0.52.6
      Adding windows_x86_64_gnu v0.52.6
      Adding windows_x86_64_gnullvm v0.52.6
      Adding windows_x86_64_msvc v0.52.6
      Adding winnow v0.7.6
      Adding wit-bindgen-rt v0.39.0
```

</details>

The `Cargo.toml` file will looks like this:

```toml
[package]
name = "cbindgen_example"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[build-dependencies]
cbindgen = "0.28.0"
```

The `crate-type = ["cdylib"]` line tells Cargo to build a dynamic library that can be used from C.

## Installing cbindgen

Add cbindgen to your development dependencies, so that you can call `cbindgen` manually to generate c/cpp header files:

```bash
cargo install cbindgen
```

## Configuring cbindgen

Create a `cbindgen.toml` file in your project root to configure how your bindings are generated:

```toml
# Basic configuration
language = "C++"  # or "C" for C bindings

# Header customization
header = "/* Copyright (c) 2023 Your Name */"
include_guard = "RUST_BINDINGS_H"
pragma_once = true
autogen_warning = "/* Warning, this file is autogenerated by cbindgen. Don't modify this manually. */"

# Code style
braces = "SameLine"
line_length = 100
tab_width = 2
documentation = true

# C++ specific options (ignored when generating C bindings)
namespace = "rust"
namespaces = []
```

You can also refer to https://raw.githubusercontent.com/mozilla/cbindgen/refs/heads/master/template.toml for more configuration.

## Generating Bindings

Now you can generate the bindings manually by running `cbindgen` binary:

```bash
cbindgen --config cbindgen.toml --output bindings.h
```

Or, for more automation, add a build script (`build.rs`) to your project:

```rust
extern crate cbindgen;

use std::env;

fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    // Generate C bindings
    cbindgen::Builder::new()
        .with_crate(&crate_dir)
        .with_language(cbindgen::Language::C)
        .generate()
        .expect("Unable to generate bindings")
        .write_to_file("bindings.h");

    // Generate C++ bindings
    cbindgen::Builder::new()
        .with_crate(&crate_dir)
        .generate()
        .expect("Unable to generate bindings")
        .write_to_file("bindings.hpp");
}
```

This build script will automatically generate both C and C++ bindings whenever you build your Rust project.

## Using the Generated Bindings in C

Here's how you might use your Rust functions from C:

```c
#include <stdio.h>
#include "bindings.h"

int main() {
    // Call the add function
    uint64_t result = add(5, 7);
    printf("5 + 7 = %llu\n", result);

    // Use the add_in_place function
    uint64_t value = 10;
    add_in_place(&value, 5);
    printf("10 + 5 = %llu\n", value);

    // Use the Counter API
    struct Counter *counter = counter_new();
    counter_incr(counter);
    counter_incr(counter);
    printf("Counter value: %u\n", counter_get(counter));
    counter_destory(counter);

    return 0;
}
```

## Using the Generated Bindings in C++

And here's how you might use them from C++:

```cpp
#include <iostream>
#include "bindings.hpp"

int main() {
    // Call the add function
    uint64_t result = add(5, 10);
    std::cout << "5 + 10 = " << result << std::endl;

    // Use the add_in_place function
    uint64_t value = 10;
    add_in_place(&value, 20);
    std::cout << "10 + 20 = " << value << std::endl;

    // Use the Counter API
    Counter *counter = counter_new();
    counter_incr(counter);
    counter_incr(counter);
    counter_incr(counter);
    std::cout << "Counter value: " << counter_get(counter) << std::endl;
    counter_destory(counter);

    return 0;
}
```

## Building and Linking

To compile and link your C/C++ code with your Rust library:

1. Build your Rust library:

```bash
cargo build --release
```

2. Compile your C/C++ code and link against the Rust library:

```bash
# For C
gcc main.c -o add-c -L./target/release -lcbindgen_example

# For C++
g++ main.cpp -o add-cpp -L./target/release -lcbindgen_example
```

You can also use a Makefile to automate this process:

```makefile
build:
	cargo build

build-cpp:
	g++ main.cpp -o add-cpp -L./target/debug -lcbindgen_example

build-c:
	gcc main.c -o add-c -L./target/release -lcbindgen_example

run-cpp: build-cpp
	./add-cpp

run-c: build-c
	./add-c
```

## Advanced cbindgen Features

### Type Mapping

You can customize how Rust types are mapped to C/C++ types:

```toml
[export]
# Prefix all exported symbols
prefix = "RUST_"

[export.rename]
# Rename specific items
"RustStruct" = "CStruct"

[parse]
# Parse dependencies
parse_deps = true
# Include only specific items
include = ["my_module"]
# Exclude specific items
exclude = ["internal_module"]
```

### Handling Opaque Types

Sometimes you want to expose a Rust type to C/C++ without exposing its internal structure. You can do this by using opaque types:

```rust
// In Rust
#[repr(C)]
pub struct OpaqueType {
    // Fields not visible to C/C++
    private_field: i32,
}

#[no_mangle]
pub extern "C" fn create_opaque() -> *mut OpaqueType {
    Box::into_raw(Box::new(OpaqueType { private_field: 42 }))
}
```

In the generated C header, `OpaqueType` will be declared as an incomplete type:

```c
// In C
typedef struct OpaqueType OpaqueType;

OpaqueType* create_opaque(void);
```

### Handling Callbacks

You can also pass callbacks from C/C++ to Rust:

```rust
// In Rust
pub type Callback = extern "C" fn(i32) -> i32;

#[no_mangle]
pub extern "C" fn call_with_callback(callback: Callback, arg: i32) -> i32 {
    callback(arg)
}
```

```c
// In C
typedef int (*Callback)(int);

int call_with_callback(Callback callback, int arg);

int my_callback(int x) {
    return x * 2;
}

int main() {
    int result = call_with_callback(my_callback, 5);
    printf("Result: %d\n", result);  // Prints "Result: 10"
    return 0;
}
```

## Conclusion

cbindgen is a powerful tool that makes it easy to create C and C++ bindings for your Rust code. By following the steps outlined in this post, you can seamlessly integrate Rust into your existing C/C++ projects or create new libraries that can be used from multiple languages.

Remember that when working with FFI, you're responsible for ensuring memory safety across the language boundary. Rust's safety guarantees only apply within Rust code, so be careful when passing pointers or handling resources that cross the FFI boundary.

Happy coding!
