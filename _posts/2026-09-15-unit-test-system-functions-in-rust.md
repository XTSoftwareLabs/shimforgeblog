---
layout: default
title: Unit test Rust code that calls system functions
description: Use shimforge to control time and environment reads without adding test-only traits to production code.
date: 2026-09-15
---

# Unit test Rust code that calls system functions

System functions are useful because they connect a program to the machine it is running on. They are also a common source of brittle tests.

Code that reads `SystemTime::now()` can cross a boundary while a test is running. Code that reads an environment variable can see a value left by another test. Code that calls the file system or a process API can require setup that is slow, platform-specific, or difficult to clean up.

The usual answer is to add an abstraction: define a clock trait, pass it through the call graph, and provide a real implementation in production. That is a good design when the dependency is part of the application's domain. It is unnecessary when the only goal is to test a small function that already calls a standard-library function directly.

Shimforge puts the seam in the test. The production function stays unchanged, while the test replaces the system function for its session.

## A function that reads the clock

Here is ordinary business code. It decides whether the current UTC hour is in the morning:

```rust
use std::time::{SystemTime, UNIX_EPOCH};

fn is_morning() -> bool {
    let seconds = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .expect("the clock is after 1970")
        .as_secs();
    let hour = seconds % 86_400 / 3_600;
    (7..12).contains(&hour)
}
```

Without a seam, this test depends on the time at which it happens to run. A clock trait would make the time controllable, but it would also change the function's signature:

```rust
trait Clock {
    fn now(&self) -> SystemTime;
}

fn is_morning<C: Clock>(clock: &C) -> bool {
    // ...read clock.now() instead of SystemTime::now()...
    # true
}
```

With shimforge, the original function is the code under test:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use shimforge::{mock, Session};
    use std::time::Duration;

    const NEW_YEAR: u64 = 1_767_225_600;

    fn at(seconds: u64) -> SystemTime {
        UNIX_EPOCH + Duration::from_secs(seconds)
    }

    #[test]
    fn morning_is_checked_at_known_times() {
        let mut session = Session::new();
        let now = mock!(session, SystemTime::now, fn() -> SystemTime);

        now.expect().once().returns(at(NEW_YEAR + 10 * 3_600));
        now.expect().once().returns(at(NEW_YEAR + 13 * 3_600));

        assert!(is_morning());
        assert!(!is_morning());
        session.verify();
    }
}
```

The `mock!` call names the function and its signature. The test then supplies two clock readings: one inside the morning range and one outside it. There is no sleeping, date arithmetic in the test setup, or clock trait to thread through production code.

This also makes boundary tests straightforward:

```rust
#[test]
fn noon_is_not_morning() {
    let mut session = Session::new();
    let now = mock!(session, SystemTime::now, fn() -> SystemTime);

    now.expect().once().returns(at(NEW_YEAR + 12 * 3_600));

    assert!(!is_morning());
    session.verify();
}
```

## A function that reads the environment

Environment variables have a different problem: changing the process environment affects every test in that process. In Rust 2024, environment mutation is also an unsafe operation because other threads may read the environment at the same time.

Consider a path formatter that reads `HOME` directly:

```rust
use std::env;
use std::path::{Path, PathBuf};

fn home_dir() -> Option<PathBuf> {
    env::var_os("HOME")
        .filter(|home| !home.is_empty())
        .map(PathBuf::from)
}

fn display_path(path: &Path) -> String {
    let Some(home) = home_dir() else {
        return path.display().to_string();
    };

    match path.strip_prefix(&home) {
        Ok(rest) if rest.as_os_str().is_empty() => "~".to_owned(),
        Ok(rest) => format!("~/{}", rest.to_string_lossy()),
        Err(_) => path.display().to_string(),
    }
}
```

The test can provide a virtual home directory without calling `env::set_var`:

```rust
#[test]
fn a_path_under_home_is_shortened() {
    use shimforge::{mock, Session};
    use std::ffi::OsString;

    let mut session = Session::new();
    let var_os = mock!(
        session,
        env::var_os::<&str>,
        fn(&str) -> Option<OsString>
    );

    var_os
        .expect()
        .with(|name| *name == "HOME")
        .once()
        .returns(Some(OsString::from("/home/ops")));

    assert_eq!(
        display_path(Path::new("/home/ops/projects/shimforge")),
        "~/projects/shimforge"
    );
    session.verify();
}
```

Each `Session::new()` session is local to the current thread. Tests can therefore choose their own environment result without changing the machine environment or coordinating through a global mutex. The expectation also checks that the code asks for `HOME`, rather than silently reading a different variable.

## What about other system calls?

The same pattern applies to direct calls such as:

- `std::fs::read` and `std::fs::create_dir_all`
- `Path::exists` and `Path::is_dir`
- `std::env::current_dir`
- `SystemTime::now`
- functions imported from C or the operating system

Use `mock!` when the test needs argument matching, call counts, or return values. Use `replace!` when a simple replacement is enough:

```rust
fn fixed_process_id() -> u32 {
    std::process::id()
}

fn fake_process_id() -> u32 {
    42
}

#[test]
fn process_id_can_be_replaced() {
    use shimforge::{replace, Session};

    let mut session = Session::new();
    replace!(session, std::process::id => fake_process_id, fn() -> u32);

    assert_eq!(fixed_process_id(), 42);
}
```

The signature in the macro is checked by the compiler. A replacement that accepts the wrong arguments or returns the wrong type fails during compilation, before the test runs.

## Keep the seam where the uncertainty is

Traits and dependency injection are valuable when an application needs multiple implementations or wants the dependency to be explicit in its architecture. For a small function that directly calls the standard library, they can make a narrow test concern spread through otherwise stable code.

Shimforge lets the test control the system call at the point where the uncertainty occurs. The production API stays small, tests can exercise exact boundary conditions, and the real environment remains untouched.

Add shimforge as a development dependency:

```toml
[dev-dependencies]
shimforge = "0.1"
```

## Learn more

- [shimforge home page](https://shimforge.com)
- [shimforge on GitHub](https://github.com/XTSoftwareLabs/shimforge)
