+++
title = "Understanding stream in rust"
date = "2024-07-03"

[taxonomies]
tags=["low-level design"]

[extra]
repo_view = true
comment = true
+++

# Comprehensive Guide to Futures::Stream in Rust

## Introduction
`futures::Stream` is a fundamental abstraction in asynchronous Rust programming that represents a sequence of asynchronous values. This guide explores its design, implementation patterns, and practical considerations.

## Basic Concepts

### Stream Trait Definition
```rust
pub trait Stream {
    type Item;
    
    fn poll_next(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;
}
```

### Key Differences from Iterator
```rust
// Iterator (synchronous)
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Stream (asynchronous)
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) 
        -> Poll<Option<Self::Item>>;
}
```

## Implementation Patterns

### 1. Basic Stream Implementation
```rust
use futures::{Stream, StreamExt};
use std::pin::Pin;
use std::task::{Context, Poll};

struct CounterStream {
    current: u64,
    max: u64,
}

impl Stream for CounterStream {
    type Item = u64;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        _cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        if self.current >= self.max {
            return Poll::Ready(None);
        }
        
        let result = self.current;
        self.current += 1;
        Poll::Ready(Some(result))
    }
}
```

### 2. Buffered Stream Implementation
```rust
use std::collections::VecDeque;

struct BufferedStream<T> {
    buffer: VecDeque<T>,
    source: Box<dyn Stream<Item = T> + Unpin>,
    capacity: usize,
}

impl<T> BufferedStream<T> {
    fn new(source: impl Stream<Item = T> + Unpin + 'static, capacity: usize) -> Self {
        Self {
            buffer: VecDeque::with_capacity(capacity),
            source: Box::new(source),
            capacity,
        }
    }
}

impl<T> Stream for BufferedStream<T> {
    type Item = T;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        // Try to fill buffer
        while self.buffer.len() < self.capacity {
            match Pin::new(&mut self.source).poll_next(cx) {
                Poll::Ready(Some(item)) => self.buffer.push_back(item),
                Poll::Ready(None) => break,
                Poll::Pending => break,
            }
        }

        // Return item from buffer if available
        if let Some(item) = self.buffer.pop_front() {
            Poll::Ready(Some(item))
        } else if self.buffer.is_empty() {
            Pin::new(&mut self.source).poll_next(cx)
        } else {
            Poll::Pending
        }
    }
}
```

## Advanced Stream Patterns

### 1. Back-Pressure Implementation
```rust
struct BackPressuredStream<T> {
    inner: Pin<Box<dyn Stream<Item = T>>>,
    high_watermark: usize,
    current_pressure: usize,
}

impl<T> BackPressuredStream<T> {
    fn new(
        inner: impl Stream<Item = T> + 'static,
        high_watermark: usize
    ) -> Self {
        Self {
            inner: Box::pin(inner),
            high_watermark,
            current_pressure: 0,
        }
    }
}

impl<T> Stream for BackPressuredStream<T> {
    type Item = T;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        if self.current_pressure >= self.high_watermark {
            return Poll::Pending;
        }

        match self.inner.as_mut().poll_next(cx) {
            Poll::Ready(Some(item)) => {
                self.current_pressure += 1;
                Poll::Ready(Some(item))
            }
            other => other,
        }
    }
}
```

### 2. Rate-Limited Stream
```rust
use tokio::time::{Duration, Instant};

struct RateLimitedStream<S> {
    inner: S,
    rate: Duration,
    last_emit: Option<Instant>,
}

impl<S: Stream> Stream for RateLimitedStream<S> {
    type Item = S::Item;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        let now = Instant::now();
        
        if let Some(last_emit) = self.last_emit {
            let elapsed = now - last_emit;
            if elapsed < self.rate {
                // Not enough time has passed
                cx.waker().wake_by_ref();
                return Poll::Pending;
            }
        }

        match unsafe { Pin::new_unchecked(&mut self.inner) }.poll_next(cx) {
            Poll::Ready(Some(item)) => {
                self.last_emit = Some(now);
                Poll::Ready(Some(item))
            }
            other => other,
        }
    }
}
```

