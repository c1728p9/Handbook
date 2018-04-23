## Gracefully deinit of async code

### Introduction

This document covers the proper way to cleanup async operations.

Background information
- `Class` is serialized with `mutex`
- `Class::init()` must be called before using an instance of the class and `Class::deinit()` must be called when finished.
- `Class::async_operation()` is used to process interrupts or other events from thread context.

```C
    instance.init();
    ...
    instance.deinit(); // All async tasks need to be completed before this returns
```

```C
void Class::deinit()
{
    mutex.lock();

    disable_interrupts();
    semaphore.wait();      // This could deadlock since lock() prevents release()

    mutex.unlock();
}
```

```C
void Class::async_operation()
{
    mutex.lock();

    handle_deferred_interrupts();
    semaphore.release();

    mutex.unlock();
}
```

### Solution 1 - move wait outside of lock

Move the semaphore wait outside the mutex lock. Also ensure that init() / deinit() are protected with respect to each other.

Precautions
- deinit() / deinit() must not be called with `mutex` locked.

```C
void Class::init()
{
    mutex_init.lock();
    mutex.lock();

    enable_interrupts();

    mutex.unlock();
    mutex_init.unlock();
}

void Class::deinit()
{
    mutex_init.lock();
    mutex.lock();

    disable_interrupts();

    mutex.unlock();

    semaphore.wait();

    mutex_init.unlock();
}
```

### Solution 2 - try lock the mutex from the async operation

Attempt to lock the mutex from the async operation but abort if it cannot be immediately locked.

Precautions
- All calls async_operation() must be serialized externally. This could be running on a queue or running on a dedicated thread.
- The code which is holding the mutex must ensure that handle_deferred_interrupts() is called.

```C
void Class::async_operation()
{
    if (mutex.trylock()) {

        handle_deferred_interrupts();
        semaphore.release();

        mutex.unlock();
    }
}
```

### Solution 3 - extra async operation lock

Add a second mutex which is used only for aborting async operations.

Precautions
- All calls async_operation() must be serialized externally. This could be running on a queue or running on a dedicated thread.
- deinit() must not be called with `mutex` already locked

```C
void Class::deinit()
{
    mutex_async_abort.lock();
    mutex.lock();

    disable_interrupts();
    semaphore.wait();

    mutex.unlock();
    mutex_async_abort.unlock();
}
```

```C
void Class::async_operation()
{
    if (mutex_async_abort.trylock()) {
        mutex.lock();

        handle_deferred_interrupts();

        mutex.unlock();
        mutex_async_abort.unlock();
    }

    semaphore.release();
}
```
