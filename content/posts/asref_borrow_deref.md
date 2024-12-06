+++
title = "AsRef vs Borrow vs Deref Busted"
date = "2024-12-06"

[taxonomies]
tags=["coding"]

[extra]
repo_view = true
comment = true
+++

It is common to encounter a situation where you may require to return a different type either as a immutable or mutable reference which can be a field of a struct or a completely new type composed out of a given type for some operation

You may also have come across such type expression `impl AsRef<str>` or `where T: Borrow<[u8]>` or something like `impl Deref<Target = [u8]>` these kinds of expression are common to come across which can be tricky sometimes to understand and draw a distinction among it.

It's quite common to come across these traits in standard library `AsRef<T>` / `AsMut<T>`, `Borrow` / `BorrowMut` and `Deref` / `DerefMut`.

Unless you have good background with rust, The first thought which may come to mind after looking at these traits is they are naturally made for producing a subtype or newtype from a given type when called! well this is unfortunately not true. If this was true then standard library won't be having these many traits!  

# Similarity
But before we start to look over there **characteristics** and **differences**, Let's cover there similarities first, keep this in mind following similarities are not strongly same but similar or close to being similar depending upon situation, and in some simple situations could be used interchangibely, let's check them out !

 - Simplification of borrow: All the above traits are used to simplify access to data. They allow for different ways of referring to an object, typically without taking ownership of real one.
    - with AsRef & AsMut
    ```rust
    impl AsRef<Car> for FourWheeler {
        ...
    } 
    ```
    - with Borrow trait 
    ```rust
    impl Borrow<Car> for FourWheeler {
        ...
    }
    ```
    - with Deref and DerefMut 
    ```rust
    impl Deref for FourWheeler {
        type Target = Car; 
        ...
    }
    ```
<!-- 
 - Implicit Coercion in Function Calls: Rust uses these traits to perform implicit coercion of types, allowing functions to accept arguments of different types more flexibly through the usage `Deref` and `AsRef` traits. For example, functions that take a parameter of type &T where T: AsRef<U> can accept any argument that can be referenced as &U.
    - with AsRef -->
 - Conversion support to Non-sized types: Each trait definition allows conversion to non-sized types, as shown in the definition of each trait bellow, you will notice this.
   - With `AsRef` & `AsMut` 
   ```rust
   pub trait AsRef<T>
    where
        T: ?Sized, // Non-sized type
    {
        // Required method
        fn as_ref(&self) -> &T;
    }
    ```
    ```rust
    pub trait AsMut<T>
    where
        T: ?Sized, // Non-sized type
    {
        // Required method
        fn as_mut(&mut self) -> &mut T;
    }
    ```
   - With `Borrow` & `BorrowMut` trait
   ```rust
   pub trait Borrow<Borrowed>
    where
        Borrowed: ?Sized, // Non-sized type
    {
        // Required method
        fn borrow(&self) -> &Borrowed;
    }
    ```
    ```rust
    pub trait BorrowMut<Borrowed>: Borrow<Borrowed>
    where
        Borrowed: ?Sized, // Non-sized type
    {
        // Required method
        fn borrow_mut(&mut self) -> &mut Borrowed;
    }
    ```
   - With `Deref` & `DerefMut` 
   ```rust
    pub trait Deref {
        type Target: ?Sized; // Non-sized type

        // Required method
        fn deref(&self) -> &Self::Target;
    }
    ```
    ```rust
    pub trait DerefMut: Deref {
        // Required method
        fn deref_mut(&mut self) -> &mut Self::Target;
    }
    ```
 - Used in Generic Programming: All three traits are highly useful in generic programming, where you want to write functions or structs that can operate on references to data rather than the data itself.
    - With `AsRef` & `AsMut` you may have the impelmentation for any given type such following
    ```rust
    //case 1 
    impl AsRef<str> for T { 
        ...
    } 
    //case 2
    fn compute<T>(s: impl AsRef<T>) where T: ToString ..

    //case 3 
    struct Car<I> {
        tires: Box<dyn AsRef<I>>
        ...
    }
    ```
    - with `Borrow` & `BorrowMut` traits you can still acheive same exact thing as above
  
    - with `Deref` & `DerefMut` things get bit special due to associated type in it's signature definition
    ```rust
    //case 1
    impl<Y> Deref for SomeType 
     where Y: std::fmt::Debug {
        type Target = Y;
        fn deref(&self) -> &Self::Target {
            ...
        } 
    }
    //case 2
    fn accept_deref<U>(s: impl Deref<Target=U>) where U: ToString ...

    //case 3
    struct Truck<I> {
        tires: Box<dyn Deref<Target = I>>
        ...
    }
    ```


