+++
title = "Understanding panic handling in larger codebase"
date = "2024-07-05"

[taxonomies]
tags=["low-level design"]

[extra]
repo_view = true
comment = true
+++

# Comprehensive Guide to Panic Handling in Async Rust

## Introduction
Panic handling in async Rust requires special consideration, especially in larger codebases where reliability and graceful error handling are crucial. This guide explores various aspects of panic handling, including best practices, common scenarios, and practical solutions.

## Understanding Panics in Rust

### What is a Panic?
```rust
fn example_panic() {
    panic!("This is a panic!");  // Explicit panic
    let v = vec![1, 2, 3];
    v[99];  // Runtime panic: index out of bounds
}
```

### Common Panic Scenarios
1. **Array/Vector Index Out of Bounds**
```rust
fn array_panic() {
    let arr = [1, 2, 3];
    let index = 5;
    println!("{}", arr[index]);  // PANIC!
}
```

2. **Integer Overflow in Debug Mode**
```rust
fn overflow_panic() {
    let x: u8 = 255;
    let y = x + 1;  // PANIC in debug mode
}
```

3. **Unwrap on None**
```rust
fn unwrap_panic() {
    let opt: Option<i32> = None;
    opt.unwrap();  // PANIC!
}
```

## Panic Handling in Async Context

### The Challenge with Async Panics
Async code introduces additional complexity because panics can occur in different contexts:

1. **Task Panics**
```rust
async fn task_panic() {
    tokio::spawn(async {
        panic!("Task panic!");
    });
}
```

2. **Future Panics**
```rust
async fn future_panic() -> Result<(), Box<dyn std::error::Error>> {
    let future = async {
        panic!("Future panic!");
    };
    future.await
}
```

### Implementing Robust Panic Handling

#### 1. Using catch_unwind in Async Context
```rust
use std::panic::{catch_unwind, AssertUnwindSafe};

async fn handle_async_panic() -> Result<(), Box<dyn std::error::Error>> {
    let result = catch_unwind(AssertUnwindSafe(async {
        // Potentially panicking async code
        panic!("Async panic!");
    })).await;

    match result {
        Ok(_) => Ok(()),
        Err(e) => {
            eprintln!("Caught panic: {:?}", e);
            Err("Async operation panicked".into())
        }
    }
}
```

#### 2. Task-Level Panic Handling with Tokio
```rust
use tokio::task::JoinHandle;

async fn spawn_supervised_task() -> Result<(), Box<dyn std::error::Error>> {
    let handle: JoinHandle<Result<(), Error>> = tokio::spawn(async {
        // Your task code here
        Ok(())
    });

    match handle.await {
        Ok(Ok(_)) => Ok(()),
        Ok(Err(e)) => Err(e.into()),
        Err(e) if e.is_panic() => {
            eprintln!("Task panicked: {:?}", e);
            Err("Task panic".into())
        }
        Err(e) => Err(e.into()),
    }
}
```

### Custom Panic Hooks for Application-Wide Handling

```rust
use std::panic;
use log::{error, info};

fn setup_panic_hooks() {
    panic::set_hook(Box::new(|panic_info| {
        error!("Application panic occurred!");
        error!("Panic info: {:?}", panic_info);
        
        // Optional: Send panic info to monitoring service
        send_to_monitoring(panic_info);
        
        // Optional: Initiate graceful shutdown
        initiate_shutdown();
    }));
}

fn send_to_monitoring(panic_info: &panic::PanicInfo) {
    // Implementation for sending to monitoring service
}

fn initiate_shutdown() {
    // Implementation for graceful shutdown
}
```

## Structured Panic Handling for Large Applications

### 1. Panic Recovery Middleware
```rust
use std::future::Future;
use std::pin::Pin;

pub struct PanicRecoveryMiddleware<S> {
    inner: S,
}

impl<S> PanicRecoveryMiddleware<S> {
    pub fn new(inner: S) -> Self {
        Self { inner }
    }

    pub async fn handle_request<F, R>(&self, fut: F) -> Result<R, Error>
    where
        F: Future<Output = R> + Send + 'static,
        R: Send + 'static,
    {
        let result = tokio::spawn(async move {
            catch_unwind(AssertUnwindSafe(async move {
                fut.await
            })).await
        }).await??;

        Ok(result)
    }
}
```

### 2. Supervisor Pattern for Long-Running Tasks
```rust
pub struct TaskSupervisor {
    tasks: Vec<JoinHandle<()>>,
    shutdown_tx: broadcast::Sender<()>,
}

impl TaskSupervisor {
    pub fn new() -> Self {
        let (shutdown_tx, _) = broadcast::channel(1);
        Self {
            tasks: Vec::new(),
            shutdown_tx,
        }
    }

    pub fn spawn_supervised<F>(&mut self, task: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let mut shutdown_rx = self.shutdown_tx.subscribe();
        
        let handle = tokio::spawn(async move {
            tokio::select! {
                _ = shutdown_rx.recv() => {
                    info!("Task received shutdown signal");
                }
                result = catch_unwind(AssertUnwindSafe(task)) => {
                    if let Err(e) = result {
                        error!("Task panicked: {:?}", e);
                    }
                }
            }
        });

        self.tasks.push(handle);
    }

    pub async fn shutdown(self) {
        let _ = self.shutdown_tx.send(());
        
        for task in self.tasks {
            if let Err(e) = task.await {
                error!("Error shutting down task: {:?}", e);
            }
        }
    }
}
```

