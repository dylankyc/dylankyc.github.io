# Extract Enum Variants in Rust

When working with large Rust enums, sometimes we need to extract just the variant names without their associated data. In this tutorial, we'll explore two approaches to extract enum variants: using Rust with the `syn` crate and using Python with regex.

## The Problem

Let's say we have a large enum like `TokenInstruction` that looks like this:

```rust
pub enum TokenInstruction<'a> {
    InitializeMint {
        decimals: u8,
        mint_authority: Pubkey,
        freeze_authority: COption<Pubkey>,
    },
    InitializeAccount,
    // ... many more variants
}
```

And we want to extract just the variant names to get something like:

```rust
pub enum TokenInstruction<'a> {
    InitializeMint {},
    InitializeAccount {},
    // ... other variants
}
```

## Solution 1: Using Rust with syn

The first approach uses Rust's `syn` crate to parse the code as an AST (Abstract Syntax Tree) and extract the variants. Here's how we can do it:

```rust
use proc_macro2::TokenStream;
use quote::quote;
use std::env;
use std::fs;
use syn::parse_str;
use syn::Item;

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Usage: {} <rust_file>", args[0]);
        std::process::exit(1);
    }

    let file_path = &args[1];
    let content = match fs::read_to_string(file_path) {
        Ok(content) => content,
        Err(e) => {
            eprintln!("Error reading file: {}", e);
            std::process::exit(1);
        }
    };

    let ast = match syn::parse_file(&content) {
        Ok(ast) => ast,
        Err(e) => {
            eprintln!("Error parsing Rust file: {}", e);
            std::process::exit(1);
        }
    };

    // Iterate AST to search for enum
    for item in ast.items {
        if let Item::Enum(item_enum) = item {
            let enum_name = &item_enum.ident;
            let generics = &item_enum.generics;

            // print enum header part
            println!("pub enum {}{} {{", enum_name, quote!(#generics));

            // print each variant
            for variant in &item_enum.variants {
                let variant_name = &variant.ident;
                println!("    {} {{}},", variant_name);
            }

            // print end
            println!("}}");
        }
    }
}
```

This approach is more robust as it properly handles Rust syntax and preserves the enum's generic parameters.

## Solution 2: Using Python with Regex

For a simpler but less robust approach, we can use Python with regular expressions:

```python
import sys
from pathlib import Path
import re

def clean_rust_code(content):
    # remove multiline comments
    content = re.sub(r'///.*?\n', '\n', content, flags=re.MULTILINE)
    # remove single comments
    content = re.sub(r'//.*?\n', '\n', content)
    return content

def extract_enum_variants(file_path):
    with open(file_path, 'r') as f:
        content = f.read()

    # clean comments
    content = clean_rust_code(content)

    # extract enum name and generic parameter
    enum_pattern = re.compile(r'pub\s+enum\s+(\w+)(<.*?>)?')
    enum_match = enum_pattern.search(content)
    if not enum_match:
        print("No enum found")
        return

    enum_name = enum_match.group(1)
    enum_generic = enum_match.group(2) or ''

    # extract variants
    variant_pattern = re.compile(r'\s+(\w+)(?:\s*{[^}]*}|\s*,)')
    variants = variant_pattern.findall(content)

    # build the output
    output = [f"pub enum {enum_name}{enum_generic} {{"]
    for variant in variants:
        output.append(f"    {variant} {{}},")
    output.append("}")

    print("\n".join(output))

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python extract_enum.py <file>")
        sys.exit(1)

    file_path = sys.argv[1]
    if not Path(file_path).exists():
        print(f"File {file_path} does not exist")
        sys.exit(1)

    extract_enum_variants(file_path)
```

## Comparing the Approaches

| Approach              | Pros                                                                                                  | Cons                                                                                                           |
| --------------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Rust with syn**     | • Properly handles Rust syntax<br>• Maintains generic parameters<br>• More reliable for complex enums | • Requires additional dependencies<br>• More complex implementation                                            |
| **Python with regex** | • Simpler implementation<br>• No Rust-specific dependencies<br>• Faster to implement                  | • Less robust<br>• May break with complex Rust syntax<br>• Regex patterns might need adjustment for edge cases |

## Conclusion

Both approaches can help you extract enum variants, but choose based on your needs:

- Use the Rust approach for production code or when dealing with complex Rust syntax
- Use the Python approach for quick scripts or simple enums

Remember that the Rust approach using `syn` is generally more reliable as it properly parses the Rust syntax tree, while the Python regex approach is more suitable for quick, one-off tasks.