# AsRef vs Borrow vs Deref in Rust

## AsRef & AsMut trait
`AsRef` is a trait for performing cheap reference-to-reference conversions. This includes read and write borrows! It's most commonly used for functions that can accept multiple types of concrete references. Thought it can be used to borrow Non-sized types too! 

let's checkout a simple example first! 
In the below example we are trying to borrow a type as a `str` but we are allowed to passed each type as a reference as illustrated below 

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
In above example we have seen how we can pass around all those types which does `AsRef` implementation are now qualified to be passed as a owned type and later using `as_ref()` method we can call that type to be represented into intented type as a read only reference.

Since there exists implementation for `&str` and `String` type as shown below
```rust
impl AsRef<str> for String {
    #[inline]
    fn as_ref(&self) -> &str {
        self
    }
}
```
and for `str` type 

```rust
impl AsRef<str> for str {
    #[inline(always)]
    fn as_ref(&self) -> &str {
        self
    }
}
```

Interestingly if you observe carefully here you will find that `AsRef` doesn't writes following type signature for the generic position `AsRef<&str>` instead it writes `AsRef<str>`, well if you are wondering why this is check out these two posts respectively [Rust nomicon reference](https://doc.rust-lang.org/nomicon/exotic-sizes.html) and [Niko matsakis](https://smallcultfollowing.com/babysteps/blog/2024/04/23/dynsized-unsized/), to understand the difference, now if you look on trait function `fn as_ref(&self) -> &str` it returns a borrowed version, Frankly the explanation behind how `str` is borrowed into `&str` deserves it's own post, but to give a short answer Inside the method, `self` is already of type `&str` because we're implementing a method that takes `&self`, so we can just return it directly since it matches the return type.

Now let's move our attention towards `AsMut` which let's us borrow a type mutably.
If you visit standard library you will find there exists implementation for `AsMut` for `str` & `String` types, 
these implementations are as follows

for string type
```rust
impl AsMut<str> for String {
    #[inline]
    fn as_mut(&mut self) -> &mut str {
        self
    }
}
```
for str type
```rust
impl AsMut<str> for str {
    #[inline(always)]
    fn as_mut(&mut self) -> &mut str {
        self
    }
}
```
Interestingly this time we borrow both of our types as mutable reference `&mut str` 

By now Iam hoping this would have bought some clarity regarding how `AsRef` & `AsMut` trait could be used, but the real power of these trait lies in flexibility regarding how they can be used 

Pay attention to how `AsRef` & `AsMut` are totally independent of each other, meaning they don't have any associated type dependency or super trait requirement unlike we previously saw for `Borrow`, `BorrowMut`, `Deref` & `DerefMut` traits, Hence we are given complete flexibity in regards how one Concrete type could be cheaply converted into another type, let's understand this explnation through a example

let's think of a situation where you wish to convert a books type `Book` into some primitive representation such `&str` but as a immutable borrow only & also as `Vec<String>` but as a mutable reference.
```rust
struct Book {
    title: String,
    pages: Vec<String>,
}
```

writing a simple `AsRef` conversion is possible, given if it **makes sense to use that type temporarly**

