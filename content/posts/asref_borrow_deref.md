+++
title = "AsRef vs Borrow vs Deref Busted"
date = "2024-07-09"

[taxonomies]
tags=["coding"]

[extra]
repo_view = true
comment = true
+++

Havn't you come across the trait `AsRef<T>` , `Borrow` and `Deref` 

# AsRef vs Borrow vs Deref in Rust

## AsRef
`AsRef` is a trait for performing cheap reference-to-reference conversions. It's most commonly used for functions that can accept multiple types of string references.

```rust
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}
```

### Common Use Cases
```rust
fn process_string<T: AsRef<str>>(s: T) {
    let s_ref: &str = s.as_ref();
    // Process the string slice
}

// Can be called with different types
let string = String::from("hello");
let str_literal = "world";

process_string(&string);  // works
process_string(str_literal);  // also works
```

## Borrow
`Borrow` is similar to `AsRef` but with an additional contract: the borrowed form must hash, compare, and order the same as the owned form.

```rust
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

### Use Cases
```rust
use std::collections::HashMap;
use std::borrow::Borrow;

let mut map = HashMap::new();
map.insert(String::from("key"), 42);

// Can lookup with &str because String: Borrow<str>
assert_eq!(map.get("key"), Some(&42));
```

### Key Differences from AsRef
1. Stricter semantic requirements
2. Essential for hash maps and collections
```rust
// Borrow guarantees that these are equivalent:
hash(owned_value) == hash(borrowed_value)
owned_value == owned_value2 ⟺ borrowed_value == borrowed_value2
```

## Deref
`Deref` is used for dereferencing operations using the `*` operator and for automatic dereferencing.

```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

### Common Use Cases
```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    
    fn deref(&self) -> &T {
        &self.0
    }
}

let x = MyBox(5);
assert_eq!(5, *x);  // Deref coercion
```

### Deref Coercion
One of the most powerful features of `Deref`:
```rust
fn takes_str(s: &str) {
    println!("{}", s);
}

let string = String::from("hello");
takes_str(&string);  // Works because String: Deref<Target=str>
```

## Comparison Table

| Trait | Primary Purpose | Key Characteristics |
|-------|----------------|-------------------|
| `AsRef` | Cheap reference conversions | - No guarantees about equality or hashing<br>- Useful for generic functions accepting references |
| `Borrow` | Borrowed data equivalence | - Guarantees about equality and hashing<br>- Essential for collections like HashMap |
| `Deref` | Smart pointer functionality | - Enables `*` operator<br>- Enables deref coercion<br>- Affects method resolution |

## When to Use Each

### Use AsRef when:
- Writing generic functions that need reference conversions
- No need for hash/equality guarantees
```rust
fn print_contents<T: AsRef<str>>(s: T) {
    println!("{}", s.as_ref());
}
```

### Use Borrow when:
- Working with collections
- Need hash/equality guarantees
```rust
use std::collections::HashSet;

let set: HashSet<String> = ["a", "b"].iter().map(|&s| s.to_string()).collect();
assert!(set.contains("a")); // Works because of Borrow
```

### Use Deref when:
- Implementing smart pointer types
- Need automatic dereferencing behavior
```rust
struct SmartPtr<T>(Box<T>);

impl<T> Deref for SmartPtr<T> {
    type Target = T;
    fn deref(&self) -> &T { &self.0 }
}
```

## Common Gotchas

1. **Deref Coercion Costs**
```rust
// Multiple deref coercions can have performance implications
let s = Box::new(String::from("hello"));
takes_str(&s);  // Box -> String -> str, two dereferences
```

2. **Borrow vs AsRef Choice**
```rust
// Wrong choice for HashMap key type
fn bad_design<T: AsRef<str>>(map: &HashMap<T, u32>) {} // Won't work well

// Correct choice
fn good_design<T: Borrow<str>>(map: &HashMap<T, u32>) {}
```

3. **Deref for Non-Pointer Types**
```rust
// Generally avoid implementing Deref for non-pointer-like types
// Could be confusing for other developers
struct Seconds(u64);
// Avoid this:
impl Deref for Seconds {
    type Target = u64;
    fn deref(&self) -> &u64 { &self.0 }
}
```

This overview covers the main differences and use cases for these three important Rust traits. The key is understanding their specific purposes and contracts to use them effectively in your code.