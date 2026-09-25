---
layout: default
title: Test hardware drivers without hardware
description: Use shimforge to test Rust wrappers around unsafe C APIs without touching a device.
date: 2026-09-25
---

# Test hardware drivers without hardware

Hardware code often has a small Rust wrapper around a vendor C function. The wrapper passes a pointer, reads a status value, and turns a return code into a Rust result.

The difficult part is testing the wrapper. A real call may need a device, a loaded driver, a file descriptor, or a board connected to the machine running the test. Error paths are even harder to reach.

Shimforge can replace the imported C function for one test session. The Rust wrapper stays unchanged, but the test controls the status code and the data written through the pointer.

## A hardware wrapper

Assume a vendor library exposes this function. It reads a status register into the pointer supplied by the caller:

```rust
use std::ffi::c_int;

// This symbol is provided by the vendor library.
unsafe extern "C" {
    fn board_read_status(fd: c_int, status: *mut u32) -> c_int;
}
```

The wrapper turns the C result into a small Rust API:

```rust
const BOARD_READY: u32 = 1;

fn board_is_ready(fd: c_int) -> Result<bool, &'static str> {
    let mut status = 0;
    let code = unsafe {
        // SAFETY: status points to a live local value with space for one u32.
        board_read_status(fd, &mut status)
    };

    if code != 0 {
        return Err("board_read_status failed");
    }

    Ok(status & BOARD_READY != 0)
}
```

This is the code we want to test. It should report a ready board when the bit is set and return an error when the C call fails.

## Replace the unsafe C call

Add shimforge as a development dependency:

```toml
[dev-dependencies]
shimforge = "0.1"
```

`mock!` accepts an `unsafe extern "C" fn` signature. Installing the mock does not need an `unsafe` block. The call inside the wrapper still does, because that is where the unsafe operation happens.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use shimforge::{mock, Session};

    #[test]
    fn ready_bit_is_decoded() {
        let mut session = Session::new();
        let read = mock!(
            session,
            board_read_status,
            unsafe extern "C" fn(c_int, *mut u32) -> c_int
        );

        read.expect()
            .with(|fd, status| *fd == 7 && !(*status).is_null())
            .once()
            .returning(|_, status| {
                // SAFETY: the wrapper passes a valid pointer to its local status value.
                unsafe { *status = BOARD_READY; }
                0
            });

        assert_eq!(board_is_ready(7), Ok(true));
        session.verify();
    }

    #[test]
    fn c_error_becomes_a_rust_error() {
        let mut session = Session::new();
        let read = mock!(
            session,
            board_read_status,
            unsafe extern "C" fn(c_int, *mut u32) -> c_int
        );
        read.expect().once().returns(-1);

        assert_eq!(board_is_ready(7), Err("board_read_status failed"));
        session.verify();
    }
}
```

The tests never call the board. They still check the important parts of the wrapper: the file descriptor is passed through, the output pointer is valid, the ready bit is decoded, and a C error is returned to the caller.

The same pattern works for register reads, device writes, driver queries, and C runtime calls such as `read`. A mock can fill an output buffer, return a short read, or report a hardware error on demand.

If the wrapper runs on a worker thread, create the session with `Session::new_global()`. A normal `Session::new()` applies to the current test thread; a global session also covers threads started by the code under test.

## Keep the hardware boundary small

Shimforge does not make unsafe code safe. It makes the boundary testable. The unsafe call remains visible in one wrapper, while the tests cover the decisions made after that call without requiring a device.

That gives low-level code ordinary unit tests for success, bad status values, short reads, and driver errors. Hardware testing still has its place, but the wrapper no longer has to wait for a board before its basic behavior can be checked.

## Learn more

- [Shimforge system and C runtime functions](https://github.com/XTSoftwareLabs/shimforge#system-and-c-runtime-functions)
- [shimforge home page](https://shimforge.com)
- [shimforge on GitHub](https://github.com/XTSoftwareLabs/shimforge)