```rust
impl AsRef<str> for Book {
    fn as_ref(&self) -> &str {
        &self.title  // We can reference the title as a &str
    }
}
```
in a simple manner it is possible to write a `AsMut<Vec<String>>` conversion as well
```rust
// Implement AsMut<Vec<String>> for Book to modify pages
impl AsMut<Vec<String>> for Book {
    fn as_mut(&mut self) -> &mut Vec<String> {
        &mut self.pages  // We can get a mutable reference to pages
    }
}
```
following from previous, it is possible to write a `AsMut<[str]>` conversion as well
```rust
impl AsMut<[String]> for Book {
    fn as_mut(&mut self) -> &mut [String] {
        &mut self.pages  // We can get a mutable reference to pages
    }
}
```

### Points to note while using AsRef & AsMut

- Usage of these traits is best suited when you have to derive a type which is mostly a field of a struct and conversion doesn't involve any expensive computation 
- These traits gives you flexibility in usage, meaning you may have a single `AsRef` for some type, where as you may have severals `AsMut` for distinct types and they don't depend on each other in any way ! 
- It's better to avoid writing these traits for `Non-Sized` types such as `&dyn ..` or `&mut dyn..`  



## Borrow & BorrowMut
`Borrow` trait is similar to `AsRef` trait in a way that it supports borrowing of type A into type B, but the catch lies that it let's you borrow rather than converting something, Hence point of comparsion shall be between **how to distinguish between borrow and conversion**

**Conversion of any type to another type is straight forward since it doesn't enforces us to carry any further more semantic information, while borrowing shall satisfy such norms**

From above information `Borrow` is similar to `AsRef` but with an additional contract: the borrowed form must **hash, compare, and order the same as the owned form**.

Let's checkout it's definition first

```rust
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```
In above definition we have declared out borrow to be a generic just like `AsRef` which can be Non-sized type and borrows immutably, but we would also need to borrow mutably, let's checkout the definition for `BorrowMut` trait in detail

```rust
pub trait BorrowMut<Borrowed>: Borrow<Borrowed>
where
    Borrowed: ?Sized,
{
    // Required method
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
```
For `BorrowMut` definition as we can clearly observe, it depends over `Borrow<Borrowed>` implementation which is generic over `Borrowed`, therefore types which are going to implement `BorrowMut<Borrowed>` must first implement immutable version i.e. `Borrow<Borrowed>` this enforces a type to be borrowable in both forms i.e. `&T` & `&mut T` , unlike `AsRef` & `AsMut` trait which can be totally off to each other and there doesn't exists any correlation between them.

Let's checkout `std::collections::HashMap` definition regarding `Borrow` trait 

### Use Cases
```rust
// struct definition
pub struct HashMap<K, V, S = RandomState>

// method get definition 
pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
where
    K: Borrow<Q>, // Observe how Borrow trait is used 
    Q: Hash + Eq,
{
    ...
}
```
The type definition of `HashMap<K, V>` is generic over three params but we will only count `K` & `V` which are for key and value respectively for simplicity, now if get look on method `get` it declares a another generic `Q` which can be borrowed from `k` and `Q` must implement `Hash + Eq` traits as bounds, Since it requires that keys can be distinguishible.

This requirement is present for multiple methods of collections types within std library.

**Note: `Borrow` & `BorrowMut` doesn't themselves enforces the semantics requirement through the definition, it is upto the defining method or struct to declare how semantics are going to be enforced via them**

Borrow traits can also be useful for types where we want to retreive a internal type rather than composed one, such as for `Vec<T>` which can be borrowed as mutable slice `&mut [T]`  and wish to modify over that rather than going through the route of converting it first

### Points to note
1. Stricter semantic requirements, though not required by default.
2. Essentially used for collections types such as HashMap and BtreeMap.
3. You require to work on subtype of some composed type which might require modification as well. 
4. There may exist several definitions for a given type to be borrowed in different types.


## Deref & DerefMut
The trait `Deref` is one of my personel favorite and quite powerful one, among all.
`Deref` is used for dereferencing operations using the `*` operator and for automatic dereferencing to a defined type.
unlike traits `Borrow` & `AsRef` ,deref carries a associated type as part of it's definition and can only be applied once.
Let's checkout it's definition. 