### 3. Error Context and Reporting
```rust
#[derive(Debug)]
pub struct PanicContext {
    pub timestamp: DateTime<Utc>,
    pub thread: String,
    pub backtrace: Backtrace,
    pub message: String,
}

impl PanicContext {
    pub fn capture(panic_info: &panic::PanicInfo) -> Self {
        Self {
            timestamp: Utc::now(),
            thread: thread::current().name()
                .unwrap_or("unnamed")
                .to_string(),
            backtrace: Backtrace::capture(),
            message: panic_info.to_string(),
        }
    }

    pub async fn report(&self) {
        // Implementation for reporting panic context
        // Could send to logging service, monitoring system, etc.
    }
}
```

## Best Practices for Panic Prevention

### 1. Defensive Programming
```rust
fn defensive_array_access(arr: &[i32], index: usize) -> Option<i32> {
    arr.get(index).copied()
}

async fn process_data(data: &[i32]) -> Result<(), Error> {
    for i in 0..data.len() {
        if let Some(value) = defensive_array_access(data, i) {
            process_value(value).await?;
        }
    }
    Ok(())
}
```

### 2. Resource Cleanup with Drop
```rust
struct DatabaseConnection {
    client: Client,
}

impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        // Ensure resources are cleaned up even in case of panic
        if let Err(e) = self.client.close() {
            error!("Error closing database connection: {:?}", e);
        }
    }
}
```

### 3. Structured Logging for Better Debugging
```rust
use tracing::{info, error, instrument};

#[instrument]
async fn critical_operation() -> Result<(), Error> {
    info!("Starting critical operation");
    
    if let Err(e) = perform_operation().await {
        error!(error = ?e, "Critical operation failed");
        return Err(e);
    }
    
    info!("Critical operation completed successfully");
    Ok(())
}
```

## Testing Panic Handling

### 1. Unit Tests for Panic Recovery
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_panic_recovery() {
        let result = catch_unwind(AssertUnwindSafe(async {
            panic!("Test panic");
        })).await;

        assert!(result.is_err());
    }
}
```

### 2. Integration Tests
```rust
#[tokio::test]
async fn test_supervisor_recovery() {
    let mut supervisor = TaskSupervisor::new();
    
    supervisor.spawn_supervised(async {
        panic!("Supervised task panic");
    });

    // Wait for some time
    tokio::time::sleep(Duration::from_secs(1)).await;
    
    // Verify supervisor state
    supervisor.shutdown().await;
}
```

## Production Considerations

### 1. Memory Safety in Panic Situations
```rust
struct SafeResource<T> {
    inner: Option<T>,
}

impl<T> SafeResource<T> {
    pub fn new(resource: T) -> Self {
        Self {
            inner: Some(resource)
        }
    }

    pub async fn use_resource<F, R>(&mut self, f: F) -> Result<R, Error>
    where
        F: FnOnce(&mut T) -> R,
    {
        let resource = self.inner.as_mut()
            .ok_or_else(|| Error::ResourceUnavailable)?;
            
        Ok(f(resource))
    }
}

impl<T> Drop for SafeResource<T> {
    fn drop(&mut self) {
        if let Some(resource) = self.inner.take() {
            // Clean up resource
        }
    }
}
```

### 2. Monitoring and Alerting
```rust
async fn monitor_task_health(
    task_metrics: Arc<TaskMetrics>,
    alert_client: AlertClient,
) {
    loop {
        if task_metrics.panic_count.load(Ordering::Relaxed) > 0 {
            alert_client.send_alert(
                AlertLevel::Critical,
                "Task panic detected",
            ).await;
        }
        
        tokio::time::sleep(Duration::from_secs(60)).await;
    }
}
```

### 3. Graceful Shutdown Implementation
```rust
pub struct Application {
    supervisor: TaskSupervisor,
    shutdown_signal: broadcast::Receiver<()>,
}

impl Application {
    pub async fn run(mut self) -> Result<(), Error> {
        loop {
            tokio::select! {
                _ = self.shutdown_signal.recv() => {
                    info!("Shutdown signal received");
                    break;
                }
                err = self.supervisor.check_tasks() => {
                    if let Err(e) = err {
                        error!("Task error: {:?}", e);
                        break;
                    }
                }
            }
        }

        self.supervisor.shutdown().await;
        Ok(())
    }
}
```

## Conclusion

Proper panic handling in async Rust requires:
1. Understanding different panic scenarios
2. Implementing appropriate recovery mechanisms
3. Using supervisor patterns for task management
4. Ensuring proper resource cleanup
5. Implementing monitoring and alerting
6. Testing panic handling thoroughly
7. Following best practices for prevention

Remember that while panic handling is important, it's better to design your code to avoid panics where possible, using Result and Option types for expected error cases.