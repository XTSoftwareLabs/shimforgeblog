---
layout: default
title: How to mock the file system in Rust
description: A practical comparison of Mockall and shimforge for testing std::fs calls.
date: 2026-09-14
---

# How to mock the file system in Rust

File system code is simple to write and easy to make flaky in tests. A test that uses the real disk needs temporary paths, cleanup, and platform specific ways to produce errors. It can also leave tests coupled to the machine running them.

This example uses Mockall first, then shimforge. Both can test the behavior. The difference is where the test seam lives.

## The code we want to test

Start with ordinary Rust code:

```rust
use std::fs;
use std::path::Path;

fn prepare_cache(root: &Path) -> Result<(), String> {
    fs::create_dir_all(root)
        .map_err(|error| format!("cannot prepare cache: {error}"))?;
    Ok(())
}
```

We want to test that the function passes the right path and handles a permission error. The function calls `std::fs::create_dir_all` directly. There is no file system abstraction in the production code.

## The Mockall version

Mockall is a good choice when the code already has a trait boundary. To use it here, we first add one:

```rust
use mockall::automock;
use std::io;
use std::path::Path;

#[automock]
trait CacheFileSystem {
    fn create_dir_all(&self, root: &Path) -> io::Result<()>;
}

fn prepare_cache<F: CacheFileSystem>(fs: &F, root: &Path) -> Result<(), String> {
    fs.create_dir_all(root)
        .map_err(|error| format!("cannot prepare cache: {error}"))?;
    Ok(())
}
```

The test is clear:

```rust
#[test]
fn cache_setup_succeeds() {
    let root = Path::new("virtual/cache/reports");
    let mut fs = MockCacheFileSystem::new();
    fs.expect_create_dir_all()
        .withf(move |path| *path == root)
        .times(1)
        .returning(|_| Ok(()));

    assert!(prepare_cache(&fs, root).is_ok());
}
```

But the production function is no longer the original function. Every caller now needs a file system value or a generic parameter. The real implementation also needs a trait implementation:

```rust
struct RealCacheFileSystem;

impl CacheFileSystem for RealCacheFileSystem {
    fn create_dir_all(&self, root: &Path) -> io::Result<()> {
        std::fs::create_dir_all(root)
    }
}
```

That is a reasonable design when choosing a file system is part of the application. For a helper that only needs to call the standard library, it adds a permanent layer for a test concern.

## The shimforge version

Shimforge lets the production function stay as it was. It replaces the function called by the code under test for the session:

```rust
use shimforge::{mock, Session};
use std::io;
use std::path::Path;

#[test]
fn cache_setup_succeeds_without_creating_a_directory() {
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
    assert!(!root.exists());
}
```

The production function still has the small signature from the first example. There is no trait, real implementation, generic parameter, or extra argument to carry through the application.

The error path is just as direct:

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

The test controls the exact `io::Error` returned to the caller. It does not need a read-only directory, special permissions, or a platform specific setup.

## Where shimforge has the edge

For direct file system calls, shimforge has a few practical advantages over the Mockall version.

### No production refactor

Mockall needs a trait for this example. Shimforge works with the function that already exists. That matters when the code is stable, small, or shared by many callers. You can add a test without changing the function signature and then review only the behavior under test.

### No dependency injection through the call graph

With Mockall, the file system object has to reach `prepare_cache`, either as an argument or through a field on another type. As the call graph grows, that value moves through more constructors and methods. Shimforge keeps the test setup at the test boundary.

### It can mock code you do not own

The trait approach works when you can put your own interface in front of a dependency. Shimforge can target a free function from the standard library or another crate directly. That is useful for `std::fs::read`, `std::fs::write`, `File::open`, `Path::exists`, and `Path::is_dir`.

For example, a reader can be tested without a file:

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
fn reads_test_data_without_opening_a_file() {
    let mut session = Session::new_global();
    replace!(
        session,
        fs::read::<&Path> => fake_members,
        fn(&Path) -> io::Result<Vec<u8>>
    );

    assert_eq!(
        load_members(Path::new("virtual/members.txt")).unwrap(),
        ["alice", "bob"]
    );
}
```

The test targets the same `fs::read` call that production code uses. No adapter is needed just to make the dependency mockable.

### The function signature is checked

The signature in `mock!` and `replace!` is checked by the compiler. The mock for `create_dir_all` must accept a `&Path` and return `io::Result<()>`. A replacement with a different signature will fail to compile before the test runs.

That keeps the direct approach readable: the test names the function, states its signature, and supplies the behavior. There is no separately maintained trait that can drift away from the call being replaced.

## Which one should you use?

Mockall remains a strong fit when an interface is already part of the design or when several implementations are a real product feature. For code that directly calls `std::fs`, shimforge has the smaller change: keep the production code intact and replace the narrow call inside the test.

That is the main reason to prefer shimforge for file system seams. It lets the test check paths, calls, bytes, and OS errors without adding a test-only abstraction to the application.

Add shimforge as a development dependency:

```toml
[dev-dependencies]
shimforge = "0.1"
```

The test profile also needs low optimization so calls keep a patchable entry point:

```toml
[profile.test]
opt-level = 0
debug = true
lto = false
codegen-units = 1
incremental = false
```

## Learn more

- [shimforge home page](https://shimforge.com)
- [shimforge on GitHub](https://github.com/XTSoftwareLabs/shimforge)