```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```
In the above definition it returns a associated type as immutable reference, now let's take a look for `DerefMut` 
```rust
pub trait DerefMut: Deref {
    // Required method
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```
as you can clearly see it just like `BorrowMut`, `DerefMut` also depends over `Deref` to be pre-implemented for a type and interestingly the return type belongs from super trait `Deref` which can be borrowed mutably, this has majorly two advantages
  
  - Output type for both borrows can't be changed since it is based upon associated type.
  - Prevent's the implementation to exist for more than one type

To find out more about how associated types are different generics I recommend you to read this blog [here](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)

Once a type impelement deref traits can now automatically call the methods of derefed types using dot `.` operator

let's look at a simple example here
### Common Use Cases
```rust
use std::ops::Deref;

struct MyBox(Vec<u8>);

impl MyBox {
    pub fn my_box_len(&self) -> usize {
        self.0.len()
    }
}

impl Deref for MyBox {
    type Target = Vec<u8>;
    
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

let mut x = MyBox(vec![4]);

// valid
let len_one = x.my_box_len(); 

// also valid, we are directly Vec::len() method since it only takes &self
let len = x.len() 

```
Incase we require some mutable borrows 

```rust
impl DerefMut for MyBox {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}
```
now we can call any mutable method over `Vec<u8>` 
```rust
let _val = x.remove(0); // x -> .remove()
```

Those types which implement `Deref` trait are automatically qualified to use `*` operator for explicit dereferencing, 
```rust
let _val = (*x).remove(0);  // x -> Vec<u8> -> .remove()
```
Now you might wonder why are both expression are making valid call to `remove`, once a type implements `Deref` trait it will attempt to call the respective method through implicit derefencing **If required** and will make multiple attempts if the subtype too implements `Deref` 

here is a an straight forward exmaple, 

```rust
let _val: Option<&mut u8> = x.last_mut(); // x -> Vec<u8> -> &mut [u8]
```
If you visit `Vec<u8>` you will find that it implements [`Deref<Target=[u8]>`](https://doc.rust-lang.org/std/vec/struct.Vec.html#deref-methods-%5BT%5D) hence all the methods on &mut [u8] are directly accessible to us 

this is very useful and if not used cautiously could be the cause for bugs sometimes ! since you may call methods on some other type other than desired type! 
Luckily in such a case only methods which are first observed are called through implicty dereference are called 

### Deref Coercion
One of the most powerful features of `Deref` is onsite coercion if a type implement deref traits, this could be helpful where we wish to avoid explicit casting/ conversions:
```rust
fn takes_str(s: &str) {
    println!("{}", s);
}

let string = String::from("hello");
takes_str(&string);  // Works because String: Deref<Target=str>
takes_str("string2");  // also works because it is a concrete type
```
but hold a second, why would we need a such onsite coercion feature if we already have `AsRef` & `Borrow` traits to take care of job, well glad you asked !

If you observe the above definition carefully, you will notice it is a concrete type which is `&str` but for Asref & Borrow traits you will need to have type which must implement it anyhow, therefore limiting the application and optimizations available.

Deref coercion is a powerful language feature if used correctly, you can read more about it [here](https://dev.to/artechgit/rust-deref-coercion-simplifying-borrowing-and-dereferencing-1a4a)

Now that we have built some understanding around it, let's quickly compare them through a simple table.
## Comparison Table

| Trait | Primary Purpose | Key Characteristics |
|-------|----------------|-------------------|
| `AsRef` | Cheap reference conversions | - No guarantees about equality or hashing<br>- Useful for generic functions accepting references |
| `Borrow` | Borrowed data equivalence | - Guarantees about equality and hashing<br>- Essential for collections types like HashMap/BtreeMap |
| `Deref` | Smart pointer functionality | - Enables `*` operator<br>- Enables deref coercion<br>- Affects method resolution |


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

## Summary
This overview covers the main differences and use cases for these three important Rust traits. The key is understanding their specific purposes and contracts to use them effectively in your code.