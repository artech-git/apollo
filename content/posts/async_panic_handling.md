+++
title = "Dealing with Panics 🔥 in async rust 🦀"
date = "2024-12-9"

[taxonomies]
tags=["low-level design", "programming", "coding"]

[extra]
repo_view = true
comment = true
+++

# Intro
You might have come across the term "fearless concurrency" quite often if you have some background in Rust, but nothing is absolute in this world, and neither is concurrency. Additionally, it's very unlikely to find the term "**fearless async concurrency**" in Rust's ecosystem. So it's becomes nessecary to know how to manage some spooky panics, Let's delve deeper into this.

---
# Problem at Hand

While working with some async codebases, your logical definition of code might have some quirky or spooky possibility of panicking somewhere or at some part of the code. This could be anticipated in advance or maybe totally undetected. As illustrated with the example below, let's study it to find out more about it.

```rust

async fn fetch_all_post(n: u32) {
    let pagination_factor = 4; 
    let pagination_stub_factor: u32 = (n / 4) as u32;
    let url = format!("https://random_domain.com/posts/pagination/{}", pagination_stub_factor);

    let _res = reqwest::get(url.into()).await; 
}

```

In the above example, we can see that we are trying to determine the pagination factor for a given URL (just as an example). If you are a bit experienced, you may have already spotted the problem present here! Let's look at those:

- What if `pagination_factor` is greater than `n`?
- There could be problems in casting `f32` to `u32`.

We will focus on the first point since it's easier to prove the point. Imagine that you update your code with some assert statement like this, which could panic if any value we supply is checked against `pagination_factor`.

```rust
async fn fetch_all_post(n: u32) {
    let pagination_factor = 4; 
    assert!(n >= pagination_factor);
    let pagination_stub_factor: u32 = (n / 4) as u32;
    let url = format!("https://random_domain.com/posts/pagination/{}", pagination_stub_factor);

    let _res = reqwest::get(url.into()).await; 
}
```

Hmm, looks fine now but we still havn't eliminated the possibilty of panicking in our code

```bash
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 2.25s
     Running `target/debug/playground`
thread 'main' panicked at src/main.rs:16:5:
assertion failed: n >= pagination_factor
```
This was just a very straight forward example we saw, but in real world we have have something as serious as socket unavailable, file not found or what not! 

Somehow we should be able to control the consequence of panic, we will study those in the coming sections.

---
# Async Runtime and Panic Handling Strategy

If you are writing async Rust, then you have to choose an async executor which will run your futures/async tasks forward. Below is a list of some widely adopted and often used executors:

- tokio
- async-std
- smol-rs

Each async executor has its own strategy for dealing with panics that occur within an async block.

With Tokio, if a panic occurs, it isolates the given task. The runtime doesn't shut down (though this behavior can be modified) and continues to operate while printing the standard error message to standard out, simply restarting.

The case with async-std is the same as above, where tasks are isolated based on how they are spawned. If a panic occurs in some task, it won't crash the entire runtime, and other tasks would simply restart and continue from that point onwards.

For smol-rs, it deals with panics similarly to how the above two handle them, except with some internal exceptions.

But mostly, for all the tasks (futures) which are spawned using these runtimes, a handle is returned which can carry the panic message as an `Err` variant as described below.

```rust
//assuming we are in some async context or block already 

let join_handle: tokio::task::JoinHandle<()> = tokio::task::spawn(
    async move { 
        panic!("some error") //we panic inside our task 
    }
    );
// we get a result type whose error variant is of std::any::Any type
let res: Result<(), Box<dyn std::any::Any>> = join_handle.await; 
```
Looking at the above example, we find that during panics, it gets converted to the trait object `std::any::Any`, which can universally represent any type. Later, you may downcast it based on your error handling strategy. As you can see, the runtime does give you these abilities, and this is valid for most runtimes since these tasks are isolated from each other.

Note: Error types passed from `panic!(..)` might look easy to deal with, but it is an expensive operation to convert from `Box<dyn std::any::Any>` to some other type. If you are designing your application to be performant, you may require something more direct that you can control.

---
# std::panic::catch_unwind to your rescue

We briefly looked over how runtimes deal with futures that may panic, but luckily we have more control already available if you wish to avoid the route of runtime handling panics of your future.