## Performance Considerations

### 1. Memory Management
```rust
struct ChunkedStream<S: Stream> {
    inner: S,
    chunk_size: usize,
    current_chunk: Vec<S::Item>,
}

impl<S: Stream> Stream for ChunkedStream<S> {
    type Item = Vec<S::Item>;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        while self.current_chunk.len() < self.chunk_size {
            match unsafe { Pin::new_unchecked(&mut self.inner) }.poll_next(cx) {
                Poll::Ready(Some(item)) => {
                    self.current_chunk.push(item);
                }
                Poll::Ready(None) if !self.current_chunk.is_empty() => {
                    return Poll::Ready(Some(
                        std::mem::replace(&mut self.current_chunk, Vec::new())
                    ));
                }
                Poll::Ready(None) => return Poll::Ready(None),
                Poll::Pending => return Poll::Pending,
            }
        }

        if self.current_chunk.len() == self.chunk_size {
            Poll::Ready(Some(
                std::mem::replace(&mut self.current_chunk, Vec::with_capacity(self.chunk_size))
            ))
        } else {
            Poll::Pending
        }
    }
}
```

### 2. CPU-Bound Operations
```rust
struct ComputeBoundStream<S, F> {
    inner: S,
    transform: F,
    runtime_handle: tokio::runtime::Handle,
}

impl<S, F, T, U> Stream for ComputeBoundStream<S, F>
where
    S: Stream<Item = T> + Unpin,
    F: Fn(T) -> U + Send + 'static,
    T: Send + 'static,
    U: Send + 'static,
{
    type Item = U;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        match Pin::new(&mut self.inner).poll_next(cx) {
            Poll::Ready(Some(item)) => {
                let transform = self.transform.clone();
                let handle = self.runtime_handle.clone();
                
                Poll::Ready(Some(handle.block_on(async move {
                    tokio::task::spawn_blocking(move || transform(item)).await.unwrap()
                })))
            }
            Poll::Ready(None) => Poll::Ready(None),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

## Error Handling

### 1. Recoverable Errors
```rust
struct RetryStream<S> {
    inner: S,
    retry_count: usize,
    max_retries: usize,
}

impl<S, T, E> Stream for RetryStream<S>
where
    S: Stream<Item = Result<T, E>> + Unpin,
    E: std::error::Error,
{
    type Item = Result<T, E>;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        loop {
            match Pin::new(&mut self.inner).poll_next(cx) {
                Poll::Ready(Some(Ok(item))) => {
                    self.retry_count = 0;
                    return Poll::Ready(Some(Ok(item)));
                }
                Poll::Ready(Some(Err(e))) => {
                    if self.retry_count < self.max_retries {
                        self.retry_count += 1;
                        continue;
                    }
                    return Poll::Ready(Some(Err(e)));
                }
                other => return other,
            }
        }
    }
}
```

### 2. Unrecoverable Errors
```rust
struct FallibleStream<S> {
    inner: S,
    error_handler: Box<dyn Fn(Box<dyn std::error::Error>) + Send>,
}

impl<S: Stream> Stream for FallibleStream<S> {
    type Item = S::Item;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        match unsafe { Pin::new_unchecked(&mut self.inner) }.poll_next(cx) {
            Poll::Ready(Some(item)) => Poll::Ready(Some(item)),
            Poll::Ready(None) => Poll::Ready(None),
            Poll::Pending => {
                // Handle potential errors during pending state
                Poll::Pending
            }
        }
    }
}
```

## Testing Strategies

### 1. Mock Streams
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use futures::stream;
    use std::time::Duration;

    struct MockStream<T> {
        items: Vec<T>,
        delay: Duration,
    }

    impl<T: Clone> Stream for MockStream<T> {
        type Item = T;

        fn poll_next(
            mut self: Pin<&mut Self>, 
            cx: &mut Context<'_>
        ) -> Poll<Option<Self::Item>> {
            if self.items.is_empty() {
                Poll::Ready(None)
            } else {
                tokio::time::sleep(self.delay).await;
                Poll::Ready(Some(self.items.remove(0)))
            }
        }
    }

    #[tokio::test]
    async fn test_stream_behavior() {
        let mock = MockStream {
            items: vec![1, 2, 3],
            delay: Duration::from_millis(100),
        };

        let mut stream = BufferedStream::new(mock, 2);
        
        assert_eq!(stream.next().await, Some(1));
        assert_eq!(stream.next().await, Some(2));
        assert_eq!(stream.next().await, Some(3));
        assert_eq!(stream.next().await, None);
    }
}
```

