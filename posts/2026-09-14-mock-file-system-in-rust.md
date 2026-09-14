---
title: How to mock the file system in Rust
description: Test Rust file system code without touching disk or changing production code.
date: 2026-09-14
---

# How to mock the file system in Rust

File system code is easy to write and awkward to test.

Your code may only need to create a directory, read a file, or handle a permission error. A normal unit test then has to create files, remove them, choose a safe temporary path, and hope another test is not using the same path. Tests can become slow and still miss the errors that happen on a real machine.

This post shows how to test those paths with [shimforge](https://github.com/XTSoftwareLabs/shimforge). The production code keeps calling `std::fs`. The test replaces that call for the life of a session.

## The problem

Imagine a small cache setup function:

```rust
use std::fs;
use std::path::Path;

fn prepare_cache(root: &Path) -> Result<(), String> {
    fs::create_dir_all(root)
        .map_err(|error| format!("cannot prepare cache: {error}"))?;
    Ok(())
}
```

There are two useful paths to test:

- the directory is created successfully;
- the operating system rejects the request.

A test that uses the real disk can cover the first path with a temporary directory. The failure path is harder. You need a read-only location, platform-specific setup, or a test machine that happens to behave the same way.

The test also checks an implementation detail: did the code create the directory that the caller passed in?

## The usual mock-library approach

Many Rust mock libraries work through a trait. You put file operations behind an interface, then pass either a real implementation or a mock implementation to the code under test:

```rust
trait CacheFileSystem {
    fn create_dir_all(&self, path: &Path) -> io::Result<()>;
}

struct RealFileSystem;

impl CacheFileSystem for RealFileSystem {
    fn create_dir_all(&self, path: &Path) -> io::Result<()> {
        std::fs::create_dir_all(path)
    }
}
```

The application function now needs a file system parameter:

```rust
fn prepare_cache<F: CacheFileSystem>(fs: &F, root: &Path) -> Result<(), String> {
    fs.create_dir_all(root)
        .map_err(|error| format!("cannot prepare cache: {error}"))?;
    Ok(())
}
```

That design is a good choice when the file system is part of the business boundary, or when the application needs several interchangeable implementations. It also works on every Rust target without runtime code patching.

For a small helper, though, the trait can be more code than the feature. The trait, real implementation, generic parameter, and extra argument spread through callers. The production code now carries a test seam forever, even if it only ever uses one implementation.

## Mock the standard library call with shimforge

Shimforge can intercept the function that the existing code calls:

```rust
use shimforge::{mock, Session};
use std::io;
use std::path::Path;

#[test]
fn cache_setup_does_not_touch_disk() {
    let root = Path::new("virtual/cache/reports");
    let mut session = Session::new();
    let create = mock!(
        session,
        std::fs::create_dir_all::<&Path>,
        fn(&Path) -> io::Result<()>
    );

    create
        .expect()
        .with(|path| **path == *root)
        .once()
        .returning(|_| Ok(()));

    assert!(prepare_cache(root).is_ok());
    session.verify();
    assert!(!root.exists());
}
```

The test makes three claims:

1. `prepare_cache` calls `create_dir_all`.
2. It passes the expected path.
3. No directory is created on the real disk.

There is no wrapper around `std::fs`, and `prepare_cache` is unchanged.

The failure path is just as direct:

```rust
#[test]
fn cache_setup_reports_permission_errors() {
    let mut session = Session::new();
    let create = mock!(
        session,
        std::fs::create_dir_all::<&Path>,
        fn(&Path) -> io::Result<()>
    );

    create
        .expect()
        .once()
        .returning(|_| Err(io::ErrorKind::PermissionDenied.into()));

    let error = prepare_cache(Path::new("virtual/cache/reports")).unwrap_err();
    assert!(error.starts_with("cannot prepare cache:"));
}
```

The application sees the same `io::Error` it would get from the operating system. The test does not need a read-only directory or special permissions on the host.

## Use `replace!` when you only need a fake function

`mock!` is useful when the test needs argument matching or call counts. When the replacement itself is the important part, `replace!` is shorter:

```rust
use shimforge::{replace, Session};
use std::fs;
use std::io;
use std::path::Path;

fn load_members(path: &Path) -> io::Result<Vec<String>> {
    let bytes = fs::read(path)?;
    Ok(String::from_utf8_lossy(&bytes)
        .lines()
        .map(str::to_owned)
        .collect())
}

fn fake_members(_: &Path) -> io::Result<Vec<u8>> {
    Ok(b"alice\nbob\n".to_vec())
}

#[test]
fn reading_members_uses_test_data() {
    let mut session = Session::new_global();
    replace!(
        session,
        fs::read::<&Path> => fake_members,
        fn(&Path) -> io::Result<Vec<u8>>
    );

    assert_eq!(load_members(Path::new("virtual/members.txt")).unwrap(), ["alice", "bob"]);
}
```

The signature in the macro is checked at compile time. A replacement with the wrong arguments or return type does not silently install.

You can also capture what the application writes:

```rust
fn capture_write(path: &Path, data: &[u8]) -> io::Result<()> {
    assert_eq!(path, Path::new("virtual/members.txt"));
    assert_eq!(data, b"alice\nbob\n");
    Ok(())
}

replace!(
    session,
    fs::write::<&Path, &[u8]> => capture_write,
    fn(&Path, &[u8]) -> io::Result<()>
);
```

This checks the generated path and bytes without opening a file.

## Why shimforge is often the better fit

For code that directly uses `std::fs`, shimforge keeps the test close to the code that matters. That has practical benefits:

- Production functions keep their simple signatures.
- Tests cover OS errors without changing permissions or relying on a particular file system.
- The mock can check paths, bytes, call counts, and returned errors.
- The test can intercept a function in code you do not own, including a dependency's free function.
- The test can be added after the production code exists. You do not need to design a trait before you know what needs testing.

This is why shimforge is recommended for narrow seams such as file reads, writes, directory creation, clocks, and environment lookups. It removes test-only plumbing from the application while keeping the test specific.

A trait remains a better choice when multiple file system implementations are a real product feature, when the boundary must be visible in the type system, or when the code must run under tools that cannot execute runtime patches. Shimforge is a testing tool, not a reason to hide an important domain boundary.

## Local sessions and parallel tests

`Session::new()` creates a thread-local session. The mock affects the current test thread, so unrelated tests can use their own sessions. A global session created with `Session::new_global()` affects all threads and takes an exclusive lock. Use it when the code starts work on threads that the test does not control.

Keep the session alive for every call that should be mocked. Dropping it restores the original function and checks the expectations. `session.verify()` lets the test check them earlier.

Before adding shimforge, put these settings in the workspace root `Cargo.toml`:

```toml
[profile.test]
opt-level = 0
debug = true
lto = false
codegen-units = 1
incremental = false
```

Then add the crate as a development dependency:

```toml
[dev-dependencies]
shimforge = "0.1"
```

Install the first mock before other threads call that function. Later local installs and cleanup do not rewrite the code, but the first patch still needs that target to be idle while it is installed.

## The trade-off

Shimforge changes function entry points at runtime, so it has rules that a trait mock does not. Keep calls away from a target during the first installation, use a global session for work that crosses threads, and keep the test profile settings above. Its safe macros check function signatures, but runtime patching still cannot protect every invariant inside arbitrary code.

For a small piece of file system code, those rules are usually a fair trade. You get a focused test, deterministic errors, and production code that does not need a test-only abstraction.