To give it a simple start, there can be some suspicious part of the code where we suspect that it might panic somehow! Let's check out the previous example again.

```rust
1 async fn fetch_all_post(n: u32) {
2    let pagination_factor = 4; 
3    assert!(n >= pagination_factor);
4    let pagination_stub_factor: u32 = (n / 4) as u32;
5    let url = format!("https://random_domain.com/posts/pagination/{}", pagination_stub_factor);
6
7    let _res = reqwest::get(url.into()).await; 
}
```
Here we know line number 3 might panic based on user input. To prevent this from happening, we can simply wrap this part of the code in the `std::panic::catch_unwind` function like this:

```rust
1 async fn fetch_all_post(n: u32) {
2    let url = std::panic::catch_unwind(||
3        let pagination_factor = 4; 
4        assert!(n >= pagination_factor);
5        let pagination_stub_factor: u32 = (n / 4) as u32;
6        format!("https://random_domain.com/posts/pagination/{}", pagination_stub_factor)
7    );
8
9    match url_block {
10        Ok(url_val) => {
11            let _res = reqwest::get(url_val.into()).await; 
12        }
13        Err(err) => {
14            println!("some error: {}", err); 
15        }
16    };
}
```

So what's changed now 🤔? We wrapped lines 2-5 (of the original method) into the `catch_unwind` method and returned the computed URL by performing matching on it since it is a result type. If line 4 panics, we won't crash or show an error message on our async runtime. Instead, this will be handled within our code by ourselves.
<br/>

Let's now check out the definition of `std::panic::catch_unwind` from the std library:

```rust
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R>
```

Here you will observe two generics: `F`, which accepts a closure and returns `R`, which should also implement the `UnwindSafe` trait (we will discuss this in more detail). Interestingly, the return type here has to be the following:

```rust
pub type Result<T> = Result<T, Box<dyn Any + Send + 'static>>;
``` 
Basically, the Result type is aliased to have the error variant be thread-safe if encountered.

Hmm, so now you see how powerful `std::panic::catch_unwind` can be. Remember, this is just a single example; there could be several more possibilities depending on how you wish to handle such panics.

---
# Using catch_unwind with your futures