### 2. Property-Based Testing
```rust
#[cfg(test)]
mod tests {
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn test_stream_properties(inputs: Vec<i32>) {
            let stream = stream::iter(inputs.clone());
            let buffered = BufferedStream::new(stream, 10);
            
            tokio_test::block_on(async {
                let collected: Vec<_> = buffered.collect().await;
                prop_assert_eq!(inputs, collected);
            });
        }
    }
}
```

## Best Practices and Optimizations

### 1. Stream Fusion
```rust
struct FusedStream<S> {
    inner: S,
    is_terminated: bool,
}

impl<S: Stream> Stream for FusedStream<S> {
    type Item = S::Item;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        if self.is_terminated {
            return Poll::Ready(None);
        }

        match unsafe { Pin::new_unchecked(&mut self.inner) }.poll_next(cx) {
            Poll::Ready(None) => {
                self.is_terminated = true;
                Poll::Ready(None)
            }
            other => other,
        }
    }
}
```

### 2. Batch Processing
```rust
struct BatchProcessor<S> {
    inner: S,
    batch_size: usize,
    current_batch: Vec<S::Item>,
}

impl<S: Stream> Stream for BatchProcessor<S> {
    type Item = Vec<S::Item>;

    fn poll_next(
        mut self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>> {
        while self.current_batch.len() < self.batch_size {
            match unsafe { Pin::new_unchecked(&mut self.inner) }.poll_next(cx) {
                Poll::Ready(Some(item)) => {
                    self.current_batch.push(item);
                }
                Poll::Ready(None) if !self.current_batch.is_empty() => {
                    return Poll::Ready(Some(
                        std::mem::replace(&mut self.current_batch, Vec::new())
                    ));
                }
                Poll::Ready(None) => return Poll::Ready(None),
                Poll::Pending => {
                    if !self.current_batch.is_empty() {
                        return Poll::Ready(Some(
                            std::mem::replace(&mut self.current_batch, Vec::new())
                        ));
                    }
                    return Poll::Pending;
                }
            }
        }

        Poll::Ready(Some(
            std::mem::replace(&mut self.current_batch, Vec::with_capacity(self.batch_size))
        ))
    }
}
```

## Pros and Cons

### Pros:
1. **Asynchronous Processing**
   - Natural fit for I/O-bound operations
   - Efficient resource utilization
   - Support for back-pressure

2. **Composability**
   - Easy to chain transformations
   - Rich set of combinators
   - Integration with async/await

3. **Memory Efficiency**
   - Lazy evaluation
   - Controlled buffering
   - Stream fusion opportunities

### Cons:
1. **Complexity**
   - More complex than Iterator
   - Pin requirements
   - Error handling complexity

2. **Runtime Overhead**
   - Task scheduling overhead
   - Memory allocation for futures
   - Context switching costs

3. **Learning Curve**
   - Understanding async concepts
   - Dealing with lifetimes
   - Managing pinning

## Conclusion

`futures::Stream` is a powerful abstraction for async programming in Rust, offering:
- Efficient handling of asynchronous sequences
- Rich composition capabilities
- Good performance characteristics

However, it requires careful consideration of:
- Implementation complexity
- Memory management
- Error handling strategies
- Performance implications

The key to successful usage is understanding these trade-offs and choosing appropriate patterns for your specific use case.

Remember to:
- Profile your stream implementations
- Test edge cases thoroughly
- Consider backpressure mechanisms
- Handle errors appropriately
- Optimize for your specific use case

This comprehensive overview shoul