Earlier, we saw how we can manually establish control over panics and produce a Result type as a consequence, but something even more convenient is already available for you from the `Futures` crate. This is the [`catch_unwind()`](https://docs.rs/futures/latest/futures/future/trait.FutureExt.html#method.catch_unwind) method, which creates an intermediate object [`CatchUnwind<Fut>`](https://docs.rs/futures/latest/futures/future/struct.CatchUnwind.html) that implements [`futures::future::Future`](https://docs.rs/futures/latest/futures/future/trait.Future.html). Hence, after awaiting it, it produces a result as shown in the description below.

```rust
impl<Fut> Future for CatchUnwind<Fut>
where
    Fut: Future + UnwindSafe,
{
    //notice the output type here 
    type Output = Result<Fut::Output, Box<dyn Any + Send>>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        ...
    }
}
```
For the associated type `Output`, it returns the `Ok` variant with the output for which this was called, and the `Err` variant is again a `Box<dyn Any + Send>` trait object that can be sent anywhere. Now let's see it in action.

```rust
use futures::future::FutureExt; 

1 async fn fetch_all_post(n: u32) {
2
3    let catch_url_block: Result<String, Box<dyn Any + Send>> = async move {
4       let pagination_factor = 4; 
5       assert!(n >= pagination_factor);
6       let pagination_stub_factor: u32 = (n / 4) as u32;
7       let url = format!("https://random_domain.com/posts/pagination/{}", pagination_stub_factor);
8    }.catch_unwind().await;      // we apply catch_unwind to this
9
10    match url_block {
10        Ok(url_val) => {
11            let _res = reqwest::get(url_val.into()).await; 
12        }
13        Err(err) => {
14            println!("some error: {}", err); 
15        }
16    };
17 }
```
Look at line number 8, you will find that we have used `.catch_unwind()`. Now we can also call `.await` on the `CatchUnwind` object. We can apply `catch_unwind` on any type that implements the `Future` trait. This can be smartly chained with other futures to create a more complex expression that can be later awaited for a response.

Let's check out an example of `catch_unwind` with streams:

```rust
1 
2 let mut client = futures::stream::unfold(0, |count| async move { 
3     let next_count = count + 1; 
4    // peform some expensive work ...
5    Some((count, next_count)) // this will return count from it
6 });
7
8 let mut responses: Vec<i32> = client
9        .catch_unwind()
10        .filter_map(|val| val.ok()) // convert Result<i32, Box<dyn Any + Send>> to Option<i32>
11        .take(50)
12        .collect()
13        .await;
14
```

In the above example, you see how we are using streams in the future to compute something. You might wonder how `catch_unwind` is implementable on something like `future::stream::StreamExt`. Note that this method is also available under [`futures::stream::StreamExt`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.catch_unwind). On line 10, we convert `Result<i32, Box<dyn Any + Send>>` to `Option<i32>` type simply because the [`filter_map`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.filter_map) method requires us to pass any data encapsulated as an option type. Then, we take 50 of those and collect them.

This example was too straightforward, honestly, but the quirky nature of this method is not yet revealed. Let's check out a more realistic example that does something more interesting...


```rust
let mut client = futures::stream::unfold(0, |count| async move { 
        let next_count = count + 1; 
        let url = format!("https://dummyjson.com/quotes/random");
        let req = reqwest::get(url).await; // if this panics !
        Some((req, next_count))
    });

async fn filter_posts(
    catch_unwind_req: Result<Result<Response, reqwest::Error>, Box<dyn Any + Send>>
) -> Option<String> {
    match catch_unwind_req {
        Ok(Ok(response)) => {
            response.text().await.ok() //convert from Result<T,E> to Option<T>
        }
        _ => {
            None
        }
    }
}
let mut responses: Vec<String> = client
        .catch_unwind()
        .filter_map(filter_posts)
        .take(50)
        .collect()
        .await;

```
Above is a good example regarding how `catch_unwind` could be potentially used in this!, Let's run this first 

```bash
|     fn catch_unwind(self) -> CatchUnwind<Self>
     |        ------------ required by a bound in this associated function
1324 |     where
1325 |         Self: Sized + std::panic::UnwindSafe,
     |                       ^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `StreamExt::catch_unwind`
     
```
An error, hmmm... This is one of the problems that shows up. Let's revisit the definition of `catch_unwind` again:

```rust
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R>
```

You will see the `UnwindSafe` trait has to be applied to the return type! If you visit the definition of [`reqwest::Response`](https://docs.rs/reqwest/latest/reqwest/struct.Response.html#impl-UnwindSafe-for-Response), it contains a negative implementation for `!UnwindSafe`, hence it can't be sent across the `catch_unwind` boundary.

Now you might think, is there a way around this? Well, the answer is not that simple, but there exists a solution! Welcome [`std::panic::AssertUnwindSafe`](https://doc.rust-lang.org/beta/std/panic/struct.AssertUnwindSafe.html#examples).

```rust
pub struct AssertUnwindSafe<T>(pub T);

impl<T> UnwindSafe for AssertUnwindSafe<T> {}
```
Type `std::panic::AssertUnwindSafe` implements the `UnwindSafe` trait even if the type contains `!UnwindSafe`. So, if a type implements `!UnwindSafe`, it can be wrapped in this type and returned from the `catch_unwind` boundary.

Coming back to our previous example, let's modify it to support the usage of `AssertUnwindSafe` here:

```rust
// we wrap our UnFold type into AssertUnwindSafe
let mut client = AssertUnwindSafe(futures::stream::unfold(0, |count| async move { 
        let next_count = count + 1; 
        let url = format!("https://dummyjson.com/quotes/random");
        let req = reqwest::get(url).await; // if this panics !
        Some((req, next_count))
    }));

async fn filter_posts(
    catch_unwind_req: Result<Result<Response, reqwest::Error>, Box<(dyn Any + Send + 'static)>>
) -> Option<String> {
    match catch_unwind_req {
        Ok(Ok(response)) => {
            response.0.text().await.ok() //convert from Result<T,E> to Option<T>
        }
        _ => {
            None
        }
    }
}

let mut responses: Vec<String> = client
        .catch_unwind() // check each post incoming
        .filter_map(filter_posts)
        .take(50)
        .collect()
        .await;
```
We have modified the `client` type to be wrapped under `AssertUnwindSafe`. Under the hood, the `futures::stream::unfold` method returns an object [`future::stream::Unfold`](https://docs.rs/futures/latest/futures/stream/struct.Unfold.html). Here, we have also utilized `filter_map` for filtering out, but there is a catch! Let's see that too!

If we explicitly perform a panic just for the sake of testing, we will find something very interesting happening. The example below is taken from the official docs!


```rust
use futures::stream::{self, StreamExt};

let stream = stream::iter(vec![Some(10), None, Some(11)]);
// Panic on second element
let stream_panicking = stream.map(|o| o.unwrap());
// Collect all the results
let stream = stream_panicking.catch_unwind();

let results: Vec<Result<i32, _>> = stream.collect().await;
match results[0] {
    Ok(10) => {}
    _ => panic!("unexpected result!"),
}
assert!(results[1].is_err());
assert_eq!(results.len(), 2);
```
This is something where you can find it problematic to deal with. The official docs for [`futures::stream::catch_unwind`](https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.catch_unwind) mention the following:

**Caught panic (if any) will be the last element of the resulting stream**

So you have to know how to use it correctly if you wish to utilize this.

---
# Panic hooks 
Remember the topic we covered earlier about async runtime panic handling? That topic only contained information regarding how the runtime internally handled panics. For example, to go more in-depth, the Tokio runtime builder object [tokio::runtime::Builder::unhandled_panic](https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.unhandled_panic) gives you a method for dealing with panics that might happen within Tokio's async runtime context. Though unstable, it is still available.

Interestingly, we have another technique available specifically to deal with this: introducing panic hooks.

```rust
pub fn set_hook(hook: Box<dyn Fn(&PanicHookInfo<'_>) + Sync + Send + 'static>)
```

If somewhere within your codebase there happens to be a panic, this hook will be called automatically. The signature of this method is straightforward; it accepts a boxed closure with a signature where the type must be `Send` and `Sync` compatible. Let's see an example of this.

```rust
    std::panic::set_hook(Box::new(|panic_info| {
    
        let timestamp = chrono::Local::now().format("%Y-%m-%d %H:%M:%S");
        let backtrace = backtrace::Backtrace::new();
        
        let panic_message = format!("[{}] Panic: {}\nBacktrace:\n{:?}\n\n", 
            timestamp, panic_info, backtrace);
            
        // Log to file
        if let Ok(mut file) = OpenOptions::new()
            .create(true)
            .append(true)
            .open("panic.log") 
        {
            let _ = file.write_all(panic_message.as_bytes());
        }
        }));

    fn process_request() {
       panic!("Database connection failed"); // Example panic
    }

```
In the above example, we are calling a panic hook to generate a timestamp and backtrace, then format this and write it out to a file. Essentially, we are logging any panics that occur. No matter at what point in the code a panic is triggered, `set_hook` will be invoked automatically.

`set_hook` is more powerful in a way that it supports obtaining references to the type which was passed under `panic` macro, here is a example for that

```rust
panic::set_hook(Box::new(|panic_info| {
    if let Some(s) = panic_info.payload().downcast_ref::<&str>() {
        println!("panic occurred: {s:?}");
    } else if let Some(s) = panic_info.payload().downcast_ref::<String>() {
        println!("panic occurred: {s:?}");
    } else {
        println!("panic occurred");
    }
}));

panic!("Normal panic");
```

In the above, we panic with a `&str` message and later call the `payload` method over it, which returns `&(dyn Any + Send)`. We can then apply `downcast_ref` to obtain a read-only reference for that type. Here is another example with a `Vec<i32>`:

```rust
std::panic::set_hook(Box::new(|panic_info| {
        if let Some(s) = panic_info.payload().downcast_ref::<Vec<i32>>() {
            println!("panic occurred: {:?}", s);
        } else {
            println!("panic occurred with unknown payload");
        }
    }));
    
let val = vec![1,2,3,4,5];
    
std::panic::panic_any(val);
```

If you notice, in the last line we have utilized the [`std::panic::panic_any`](https://doc.rust-lang.org/std/panic/fn.panic_any.html) method rather than the `panic` macro. This is because the `panic` macro only allows string literals to be passed within itself, whereas `std::panic::panic_any` allows us to use `'static + Any + Send` as our argument. If you are wondering what the purpose of `std::any::Any` is, you may check out [this](https://quinedot.github.io/rust-learning/dyn-any.html) blog post.

I hope you learnt something interesting and useful through this blog post, feel free to share your thoughts with me ;) 